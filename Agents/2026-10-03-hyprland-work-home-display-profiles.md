# Hyprland Work/Home Display Profiles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the existing dotfiles repository with a safe, automatic, and reversible Work/Home display-profile system for Hyprland, including a dedicated popup menu for selecting Auto, Home, or Work mode.

**Architecture:** A single Bash controller owns all display and Waybar state. A lightweight Hyprland IPC watcher calls that controller when `DP-2` is connected or disconnected, and a separate popup-menu script exposes the same controller for interactive selection. Two Waybar configuration files select the correct output, while manual profile and menu keybindings provide reliable fallbacks. Every operation must fail safely to the laptop-only profile.

**Tech Stack:** Bash, Hyprland `hyprctl`, Hyprland socket2 IPC, `jq`, `socat`, `flock`, the launcher already used by the dotfiles (`rofi`, `wofi`, `fuzzel`, or `tofi`), Waybar JSONC, `notify-send`, shell test fixtures.

**Spec:** Approved in-chat design on 2026-10-03; all requirements are restated below so this document is self-contained.

## Global Constraints

- Do not modify NVIDIA drivers, kernel packages, initramfs, GRUB, or boot parameters.
- Do not install another display-profile manager such as `kanshi`.
- Implement this feature inside the existing dotfiles repository; do not create a second repository.
- Work on a dedicated branch such as `feat/hypr-display-profiles` when the existing dotfiles repository is clean enough to branch safely.
- Reuse the popup launcher already configured by the dotfiles. Do not install a new launcher without user approval.
- Preserve the current Waybar modules, CSS, top bar, and bottom bar.
- Preserve unrelated user changes. Inspect files before editing and patch only the required lines.
- Back up every existing file before changing it.
- Use Bash explicitly: `#!/usr/bin/env bash` and `set -Eeuo pipefail`.
- Treat `eDP-1` as the always-safe laptop output.
- Never disable `eDP-1` until another usable monitor has been confirmed active. In the approved profiles below, `eDP-1` remains active in both modes.
- If `DP-2` is missing, disconnected, or fails to activate, fall back to Work mode automatically.
- Prevent concurrent profile changes with `flock` and debounce rapid monitor events.
- Do not leave multiple Waybar processes running.
- Make every change reversible and document the rollback procedure.
- Stop immediately on failed verification. Do not continue by guessing.

## Approved Display Profiles

| Profile | Output | Mode | Position | Scale | Waybar |
|---|---|---:|---:|---:|---|
| Work | `eDP-1` | `1920x1080@60` | `0x0` | `1.5` | Top and bottom bars on `eDP-1` |
| Work | `DP-2` | Disabled | — | — | None |
| Home | `eDP-1` | `1920x1080@60` | `0x0` | `1.5` | None |
| Home | `DP-2` | `1920x1080@119.88` | `1280x0` | `1` | Top and bottom bars on `DP-2` |

Notes:

- The external panel is physically on the right.
- `1280x0` is intentional because `eDP-1` has a logical width of 1280 at scale `1.5`.
- The HDMI path supports approximately 120 Hz, so use the advertised mode `1920x1080@119.88`, not 180 Hz and not a guessed `120` value.
- In Home mode, focus `DP-2` after the layout is applied so newly opened applications start on the external display when Hyprland permits it.
- Existing mouse and keyboard handling already works globally; do not create device-specific input profiles.

## Review Focus

Review the implementation especially for these failure modes:

1. Home mode requested while `DP-2` is absent: must return to a usable Work profile.
2. Rapid connect/disconnect events: must not create racing `hyprctl` calls or duplicate Waybar processes.
3. Hyprland restart or socket replacement: watcher must reconnect without user intervention.
4. Missing `jq`, `socat`, `flock`, or `hyprctl`: must produce a clear diagnostic and avoid partial changes.
5. Waybar restart failure: displays must remain usable and the failure must be logged.
6. Incorrect sourced config path or occupied keybinding: Codex must report the conflict instead of overwriting it silently.
7. Menu cancellation, unknown launcher output, or an already-active profile: must not execute an unintended profile command.

