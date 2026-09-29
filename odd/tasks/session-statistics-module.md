# Session statistics module

## Objective / authorization

Turn the upstream ask filed as `Gentleman-Programming/gentle-shell#1545` (the shell's cost number excludes delegated subagent work) into a complete, self-contained statistics module: total session cost, cost per subagent, per-model statistics, and total/ per-agent/ per-tool time. The declared use case is an operator billing hours at an hourly rate, who needs the same panel to produce an hours report and its cost.

User authorized, in this order: resolve the upstream issue but scoped as a complete extension; split it into logical issues in the user's own fork `efirvida/gentle-shell`; produce the architecture for how the data is obtained and communicated to the orchestrator and the frontend, a catalog of which statistics are actually obtainable, and then the visual interface.

This turn's deliverable: the architecture, the obtainable-statistics catalog, the transport design, and the decomposed issue set created in the fork. The visual interface is a later deliverable, gated on the data layer landing first.

## Problem and constraints

The measured facts behind the upstream issue: the shell renders one session-level cost number, and it is the parent session's own spend (`sessionCost()` over `ctx.sessionManager.getEntries()`, `extensions/gentle-shell.ts:244-252`), while delegated cost accumulates per task (`lib/agents-protocol.ts:379-380`) and is never folded in. Each child is a separate `pi --mode rpc` process with its own `--session-dir` (`lib/agents-runner.ts:10,241`), so parent entries structurally cannot contain child usage. There is no cross-agent aggregate anywhere, and the desktop RPC projection does not carry cost or tokens at all.

Constraints that shape the design:

1. Attribution must be honest. A statistics panel used for a billing report cannot silently omit a class of work, and must not present a modeled or estimated number as a measured one.
2. Time is the harder half. Cost has a single source of truth per usage record; time must be defined (wall clock, active time, per-tool time) before it can be summed, and tool-level time may only be partially observable.
3. Data-availability limits are part of the deliverable, not a footnote. The catalog must state what is measurable, what is derivable, and what is not obtainable at all.
4. Telemetry policy is authoritative. `docs/telemetry.md` governs approved fields and source limitations.
5. The frontend contract must be versioned like the existing `gentle-agents.activity/v1` schema, and must not require the UI to re-derive aggregates the backend already knows.

## Route and delivery

Exploration delegated: two read-only scouts run in parallel (data availability; transport, persistence and telemetry policy), because the question spans both `lib/agents-*` and the extension/UI surfaces. The parent owns the architecture, the catalog, and the issue decomposition. No writer agent is used: the deliverable is design plus issue creation, not source implementation.

No SDD: not requested. ODD applies.

## Tasks

- [x] S1: Map every available cost, usage and time field with its source and persistence, and the explicit gaps.
- [x] S2: Map the transport and UI surface options, and the telemetry policy that constrains them.
- [x] S3: Design the module architecture: collection, aggregation, persistence, transport, and the UI contract.
- [x] S4: Produce the obtainable-statistics catalog, separating measured, derivable and unavailable.
- [x] S5: Decompose into dependency-ordered logical issues and create them in `efirvida/gentle-shell`.
- [x] S6: Harvest the upstream backlog for overlapping proposals and fold them in instead of duplicating.

## Delivered issue set (fork `efirvida/gentle-shell`)

Issues were disabled on the fork by GitHub's fork default, so `has_issues` was enabled and the four missing labels (`epic`, `type:feature`, `type:bug`, `status:needs-review`) were created before publishing.

| # | Issue |
| --- | --- |
| 1 | epic(shell): session statistics module — architecture, data catalog and decomposition |
| 2 | feat(statistics): canonical SessionUsageRecord and stop coercing unreported cost to zero |
| 3 | feat(statistics): transcript reader and replay for parent and child sessions |
| 4 | feat(statistics): pure aggregation engine for session, subagent, model and agent-class totals |
| 5 | feat(statistics): timeline with per-turn model latency and per-tool duration |
| 6 | feat(statistics): local opt-in JSONL statistics store with bounded retention |
| 7 | feat(statistics): versioned RPC transport and report exporters (markdown, CSV, JSON) |
| 8 | feat(shell): statistics overlay view and command |
| 9 | feat(statistics): billable hours report with hourly rate and export |

Verified after publication: every issue read back from the target host, its body passes a structural check for the expected sections, and all eight children reference epic #1. The epic carries `epic`, `type:feature` and `status:needs-review`; each child carries `type:feature`.

## Findings that changed the design

**The session transcripts are the real backbone, not the live event stream.** Verified directly against this machine's data:

- Parent sessions: `~/.pi/agent/sessions/<project>/<session>.jsonl`.
- Child sessions: `~/.pi/agent/gentle-agents/sessions/*.jsonl` (7 files on this machine at the time of writing).

Every line carries `type`, `version`, `id`, `timestamp`, `parentId`, and `message`. An assistant message carries `message.model` plus a full `message.usage`:

```json
{"input":499,"output":394,"cacheRead":11008,"cacheWrite":0,"reasoning":184,"totalTokens":11901,
 "cost":{"input":0.00007485,"output":0.0002364,"cacheRead":0.000033024,"cacheWrite":0,"total":0.000344274}}
```

So per assistant message, for parent AND child, the transcript already holds: model id, the six token counters including reasoning, and a cost breakdown split by input/output/cacheRead/cacheWrite. The parent's own `sessionCost()` reads only `cost.total` from exactly this shape, so parent and child records are the same shape and can share one normalizer.

**This corrects upstream issue #1263's caveat** ("the child transcripts carry no `usage`"), which was measured on Pi 0.85.1 / gentle-pi 3.3.0. On the current format the child transcript does carry usage and cost, which removes the main gap #1263 reported.

**Corollaries that change the module's shape:**

1. Post-hoc replay is the primary path. Cost, per-model split, cache split and reasoning share are readable from disk after the fact, for sessions that already ran, without any live capture. Live collection is needed only for in-session display and for a task that never finishes.
2. Per-tool time becomes derivable, not unavailable. `#1263` derived it by pairing message timestamps in the transcript: `assistant -> toolResult` is tool execution, `toolResult -> assistant` is model latency. Its documented caveat must be carried: with parallel tool calls the first result's gap absorbs the whole batch, and the harness-prepended `cd <cwd> &&` defeats naive command grouping.
3. Per-response latency is derivable the same way, which contradicts the runtime-metrics module's `missing` duration (that module states Pi hooks lack request correlation; the transcript is the way around it).
4. Live child usage is also thinner than the transcript: the live RPC `TASK_EVENT.USAGE` carries only scalar `tokens` and `cost` (`lib/agents-protocol.ts:252-262`), so anything per-model or per-turn for a child comes from the transcript or from `ChildResponseObservation`.

Still genuinely unavailable, and the module must say so rather than estimate: per-tool cost (usage is per assistant message, never per tool), retry attempts and their cost, and a distinction between a provider-reported zero and an unreported cost, because both ingestion sites coerce with `?? 0` / `|| 0` (`extensions/gentle-shell.ts:250`, `lib/agents-protocol.ts:258`).

## Upstream overlap folded in

The upstream backlog was harvested (913 issues, 636 PRs; 24 keyword queries). Relevant clusters:

| Upstream | State | Relationship to this module |
| --- | --- | --- |
| #1545 | open | This module's origin. Directly satisfied. |
| #1547 | open, `bug` + `status:approved` | Same root class: the usage surface omits providers used by active-profile subagents. Directly satisfied by subagent-aware aggregation. |
| #975 | open | Token breakdown (read/write/output) per subagent. Directly satisfied; the transcript supplies the split. |
| #1263 | open | Delegated child wall-clock, measured from transcripts. Directly satisfied as a report; its method is adopted and its caveats carried. |
| #636 | open | Cumulative input tokens across gentle-* agents; explicitly asks reporting to separate cumulative input from unique context and identify cached vs uncached. Reporting half satisfied; the behavioural half is not. |
| #731 / #729 | open / closed | 12.6M cache-read tokens; visibility half addressed, root cause not. |
| #1081 | open | Configurable status telemetry including tokens, cost, currency, subscription. A consumer of this module's aggregates, not a duplicate. |
| #1063 | open | The sidebar rail is a hardcoded section list, and this issue already proposes the extension point. A dependency only if the panel is a rail; an overlay view avoids it. |
| #1302 | open | Prior art to copy: a local, silent JSONL recorder (`$HOME/.gentle-ai/odd-adherence.jsonl`), opt-out via `DO_NOT_TRACK`/`CI`, never sent. The statistics store must follow the same shape. |
| #1434 / #1143 / #997 / #1315 / #1306 | closed | Formatting, metric retention, sidebar summary and elapsed-on-cards precedents the panel must not contradict. |
| #375 / #1244 / #36 / #571 | open | The startup HUD "stats panel" cluster is a different surface; must not be conflated with session statistics. |
| subscription cluster (#687, #800, #872, #1047, #1122, #1155, #1177, #1261, #1399, #1442) | open | Provider quota windows, already surfaced by `/gentle:usage`. Adjacent, explicitly out of scope, and the reason the statistics panel needs a distinct command name. |

Exploration note: the first delegated harvest failed and was reported as a fallback — `gentle-ai-explore` has no shell tool (only `read`/`grep`/`find`/`codegraph`), so it could not run `gh`. It correctly refused to fabricate findings. The parent then ran the harvest inline.

## Reference implementation studied (upstream-independent)

The user supplied `https://github.com/Cateds/opencode-stats` (`oc-stats`) as reference code. It was cloned to `/tmp/ref-opencode-stats` and read: 51 files, 35 of them Rust, ratatui TUI, reads OpenCode's local SQLite database or a JSON export. It is an independent community project for a different tool, so nothing is copied; what follows is what it validates, what it upgrades, and what does not apply.

### It validates decisions already in the issue set

- **One canonical record with model and agent attribution.** Its `UsageEvent` (`src/db/models.rs:32`) carries `session_id`, `parent_session_id`, `provider_id`, `model_id`, `agent`, `tokens`, created/completed timestamps and `stored_cost_usd: Option<Decimal>`. That is the same shape as `SessionUsageRecord` in I1, including the parent/child relation and per-agent attribution.
- **Provenance on money.** `PriceSummary { known, has_known, missing }` (`src/utils/pricing.rs:24`) is exactly the measured-versus-partial flagging proposed in I3. An independent implementation converging on the same model is good evidence for that decision.
- **Percentiles, not means.** `p50_output_tokens_per_second` computed from a `median(...)` over per-event rates (`src/analytics/model_stats.rs:24,95-121,176-202`). This matches #1263's explicit warning that means are poisoned by outliers.
- **Duration is optional, never zero.** `duration_ms()` returns `Option` and only when the delta is greater than zero (`src/db/models.rs:76-81`). Same rule as the I1 nullability work.
- **Dual source.** SQLite as the live store plus `--json` import, tagged by `DataSourceKind { Sqlite, Json }`. Same two-path idea as transcript replay plus a local store in I2 and I5.

### It upgrades the design in four concrete ways

1. **A pricing catalog IS obtainable, and this closes a gap recorded as permanent.** The catalog comes from `https://models.dev/api.json`, cached at the platform cache dir with a 60-minute TTL (`MODELS_DEV_URL`, `CACHE_TTL_SECS` at `src/cache/models_cache.rs:15-16`), combined with local pricing overrides read from the tool's own config (`load_pricing_overrides`), and published with an availability state `PricingAvailability { Cached, OverridesOnly, Empty }` plus `load_notice` and `refresh_failure_hint()` for offline degradation. Per-million arithmetic is `(input·p_in + output·p_out + cache_write·p_cw + cache_read·p_cr) / 1_000_000` using `Decimal` (`src/cache/models_cache.rs:121-128`). The epic currently says "a computed or unit price" is not obtainable because no catalog exists in this repository. That remains true of this repository, but the honest statement is that a catalog is available from an ecosystem source and would need to be added. This is a bounded new work item.
2. **Missing catalog fields have documented defaults.** `ModelPricing::with_fallbacks()` (`src/cache/models_cache.rs:33-43`): an absent `cache_write` price falls back to the `input` price, and an absent `cache_read` price to `input × 0.1`. Explicit, reviewable rules instead of silent zeros.
3. **Zero cost is a policy, not only a bug.** `ZeroCostBehavior { EstimateWhenZero, KeepZero }` (`src/utils/pricing.rs:7-20`): when a stored cost is zero but tokens are non-zero, the implementation either keeps the zero or falls back to the catalog estimate, chosen by a user-facing flag. I1 currently treats the coercion as a plain defect. The reference is more correct: a genuine zero is legitimate for local or self-hosted models, so the user must be able to choose. I1 should carry this policy rather than hard-code one behaviour.
4. **Money needs decimal arithmetic.** The reference uses `rust_decimal::Decimal` throughout, never a float. A billing report summing thousands of per-message costs accumulates float error in JavaScript. The issue set does not yet say this.

Two smaller adoptions: a documented inclusion rule for rate statistics (`is_rate_eligible()` requires `output >= 100`, excludes `finish_reason == "tool-calls"`, and requires a known duration, `src/db/models.rs:83-88`), which prevents tool-call turns from poisoning latency numbers; and the README's consistency invariant, that its calculation "aligns with `opencode stats`", which is worth mirroring as a rule that this module's numbers must match what gentle-shell's own surfaces report.

### Export precedent

A shareable image card with an automatic fallback to a text summary, built on `ab_glyph`/`image`/`imageproc`/`tiny-skia` with the clipboard via `arboard`, plus a versioned JSON import schema (`JsonMessageRecord` and friends, `src/db/models.rs:250-298`). Relevant to I6: a report that a human can paste somewhere, and a versioned JSON shape rather than an ad-hoc dump.

### What does not apply

- Rust, ratatui and SQLite specifics. This module is an in-process extension, since it needs the live session and the overlay surface; `oc-stats` is a standalone binary pointed at another tool's database.
- It has no per-tool duration, no idle classification and no activity segments: its duration is per message/response. I4's timeline work stays additive beyond the reference.
- Its `TokenUsage` has four counters (`input`, `output`, `cache_read`, `cache_write`); gentle-shell has six including `reasoning`, so ours is a superset.
- Its overview KPIs (total tokens, cost, sessions, messages, prompts, models used, active days, a "fun comparison" line) are a useful checklist but its surface layout is not transplantable.

### Proposed issue amendments — deferred by explicit user decision (2026-09-29)

The user chose to keep the issue set unchanged for now and fold these in when each piece is implemented. They are recorded here so they are not lost:

| Issue | Amendment |
| --- | --- |
| new I10 | Model pricing catalog: `models.dev/api.json` plus local overrides, TTL-cached, with availability states and the documented fallback rules. This is the item that closes the pricing gap the epic currently records as unobtainable. |
| I1 (#2) | Add the zero-cost **policy** rather than only fixing the coercion, and require decimal-safe money arithmetic. |
| I3 (#4) | Adopt the `PriceSummary`-style provenance shape, and add the alignment invariant: these numbers must match what gentle-shell's own surfaces report. |
| I4 (#5) | Adopt a rate-eligibility rule so tool-call turns cannot poison latency statistics. |
| I6 (#7) | Add a shareable export card with a text fallback, and a versioned JSON export schema. |

For reference, the epic's data catalog currently states that a computed or unit price is not obtainable because no pricing catalog exists in this repository. That remains true of this repository, and the correction to record at implementation time is that an ecosystem source exists and adding it is bounded work.

## T1 (issue I1 / #2) detail and the re-slice, decided 2026-09-29

### Decisions taken

**Money representation: integer nano-USD.** Decided by the parent after the user's answer addressed display rather than representation; the user did not object. Rationale: the provider costs observed in real transcripts carry nine decimal places (`0.000344274`), so `cost × 10^9` is exact; integer addition is exact; no new dependency. `float64` represents integers exactly up to 2^53, so the accumulated ceiling is roughly nine million USD per total, which is ample for a session and worth documenting. Rejected: BigInt with an explicit scale (exact but more code and awkward JSON), and float with disciplined rounding (not exact, and not defensible in a billing report).

**Backward compatibility of the persisted `cost` field: deferred to I5.** The user chose this. It shortens I1's review surface, at the cost that the old on-disk shape (`cost: 0.42` in `gentle-agents/tasks/*.json`) and the new one coexist until I5 lands.

**The zero-cost policy does not belong in I1.** Refinement of an earlier note: `EstimateWhenZero` requires the pricing catalog, which is the deferred I10. I1 therefore only preserves the *distinction* between a reported zero and an absent cost, and the policy flag lands with the catalog.

**I1 changes no pixels.** `sessionCost()` returns a provenance-carrying total, the render path reads the known value through a single conversion point, and the rendered output stays byte-identical, so the existing fixtures pass untouched. The `complete` flag becomes useful in #10 and I3.

### The transport finding that forced the re-slice

The bar cannot see delegated cost, and not because it omits it: the two halves live in different extensions with no shared state. `extensions/gentle-shell.ts` renders the bar and calls `buildShellBarModel` at `:253`, `:298` and `:1638`, while the `TaskStore` is owned by `extensions/gentle-agents.ts:312`, and `gentle-shell.ts` imports nothing from `lib/agents-protocol.ts`.

The sanctioned channel already exists: a versioned event on Pi's extension bus. `lib/runtime-metrics-children.ts:6` defines `CHILD_METRICS_EVENT = "gentle:runtime-metrics:child/v1"`, `extensions/gentle-agents.ts:44` emits it, and `extensions/runtime-metrics.ts:46` subscribes with `pi.events.on(...)`. The alternative pattern is terminal-owned symbol state (`Symbol.for("gentle-pi.experimental-sidebar.state")`), used by the sidebar.

### Display decision

One figure in the bar: the session total, with a trailing `+` whenever the total is partial, driven by I1's `complete` flag. The orchestrator/subagent split belongs in the overlay panel, not in a glance surface that already degrades by giving up the path on narrow terminals. The marker must be opt-in per call site, because `formatCost` (`lib/shell-bar.ts:113`) is shared by the bar, the agents card and the agents view. The existing subscription suffix and the `usageCost` visibility gate are preserved.

### Re-slice

I1 is a pure internal change, so delivering the originating request (#1545) only at I7 would have put the visible fix seventh in line. Issue **#10** was therefore created and inserted directly after I1:

1. I1 (#2) — foundation: types, no coercion. No visible change.
2. **#10** — session cost total (orchestrator + subagents) in the bar and header, over the versioned cross-extension event. Delivers #1545 and satisfies #1547.
3. I2..#9 — the rest of the module.

The epic's child table was updated with the new row plus a dated revision note. The diff was verified to contain only those two additions.

### I1 edit surface, verified

| File | Change |
| --- | --- |
| `lib/session-cost.ts` (new) | `ReportedCost`, `CostTotal`, `addCost()`, the scale constant, the rounding rule, and the single `nanoUsdToUsd()` conversion point |
| `extensions/gentle-shell.ts:249` | Drop `?? 0`; `sessionCost()` returns `CostTotal` |
| `lib/agents-protocol.ts:71` | `UsageEvent.cost` becomes `ReportedCost` |
| `lib/agents-protocol.ts:258` | Drop `\|\| 0`; parse the provider decimal to nano-USD, preserving absence |
| `lib/agents-protocol.ts:137-138,380` | `TaskRecord.cost` becomes `CostTotal`; accumulate with `addCost()` |
| `lib/agents-runner.ts:341` | `cost: 0` becomes the empty total |
| `lib/agents-widget.ts:184,188,196,204,213` | Render through the conversion point; blank when absent |
| `lib/agents-view.ts:146` | Same |
| 6 test files | Fixtures plus the three-state cases |

Five existing source files plus one new, and six test files (agents-protocol, agents-view, agents-view-thread-identity, agents-widget, gentle-shell, shell-bar). Known risk, not yet counted: every `cost:` literal in those tests breaks the typecheck, which is the bulkiest but most mechanical part of the change. Because the type is shared, the full suite must run before closing, not just the focused files.

## Session coherence pass, 2026-09-29

The user resolves one issue per dedicated session and asked for the set to be self-sufficient under a prompt as short as `Solve issue I1`. Three gaps were verified and closed:

1. **No title contained an ID**, and `gh` search for `I1` returned #1, #6, #10 and #3 — never I1 itself (#2). Every child was retitled with the ID as a prefix, e.g. `I1 · feat(statistics): canonical SessionUsageRecord ...`.
2. **The architecture lived only in this untracked file**, which a linked worktree does not see. The reachable source of truth is now the epic #1 on GitHub: verified facts, settled decisions, working rules, and the deferred amendments.
3. **No operational brief.** Every child now carries a `Session brief` at the top: ID, dependency, read-first pointer, worktree rule, exact verification commands with the 188-diagnostic baseline, the constraint that tests are the only verification, and the commit authority.

IDs were realigned to **execution order**, so ID order is execution order and the epic's table is the single authoritative index. The remap was done in two passes with placeholder tokens to avoid cascading, then verified: every child's brief matches the epic table, all nine carry a brief, and no ID reference anywhere is out of range.

| ID | Issue | Scope | Depends on |
| --- | --- | --- | --- |
| I1 | #2 | Canonical `SessionUsageRecord`; remove the unreported-to-zero coercion | the epic |
| I2 | #10 | Session cost total in the bar and header — the visible fix | I1 |
| I3 | #3 | Transcript reader and replay | I1 |
| I4 | #4 | Pure aggregation engine | I3 |
| I5 | #5 | Timeline: per-turn latency and per-tool duration | I3, I4 |
| I6 | #6 | Local opt-in JSONL store | I1 |
| I7 | #7 | Versioned RPC transport and exporters | I4, I5 |
| I8 | #8 | Statistics overlay view and command | I4, I5, I7 |
| I9 | #9 | Billable hours report | I7, I8 |

Cross-references inside the child bodies were remapped as well and spot-checked (for example #10's pointer to the overlay panel now reads I8, and the `complete` flag pointer reads I1).

Authority granted to dedicated sessions by explicit user decision: commit work units on their own branch; push, pull request and merge remain the human's decisions.

## Non-blocking requirement, added 2026-09-29

The user raised a hard requirement: session or general telemetry must never block work, and must never slow tasks down. It was turned into an invariant rather than prose.

**A live instance of the defect already exists in the code path this module touches.** `sessionCost()` iterates every session entry, and it runs on **every render**: `extensions/gentle-shell.ts:297` renders by calling `buildShellBarModel`, which calls `sessionCost()`, and there is no cache. That is O(session entries) per frame today. The module must not add to it, and issue I2 is where bounding or caching that scan belongs.

**The repository has already paid for this class of defect**, which is why the requirement is not hypothetical:

| Evidence | What it shows |
| --- | --- |
| #1115 (open) | `quiet-tools` SHA-256s a ~26 MB binary on every bash render call |
| #1142 (closed) | the working pulse repainted the whole view 12.5x/s, each repaint O(transcript) |
| #1254 (closed) | per-render re-measurement made fullscreen about 11x the render cost of vanilla pi |
| #1011 (closed) | the 1 Hz presence poll re-rendered the full transcript |
| #1262 (open) | re-reading unchanged remote activity files on every presence poll |
| #994 (open) | the render pulse drives high CPU in long regular-mode sessions |

**The existing telemetry already satisfies the requirement by contract**, so the risk is the new module, not the old one. `docs/telemetry.md` allows one asynchronous attempt per event, discards when busy, and forbids an outbox, retry, backoff, cooldown, daemon or session reconstruction. `extensions/runtime-metrics.ts` implements that with an attempt slot (`owner.attempt.offer(async signal => ...)`) and swallows errors, with the comment "No telemetry error enters the shared event bus."

**The eight rules**, recorded in the epic section `Non-blocking requirement (hard)`:

1. No filesystem or network I/O in a render path; the bar and header read a cached value that events update.
2. No synchronous I/O in any render, tool-call or message hook.
3. Fire-and-forget with a single attempt slot, per the existing telemetry contract.
4. Bounded work per event: streamed or chunked transcript reads, never a full synchronous load.
5. Shutdown is never delayed by a report or store write.
6. Network is optional and off the critical path; the deferred pricing catalog must be background with a cached or offline fallback.
7. Failures are silent to the session and never enter the shared event bus.
8. Asserted is not proven: every issue carries a Non-blocking acceptance criterion, and an issue may not close with it asserted only in prose.

**Placement**: the epic gained invariant 8 plus the full section, and all nine children gained a tailored Non-blocking acceptance criterion. Verified after the patch: exactly one bullet per issue, each inside its Acceptance criteria section, and no duplicate headings in the epic.

Authoring note: the patching script failed on two of its own bugs (a stray identifier and a write-order problem) and reported errors while still producing the correct files. The result was confirmed by reading every artifact back rather than by trusting the script's exit status.

## Scope boundary: the quota path is not an input, 2026-09-29

The user asked that nothing outside the statistics module be touched, and questioned whether the subscription quota refresh is used for statistics.

**It is not, and it must not be.** Verified: `lib/shell-usage.ts` models `UsageWindow { usedPercent, used?, budget?, windowSeconds, resetAt }` over provider quota endpoints (Codex, Anthropic, NaN). That measures a provider's rolling window allowance, not what a session spent, so the two numbers are not interchangeable.

The nuance worth recording: NaN's quota payload does carry token fields (`windowTokens`, `fullWindowTokens`, `tokensUsed`, `windowTokensUsed`), but they describe the model's allowance for a billing period. Folding them into session statistics would present plan consumption as session consumption. That is a correctness decision, not just a scope one.

No issue in the set pulled the quota path into scope; the existing mentions were correct (the epic listed it as measured-but-out-of-scope, and I8 and I2 name `/gentle:usage` only to keep the new command distinct).

Action taken so a dedicated session cannot wander there:

- The epic's `Out of scope` now names the off-limits modules explicitly: `lib/shell-usage.ts`, the `USAGE_REFRESH_MS` poll and `refreshUsage` in `extensions/gentle-shell.ts`, and the `/gentle:usage` panel.
- Non-blocking rule 6 was hardened so it cannot be read as covering the quota refresh: the only network access this module may add is the deferred pricing catalog, and the quota refresh is explicitly not part of the module.

Verified after the patch: 14 headings with no duplicates, rule 6 hardened once, and all nine epic sections still present exactly once.

## Plugin-boundary seam: keep I1/I2 thin, 2026-09-29

The epic gained a plugin-boundary section: a separate package can carry I3–I9, but I1 is a producer-side edit and I2 needs gentle-shell's single-owner footer, so the honest and visible halves stay in the core. The user asked for the architecture to be as non-intrusive to the agent code as possible, exposing only the functions I1 and I2 need, so a future split does not have to untangle statistics logic from the agent.

Decision and shape:

- **One pure module owns the contract.** `lib/session-usage.ts` holds the canonical `SessionUsageRecord`, `UsageCost`, `UsageTokens`, `SessionCostTotal`, `TimeSegment` and `TurnRecord`, the nano-USD primitive, and the aggregation folds. It imports nothing from `lib/agents-*` or any extension, so it has no agent coupling to remove later.
- **The agent code only adapts.** Two thin call sites remain: `extensions/gentle-shell.ts` `sessionCost(ctx)` delegates to `sessionCostFromEntries(ctx.sessionManager.getEntries())`, and `lib/agents-protocol.ts` uses `childUsageCost(usage)` plus `accumulateTaskCost(...)` in `recordPatch`. Neither site contains statistics math.
- **The exposed ingestion seam.** The functions I1 and I2 need are exported from the pure module over plain shapes: `sessionCostFromEntries` (the parent, I1), `childUsageCost` (a child assistant message, I1), `accumulateTaskCost` (the `TaskRecord` delta rule, I1), and `delegatedCostFromTasks` (the subagent total, the input I2 will publish over the delegated-cost topic).
- **Why this helps the split.** The future plugin reads the same inputs through public Pi surfaces (`ctx.sessionManager.getEntries()` for the parent, the delegated-cost topic for children) and applies the same exported rules; the core keeps only the producer fix and, in I2, the single merged bar figure. No statistics module internals leak into `lib/agents-*`.
- **Naming note.** The epic's I1 row predicted `lib/session-cost.ts`; the landed module is `lib/session-usage.ts` because it owns the whole `SessionUsageRecord` contract, not only cost. The epic row should be corrected when convenient.

## Acceptance and checks

- Every claimed data source is cited as `file:line` and was read, not assumed.
- Every "not obtainable" claim names the reason (no source, not persisted, lost on restart, policy-restricted).
- The issue set is created in the fork and each issue states its dependency, its acceptance criteria, and what it must not do.
- Honest reporting: anything not verified is marked as not verified.

## Progress

2026-09-29: Feature opened. Upstream reference issue is #1545 in the parent repository. Deliverable scoped to architecture plus issue decomposition; implementation is not started in this turn.

2026-09-29 (S1-S4): Data inventory and transport mapping completed via two read-only scouts plus parent verification. The decisive correction is that the session transcripts, not the live event stream, are the backbone, and that the child transcript does carry model and the full usage and cost breakdown per assistant message. Full findings, the upstream integration map, and the corrections are recorded above.

2026-09-29 (I1, issue #2): Implementation started and completed on branch `feat/session-usage-record` in worktree `~/Desktop/dev/gentle-shell-i1`. Added `lib/session-usage.ts` (canonical record, nano-USD money, exposed ingestion seam), made both ingestion sites preserve absence (`extensions/gentle-shell.ts` `sessionCost` and `lib/agents-protocol.ts` `normalizeRpcEvent`/`recordPatch`), and added the `costComplete` provenance to `TaskRecord`. Tests: `tests/session-usage.test.ts` (new), `tests/agents-protocol.test.ts` (updated contract, reported/zero/absent), `tests/gentle-shell.test.ts` (parent provenance). Verified: focused 103 + 225 tests green; `check:runtime-modules` green; typecheck 188 diagnostics, no regressions. Full suite at default parallelism flaked on four unrelated timing files that pass in isolation; serialized run green (see the session report for exact numbers). Committed and pushed on `feat/session-usage-record`; PR #11 (base `main`, fork `efirvida/gentle-shell`).

2026-09-29 (I2, issue #10): Implementation on branch `feat/delegated-session-cost` (stacked on I1), worktree `~/Desktop/dev/gentle-shell-i2`. The agents extension publishes the delegated total on the versioned `gentle:session-cost:delegated/v1` topic (derived from `store.list`, so it survives the card's finished-row TTL); `gentle-shell` subscribes, caches both the orchestrator total (refreshed on `turn_end`/`agent_end`/`session_tree`, never scanned during render) and the delegated total, folds them with `mergeSessionCost`, and paints the opt-in `+` partial marker in the bar and header only. New `lib/session-delegated-cost.ts` codec; `RunnerHooks.onUsage`; `SessionCostTotal` provenance drives the marker. Verified: focused 336 + agents-runner 74 + gentle-agents 126 green; typecheck 188 diagnostics, no regressions; `check:runtime-modules` green. Not yet committed at the time of this entry.

2026-09-29 (I1 delivery): the fork's `main` advanced to upstream `08de420c`; PR #11 conflicted only on the import block of `tests/gentle-shell.test.ts`. Merged `origin/main` into `feat/session-usage-record`, combined both import lines, focused tests 300/300, typecheck 187 no regressions, independent verification passed, and pushed the merge commit. PR #11 back to MERGEABLE.

2026-09-29 (I2 bug found on real resume): the live bar showed exactly the orchestrator-only total ($0.264 when parent-only was $0.264 and parent+subagents was $0.284). Root cause: on a resumed session the delegated total is published while the TaskStore restores, before the shell's session is ready, and it was never re-published, so the restored subagents' cost was lost. Fix: `gentle-agents` re-publishes the delegated total on a scheduled task after `session_start` (all handlers have run), and `gentle-shell` gates the subscriber on the live context and stashes a payload that arrives before `session_start`. Verified live via temporary debug logs: publish tasks:2 nanoUsd:19644498 -> shell apply. Focused tests 299 + gentle-agents 126 green; typecheck 187 no regressions.

## Next step

Commit the I1 work units on `feat/session-usage-record`, then implement I2 (issue #10), which consumes `delegatedCostFromTasks` and publishes the delegated-cost topic. Deliberately not done: pushing, opening a PR, or commenting on upstream #1545.
