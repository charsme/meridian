# loreKeeper Integration — Deferred TODO

**Status:** parked, not started.
**Owner:** Charis.
**Last reviewed:** 2026-05-18 on branch `prod` @ 9bf0853.

## Summary

Augment Meridian's runtime knowledge accumulation with the loreKeeper MCP store as a **semantic recall layer**. Keep the existing numeric feedback loops local — do not migrate them.

## Context

Meridian today accumulates trading experience via four autonomous closed-loop mechanisms:

1. `lessons.js:311-456` `evolveThresholds()` — perf data → screening config mutation, every 5 closes.
2. `signal-weights.js:94-199` `recalculateWeights()` — Darwinian signal lift, every 5 closes.
3. `pool-memory.js:101-216` `recordPoolDeploy()` — per-pool win rates + cooldowns, every close.
4. `hivemind.js:191-346` — collective swarm push/pull every 15 min.

All persistence is local JSON. Retrieval into the LLM prompt (`prompt.js:14-176`) is purely structural — filter by role tag, recency, pinned flag. No semantic search. Old lessons age out of the recency window even when situationally relevant. HiveMind incoming has no dedup or conflict detection.

loreKeeper (MCP server at `mcp.lore.ai.chars.me`, namespace `private/trading`) provides:

- Semantic search via embeddings (Qdrant + LiteLLM) — `search_knowledge`, `suggest_topic`.
- Content-hash dedup at save time — `dedupHit` flag.
- Promotion ladder: `working → episodic → canonical`.
- Conflict detection — `list_conflicts`, `review_conflict`.
- Policy gating — `create_policy`, `approve_policy`.

Two memories already saved under `private/trading` documenting the current architecture and learning loops:
- Topic `meridian-architecture` — id `35a3d7d4-3a7c-4711-b64e-ae7058b795b7`.
- Topic `meridian-learning-loops` — id `7dd63896-577d-4e6f-9db6-b33355b220ec`.

## Scope

### IN scope — augment with loreKeeper

| Surface | Why |
|---|---|
| English text-lesson layer of `lessons.json` | Semantic recall across full history, not just recency window. |
| HiveMind shared lesson intake | Dedup via content hash + conflict surfacing prevents swarm noise. |
| Closed-position post-mortems | Rich text, low write rate — ideal semantic search target. |

### OUT of scope — keep local

| Surface | Why |
|---|---|
| `evolveThresholds()` numeric math | Stats over typed columns. Markdown round-trip loses queryability. |
| `signal-weights.js` numeric multipliers | Same reason. Performance-critical math. |
| `pool-memory.json` per-pool stats | Numeric, accessed every cycle. Latency-sensitive. |
| `state.json` live position registry | Operational state, not knowledge. Wrong abstraction. |
| `token-blacklist`, `deployer-blacklist`, `dev-blocklist` | Need microsecond `Set.has(mint)` per candidate. HTTP would add ~50–500ms × ~50 candidates per screen. |
| 10s PnL poll prompt builds | Latency budget tight. Adding HTTP per tick risks missing OOR thresholds. |

## Design

### Save path

In `lessons.js:96-155` `recordPerformance()`, after the local `lessons.json` write and after `deriveLessonFromPerf()`:

```js
// pseudo
if (config.lorekeeper?.enabled && derivedLesson) {
  await safelyMirrorToLoreKeeper({
    workspace: "private",
    domain: "trading",
    topic: lessonTopic(perf),         // see Topic strategy below
    memory_type: "working",            // promoted later via reinforcement
    content: derivedLesson.text + "\n\nContext: " + JSON.stringify(perf.signal_snapshot)
  }).catch(err => logger.warn("lorekeeper mirror failed", err));
}
```

- Mirror is fire-and-forget. Local `lessons.json` write remains the source of truth.
- Failure never blocks the close path.

### Topic strategy

Per the `lorekeeper-topic-selection` skill (`~/.claude/skills/lorekeeper-topic-selection`):
- One topic per **stable concern**, not per lesson.
- Candidates: `meridian-close-postmortems`, `meridian-screening-rules`, `meridian-pool-patterns`, `meridian-launchpad-signals`.
- Reuse-first via `list_topics` cached at process start; mint new only when no semantic match.

### Retrieval path

In `lessons.js:623-689` `getLessonsForPrompt()`, replace the structural-only filter with a hybrid:

```js
// pseudo
const local = filterLocalLessons({ recency, role, pinned });

let semantic = [];
if (config.lorekeeper?.enabled && currentPoolContext) {
  semantic = await raceWithTimeout(
    searchKnowledge({
      workspace: "private",
      domain: "trading",
      topic: pickTopicForContext(currentPoolContext),
      query: serializeContextForEmbedding(currentPoolContext)
    }),
    500  // ms
  ).catch(() => []);
}

return mergeAndRank(local, semantic, { semanticBudget: 3 });
```

