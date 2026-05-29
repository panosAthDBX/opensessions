# Rust v0.2.0-alpha.6 Maximal-Parity Port — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-derive the TypeScript fork's behavior on the Rust/ratatui base so the live tmux sidebar is functionally 1:1, benchmark-verified, installed, and PR-ready.

**Architecture:** Work on branch `rust-port` in worktree `~/Developer/opensessions-port` (base `origin/main` @ `35cc0ab`). Port only confirmed gaps; the rewrite already absorbed the rest. Each task is TDD (failing test → implement → green → commit). macOS-specific code is `cfg(target_os = "macos")`-gated with no-op fallbacks so the upstream PR stays cross-platform.

**Tech Stack:** Rust (cargo workspace, 6 crates), `tokio`, `ratatui`, `tokio-websockets`, `serde_json`, `notify` (to be added for the appearance watcher). Tests: `cargo test`.

---

## Revised port inventory (post-investigation)

| # | Item | Verdict | Action |
|---|------|---------|--------|
| 1 | Auto theme-follows-system | absent | **PORT** (Phase D) — config + `system_theme` module + server wiring + set-theme persistence |
| 2 | `tango-adapted` theme | absent | **PORT** (Phase A) — down-map to 18-field palette |
| 3 | `tokyo-night-storm` theme | absent | **PORT** (Phase A) |
| 5 | Unseen hard-prune @15min | absent (Rust never prunes unseen) | **PORT** (Phase B) — extend `prune_terminal` |
| 6 | opencode local eviction | verify | **VERIFY+PORT** (Phase C) |
| 7 | Server singleton guard | GAP (bare bind error) | **PORT** (Phase E) |
| 4 | bg `crust`→`base` | N/A (no such fields; header bg `None`, focused `surface1`) | SKIP |
| 8 | idle-timeout 5min | moot (Rust has no idle shutdown) | SKIP |
| 9 | broadcast dedup / drop eventTimestamps | already handled (`strip_ts_field`+hash) | SKIP (validate in Phase F) |
| 10 | pane-cache reuse | already handled | SKIP (validate in Phase F) |
| 11 | async switch | already upstream | SKIP |
| 12 | bun/shell bootstrap | superseded by cargo | SKIP |
| 14 | Ella bridge payload | MATCH (camelCase aligns) | e2e smoke only (Phase G) |

**Reference data:**
- Rust `Palette` (18 `Rgb` fields): `white, black, blue, lavender, pink, yellow, green, red, peach, teal, sky, text, subtext0, subtext1, overlay0, overlay1, surface1, surface2` — `packages/sidebar-core-rs/src/renderer.rs:1398`. No `base/crust/mantle/surface0`.
- Resolver: `palette_for_theme(name: Option<&str>) -> Palette` at `renderer.rs:1868`. Registry string array at `renderer.rs:1843`.
- Config: `OpensessionsConfig` (camelCase serde) at `packages/runtime-rs/src/config.rs:18`; path `~/.config/opensessions/config.json`; `load_config_from_home`/`save_config_to_home` (merge-on-save) at `config.rs:47/64`.
- Server theme state: `ReadOnlyMuxStateSource.theme: Mutex<Option<String>>`; `set-theme` handler `apps/server-rs/src/lib.rs:554`; broadcast via `snapshot_json()` on the `state_updates` channel.
- Server bind: `apps/server-rs/src/lib.rs:1972` (`TcpListener::bind`), PID file written after bind at ~1983.
- Tracker prune: `packages/runtime-rs/src/tracker.rs:210` (`prune_terminal`), `TERMINAL_PRUNE_MS = 5*60*1000` at `tracker.rs:6`.

---

## Phase A — New themes (items 2, 3)

### Task A1: Add `tokyo-night-storm` and `tango-adapted` palettes + registry + resolver

**Files:**
- Modify: `packages/sidebar-core-rs/src/renderer.rs` (palette consts near ~1503; registry array ~1843; resolver `palette_for_theme` ~1868)
- Test: `apps/tui-rs/tests/theme.rs`

- [ ] **Step 1: Write failing tests** — append to `apps/tui-rs/tests/theme.rs`. (These call the same public surface the existing 7 theme tests use; match their import style — likely `use opensessions_sidebar_core::renderer::palette_for_theme;`. Confirm the exact path from the top of the existing file before writing.)

```rust
#[test]
fn tokyo_night_storm_resolves_to_distinct_palette() {
    let storm = palette_for_theme(Some("tokyo-night-storm"));
    let night = palette_for_theme(Some("tokyo-night"));
    let default = palette_for_theme(None);
    // storm is a real, distinct entry (not the unknown→default fallback)
    assert_ne!(storm, default, "storm must not fall back to default");
    assert_ne!(storm, night, "storm must differ from base tokyo-night");
}

#[test]
fn tango_adapted_is_a_light_palette() {
    let tango = palette_for_theme(Some("tango-adapted"));
    let default = palette_for_theme(None);
    assert_ne!(tango, default, "tango must be a real entry");
    // Light theme: text is near-black, surfaces near-white.
    assert!(tango.text.is_dark(), "tango text should be dark on light bg");
}
```