## Planned File Map

Codex must verify the exact existing autostart and keybinding paths before editing them.

| File | Action | Responsibility |
|---|---|---|
| `~/.config/hypr/scripts/DisplayProfile.sh` | Create | Apply `auto`, `home`, or `work`; control monitors and Waybar |
| `~/.config/hypr/scripts/DisplayProfileMenu.sh` | Create | Show a dedicated popup menu and dispatch a constant profile choice |
| `~/.config/hypr/scripts/MonitorWatcher.sh` | Create | Listen for Hyprland monitor events and invoke automatic selection |
| `~/.config/hypr/scripts/tests/test-display-profile.sh` | Create | Test profile decisions and generated commands with fake dependencies |
| `~/.config/hypr/scripts/tests/test-display-profile-menu.sh` | Create | Test menu choices, cancellation, current-state display, and safe dispatch |
| `~/.config/hypr/scripts/tests/test-monitor-watcher.sh` | Create | Test event filtering, debounce, and reconnect behavior |
| `~/.config/hypr/waybar/config-home` | Create from active config | Two bars restricted to `DP-2` |
| `~/.config/hypr/waybar/config-work` | Create from active config | Two bars restricted to `eDP-1` |
| Existing sourced Hyprland autostart file | Modify after discovery | Start exactly one watcher per Hyprland session |
| Existing sourced Hyprland keybinding file | Modify after discovery | Add manual Work/Home bindings and a dedicated menu binding |
| `~/.config/hypr/scripts/README-display-profiles.md` | Create | Usage, verification, troubleshooting, and rollback |

## Codex Execution Rules

- Read this entire plan before changing anything.
- Execute tasks in order and mark each checkbox only after its verification passes.
- Inspect the live configuration and resolve symlinks before editing.
- Locate the actual dotfiles repository root and edit repository-owned source files rather than generated targets or symlink destinations. Do not create a new repository.
- Use patch-based edits; do not replace entire user-owned config files unnecessarily.
- Run the smallest relevant test after every change, then run the full verification suite.
- If the discovered monitor names or supported modes differ from this plan, stop and ask the user before adapting the values.
- If the discovered autostart/keybinding structure differs, explain the exact difference and propose the smallest integration change.
- Do not initialize a Git repository. If the dotfiles are already version-controlled, commit only relevant files and never include unrelated changes.
- Report completion with command output proving the resulting monitor state, Waybar process count, and automatic switching behavior.

---

## Task 1: Discover the Live Configuration and Create Backups

**Files:**

- Inspect: `~/.config/hypr/`
- Inspect: `~/.config/hypr/waybar/config`
- Inspect: every config file sourced by the active Hyprland config
- Create: `~/.config/hypr/backups/display-profiles-<timestamp>/`

- [ ] **Step 1: Confirm the active Hyprland entry point and all sourced files**

Run:

```bash
hyprctl version
rg -n '^\s*source\s*=|^\s*exec-once\s*=|^\s*bind[a-z]*\s*=' ~/.config/hypr
readlink -f ~/.config/hypr/hyprland.conf 2>/dev/null || true
readlink -f ~/.config/hypr/hyprland.lua 2>/dev/null || true
```

Expected: Codex identifies the exact active autostart file, exact keybinding file, and whether any relevant path is a symlink. If the entry point is ambiguous, stop and ask the user.

- [ ] **Step 2: Record the current monitor and Waybar baseline**

Run:

```bash
hyprctl -j monitors | jq .
hyprctl monitors
pgrep -af waybar || true
hyprctl monitors | rg 'Monitor|reserved|scale|at'
```

Expected: `eDP-1` and, when connected, `DP-2` are visible. Confirm that `DP-2` advertises `1920x1080@119.88` or an equivalent exact mode before using it.

- [ ] **Step 3: Check dependencies without changing the system**

Run:

```bash
for command in bash hyprctl jq socat flock waybar pgrep pkill; do
  command -v "$command" || printf 'MISSING: %s\n' "$command"
done
command -v notify-send || printf 'OPTIONAL MISSING: notify-send\n'
```

