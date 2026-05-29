# opensessions fork → Rust v0.2.0-alpha.6: maximal-parity port

- **Date:** 2026-05-29
- **Status:** approved (design); driving the implementation plan
- **Base commit:** `origin/main` @ `35cc0ab` (v0.2.0-alpha.6)
- **Worktree:** `~/Developer/opensessions-port` on branch `rust-port`
- **Baseline:** `cargo test --workspace` → 212 passed / 0 failed / 0 ignored

## TL;DR

Upstream replaced the entire TypeScript/Bun runtime with a Rust/ratatui rewrite and deleted the TS code our fork patched. We re-derive our fork's behavior on the Rust base — **functional parity, not a line port** — verify it with before/after benchmarks (maximal-parity bar), install locally, then split the result into an upstream PR (portable features) and fork-only commits (personal/local glue).

The rewrite already absorbed most of our perf/robustness work and **preserves the Ella bridge contract** (`127.0.0.1:7391` + `POST /api/agent-event`). The genuine remaining work is concentrated: the auto dark/light "theme-follows-system" feature, two themes, and one agent-prune correctness tweak.

## Goals / Non-goals

**Goals**
1. Functional 1:1 parity of our fork's user-observable behavior + perf/robustness properties on the Rust base.
2. Prove parity with before/after benchmarks; port any item where stock Rust regresses against a property our patch guaranteed.
3. Build + install locally so the live tmux sidebar runs the ported Rust build, Ella bridge intact.
4. Leave a clean, PR-ready commit set: portable features → upstream `Ataraxy-Labs/opensessions`; personal glue → `panosAthDBX` fork.

**Non-goals**
- No protocol/wire changes (keep WS + `/api/agent-event` contract).
- No zellij work.
- No re-porting of items the rewrite already solved (see inventory "skip" rows).
- Not rewriting beyond parity; no unrelated refactors.

## Current state (verified this session)

- Base = 6 Rust crates: `runtime-rs`, `sidebar-core-rs`, `sidebar-protocol-rs`, `server-rs`, `tui-rs`, `sidebar-shim-rs`. `package.json`/`turbo.json` vestigial; build/test is pure `cargo`.
- **Bridge survives:** `server-rs/src/lib.rs` binds `127.0.0.1:7391` (fixed default, `None => 7391`) and routes `POST /api/agent-event` → `apply_agent_event` → `agent_tracker.apply_event`. The Ella plugin (`~/.hermes/plugins/opensessions-bridge`) POSTs `/api/agent-event` to `OPENSESSIONS_URL` (default `http://127.0.0.1:7391`) over the SSH reverse tunnel. Transport intact; **only payload-shape needs verification**.
- `port_discovery.rs`/`portless.rs` are for discovering localhost dev-server ports *inside sessions*, NOT the server's own port. Not a bridge risk.
- **Already absorbed by the rewrite (skip):** async fire-and-forget session switch (`tmux_provider.rs`), unseen-aware terminal prune at 5min (`tracker.rs`), `tokyo-night` built-in theme.
- **`AGENTS.md` is stale** — still documents the Bun/TS/OpenTUI architecture. Optional upstream freebie; not required for parity.

## Port inventory (disposition · target · test · tag)

Tags: **[upstream]** = portable, goes in the upstream PR · **[local]** = fork-only · **[verify]** = confirm gap before porting.