Note: if `Palette`/`Rgb` don't derive `PartialEq` or lack an `is_dark()` helper, adjust the assertions to compare a representative channel (e.g. `assert!(tango.text == Rgb::new(0,0,0))` and `assert!(storm.surface1 == Rgb::new(52,58,82))`). Check the derives on `Palette`/`Rgb` first and write assertions against what exists.

- [ ] **Step 2: Run, expect fail**

Run: `cargo test -p opensessions-tui --test theme`
Expected: FAIL — both new names currently resolve to `CATPPUCCIN_MOCHA` (so `assert_ne!(_, default)` fails).

- [ ] **Step 3: Add the two palette consts** in `renderer.rs` (place after the existing `TOKYO_NIGHT` const ~line 1522):

```rust
const TOKYO_NIGHT_STORM: Palette = Palette {
    white: Rgb::new(255, 255, 255),
    black: Rgb::new(0, 0, 0),
    blue: Rgb::new(122, 162, 247),
    lavender: Rgb::new(187, 154, 247),
    pink: Rgb::new(187, 154, 247),
    yellow: Rgb::new(224, 175, 104),
    green: Rgb::new(158, 206, 106),
    red: Rgb::new(247, 118, 142),
    peach: Rgb::new(255, 158, 100),
    teal: Rgb::new(115, 218, 202),
    sky: Rgb::new(125, 207, 255),
    text: Rgb::new(192, 202, 245),
    subtext0: Rgb::new(169, 177, 214),
    subtext1: Rgb::new(154, 165, 206),
    overlay0: Rgb::new(78, 85, 117),
    overlay1: Rgb::new(59, 66, 97),
    surface1: Rgb::new(52, 58, 82),
    surface2: Rgb::new(65, 72, 104),
};

const TANGO_ADAPTED: Palette = Palette {
    white: Rgb::new(255, 255, 255),
    black: Rgb::new(0, 0, 0),
    blue: Rgb::new(0, 162, 255),
    lavender: Rgb::new(193, 126, 204),
    pink: Rgb::new(233, 167, 225),
    yellow: Rgb::new(227, 190, 0),
    green: Rgb::new(89, 214, 0),
    red: Rgb::new(255, 0, 0),
    peach: Rgb::new(206, 92, 0),
    teal: Rgb::new(0, 208, 214),
    sky: Rgb::new(136, 201, 255),
    text: Rgb::new(0, 0, 0),
    subtext0: Rgb::new(60, 60, 60),
    subtext1: Rgb::new(85, 85, 85),
    overlay0: Rgb::new(143, 146, 139),
    overlay1: Rgb::new(192, 197, 187),
    surface1: Rgb::new(220, 220, 220),
    surface2: Rgb::new(200, 200, 200),
};
```

- [ ] **Step 4: Register the names.** In the resolver `palette_for_theme` (~1868) add two arms before the `_ => CATPPUCCIN_MOCHA` line:

```rust
        Some("tokyo-night-storm") => TOKYO_NIGHT_STORM,
        Some("tango-adapted") => TANGO_ADAPTED,
```

And add both names to the registry string array (~1843, the list that begins `"catppuccin-mocha", ...`) so the theme cycler offers them:

```rust
    "tokyo-night-storm",
    "tango-adapted",
```

- [ ] **Step 5: Run tests + full suite**

Run: `cargo test -p opensessions-tui --test theme` → PASS
Run: `cargo test --workspace` → 214+ pass, 0 fail. If `render_snapshots.rs` fails because a snapshot enumerates theme names, update the snapshot intentionally and eyeball the diff.

- [ ] **Step 6: Commit**

```bash
git add packages/sidebar-core-rs/src/renderer.rs apps/tui-rs/tests/theme.rs
git commit -m "feat(themes): add tokyo-night-storm and tango-adapted" -m "Co-authored-by: Isaac"
```

---

## Phase B — Unseen terminal hard-prune (item 5)

### Task B1: Two-tier `prune_terminal` (seen→5min, unseen→15min, alive→never)

**Files:**
- Modify: `packages/runtime-rs/src/tracker.rs` (const ~line 6; `prune_terminal` ~210-237)
- Test: `packages/runtime-rs/tests/tracker.rs`

Parity target (from TS WIP): unseen terminal instances currently survive forever in Rust (`prune_terminal` skips `unseen_instances`). Add a hard cap: prune unseen terminal instances after 15min; keep the 5min prune for seen ones; never prune `Alive`. Also clean `unseen_instances` when hard-pruning.