Expected: required commands exist. If a required command is missing, report the package needed and wait for user approval before installing anything.

- [ ] **Step 4: Check for binding and process conflicts**

Run:

```bash
rg -n 'SUPER.*ALT.*[HW]|MonitorWatcher|DisplayProfile|waybar' ~/.config/hypr
pgrep -af 'MonitorWatcher|DisplayProfile|waybar' || true
```

Expected: no collision for `SUPER+ALT+H` and `SUPER+ALT+W`, or the collision is reported to the user before proceeding.

- [ ] **Step 5: Create a timestamped backup**

Back up at minimum:

- the active Hyprland entry point;
- the discovered autostart file;
- the discovered keybinding file;
- `~/.config/hypr/waybar/config`;
- any existing files at the planned script/config paths.

After copying, list the backup contents and store the final backup path for the README.

- [ ] **Step 6: Check repository state if applicable**

Run from the resolved dotfiles root:

```bash
git status --short
```

Expected: Codex records pre-existing changes and avoids them. If this configuration is unexpectedly not backed by the existing dotfiles repository, stop and ask the user for its path rather than initializing a new repository.

- [ ] **Step 7: Create a feature branch when safe**

If the repository state permits it, run:

```bash
git switch -c feat/hypr-display-profiles
```

If the branch already exists, switch to it. If unrelated uncommitted changes make switching unsafe, stop and ask the user how to preserve them.

---

## Task 2: Define and Test the Profile Controller Contract

**Files:**

- Create: `~/.config/hypr/scripts/tests/test-display-profile.sh`
- Create test fixtures under: `~/.config/hypr/scripts/tests/fixtures/`

- [ ] **Step 1: Define the public command interface in the test**

Required interface:

```text
DisplayProfile.sh auto|home|work [--dry-run]
```

Required exit behavior:

- `0`: requested profile was applied successfully, or `auto` selected and applied a valid profile.
- non-zero: invalid arguments, missing hard dependency, or unrecoverable application failure.
- `home` without `DP-2`: apply Work mode safely, emit a fallback message, and return success only if Work mode succeeds.

- [ ] **Step 2: Build fake monitor fixtures**

Create representative JSON fixtures for:

1. laptop only;
2. laptop plus connected `DP-2` supporting `119.88` Hz;
3. `DP-2` listed but not active;
4. malformed or empty monitor output.

Do not depend on the physical monitor being connected to run unit tests.

- [ ] **Step 3: Add failing tests for command generation**

Tests must assert:

- Work activates `eDP-1` at `1920x1080@60`, position `0x0`, scale `1.5`.
- Work disables `DP-2` only after the laptop output command succeeds.
- Home activates `eDP-1` first.
- Home activates `DP-2` at exactly `1920x1080@119.88`, position `1280x0`, scale `1`.
- Home verifies `DP-2` before restarting Waybar or moving focus.
- `auto` selects Home only when `DP-2` is connected and usable.
- `auto` selects Work when `DP-2` is absent.
- explicit Home mode safely falls back to Work when `DP-2` is absent.
- invalid arguments fail with a usage message.
- `--dry-run` emits intended commands without changing the current session.
- the selected Waybar config is `config-home` or `config-work` as appropriate.

- [ ] **Step 4: Run the test and confirm the expected failure**

Run:

```bash
bash ~/.config/hypr/scripts/tests/test-display-profile.sh
```

Expected: failure because `DisplayProfile.sh` has not been implemented yet. If it passes unexpectedly, fix the test so it proves behavior rather than merely checking file existence.

---

## Task 3: Implement the Profile Controller

**Files:**

- Create: `~/.config/hypr/scripts/DisplayProfile.sh`
- Test: `~/.config/hypr/scripts/tests/test-display-profile.sh`

- [ ] **Step 1: Implement argument parsing and dependency validation**

Accept only `auto`, `home`, or `work`, followed optionally by `--dry-run`. Validate required commands before applying any change. Treat `notify-send` as optional.

- [ ] **Step 2: Implement structured helper functions**

Use small functions with clear responsibilities, such as:

