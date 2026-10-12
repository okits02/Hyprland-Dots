# Hyprland Multi-Monitor Layout Manager — Design Spec

> **For agentic workers:** this is a spec, not a plan. Produce an implementation
> plan from it with superpowers:writing-plans before writing code.

**Goal:** replace the hardcoded two-output Work/Home display profiles with a
device-identity-based layout manager that works for any number of monitors, on
laptops and desktops alike, so that anyone who installs these dotfiles can set
up their own monitors without editing code.

**Architecture:** a pure-logic Bash library owns the data model (monitor
identity, monitor-set fingerprint, layout resolution). A single controller
script is the only component allowed to change session state (monitors, focus,
Waybar). A rofi menu and a rofi parameter table are thin front-ends over the
library plus the controller. Drag-and-drop arrangement is delegated to
`nwg-displays`, which the dotfiles already ship in Quick Settings; this system
contributes the part `nwg-displays` cannot do — remembering an arrangement per
monitor set and restoring it automatically. A socket2 watcher calls the
controller when the set of connected monitors changes or the Hyprland config
reloads.

**Tech stack:** Bash (`#!/usr/bin/env bash`, `set -Eeuo pipefail`), `hyprctl`
(including `hyprctl eval` with the Lua config parser), Hyprland socket2 IPC via
`socat`, `jq`, `flock`, `rofi`, `nwg-displays` (optional), Waybar JSONC,
`notify-send`, shell test fixtures.

**Spec provenance:** approved in-chat design, 2026-10-04, sections 1–6, plus
the binding user decisions recorded under "User decisions" below. All
requirements are restated here so this document is self-contained.

**Supersedes:** `Agents/2026-10-03-hyprland-work-home-display-profiles.md` and
the implementation it produced. The two-output vocabulary (laptop/external,
Home/Work, `eDP-1` as a privileged output) is removed.

## User decisions (binding)

1. **Zero-config first.** With no saved layouts the system still works: it
   detects the connected monitors and arranges them. The UI exists to refine
   and remember, never as a precondition for the feature to function.
2. **Identity by device fingerprint**, not by port name.
3. **Any output may be disabled**, including a laptop panel. The old invariant
   "never disable `eDP-1`" is replaced by "never end with zero active outputs".
4. **Not laptop-specific.** Desktops with many monitors are a first-class case.
5. **Grid input** (row + order within row + vertical alignment), never raw
   coordinate entry.
