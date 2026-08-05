# Upstream notes — ACP native multi-model + effort UX

This file is duplicated verbatim in both forks this work spans —
`ryanskidmore/OpenHands` (Agent Canvas, this repo) and
`ryanskidmore/software-agent-sdk` — so a reviewer reading either repo's root
sees the whole picture. It is a tight summary for when
`feat/acp-native-models` gets PR'd upstream to `OpenHands/agent-canvas` and
`OpenHands/software-agent-sdk`; the day-to-day working log lives in
`PROGRESS.md` and `.claude/plans/acp-native-models.md` (this repo only).

All work happened on short-lived `acp-models/mN-*` (Canvas) / `acp-models/bN-*`
(SDK) branches, PR'd with base `feat/acp-native-models` on each
`ryanskidmore` fork — never against `main`, never self-merged.

## PR cross-reference

| Repo | # | Title |
|---|---|---|
| Canvas (`ryanskidmore/OpenHands`) | #1 | M0: ACP native-models plan, spike findings, progress tracker |
| Canvas | #2 | M1: models.dev catalog service |
| Canvas | #3 | M2: dynamic model choices + remembered custom models in the ACP profile editor |
| Canvas | #4 | M3: live session models + dynamic choices in the chat model pill |
| Canvas | #5 | M4: effort selection in the ACP profile editor |
| Canvas | #6 | M5: mid-session effort switching in the chat model pill |
| Canvas | #7 *(pending)* | M6: hardening — mock ACP server `configOptions` + e2e (M6a); docs, local-SDK-fork dev wiring, this file (M6b) |
| SDK (`ryanskidmore/software-agent-sdk`) | #1 | B1: split Claude Code model/effort composite ids into config options |
| SDK | #2 | B2: surface ACP session effort state on `ConversationInfo` |
| SDK | #3 | B3: grouped select options + live `config_option_update` handling |