```text
log
notify
require_command
run_cmd
is_dp2_connected
verify_output_active
apply_work
apply_home
restart_waybar
focus_home_output
```

Tests should be able to substitute fake `hyprctl`, `jq`, Waybar, and process-control commands through `PATH` or environment variables.

- [ ] **Step 3: Add process locking**

Use `flock` on a per-user runtime lock under `${XDG_RUNTIME_DIR:-/tmp}`. A second invocation should wait briefly or exit cleanly; it must not interleave display and Waybar operations.

- [ ] **Step 4: Implement Work mode in a safe order**

Required order:

1. enable/configure `eDP-1`;
2. verify `eDP-1` is active;
3. disable `DP-2` if present;
4. restart Waybar with `config-work` and the existing shared stylesheet;
5. log and optionally notify success.

- [ ] **Step 5: Implement Home mode in a safe order**

Required order:

1. confirm `DP-2` is connected and advertises the approved mode;
2. enable/configure `eDP-1`;
3. enable/configure `DP-2` at `1920x1080@119.88,1280x0,1`;
4. verify `DP-2` is active with the intended refresh rate and placement;
5. restart Waybar with `config-home` and the existing shared stylesheet;
6. focus `DP-2` using the appropriate Hyprland dispatcher;
7. log and optionally notify success.

If any step before verification fails, call `apply_work` and report the fallback.

- [ ] **Step 6: Implement Waybar replacement without duplication**

Use the existing command shape:

```bash
waybar -c "$HOME/.config/hypr/waybar/config-<profile>" \
  -s "$HOME/.config/hypr/waybar/style.css"
```

Terminate only the current user's Waybar process, wait for it to exit, launch exactly one replacement process, and verify it remains alive. Avoid broad root-level process termination.

- [ ] **Step 7: Implement `auto` and `--dry-run`**

- `auto`: choose Home only if `DP-2` is genuinely connected and usable; otherwise choose Work.
- `--dry-run`: perform detection and print the commands in execution order, but do not call `hyprctl`, stop Waybar, launch Waybar, focus a monitor, or send a notification.

- [ ] **Step 8: Run syntax checks and unit tests**

Run:

```bash
bash -n ~/.config/hypr/scripts/DisplayProfile.sh
bash -n ~/.config/hypr/scripts/tests/test-display-profile.sh
bash ~/.config/hypr/scripts/tests/test-display-profile.sh
command -v shellcheck >/dev/null && shellcheck ~/.config/hypr/scripts/DisplayProfile.sh ~/.config/hypr/scripts/tests/test-display-profile.sh || true
```

Expected: syntax and behavior tests pass. Review every `shellcheck` warning; do not suppress warnings without documenting why.

- [ ] **Step 9: Commit if and only if the dotfiles are already version-controlled**

Suggested commit:

```bash
git add -- ~/.config/hypr/scripts/DisplayProfile.sh ~/.config/hypr/scripts/tests
git commit -m "feat(hypr): add safe display profile controller"
```

Adapt paths to the repository root. Do not commit unrelated files.

---

## Task 4: Create Output-Specific Waybar Configurations

**Files:**

- Source: `~/.config/hypr/waybar/config`
- Create: `~/.config/hypr/waybar/config-home`
- Create: `~/.config/hypr/waybar/config-work`

- [ ] **Step 1: Copy the active dual-bar config twice**

Preserve the JSONC array, includes, comments, modules, margins, and the current top/bottom bar design.

- [ ] **Step 2: Restrict both Home bars to `DP-2`**

Add this property to each of the two bar objects in `config-home`:

```jsonc
"output": "DP-2",
```

- [ ] **Step 3: Restrict both Work bars to `eDP-1`**

Add this property to each of the two bar objects in `config-work`:

```jsonc
"output": "eDP-1",
```

- [ ] **Step 4: Add static assertions**

Run:

```bash
test "$(rg -c '"output": "DP-2"' ~/.config/hypr/waybar/config-home)" -eq 2
test "$(rg -c '"output": "eDP-1"' ~/.config/hypr/waybar/config-work)" -eq 2
diff -u ~/.config/hypr/waybar/config-home ~/.config/hypr/waybar/config-work || true
```

