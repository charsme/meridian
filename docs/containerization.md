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
| Wallet secret delivery | Plain `.env` mounted from host. `envcrypt.js` available as future hardening (see §9). `[CONFIRMED 2026-05-18 round 4]` after CLI shape verification |
| Encrypted-env passphrase delivery | N/A for v1 (plain .env). When hardening: `ENVCRYPT_KEY` env var via compose `environment:`. |
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
| Healthcheck | `node docker/healthcheck.js` every 60s, timeout 10s, start-period 90s, retries 3 | Reads `managementIntervalMin` from `/data/user-config.json` and treats `state.json` as stale if mtime exceeds `max(2 × managementIntervalMin, 30) min`. Operator override via `HEALTHCHECK_STALE_MIN` env. See §5 for script. |
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
└── .env                          # plain dotenv file, owner mode 600
                                  # (when hardened: mixed plaintext + base64 entries
                                  #  with `# encrypted` markers, plus ENVRYPT_KEY env or /data/.envrypt key file)

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

# Plain .env (v1 default). When hardening later, see §9.
sudo install -m 600 -o 1000 -g 1000 /dev/null /data/volume/meridian/.env
sudoedit /data/volume/meridian/.env   # paste WALLET_PRIVATE_KEY, RPC_URL, OPENROUTER_API_KEY, etc
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

# Healthcheck — reads managementIntervalMin from /data/user-config.json + env override
HEALTHCHECK --interval=60s --timeout=10s --start-period=90s --retries=3 \
  CMD node /app/docker/healthcheck.js || exit 1

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/entrypoint.sh"]
CMD ["node", "index.js"]
```

### entrypoint.sh

Symlinks cwd-relative state files + `.env` + `logs/` to the persistent volume. No decryption step — `envcrypt.js` runs automatically at `index.js` module load via `import "./envcrypt.js"` (see `index.js:1`) and decrypts marked entries in-process.

```sh
#!/usr/bin/env sh
set -eu

cd /app

# 1. Wire cwd-relative state file paths to the persistent volume.
# These names match the const declarations in state.js / lessons.js / pool-memory.js / etc.
for f in state.json lessons.json pool-memory.json strategy-library.json \
         smart-wallets.json token-blacklist.json deployer-blacklist.json \
         dev-blocklist.json signal-weights.json decision-log.json \
         hivemind-cache.json discord-signals.json user-config.json; do
  rm -f "/app/$f"
  ln -sfn "/data/$f" "/app/$f"
done

# 2. dotenv reads from cwd ./.env (envcrypt.js DEFAULT_ENV_PATH = path.join(cwd, ".env")).
ln -sfn /data/.env /app/.env

# 3. Logger writes to ./logs. Symlink to persistent log volume.
rm -rf /app/logs
ln -sfn /var/log/meridian /app/logs

# 4. Hand off to node. envcrypt.js loadEnv() runs automatically at import time.
exec "$@"
```

### docker/healthcheck.js

```js
#!/usr/bin/env node
import fs from "fs";

const STATE_PATH = "/data/state.json";
const CONFIG_PATH = "/data/user-config.json";

function readIntervalMin() {
  // Env override wins. Default fallback is 30 min until config exists.
  const override = Number(process.env.HEALTHCHECK_STALE_MIN);
  if (Number.isFinite(override) && override > 0) return override;

  try {
    const cfg = JSON.parse(fs.readFileSync(CONFIG_PATH, "utf8"));
    const mi = Number(cfg.managementIntervalMin ?? cfg.schedule?.managementIntervalMin);
    if (Number.isFinite(mi) && mi > 0) {
      // Stale window = 2× management interval, floored at 30 min so short cycles don't false-positive on a slow tick.
      return Math.max(mi * 2, 30);
    }
  } catch {
    /* missing or unreadable config — fall through to default */
  }
  return 30;
}

function main() {
  let stat;
  try {
    stat = fs.statSync(STATE_PATH);
  } catch {
    // Until first management cycle writes state.json, allow the start-period to cover it.
    process.exit(1);
  }
  const staleMs = readIntervalMin() * 60 * 1000;
  if (Date.now() - stat.mtimeMs > staleMs) process.exit(1);
  process.exit(0);
}