| # | Item | Disposition | Lands in | Test | Tag |
|---|------|-------------|----------|------|-----|
| 1 | Auto dark/light theme-follows-system | **Confirmed absent → port** | new `runtime-rs/src/system_theme.rs` + config + theme-apply wiring in `server-rs` | new `runtime-rs/tests/system_theme.rs` + extend `tests/config_and_shared.rs` | upstream |
| 2 | `tango-adapted` light theme | **Confirmed absent → port** | `sidebar-core-rs/src/renderer.rs` (palette + registry + resolver) | extend `apps/tui-rs/tests/theme.rs` | upstream |
| 3 | `tokyo-night-storm` variant | absent (base `tokyo-night` exists) → port | `renderer.rs` | extend `theme.rs` | upstream |
| 4 | Header bg `crust` → `base` | **[verify]** Rust default, then match | `renderer.rs` / `sidebar-core-rs` app default | `render_snapshots.rs` (update) | verify |
| 5 | Unseen agents hard-prune @ 15min | **port** (upstream `prune_terminal` never prunes unseen → unbounded) | `runtime-rs/src/tracker.rs` | extend `tests/tracker.rs` | upstream |
| 6 | opencode local-snapshot eviction @ 15min | **[verify]** `agent_watchers.rs`, port if absent | `runtime-rs/src/agent_watchers.rs` | extend `tests/agent_watchers.rs` | upstream if gap |
| 7 | Server singleton guard (PID probe) | **[verify]** does Rust bind path already handle EADDRINUSE/singleton? | `server-rs/src/lib.rs` | server test | upstream if gap |
| 8 | Idle-timeout 30s → 5min | **[verify]** Rust const value | `server-rs` | — | upstream if gap |
| 9 | Broadcast hash-dedup / drop `eventTimestamps` | **[verify]** does Rust re-broadcast no-ops? does protocol carry timestamps? | `server-rs`/`runtime-rs` `server_state.rs`, `protocol.rs` | benchmark-driven | port if gap (likely N/A) |
| 10 | tmux pane-cache reuse (halve `list-panes -a`) | **[verify]** `sidebar_width_sync.rs` | `runtime-rs` | `sidebar_coordinator.rs` | port if gap |
| 11 | async session switch | **already upstream → skip** | — | — | skip |
| 12 | `opensessions.tmux` bun bootstrap, `server-common.sh` rewrite | **superseded by cargo launch → skip** | — | — | skip |
| 13 | `perf-notes-2026-05-06.md` (TS) | replaced by fresh Rust benchmark note | `docs/` (new) | — | local |
| 14 | Ella-bridge payload-shape verification + e2e | **verify + fix if needed** | bridge plugin (not in repo) and/or doc | manual e2e | local |

## Feature design: auto theme-follows-system (item 1, the real work)

Replicate the three TS primitives from `system-theme.ts` idiomatically in Rust, `cfg(target_os = "macos")`-gated so the upstream PR stays cross-platform.

- `enum SystemAppearance { Dark, Light }`.
- `read_mac_system_appearance() -> SystemAppearance` — macOS: spawn `defaults read -g AppleInterfaceStyle`; `"Dark"` → Dark, absent/error → Light. Non-macOS: always Light (no spawn).
- `theme_for_system_mode(mode, dark_theme, light_theme) -> String` — pure; Dark→dark, Light→light. Cross-platform unit-testable.
- `watch_mac_system_appearance(on_change, safety_poll)` — push-based: watch `~/Library/Preferences/.GlobalPreferences.plist`; on each fs event re-read appearance and fire `on_change` **only when the value changed**; 60s safety-poll thread for the atomic-rename case; fire once on start with the initial mode; return a handle with idempotent `stop()`. Non-macOS: no-op handle.
  - Watch mechanism: prefer reusing the runtime's existing file-watch dependency (HEAD/git-cache watchers already exist) to avoid reintroducing a poll loop. **[verify]** whether `notify` (or equivalent) is already in `Cargo.lock`; if not, add `notify`. Polling `defaults` on a timer is NOT acceptable (defeats the perf property in item 9/benchmarks).
- **Override semantics (persist manual override per system appearance):** auto-switch the active theme on appearance change *unless* the user has manually set a theme for the current appearance; a manual theme change records an override keyed by the appearance that was active when they set it. **[read during impl]** the exact state machine from our fork's `server/index.ts` + `config.ts` to match it precisely; mirror into `server-rs` theme-apply path + `runtime-rs/config.rs`.

## Benchmark plan (maximal-parity bar)