Expected: exactly two output selectors in each file, and the meaningful diff between the two files is only the output name unless a documented compatibility adjustment is required.

- [ ] **Step 5: Test each config in its matching live profile**

First use the controller with `--dry-run`. Then, only after reviewing the output, apply the matching profile and inspect Waybar logs.

Expected:

- Work: one top bar and one bottom bar on `eDP-1` only.
- Home: one top bar and one bottom bar on `DP-2` only.
- `pgrep -af waybar` reports exactly one Waybar process.
- Modules such as weather, tray, audio, and workspaces are not duplicated.

- [ ] **Step 6: Re-run controller tests**

Run:

```bash
bash ~/.config/hypr/scripts/tests/test-display-profile.sh
```

- [ ] **Step 7: Commit if applicable**

Suggested commit:

```bash
git add -- ~/.config/hypr/waybar/config-home ~/.config/hypr/waybar/config-work
git commit -m "feat(waybar): add work and home output configs"
```

---

## Task 5: Implement and Test the Monitor Event Watcher

**Files:**

- Create: `~/.config/hypr/scripts/MonitorWatcher.sh`
- Create: `~/.config/hypr/scripts/tests/test-monitor-watcher.sh`

- [ ] **Step 1: Write failing watcher tests with a fake event stream**

The fake stream must cover:

- `monitoradded>>DP-2`;
- `monitorremoved>>DP-2`;
- v2 monitor-added/removed variants emitted by the installed Hyprland version, if present;
- unrelated monitor and window events;
- several `DP-2` events arriving rapidly;
- stream closure followed by reconnection.

Tests must prove that unrelated events do nothing and a burst produces a single debounced `DisplayProfile.sh auto` call.

- [ ] **Step 2: Run the test and confirm it fails before implementation**

Run:

```bash
bash ~/.config/hypr/scripts/tests/test-monitor-watcher.sh
```

- [ ] **Step 3: Implement startup reconciliation**

On launch, call:

```bash
~/.config/hypr/scripts/DisplayProfile.sh auto
```

This makes the correct profile active even if the monitor was already connected before Hyprland started.

- [ ] **Step 4: Implement socket2 monitoring**

Build the socket path from:

```text
$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock
```

Connect using `socat -U - UNIX-CONNECT:<socket>`. React only to relevant add/remove events for `DP-2`.

- [ ] **Step 5: Implement debounce, singleton behavior, and reconnect**

- use a watcher-specific `flock` so only one watcher exists;
- coalesce rapid DP-2 events with a short delay;
- after the event stream closes, wait briefly and reconnect;
- avoid a tight failure loop when the socket is temporarily unavailable;
- log reconnection attempts without spamming notifications.

- [ ] **Step 6: Run syntax and behavior tests**

Run:

```bash
bash -n ~/.config/hypr/scripts/MonitorWatcher.sh
bash -n ~/.config/hypr/scripts/tests/test-monitor-watcher.sh
bash ~/.config/hypr/scripts/tests/test-monitor-watcher.sh
command -v shellcheck >/dev/null && shellcheck ~/.config/hypr/scripts/MonitorWatcher.sh ~/.config/hypr/scripts/tests/test-monitor-watcher.sh || true
```

- [ ] **Step 7: Commit if applicable**

Suggested commit:

```bash
git add -- ~/.config/hypr/scripts/MonitorWatcher.sh ~/.config/hypr/scripts/tests/test-monitor-watcher.sh
git commit -m "feat(hypr): switch display profiles on monitor events"
```

---

## Task 6: Implement the Dedicated Display Profile Menu

**Files:**

- Create: `~/.config/hypr/scripts/DisplayProfileMenu.sh`
- Create: `~/.config/hypr/scripts/tests/test-display-profile-menu.sh`
- Consume: `~/.config/hypr/scripts/DisplayProfile.sh`

**Interface:**

```text
DisplayProfileMenu.sh [--dry-run]
```

The menu must contain these choices in this order:

```text
Automatic
Home — Dual monitors
Work — Laptop only
Cancel
```