main();
```

`[CONFIRMED]` `envcrypt.js` CLI shape verified 2026-05-18. Module exports `loadEnv()` and auto-runs it on import (`envcrypt.js:121`). `index.js:1`, `cli.js:7`, `setup.js:7` all import it. Encryption model = per-key inline at module load, NOT whole-file external decryption. v1 spec uses plain `.env`; encryption is opt-in hardening (see §9).

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
    # NOTE: do NOT set env_file here. dotenv reads /app/.env at runtime
    # (symlinked to /data/.env by entrypoint). Putting it under env_file
    # would double-load + bypass envcrypt decryption on encrypted entries.
    environment:
      TZ: Asia/Jakarta
      NODE_OPTIONS: "--max-old-space-size=1536"
      # HEALTHCHECK_STALE_MIN: 60     # uncomment to override default 2×managementIntervalMin
      # ENVRYPT_KEY: "${ENVRYPT_KEY}" # ONLY when hardening (§9). Pass-through from host shell.
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
    # Listener reads its own subset of env from /data/.env (mounted below).
    environment:
      TZ: Asia/Jakarta
    volumes:
      - /data/volume/meridian:/data:rw           # shared discord-signals.json + .env
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

### v1 default — plain `.env`

- `/data/volume/meridian/.env` — plain dotenv file, mode 600, UID/GID 1000.
- Mounted into the container via the `/data/volume/meridian → /data` bind mount.
- `entrypoint.sh` symlinks `/data/.env → /app/.env` so dotenv (which `envcrypt.js` calls with `DEFAULT_ENV_PATH = path.join(cwd, ".env")`) finds it.
- `index.js:1` imports `envcrypt.js` → `loadEnv()` runs → dotenv populates `process.env`. No marked-encrypted entries → no passphrase needed → boots.

Required entries (mirror existing `.env.example`):
```
WALLET_PRIVATE_KEY=...
RPC_URL=https://...
OPENROUTER_API_KEY=...
# optional
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...
HELIUS_API_KEY=...
HIVE_MIND_URL=...
HIVE_MIND_API_KEY=...
LLM_MODEL=...
LLM_BASE_URL=...
```

Risk posture: secret on host disk at mode 600. Attacker with host root reads it. No image-layer leakage. `docker inspect` does NOT reveal contents (no `env_file` directive used).

### Future hardening — `envcrypt.js` in-process decryption

The repo already supports per-key encryption via `envcrypt.js`. Model (verified 2026-05-18 by reading `/data/repos/charsme/meridian/envcrypt.js`):

- `.env` itself is the source-of-truth file. It contains mixed plaintext and base64-encrypted entries.
- Each encrypted entry is preceded by a `# encrypted` marker line. Example:
  ```
  TELEGRAM_BOT_TOKEN=plain-here
  # encrypted
  WALLET_PRIVATE_KEY=base64xxxx
  ```
- Passphrase lookup order at boot: `process.env.ENVRYPT_KEY` → `process.env.ENVCRYPT_KEY` → `./.envrypt` file (passphrase as plaintext). Minimum 8 chars enforced.
- `loadEnv()` decrypts marked entries IN PROCESS at module load. No external CLI step needed.

To enable for a deployed container:

1. On host: `cd /path/to/repo && cp /data/volume/meridian/.env ./.env.raw && ENVRYPT_KEY=<chosen-passphrase> node scripts/envrypt.js encrypt .env.raw .env.encrypted`.
2. Replace `/data/volume/meridian/.env` with `.env.encrypted` (renamed to `.env`).
3. Provide passphrase via compose `environment: ENVRYPT_KEY: "${ENVRYPT_KEY}"` (pass through from operator shell at `docker compose up`); OR drop a `/data/volume/meridian/.envrypt` file containing only the passphrase line (mode 600) and add an entrypoint symlink to `/app/.envrypt`.
4. Verify boot: `docker compose logs trader` should show the agent starting without "no envrypt key" errors.

Risks: passphrase still on host (either in shell history or in `.envrypt`). Defense in depth limited but stronger than plain — an attacker who only steals the `.env` cannot use it without the separate passphrase channel.

`[REVIEW]` Confirm operator wants v1 to ship with plain `.env` and harden later, vs. enable encryption from day one. Default proposed: plain v1 → harden after first stable deploy.

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
sudo chmod 700 /data/volume/meridian
sudo install -m 600 -o 1000 -g 1000 /dev/null /data/volume/meridian/.env
sudoedit /data/volume/meridian/.env   # paste secrets, save

# Build
TAG=$(git rev-parse --short HEAD)
echo "MERIDIAN_TAG=$TAG" > .env.stack  # next to docker-compose.yml
                                       # named .env.stack to avoid colliding with the
                                       # repo's app-level .env handling
docker compose --env-file .env.stack build

# Run
docker compose --env-file .env.stack up -d

# Logs
docker compose logs -f trader
docker compose logs -f discord-listener

