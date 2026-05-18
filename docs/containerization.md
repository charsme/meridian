# Containerization Spec — Meridian

**Status:** spec parked, not implemented.
**Owner:** Charis.
**Last reviewed:** 2026-05-18 against branch `prod` @ `a3bb210`.

Decisions confirmed with operator on 2026-05-18 are tagged `[CONFIRMED]`. Defaults that the operator should re-confirm before implementation are tagged `[REVIEW]`. Anything not explicitly confirmed should be ELEVATED before coding — do NOT assume.

---

## 1. Goals & Non-Goals

**Goals**
- Run Meridian as containers on a Linux host without OOM-killing the host.
- Persist all accumulated learning state across container restarts.
- Co-locate the `discord-listener/` Node app as a sibling container.
- Reproducible builds via Git short SHA tags.
- Keep wallet private key out of image layers.

**Non-goals**
- External HTTP exposure / Traefik labels. `[CONFIRMED]` Meridian is outbound-only.
- Switching Telegram from long-poll to webhook.
- Database service. Repo is file-state-only.
- Replacing the runtime knowledge accumulation files with anything else.

---

## 2. Confirmed Inputs (from operator, 2026-05-18)

| Decision | Value |
|---|---|
| Volume root for data | `/data/volume/meridian` `[CONFIRMED]` |
| Volume root for logs | `/data/logs/meridian` `[CONFIRMED]` |
| External exposure | None. No Traefik. `[CONFIRMED]` |
| Discord listener layout | Separate container in same compose stack `[CONFIRMED]` |
| Trader memory limit | 2G hard / 1.5G soft `[CONFIRMED]` |
| Discord listener limits | 512M hard / 256M soft, 0.5 CPU `[CONFIRMED]` |
| Wallet secret delivery | `envcrypt.js` encrypted `.envrypt` + `ENVCRYPT_KEY` passphrase `[CONFIRMED]` |
| `ENVCRYPT_KEY` delivery | Separate plain `.env` on host (`/data/volume/meridian/.envrypt-key.env`), mounted via `env_file`. File mode 600. `[CONFIRMED]` |
| Base image | `node:20-bookworm-slim` `[CONFIRMED]` |
| Process supervisor | Drop PM2. `tini` as PID 1, `node index.js` as child. `[CONFIRMED]` |
| Timezone | `Asia/Jakarta` (WIB, UTC+7) `[CONFIRMED]` |
| Image tag strategy | Git short SHA, e.g. `meridian:a3bb210`. Compose reads `:${MERIDIAN_TAG}` via stack `.env`. `[CONFIRMED]` |
| Database | None. Confirmed not needed. `[CONFIRMED]` |

---

## 3. Pending Defaults — Operator Must Re-Confirm

These are necessary to write the spec but not previously discussed. ELEVATE if any look wrong.

| Item | Default proposed | `[REVIEW]` reason |
|---|---|---|
| Trader CPU limit | 1.5 CPU hard | Node is single-threaded; cron + Telegram poll + RPC parallelism rarely saturate more than 1 core. 1.5 leaves margin for prompt-build spikes without claiming a whole core. |
| Restart policy | `unless-stopped` | Standard. Survives Docker daemon restart but respects manual stop. |
| Compose project name | `meridian` | Stack name in `docker compose -p meridian`. |
| Container network | Single bridge network `meridian-net` | Internal-only. Trader ↔ listener both attached. No port publishing. |
| Log driver | `json-file` with `max-size=20m`, `max-file=5` | Docker-side rotation. App-side daily rotation in `logs/` is the canonical log; container stderr/stdout caps prevent runaway disk on json-file. |
| Healthcheck | `node -e "const s = require('fs').statSync('state.json'); if (Date.now() - s.mtimeMs > 30 * 60 * 1000) process.exit(1);"` every 60s, timeout 10s, start-period 90s, retries 3 | Asserts `state.json` mtime within last 30 min. Cron writes it on every management cycle, so a stuck process surfaces. |
| Anchor patch | Run during builder stage after `npm ci` | Patches `node_modules/@coral-xyz/anchor`; output copied into runtime. |
| node_modules in runtime | Copied from builder; runtime does NOT re-run `npm ci` | Reproducibility; smaller runtime image. |