The prompt or message must also show the detected current state: `Home`, `Work`, or `Unknown`. The current-state label is informational and must not be parsed as a shell command.

- [ ] **Step 1: Discover and record the existing launcher command**

Run:

```bash
rg -n 'rofi|wofi|fuzzel|tofi' ~/.config/hypr
for command in rofi wofi fuzzel tofi; do command -v "$command" && printf 'FOUND: %s\n' "$command"; done
```

Expected: identify the launcher already used by the dotfiles and preserve its theme arguments where practical. If no supported launcher is configured, stop and ask the user before installing or choosing one.

- [ ] **Step 2: Write failing menu tests**

Use a fake launcher whose selected line is controlled by the test. Tests must assert:

- `Automatic` invokes `DisplayProfile.sh auto` exactly once;
- `Home — Dual monitors` invokes `DisplayProfile.sh home` exactly once;
- `Work — Laptop only` invokes `DisplayProfile.sh work` exactly once;
- `Cancel`, an empty result, launcher failure, or an unknown string makes no profile call and exits cleanly;
- no selection is passed to `eval` or executed as shell text;
- current state is reported as Home only when both approved outputs are active with `DP-2` in the expected Home layout;
- current state is Work when only `eDP-1` is active;
- `--dry-run` displays or prints the mapped controller command but does not execute it.

- [ ] **Step 3: Run the menu test and verify the expected failure**

Run:

```bash
bash ~/.config/hypr/scripts/tests/test-display-profile-menu.sh
```

Expected: failure because the menu script does not exist yet.

- [ ] **Step 4: Implement constant choice mapping**

Implement the selected launcher command without `eval`. Map only the four exact constant labels listed above. Dispatch valid selections to `DisplayProfile.sh`; treat every other value as cancellation.

- [ ] **Step 5: Implement current-profile detection**

Use `hyprctl -j monitors` plus `jq`. Detection is informational only:

- Home: active `eDP-1` and active `DP-2` matching the approved Home arrangement;
- Work: active `eDP-1` with no active `DP-2`;
- Unknown: every other arrangement.

Do not silently change the layout merely because the menu opens.

- [ ] **Step 6: Match the existing launcher style**

Reuse the launcher and theme flags already used by the dotfiles. Give the popup a distinct prompt such as `Display profile`. Do not modify the user's general application menu.

- [ ] **Step 7: Run syntax, behavior, and static-safety checks**

Run:

```bash
bash -n ~/.config/hypr/scripts/DisplayProfileMenu.sh
bash -n ~/.config/hypr/scripts/tests/test-display-profile-menu.sh
bash ~/.config/hypr/scripts/tests/test-display-profile-menu.sh
! rg -n '\beval\b' ~/.config/hypr/scripts/DisplayProfileMenu.sh
command -v shellcheck >/dev/null && shellcheck ~/.config/hypr/scripts/DisplayProfileMenu.sh ~/.config/hypr/scripts/tests/test-display-profile-menu.sh || true
```

Expected: tests pass and the implementation contains no `eval`.

- [ ] **Step 8: Commit if applicable**

Suggested commit:

```bash
git add -- ~/.config/hypr/scripts/DisplayProfileMenu.sh ~/.config/hypr/scripts/tests/test-display-profile-menu.sh
git commit -m "feat(hypr): add display profile popup menu"
```

---

## Task 7: Integrate Autostart and Manual Fallback Keybindings

**Files:**

- Modify: the exact sourced autostart file discovered in Task 1
- Modify: the exact sourced keybinding file discovered in Task 1

- [ ] **Step 1: Reconfirm the files are active and unchanged since backup**

Compare them with the copies in the timestamped backup. If they changed during implementation, inspect the new changes before patching.

- [ ] **Step 2: Add exactly one watcher autostart entry**

Use the syntax already present in the user's config, equivalent to:

```ini
exec-once = $HOME/.config/hypr/scripts/MonitorWatcher.sh
```

Do not launch `DisplayProfile.sh` separately at startup because the watcher already performs startup reconciliation.