- Hard 500ms timeout. Local fallback always available.
- Semantic budget ≤ 3 lessons per prompt — prevents prompt bloat.
- `serializeContextForEmbedding(ctx)` is a short string like `"launchpad=pumpfun vol=4.2 organic=55 mcap=180k binStep=80"` so the embedding is dense and on-point.

### Promotion policy

Working → episodic when a lesson's `pool_address` or `signal_pattern` recurs in ≥ 2 perf records with consistent outcome direction. Implement as a background pass inside `evolveThresholds()` since it already iterates perf history every 5 closes.

Canonical only via manual approval through `approve_policy` or explicit operator flag. Rationale: canonical lessons get pinned into every prompt — wrong canonicalization poisons the agent for many cycles.

### Conflict handling

After save, periodically (every 10 closes) call `list_conflicts` for `private/trading`. If conflicts found, surface in Telegram briefing (`briefing.js`) for operator review. Do not auto-resolve.

### Config flag

Add to `config.js` and `user-config.example.json`:

```js
lorekeeper: {
  enabled: false,             // off by default; opt-in
  baseUrl: "https://mcp.lore.ai.chars.me",
  apiKey: process.env.LOREKEEPER_API_KEY,
  workspace: "private",
  domain: "trading",
  semanticTimeoutMs: 500,
  semanticBudget: 3,
  mirrorOnSave: true,
  conflictCheckEveryNCloses: 10,
}
```

Add to `update_config` whitelist in `tools/executor.js` `CONFIG_MAP`.

### Failure modes

| Failure | Behavior |
|---|---|
| loreKeeper unreachable on save | Log warn. Local `lessons.json` write succeeds. No retry queue. |
| loreKeeper slow on retrieve | 500ms timeout → use local-only lessons. |
| Embedding API throws | Same — local-only. |
| Wrong topic minted | `lorekeeper-topic-selection` skill mandates reuse-first; misuse caught at code-review. |
| Privacy leak (wallet addrs, PnL) | Wallet address NOT included in lesson text — only signal patterns + outcome direction. PnL bucketed (e.g. `>20%`, `0–10%`, `loss`). |

## Prereqs before starting

1. **Volume of data.** Meaningful lift starts around 100+ closed positions. Today's lesson volume insufficient — premature integration adds infra without payoff.
2. **HiveMind agent count.** Dedup is most valuable when ≥ 3 agents are pushing. If solo, defer.
3. **Operator trust model.** Confirm trading strategy fingerprints on `mcp.lore.ai.chars.me` are acceptable. The server is self-hosted at `diricare.com` (see existing `mcp__lorekeeper-local__*` usage), so likely fine for Charis but document explicitly.

## Expected lift

- ~5–15% screening hit rate improvement on long-horizon agents (>200 closes), best estimate.
- Larger on multi-agent HiveMind (dedup eliminates lesson-flood pollution).
- Negligible on fresh agents (<50 closes).
- Zero impact on numeric loops by design.

## Anti-goals

- Do NOT replace `lessons.json`, `pool-memory.json`, `signal-weights.json` with loreKeeper. Numeric state stays local.
- Do NOT make any path block on loreKeeper. Every call is best-effort with local fallback.
- Do NOT canonicalize lessons autonomously without operator approval.

## Trigger condition to start

Either:
- 100 closed positions accumulated AND HiveMind sync active with ≥ 2 other agents pushing; or
- Operator explicitly asks despite low data volume.

## Implementation order when revived

1. Add config flag + `update_config` plumbing.
2. Write `lessons.js` mirror-on-save shim. Test in DRY_RUN.
3. Add hybrid retrieval to `getLessonsForPrompt()`.
4. Add promotion pass inside `evolveThresholds()`.
5. Add conflict-check + Telegram surfacing.
6. Document operator override flow for canonical promotion.

## Files this touches when implemented

- `config.js`, `user-config.example.json`
- `lessons.js`
- `tools/executor.js` (`update_config` whitelist)
- `briefing.js` (conflict surfacing)
- `package.json` if a loreKeeper HTTP client lib is added (or use `fetch` directly — preferred, zero deps)

## Related memories in loreKeeper

- `private/trading/meridian-architecture` — id `35a3d7d4-3a7c-4711-b64e-ae7058b795b7`.
- `private/trading/meridian-learning-loops` — id `7dd63896-577d-4e6f-9db6-b33355b220ec`.