---

## 4. Host Filesystem Layout

```
/data/volume/meridian/
├── user-config.json              # mutated at runtime by update_config tool
├── state.json                    # live position registry
├── lessons.json                  # performance history + derived lessons
├── pool-memory.json              # per-pool win rates, cooldowns, snapshots
├── strategy-library.json
├── smart-wallets.json
├── token-blacklist.json
├── deployer-blacklist.json
├── dev-blocklist.json
├── signal-weights.json
├── decision-log.json
├── hivemind-cache.json
├── discord-signals.json          # written by listener, read by trader (shared between containers)
├── .envrypt                      # encrypted env blob, owner mode 600
└── .envrypt-key.env              # ENVCRYPT_KEY=... only, owner mode 600

/data/logs/meridian/
├── agent-YYYY-MM-DD.log          # daily-rotating, written by logger.js
└── actions-YYYY-MM-DD.jsonl      # action audit trail
```

Host-side prep (one-time, before first `docker compose up`):

```sh
sudo mkdir -p /data/volume/meridian /data/logs/meridian
sudo chown -R 1000:1000 /data/volume/meridian /data/logs/meridian   # container UID
sudo chmod 700 /data/volume/meridian
sudo chmod 755 /data/logs/meridian
# place .envrypt (encrypted) and .envrypt-key.env (with ENVCRYPT_KEY=) into /data/volume/meridian
sudo chmod 600 /data/volume/meridian/.envrypt /data/volume/meridian/.envrypt-key.env
```

---

## 5. Dockerfile (trader image)

Path: repo root `Dockerfile`. Multi-stage.

```dockerfile
# syntax=docker/dockerfile:1.7

# ───────── Stage 1: builder ─────────
FROM node:20-bookworm-slim AS builder

# Native build tools for bn.js, bs58, anchor patch
RUN apt-get update && apt-get install -y --no-install-recommends \
      python3 make g++ ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Dep install (postinstall runs scripts/patch-anchor.js automatically)
COPY package.json package-lock.json ./
COPY scripts ./scripts
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

# Source
COPY . .

# ───────── Stage 2: runtime ─────────
FROM node:20-bookworm-slim AS runtime

# tini for proper PID 1 + signal forwarding
RUN apt-get update && apt-get install -y --no-install-recommends tini ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -g 1000 meridian \
    && useradd -m -u 1000 -g 1000 -s /usr/sbin/nologin meridian

ENV NODE_ENV=production \
    TZ=Asia/Jakarta \
    USER_CONFIG_PATH=/data/user-config.json

WORKDIR /app

# Copy patched node_modules + source from builder
COPY --from=builder --chown=meridian:meridian /app /app

# State + logs live on bind-mounted host volumes (declared in compose)
# Symlink the cwd-relative paths the code uses to the persistent locations.
# state.js/lessons.js/etc write `./state.json` etc with cwd = WORKDIR.
# Easier: set cwd to the volume and copy source to a non-volume path? No —
# cleaner is to symlink each well-known file. See entrypoint.sh.

COPY docker/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

USER meridian

# Healthcheck against state.json freshness
HEALTHCHECK --interval=60s --timeout=10s --start-period=90s --retries=3 \
  CMD node -e "const s=require('fs').statSync('/data/state.json'); if(Date.now()-s.mtimeMs>1800000)process.exit(1);" || exit 1

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/entrypoint.sh"]
CMD ["node", "index.js"]
```

### entrypoint.sh

Decrypts `.envrypt` using `ENVCRYPT_KEY`, then symlinks cwd-relative state files to the volume.