- [ ] **Step 3: Add manual fallback bindings only after checking conflicts**

Use the config's established binding syntax, equivalent to:

```ini
bind = SUPER ALT, H, exec, $HOME/.config/hypr/scripts/DisplayProfile.sh home
bind = SUPER ALT, W, exec, $HOME/.config/hypr/scripts/DisplayProfile.sh work
bind = SUPER ALT, P, exec, $HOME/.config/hypr/scripts/DisplayProfileMenu.sh
```

Approved mapping:

- `Super+Alt+H`: Home profile.
- `Super+Alt+W`: Work profile.
- `Super+Alt+P`: open the dedicated display-profile menu.

If either combination is already used, stop and ask the user to select another binding.

- [ ] **Step 4: Reload Hyprland and inspect errors**

Run:

```bash
hyprctl reload
hyprctl configerrors
pgrep -af MonitorWatcher || true
```

Expected: no new configuration errors. Because `exec-once` does not necessarily run on reload, start the watcher manually for the current session if it is not running; rely on `exec-once` for the next login.

- [ ] **Step 5: Verify singleton behavior**

Attempt to start the watcher a second time. Expected: the existing watcher remains the only active watcher.

- [ ] **Step 6: Test all three hotkeys**

Before pressing a hotkey, keep a terminal visible and have the rollback command ready. Verify:

- Work produces a usable laptop-only layout and Work Waybar.
- Home produces the approved dual-screen layout when `DP-2` is connected.
- Home safely falls back to Work when `DP-2` is disconnected.
- Menu opens independently of the application launcher, shows current state, and correctly dispatches Auto/Home/Work.

- [ ] **Step 7: Commit if applicable**

Suggested commit:

```bash
git add -- <resolved-autostart-file> <resolved-keybinding-file>
git commit -m "feat(hypr): integrate automatic display profiles"
```

---

## Task 8: Perform End-to-End Verification

**Files:**

- Test all files created or modified in Tasks 1–6

- [ ] **Step 1: Run all automated checks**

Run:

```bash
bash -n ~/.config/hypr/scripts/DisplayProfile.sh
bash -n ~/.config/hypr/scripts/MonitorWatcher.sh
bash ~/.config/hypr/scripts/tests/test-display-profile.sh
bash ~/.config/hypr/scripts/tests/test-display-profile-menu.sh
bash ~/.config/hypr/scripts/tests/test-monitor-watcher.sh
hyprctl configerrors
```

Expected: all commands succeed and Hyprland reports no new config errors.

- [ ] **Step 2: Verify Work mode with the external monitor disconnected**

Run Work mode, then collect:

```bash
hyprctl -j monitors | jq '[.[] | {name, width, height, refreshRate, x, y, scale, focused}]'
pgrep -af waybar
pgrep -af MonitorWatcher
```

Expected:

- `eDP-1`: active, 1920×1080 near 60 Hz, position `0x0`, scale `1.5`.
- `DP-2`: absent or disabled.
- top and bottom bars appear on `eDP-1` only.
- one Waybar process and one watcher process.

- [ ] **Step 3: Verify automatic Home transition**

Connect the external display and wait for the debounce period. Collect the same evidence.

Expected:

- `eDP-1`: left, scale `1.5`.
- `DP-2`: right at logical position `1280x0`, scale `1`, approximately `119.88` Hz.
- `DP-2` is focused after the transition.
- top and bottom bars appear only on `DP-2`.
- one Waybar process and one watcher process.

- [ ] **Step 4: Verify automatic Work transition**

Disconnect `DP-2`.

Expected: the session remains visible on `eDP-1`; Work Waybar starts automatically; no black screen and no duplicate Waybar.

- [ ] **Step 5: Test rapid reconnection**

Perform one controlled disconnect/reconnect cycle with only a brief interval. Do not repeatedly stress the physical connector.

Expected: final state matches the actual connection state; process counts remain one; no race-related errors appear in logs.

- [ ] **Step 6: Test manual fallback paths**