- [ ] **Step 1: Write failing test** — append to `packages/runtime-rs/tests/tracker.rs`. First read the file header for the existing helpers/imports (how tests build an `AgentTracker`, craft an `AgentEvent`, and manipulate `ts`/time). Reuse those. The test must: insert a terminal (`Done`) unseen event with `ts` 16 min in the past, call `prune_terminal()`, assert it is gone; and insert an unseen terminal event 10 min old, assert it survives.

```rust
#[test]
fn prune_terminal_hard_prunes_unseen_after_15min() {
    let mut t = AgentTracker::new();
    // unseen because session is not active when a terminal event lands
    t.apply_seed_event(done_event("sess", "claude", /*ts*/ now_ms() - 16 * 60 * 1000));
    assert!(t.is_unseen("sess"));
    t.prune_terminal();
    assert!(t.get_agents("sess").is_empty(), "16-min-old unseen terminal must be hard-pruned");
    assert!(!t.is_unseen("sess"), "unseen flag cleared on hard-prune");
}

#[test]
fn prune_terminal_keeps_unseen_under_15min() {
    let mut t = AgentTracker::new();
    t.apply_seed_event(done_event("sess", "claude", now_ms() - 10 * 60 * 1000));
    t.prune_terminal();
    assert_eq!(t.get_agents("sess").len(), 1, "10-min-old unseen terminal must survive");
}
```

If `done_event`/`now_ms` test helpers don't exist, add small local helpers in the test mirroring the existing tests' construction of `AgentEvent` (status `AgentStatus::Done`, the given `ts`, `liveness: None`).

- [ ] **Step 2: Run, expect fail**

Run: `cargo test -p opensessions-runtime --test tracker prune_terminal`
Expected: `prune_terminal_hard_prunes_unseen_after_15min` FAILS (unseen never pruned today).

- [ ] **Step 3: Implement.** Add the const near `tracker.rs:6`:

```rust
const TERMINAL_HARD_PRUNE_MS: u64 = 15 * 60 * 1000;
```

Replace the filter + removal in `prune_terminal` (~218-230) so it computes per-key age and unseen status, and collects keys to also clear from `unseen_instances`:

```rust
        for session in sessions {
            let unseen_instances = self.unseen_instances.clone();
            let mut empty = false;
            let mut cleared_unseen: Vec<String> = Vec::new();
            if let Some(session_instances) = self.instances.get_mut(&session) {
                let keys = session_instances
                    .iter()
                    .filter(|(key, event)| {
                        if !is_terminal_status(event.status)
                            || event.liveness == Some(AgentLiveness::Alive)
                        {
                            return false;
                        }
                        let age = now.saturating_sub(event.ts);
                        let is_unseen =
                            unseen_instances.contains(&format!("{session}\0{key}"));
                        if is_unseen {
                            age > TERMINAL_HARD_PRUNE_MS
                        } else {
                            age > TERMINAL_PRUNE_MS
                        }
                    })
                    .map(|(key, _)| key.clone())
                    .collect::<Vec<_>>();
                for key in keys {
                    session_instances.remove(&key);
                    cleared_unseen.push(format!("{session}\0{key}"));
                }
                empty = session_instances.is_empty();
            }
            for ukey in cleared_unseen {
                self.unseen_instances.remove(&ukey);
            }
            if empty {
                self.instances.remove(&session);
            }
        }
```

- [ ] **Step 4: Run tests**

Run: `cargo test -p opensessions-runtime --test tracker` → PASS (existing 9 + 2 new)
Run: `cargo test --workspace` → green.

- [ ] **Step 5: Commit**

```bash
git add packages/runtime-rs/src/tracker.rs packages/runtime-rs/tests/tracker.rs
git commit -m "feat(tracker): hard-prune unseen terminal agents after 15min" -m "Co-authored-by: Isaac"
```

---

## Phase C — opencode local eviction (item 6, verify-then-port)

### Task C1: Verify whether the Rust opencode watcher already bounds its local map

**Files:** read `packages/runtime-rs/src/agent_watchers.rs`, `packages/runtime-rs/tests/agent_watchers.rs`

- [ ] **Step 1:** Read `agent_watchers.rs`. Find the opencode watcher's local per-session cache (the analogue of the TS `this.sessions: Map<id, {..., lastGrowthAt}>`). Answer:
  - Is there a local map keyed by session id that persists across poll cycles?
  - Does it track a "last DB growth" timestamp per session?
  - Is there any time-based eviction of entries not seen in the current poll?

- [ ] **Step 2: Decide.**
  - If the Rust watcher is **stateless across polls** (re-reads full state each cycle, no growing map) → **the concern doesn't exist; SKIP Phase C.** Record this in the commit-less plan note and move on.
  - If it keeps an **unbounded growing map** with no eviction → proceed to C2.

### Task C2 (only if gap): Add 15-min local eviction