Canvas #2–#6 stack on each other in order (#6→#5→#4→#3→#2→#1); SDK #1–#3
stack similarly (#3→#2→#1). Canvas #4–#6 pair with SDK #1–#3 for live data
but do not hard-depend on them — every Canvas feature below degrades
gracefully against a stock (un-forked) agent-server; see "Compatibility"
in `docs/ACP_AGENTS.md`.

## What changed — Canvas (`ryanskidmore/OpenHands`)

- **Dynamic model choices** (`src/hooks/use-acp-model-choices.ts`,
  `src/api/models-dev-catalog.ts`): the ACP model picker — both the
  Settings → Agent profile editor and the in-chat model pill — now merges
  four sources in precedence order: live session models > the provider's
  curated registry > per-profile remembered custom model ids > the
  [models.dev](https://models.dev) catalog as a fallback. Previously the
  list was the static curated registry only, and a **Custom** ACP server got
  no picker at all (empty curated list, nothing else to show).
- **Live models in the chat pill** (`src/hooks/use-chat-input-model-state.ts`,
  `src/hooks/use-acp-model-choices.ts`): `ConversationInfo.available_models`
  — already forwarded by a stock agent-server, just never read by Canvas —
  is parsed into `AppConversation.acp_live_models` and merged as the
  highest-precedence source. This is what makes a Custom server's picker
  non-empty once a session has actually reported models.
- **Reasoning-effort UI**: a composite `"<base>/<effort>"` id convention
  (`src/utils/acp-model-id.ts`: `parseAcpModelId` / `composeAcpModelId` /
  `getAcpEffortLevels`) for Claude Code and Codex, an Effort dropdown in the
  profile editor, and a mid-session effort switcher in the chat pill (same
  `useSwitchAcpModel` mutation a model pick already uses). `"default"` composes
  to no suffix. Conversation chips render `"<label> · <effort>"`
  (`labelForAcpModel` in `src/constants/acp-providers.ts`).
- **models.dev catalog service** (`src/api/models-dev-catalog.ts`,
  `src/hooks/query/use-models-dev-catalog.ts`): fetches, trims, and caches
  (`localStorage`, 24h TTL, ETag revalidation) `https://models.dev/api.json`
  — unauthenticated, never throws, degrades to the curated list on any
  failure.
- **Per-profile custom-model store** (`src/stores/acp-custom-models-store.ts`):
  a zustand-persisted `Record<profileId, string[]>` so a custom model id
  typed once is offered back as a suggestion, instead of being a one-shot
  free-text override.

### Why upstream-worthy

- Fixes a real gap, not a cosmetic one: a stock agent-server has forwarded
  `available_models` since before this work started — Canvas simply never
  read it, so every ACP conversation showed a static, occasionally stale
  model list, and Custom servers were unusable without knowing the exact
  model id in advance.
- Effort selection unlocks a capability (`claude-agent-acp`'s `effort`
  config option, `codex-acp`'s `reasoning_effort`) that was previously only
  reachable by hand-typing a composite id into the free-text model
  override and hoping it round-tripped correctly.
- Every addition is purely additive to Canvas's own data model (no new
  required fields, no breaking `agent_settings_diff` shape change) and
  degrades gracefully field-by-field against older/stock agent-servers —
  see `docs/ACP_AGENTS.md` → "Compatibility with older agent-servers".
- The models.dev catalog costs nothing extra in the common case (24h cache,
  gated to ACP contexts only) and turns "no curated list" into "a real,
  ranked list" for any Custom ACP server pointed at a provider models.dev
  already indexes.

### Known scope cuts (Canvas)

- **`pruneMissingProfiles`** (`src/stores/acp-custom-models-store.ts`) exists
  and is unit-tested but is **not wired to any call site** — a deleted
  profile's remembered custom model ids are never garbage-collected from
  `localStorage`. Documented as a deliberate M2 follow-up, still open as of
  M6.
- Custom ACP servers get **no models.dev catalog extras** — `custom` isn't
  in `MODELS_DEV_PROVIDER_BY_ACP_SERVER`, and no call site passes the
  catalog's `opts.providerKey` override for it. A Custom server's list is
  live + remembered-custom only; wiring a manual provider-key picker for
  Custom was scoped out.
- Effort demotion after a **live, out-of-band model change** (a
  `config_option_update` push the SDK's B3 receives with no matching
  `set_acp_model` call) can surface a composite id split back to its bare
  base in the pill — see the SDK-side "composite demotion" scope cut below;
  Canvas has no independent mitigation for this, since it only ever renders
  whatever `current_model_id`/`current_effort` the server reports.

## What changed — SDK (`ryanskidmore/software-agent-sdk`)

All three land in `openhands-sdk/openhands/sdk/agent/acp_agent.py` (the ACP
client) plus additive fields on `ConversationInfo`
(`openhands-agent-server`).

- **B1 — Claude effort splitter**: extends the existing Codex
  `model/effort` composite-id convention (`_codex_model_config_options`) to
  claude-code (new `_claude_model_config_options`, `_CLAUDE_EFFORT_LEVELS`
  incl. `max`, which Codex doesn't support). `acp_model` values like
  `sonnet/high` now apply as two `session/set_config_option` calls (`model`
  + `effort`, claude-agent-acp's config id, category `thought_level`).
  Rides every existing plumbing path — init, resume-reapply, and the
  `switch_acp_model` REST endpoint — via the shared `_model_config_options`
  dispatcher, so no schema or API change was needed. `current_model_id`
  keeps the full composite string (matches Codex semantics, locked in by
  test). 446 tests pass.
- **B2 — effort in `ConversationInfo`**: mirrors the model-state pipeline
  for effort. `_extract_session_efforts` (mirroring the existing
  `_extract_session_models`) reads the `effort`/`reasoning_effort` select
  config option, stores `current_effort`/`available_efforts` on the agent,
  persists them into `agent_state`, and lifts them into `ConversationInfo`
  as new additive optional fields (live→persisted precedence, matching the
  model fields). `switch_acp_model` in `local_conversation.py` now also
  persists `acp_current_effort` unconditionally, closing the same
  cold-read staleness gap the model id already had fixed. 469+22 tests.
- **B3 — grouped selects + live refresh**: (1) `SessionConfigSelectGroup`
  entries in a select's `options` are flattened (order-preserving) before
  extraction in both `_extract_session_models` and
  `_extract_session_efforts` — previously an agent using grouped selects
  silently produced an empty model list (see "pre-existing issues" below).
  (2) `ConfigOptionUpdate` session notifications (the agent proactively
  pushing its full configOptions state — after a `set_config_option` call,
  or spontaneously, e.g. on a rate-limit model fallback) are now dispatched
  to the agent and refresh `current_model_id`/`available_models`/
  `current_effort`/`available_efforts` in memory. (3)
  `_reconcile_current_model_id` (pure, unit-tested): keeps a tracked
  composite (`sonnet/high`) verbatim when the server reports the matching
  split, recomposes it when the server reports a genuinely different
  state. 508 tests pass (39 new).

### Why upstream-worthy

- Claude effort was a one-line extension of an already-shipped pattern
  (Codex's composite splitter) — small diff, immediately consistent
  behavior across both providers Canvas exposes effort for.
- `ConversationInfo.current_effort`/`available_efforts` are additive fields
  behind the repo's own REST/oasdiff non-breaking-change policy — no
  version bump or client-compat window needed to ship them.
- B3's grouped-select fix is a **correctness fix for existing behavior**,
  not new surface area: any agent whose adapter groups its `model` or
  `effort` select options was silently getting an empty list before this,
  with no error or log — see "pre-existing issues."
- The `config_option_update` handling closes a real staleness gap: without
  it, a session that changes its own model (e.g. an automatic fallback on
  rate-limit) leaves Canvas showing a model the session is no longer
  running until the next full state reload.

### Known scope cuts (SDK)

- **Notification-path persistence**: `_on_config_options_update`
  (`acp_agent.py:2850`), invoked from the ACP bridge on every
  `ConfigOptionUpdate`, only mutates the agent's own in-memory
  `_current_model_id`/`_available_models`/`_current_effort`/
  `_available_efforts` attributes. Its own docstring is explicit: "there is
  no established hook from here into `ConversationState.agent_state` — that
  write happens only in `init_state`… Refreshed values therefore live
  in-memory only until the next `init_state`… a mid-life update is not
  durable across an agent-server restart. This is a deliberate scope cut,
  not an oversight." (`acp_agent.py:2893-2904`). Persistence-to-disk only
  happens in `init_state` (`acp_agent.py:2309`), which runs at session
  start/resume, never from the notification path itself.
- **Composite demotion nuance after a runtime switch**:
  `_reconcile_current_model_id` (`acp_agent.py:689`) only re-composes a
  tracked `model/effort` composite when the update's `reported_effort` is
  present (gate at `acp_agent.py:746`); if a pushed update reports the model
  option but omits effort, the tracked composite (`sonnet/high`) demotes to
  the bare id (`sonnet`) — exercised directly by
  `test_no_effort_reported_adopts_bare_server_model` and
  `test_non_composite_provider_adopts_bare_server_model`
  (`tests/sdk/agent/test_acp_agent.py:8791`, `:8815`). The code's own
  docstring frames this as *correct* behavior for a provider that isn't
  splitting the id ("the correct behavior for providers that don't split"),
  not a bug — but it's worth flagging upstream because it's easy to read a
  demoted `current_model_id` as "the effort got dropped," when in practice
  effort stays correct via the independently-tracked `current_effort`
  field (B2); only the composite *string* is momentarily under-specified.
- **Reconciliation gating**: gated on both (a)
  `composite_provider = provider.key in ("codex", "claude-code")`
  (`acp_agent.py:742-745`, via `detect_acp_provider_by_agent_name`) and (b)
  `model_override_applied` — a runtime `set_acp_model` switch does not set
  the latter, so a push arriving before the next full reconciliation point
  is the scenario the demotion bullet above describes. Every other
  provider (gemini-cli, unknown/custom ACP servers) always takes the
  server's reported id verbatim, bare, never split or recomposed — matching
  Canvas's own client-side `parseAcpModelId` gating, so the two layers can't
  disagree about which ids are composite
  (`test_unknown_provider_adopts_bare_server_model`,
  `test_override_not_applied_adopts_bare_server_model`).

## Pre-existing issues flagged for upstream attention

These predate this work; two are worked around rather than fixed (both need
an upstream SDK change to retire the workaround), one was fixed in-repo (B3)
and is flagged so upstream can confirm the same fix applies to their `main`.

- **`acp<0.11` pin rationale**: `openhands-sdk`'s own `pyproject.toml`
  (`openhands-sdk/pyproject.toml:8`) declares `agent-client-protocol>=0.10.1`
  with **no upper bound**, and `uv.lock` currently resolves exactly `0.10.1`
  — the SDK repo carries **no in-code comment anywhere** flagging that as
  load-bearing. Canvas's own M0 spike (2026-08-05, recorded in
  `config/defaults.json`'s `constraints.agentClientProtocol` comment and
  `PROGRESS.md`) found the reason it needs to stay that way: newer
  `agent-client-protocol` releases changed the `prompt()` call's argument
  order in a way that broke the SDK's ACP client with a `PromptRequest`
  validation error. Canvas pins the upper bound itself, downstream, at the
  `uvx` install layer (`scripts/dev-safe.mjs`) and in CI
  (`.github/workflows/mock-llm-e2e.yml`) — **the fix belongs in
  `openhands-sdk/pyproject.toml` itself** (an explicit upper bound + a
  comment explaining why), not in a downstream consumer's install script.
  Since PyPI's `agent-client-protocol` has already shipped past 0.11 (latest
  is 0.12.0 as of this writing), anyone installing `openhands-sdk` today
  without Canvas's downstream workaround can hit this for real, not just
  hypothetically.
- **`acp` 0.12 removed `set_session_model`, and the fallback branch that
  calls it has no version guard**: `_apply_acp_model`'s legacy branch
  (`acp_agent.py:503`, `await conn.set_session_model(model_id=model,
  session_id=session_id)`) — used for ACP servers that don't advertise
  `session/set_config_option` for `model` — calls `set_session_model`
  unconditionally, with no `hasattr`/version check and no comment about
  the method's removal in `ClientSideConnection` as of `acp` 0.12. Combined
  with the unbounded dependency above, bumping `agent-client-protocol` past
  0.11 would silently break this fallback for any legacy-only ACP server
  at import or first call, not at pin time. Worth an explicit guard (or a
  documented decision to drop legacy-mode support once the minimum ACP
  surface is raised).
- **`claude-agent-acp` pinned at 0.44.0 vs. 0.64.x current**:
  `CLAUDE_AGENT_ACP_VERSION = "0.44.0"` at
  `openhands-sdk/openhands/sdk/settings/acp_providers.py:393` pins the
  spawned CLI. A live stdio probe against the actual current release
  (`@agentclientprotocol/claude-agent-acp@0.64.2` — confirmed current on the
  npm registry) during this work's M0 spike found materially more/different
  `configOptions` behavior (model list sourced live from the Claude Agent
  SDK's `initializationResult`, richer effort levels) than the 0.44.0
  baseline the pin predates. Nothing in this work required moving the pin,
  but a stale CLI pin caps what model/effort surface Canvas can ever see
  server-side — worth a deliberate version-bump pass upstream, not folded
  silently into this PR stack.
- **Grouped selects silently yielded empty model lists**: before B3
  (`ryanskidmore/software-agent-sdk#3`), `_extract_session_models` (and the
  effort equivalent) only read flat `SessionConfigSelect.options`; an
  adapter that groups its options via `SessionConfigSelectGroup` produced no
  error and no log — just a silently empty model/effort list, which from
  Canvas's side is indistinguishable from "this provider has no models."
  Documented directly in `_flatten_select_options`'s own docstring
  (`acp_agent.py:517-521`): reading every entry as a flat option, the prior
  behavior, "silently drops group entries… leaving an agent using grouped
  selects with an apparently empty model/effort list." Fixed by flattening
  groups (order-preserving, `_flatten_select_options`,
  `acp_agent.py:506`) before extraction, regression-tested in
  `TestFlattenSelectOptions` and
  `test_extracts_models_from_grouped_select`/
  `test_empty_groups_yield_no_usable_models`
  (`tests/sdk/agent/test_acp_agent.py:8076,8404,8456`). Worth confirming
  against `OpenHands/software-agent-sdk` `main` in case any adapter in the
  wild already groups its options.