- With `DP-2` connected, invoke Work, then Home.
- With `DP-2` disconnected, invoke Home and confirm safe Work fallback.
- Invoke `auto` explicitly in both physical states.
- Open the dedicated menu in both physical states and select each valid profile.
- Cancel the menu and confirm no display or Waybar command runs.

- [ ] **Step 7: Check for regressions**

Confirm:

- workspaces move and focus normally;
- mouse crosses the screen boundary in the expected direction;
- keyboard and mouse remain available;
- audio, network, tray, weather, and workspace modules work;
- no duplicate weather requests or duplicate tray icons;
- idle CPU usage remains reasonable after the watcher settles;
- logout/login starts the watcher automatically and selects the correct profile.

- [ ] **Step 8: Capture verification evidence**

Save concise output showing:

- both final monitor states;
- exact Waybar and watcher process counts;
- successful automated tests;
- `hyprctl configerrors` output;
- any non-fatal limitations discovered.

---

## Task 9: Document Operation and Rollback

**Files:**

- Create: `~/.config/hypr/scripts/README-display-profiles.md`

- [ ] **Step 1: Document normal usage**

Include:

- automatic behavior on `DP-2` connect/disconnect;
- `Super+Alt+H` for Home;
- `Super+Alt+W` for Work;
- `Super+Alt+P` for the dedicated profile menu;
- menu choices, current-state indicator, and cancellation behavior;
- direct commands for `auto`, `home`, `work`, and `--dry-run`;
- approved monitor geometry and refresh-rate rationale.

- [ ] **Step 2: Document troubleshooting**

Include commands for:

- viewing monitor state;
- checking Waybar and watcher process counts;
- running both test scripts;
- viewing watcher/controller logs;
- checking the socket path;
- manually restoring Work mode.

The first recovery command should be:

```bash
~/.config/hypr/scripts/DisplayProfile.sh work
```

- [ ] **Step 3: Document exact rollback using the backup from Task 1**

Rollback must explain how to:

1. stop the watcher;
2. restore the original autostart, keybinding, and Waybar files from the recorded backup directory;
3. remove only the newly created display-profile files;
4. reload Hyprland;
5. restart the original Waybar command;
6. verify the restored state.

Do not include destructive wildcards. Use exact paths from the recorded backup.

- [ ] **Step 4: Run a documentation review**

Follow the README exactly in a dry run. Confirm every command and path matches the installed implementation.

- [ ] **Step 5: Make a final documentation commit if applicable**

Suggested commit:

```bash
git add -- ~/.config/hypr/scripts/README-display-profiles.md
git commit -m "docs(hypr): document display profiles and rollback"
```

---

## Final Acceptance Criteria

The implementation is complete only when all of the following are true:

- [ ] Work mode is stable with only `eDP-1`, scale `1.5`, and two Waybar bars on the laptop display.
- [ ] Home mode is stable with `eDP-1` on the left and `DP-2` on the right at `119.88` Hz.
- [ ] Home Waybar appears only on `DP-2`; Work Waybar appears only on `eDP-1`.
- [ ] Connecting and disconnecting `DP-2` switches profiles automatically.
- [ ] Manual Home and Work hotkeys function as fallbacks.
- [ ] The dedicated popup menu shows current state and safely selects Auto, Home, or Work.
- [ ] Cancelling or closing the menu changes nothing.
- [ ] Requesting Home without `DP-2` never leaves a black screen.
- [ ] Exactly one Waybar and one watcher process are active.
- [ ] Hyprland reports no new config errors.
- [ ] Automated controller and watcher tests pass.
- [ ] Automated menu tests pass.
- [ ] No NVIDIA, kernel, initramfs, or GRUB settings were changed.
- [ ] All implementation files live in the existing dotfiles repository on the dedicated feature branch; no new repository was created.
- [ ] The backup and exact rollback instructions have been verified.

## Final Report Required from Codex

At completion, Codex should report:

1. files created and modified;
2. backup directory used;
3. exact discovered autostart and keybinding files;
4. tests and verification commands run, with summarized results;
5. final Work and Home monitor states;
6. active Waybar and watcher process counts;
7. any deviations from this plan and why;
8. the exact rollback command sequence.