**Files:** Modify `packages/runtime-rs/src/agent_watchers.rs`; Test `packages/runtime-rs/tests/agent_watchers.rs`

Parity target (TS): `LOCAL_EVICT_MS = 15*60*1000`; at end of each poll cycle, for each cached session not seen this cycle, if `now - last_growth_at >= LOCAL_EVICT_MS`, remove it.

- [ ] **Step 1: Write a failing test** matching the watcher's actual constructor/poll seam (read the existing 4 tests for the pattern — they likely inject rows / a fake clock). Assert a session absent from the latest rows for >15min is dropped from the internal map, while one absent <15min is retained. (Write concrete assertions against the real public/test surface found in Step C1.)
- [ ] **Step 2: Run → fail.** `cargo test -p opensessions-runtime --test agent_watchers`
- [ ] **Step 3: Implement** the eviction loop at the end of the opencode poll, mirroring the TS condition: track `last_growth_at` per cached session (set to `now` when the row's `time_updated` advances), build a `seen_this_cycle` set, then evict unseen entries older than `LOCAL_EVICT_MS`.
- [ ] **Step 4: Run → pass; `cargo test --workspace` green.**
- [ ] **Step 5: Commit**

```bash
git add packages/runtime-rs/src/agent_watchers.rs packages/runtime-rs/tests/agent_watchers.rs
git commit -m "feat(opencode): evict stale local session snapshots after 15min" -m "Co-authored-by: Isaac"
```

---

## Phase D — Auto theme-follows-system (item 1, the main feature)

macOS-gated; mirrors the TS `system-theme.ts` + `server/index.ts` semantics. Built bottom-up: config → pure helpers → watcher → server wiring → manual-override persistence.

### Task D1: Config fields `autoThemeFollowsSystem`, `darkTheme`, `lightTheme`

**Files:** Modify `packages/runtime-rs/src/config.rs` (struct ~18-39); Test `packages/runtime-rs/tests/config_and_shared.rs`

- [ ] **Step 1: Failing test** — append to `tests/config_and_shared.rs` (reuse its temp-home pattern):

```rust
#[test]
fn config_roundtrips_auto_theme_fields() {
    let dir = tempdir_home(); // use the file's existing temp-home helper
    save_config_to_home(dir.path(), OpensessionsConfig {
        auto_theme_follows_system: Some(true),
        dark_theme: Some("tokyo-night-storm".into()),
        light_theme: Some("tango-adapted".into()),
        ..Default::default()
    }).unwrap();
    let loaded = load_config_from_home(dir.path());
    assert_eq!(loaded.auto_theme_follows_system, Some(true));
    assert_eq!(loaded.dark_theme.as_deref(), Some("tokyo-night-storm"));
    assert_eq!(loaded.light_theme.as_deref(), Some("tango-adapted"));
}
```

- [ ] **Step 2: Run → fail** (fields don't exist). `cargo test -p opensessions-runtime --test config_and_shared`
- [ ] **Step 3: Implement** — add to `OpensessionsConfig` (serde already `rename_all = "camelCase"`, so snake_case field names map to camelCase JSON automatically):

```rust
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub auto_theme_follows_system: Option<bool>,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub dark_theme: Option<String>,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub light_theme: Option<String>,
```

- [ ] **Step 4: Run → pass; `cargo test --workspace` green.**
- [ ] **Step 5: Commit** `feat(config): add autoThemeFollowsSystem/darkTheme/lightTheme`.

### Task D2: `system_theme` module — pure mapping + appearance read

**Files:** Create `packages/runtime-rs/src/system_theme.rs`; Modify `packages/runtime-rs/src/lib.rs` (add `pub mod system_theme;`); Create `packages/runtime-rs/tests/system_theme.rs`

- [ ] **Step 1: Failing tests** (`tests/system_theme.rs`):

```rust
use opensessions_runtime::system_theme::{theme_for_system_mode, SystemAppearance, read_mac_system_appearance};

#[test]
fn theme_for_system_mode_maps_dark_and_light() {
    assert_eq!(theme_for_system_mode(SystemAppearance::Dark, "catppuccin-mocha", "catppuccin-latte"), "catppuccin-mocha");
    assert_eq!(theme_for_system_mode(SystemAppearance::Light, "catppuccin-mocha", "catppuccin-latte"), "catppuccin-latte");
}

#[test]
fn read_appearance_is_total_and_returns_a_variant() {
    // On non-macOS this is always Light; on macOS it reads the real setting.
    let m = read_mac_system_appearance();
    assert!(matches!(m, SystemAppearance::Dark | SystemAppearance::Light));
}
```

- [ ] **Step 2: Run → fail** (module missing). `cargo test -p opensessions-runtime --test system_theme`
- [ ] **Step 3: Implement** `system_theme.rs`:

```rust
//! macOS system-appearance helpers (parity with the TS system-theme module).

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SystemAppearance {
    Dark,
    Light,
}

/// Pure: map a detected appearance + configured theme names to the theme to apply.
pub fn theme_for_system_mode(mode: SystemAppearance, dark_theme: &str, light_theme: &str) -> String {
    match mode {
        SystemAppearance::Dark => dark_theme.to_string(),
        SystemAppearance::Light => light_theme.to_string(),
    }
}

/// Read the current macOS Appearance. `defaults read -g AppleInterfaceStyle`
/// prints "Dark" in dark mode and exits non-zero/empty in light mode.
/// Non-macOS: always Light. Never panics.
#[cfg(target_os = "macos")]
pub fn read_mac_system_appearance() -> SystemAppearance {
    use std::process::Command;
    match Command::new("defaults").args(["read", "-g", "AppleInterfaceStyle"]).output() {
        Ok(out) => {
            let s = String::from_utf8_lossy(&out.stdout);
            if s.trim() == "Dark" { SystemAppearance::Dark } else { SystemAppearance::Light }
        }
        Err(_) => SystemAppearance::Light,
    }
}

#[cfg(not(target_os = "macos"))]
pub fn read_mac_system_appearance() -> SystemAppearance {
    SystemAppearance::Light
}
```

Add `pub mod system_theme;` to `packages/runtime-rs/src/lib.rs`.

- [ ] **Step 4: Run → pass; workspace green.**
- [ ] **Step 5: Commit** `feat(runtime): system_theme appearance read + pure theme mapping`.

### Task D3: Appearance watcher (push-based, macOS-gated)

**Files:** Add `notify` dep to `packages/runtime-rs/Cargo.toml`; Modify `packages/runtime-rs/src/system_theme.rs`; Test `packages/runtime-rs/tests/system_theme.rs`

Parity target: watch `~/Library/Preferences/.GlobalPreferences.plist`; on event re-read appearance and fire `on_change` only when the value changed; 60s safety poll; fire once on start; `stop()` idempotent; no-op on non-macOS.

- [ ] **Step 1: Failing test** (cross-platform-safe — exercises the no-op path + handle shape):

```rust
use opensessions_runtime::system_theme::watch_mac_system_appearance;
use std::sync::{Arc, atomic::{AtomicUsize, Ordering}};

#[test]
fn watcher_handle_stop_is_idempotent() {
    let calls = Arc::new(AtomicUsize::new(0));
    let c = calls.clone();
    let w = watch_mac_system_appearance(move |_mode| { c.fetch_add(1, Ordering::SeqCst); }, Some(60_000));
    w.stop();
    w.stop(); // must not panic
    // On macOS the initial check may fire once; on non-macOS it stays 0. Either is fine.
    assert!(calls.load(Ordering::SeqCst) <= 1);
}
```

- [ ] **Step 2: Run → fail** (fn missing).
- [ ] **Step 3: Add dep** to `packages/runtime-rs/Cargo.toml`:

```toml
[target.'cfg(target_os = "macos")'.dependencies]
notify = "6"
```

(Run `cargo build -p opensessions-runtime` once to resolve/lock the crate.)

- [ ] **Step 4: Implement** `watch_mac_system_appearance` in `system_theme.rs`:

```rust
pub struct AppearanceWatcher {
    stop: std::sync::Arc<std::sync::atomic::AtomicBool>,
}

impl AppearanceWatcher {
    pub fn stop(&self) {
        self.stop.store(true, std::sync::atomic::Ordering::SeqCst);
    }
}

#[cfg(not(target_os = "macos"))]
pub fn watch_mac_system_appearance<F>(_on_change: F, _safety_poll_ms: Option<u64>) -> AppearanceWatcher
where
    F: Fn(SystemAppearance) + Send + 'static,
{
    AppearanceWatcher { stop: std::sync::Arc::new(std::sync::atomic::AtomicBool::new(true)) }
}

#[cfg(target_os = "macos")]
pub fn watch_mac_system_appearance<F>(on_change: F, safety_poll_ms: Option<u64>) -> AppearanceWatcher
where
    F: Fn(SystemAppearance) + Send + 'static,
{
    use std::sync::{Arc, Mutex};
    use std::sync::atomic::{AtomicBool, Ordering};
    use std::time::Duration;

    let stop = Arc::new(AtomicBool::new(false));
    let last: Arc<Mutex<Option<SystemAppearance>>> = Arc::new(Mutex::new(None));
    let on_change = Arc::new(on_change);

    let plist = dirs_home().join("Library/Preferences/.GlobalPreferences.plist");

    let check = {
        let last = last.clone();
        let on_change = on_change.clone();
        move || {
            let mode = read_mac_system_appearance();
            let mut guard = last.lock().unwrap();
            if *guard != Some(mode) {
                *guard = Some(mode);
                on_change(mode);
            }
        }
    };

    // Initial fire so the consumer learns the starting mode.
    check();

    // notify watcher → re-check on any plist write.
    {
        use notify::{RecursiveMode, Watcher};
        let check = check.clone();
        let stop_w = stop.clone();
        std::thread::spawn(move || {
            let (tx, rx) = std::sync::mpsc::channel();
            let Ok(mut watcher) = notify::recommended_watcher(move |res| { let _ = tx.send(res); }) else { return; };
            if watcher.watch(&plist, RecursiveMode::NonRecursive).is_err() { return; }
            while !stop_w.load(Ordering::SeqCst) {
                match rx.recv_timeout(Duration::from_millis(500)) {
                    Ok(_) => check(),
                    Err(std::sync::mpsc::RecvTimeoutError::Timeout) => {}
                    Err(_) => break,
                }
            }
        });
    }

    // 60s safety poll (atomic-rename case).
    {
        let poll = Duration::from_millis(safety_poll_ms.unwrap_or(60_000));
        let stop_p = stop.clone();
        std::thread::spawn(move || {
            while !stop_p.load(Ordering::SeqCst) {
                std::thread::sleep(poll);
                if stop_p.load(Ordering::SeqCst) { break; }
                check();
            }
        });
    }

    AppearanceWatcher { stop }
}

#[cfg(target_os = "macos")]
fn dirs_home() -> std::path::PathBuf {
    std::env::var_os("HOME").map(std::path::PathBuf::from).unwrap_or_default()
}
```

Note: the closure must be `Clone` (wrap in `Arc` as above) since it's used by both threads and the initial fire. Adjust capture if the compiler complains; keep behavior identical.

- [ ] **Step 5: Run → pass; workspace green.**
- [ ] **Step 6: Commit** `feat(runtime): push-based macOS appearance watcher (notify + 60s safety poll)`.

### Task D4: Wire the watcher into the server (auto-switch on appearance change)

**Files:** Modify `apps/server-rs/src/lib.rs` (server start/event-loop region; theme mutex + `state_updates` channel)

Parity target (TS `syncSystemTheme`): when `auto_theme_follows_system` and macOS: start the watcher; on each change re-load config, compute `theme_for_system_mode(mode, dark_theme||default_dark, light_theme||default_light)`, and if it differs from the current theme, set `*self.theme.lock()` and broadcast a fresh `snapshot_json()`. Defaults: dark=`catppuccin-mocha`, light=`catppuccin-latte`.

- [ ] **Step 1:** Read the server start path (~`start_server`/`run_*_loop`, lib.rs:~1300-2050) to find: (a) the `Arc` handle to the state source (so the watcher closure can lock `theme` + call `snapshot_json`), (b) the `state_updates` sender used to broadcast, (c) where config is loaded at startup. Identify the exact handle/sender names.

- [ ] **Step 2: Add an integration test** at the server layer if a seam exists (e.g. a function that, given a mode + config, returns the theme to apply). If no clean seam, add a small pure helper `resolve_auto_theme(mode, &OpensessionsConfig) -> String` in `runtime-rs` and unit-test it:

```rust
#[test]
fn resolve_auto_theme_uses_config_with_defaults() {
    let mut c = OpensessionsConfig::default();
    assert_eq!(resolve_auto_theme(SystemAppearance::Dark, &c), "catppuccin-mocha");
    assert_eq!(resolve_auto_theme(SystemAppearance::Light, &c), "catppuccin-latte");
    c.dark_theme = Some("tokyo-night-storm".into());
    assert_eq!(resolve_auto_theme(SystemAppearance::Dark, &c), "tokyo-night-storm");
}
```

Implement `resolve_auto_theme` in `system_theme.rs` (keeps the policy testable; the server just calls it).

- [ ] **Step 3: Run → fail → implement helper → pass.**

- [ ] **Step 4: Wire in `lib.rs`:** after the state source + `state_updates` sender exist and config is loaded, add (using the real names found in Step 1):

```rust
    // macOS: follow system appearance and auto-switch themes.
    if startup_config.auto_theme_follows_system.unwrap_or(false) {
        let source = source.clone();              // Arc to the state source
        let updates = state_updates.clone();       // broadcast sender
        let _appearance_watcher = opensessions_runtime::system_theme::watch_mac_system_appearance(
            move |mode| {
                let cfg = opensessions_runtime::config::load_config_from_home(&home_dir());
                let desired = opensessions_runtime::system_theme::resolve_auto_theme(mode, &cfg);
                let mut guard = source.theme.lock().unwrap();
                if guard.as_deref() != Some(desired.as_str()) {
                    *guard = Some(desired);
                    drop(guard);
                    let _ = updates.send(source.snapshot_json());
                }
            },
            Some(60_000),
        );
        // keep the handle alive for the server's lifetime
        std::mem::forget(_appearance_watcher); // or store on the server handle struct
    }
```

Match the real field visibility (`theme` is currently private to the struct — expose a method like `set_theme_and_snapshot(&self, name: String) -> Option<String>` on `ReadOnlyMuxStateSource` if direct field access isn't available, and call that instead). Prefer adding the method over making the field `pub`.

- [ ] **Step 5: Build + run** `cargo build -p opensessions-server`; `cargo test --workspace` green.
- [ ] **Step 6: Commit** `feat(server): auto-switch theme on macOS appearance change`.

### Task D5: Manual-override-per-appearance persistence on `set-theme`

**Files:** Modify `apps/server-rs/src/lib.rs` (`set-theme` handler ~554)

Parity target (TS): when auto-follow is active, a manual `set-theme` persists to the appearance-specific slot (`darkTheme` if current mode dark, `lightTheme` if light, else `theme`) so the next appearance poll honors it instead of clobbering it. When not auto-following, persist `theme` as today.

- [ ] **Step 1:** Confirm whether the current Rust `set-theme` persists to config at all (the handler at :554 only sets the mutex). If it does not persist, this task also *adds* persistence (matching TS, which calls `saveConfig`). Decide based on the read: the goal is parity with TS `saveConfig` behavior.

- [ ] **Step 2: Add a pure helper** (testable) in `system_theme.rs`:

```rust
/// Which config field a manual theme choice should persist to.
pub enum ThemePersistSlot { Theme, DarkTheme, LightTheme }

pub fn manual_persist_slot(auto_following: bool, current_mode: Option<SystemAppearance>) -> ThemePersistSlot {
    if !auto_following { return ThemePersistSlot::Theme; }
    match current_mode {
        Some(SystemAppearance::Dark) => ThemePersistSlot::DarkTheme,
        Some(SystemAppearance::Light) => ThemePersistSlot::LightTheme,
        None => ThemePersistSlot::Theme,
    }
}
```

Unit-test the three branches in `tests/system_theme.rs`.

- [ ] **Step 3:** Track `auto_following: bool` and the latest observed `current_mode: Mutex<Option<SystemAppearance>>` on the server state (set `current_mode` inside the D4 watcher closure before computing the theme). In the `set-theme` handler, after setting the mutex, compute the slot via `manual_persist_slot(..)` and call `save_config_to_home(&home, OpensessionsConfig{ <slot>: Some(name), ..Default::default() })`.

- [ ] **Step 4:** `cargo test --workspace` green; manual reasoning check that a dark-mode `set-theme` writes `darkTheme`.
- [ ] **Step 5: Commit** `feat(server): persist manual theme override per system appearance`.

---

## Phase E — Server singleton guard (item 7)

### Task E1: Clean "already running" handling on bind conflict

**Files:** Modify `apps/server-rs/src/lib.rs` (bind path ~1972, PID-file write ~1983); Test: server test crate

Parity target (TS): on a port conflict, instead of a bare I/O error, probe the PID file; if a live opensessions server owns the port, print "another server is already running (pid N). Exiting." and exit cleanly (success), rather than erroring.

- [ ] **Step 1:** Read the bind path + PID-file write to learn the PID-file path + format. Determine if the launcher already pre-checks for a running server (architecture note suggests `ensureServer()` historically did). If the launcher already prevents double-launch, scope this to: convert the `AddrInUse` error into a clean, logged early-return rather than a propagated error.

- [ ] **Step 2: Failing test:** add a unit test for a pure helper `is_addr_in_use(&io::Error) -> bool` (and, if a PID file is used, `pid_is_alive(pid) -> bool` via `kill(pid, 0)`), then a test asserting the bind path maps an `AddrInUse` to the "already running" outcome rather than a hard error. Match the server crate's existing test seams.

- [ ] **Step 3: Implement** — before `TcpListener::bind`, attempt to read the PID file; if present and `pid_is_alive`, log "another server is already running (pid {pid}). Exiting." and return the clean early-exit variant. Wrap the `bind` so an `ErrorKind::AddrInUse` produces the same clean outcome (covers the race where the PID file isn't written yet).

- [ ] **Step 4:** `cargo test --workspace` green.
- [ ] **Step 5: Commit** `fix(server): clean singleton guard on port conflict`.

---

## Phase F — Benchmark (maximal-parity bar)

### Task F1: Build both, measure, record

**Files:** Create `docs/explanation/perf-rust-port-2026-05.md`

- [ ] **Step 1: Build baseline + after binaries.**

```bash
# after (current rust-port HEAD)
cargo build --profile release-dev
cp ~/rust-target/release-dev/opensessions-server /tmp/os-after-server 2>/dev/null || cp target/release-dev/opensessions-server /tmp/os-after-server
# baseline (clean origin/main) in a throwaway checkout
git worktree add /tmp/os-baseline origin/main && (cd /tmp/os-baseline && cargo build --profile release-dev)
```

- [ ] **Step 2: Measure** each metric below for baseline vs after, same tmux + ambient agent load:
  - **Idle CPU% + RSS:** `ps -o %cpu,rss -p <server_pid>` sampled @5s over 30s, 4 TUI clients attached.
  - **Theme-detection spawns:** with `autoThemeFollowsSystem:true`, over a 10-min idle window confirm ~0 `defaults` spawns (watch is push-based). `sudo fs_usage -w -f exec | grep defaults` or a dtrace count; pass = no periodic spawn.
  - **Session-switch latency:** timestamp deltas in the server debug log across 10 switches.
  - **Broadcast dedup:** confirm identical states are suppressed (already implemented) — count broadcasts vs `/api/agent-event` posts over 30s of repeated same-status pings.
  - **Singleton:** start the server twice; second prints "already running" and exits clean.

- [ ] **Step 3: Record** results in `docs/explanation/perf-rust-port-2026-05.md` (baseline vs after table). **Regression rule:** if any axis is worse on `after`, open a follow-up task to port the corresponding fix; otherwise note parity/no-regression.

- [ ] **Step 4: Cleanup + commit.**

```bash
git worktree remove /tmp/os-baseline --force
git add docs/explanation/perf-rust-port-2026-05.md
git commit -m "docs: rust-port benchmark results (parity verified)" -m "Co-authored-by: Isaac"
```

---

## Phase G — Install locally + end-to-end smoke

### Task G1: Build release, swap the live plugin, verify

- [ ] **Step 1: Build release** in the worktree: `cargo build --release` (full `opt-level=z, lto=fat` profile).
- [ ] **Step 2: Point the live plugin at `rust-port`.** In `~/.tmux/plugins/opensessions`: `git fetch && git checkout rust-port && cargo build --release`. (The previous TS branch remains in git for rollback.)
- [ ] **Step 3: Reload + smoke.** `tmux source-file ~/.tmux.conf`; open sidebar `prefix o → s`. Verify: sidebar renders; `tokyo-night-storm` / `tango-adapted` selectable; with `autoThemeFollowsSystem:true` in `~/.config/opensessions/config.json`, flip macOS Appearance and confirm the theme switches.
- [ ] **Step 4: Ella bridge e2e.** With the SSH reverse tunnel up, trigger a remote Hermes event (or `curl -X POST http://127.0.0.1:7391/api/agent-event -H 'content-type: application/json' -d '{"agent":"hermes","status":"running","ts":<ms>,"tmuxSession":"<session>"}'`) and confirm the pill appears in the sidebar.
- [ ] **Step 5:** Report results to the user. (No commit — this is environment config.)

---

## Phase H — PR split prep (upstream + fork)

### Task H1: Slice upstreamable commits onto a clean feature branch

- [ ] **Step 1:** Create `git checkout -b upstream/auto-theme-and-themes origin/main` and cherry-pick the portable commits: themes (A1), unseen hard-prune (B1), opencode eviction (C2 if done), system_theme module + config + server wiring + override (D1-D5), singleton guard (E1). Exclude: the spec/plan docs and the benchmark note (fork-local) unless desired.
- [ ] **Step 2:** `cargo test --workspace` green on the clean branch; `cargo clippy --workspace` clean; `cargo fmt --check`.
- [ ] **Step 3:** (Optional) refresh stale `AGENTS.md` to describe the Rust architecture — separate commit, clearly valuable upstream.
- [ ] **Step 4:** Push to fork; open PR to `Ataraxy-Labs/opensessions` **only after the user confirms** they're happy with the local install. Keep fork-only commits (spec, plan, benchmark note) on `rust-port`.

---

## Self-review

- **Spec coverage:** every spec inventory row maps to a task or an explicit SKIP with rationale (items 4,8,9,10,11,12 skipped per investigation verdicts; 1→D, 2/3→A, 5→B, 6→C, 7→E, 14→G4, benchmark→F, install→G, PR split→H). ✓
- **Placeholders:** the `[verify]`/"read Step 1" actions are genuine investigation steps with explicit decision criteria, not hand-waving; concrete code is given for every implementation step whose shape is known. Phase C/D4/D5/E contain "read the real seam then match names" because the exact private symbols must be confirmed against current code before editing — each specifies precisely what to find and the code to add once found.
- **Type consistency:** `SystemAppearance`, `theme_for_system_mode`, `read_mac_system_appearance`, `watch_mac_system_appearance`, `AppearanceWatcher`, `resolve_auto_theme`, `manual_persist_slot`, `ThemePersistSlot` are used consistently across D2-D5; config field names (`auto_theme_follows_system`/`dark_theme`/`light_theme`) consistent D1→D4→D5; `TERMINAL_HARD_PRUNE_MS` consistent in B1.