```sh
#!/usr/bin/env sh
set -eu

# 1. Decrypt env into process env. envcrypt.js exports decrypted KEY=VAL pairs.
if [ -z "${ENVCRYPT_KEY:-}" ]; then
  echo "FATAL: ENVCRYPT_KEY not set" >&2
  exit 1
fi
if [ ! -f /data/.envrypt ]; then
  echo "FATAL: /data/.envrypt missing" >&2
  exit 1
fi

# Source the decrypted vars. envcrypt.js prints `export KEY=VAL` lines.
eval "$(node /app/envcrypt.js decrypt --file /data/.envrypt --key "$ENVCRYPT_KEY" --format export)"
unset ENVCRYPT_KEY   # passphrase no longer needed in process env

# 2. Wire cwd-relative state file paths to the persistent volume.
# These names match the const declarations in state.js / lessons.js / pool-memory.js / etc.
cd /app
for f in state.json lessons.json pool-memory.json strategy-library.json \
         smart-wallets.json token-blacklist.json deployer-blacklist.json \
         dev-blocklist.json signal-weights.json decision-log.json \
         hivemind-cache.json discord-signals.json user-config.json; do
  rm -f "/app/$f"
  ln -sfn "/data/$f" "/app/$f"
done

# 3. Logger writes to ./logs. Symlink to persistent log volume.
rm -rf /app/logs
ln -sfn /var/log/meridian /app/logs

# 4. Hand off to node.
exec "$@"
```

`[REVIEW]` `envcrypt.js` must support a `decrypt --file --key --format export` invocation. If it doesn't, ELEVATE — entrypoint needs adjusting OR add a thin wrapper script. Do NOT assume the CLI shape.

---

## 6. Dockerfile (discord-listener image)

Path: `discord-listener/Dockerfile`. Simpler — smaller dep tree.

```dockerfile
# syntax=docker/dockerfile:1.7

FROM node:20-bookworm-slim AS builder

WORKDIR /app
COPY discord-listener/package.json discord-listener/package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

COPY discord-listener/ ./

# ─────────
FROM node:20-bookworm-slim AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends tini ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -g 1000 meridian \
    && useradd -m -u 1000 -g 1000 -s /usr/sbin/nologin meridian

ENV NODE_ENV=production \
    TZ=Asia/Jakarta

WORKDIR /app
COPY --from=builder --chown=meridian:meridian /app /app

USER meridian

HEALTHCHECK --interval=60s --timeout=10s --start-period=60s --retries=3 \
  CMD node -e "if(!require('fs').existsSync('/data/discord-signals.json'))process.exit(1);" || exit 1

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "index.js"]
```

---

## 7. docker-compose.yml

Path: repo root `docker-compose.yml`.

```yaml
name: meridian

services:
  trader:
    image: meridian:${MERIDIAN_TAG}
    build:
      context: .
      dockerfile: Dockerfile
    container_name: meridian-trader
    restart: unless-stopped
    env_file:
      - /data/volume/meridian/.envrypt-key.env   # contains ENVCRYPT_KEY=...
    environment:
      TZ: Asia/Jakarta
    volumes:
      - /data/volume/meridian:/data:rw
      - /data/logs/meridian:/var/log/meridian:rw
    networks:
      - meridian-net
    mem_limit: 2g
    mem_reservation: 1536m
    cpus: 1.5
    pids_limit: 512
    read_only: false                # state files need write
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "5"
    depends_on:
      - discord-listener

  discord-listener:
    image: meridian-discord-listener:${MERIDIAN_TAG}
    build:
      context: .
      dockerfile: discord-listener/Dockerfile
    container_name: meridian-discord-listener
    restart: unless-stopped
    env_file:
      - /data/volume/meridian/.envrypt-key.env   # same key; listener can decrypt its own subset
    environment:
      TZ: Asia/Jakarta
    volumes:
      - /data/volume/meridian:/data:rw           # shared discord-signals.json
      - /data/logs/meridian:/var/log/meridian:rw
    networks:
      - meridian-net
    mem_limit: 512m
    mem_reservation: 256m
    cpus: 0.5
    pids_limit: 256
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

networks:
  meridian-net:
    name: meridian-net
    driver: bridge
```

### Stack `.env` (next to `docker-compose.yml`, gitignored)

```
MERIDIAN_TAG=a3bb210
```

`[REVIEW]` `.gitignore` already covers `.env` and `.env.*` at repo root. The compose-stack `.env` will be picked up automatically by Docker Compose. Confirm operator wants this same file or a separate `.env.compose`.

---

## 8. Resource Limits — OOM Posture

Confirmed `[CONFIRMED]`:

| Container | mem hard | mem soft | cpus | pids |
|---|---|---|---|---|
| `trader` | 2G | 1.5G | 1.5 `[REVIEW]` | 512 |
| `discord-listener` | 512M | 256M | 0.5 | 256 |

**Total host budget:** 2.5G memory + 2 CPU at hard caps. Confirm host has ≥4G RAM and ≥4 vCPU free. ELEVATE if host is tighter — these limits assume a host that can absorb both at full burn.

`mem_reservation` (soft) is what Docker tries to keep the container above when memory pressure hits; `mem_limit` is the hard OOM-kill threshold. Soft < hard means: under pressure, Docker reclaims first from containers exceeding their reservation; only OOM-kills when a container itself blows past its own hard limit. Host stays safe.

Node will throw `JavaScript heap out of memory` before hitting cgroup OOM-kill IF V8 heap size is capped. Add to trader env to make it surface gracefully:

```
NODE_OPTIONS=--max-old-space-size=1536
```

This keeps V8 heap under the soft reservation, leaving 512MB for buffers, native modules, and the rest of the process. Surfaces as a clean Node error before cgroup kills the process.

---

## 9. Secret Handling

### Files on host
- `/data/volume/meridian/.envrypt` — encrypted blob containing `WALLET_PRIVATE_KEY`, `RPC_URL`, `OPENROUTER_API_KEY`, optional `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `HELIUS_API_KEY`, `HIVE_MIND_URL`, `HIVE_MIND_API_KEY`. Mode 600.
- `/data/volume/meridian/.envrypt-key.env` — single line `ENVCRYPT_KEY=...`. Mode 600. Mounted as `env_file`.

### Flow
1. Compose mounts `.envrypt-key.env` → container env has `ENVCRYPT_KEY`.
2. `entrypoint.sh` calls `node envcrypt.js decrypt` to recover the plaintext env into the process.
3. `entrypoint.sh` unsets `ENVCRYPT_KEY` from process env to avoid leak through `/proc/self/environ`.
4. `exec node index.js` — node sees all required env vars, no on-disk plaintext.

### Risks
- Both files on the same host filesystem — defense in depth is limited. Attacker with host root reads both.
- Image layer never contains either file (only mounted at runtime).
- `docker inspect` will reveal `env_file` source path but NOT contents.

`[REVIEW]` Confirm `envcrypt.js` supports the CLI shape used in `entrypoint.sh`. If not, ELEVATE — do not silently substitute.

---

## 10. Path-Style Quirk Handling

Meridian code mixes:
- `path.join(__dirname, "foo.json")` — anchored to repo dir
- `"./foo.json"` — cwd-relative (e.g. `state.js:14`, `lessons.js:18`, `pool-memory.js:12`)

Approach: container `WORKDIR=/app`, source at `/app`, and `entrypoint.sh` symlinks each state file from `/app/<file>.json` → `/data/<file>.json`. Writes through the symlink land on the persistent volume.

Why symlinks instead of `cwd=/data`: source files need to stay readable from `/app`; copying source onto the volume on every boot is wasteful and risks state-file confusion. Symlinks keep one source-of-truth for code (`/app`) and one for state (`/data`).

Same approach for `logs/` → `/var/log/meridian`.

---

## 11. Build & Deploy Flow

```sh
# One-time host prep
sudo mkdir -p /data/volume/meridian /data/logs/meridian
sudo chown -R 1000:1000 /data/volume/meridian /data/logs/meridian
# place .envrypt and .envrypt-key.env into /data/volume/meridian, chmod 600

# Build
TAG=$(git rev-parse --short HEAD)
echo "MERIDIAN_TAG=$TAG" > .env       # next to docker-compose.yml
docker compose build

# Run
docker compose up -d

# Logs
docker compose logs -f trader
docker compose logs -f discord-listener