# Stop
docker compose down                    # state persists on /data/volume/meridian
```

`[REVIEW]` The compose stack `.env` lives at the repo root and would collide with Meridian's app `.env`. Spec uses `.env.stack` with `--env-file .env.stack` to disambiguate. Add `.env.stack` to `.gitignore`.

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
4. Copy plain `.env` from host repo into the volume: `sudo install -m 600 -o 1000 -g 1000 /path/to/repo/.env /data/volume/meridian/.env`. (Skip the encryption step for v1; see §9 to harden later.)
5. `docker compose --env-file .env.stack up -d`.
6. Watch `docker compose logs -f trader` for the first management cycle. Confirm `state.json` mtime updates → healthcheck stays green.

ELEVATE if any state file is held open by the PM2 process during copy. Prefer stopping PM2 fully first; never copy a live-written JSON.

---

## 13. Open Items / ELEVATE Before Implementing

1. ~~`envcrypt.js` CLI shape~~ **RESOLVED 2026-05-18.** Verified per-key inline model. v1 spec uses plain `.env`; encryption is opt-in hardening (see §9). Decision: ship v1 with plain `.env`, harden later.
2. Trader CPU limit `1.5` — measure actual usage in a non-prod run for ≥24h before locking. May need to bump to 2 if RPC concurrency saturates.
3. `NODE_OPTIONS=--max-old-space-size=1536` — assumes 2G hard / 1.5G soft. Re-tune if mem caps change.
4. ~~Healthcheck threshold~~ **RESOLVED 2026-05-18.** `docker/healthcheck.js` reads `managementIntervalMin` from `/data/user-config.json` and uses `max(2 × interval, 30) min`. Operator can override via `HEALTHCHECK_STALE_MIN` env in compose. No more false positives on long intervals.
5. `discord-signals.json` write semantics — confirm listener writes are atomic (rename-over) so trader doesn't read a half-written file. If non-atomic, add file locking or switch to a named pipe. ELEVATE if unclear.
6. Whether `discord-listener` should be allowed to crash without restarting `trader` — current compose has `depends_on` only for start ordering, not restart coupling. Confirm OK.
7. CI: building images on every commit vs operator-triggered. Out of this spec; add a `.github/workflows/build.yml` separately when CI is wanted.
8. Backup story for `/data/volume/meridian` — nightly tarball to remote? Out of this spec but ELEVATE: losing this volume = lobotomizing the agent.

## 13b. Future Improvement — Nano-Webhook Wiring

`[PLANNED, not in v1 scope]` Operator wants Meridian to publish events to Nano (the same channel used by the `nano-reminder` skill which polls a queue file every 15 min and turns entries into Telegram pings). Goal: push trade events (deploy, close, OOR, threshold-evolution, claim) out of the trader's local Telegram notifier and into Nano so they aggregate with other Nano-driven alerts under one operator UI.

Open questions to ELEVATE before designing:
- Direction: Meridian → Nano (push) only, or also Nano → Meridian (commands)? Today Telegram is bidirectional (commands like `/positions`, `/close`).
- Nano-webhook surface: HTTP POST endpoint? Queue file path? Telegram bot impersonation? Operator must specify.
- Auth: bearer token? Shared secret in `.env`? loreKeeper-issued agent key (`mcp__lorekeeper-local__create_agent_key`)?
- Failure mode: if Nano is down, does the trader queue events locally and replay, or drop?
- Coexistence with the existing Telegram notifier — replace or run in parallel? Migration story.
- Does this require an external HTTP endpoint on Meridian (for Nano → Meridian commands)? If yes, revisit Traefik decision from §1.

Implementation sketch when revived:
- Add `nano: { enabled, baseUrl, apiKey, channels: [...] }` block to config.
- `telegram.js` already centralizes outbound notifications; add a parallel `nano.js` notifier behind `if (config.nano?.enabled)`.
- Events to push: `position_opened`, `position_closed`, `oor_detected`, `fees_claimed`, `threshold_evolved`, `cooldown_triggered`.
- Best-effort fire-and-forget with a short retry queue in memory. Never block the trade path on Nano availability.

Park as separate spec (`docs/nano-webhook-integration.md`) when operator is ready to scope it. Trigger to start: operator provides the Nano endpoint contract.

---

## 14. Acceptance Checklist (run after first deploy)

- [ ] `docker compose ps` shows both `trader` and `discord-listener` `healthy`.
- [ ] `docker compose exec trader ls -l /data/state.json` shows recent mtime.
- [ ] `docker compose exec trader ls -l /app/.env` shows it is a symlink → `/data/.env`.
- [ ] `docker compose exec trader env | grep -E 'WALLET|RPC|OPENROUTER'` confirms env loaded (values masked in real check).
- [ ] (v1 only) `docker compose exec trader env | grep -E 'ENVRYPT_KEY|ENVCRYPT_KEY'` returns EMPTY — confirms no passphrase needed.
- [ ] `docker compose exec trader node -e "console.log(JSON.parse(require('fs').readFileSync('/data/user-config.json')).schedule?.managementIntervalMin ?? JSON.parse(require('fs').readFileSync('/data/user-config.json')).managementIntervalMin)"` returns a number — confirms healthcheck can read the interval.
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