Two release builds: **baseline** = clean `origin/main`; **after** = ported branch. Measure on the same machine, same tmux, comparable ambient agent load. Metrics + method from the original perf note, re-measured on Rust:

| Metric | Method | Pass bar |
|--------|--------|----------|
| Idle CPU% + RSS (4 TUI clients) | `ps -o %cpu,rss -p <pid>` sampled @5s over 30s | after ≤ baseline (no regression) |
| Theme-detection subprocess spawns | count `defaults` spawns over a 10-min idle window | ~0 idle; spawn only on real appearance change |
| Session-switch latency | timestamp deltas in debug log | after ≤ baseline |
| Broadcast dedup under agent storm | broadcasts vs `agent-emit` ratio over 30s | after ≤ baseline (no-op pings suppressed) |
| EADDRINUSE on rapid respawn | start server twice in succession | second exits cleanly, no port clash |

**Regression rule:** any axis where *after < baseline*, or where stock Rust fails a property our fork's patch guaranteed → port the corresponding fix (items 7–10). Record results in a new `docs/` perf note (replaces the TS one).

## Testing strategy

- TDD for features (items 1, 2, 3, 5): write the Rust test first, then implement.
- Extend existing suites at the right layer: `tests/tracker.rs`, `tests/agent_watchers.rs`, `apps/tui-rs/tests/theme.rs`, `tests/config_and_shared.rs`; update `render_snapshots.rs` if palettes/bg shift snapshot bytes.
- `cargo test --workspace` green at every step; never regress the 212 baseline.
- Honor `docs/explanation/sidebar-behavior.md` invariants if any width/coordinator code is touched (per repo guidelines).

## Integration / install

1. All work on `rust-port` in the worktree; atomic commits tagged `[upstream]`/`[local]`.
2. When green + benchmarked + approved: in the live plugin dir (`~/.tmux/plugins/opensessions`) check out the ported branch and `cargo build --release`; reload tmux; `prefix o → s` smoke.
3. Acceptance: sidebar renders; theme auto-switches on macOS appearance flip; both new themes selectable; Ella bridge pill updates from a remote Hermes event over the 7391 tunnel; benchmarks recorded.
4. Rollback: the previous TS build/branch remains in git; `git checkout` + rebuild restores it.

## Commit / PR strategy (upstream + fork split)

- Upstream feature branches off `origin/main`, cherry-picking only portable commits: auto-theme-follow (item 1), `tango-adapted` (+ `tokyo-night-storm`) (2,3), unseen hard-prune (5), and any verified robustness gaps (6–10). Optional: AGENTS.md refresh.
- Fork-only: bridge-specific glue (14), the perf note (13), this spec, and anything not cleanly cross-platform.
- Keep the macOS feature `cfg`-gated and the upstream commits self-contained so each is independently reviewable.

## Risks & mitigations

- **macOS appearance watch reliability in Rust** → wrap detection behind a small trait; reuse existing fs-watch dep; 60s safety poll as backstop; pure mapping fn keeps the logic testable without a real flip.
- **Cross-platform upstream PR** → `cfg(target_os="macos")` with no-op fallbacks (mirrors the TS non-darwin behavior, which has explicit tests).
- **Snapshot churn from new palettes/bg** → expected; regenerate snapshots deliberately and eyeball the diff.
- **Override state-machine mismatch** → port from the TS source verbatim in semantics; cover with config tests.
- **First release build time** (`lto=fat, opt-level=z`) → use `release-dev` profile for iteration, `release` only for the final install/benchmark.

## Open items to resolve during implementation

- [verify] header bg default in Rust (item 4).
- [verify] `notify`/fs-watch dep already present (item 1 watcher).
- [verify] singleton/idle/dedup/pane-cache are real gaps (items 7–10) — benchmark-driven.
- [read] exact theme-override state machine from fork `server/index.ts` + `config.ts`.
- [verify] `/api/agent-event` payload shape: bridge sends `{status, thread_name, ...}`; confirm against `apply_agent_event` field expectations (agent, status, session).