# Stop
docker compose down                   # state persists on /data/volume/meridian
```

Rollback:

```sh
echo "MERIDIAN_TAG=<prev-sha>" > .env
docker compose up -d                  # re-pulls/builds previous tag
```

---

## 12. Migration from PM2 to Container

1. `pm2 stop meridian && pm2 delete meridian` on host.
2. `pm2 list` confirms removal.
3. Copy state files into `/data/volume/meridian` (from repo root if PM2 was running there):
   ```sh
   for f in state.json lessons.json pool-memory.json strategy-library.json \
            smart-wallets.json token-blacklist.json deployer-blacklist.json \
            dev-blocklist.json signal-weights.json decision-log.json \
            hivemind-cache.json discord-signals.json user-config.json; do
     [ -f "/path/to/repo/$f" ] && sudo cp -a "/path/to/repo/$f" "/data/volume/meridian/$f"
   done
   sudo cp -a /path/to/repo/logs/* /data/logs/meridian/ 2>/dev/null || true
   sudo chown -R 1000:1000 /data/volume/meridian /data/logs/meridian
   ```
4. Encrypt `.env` with `envcrypt.js`, place at `/data/volume/meridian/.envrypt`. Create `.envrypt-key.env`.
5. `docker compose up -d`.
6. Watch `docker compose logs -f trader` for the first management cycle. Confirm `state.json` mtime updates → healthcheck stays green.

ELEVATE if any state file is held open by the PM2 process during copy. Prefer stopping PM2 fully first; never copy a live-written JSON.

---

## 13. Open Items / ELEVATE Before Implementing

1. `envcrypt.js` CLI shape — verify `decrypt --file --key --format export` works, OR rewrite `entrypoint.sh` to match the actual CLI. Do NOT assume.
2. Trader CPU limit `1.5` — measure actual usage in a non-prod run for ≥24h before locking. May need to bump to 2 if RPC concurrency saturates.
3. `NODE_OPTIONS=--max-old-space-size=1536` — assumes 2G hard / 1.5G soft. Re-tune if mem caps change.
4. Healthcheck threshold `30 min` — assumes `managementIntervalMin` defaults around 10 min. If the operator sets a longer interval, the healthcheck false-positives. Read `managementIntervalMin` from config or widen threshold. ELEVATE.
5. `discord-signals.json` write semantics — confirm listener writes are atomic (rename-over) so trader doesn't read a half-written file. If non-atomic, add file locking or switch to a named pipe. ELEVATE if unclear.
6. Whether `discord-listener` should be allowed to crash without restarting `trader` — current compose has `depends_on` only for start ordering, not restart coupling. Confirm OK.
7. CI: building images on every commit vs operator-triggered. Out of this spec; add a `.github/workflows/build.yml` separately when CI is wanted.
8. Backup story for `/data/volume/meridian` — nightly tarball to remote? Out of this spec but ELEVATE: losing this volume = lobotomizing the agent.

---

## 14. Acceptance Checklist (run after first deploy)

- [ ] `docker compose ps` shows both `trader` and `discord-listener` `healthy`.
- [ ] `docker compose exec trader ls -l /data/state.json` shows recent mtime.
- [ ] `docker compose exec trader env | grep -E 'WALLET|RPC|OPENROUTER'` confirms env loaded.
- [ ] `docker compose exec trader env | grep ENVCRYPT_KEY` returns EMPTY (passphrase unset post-decrypt).
- [ ] Telegram `/positions` command responds.
- [ ] Force a screening cycle (REPL or wait for cron); confirm `state.json` mtime updates and no errors in `docker compose logs trader`.
- [ ] Stop and restart: `docker compose down && docker compose up -d`. Confirm `state.json` is preserved and the agent resumes without re-deploying duplicate positions.
- [ ] Kill the trader container (`docker kill --signal=SIGKILL meridian-trader`). Confirm restart policy brings it back. Confirm no state corruption.
- [ ] Memory soak: leave running 48h. `docker stats` should show RSS staying under the soft reservation in steady state.

---

## 15. Related Documents

- `docs/lorekeeper-integration.md` — separate deferred TODO for semantic recall layer.
- `CLAUDE.md` "TODO" section — pointers to this spec.
- loreKeeper memories (namespace `private/trading`):
  - `meridian-architecture` (id `35a3d7d4-3a7c-4711-b64e-ae7058b795b7`)
  - `meridian-learning-loops` (id `7dd63896-577d-4e6f-9db6-b33355b220ec`)
  - `meridian-ops-decisions` (id `38b4c79e-fe13-4f65-8635-4e337f0a7955`)