6. **Several named layouts per monitor set**, one marked default.
7. **Waybar outputs are chosen per layout.**
8. **Drag-and-drop via `nwg-displays`**, with this system capturing the result.
9. **Upstream call sites are fixed, not documented around** (see "Upstream
   edits").

## Global constraints

- Do not modify NVIDIA drivers, kernel packages, initramfs, GRUB, or boot
  parameters.
- Do not install another display manager such as `kanshi`. `nwg-displays` is
  used only if already present.
- Work inside this dotfiles repository; do not create a second repository and
  do not initialize git anywhere.
- Do not commit. The user reviews and commits.
- Back up every existing file before changing it; record the backup directory
  in the README so rollback is exact.
- Preserve unrelated user changes: inspect each file before editing and patch
  only the required lines.
- Never leave more than one Waybar process running.
- `hyprctl keyword` does not work with this fork's Lua config parser — it
  prints an error and **still exits 0**. Use `hyprctl eval '<lua>'` and inspect
  the reply text for rejection.
- Every change is reversible; the README documents the rollback procedure.
- Stop on failed verification. Never continue by guessing.

## Terminology

| Term | Meaning |
|---|---|
| **output** | A port name as Hyprland reports it: `eDP-1`, `DP-2`, `HDMI-A-1`. Not stable across cable moves. |
| **identity** | A stable device key: `make\|model\|serial`. |
| **fingerprint** | The identities of all currently connected monitors, sorted and joined — the key of a monitor set. |
| **layout** | A named arrangement for one fingerprint: per-monitor enable/mode/scale/transform/position, plus primary and Waybar settings. |
| **store** | `~/.config/hypr/UserConfigs/display-layouts.json`. |

## Data model

### Identity

`identity = make + "|" + model + "|" + serial`, taken from
`hyprctl -j monitors all`.

- Observed values on the development machine: `BOE|0x0630|` (laptop panel, no
  serial) and `ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105`.
- The output name is deliberately **not** part of the identity: that is what
  allows a monitor moved from `DP-2` to `HDMI-A-1` to still be recognized.
- If `make` and `model` are both empty, identity falls back to the output name.
  Such an identity is port-dependent by necessity; the README says so.

### Fingerprint

The identities of all connected monitors (`hyprctl -j monitors all` — a
physically disconnected monitor disappears from this list entirely; a disabled
but connected one remains with `"disabled": true`), sorted lexicographically
and joined with `" + "`.

The joined string is used directly as the JSON object key. No hashing: the key
is longer but a human opening the file can tell which monitor set it describes,
and there is no hash implementation to get wrong.

### Duplicate identities

Two identical monitors with empty serials produce equal identities, so identity
alone cannot say which one is on the left. Therefore every monitor entry inside
a layout stores **both** `identity` and the `port` it occupied at capture time,
and assignment at apply time proceeds:

1. Match each layout entry to a connected monitor by `identity`.
2. Where several connected monitors share an identity, match those by `port`.
3. Where ports do not match either, assign the remaining candidates in
   ascending port-name order, deterministically.

### Store schema

Path: `~/.config/hypr/UserConfigs/display-layouts.json`. The repository ships
`{"version":1,"fingerprints":{}}` — empty means pure auto-detection, and
`version` provides a migration path.

```json
{
  "version": 1,
  "fingerprints": {
    "ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105 + BOE|0x0630|": {
      "label": "Nha",
      "default": "Home",
      "layouts": {
        "Home": {
          "monitors": [
            {
              "identity": "BOE|0x0630|",
              "port": "eDP-1",
              "enabled": true,
              "mode": "1920x1080@60.00",
              "scale": 1.5,
              "transform": 0,
              "x": 0,
              "y": 0,
              "grid": { "row": 1, "order": 1, "valign": "bottom" }
            },
            {
              "identity": "ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105",
              "port": "DP-2",
              "enabled": true,
              "mode": "1920x1080@119.88",
              "scale": 1,
              "transform": 0,
              "x": 1280,
              "y": 0,
              "grid": { "row": 1, "order": 2, "valign": "bottom" }
            }
          ],
          "primary": "ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105",
          "waybar": {
            "mode": "selected",
            "outputs": ["ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105"]
          }
        }
      }
    }
  }
}
```

Field rules:

- `mode` is `WIDTHxHEIGHT@REFRESH`. `hyprctl` reports available modes with a
  trailing `Hz` (`"1920x1080@119.88Hz"`); the trailing `Hz` is stripped before
  storing, because `hl.monitor` expects `1920x1080@119.88`.
- `scale` is a number, `transform` an integer 0–7 as Hyprland defines it.
  (`HL.MonitorSpec` in `hl.meta.lua` declares `transform? integer|boolean`, so
  it is settable through `hl.monitor`.)
- `x`, `y` are integers and are **the single source of truth when applying**.
- `grid` is optional **editor metadata only**. The parameter table recomputes
  `x`, `y` from `grid` on save; the applier never reads `grid`. This is why
  drag-and-drop output (arbitrary coordinates) and table output (grid) can
  coexist without two competing sources of truth.
- `primary` is an identity. Focus moves there after a layout is applied.
- `waybar.mode` is `all` | `primary` | `selected`; `waybar.outputs` is a list
  of identities and is read only when `mode` is `selected`.
- Layout names and the monitor-set `label`: `[A-Za-z0-9 _.-]`, 1–32
  characters. These two are the only free-text inputs in the system; both are
  validated by the same function at the point of entry.

### Derived geometry (grid → coordinates)

Logical size of a monitor: `w = mode_width / scale`, `h = mode_height / scale`,
with `w` and `h` swapped when `transform` is 1, 3, 5, or 7 (90°/270°).

- Rows are processed in ascending `row`; within a row, monitors in ascending
  `order`.
- Row height is the greatest logical height in that row. A row's `y` origin is
  the sum of the heights of the rows above it.
- Within a row, `x` is the running sum of the logical widths to its left.
- `valign` places the monitor inside its row band: `top` → `row_y`; `center` →
  `row_y + (row_h - h) / 2`; `bottom` → `row_y + row_h - h`.
- All results are rounded to integers.

This is what removes the error the old design invited: the old
`EXTERNAL_POSITION=1280x0` was really `1920 / 1.5`, and changing the scale
without recomputing it left a gap or an overlap.

## Resolution algorithm (`auto`)

1. Read `hyprctl -j monitors all`; build the connected set.
2. Compute the fingerprint.
3. **Exact fingerprint found in the store** → apply its `default` layout. This
   is the everyday path.
4. **No match** → generate a layout, and do not invent settings: for each
   connected identity, reuse `mode`, `scale` and `transform` from any stored
   layout containing that identity. Search order is fixed so results are
   reproducible in tests: fingerprints by monitor count descending, then by key
   ascending; within a fingerprint, its `default` layout; first hit wins.
   A generated layout is **not** written to the store. The menu offers to save
   it.
5. Apply.

Generated layout rules: every connected monitor enabled; `mode` = highest
resolution (greatest width x height) then highest refresh from
`availableModes` (falling back to the monitor's current mode); `scale` inherited per rule 4 else `1`; `transform`
inherited else `0`; single row, `y = 0`, ordered by ascending port name; `x`
cumulative by logical width; `primary` = the first monitor; `waybar.mode` =
`all`.

## Apply algorithm and safety invariants

Because any output may now be disabled, the governing invariant is: **never
end with zero active outputs.**

1. Reject before touching the session: a layout in which every monitor is
   `enabled: false`, a malformed layout, or a store that failed validation.
2. Configure every `enabled` monitor via
   `hyprctl eval 'hl.monitor({ output, mode, position, scale, transform })'`,
   inspecting the reply text (`hl.monitor` returns `ok` even for an output that
   does not exist, so the reply is not sufficient on its own — step 3 is).
3. **Verify** against `hyprctl -j monitors`: **every** monitor the layout
   enables reports its intended `width`, `x`, `y`, `scale` (±0.01) and
   `refreshRate` (±0.5), and at least one of them is active. On any mismatch:
   stop, log, notify, and **disable nothing**.
4. Disable the remaining monitors via
   `hyprctl eval 'hl.monitor({ output = "X", disabled = true })'`.
5. Verify again that at least one monitor is active. On failure, apply the
   **rescue layout**: every connected monitor enabled at its current mode (or
   `preferred` when that is unavailable), `scale` 1, `transform` 0, single row,
   `y = 0`, ordered by ascending port name. The rescue layout ignores the store
   entirely, so a bad stored layout cannot keep the screen dark.
6. Move focus to `primary` via
   `hyprctl eval 'hl.dispatch(hl.dsp.focus({ monitor = "X" }))'`.
   (`hyprctl dispatch focusmonitor` is rejected by the Lua parser.)
7. Apply Waybar (see below).
8. Write the current state (fingerprint + layout name) to the state directory.

Windows on a monitor being disabled: Hyprland moves their workspaces to another
monitor and does not move them back. This system logs which workspaces moved
and the README states the behavior. It is not worked around.

## Waybar

Three modes per layout, resolved from identities to current port names at apply
time, so the bars follow the monitor rather than the port:

- `all` — **generate nothing**; run Waybar with the user's own config path.
- `primary` — restrict to the layout's primary monitor.
- `selected` — restrict to the chosen monitors. Waybar accepts an array:
  `"output": ["DP-2", "eDP-1"]`.

Generation mechanism (carried over, extended from a string to an array): copy
the current `~/.config/hypr/waybar/config` symlink target and inject an
`"output"` key into every bar object, keyed off each bar's `"layer"` line,
removing any pre-existing `"output"` key including a multi-line array form. The
result is written to a single file, `config-active`, in the state directory —
one path, because layout names are arbitrary and cannot be used as filenames.

Restart suppression compares **both the path and the content**: if Waybar is
already running with the same config path and that file's content is identical
to the newly generated one, do not restart. The path matters because `all` mode
runs the user's own config while the other modes run `config-active`.
This is strictly stronger than the previous filename check — it also notices a
Waybar layout changed by `WaybarLayout.sh`.

A bar object with no `"layer"` key cannot be targeted; log a warning and fall
back to `all` for that run.

Waybar must be restarted through a transient systemd unit
(`systemd-run --user`) when available, falling back to `( ... & )` otherwise.
Reason, learned from `Refresh.sh:86-91`: a script invoked from a Waybar module
runs inside `waybar.service`'s cgroup, and killing Waybar from there makes
systemd tear down the cgroup, taking the restart command with it — the bar
never comes back.

## Components

The single boundary that matters is: who may change session state.

| Component | Responsibility | Touches session |
|---|---|---|
| `scripts/lib_display_store.sh` *(new)* | identity, fingerprint, store read/validate, layout resolution, layout generation, capture, save, delete | **No** — JSON in, JSON out |
| `scripts/DisplayProfile.sh` | apply, verify, rescue, focus, Waybar | Yes — sole owner |
| `scripts/DisplayProfileMenu.sh` | rofi menu, orchestration | No |
| `scripts/DisplayProfileSetup.sh` *(new)* | grid parameter table | No — writes JSON, then calls apply |
| `scripts/MonitorWatcher.sh` | socket2 events, debounce, pause, `configreloaded` | No — calls `auto` |

The library's independence from session state is what makes the system
testable: fingerprinting, matching and generation all run from JSON fixtures
with no monitors attached.

### CLI

```
DisplayProfile.sh auto                  Resolve and apply (the automatic path)
DisplayProfile.sh <layout-name>         Apply a named layout of the current set
DisplayProfile.sh capture <name>        Save the live state as a named layout
DisplayProfile.sh set-default <name>    Mark a layout as this set's default
DisplayProfile.sh delete <name>         Delete a layout
DisplayProfile.sh label <text>          Name the current monitor set
DisplayProfile.sh list                  Machine-readable listing for the menu
DisplayProfile.sh waybar-config         Print the Waybar config path to use
DisplayProfile.sh --dry-run             Print intended actions, change nothing
```

A bare layout name as the verb keeps the existing `SUPER+ALT+W` keybind working
unchanged, as long as a layout named `Work` exists.

Test seams follow the existing convention: `DP_HYPRCTL`, `DP_JQ`, `DP_WAYBAR`,
`DP_PGREP`, `DP_PKILL`, `DP_NOTIFY`, `DP_FLOCK`, `DP_WAYBAR_CONFIG`,
`DP_WAYBAR_STYLE`, `DP_RUNTIME_DIR`, `DP_LOG_FILE`, `DP_SKIP_LOCK`, plus
`DP_STORE_FILE`, `DP_MONITORS_LUA`, `DP_NWG_DISPLAYS`, `DP_SYSTEMD_RUN`.

### Menu (`SUPER+ALT+D`)

```
Monitor set: Nha (2 monitors) · running: Home

  Home                    * default, running
  Work
  Game 144Hz
  ----------------
  Automatic
  Arrange monitors — drag and drop (nwg-displays)
  Save current state as...
  Edit parameters (table)
  Set as default...
  Rename monitor set...
  Delete layout...
  Exit
```

The row count is now variable, so the menu uses `rofi -dmenu -format i`: rofi
returns the **index** of the chosen row and the script indexes a parallel
action array. User-supplied layout names are therefore never executed — the
existing "no `eval`" rule is preserved and tested.

### Parameter table (`DisplayProfileSetup.sh`)

```
Monitor set: Nha · layout: Home
 ASUS VG249Q3A (DP-2)   on · 1920x1080@119.88 · x1   · row 1 · order 2 · center
 BOE 0x0630 (eDP-1)     on · 1920x1080@60     · x1.5 · row 1 · order 1 · bottom
 Primary                ASUS VG249Q3A
 Waybar                 selected -> ASUS VG249Q3A
 Position (computed)    eDP-1 0x0 · DP-2 1280x0
 Save & apply  /  Cancel
```

Selecting a monitor row opens that monitor's submenu: on/off, mode, scale,
transform, row, order, vertical alignment. Every choice comes from the real
`availableModes` of that monitor, so invalid values cannot be entered.

The table is not only a fallback for machines without `nwg-displays`:
`nwg-displays` can express geometry, mode, scale and transform, but knows
nothing about `primary`, per-layout `enabled`, or Waybar outputs. It must also
be able to create a layout from scratch, not only edit an existing one — that
is how a laptop-only layout gets authored without first being in it.

### Drag-and-drop integration and the pause protocol

Flow: write a pause file containing the PID → launch
`nwg-displays -m <discard path in the state directory>` → the user arranges and
applies, seeing real results → the GUI exits → remove the pause file → reopen
the menu with "Save current state as..." first and a message that the current
state is unsaved.

- The pause file is mandatory, not cosmetic: applying from `nwg-displays` emits
  a burst of monitor events, and without the pause the watcher would
  immediately re-apply the stored layout and silently undo the user's work.
- The watcher ignores events while the pause file exists **and** its PID is
  alive. A stale pause file whose PID is dead is ignored and removed.
- `-m` points at a discard path because `nwg-displays` writes
  `~/.config/hypr/monitors.conf`, a file this Lua fork does not read. Capture,
  not that file, is how an arrangement persists.
- If `nwg-displays` is absent, the row is still shown and selecting it notifies
  that it must be installed — matching the pattern already used at
  `Kool_Quick_Settings.sh:507`.

## Watcher

- Events: `monitoradded`, `monitorremoved`, `monitorremovedv2`, debounced into
  a single `auto`, with retries on failure.
- New: honor the pause file (above).
- New: handle `configreloaded`. `SUPER+ALT+R` and any config reload re-apply
  `monitors.lua` over the running state; on this event the watcher re-applies
  the **current** layout.
- New: write the current state (fingerprint + layout name) so the menu can show
  what is running, and so a manually chosen layout survives a config reload. A
  manual choice persists until the **monitor set changes**; a new fingerprint
  returns to that set's `default`.
- Unchanged: singleton via `exec 9>` + `flock -n 9`, socket path re-resolved on
  every attempt, `--print-socket`.

## Coexistence with the dotfiles

`MonitorProfiles.sh` writes `UserConfigs/monitors.lua` and reloads the config.
Combined with the new `configreloaded` handling, this system would immediately
override whatever profile the user just chose. The rule:

**This system manages only what it has been given.** At startup, and on each
`auto`, inspect `UserConfigs/monitors.lua` for any non-comment `hl.monitor(`
line:

- Present, and the current fingerprint has **no** stored layout → **stand
  down**, log once. The user configured their monitors by hand or through
  `MonitorProfiles.sh`; it is not this system's business.
- Absent (the shipped state — the file ships as comments only) → detect and
  manage. This is what makes the zero-config decision work for new users.
- The fingerprint **has** a stored layout → manage it regardless of
  `monitors.lua`. Saving a layout is the explicit act of handing over control.

Trade-off, accepted: a user who has written their own `monitors.lua` sees the
feature do nothing until they save a layout. The README must state this
plainly, because "it does nothing" is otherwise indistinguishable from a bug.

### Upstream edits

Each edit must keep working on a machine where `DisplayProfile.sh` does not
exist, so each call site needs a fallback. The read-only
`DisplayProfile.sh waybar-config` verb exists for exactly this.

| File | Change |
|---|---|
| `scripts/Refresh.sh:105` | Take the Waybar config path from `waybar-config`, falling back to `$waybar_dir/config`. Today it hardcodes the unrestricted config and undoes the per-layout Waybar restriction on every refresh. |
| `scripts/WaybarLayout.sh` (`apply_config`) | Same, after switching the symlink. |
| `scripts/Kool_Quick_Settings.sh:511` | Route `nwg-displays` through the pause protocol and the discard path, then offer capture. Today it writes a real `monitors.conf` that this fork never reads, so a user's drag-and-drop work is silently lost. |

## Error handling

Governing principle: **never destroy user data.**

| Situation | Behavior |
|---|---|
| Store file missing | Treat as empty, run `auto`. Never a hard failure. |
| Store malformed (`jq` cannot parse) | Log, notify once, run `auto`, and **refuse all writes** to that file. Only the menu's Reset writes, and it backs up to `display-layouts.json.bak-<timestamp>` first. |
| `version` newer than known | Read best-effort, refuse writes, notify. |
| Layout references an absent identity | Skip that monitor (the inheritance path). |
| Every monitor disabled in a layout | Reject at read time, before touching the session. |
| `jq` missing | Hard failure with a clear message. `jq` is already a dotfiles dependency. |
| `nwg-displays` missing | Row shown; selecting it notifies to install it. |
| `flock` missing | Warn and continue without a lock, as today. |
| Apply verification fails | Stop; disable nothing; notify. |
| Zero active outputs after disabling | Rescue layout. |

## Concurrency and file-descriptor rules

These are requirements, not aspirations. The two most expensive bugs in the
previous implementation were both file-descriptor leaks that silently disabled
the feature rather than reporting an error: Waybar inherited `flock`'s
descriptor and held the lock for its whole lifetime, and later `socat` did the
same with the watcher's singleton lock.

- Locking is always `exec 9>"$LOCK_FILE"` followed by `flock -w 10 9`. Never
  `exec flock ... "$0"`.
- **Every** spawned child closes the lock descriptor: `9>&-` on Waybar, on
  `socat`, on `nwg-displays`, and on the watcher's call into the controller.
- `--dry-run` takes no lock.
- All store writes are atomic: a temporary file in the same directory, then
  `mv`, under the lock.
- `log()` writes to **stderr** in every script, because functions such as the
  Waybar config generator return a path on stdout. A log line on stdout
  previously produced a corrupt Waybar config path and made the bars vanish.

## State and logging

Under `$XDG_RUNTIME_DIR/kooldots-display-profiles/`:
`display-profile.log`, `monitor-watcher.log`, `current` (fingerprint + layout
name), `config-active` (generated Waybar config), `nwg-monitors.conf.discard`,
`pause` (PID). Locks remain at `$XDG_RUNTIME_DIR/kooldots-display-profile.lock`
and `kooldots-monitor-watcher.lock`. Stale `config-home` / `config-work` files
from the previous implementation are removed on first run.

## Migration and distribution

No migration code. Reading the eight old constants to synthesize two layouts
would be code that runs once, for one machine, and then ships to people who
never had the previous version. Instead, the README documents two steps:

- With the external monitor connected: menu → "Save current state as..." →
  `Home`. This captures exactly what is running.
- `Work` cannot be captured without first being in it, so it is authored in the
  table: turn the external monitor off, save as `Work`.
- `SUPER+ALT+W` needs no change; a layout named `Work` makes it work.

Distribution: `copy.sh` ships `UserConfigs/display-layouts.json` as
`{"version":1,"fingerprints":{}}`, and `MonitorWatcher.sh` stays enabled in
`user_startup.lua` — safe because of the stand-down rule above. A new user
installs the dotfiles, plugs in a second monitor, and gets a usable
arrangement; saving a layout is how they make it permanent.

## Testing requirements

The existing harness is reused: `tests/lib_test.sh` (fake-command sandbox,
`assert_order`, `assert_count_re`), with tests run under `timeout` and an empty
`XDG_RUNTIME_DIR` so they can never discover the live Hyprland socket.

### Fixtures

`hyprctl -j monitors all` captures for: laptop plus one external (the
development machine, so the existing checks keep their meaning); a three-monitor
desktop with mixed resolutions (2560x1440 plus two 1920x1080); a four-monitor
desktop in **two rows**; **two identical monitors with empty serials**; a
monitor with empty make and model; a rotated monitor (`transform: 1`). Store
fixtures: empty, valid v1, malformed, `version: 99`.

### Required test coverage

- **No hardcoded outputs.** A scan asserting the strings `eDP-1` and `DP-2`
  appear nowhere in the scripts outside fixtures and documentation. This is the
  test that answers the originating request — that the configuration is no
  longer specific to one machine — and it is what keeps those names from
  creeping back.
- **Library (pure):** identity; fingerprint; assignment with duplicate
  identities; exact match; inheritance when no fingerprint matches, including
  the determinism of the search order; layout generation; grid → coordinates
  for scale 1, 1.25 and 1.5, for multiple rows, for all three vertical
  alignments, and for a rotated monitor; capture; every validation rejection.
- **Applier:** enable-before-disable ordering (`assert_order`); a failed
  verification disables nothing; zero active outputs triggers the rescue
  layout; focus goes to `primary`; all three Waybar modes including array
  injection; restart suppressed when the generated content is unchanged; a
  structural test asserting `9>&-` on every child spawn, in the same spirit as
  the existing "no `eval`" test.
- **Watcher:** pause file honored with a live PID and ignored with a dead one;
  `configreloaded` re-applies the current layout rather than the default;
  debounce; singleton; stand-down when `monitors.lua` has a non-comment
  `hl.monitor` line.
- **Menu:** `-format i` index mapping with a varying row count; layout-name
  validation; no `eval`; no writes when the store is malformed; Reset backs up
  before writing.
- **Upstream edits:** each of the three call sites falls back to its previous
  behavior when `DisplayProfile.sh` is absent. Stated limit: those scripts do
  much else, so the tests cover the fallback contract, not a full run of them.

### Out of automated reach

Declared up front, to be handed to the user rather than implied as covered:

1. A real apply on three or more monitors. The development machine has two.
2. Drag-and-drop inside `nwg-displays` — a GUI interaction.
3. Disabling a laptop panel while it is in use, including the workspace
   migration that follows.

## Out of scope

- Per-workspace monitor pinning (`workspaces.conf` stays the user's).
- Wallpaper, VRR, HDR, and color management per monitor.
- Replacing `MonitorProfiles.sh` or `Monitor_Profiles/*.lua`; they keep working
  and the stand-down rule keeps the two systems out of each other's way.
- Writing `monitors.conf` or `monitors.lua`. This system owns runtime state
  only; `hyprctl eval` is its single mechanism. The `nwg-displays` discard path
  exists solely to stop that program writing the real `monitors.conf`.

## Rollback

The README records the backup directory created before any file is changed.
Rollback: stop the watcher, restore the backed-up `UserConfigs/*.lua` and the
three upstream scripts, delete the new scripts, delete
`UserConfigs/display-layouts.json`, remove the state directory, and reload
Hyprland.
