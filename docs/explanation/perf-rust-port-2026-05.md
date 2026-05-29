# Perf / parity verification — Rust maximal-parity port (2026-05)

Verification that re-deriving the TypeScript fork's behavior on the Rust
v0.2.0-alpha.6 base (`origin/main` @ `35cc0ab`) does not regress performance.
Replaces the TS-era `perf-notes-2026-05-06.md`, which targeted the (now deleted)
Bun runtime.

## Method & honesty note

This is **not** a full re-run of the original 4-TUI-client CPU harness. That
harness requires a live multi-client tmux session sampled over time, which on
this machine would risk perturbing the running production sidebar (port 7391).
Instead:

- **Empirically measured** here: release binary size, the singleton guard, and
  the full test suite.
- **Code-audited** (source inspection of `origin/main`, not re-benchmarked under
  load): the hot-path perf properties the TS patches originally added. The
  finding is that the Rust rewrite already implements all of them, so there is
  nothing to re-port — only to confirm the port did not remove them.

## Empirical results

| Check | Result |
|-------|--------|
| `cargo test --workspace` | 228 passed / 0 failed (baseline 212; +16 new tests) |
| Release build | clean, ~27s (`opt-level=z, lto=fat`) |
| Singleton guard | second instance prints `another server is already running (pid N). Exiting.`, exit 0 |
| `opensessions-server` size | 938,880 → 991,712 bytes (+52 KB, +5.6%) — `notify` + system_theme + singleton |
| `opensessions-sidebar` size | +16 bytes (two new theme palettes) |
| `opensessions-sidebar-shim` size | unchanged |

## Parity audit — hot-path properties (baseline already satisfies)

| TS-era optimization | Rust baseline status | Action |
|---------------------|----------------------|--------|
| Broadcast hash-dedup (suppress no-op state pushes) | Present: `run_tmux_state_poll_loop` hashes `strip_ts_field(snapshot)` and broadcasts only on change (`lib.rs`) | none |
| Drop `eventTimestamps` from wire to enable dedup | N/A: Rust keeps `event_timestamps` on the wire but excludes `ts` at hash time, achieving the same dedup without dropping the field | none |
| tmux pane-cache reuse (one `list-panes -a` per op) | Present: `enforce_sidebar_width` calls `list_sidebar_panes` once and iterates the result | none |
| Idle-timeout bump (30s → 5min) | Moot: the Rust server has no idle shutdown — it is an always-on daemon (≥ parity) | none |
| Async fire-and-forget session switch | Present in `tmux_provider.rs` (`curl … -m 0.2 --connect-timeout 0.1 … || true`) | none |

**Conclusion:** no perf regressions to port. The ported changes are additive and
off the hot path.

## Cost of the one new runtime component (auto theme-follow)

The only feature that adds always-on runtime work is the macOS appearance
follower (`system_theme::watch_mac_system_appearance`), and only when
`autoThemeFollowsSystem` is enabled:

- **Two idle OS threads**: one blocked on `notify` filesystem events (≈0 CPU
  while idle), one sleeping on a 60s safety-poll interval.
- **No hot polling.** `defaults read -g AppleInterfaceStyle` is spawned only on
  an actual `.GlobalPreferences.plist` change plus once per 60s safety tick —
  ~1,440 spawns/day worst case, versus the old 3s poll's ~28,800/day. This
  matches the TS push-based parity target.
- Disabled by default and never started in tests (gated by `with_auto_theme_follow`,
  set only by the real bootstrap), so it has zero cost unless the user opts in.

## Verdict

Functional and performance parity confirmed. All baseline hot-path
optimizations are retained; the additions (themes, two-tier prune, watcher-cache
eviction, auto theme-follow, singleton guard) do not touch the broadcast or
tmux-poll hot paths. The singleton guard additionally improves on the bare
`AddrInUse` failure of the baseline.
