# Hyprland Multi-Monitor Layout Manager Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the hardcoded two-output Work/Home display profiles with a device-identity-based layout manager that handles any number of monitors on laptops and desktops, so anyone installing these dotfiles can set up their monitors without editing code.

**Architecture:** A pure-logic library (`lib_display_store.sh`) owns the data model and never touches session state, which makes every rule testable from JSON fixtures. One controller (`DisplayProfile.sh`) is the sole writer of session state. A rofi menu and a rofi parameter table are thin front-ends. Drag-and-drop is delegated to `nwg-displays`; this system captures its result and restores it per monitor set. A socket2 watcher calls `auto` on monitor changes and config reloads.

**Tech Stack:** Bash (`#!/usr/bin/env bash`, `set -Eeuo pipefail`), `hyprctl` + `hyprctl eval` (Lua config parser), Hyprland socket2 via `socat`, `jq`, `awk`, `flock`, `rofi`, `nwg-displays` (optional), Waybar JSONC, `notify-send`.

**Spec:** `Agents/2026-10-04-hyprland-multi-monitor-layouts-design.md`

## Global Constraints

Copied from the spec's "Global constraints", "Concurrency and file-descriptor rules", and the binding user decisions. Every task's requirements implicitly include this section.

- Do not modify NVIDIA drivers, kernel packages, initramfs, GRUB, or boot parameters.
- Do not install another display manager such as `kanshi`. `nwg-displays` is used only if already present.
- Work inside this dotfiles repository; do not create a second repository and do not initialize git anywhere.
- **Do not commit.** The user reviews and commits. Tasks therefore end with a verification step, not a commit step.
- Back up every existing file before changing it; record the backup directory in the README so rollback is exact. Use one backup directory for the whole run: `~/.config/hypr/backups/multi-monitor-<YYYYMMDD-HHMMSS>/`.
- Preserve unrelated user changes: inspect each file before editing and patch only the required lines.
- Never leave more than one Waybar process running.
- `hyprctl keyword` does not work with this fork's Lua config parser — it prints an error and **still exits 0**. Use `hyprctl eval '<lua>'` and inspect the reply text for rejection.
- All scripts: `#!/usr/bin/env bash` and `set -Eeuo pipefail`, with the KooLDots GPL header comment block copied from an existing script.
- `log()` writes to **stderr** in every script. Functions return paths on stdout; a log line on stdout previously corrupted the Waybar config path and made the bars vanish.
- Locking is always `exec 9>"$LOCK_FILE"` then `flock -w 10 9`. Never `exec flock ... "$0"`.
- **Every** spawned child closes the lock descriptor with `9>&-`: Waybar, `socat`, `nwg-displays`, and the watcher's call into the controller.
- `--dry-run` takes no lock and changes nothing.
- All store writes are atomic: temp file in the same directory, then `mv`, under the lock.
- Never end with zero active outputs. `eDP-1` is no longer privileged; any output may be disabled.
- No output name (`eDP-1`, `DP-2`, …) may appear in any script. Task 13 enforces this with a test.
- Every change is reversible; the README documents the rollback procedure.
- Stop on failed verification. Never continue by guessing.

**Paths used throughout:**

| Variable | Value |
|---|---|
| `$S` | `config/hypr/scripts` (repo) — deploy target `~/.config/hypr/scripts` |
| `$T` | `config/hypr/scripts/tests` |
| `$F` | `config/hypr/scripts/tests/fixtures` |
| store | `~/.config/hypr/UserConfigs/display-layouts.json` |
| state dir | `$XDG_RUNTIME_DIR/kooldots-display-profiles/` |

**Deployment:** the repo under `/home/okits/Documents/Dev/Hyperland/Hyprland-Dots/config/hypr/` is the source of truth; after each task, copy changed files to `~/.config/hypr/` so the live session matches, and verify with `diff`. Tests run from the repo copy.

## Review Focus

Five input classes the spec implies but whose tests do not fall out of any task's happy path. Each line's test is added to the task that owns the code, named in brackets.

1. **A monitor with an empty `availableModes` array** (cheap adapters, some virtual outputs). Generation must fall back to the monitor's current `width x height @ refreshRate` and never emit a malformed mode such as `"@"` or `"x@"`. [Task 5]
2. **A stored fingerprint whose `default` names a layout that no longer exists** (the user deleted it by hand). Resolution must not silently apply nothing: fall back to the alphabetically first layout of that fingerprint, and if there are none, to a generated layout. [Task 5]
3. **A layout name that passes validation but looks like an option**, e.g. `-n` or `--help`, used as the controller's bare verb and as a rofi argument. Every call site must use `--` guards and quoting so the name is never parsed as a flag. [Task 7, Task 10]
4. **Fractional logical widths** (e.g. 1920 ÷ 1.1 = 1745.45) across three monitors. Rounding must not accumulate into an overlap or a gap: positions come from the running un-rounded sum, rounded only at output. [Task 4]
5. **`hyprctl -j monitors all` returning `[]`** (during resume, or a GPU hiccup). Everything must stand down: no fingerprint, no apply, no rescue, no division by zero — log and exit 0. [Task 2, Task 7]

---

## File Structure

**Created:**

| File | Responsibility |
|---|---|
| `$S/lib_display_store.sh` | Pure data model: normalize, fingerprint, validate, geometry, resolve, generate, capture, atomic store writes. No `hyprctl`, no session state. |
| `$S/DisplayProfileSetup.sh` | Grid parameter table (rofi). Writes the store, then calls the controller. |
| `$T/test-display-store.sh` | Library tests (Tasks 2–6). |
| `$T/test-display-profile-setup.sh` | Parameter table tests (Task 11). |
| `$T/test-upstream-fallbacks.sh` | Upstream call-site fallback tests (Task 12). |
| `$T/test-no-hardcoded-outputs.sh` | The anti-regression scan (Task 13). |
| `$F/monitors-*.json`, `$F/store-*.json` | Fixtures (Task 1). |
| `config/hypr/UserConfigs/display-layouts.json` | Shipped empty store (Task 13). |

**Modified:**

| File | Change |
|---|---|
| `$S/DisplayProfile.sh` | Rewritten around the library; new CLI; N-monitor apply; rescue; Waybar modes (Tasks 7–8). |
| `$S/MonitorWatcher.sh` | Pause protocol, `configreloaded`, state file, stand-down (Task 9). |
| `$S/DisplayProfileMenu.sh` | Dynamic rows via `rofi -format i`; all new actions (Task 10). |
| `$S/Refresh.sh` (~line 105) | Waybar config path from `waybar-config`, with fallback (Task 12). |
| `$S/WaybarLayout.sh` (`apply_config`) | Same (Task 12). |
| `$S/Kool_Quick_Settings.sh` (~line 511) | `nwg-displays` through the pause protocol (Task 12). |
| `$T/lib_test.sh` | New assertions and fakes (Task 1). |
| `$S/README-display-profiles.md` | Rewritten for the new model (Task 13). |
| `config/hypr/UserConfigs/user_keybinds.lua` | Keybind descriptions (Task 13). |
| `config/hypr/copy.sh` | Ship the empty store (Task 13). |

**Deleted:** nothing. The old `DisplayProfile.sh` is rewritten in place so its keybind and path stay valid.

---

### Task 1: Fixtures and test-harness extensions

**Files:**
- Create: `$F/monitors-dual.json`, `$F/monitors-triple.json`, `$F/monitors-quad-2rows.json`, `$F/monitors-twins-noserial.json`, `$F/monitors-nomakemodel.json`, `$F/monitors-rotated.json`, `$F/monitors-empty.json`, `$F/monitors-nomodes.json`
- Create: `$F/store-empty.json`, `$F/store-v1.json`, `$F/store-malformed.json`, `$F/store-future.json`, `$F/store-missing-default.json`
- Modify: `$T/lib_test.sh`
- Test: `$T/test-fixtures.sh`

**Interfaces:**
- Consumes: nothing.
- Produces: fixture paths above; and in `lib_test.sh`, one consistent signature family used by every later task — `assert_eq <label> <expected> <actual>`, `assert_json_eq <file> <jq-filter> <expected>`, `assert_contains <label> <file> <needle>`, `assert_not_contains <label> <file> <needle>`, `assert_contains_str <label> <string> <needle>`, `assert_not_contains_str <label> <string> <needle>` (the `_str` pair takes a value, the file pair takes a path), fake `nwg-displays` honouring `FAKE_NWG_EXIT` and recording its argv, fake `systemd-run` that records argv and executes the trailing command, and `FAKE_HYPRCTL_MONITORS_FILE` so the existing fake `hyprctl` answers `-j monitors`/`-j monitors all` from a fixture.

- [ ] **Step 1: Write the failing fixture test**

Create `$T/test-fixtures.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
HERE="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=./lib_test.sh
source "$HERE/lib_test.sh"

F="$HERE/fixtures"

for f in monitors-dual monitors-triple monitors-quad-2rows monitors-twins-noserial \
         monitors-nomakemodel monitors-rotated monitors-empty monitors-nomodes \
         store-empty store-v1 store-future store-missing-default; do
  check "fixture $f is valid JSON" jq -e . "$F/$f.json"
done

check "malformed store really is malformed" bash -c '! jq -e . "'"$F"'/store-malformed.json" >/dev/null 2>&1'

assert_json_eq "$F/monitors-dual.json"          'length' 2
assert_json_eq "$F/monitors-triple.json"        'length' 3
assert_json_eq "$F/monitors-quad-2rows.json"    'length' 4
assert_json_eq "$F/monitors-empty.json"         'length' 0
assert_json_eq "$F/monitors-twins-noserial.json" '[.[] | select(.serial == "")] | length' 2
assert_json_eq "$F/monitors-rotated.json"       '[.[] | select(.transform == 1)] | length' 1
assert_json_eq "$F/monitors-nomodes.json"       '[.[] | select((.availableModes | length) == 0)] | length' 1
assert_json_eq "$F/monitors-nomakemodel.json"   '[.[] | select(.make == "" and .model == "")] | length' 1

test_summary
```

- [ ] **Step 2: Run it to verify it fails**

Run: `bash config/hypr/scripts/tests/test-fixtures.sh`
Expected: FAIL — `assert_json_eq: command not found`, and every fixture missing.

- [ ] **Step 3: Add the harness helpers**

Append to `$T/lib_test.sh`:

```bash
assert_json_eq() {
  local file=$1 filter=$2 expected=$3 actual
  actual="$(jq -r "$filter" "$file" 2>&1 || true)"
  if [[ $actual == "$expected" ]]; then
    pass "jq '$filter' on $(basename -- "$file") == $expected"
  else
    fail "jq '$filter' on $(basename -- "$file"): expected '$expected', got '$actual'"
  fi
}

assert_eq() {
  local label=$1 expected=$2 actual=$3
  if [[ $actual == "$expected" ]]; then pass "$label"
  else fail "$label: expected '$expected', got '$actual'"; fi
}

assert_contains() {       # label, FILE, needle
  local label=$1 file=$2 needle=$3
  if grep -qF -- "$needle" "$file" 2>/dev/null; then pass "$label"
  else fail "$label: '$needle' not found in $(basename -- "$file")"; fi
}

assert_not_contains() {   # label, FILE, needle
  local label=$1 file=$2 needle=$3
  if grep -qF -- "$needle" "$file" 2>/dev/null; then
    fail "$label: '$needle' must not appear in $(basename -- "$file")"
  else pass "$label"; fi
}

assert_contains_str() {   # label, STRING, needle
  local label=$1 hay=$2 needle=$3
  if [[ $hay == *"$needle"* ]]; then pass "$label"
  else fail "$label: '$needle' not found in the captured output"; fi
}

assert_not_contains_str() {
  local label=$1 hay=$2 needle=$3
  if [[ $hay == *"$needle"* ]]; then fail "$label: '$needle' must not appear"
  else pass "$label"; fi
}
```

In the fake-command sandbox, add `nwg-displays`:

```bash
cat >"$SANDBOX/bin/nwg-displays" <<'EOS'
#!/usr/bin/env bash
printf 'nwg-displays %s\n' "$*" >>"$FAKE_LOG"
exit "${FAKE_NWG_EXIT:-0}"
EOS
```

and `systemd-run`, which records its argv and then runs the command after the last option so Waybar still "starts" in tests:

```bash
cat >"$SANDBOX/bin/systemd-run" <<'EOS'
#!/usr/bin/env bash
printf 'systemd-run %s\n' "$*" >>"$FAKE_LOG"
cmd=""
for a in "$@"; do case $a in --*) ;; *) cmd="$a" ;; esac; done
[[ -n $cmd ]] && bash -c "$cmd" >/dev/null 2>&1 &
exit 0
EOS
```

Extend the existing fake `hyprctl` so `-j monitors` and `-j monitors all` print `$FAKE_HYPRCTL_MONITORS_FILE` when it is set (falling back to the current `FAKE_HYPRCTL_STDOUT` behaviour), and `chmod +x` all new fakes alongside the existing ones.

- [ ] **Step 4: Write the fixtures**

`$F/monitors-dual.json` — the development machine, captured verbatim so the earlier suite keeps its meaning:

```json
[
  {"name":"eDP-1","description":"BOE 0x0630","make":"BOE","model":"0x0630","serial":"",
   "width":1920,"height":1080,"refreshRate":60.027,"x":0,"y":0,"scale":1.5,
   "transform":0,"focused":false,"disabled":false,
   "availableModes":["1920x1080@60.00Hz"]},
  {"name":"DP-2","description":"ASUSTek COMPUTER INC VG249Q3A S7LMBS003105",
   "make":"ASUSTek COMPUTER INC","model":"VG249Q3A","serial":"S7LMBS003105",
   "width":1920,"height":1080,"refreshRate":119.879,"x":1280,"y":0,"scale":1,
   "transform":0,"focused":true,"disabled":false,
   "availableModes":["1920x1080@119.88Hz","1920x1080@100.00Hz","1920x1080@60.00Hz"]}
]
```

The remaining fixtures use the same field set. Required contents:

- `monitors-triple.json` — ports `DP-1` (`Dell|U2720Q|ABC123`, 2560x1440@59.95, scale 1), `DP-3` (`Dell|P2419H|DEF456`, 1920x1080@60.00, scale 1), `HDMI-A-1` (`AOC|24G2|GHI789`, 1920x1080@144.00, scale 1). All enabled, all `transform` 0, each with its own mode in `availableModes` plus one lower refresh.
- `monitors-quad-2rows.json` — four 1920x1080@60.00 monitors, ports `DP-1`…`DP-4`, makes `AOC`/`AOC`/`Dell`/`Dell`, serials `A1`,`A2`,`D1`,`D2`, all scale 1.
- `monitors-twins-noserial.json` — two monitors, both `make":"AOC","model":"24G2","serial":""`, ports `DP-1` and `DP-2`, 1920x1080@60.00, scale 1.
- `monitors-nomakemodel.json` — two monitors: one normal (`Dell|P2419H|DEF456`, port `DP-1`), one with `"make":"","model":"","serial":""` on port `HDMI-A-2`.
- `monitors-rotated.json` — two monitors: `DP-1` landscape 1920x1080@60.00 scale 1, `DP-2` with `"transform":1`, 1080x1920 panel reported as `"width":1080,"height":1920`, mode `1080x1920@60.00Hz`, scale 1.
- `monitors-empty.json` — `[]`.
- `monitors-nomodes.json` — one monitor, port `HDMI-A-1`, `Generic|Adapter|`, `width` 1920, `height` 1080, `refreshRate` 59.94, scale 1, `"availableModes": []`.
- `store-empty.json` — `{"version":1,"fingerprints":{}}`.
- `store-v1.json` — exactly the schema example from the spec's "Store schema" section, plus a second layout `Work` under the same fingerprint in which the `BOE|0x0630|` entry is `"enabled": true` and the ASUS entry is `"enabled": false`, and `"default": "Home"`.
- `store-malformed.json` — `{"version":1,"fingerprints":{` (deliberately truncated).
- `store-future.json` — `store-v1.json` with `"version": 99`.
- `store-missing-default.json` — one fingerprint, `"default": "Gone"`, with a single real layout named `Alpha`.

- [ ] **Step 5: Run the test to verify it passes**

Run: `bash config/hypr/scripts/tests/test-fixtures.sh`
Expected: PASS, 21 checks run, 0 failed.

- [ ] **Step 6: Verify the existing suite still passes**

Run:
```bash
for t in test-display-profile test-monitor-watcher test-display-profile-menu test-fixtures; do
  bash config/hypr/scripts/tests/$t.sh || echo "FAILED: $t"
done
```
Expected: four suites, 0 failed. No commit (see Global Constraints).

---

### Task 2: Library — normalize and fingerprint

**Files:**
- Create: `$S/lib_display_store.sh`
- Create: `$T/test-display-store.sh`

**Interfaces:**
- Consumes: Task 1 fixtures.
- Produces:
  - `ds_normalize <monitors.json>` → stdout: JSON array sorted by `port`, each element `{port, identity, make, model, serial, disabled, width, height, refreshRate, x, y, scale, transform, modes[]}` where `modes` have the trailing `Hz` stripped.
  - `ds_fingerprint <normalized.json>` → stdout: identities sorted and joined with `" + "`; empty string for an empty array, exit 0.
  - `ds_count <normalized.json>` → stdout: integer.
  - Env seams: `DS_JQ` (default `jq`).
  - Identity is implemented **once**, inside `ds_normalize`'s jq program. No Bash twin.

- [ ] **Step 1: Write the failing tests**

Create `$T/test-display-store.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
HERE="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=./lib_test.sh
source "$HERE/lib_test.sh"
F="$HERE/fixtures"
WORK="$(mktemp -d)"; trap 'rm -rf -- "$WORK"' EXIT
# shellcheck source=../lib_display_store.sh
source "$HERE/../lib_display_store.sh"

norm() { ds_normalize "$F/monitors-$1.json" >"$WORK/$1.norm.json"; printf '%s' "$WORK/$1.norm.json"; }

n_dual="$(norm dual)"
assert_json_eq "$n_dual" '.[0].port'     'DP-2'
assert_json_eq "$n_dual" '.[0].identity' 'ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105'
assert_json_eq "$n_dual" '.[1].identity' 'BOE|0x0630|'
assert_json_eq "$n_dual" '.[1].modes[0]' '1920x1080@60.00'
assert_eq "fingerprint of the dual set" \
  'ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105 + BOE|0x0630|' \
  "$(ds_fingerprint "$n_dual")"

n_nmm="$(norm nomakemodel)"
assert_eq "identity falls back to the port name when make and model are empty" \
  'HDMI-A-2' "$(jq -r '.[] | select(.make == "") | .identity' "$n_nmm")"

n_twins="$(norm twins-noserial)"
assert_eq "twins share one identity" '1' \
  "$(jq -r '[.[].identity] | unique | length' "$n_twins")"
assert_eq "twins keep distinct ports" '2' \
  "$(jq -r '[.[].port] | unique | length' "$n_twins")"

n_empty="$(norm empty)"
assert_eq "empty monitor list yields an empty fingerprint" '' "$(ds_fingerprint "$n_empty")"
assert_eq "empty monitor list counts zero" '0' "$(ds_count "$n_empty")"

n_nomodes="$(norm nomodes)"
assert_eq "a monitor with no available modes normalizes to an empty modes array" '0' \
  "$(jq -r '.[0].modes | length' "$n_nomodes")"

test_summary
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: FAIL — `lib_display_store.sh: No such file or directory`.

- [ ] **Step 3: Create the library with these two functions**

`$S/lib_display_store.sh` (KooLDots header, then):

```bash
# Pure data model for display layouts. No hyprctl, no session state.
# Sourced by DisplayProfile.sh, DisplayProfileMenu.sh, DisplayProfileSetup.sh.
DS_JQ="${DS_JQ:-jq}"

ds_log() { printf 'lib_display_store: %s\n' "$*" >&2; }

# Normalize `hyprctl -j monitors all` into the shape the rest of the system uses.
# Identity is defined here and nowhere else.
ds_normalize() {
  "$DS_JQ" -c '
    [ .[] | {
        port: .name,
        make: (.make // ""), model: (.model // ""), serial: (.serial // ""),
        identity: (
          if ((.make // "") == "" and (.model // "") == "")
          then .name
          else ((.make // "") + "|" + (.model // "") + "|" + (.serial // ""))
          end
        ),
        disabled: (.disabled // false),
        width: (.width // 0), height: (.height // 0),
        refreshRate: (.refreshRate // 0),
        x: (.x // 0), y: (.y // 0),
        scale: (.scale // 1), transform: (.transform // 0),
        modes: [ (.availableModes // [])[] | sub("Hz$"; "") ]
      } ]
    | sort_by(.port)
  ' -- "$1"
}

ds_fingerprint() { "$DS_JQ" -r '[ .[].identity ] | sort | join(" + ")' -- "$1"; }

ds_count() { "$DS_JQ" -r 'length' -- "$1"; }
```

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: PASS, 12 checks, 0 failed.

- [ ] **Step 5: Verify**

Run the four suites as in Task 1 Step 6. Expected: 0 failed.

---

### Task 3: Library — store status, read, validation, name rules

**Files:**
- Modify: `$S/lib_display_store.sh`
- Modify: `$T/test-display-store.sh`

**Interfaces:**
- Consumes: Task 2 (`DS_JQ`, `ds_log`).
- Produces:
  - `ds_store_status <store-path>` → stdout one of `missing` | `ok` | `malformed` | `future`; always exit 0.
  - `ds_store_read <store-path>` → stdout: the store JSON, or `{"version":1,"fingerprints":{}}` when status is `missing` or `malformed`; always exit 0.
  - `ds_store_writable <store-path>` → exit 0 only for `missing` or `ok`.
  - `ds_name_valid <name>` → exit 0 when the name matches `^[A-Za-z0-9 _.-]{1,32}$`.
  - `ds_validate_layout <layout.json>` → exit 0, or exit 1 with a one-line reason on stderr. Rejections: not an object; `monitors` missing or empty; every monitor `enabled: false`; a monitor with no `identity`; `scale` ≤ 0 or non-numeric; `mode` not matching `^[0-9]+x[0-9]+@[0-9.]+$`; `transform` outside 0–7; `waybar.mode` not in `all|primary|selected`.

- [ ] **Step 1: Write the failing tests**

Append to `$T/test-display-store.sh` before `test_summary`:

```bash
assert_eq "missing store"   'missing'   "$(ds_store_status "$WORK/nope.json")"
assert_eq "empty store"     'ok'        "$(ds_store_status "$F/store-empty.json")"
assert_eq "v1 store"        'ok'        "$(ds_store_status "$F/store-v1.json")"
assert_eq "malformed store" 'malformed' "$(ds_store_status "$F/store-malformed.json")"
assert_eq "future store"    'future'    "$(ds_store_status "$F/store-future.json")"

assert_eq "a malformed store reads as an empty store" '0' \
  "$(ds_store_read "$F/store-malformed.json" | jq -r '.fingerprints | length')"
assert_eq "a missing store reads as an empty store" '1' \
  "$(ds_store_read "$WORK/nope.json" | jq -r '.version')"

check "an ok store is writable"        ds_store_writable "$F/store-empty.json"
check "a missing store is writable"    ds_store_writable "$WORK/nope.json"
check "a malformed store is NOT writable" bash -c '! ds_store_writable "'"$F"'/store-malformed.json"'
check "a future store is NOT writable"    bash -c '! ds_store_writable "'"$F"'/store-future.json"'

check "plain name accepted"     ds_name_valid 'Home'
check "spaces and dots accepted" ds_name_valid 'Game 144.0_x-y'
check "empty name rejected"     bash -c '! ds_name_valid ""'
check "33 characters rejected"  bash -c '! ds_name_valid "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"'
check "slash rejected"          bash -c '! ds_name_valid "a/b"'
check "quote rejected"          bash -c '! ds_name_valid "a\"b"'
check "dollar rejected"         bash -c '! ds_name_valid "a\$b"'

jq -r '.fingerprints[].layouts.Home' "$F/store-v1.json" >"$WORK/good.json"
check "the spec's example layout validates" ds_validate_layout "$WORK/good.json"

jq '.monitors = []' "$WORK/good.json" >"$WORK/bad-empty.json"
check "a layout with no monitors is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-empty.json" 2>/dev/null'

jq '.monitors |= map(.enabled = false)' "$WORK/good.json" >"$WORK/bad-alloff.json"
check "a layout with every monitor disabled is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-alloff.json" 2>/dev/null'

jq '.monitors[0].scale = 0' "$WORK/good.json" >"$WORK/bad-scale.json"
check "scale 0 is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-scale.json" 2>/dev/null'

jq '.monitors[0].scale = -1.5' "$WORK/good.json" >"$WORK/bad-negscale.json"
check "a negative scale is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-negscale.json" 2>/dev/null'

jq '.monitors[0].mode = "preferred"' "$WORK/good.json" >"$WORK/bad-mode.json"
check "a non-WxH@R mode is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-mode.json" 2>/dev/null'

jq '.monitors[0].transform = 9' "$WORK/good.json" >"$WORK/bad-transform.json"
check "transform 9 is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-transform.json" 2>/dev/null'

jq '.waybar.mode = "sometimes"' "$WORK/good.json" >"$WORK/bad-waybar.json"
check "an unknown waybar mode is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-waybar.json" 2>/dev/null'

jq 'del(.monitors[0].identity)' "$WORK/good.json" >"$WORK/bad-noident.json"
check "a monitor without an identity is rejected" bash -c '! ds_validate_layout "'"$WORK"'/bad-noident.json" 2>/dev/null'
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: FAIL — `ds_store_status: command not found` and the rest.

- [ ] **Step 3: Implement**

Append to `$S/lib_display_store.sh`:

```bash
DS_VERSION=1

ds_store_status() {
  local f=$1 v
  [[ -f $f ]] || { printf 'missing\n'; return 0; }
  if ! "$DS_JQ" -e . -- "$f" >/dev/null 2>&1; then printf 'malformed\n'; return 0; fi
  v="$("$DS_JQ" -r '.version // 0' -- "$f" 2>/dev/null || printf '0')"
  if [[ $v =~ ^[0-9]+$ ]] && (( v > DS_VERSION )); then printf 'future\n'; return 0; fi
  printf 'ok\n'
}

ds_store_read() {
  local f=$1
  case "$(ds_store_status "$f")" in
    ok | future) "$DS_JQ" -c '{ version: (.version // 1), fingerprints: (.fingerprints // {}) }' -- "$f" ;;
    *) printf '{"version":%s,"fingerprints":{}}\n' "$DS_VERSION" ;;
  esac
}

ds_store_writable() {
  case "$(ds_store_status "$1")" in ok | missing) return 0 ;; *) return 1 ;; esac
}

ds_name_valid() {
  local n=${1-}
  [[ $n =~ ^[A-Za-z0-9\ _.-]{1,32}$ ]]
}

ds_validate_layout() {
  local f=$1 reason
  reason="$("$DS_JQ" -r '
    def bad($m): $m;
    if type != "object" then bad("not an object")
    elif (.monitors | type) != "array" then bad("monitors is not an array")
    elif (.monitors | length) == 0 then bad("monitors is empty")
    elif ([ .monitors[] | select(.enabled == true) ] | length) == 0 then bad("every monitor is disabled")
    elif ([ .monitors[] | select((.identity | type) != "string" or .identity == "") ] | length) > 0 then bad("a monitor has no identity")
    elif ([ .monitors[] | select((.scale | tonumber? // 0) <= 0) ] | length) > 0 then bad("a monitor has a scale of zero or less")
    elif ([ .monitors[] | select((.mode // "") | test("^[0-9]+x[0-9]+@[0-9.]+$") | not) ] | length) > 0 then bad("a monitor mode is not WIDTHxHEIGHT@REFRESH")
    elif ([ .monitors[] | select(((.transform // 0) | tonumber? // -1) as $t | $t < 0 or $t > 7) ] | length) > 0 then bad("a monitor transform is outside 0-7")
    elif ((.waybar.mode // "all") | IN("all","primary","selected") | not) then bad("waybar.mode is not all, primary or selected")
    else "" end
  ' -- "$f" 2>&1 || printf 'unreadable layout')"
  if [[ -n $reason ]]; then ds_log "invalid layout: $reason"; return 1; fi
  return 0
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: PASS, 37 checks, 0 failed.

- [ ] **Step 5: Verify**

Run all suites. Expected: 0 failed.

---

### Task 4: Library — grid geometry

**Files:**
- Modify: `$S/lib_display_store.sh`
- Modify: `$T/test-display-store.sh`

**Interfaces:**
- Consumes: Tasks 2–3.
- Produces:
  - `ds_logical_size <mode> <scale> <transform>` → stdout `"W H"`, rounded; exit 3 when scale ≤ 0.
  - `ds_apply_grid <layout.json>` → stdout: the layout with `x` and `y` recomputed from each monitor's `grid`. Monitors with no `grid` default to `{row:1, order:<array index+1>, valign:"top"}`. Disabled monitors take part in no row and keep `x`/`y` unchanged.
  - Rounding rule: positions accumulate un-rounded and are rounded only at output, so three fractional widths cannot drift into an overlap (Review Focus 4).

- [ ] **Step 1: Write the failing tests**

Append to `$T/test-display-store.sh` before `test_summary`:

```bash
assert_eq "scale 1 keeps the panel size"        '1920 1080' "$(ds_logical_size 1920x1080@60.00 1 0)"
assert_eq "scale 1.5 shrinks the logical size"  '1280 720'  "$(ds_logical_size 1920x1080@60.00 1.5 0)"
assert_eq "scale 1.25 on 2560x1440"             '2048 1152' "$(ds_logical_size 2560x1440@59.95 1.25 0)"
assert_eq "transform 1 swaps width and height"  '1080 1920' "$(ds_logical_size 1920x1080@60.00 1 1)"
assert_eq "transform 3 swaps too"               '1080 1920' "$(ds_logical_size 1920x1080@60.00 1 3)"
assert_eq "transform 2 does not swap"           '1920 1080' "$(ds_logical_size 1920x1080@60.00 1 2)"
check "scale 0 is refused" bash -c '! ds_logical_size 1920x1080@60.00 0 0'

# One row, mixed scales: the second monitor starts where the first logically ends.
cat >"$WORK/g1.json" <<'EOS'
{"monitors":[
 {"identity":"A","port":"DP-1","enabled":true,"mode":"1920x1080@60.00","scale":1.5,"transform":0,"grid":{"row":1,"order":1,"valign":"bottom"}},
 {"identity":"B","port":"DP-2","enabled":true,"mode":"1920x1080@119.88","scale":1,"transform":0,"grid":{"row":1,"order":2,"valign":"bottom"}}
],"primary":"B","waybar":{"mode":"all","outputs":[]}}
EOS
ds_apply_grid "$WORK/g1.json" >"$WORK/g1.out.json"
assert_json_eq "$WORK/g1.out.json" '.monitors[0].x' '0'
assert_json_eq "$WORK/g1.out.json" '.monitors[1].x' '1280'
assert_json_eq "$WORK/g1.out.json" '.monitors[0].y' '360'
assert_json_eq "$WORK/g1.out.json" '.monitors[1].y' '0'

# valign across a row whose tallest monitor is 1080 logical.
# NOTE: ds_apply_grid reads its argument twice, so never hand it /dev/stdin.
jq '.monitors[0].grid.valign = "top"'    "$WORK/g1.json" >"$WORK/g1top.in.json"
ds_apply_grid "$WORK/g1top.in.json" >"$WORK/g1top.json"
assert_json_eq "$WORK/g1top.json" '.monitors[0].y' '0'
jq '.monitors[0].grid.valign = "center"' "$WORK/g1.json" >"$WORK/g1ctr.in.json"
ds_apply_grid "$WORK/g1ctr.in.json" >"$WORK/g1ctr.json"
assert_json_eq "$WORK/g1ctr.json" '.monitors[0].y' '180'

# Two rows: the second row starts below the tallest monitor of the first.
cat >"$WORK/g2.json" <<'EOS'
{"monitors":[
 {"identity":"A","port":"DP-1","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":1,"order":1,"valign":"top"}},
 {"identity":"B","port":"DP-2","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":1,"order":2,"valign":"top"}},
 {"identity":"C","port":"DP-3","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":2,"order":1,"valign":"top"}},
 {"identity":"D","port":"DP-4","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":2,"order":2,"valign":"top"}}
],"primary":"A","waybar":{"mode":"all","outputs":[]}}
EOS
ds_apply_grid "$WORK/g2.json" >"$WORK/g2.out.json"
assert_json_eq "$WORK/g2.out.json" '[.monitors[].x] | join(",")' '0,1920,0,1920'
assert_json_eq "$WORK/g2.out.json" '[.monitors[].y] | join(",")' '0,0,1080,1080'

# A rotated monitor occupies 1080 logical width and sets its row's height to 1920.
cat >"$WORK/g3.json" <<'EOS'
{"monitors":[
 {"identity":"A","port":"DP-1","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":1,"order":1,"valign":"top"}},
 {"identity":"B","port":"DP-2","enabled":true,"mode":"1080x1920@60.00","scale":1,"transform":1,"grid":{"row":1,"order":2,"valign":"top"}},
 {"identity":"C","port":"DP-3","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":2,"order":1,"valign":"top"}}
],"primary":"A","waybar":{"mode":"all","outputs":[]}}
EOS
ds_apply_grid "$WORK/g3.json" >"$WORK/g3.out.json"
assert_json_eq "$WORK/g3.out.json" '.monitors[1].x' '1920'
assert_json_eq "$WORK/g3.out.json" '.monitors[2].y' '1920'

# Review Focus 4: fractional widths must not accumulate into an overlap.
cat >"$WORK/g4.json" <<'EOS'
{"monitors":[
 {"identity":"A","port":"DP-1","enabled":true,"mode":"1920x1080@60.00","scale":1.1,"transform":0,"grid":{"row":1,"order":1,"valign":"top"}},
 {"identity":"B","port":"DP-2","enabled":true,"mode":"1920x1080@60.00","scale":1.1,"transform":0,"grid":{"row":1,"order":2,"valign":"top"}},
 {"identity":"C","port":"DP-3","enabled":true,"mode":"1920x1080@60.00","scale":1.1,"transform":0,"grid":{"row":1,"order":3,"valign":"top"}}
],"primary":"A","waybar":{"mode":"all","outputs":[]}}
EOS
ds_apply_grid "$WORK/g4.json" >"$WORK/g4.out.json"
assert_json_eq "$WORK/g4.out.json" '[.monitors[].x] | join(",")' '0,1745,3491'
assert_eq "no overlap across three fractional widths" 'ok' \
  "$(jq -r 'if (.monitors[2].x - .monitors[1].x) >= 1745 then "ok" else "overlap" end' "$WORK/g4.out.json")"

# A disabled monitor takes no space in its row.
jq '.monitors[0].enabled = false' "$WORK/g2.json" >"$WORK/g5.in.json"
ds_apply_grid "$WORK/g5.in.json" >"$WORK/g5.json"
assert_json_eq "$WORK/g5.json" '.monitors[1].x' '0'
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: FAIL — `ds_logical_size: command not found`.

- [ ] **Step 3: Implement**

Append to `$S/lib_display_store.sh`:

```bash
# "WxH@R" scale transform -> "LOGICAL_W LOGICAL_H", rounded.
ds_logical_size() {
  local mode=$1 scale=$2 transform=${3:-0} wh w h
  wh=${mode%@*}; w=${wh%x*}; h=${wh#*x}
  awk -v w="$w" -v h="$h" -v s="$scale" -v t="$transform" 'BEGIN {
    if (s + 0 <= 0) exit 3
    lw = w / s; lh = h / s
    tm = t % 4
    if (tm == 1 || tm == 3) { tmp = lw; lw = lh; lh = tmp }
    printf "%d %d\n", int(lw + 0.5), int(lh + 0.5)
  }'
}

# Recompute x/y from each monitor's grid. Only enabled monitors occupy space.
ds_apply_grid() {
  local layout=$1 rows coords
  rows="$("$DS_JQ" -r '
    .monitors | to_entries[]
    | select(.value.enabled == true)
    | [ .key,
        (.value.grid.row   // 1),
        (.value.grid.order // (.key + 1)),
        (.value.grid.valign // "top"),
        .value.mode,
        (.value.scale // 1),
        (.value.transform // 0) ]
    | @tsv' -- "$layout")"

  coords="$(printf '%s\n' "$rows" | awk -F'\t' '
    NF >= 7 {
      n++; idx[n]=$1; row[n]=$2+0; ord[n]=$3+0; va[n]=$4
      split($5, m, "@"); split(m[1], d, "x")
      s = $6 + 0; tm = ($7 + 0) % 4
      lw = d[1]/s; lh = d[2]/s
      if (tm == 1 || tm == 3) { tmp = lw; lw = lh; lh = tmp }
      w[n] = lw; h[n] = lh
      if (lh > rowh[row[n]]) rowh[row[n]] = lh
      seen[row[n]] = 1
    }
    END {
      nr = 0
      for (r in seen) { nr++; rl[nr] = r + 0 }
      for (i = 1; i <= nr; i++)
        for (j = i + 1; j <= nr; j++)
          if (rl[j] < rl[i]) { t = rl[i]; rl[i] = rl[j]; rl[j] = t }
      ry = 0
      for (i = 1; i <= nr; i++) {
        r = rl[i]; c = 0
        for (k = 1; k <= n; k++) if (row[k] == r) { c++; mem[c] = k }
        for (a = 1; a <= c; a++)
          for (b = a + 1; b <= c; b++)
            if (ord[mem[b]] < ord[mem[a]]) { t = mem[a]; mem[a] = mem[b]; mem[b] = t }
        cx = 0
        for (a = 1; a <= c; a++) {
          k = mem[a]; y = ry
          if (va[k] == "center") y = ry + (rowh[r] - h[k]) / 2
          else if (va[k] == "bottom") y = ry + rowh[r] - h[k]
          printf "%s\t%d\t%d\n", idx[k], int(cx + 0.5), int(y + 0.5)
          cx += w[k]
        }
        ry += rowh[r]
      }
    }')"

  local map
  map="$(printf '%s\n' "$coords" | awk -F'\t' '
    BEGIN { printf "{" }
    NF == 3 { printf "%s\"%s\":[%s,%s]", (c++ ? "," : ""), $1, $2, $3 }
    END { printf "}" }')"

  "$DS_JQ" -c --argjson coords "$map" '
    .monitors = ( .monitors | to_entries | map(
        .value + ( ($coords[.key | tostring]) as $c
                   | if $c then { x: $c[0], y: $c[1] } else {} end )
      ) )
  ' -- "$layout"
}
```

Note for the implementer: `cx` accumulates the un-rounded logical width and is rounded only when printed — that is what makes Review Focus 4 pass. Do not round `w[k]` before adding it.

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: PASS, 60 checks, 0 failed.

- [ ] **Step 5: Verify**

Run all suites. Expected: 0 failed.

---

### Task 5: Library — resolution, generation, port assignment

**Files:**
- Modify: `$S/lib_display_store.sh`
- Modify: `$T/test-display-store.sh`

**Interfaces:**
- Consumes: Tasks 2–4.
- Produces:
  - `ds_generate <normalized.json> <store.json>` → stdout: a layout. Every connected monitor enabled; `mode` = greatest width×height then greatest refresh from `modes`, falling back to `<width>x<height>@<refreshRate rounded to 2dp>` when `modes` is empty (Review Focus 1); `scale`/`transform` inherited per the search order below, else `1`/`0`; single row ordered by ascending port; `primary` = first port; `waybar.mode` = `all`; `grid` filled in; `x`/`y` from `ds_apply_grid`.
  - `ds_inherit <identity> <store.json> <field>` → stdout: the inherited value or empty. Search order: fingerprints by monitor count descending, then key ascending; within each, its `default` layout if present else its layouts by name ascending; first hit wins.
  - `ds_assign_ports <layout.json> <normalized.json>` → stdout: the layout with each monitor's `port` set to the live port, dropping monitors whose identity is absent. Assignment: by identity; among duplicates by stored `port`; then remaining candidates in ascending port order.
  - `ds_layout_names <normalized.json> <store.json>` → stdout: the current fingerprint's layout names, one per line, ascending.
  - `ds_resolve <normalized.json> <store.json>` → stdout: `{"fingerprint":…,"name":…,"source":"stored"|"generated","layout":{…}}`. Exact fingerprint → its `default`; a `default` naming a missing layout → the first layout by name (Review Focus 2); no layouts at all or no fingerprint → generated with `name` = `"(auto)"`. An empty monitor list → prints nothing and exits 2 (Review Focus 5).

- [ ] **Step 1: Write the failing tests**

Append to `$T/test-display-store.sh` before `test_summary`:

```bash
# Exact fingerprint resolves to its default layout.
ds_resolve "$n_dual" "$F/store-v1.json" >"$WORK/r-dual.json"
assert_json_eq "$WORK/r-dual.json" '.source' 'stored'
assert_json_eq "$WORK/r-dual.json" '.name'   'Home'
assert_json_eq "$WORK/r-dual.json" '.layout.monitors | length' '2'

# A default naming a layout that no longer exists falls back to the first by name.
norm_missing="$(norm dual)"
ds_resolve "$norm_missing" "$F/store-missing-default.json" >"$WORK/r-md.json" || true
assert_json_eq "$WORK/r-md.json" '.source' 'generated'

cat >"$WORK/store-md-dual.json" <<EOS
{"version":1,"fingerprints":{"$(ds_fingerprint "$n_dual")":{"label":"X","default":"Gone","layouts":{
  "Alpha": $(jq -c '.fingerprints[].layouts.Home' "$F/store-v1.json"),
  "Beta":  $(jq -c '.fingerprints[].layouts.Home' "$F/store-v1.json")
}}}}
EOS
ds_resolve "$n_dual" "$WORK/store-md-dual.json" >"$WORK/r-md2.json"
assert_json_eq "$WORK/r-md2.json" '.source' 'stored'
assert_json_eq "$WORK/r-md2.json" '.name'   'Alpha'

# No match at all -> generated.
n_triple="$(norm triple)"
ds_resolve "$n_triple" "$F/store-v1.json" >"$WORK/r-triple.json"
assert_json_eq "$WORK/r-triple.json" '.source' 'generated'
assert_json_eq "$WORK/r-triple.json" '.name'   '(auto)'
assert_json_eq "$WORK/r-triple.json" '[.layout.monitors[].enabled] | unique | join(",")' 'true'
assert_json_eq "$WORK/r-triple.json" '.layout.waybar.mode' 'all'

# Generation picks the largest resolution, then the highest refresh.
ds_generate "$n_triple" "$F/store-empty.json" >"$WORK/gen-triple.json"
assert_json_eq "$WORK/gen-triple.json" '.monitors[] | select(.port == "DP-1") | .mode' '2560x1440@59.95'
assert_json_eq "$WORK/gen-triple.json" '.monitors[] | select(.port == "HDMI-A-1") | .mode' '1920x1080@144.00'
assert_json_eq "$WORK/gen-triple.json" '[.monitors[].port] | join(",")' 'DP-1,DP-3,HDMI-A-1'
assert_json_eq "$WORK/gen-triple.json" '[.monitors[].x] | join(",")' '0,2560,4480'

# Review Focus 1: no availableModes -> fall back to the current mode, never "@".
n_nm="$(norm nomodes)"
ds_generate "$n_nm" "$F/store-empty.json" >"$WORK/gen-nm.json"
assert_json_eq "$WORK/gen-nm.json" '.monitors[0].mode' '1920x1080@59.94'
check "generated layout with no modes still validates" ds_validate_layout "$WORK/gen-nm.json"

# Inheritance: the ASUS panel keeps scale 1 and the BOE panel keeps 1.5 from store-v1.
cat >"$WORK/norm-asus-only.json" <<EOS
$(jq -c '[ .[] | select(.identity | startswith("ASUSTek")) ]' "$n_dual")
EOS
ds_generate "$WORK/norm-asus-only.json" "$F/store-v1.json" >"$WORK/gen-inherit.json"
assert_json_eq "$WORK/gen-inherit.json" '.monitors[0].scale' '1'
assert_eq "inherited scale for the BOE panel" '1.5' \
  "$(ds_inherit 'BOE|0x0630|' "$F/store-v1.json" scale)"
assert_eq "inheritance misses cleanly for an unknown identity" '' \
  "$(ds_inherit 'Nobody|Nothing|' "$F/store-v1.json" scale)"

# Port assignment: identity wins over the stored port.
jq -c '.fingerprints[].layouts.Home' "$F/store-v1.json" >"$WORK/home.json"
jq -c '[ .[] | if .identity | startswith("ASUSTek") then .port = "HDMI-A-1" else . end ] | sort_by(.port)' \
  "$n_dual" >"$WORK/norm-moved.json"
ds_assign_ports "$WORK/home.json" "$WORK/norm-moved.json" >"$WORK/home-moved.json"
assert_json_eq "$WORK/home-moved.json" '.monitors[] | select(.identity | startswith("ASUSTek")) | .port' 'HDMI-A-1'

# Port assignment: twins are told apart by the stored port.
n_tw="$(norm twins-noserial)"
cat >"$WORK/twins-layout.json" <<EOS
{"monitors":[
 {"identity":"AOC|24G2|","port":"DP-1","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":1,"order":1,"valign":"top"}},
 {"identity":"AOC|24G2|","port":"DP-2","enabled":true,"mode":"1920x1080@60.00","scale":1,"transform":0,"grid":{"row":1,"order":2,"valign":"top"}}
],"primary":"AOC|24G2|","waybar":{"mode":"all","outputs":[]}}
EOS
ds_assign_ports "$WORK/twins-layout.json" "$n_tw" >"$WORK/twins-out.json"
assert_json_eq "$WORK/twins-out.json" '[.monitors[].port] | join(",")' 'DP-1,DP-2'

# Port assignment: an absent identity is dropped.
jq -c '[ .[] | select(.identity | startswith("ASUSTek") | not) ]' "$n_dual" >"$WORK/norm-laptop-only.json"
ds_assign_ports "$WORK/home.json" "$WORK/norm-laptop-only.json" >"$WORK/home-dropped.json"
assert_json_eq "$WORK/home-dropped.json" '.monitors | length' '1'

# Review Focus 5: an empty monitor list resolves to nothing, exit 2.
check "resolve refuses an empty monitor list" bash -c '! ds_resolve "'"$n_empty"'" "'"$F"'/store-empty.json" >/dev/null 2>&1'

assert_eq "layout names of the dual set" 'Home
Work' "$(ds_layout_names "$n_dual" "$F/store-v1.json")"
assert_eq "layout names of an unknown set" '' "$(ds_layout_names "$n_triple" "$F/store-v1.json")"
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: FAIL — `ds_resolve: command not found`.

- [ ] **Step 3: Implement**

Append to `$S/lib_display_store.sh`:

```bash
# Ordered list of "fingerprint<TAB>layout-name" to search for inherited settings.
ds_inherit() {
  local identity=$1 store=$2 field=$3
  "$DS_JQ" -r --arg id "$identity" --arg field "$field" '
    [ .fingerprints | to_entries[]
      | { key: .key,
          n: ((.value.layouts // {}) | length),
          count: (.key | split(" + ") | length),
          dflt: (.value.default // ""),
          layouts: (.value.layouts // {}) } ]
    | sort_by(-.count, .key)
    | map( . as $fp
           | ( [ $fp.dflt ] + ( $fp.layouts | keys | sort ) )
           | map(select(. != "" and ($fp.layouts | has(.))))
           | unique_by(.) as $names
           | [ $names[] | $fp.layouts[.] ] )
    | flatten
    | map( .monitors[]? | select(.identity == $id) | .[$field] )
    | map(select(. != null))
    | if length > 0 then (.[0] | tostring) else "" end
  ' -- "$store"
}

ds_generate() {
  local norm=$1 store=$2 tmp layout
  tmp="$(mktemp)"
  # Base layout: every monitor enabled, best mode, single row by port order.
  "$DS_JQ" -c '
    def best_mode:
      if ((.modes // []) | length) > 0
      then ( .modes
              | map( . as $m
                     | ($m | split("@")) as $p
                     | ($p[0] | split("x")) as $d
                     | { mode: $m,
                         area: (($d[0]|tonumber) * ($d[1]|tonumber)),
                         rr: ($p[1]|tonumber) } )
              | sort_by(-.area, -.rr) | .[0].mode )
      else ( (.width|tostring) + "x" + (.height|tostring) + "@"
             + ((.refreshRate * 100 | round) / 100 | tostring
                | if test("\\.") then (. + "00") else (. + ".00") end
                | capture("^(?<a>[0-9]+\\.[0-9]{2})").a) )
      end;
    { monitors: [ to_entries[] | .key as $i | .value |
        { identity, port, enabled: true,
          mode: best_mode, scale: 1, transform: 0,
          x: 0, y: 0,
          grid: { row: 1, order: ($i + 1), valign: "top" } } ],
      primary: (.[0].identity // ""),
      waybar: { mode: "all", outputs: [] } }
  ' -- "$norm" >"$tmp"

  # Inherit scale and transform per identity, then compute geometry.
  local ids id sc tr
  ids="$("$DS_JQ" -r '.monitors[].identity' -- "$tmp")"
  while IFS= read -r id; do
    [[ -n $id ]] || continue
    sc="$(ds_inherit "$id" "$store" scale)"
    tr="$(ds_inherit "$id" "$store" transform)"
    [[ -n $sc ]] && "$DS_JQ" -c --arg id "$id" --argjson v "$sc" \
      '.monitors |= map(if .identity == $id then .scale = $v else . end)' -- "$tmp" >"$tmp.n" && mv -- "$tmp.n" "$tmp"
    [[ -n $tr ]] && "$DS_JQ" -c --arg id "$id" --argjson v "$tr" \
      '.monitors |= map(if .identity == $id then .transform = $v else . end)' -- "$tmp" >"$tmp.n" && mv -- "$tmp.n" "$tmp"
  done <<<"$ids"

  layout="$(ds_apply_grid "$tmp")"
  rm -f -- "$tmp" "$tmp.n"
  printf '%s\n' "$layout"
}

ds_assign_ports() {
  local layout=$1 norm=$2
  "$DS_JQ" -c --slurpfile live "$norm" '
    ( $live[0] ) as $live
    | .monitors = (
        [ .monitors[] as $m
          | ( [ $live[] | select(.identity == $m.identity) ] ) as $cands
          | if ($cands | length) == 0 then empty
            elif ($cands | length) == 1 then ($m + { port: $cands[0].port })
            else ( ( [ $cands[] | select(.port == $m.port) ] ) as $exact
                   | if ($exact | length) > 0
                     then ($m + { port: $exact[0].port })
                     else ($m + { port: ($cands | sort_by(.port) | .[0].port) })
                     end )
            end ]
      )'  -- "$layout"
}

ds_layout_names() {
  local norm=$1 store=$2 fp
  fp="$(ds_fingerprint "$norm")"
  ds_store_read "$store" | "$DS_JQ" -r --arg fp "$fp" \
    '(.fingerprints[$fp].layouts // {}) | keys | sort | .[]'
}

ds_resolve() {
  local norm=$1 store=$2 fp fpdata name layout source
  [[ "$(ds_count "$norm")" -gt 0 ]] || { ds_log "no monitors connected; nothing to resolve"; return 2; }
  fp="$(ds_fingerprint "$norm")"
  fpdata="$(ds_store_read "$store" | "$DS_JQ" -c --arg fp "$fp" '.fingerprints[$fp] // null')"

  name=""
  if [[ $fpdata != null ]]; then
    name="$(printf '%s' "$fpdata" | "$DS_JQ" -r '
      (.default // "") as $d
      | if ($d != "" and ((.layouts // {}) | has($d))) then $d
        else ((.layouts // {}) | keys | sort | .[0] // "") end')"
  fi

  if [[ -n $name ]]; then
    source=stored
    local tmp; tmp="$(mktemp)"
    printf '%s' "$fpdata" | "$DS_JQ" -c --arg n "$name" '.layouts[$n]' >"$tmp"
    layout="$(ds_assign_ports "$tmp" "$norm")"
    rm -f -- "$tmp"
  else
    source=generated
    name='(auto)'
    layout="$(ds_generate "$norm" "$store")"
  fi

  "$DS_JQ" -cn --arg fp "$fp" --arg name "$name" --arg source "$source" \
    --argjson layout "$layout" \
    '{ fingerprint: $fp, name: $name, source: $source, layout: $layout }'
}
```

Implementer note: `ds_generate`'s `best_mode` fallback must produce exactly two decimal places so it matches the `^[0-9]+x[0-9]+@[0-9.]+$` rule and the fixture expectation `1920x1080@59.94`. If the `capture` expression proves awkward in the installed `jq`, replace that branch with an `awk`-formatted value — the required output is `printf "%.2f"` of `refreshRate`.

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: PASS, 85 checks, 0 failed.

- [ ] **Step 5: Verify**

Run all suites. Expected: 0 failed.

---

### Task 6: Library — capture and store writes

**Files:**
- Modify: `$S/lib_display_store.sh`
- Modify: `$T/test-display-store.sh`

**Interfaces:**
- Consumes: Tasks 2–5.
- Produces:
  - `ds_capture <normalized.json>` → stdout: a layout describing the live state. Enabled = not `disabled`; `mode` = `<width>x<height>@<refreshRate %.2f>`; `scale`, `transform`, `x`, `y` as reported; `grid` derived by clustering — monitors sorted by `y` then `x`, a new row whenever `y` differs from the previous monitor's `y` by more than 1, `valign` `top`; `primary` = the identity of the `focused` monitor, else the first; `waybar.mode` = `all`.
  - `ds_store_set <store-path> <fingerprint> <layout-name> <layout.json>` → merges and writes atomically; creates the fingerprint when absent and sets `default` to this layout when the fingerprint had none.
  - `ds_store_default <store-path> <fingerprint> <layout-name>`, `ds_store_delete <store-path> <fingerprint> <layout-name>` (also clears or re-points `default`, and removes the fingerprint when its last layout goes), `ds_store_label <store-path> <fingerprint> <text>`.
  - All four refuse to write unless `ds_store_writable` passes, returning 1 with a logged reason.
  - `ds_store_backup <store-path>` → copies to `<store-path>.bak-<YYYYmmdd-HHMMSS>`, prints the backup path.

- [ ] **Step 1: Write the failing tests**

Append to `$T/test-display-store.sh` before `test_summary`:

```bash
# Capture reads the live state, including a disabled monitor.
jq -c '[ .[] | if .identity | startswith("ASUSTek") then .disabled = true else . end ]' "$n_dual" \
  >"$WORK/norm-dp-off.json"
ds_capture "$WORK/norm-dp-off.json" >"$WORK/cap.json"
assert_json_eq "$WORK/cap.json" '.monitors[] | select(.identity | startswith("ASUSTek")) | .enabled' 'false'
assert_json_eq "$WORK/cap.json" '.monitors[] | select(.identity | startswith("BOE")) | .enabled' 'true'
check "a capture validates" ds_validate_layout "$WORK/cap.json"

ds_capture "$n_dual" >"$WORK/cap2.json"
assert_json_eq "$WORK/cap2.json" '.monitors[] | select(.port == "DP-2") | .mode' '1920x1080@119.88'
assert_json_eq "$WORK/cap2.json" '.monitors[] | select(.port == "eDP-1") | .scale' '1.5'
assert_json_eq "$WORK/cap2.json" '.primary' 'ASUSTek COMPUTER INC|VG249Q3A|S7LMBS003105'
assert_json_eq "$WORK/cap2.json" '[.monitors[] | .grid.row] | unique | join(",")' '1'

# Two rows are recovered from the y coordinates.
jq -c '[ .[] | if .port == "eDP-1" then (.x = 0 | .y = 1080) else . end ]' "$n_dual" >"$WORK/norm-stack.json"
ds_capture "$WORK/norm-stack.json" >"$WORK/cap3.json"
assert_json_eq "$WORK/cap3.json" '[.monitors[] | .grid.row] | unique | sort | join(",")' '1,2'

# Writes.
cp "$F/store-empty.json" "$WORK/store.json"
FP="$(ds_fingerprint "$n_dual")"
check "set writes a layout" ds_store_set "$WORK/store.json" "$FP" 'Home' "$WORK/cap2.json"
assert_eq "the first layout written becomes the default" 'Home' \
  "$(jq -r --arg fp "$FP" '.fingerprints[$fp].default' "$WORK/store.json")"
assert_eq "the layout is retrievable" '2' \
  "$(jq -r --arg fp "$FP" '.fingerprints[$fp].layouts.Home.monitors | length' "$WORK/store.json")"

check "set writes a second layout" ds_store_set "$WORK/store.json" "$FP" 'Work' "$WORK/cap.json"
assert_eq "the default is unchanged by a second write" 'Home' \
  "$(jq -r --arg fp "$FP" '.fingerprints[$fp].default' "$WORK/store.json")"

check "default can be moved" ds_store_default "$WORK/store.json" "$FP" 'Work'
assert_eq "the default moved" 'Work' \
  "$(jq -r --arg fp "$FP" '.fingerprints[$fp].default' "$WORK/store.json")"

check "label can be set" ds_store_label "$WORK/store.json" "$FP" 'Nha'
assert_eq "the label was stored" 'Nha' \
  "$(jq -r --arg fp "$FP" '.fingerprints[$fp].label' "$WORK/store.json")"

check "delete removes a layout" ds_store_delete "$WORK/store.json" "$FP" 'Work'
assert_eq "deleting the default re-points it" 'Home' \
  "$(jq -r --arg fp "$FP" '.fingerprints[$fp].default' "$WORK/store.json")"

check "delete removes the last layout and its fingerprint" ds_store_delete "$WORK/store.json" "$FP" 'Home'
assert_eq "the fingerprint is gone" '0' \
  "$(jq -r '.fingerprints | length' "$WORK/store.json")"

# A malformed store is never written to.
cp "$F/store-malformed.json" "$WORK/bad-store.json"
before="$(cat "$WORK/bad-store.json")"
check "set refuses a malformed store" bash -c '! ds_store_set "'"$WORK"'/bad-store.json" "X" "Y" "'"$WORK"'/cap2.json" 2>/dev/null'
assert_eq "the malformed store was left byte-identical" "$before" "$(cat "$WORK/bad-store.json")"

cp "$F/store-future.json" "$WORK/future-store.json"
check "set refuses a future store" bash -c '! ds_store_set "'"$WORK"'/future-store.json" "X" "Y" "'"$WORK"'/cap2.json" 2>/dev/null'

# Backup.
cp "$F/store-v1.json" "$WORK/tobackup.json"
bak="$(ds_store_backup "$WORK/tobackup.json")"
check "the backup exists" test -f "$bak"
assert_eq "the backup is identical" 'same' \
  "$(cmp -s "$WORK/tobackup.json" "$bak" && echo same || echo different)"

# Writes are atomic: no leftover temp files.
assert_eq "no temp files left behind" '0' \
  "$(find "$WORK" -maxdepth 1 -name '*.tmp*' | wc -l | tr -d ' ')"
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: FAIL — `ds_capture: command not found`.

- [ ] **Step 3: Implement**

Append to `$S/lib_display_store.sh`:

```bash
ds_capture() {
  "$DS_JQ" -c '
    ( [ .[] | select(.disabled != true) ] | sort_by(.y, .x) ) as $on
    | ( reduce $on[] as $m ( { rows: {}, n: 0, prev: null };
          if (.prev == null) or (($m.y - .prev) | fabs > 1)
          then { rows: (.rows + { ($m.port): (.n + 1) }), n: (.n + 1), prev: $m.y }
          else { rows: (.rows + { ($m.port): .n }),       n: .n,       prev: .prev }
          end ) ).rows as $rowof
    | ( [ .[] | select(.focused == true) | .identity ] | .[0] ) as $focus
    | { monitors: [ to_entries[] | .key as $i | .value |
          { identity, port,
            enabled: (.disabled != true),
            mode: ((.width|tostring) + "x" + (.height|tostring) + "@"
                   + (.refreshRate * 100 | round | . / 100
                      | tostring | if test("\\.") then . else . + ".0" end)),
            scale: .scale, transform: (.transform // 0),
            x: .x, y: .y,
            grid: { row: ($rowof[.port] // 1),
                    order: ($i + 1),
                    valign: "top" } } ],
        primary: ($focus // (.[0].identity // "")),
        waybar: { mode: "all", outputs: [] } }
  ' -- "$1" | "$DS_JQ" -c '
      .monitors |= ( . as $all
        | [ group_by(.grid.row)[] | sort_by(.x) | to_entries[]
            | .value * { grid: (.value.grid + { order: (.key + 1) }) } ] )'
}

ds__write() {  # stdin: new store JSON; $1: path
  local path=$1 tmp
  tmp="$(mktemp -- "$(dirname -- "$path")/.display-layouts.XXXXXX")"
  if cat >"$tmp" && "$DS_JQ" -e . -- "$tmp" >/dev/null; then
    mv -f -- "$tmp" "$path"
    return 0
  fi
  rm -f -- "$tmp"
  ds_log "refusing to write invalid JSON to $path"
  return 1
}

ds__guard() {
  ds_store_writable "$1" && return 0
  ds_log "store is $(ds_store_status "$1"); refusing to write $1"
  return 1
}

ds_store_set() {
  local store=$1 fp=$2 name=$3 layout=$4
  ds__guard "$store" || return 1
  ds_store_read "$store" | "$DS_JQ" -c --arg fp "$fp" --arg n "$name" \
    --slurpfile l "$layout" '
      .fingerprints[$fp] //= { label: "", default: "", layouts: {} }
      | .fingerprints[$fp].layouts[$n] = $l[0]
      | if (.fingerprints[$fp].default // "") == ""
        then .fingerprints[$fp].default = $n else . end
    ' | ds__write "$store"
}

ds_store_default() {
  local store=$1 fp=$2 name=$3
  ds__guard "$store" || return 1
  ds_store_read "$store" | "$DS_JQ" -c --arg fp "$fp" --arg n "$name" '
      if (.fingerprints[$fp].layouts // {}) | has($n)
      then .fingerprints[$fp].default = $n
      else error("no such layout: " + $n) end
    ' | ds__write "$store"
}

ds_store_delete() {
  local store=$1 fp=$2 name=$3
  ds__guard "$store" || return 1
  ds_store_read "$store" | "$DS_JQ" -c --arg fp "$fp" --arg n "$name" '
      del(.fingerprints[$fp].layouts[$n])
      | if ((.fingerprints[$fp].layouts // {}) | length) == 0
        then del(.fingerprints[$fp])
        elif (.fingerprints[$fp].default // "") == $n
        then .fingerprints[$fp].default = (.fingerprints[$fp].layouts | keys | sort | .[0])
        else . end
    ' | ds__write "$store"
}

ds_store_label() {
  local store=$1 fp=$2 text=$3
  ds__guard "$store" || return 1
  ds_store_read "$store" | "$DS_JQ" -c --arg fp "$fp" --arg t "$text" '
      .fingerprints[$fp] //= { label: "", default: "", layouts: {} }
      | .fingerprints[$fp].label = $t
    ' | ds__write "$store"
}

ds_store_backup() {
  local store=$1 bak
  bak="$store.bak-$(date +%Y%m%d-%H%M%S)"
  cp -- "$store" "$bak" && printf '%s\n' "$bak"
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-store.sh`
Expected: PASS, 108 checks, 0 failed.

- [ ] **Step 5: Verify**

Run all suites. Expected: 0 failed. Deploy the library: `cp config/hypr/scripts/lib_display_store.sh ~/.config/hypr/scripts/` and `diff` to confirm.

---

### Task 7: Controller — N-monitor apply, verification, rescue, CLI

**Files:**
- Modify: `$S/DisplayProfile.sh` (rewrite; keep the file path and the GPL header)
- Modify: `$T/test-display-profile.sh`

**Interfaces:**
- Consumes: Task 6 library (all `ds_*`).
- Produces:
  - CLI exactly as the spec's "CLI" block, with `--` guards at every call site so a layout name such as `-n` is data (Review Focus 3).
  - `dp_apply_layout <layout.json>` — the 8-step apply algorithm.
  - `dp_rescue` — the rescue layout, built from live monitors only, ignoring the store.
  - `dp_state_write <fingerprint> <name>` / `dp_state_read` — the `current` state file, two lines.
  - `dp_monitors` — `hyprctl -j monitors all` into a temp file, normalized; prints the path.
  - New seams added to the existing set: `DP_STORE_FILE`, `DP_MONITORS_LUA`, `DP_NWG_DISPLAYS`, `DP_SYSTEMD_RUN`, `DP_ROFI` (default `rofi`), `DP_SCRIPT` (the controller's own path, used by the menu and the table).
  - `dp_stand_down <normalized.json>` — the single implementation of the coexistence rule; the watcher calls the controller and never re-implements it.
  - `dp_require_tools` — `jq` missing is a hard failure with a clear message, per the spec's error table.
  - Retained from the current script and reused unchanged: `hypr_eval`, `notify`, `die`, `usage`, `wants_dry_run`, and the explicit `exec 9>` + `flock -w 10 9` tail.
  - Waybar is called through `dp_waybar_apply <layout.json>`, implemented in Task 8; in this task it is a stub that logs and returns 0.

- [ ] **Step 1: Write the failing tests**

Rewrite `$T/test-display-profile.sh` around the fixtures. The checks required in this task:

```bash
# Enable before disable, with the laptop panel as the one being switched off.
run_dp() {  # $1 fixture, $2.. args
  local fx=$1; shift
  FAKE_HYPRCTL_MONITORS_FILE="$WORK/$fx.json" \
  DP_STORE_FILE="$WORK/store.json" \
  DP_MONITORS_LUA="$WORK/monitors.lua" \
  DP_RUNTIME_DIR="$WORK/run" DP_SKIP_LOCK=1 \
  timeout 30 bash "$SCRIPT" "$@"
}

# store.json: dual set, layout "Off1" disables the BOE panel, enables the ASUS one.
assert_order "enabled monitors are configured before any monitor is disabled" \
  "$FAKE_LOG" 'hl.monitor({ output = "DP-2"' 'disabled = true'

assert_count_re "exactly one disable call" "$FAKE_LOG" 'disabled = true' 1

# A failed verification must disable nothing.
# FAKE_HYPRCTL_MONITORS_FILE answers the verification read with the pre-apply state.
check "a failed verification disables nothing" bash -c '! run_dp dual-unverifiable Off1'
assert_count_re "no disable call after a failed verification" "$FAKE_LOG" 'disabled = true' 0

# Zero active monitors after disabling triggers the rescue layout.
assert_contains "the rescue layout re-enables every monitor" "$FAKE_LOG" 'rescue'
assert_count_re "rescue enables both monitors" "$FAKE_LOG" 'hl.monitor\(\{ output = "(DP-2|eDP-1)", mode' 2

# Focus goes to the layout's primary.
assert_contains "focus moves to the primary monitor" "$FAKE_LOG" \
  'hl.dispatch(hl.dsp.focus({ monitor = "DP-2" }))'

# Review Focus 5: no monitors at all -> stand down, exit 0, no eval calls.
check "an empty monitor list exits 0" run_dp empty auto
assert_count_re "no hl.monitor calls with no monitors" "$FAKE_LOG" 'hl\.monitor' 0

# Review Focus 3: a layout named -n is treated as data.
check "a layout named -n applies" run_dp dual -- -n
assert_contains "the -n layout was applied" "$FAKE_LOG" 'applying layout -n'

# Structural: every child spawn closes fd 9.
check "every background spawn closes fd 9" bash -c '
  bad=$(grep -nE "(^|[^0-9])&[[:space:]]*$|systemd-run|socat|nwg-displays" "'"$SCRIPT"'" \
        | grep -v "9>&-" | grep -vE "^[0-9]+:[[:space:]]*#") ; [[ -z $bad ]]'

check "no eval anywhere" bash -c '! sed "s/#.*$//" "'"$SCRIPT"'" | grep -nE "\beval\b"'
check "no hardcoded output names" bash -c '! grep -nE "\b(eDP|DP|HDMI-A|DVI-D|Eval)-[0-9]" "'"$SCRIPT"'"'
check "dry-run takes no lock" bash -c 'grep -A3 "wants_dry_run" "'"$SCRIPT"'" | grep -q "DP_SKIP_LOCK\|dry"'

# jq missing is a hard failure with a readable message.
out="$(DP_JQ=/nonexistent/jq run_dp dual auto 2>&1 || true)"
assert_contains_str "a missing jq fails loudly" "$out" 'is required'
check "a missing jq exits non-zero" bash -c '! DP_JQ=/nonexistent/jq run_dp dual auto >/dev/null 2>&1'

# Stand-down is implemented here, not in the watcher.
printf 'hl.monitor({ output = "", mode = "preferred" })\n' >"$WORK/monitors.lua"
cp "$F/store-empty.json" "$WORK/store.json"
run_dp dual auto
assert_count_re "stand down means no eval calls" "$FAKE_LOG" 'hl\.monitor' 0
printf -- '-- hl.monitor({ output = "" })\n' >"$WORK/monitors.lua"
run_dp dual auto
check "a commented hl.monitor does not stand down" bash -c 'grep -q "hl.monitor" "'"$FAKE_LOG"'"'

# Stale generated Waybar configs from the previous implementation are removed.
touch "$WORK/run/kooldots-display-profiles/config-home" "$WORK/run/kooldots-display-profiles/config-work"
run_dp dual auto
check "config-home was removed" bash -c '! test -f "'"$WORK"'/run/kooldots-display-profiles/config-home"'
check "config-work was removed" bash -c '! test -f "'"$WORK"'/run/kooldots-display-profiles/config-work"'
```

Write the fixture `$WORK/dual-unverifiable.json` inside the test as a copy of `monitors-dual.json` whose ASUS entry reports `"x": 0` so the post-apply verification cannot match the layout's `x: 1280`.

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-profile.sh`
Expected: FAIL on every new check — the controller still speaks the old two-profile vocabulary.

- [ ] **Step 3: Rewrite the controller**

Replace the configuration block and `main` of `$S/DisplayProfile.sh`. Key pieces:

```bash
config_home="${XDG_CONFIG_HOME:-$HOME/.config}"
DP_STORE_FILE="${DP_STORE_FILE:-$config_home/hypr/UserConfigs/display-layouts.json}"
DP_MONITORS_LUA="${DP_MONITORS_LUA:-$config_home/hypr/UserConfigs/monitors.lua}"
DP_NWG_DISPLAYS="${DP_NWG_DISPLAYS:-nwg-displays}"
DP_SYSTEMD_RUN="${DP_SYSTEMD_RUN:-systemd-run}"
# shellcheck source=./lib_display_store.sh
source "$(dirname -- "${BASH_SOURCE[0]}")/lib_display_store.sh"

log() { printf '%s %s\n' "$(date '+%F %T')" "$*" >&2; printf '%s %s\n' "$(date '+%F %T')" "$*" >>"$DP_LOG_FILE" 2>/dev/null || true; }

dp_monitors() {
  local f="$STATE_DIR/monitors.json" raw="$STATE_DIR/monitors.raw.json"
  mkdir -p -- "$STATE_DIR"
  "$DP_HYPRCTL" -j monitors all >"$raw" 2>/dev/null || { log "ERROR: cannot read monitors"; return 1; }
  ds_normalize "$raw" >"$f" || return 1
  printf '%s\n' "$f"
}

monitor_lua() {  # identity-free: port, mode, position, scale, transform
  printf 'hl.monitor({ output = "%s", mode = "%s", position = "%sx%s", scale = "%s", transform = %s })' \
    "$1" "$2" "$3" "$4" "$5" "$6"
}

dp_verify_monitor() {  # port mode x y scale
  local port=$1 mode=$2 x=$3 y=$4 scale=$5 w rr
  w=${mode%@*}; w=${w%x*}; rr=${mode#*@}
  "$DP_HYPRCTL" -j monitors | "$DP_JQ" -e --arg p "$port" --argjson w "$w" \
    --argjson x "$x" --argjson y "$y" --argjson s "$scale" --argjson rr "$rr" '
      any(.[]; .name == $p and .width == $w and .x == $x and .y == $y
               and ((.scale - $s) | fabs) < 0.01
               and ((.refreshRate - $rr) | fabs) < 0.5)' >/dev/null
}

dp_apply_layout() {
  local layout=$1 enabled disabled port mode x y scale transform primary
  ds_validate_layout "$layout" || return 1

  # Step 2: configure every enabled monitor.
  while IFS=$'\t' read -r port mode x y scale transform; do
    [[ -n $port ]] || continue
    hypr_eval "$(monitor_lua "$port" "$mode" "$x" "$y" "$scale" "$transform")" || return 1
  done < <("$DP_JQ" -r '.monitors[] | select(.enabled == true)
            | [ .port, .mode, .x, .y, .scale, (.transform // 0) ] | @tsv' -- "$layout")

  # Step 3: verify all of them; disable nothing on a mismatch.
  while IFS=$'\t' read -r port mode x y scale transform; do
    [[ -n $port ]] || continue
    dp_verify_monitor "$port" "$mode" "$x" "$y" "$scale" || {
      log "ERROR: $port did not come up as $mode at ${x}x${y} scale $scale; disabling nothing"
      notify "Display profile" "Monitor $port did not apply; nothing was disabled"
      return 1
    }
  done < <("$DP_JQ" -r '.monitors[] | select(.enabled == true)
            | [ .port, .mode, .x, .y, .scale, (.transform // 0) ] | @tsv' -- "$layout")

  # Step 4: disable the rest.
  while IFS= read -r port; do
    [[ -n $port ]] || continue
    log "disabling $port"
    hypr_eval "hl.monitor({ output = \"$port\", disabled = true })" || true
  done < <("$DP_JQ" -r '.monitors[] | select(.enabled != true) | .port' -- "$layout")

  # Step 5: at least one monitor must be active.
  if ! "$DP_HYPRCTL" -j monitors | "$DP_JQ" -e 'length > 0' >/dev/null; then
    log "ERROR: zero active outputs after applying; entering rescue"
    dp_rescue
    return 1
  fi

  # Step 6: focus the primary.
  primary="$("$DP_JQ" -r --arg p "$("$DP_JQ" -r '.primary // ""' -- "$layout")" \
    '.monitors[] | select(.identity == $p and .enabled == true) | .port' -- "$layout" | head -n1)"
  [[ -n $primary ]] && hypr_eval "hl.dispatch(hl.dsp.focus({ monitor = \"$primary\" }))" || true

  # Step 7: Waybar (Task 8).
  dp_waybar_apply "$layout" || log "WARN: Waybar step failed; display state is applied"
  return 0
}

dp_rescue() {
  local norm port mode
  log "rescue: enabling every connected monitor in one row at scale 1"
  norm="$(dp_monitors)" || return 1
  local x=0 w
  while IFS=$'\t' read -r port mode w; do
    [[ -n $port ]] || continue
    hypr_eval "$(monitor_lua "$port" "$mode" "$x" 0 1 0)" || true
    x=$(( x + w ))
  done < <("$DP_JQ" -r '.[] | [ .port,
      ((.width|tostring) + "x" + (.height|tostring) + "@" + ((.refreshRate*100|round)/100|tostring)),
      .width ] | @tsv' -- "$norm")
}
```

CLI dispatch — note the `--` guards and that an unknown verb is treated as a layout name:

```bash
main() {
  local arg verb="" name=""
  for arg in "$@"; do [[ $arg == "--dry-run" ]] && DRY_RUN=1; done
  case "${1-}" in
    ""|--dry-run) verb=auto ;;
    --) shift; verb=layout; name=${1-} ;;
    auto|list|waybar-config) verb=$1 ;;
    capture|set-default|delete|label) verb=$1; name=${2-} ;;
    -h|--help) usage; return 0 ;;
    *) verb=layout; name=$1 ;;
  esac
  ...
}
```

For `verb=layout`: validate the name with `ds_name_valid "$name"` (the function takes the name as `$1`; a `--` would itself be validated as the name), resolve the current fingerprint, fetch that layout, `ds_assign_ports`, log `applying layout <name>`, then `dp_apply_layout`.

For `verb=auto`: stand-down check (Task 9 shares this function — implement `dp_stand_down` here and call it from both), `ds_resolve`, log `auto -> <name> (<source>)`, `dp_apply_layout`, `dp_state_write`.

Add a `dp_waybar_apply() { log "waybar: not implemented yet"; return 0; }` stub to be replaced in Task 8.

The tool guard and the coexistence rule, both called from `main` before anything else:

```bash
dp_require_tools() {
  command -v "$DP_JQ" >/dev/null 2>&1 || {
    printf 'DisplayProfile.sh: %s is required and was not found in PATH.\n' "$DP_JQ" >&2
    printf 'Install jq (it is already a KooLDots dependency) and run this again.\n' >&2
    exit 1
  }
}

# The coexistence rule, implemented once. Returns 0 when this system must NOT act.
dp_stand_down() {
  local norm=$1 fp
  [[ -f $DP_MONITORS_LUA ]] || return 1
  grep -qE '^[[:space:]]*hl\.monitor[[:space:]]*\(' -- "$DP_MONITORS_LUA" || return 1
  fp="$(ds_fingerprint "$norm")"
  if ds_store_read "$DP_STORE_FILE" \
       | "$DP_JQ" -e --arg fp "$fp" '(.fingerprints[$fp].layouts // {}) | length > 0' >/dev/null; then
    return 1   # a stored layout is an explicit handover; we own this set
  fi
  log "standing down: $DP_MONITORS_LUA configures monitors itself and no layout is saved for this set"
  return 0
}
```

`grep -E '^[[:space:]]*hl\.monitor'` is what makes a commented `-- hl.monitor(...)` line harmless: the comment marker precedes the call, so the anchored pattern does not match.

Remove the previous implementation's stale generated Waybar configs on first run, as the spec's "State and logging" section requires:

```bash
rm -f -- "$STATE_DIR/config-home" "$STATE_DIR/config-work"
```

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-profile.sh`
Expected: PASS, 0 failed.

- [ ] **Step 5: Verify against the live session, read-only**

Run: `~/.config/hypr/scripts/DisplayProfile.sh --dry-run auto` after deploying.
Expected: a `DRY:` plan naming the two real monitors with no `hyprctl eval` executed, and `hyprctl -j monitors` unchanged afterwards.

- [ ] **Step 6: Verify**

Run all suites. Expected: 0 failed.

---

### Task 8: Controller — Waybar for N monitors

**Files:**
- Modify: `$S/DisplayProfile.sh`
- Modify: `$T/test-display-profile.sh`
- Create: `$F/waybar-config-2bars.json` (reuse the existing fixture if present), `$F/waybar-config-output-array.json`, `$F/waybar-config-no-layer.json`

**Interfaces:**
- Consumes: Task 7 (`dp_waybar_apply` stub, `log`, seams).
- Produces:
  - `dp_waybar_outputs <layout.json>` → stdout: the ports the bars must appear on, one per line; empty for `all`.
  - `dp_waybar_generate <layout.json>` → stdout: the config path Waybar should use — `$DP_WAYBAR_CONFIG` for `all`, otherwise `$STATE_DIR/config-active` with an `"output"` array injected into every bar.
  - `dp_waybar_apply <layout.json>` → generates, compares path **and** content against the running instance, restarts only when either differs.
  - `waybar-config` CLI verb → prints `dp_waybar_generate` for the current layout and exits; read-only, takes no lock.
  - Restart uses `$DP_SYSTEMD_RUN --user --scope --quiet --unit=...` when available, else `( ... & )`, always with `9>&-`.

- [ ] **Step 1: Write the failing tests**

Append to `$T/test-display-profile.sh`:

```bash
# $WORK/store.json for this task holds four layouts for the dual fingerprint:
# AllBars (waybar.mode all), Home (selected -> the ASUS identity),
# BothBars (selected -> both identities), PrimaryBar (primary, primary = ASUS).
run_dp dual -- AllBars
assert_eq "mode all generates nothing" "$WORK/waybar/config" \
  "$(run_dp dual waybar-config)"

# mode selected injects an array of the chosen ports
cfg="$(run_dp dual waybar-config)"   # layout "Home": selected -> the ASUS identity
assert_eq "selected mode uses config-active" "$WORK/run/kooldots-display-profiles/config-active" "$cfg"
assert_count_re 'every bar got an output key' "$cfg" '"output"' 2
assert_contains "the output key is an array of the ASUS port" "$cfg" '"output": ["DP-2"]'
assert_not_contains "the output key is not a bare string" "$cfg" '"output": "DP-2"'

# Two selected monitors: layout "BothBars" selects both identities, in the
# order the ports sort, so the injected array is deterministic.
cfg2="$(run_dp dual -- BothBars && run_dp dual waybar-config)"
assert_contains "two selected ports appear in order" "$cfg2" '"output": ["DP-2", "eDP-1"]'

# A pre-existing output key, single-line and multi-line, is replaced not duplicated.
cp "$F/waybar-config-output-array.json" "$WORK/waybar/config"
cfg3="$(run_dp dual waybar-config)"
assert_count_re "no duplicate output keys" "$cfg3" '"output"' 2

# primary mode resolves through the layout's primary identity.
cp "$F/waybar-config-2bars.json" "$WORK/waybar/config"
cfg4="$(run_dp dual -- PrimaryBar && run_dp dual waybar-config)"
assert_contains "primary mode restricts to the primary port" "$cfg4" '"output": ["DP-2"]'

# A config with no "layer" key falls back to all, with a warning.
cp "$F/waybar-config-no-layer.json" "$WORK/waybar/config"
cfg5="$(run_dp dual waybar-config)"
assert_contains "no-layer config warns" "$FAKE_LOG" 'WARN'
assert_eq "no-layer config falls back to the user config" "$WORK/waybar/config" "$cfg5"

# restart suppression: identical content and path -> no restart
assert_count_re "waybar is not restarted when nothing changed" "$FAKE_LOG" '^(systemd-run|waybar) ' 0
# changed content -> exactly one restart, with style
assert_count_re "exactly one restart on change" "$FAKE_LOG" 'waybar -c .* -s ' 1
assert_count_re "no stray second waybar" "$FAKE_LOG" '^waybar ' 1

# the generated path never appears on stdout mixed with a log line
assert_eq "waybar-config prints exactly one line" '1' "$(run_dp dual waybar-config | wc -l | tr -d ' ')"

check "systemd-run is preferred when present" bash -c 'grep -q "systemd-run" "'"$SCRIPT"'"'
check "the waybar spawn closes fd 9" bash -c 'grep -nE "waybar|systemd-run" "'"$SCRIPT"'" | grep -q "9>&-"'
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-profile.sh`
Expected: FAIL — `waybar: not implemented yet` in the log and no `config-active`.

- [ ] **Step 3: Implement**

Replace the stub in `$S/DisplayProfile.sh`:

```bash
dp_waybar_outputs() {
  local layout=$1 mode
  mode="$("$DP_JQ" -r '.waybar.mode // "all"' -- "$layout")"
  case $mode in
    all) : ;;
    primary) "$DP_JQ" -r '.primary as $p | .monitors[]
              | select(.identity == $p and .enabled == true) | .port' -- "$layout" ;;
    selected) "$DP_JQ" -r '(.waybar.outputs // []) as $sel | .monitors[]
              | select(.enabled == true and (.identity | IN($sel[]))) | .port' -- "$layout" ;;
  esac
}

dp_waybar_generate() {
  local layout=$1 ports json src dst
  mapfile -t ports < <(dp_waybar_outputs "$layout")
  if (( ${#ports[@]} == 0 )); then printf '%s\n' "$DP_WAYBAR_CONFIG"; return 0; fi

  src="$(readlink -f -- "$DP_WAYBAR_CONFIG")"
  dst="$STATE_DIR/config-active"
  json="$(printf '%s\n' "${ports[@]}" | "$DP_JQ" -R -s -c 'split("\n") | map(select(length > 0))')"

  mkdir -p -- "$STATE_DIR"
  awk -v out="$json" '
      skipping { if ($0 ~ /\]/) { skipping = 0 } next }
      /^[[:space:]]*"output"[[:space:]]*:/ {
        if ($0 ~ /\[/ && $0 !~ /\]/) { skipping = 1 }
        next
      }
      { print }
      /^[[:space:]]*"layer"[[:space:]]*:/ { printf "\"output\": %s,\n", out; n++ }
      END { if (n == 0) exit 3 }
    ' -- "$src" >"$dst.tmp" || {
      log "WARN: no \"layer\" key found in $src; Waybar will not be restricted"
      rm -f -- "$dst.tmp"
      printf '%s\n' "$DP_WAYBAR_CONFIG"
      return 0
    }
  mv -f -- "$dst.tmp" "$dst"
  printf '%s\n' "$dst"
}

dp_waybar_apply() {
  local layout=$1 cfg running
  cfg="$(dp_waybar_generate "$layout")" || return 1
  running="$("$DP_PGREP" -af 'waybar' 2>/dev/null | sed -n 's/.*-c \([^ ]*\).*/\1/p' | head -n1 || true)"
  if [[ -n $running && $running == "$cfg" ]] && cmp -s -- "$running" "$cfg"; then
    log "waybar already runs $(basename -- "$cfg"); not restarting it"
    return 0
  fi
  log "restarting waybar with $(basename -- "$cfg")"
  "$DP_PKILL" -x waybar >/dev/null 2>&1 || true
  if command -v "$DP_SYSTEMD_RUN" >/dev/null 2>&1; then
    "$DP_SYSTEMD_RUN" --user --scope --quiet --unit="kooldots-waybar-$$-$RANDOM" \
      -- "$DP_WAYBAR" -c "$cfg" -s "$DP_WAYBAR_STYLE" >/dev/null 2>&1 9>&- &
  else
    ( "$DP_WAYBAR" -c "$cfg" -s "$DP_WAYBAR_STYLE" >"$STATE_DIR/waybar.log" 2>&1 9>&- & )
  fi
  return 0
}
```

Implementer note: `cmp -s -- "$running" "$cfg"` compares the same file with itself when the paths are equal, which is the point — the path check catches a mode switch and the content check catches a Waybar layout change under the same path. When `$running` differs from `$cfg`, restart without comparing content.

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-profile.sh`
Expected: PASS, 0 failed.

- [ ] **Step 5: Verify on the live session**

Run: `~/.config/hypr/scripts/DisplayProfile.sh waybar-config`
Expected: exactly one line, a readable path, and `pgrep -c waybar` still `1`.

- [ ] **Step 6: Verify**

Run all suites. Expected: 0 failed.

---

### Task 9: Watcher — pause, configreloaded, state, stand-down

**Files:**
- Modify: `$S/MonitorWatcher.sh`
- Modify: `$T/test-monitor-watcher.sh`
- Create: `$F/events-configreloaded`, `$F/events-pause-burst`

**Interfaces:**
- Consumes: Task 7 (`DisplayProfile.sh auto`, the state file format, `dp_stand_down`'s rule).
- Produces:
  - `mw_paused` → exit 0 while `$STATE_DIR/pause` exists and holds a live PID; removes the file and returns 1 when the PID is dead.
  - `configreloaded` handling: re-apply the **current** layout from the state file (`DisplayProfile.sh -- "<name>"`), not `auto`, unless the state file is absent in which case `auto`.
  - Stand-down: when `$DP_MONITORS_LUA` has a non-comment `hl.monitor(` line **and** the current fingerprint has no stored layout, log once per run and apply nothing.
  - Seams added: `MW_PAUSE_FILE`, `MW_STATE_FILE`, `MW_MONITORS_LUA`.

- [ ] **Step 1: Write the failing tests**

Append to `$T/test-monitor-watcher.sh`:

```bash
# A live PID in the pause file suppresses the apply.
sleep 60 & live=$!
printf '%s\n' "$live" >"$WORK/run/kooldots-display-profiles/pause"
run_watcher events-burst
kill "$live" 2>/dev/null || true
assert_count_re "no apply while paused" "$FAKE_LOG" 'DisplayProfile.sh' 0
assert_contains "the pause is logged" "$MW_LOG" 'paused'

# A dead PID is ignored and the file is removed.
printf '%s\n' '999999' >"$WORK/run/kooldots-display-profiles/pause"
run_watcher events-burst
assert_count_re "a stale pause file does not block the apply" "$FAKE_LOG" 'DisplayProfile.sh auto' 1
check "the stale pause file was removed" bash -c '! test -f "'"$WORK"'/run/kooldots-display-profiles/pause"'

# configreloaded re-applies the current layout, not the default.
printf 'FP\nWork\n' >"$WORK/run/kooldots-display-profiles/current"
run_watcher events-configreloaded
assert_contains "configreloaded re-applies the current layout" "$FAKE_LOG" 'DisplayProfile.sh -- Work'
assert_count_re "configreloaded does not call auto" "$FAKE_LOG" 'DisplayProfile.sh auto' 0

# with no state file, configreloaded falls back to auto
rm -f "$WORK/run/kooldots-display-profiles/current"
run_watcher events-configreloaded
assert_count_re "configreloaded falls back to auto" "$FAKE_LOG" 'DisplayProfile.sh auto' 1

# Stand-down when the user wrote their own monitors.lua and nothing is stored.
printf 'hl.monitor({ output = "", mode = "preferred" })\n' >"$WORK/monitors.lua"
cp "$F/store-empty.json" "$WORK/store.json"
run_watcher events-burst
assert_count_re "stand down: no apply" "$FAKE_LOG" 'DisplayProfile.sh' 0
assert_contains "stand down is logged once" "$MW_LOG" 'standing down'

# A commented hl.monitor does not trigger stand-down.
printf -- '-- hl.monitor({ output = "" })\n' >"$WORK/monitors.lua"
run_watcher events-burst
assert_count_re "a commented hl.monitor does not stand down" "$FAKE_LOG" 'DisplayProfile.sh auto' 1

# A stored layout for this fingerprint overrides the stand-down.
printf 'hl.monitor({ output = "", mode = "preferred" })\n' >"$WORK/monitors.lua"
cp "$WORK/store-with-dual.json" "$WORK/store.json"
run_watcher events-burst
assert_count_re "a stored layout wins over monitors.lua" "$FAKE_LOG" 'DisplayProfile.sh auto' 1

check "socat closes fd 9" bash -c 'grep -n "stream_events" "'"$SCRIPT"'" | grep -q "9>&-"'
```

`$F/events-configreloaded` contains a single line `configreloaded>>`.
`$F/events-pause-burst` contains three `monitoradded>>`/`monitorremoved>>` lines in quick succession.

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-monitor-watcher.sh`
Expected: FAIL — pause and `configreloaded` are unknown to the watcher.

- [ ] **Step 3: Implement**

In `$S/MonitorWatcher.sh`:

```bash
MW_PAUSE_FILE="${MW_PAUSE_FILE:-$STATE_DIR/pause}"
MW_STATE_FILE="${MW_STATE_FILE:-$STATE_DIR/current}"
MW_MONITORS_LUA="${MW_MONITORS_LUA:-${XDG_CONFIG_HOME:-$HOME/.config}/hypr/UserConfigs/monitors.lua}"

mw_paused() {
  local pid
  [[ -f $MW_PAUSE_FILE ]] || return 1
  pid="$(head -n1 -- "$MW_PAUSE_FILE" 2>/dev/null || true)"
  if [[ $pid =~ ^[0-9]+$ ]] && kill -0 "$pid" 2>/dev/null; then
    log "paused by pid $pid; ignoring monitor events"
    return 0
  fi
  log "removing a stale pause file (pid ${pid:-unknown} is gone)"
  rm -f -- "$MW_PAUSE_FILE"
  return 1
}
```

Add `configreloaded` to the relevant-event filter, and in the apply path:

```bash
apply_current() {
  local name=""
  [[ -f $MW_STATE_FILE ]] && name="$(sed -n '2p' -- "$MW_STATE_FILE" 2>/dev/null || true)"
  if [[ -n $name && $name != "(auto)" ]]; then
    "$MW_PROFILE_SCRIPT" -- "$name" 9>&- && return 0
    log "WARN: re-applying $name failed; falling back to auto"
  fi
  "$MW_PROFILE_SCRIPT" auto 9>&-
}
```

Route `monitoradded`/`monitorremoved*` to `apply_auto` and `configreloaded` to `apply_current`, both gated on `mw_paused` returning non-zero. The stand-down check lives in the controller (Task 7's `dp_stand_down`); the watcher only logs what the controller reports, so a single rule has a single implementation.

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-monitor-watcher.sh`
Expected: PASS, 0 failed.

- [ ] **Step 5: Verify on the live session**

Deploy, then restart the watcher in two separate commands (never one command line that both matches and spawns the pattern — `pkill -f` has matched the invoking shell before):

```bash
pkill -f 'Monitor[W]atcher' || true
```
then
```bash
setsid ~/.config/hypr/scripts/MonitorWatcher.sh >/dev/null 2>&1 &
```
Then: `pgrep -af 'Monitor[W]atcher' | wc -l` → `1`, and `lsof -p "$(pgrep -f 'Monitor[W]atcher' | head -n1)" | grep -c kooldots-monitor-watcher.lock` → `1` (only the watcher's own bash holds fd 9).

- [ ] **Step 6: Verify**

Run all suites. Expected: 0 failed.

---

### Task 10: Menu — dynamic rows and every action

**Files:**
- Modify: `$S/DisplayProfileMenu.sh`
- Modify: `$T/test-display-profile-menu.sh`

**Interfaces:**
- Consumes: Tasks 6–8 (`ds_*`, the controller CLI).
- Produces:
  - Rows built at runtime: the current fingerprint's layouts (ascending, the default marked `*`, the running one marked), a separator, then `Automatic`, `Arrange monitors — drag and drop (nwg-displays)`, `Save current state as...`, `Edit parameters (table)`, `Set as default...`, `Rename monitor set...`, `Delete layout...`, `Reset layout store...`, `Exit`.
  - `rofi -dmenu -format i` is the only selection mechanism; a parallel `ACTIONS` array maps index → action token. No `eval`, no name interpolated into a command string.
  - `mesg` line: `Monitor set: <label или fingerprint> (<n> monitors) · running: <name>`.
  - Name prompts validate with `ds_name_valid` and re-prompt once on failure.
  - `Reset layout store...` confirms, calls `ds_store_backup`, then writes the empty store — the only writer permitted when the store is malformed.
  - The `nwg-displays` row writes the pause file, launches the GUI with `9>&-` and `-m "$STATE_DIR/nwg-monitors.conf.discard"`, removes the pause file, then re-opens the menu with the save row first.

- [ ] **Step 1: Write the failing tests**

Append to `$T/test-display-profile-menu.sh`:

```bash
# The row list grows with the store.
FAKE_ROFI_DUMP_ROWS=1 run_menu dual store-v1 >"$WORK/rows.txt" 2>/dev/null || true
assert_contains "Home is listed"               "$WORK/rows.txt" 'Home'
assert_contains "Work is listed"               "$WORK/rows.txt" 'Work'
assert_contains "the default is marked"        "$WORK/rows.txt" '*'
assert_contains "the drag-and-drop row exists" "$WORK/rows.txt" 'nwg-displays'
assert_contains "the reset row exists"         "$WORK/rows.txt" 'Reset layout store'

# Index mapping: selecting index 0 applies the first layout, whatever it is named.
FAKE_ROFI_INDEX=0 run_menu dual store-v1
assert_contains "index 0 applied the first layout" "$FAKE_LOG" 'DisplayProfile.sh -- Home'

# The index of a fixed action shifts with the number of layouts and still works.
FAKE_ROFI_INDEX=3 run_menu dual store-v1     # 2 layouts + separator -> Automatic
assert_contains "the Automatic row maps to auto" "$FAKE_LOG" 'DisplayProfile.sh auto'
FAKE_ROFI_INDEX=2 run_menu dual store-one-layout
assert_contains "Automatic still maps to auto with one layout" "$FAKE_LOG" 'DisplayProfile.sh auto'

# Name validation.
FAKE_ROFI_INDEX=5 FAKE_ROFI_TEXT='bad/name' run_menu dual store-v1
assert_count_re "an invalid name is never saved" "$FAKE_LOG" 'capture' 0
assert_contains "the user is told why" "$FAKE_LOG" 'notify-send'
FAKE_ROFI_INDEX=5 FAKE_ROFI_TEXT='Good Name' run_menu dual store-v1
assert_contains "a valid name is captured" "$FAKE_LOG" 'DisplayProfile.sh capture -- Good Name'

# Review Focus 3: a name that looks like an option is passed as data, not a flag.
FAKE_ROFI_INDEX=5 FAKE_ROFI_TEXT='-n' run_menu dual store-v1
assert_contains "an option-like name is passed after --" "$FAKE_LOG" 'DisplayProfile.sh capture -- -n'
FAKE_ROFI_INDEX=0 run_menu dual store-dashn
assert_contains "applying an option-like layout name uses --" "$FAKE_LOG" 'DisplayProfile.sh -- -n'

# Escape (rofi exiting non-zero) must change nothing.
before_store="$(cat "$WORK/store.json")"
FAKE_ROFI_EXIT=1 run_menu dual store-v1 || true
assert_eq "Escape wrote nothing" "$before_store" "$(cat "$WORK/store.json")"
assert_count_re "Escape ran no controller command" "$FAKE_LOG" 'DisplayProfile.sh' 0

# A malformed store is readable but never written, except by Reset.
FAKE_ROFI_INDEX=0 run_menu dual store-malformed
assert_eq "the malformed store is untouched" "$(cat "$F/store-malformed.json")" "$(cat "$WORK/store.json")"
FAKE_ROFI_INDEX=9 FAKE_ROFI_CONFIRM=yes run_menu dual store-malformed
check "reset backed up first" bash -c 'ls "'"$WORK"'"/store.json.bak-* >/dev/null 2>&1'
assert_json_eq "$WORK/store.json" '.fingerprints | length' '0'

# nwg-displays: pause written, GUI launched with the discard path, pause removed.
FAKE_ROFI_INDEX=4 run_menu dual store-v1
assert_contains "the GUI got the discard path" "$FAKE_LOG" 'nwg-displays -m'
assert_contains "the discard path is inside the state dir" "$FAKE_LOG" 'nwg-monitors.conf.discard'
check "the pause file was removed afterwards" bash -c '! test -f "'"$WORK"'/run/kooldots-display-profiles/pause"'

# Missing GUI notifies instead of failing.
FAKE_ROFI_INDEX=4 DP_NWG_DISPLAYS=/nonexistent run_menu dual store-v1
assert_contains "a missing GUI notifies" "$FAKE_LOG" 'notify-send'

check "no eval in the menu" bash -c '! sed "s/#.*$//" "'"$SCRIPT"'" | grep -nE "\beval\b"'
check "rofi is called with -format i" bash -c 'grep -q -- "-format i" "'"$SCRIPT"'"'
check "no hardcoded output names" bash -c '! grep -nE "\b(eDP|DP|HDMI-A)-[0-9]" "'"$SCRIPT"'"'
```

`store-dashn` is written by the test: a copy of `store-v1.json` for the dual fingerprint whose single layout is named `-n`.

Extend the fake `rofi` in `$T/lib_test.sh` to honour `FAKE_ROFI_DUMP_ROWS` (print stdin and exit 1), `FAKE_ROFI_INDEX` (print that index), `FAKE_ROFI_TEXT` (print that text for a prompt without rows), `FAKE_ROFI_CONFIRM`, and `FAKE_ROFI_EXIT` (exit with that status, printing nothing — the Escape case).

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-profile-menu.sh`
Expected: FAIL — the menu still has four fixed labels and a `case`.

- [ ] **Step 3: Implement**

Rewrite the menu's body around two parallel arrays:

```bash
ROWS=(); ACTIONS=()
add_row() { ROWS+=("$1"); ACTIONS+=("$2"); }

while IFS= read -r name; do
  [[ -n $name ]] || continue
  mark=""
  [[ $name == "$default_name" ]] && mark="  *"
  [[ $name == "$running_name" ]] && mark="$mark  <-"
  add_row "$name$mark" "layout:$name"
done < <(ds_layout_names "$norm" "$DP_STORE_FILE")

(( ${#ROWS[@]} )) && add_row "────────────" "noop"
add_row "Automatic"                                    "auto"
add_row "Arrange monitors — drag and drop (nwg-displays)" "arrange"
add_row "Save current state as..."                     "save"
add_row "Edit parameters (table)"                      "table"
add_row "Set as default..."                            "setdefault"
add_row "Rename monitor set..."                        "label"
add_row "Delete layout..."                             "delete"
add_row "Reset layout store..."                        "reset"
add_row "Exit"                                         "exit"

idx="$(printf '%s\n' "${ROWS[@]}" | "$DP_ROFI" -i -dmenu -format i \
        -config "$rofi_theme" -mesg "$mesg" -p "Display profile")" || exit 0
[[ $idx =~ ^[0-9]+$ ]] || exit 0
(( idx >= 0 && idx < ${#ACTIONS[@]} )) || exit 0

action=${ACTIONS[$idx]}
case $action in
  layout:*) exec "$DP_SCRIPT" -- "${action#layout:}" ;;
  auto)     exec "$DP_SCRIPT" auto ;;
  ...
esac
```

The `arrange` action:

```bash
arrange_monitors() {
  if ! command -v "$DP_NWG_DISPLAYS" >/dev/null 2>&1; then
    notify "Display profile" "Install nwg-displays to arrange monitors by dragging"
    return 0
  fi
  mkdir -p -- "$STATE_DIR"
  local discard="$STATE_DIR/nwg-monitors.conf.discard"
  "$DP_NWG_DISPLAYS" -m "$discard" 9>&- &
  local pid=$!
  printf '%s\n' "$pid" >"$STATE_DIR/pause"
  wait "$pid" || true
  rm -f -- "$STATE_DIR/pause"
  UNSAVED=1
  return 0
}
```

With `UNSAVED=1`, the menu is rebuilt with `Save current state as...` as the first row and `mesg` prefixed `Unsaved arrangement · `.

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-profile-menu.sh`
Expected: PASS, 0 failed.

- [ ] **Step 5: Verify**

Run all suites. Expected: 0 failed. Deploy and press `SUPER+ALT+D`; the menu must list the live monitor set and exit cleanly on Escape without touching the store (`cmp` the store before and after).

---

### Task 11: Parameter table

**Files:**
- Create: `$S/DisplayProfileSetup.sh`
- Create: `$T/test-display-profile-setup.sh`

**Interfaces:**
- Consumes: Tasks 6–8.
- Produces:
  - `DisplayProfileSetup.sh [layout-name]` — edits that layout of the current fingerprint, or starts from a captured draft when the name is new or omitted.
  - Rows: one per connected monitor (`<make> <model> (<port>)  on|off · <mode> · x<scale> · row N · order N · <valign>`), then `Primary`, `Waybar`, `Position (computed)` (read-only), `Save & apply`, `Cancel`.
  - A monitor row opens a submenu: `On/Off`, `Mode` (from that monitor's real `modes`), `Scale` (1, 1.25, 1.333333, 1.5, 1.6, 1.75, 2), `Transform` (0, 1, 2, 3), `Row` (1–4), `Order` (1–8), `Vertical alignment` (top, center, bottom).
  - Every edit updates an in-memory draft; `ds_apply_grid` recomputes the `Position (computed)` row after each change.
  - `Save & apply` validates, writes with `ds_store_set`, then `exec`s the controller with the layout name.
  - `Cancel` writes nothing.

- [ ] **Step 1: Write the failing tests**

Create `$T/test-display-profile-setup.sh` with these checks:

```bash
FAKE_ROFI_DUMP_ROWS=1 run_setup triple >"$WORK/rows.txt" 2>/dev/null || true
assert_count_re "one row per monitor" "$WORK/rows.txt" '\(DP-1\)|\(DP-3\)|\(HDMI-A-1\)' 3
assert_contains "the primary row exists"           "$WORK/rows.txt" 'Primary'
assert_contains "the waybar row exists"            "$WORK/rows.txt" 'Waybar'
assert_contains "the computed position row exists" "$WORK/rows.txt" 'Position (computed)'
assert_contains "the save row exists"              "$WORK/rows.txt" 'Save & apply'

# The computed row reflects the draft, not the stored x/y.
assert_contains "computed positions are shown for three monitors" "$WORK/rows.txt" 'DP-1 0x0'

# Turning a monitor off and saving writes enabled:false and recomputes the rest.
FAKE_ROFI_SCRIPT='row:0 submenu:Off row:save text:Solo' run_setup triple
assert_json_eq "$WORK/store.json" '[.fingerprints[].layouts.Solo.monitors[] | select(.enabled == false)] | length' '1'
assert_json_eq "$WORK/store.json" '.fingerprints[].layouts.Solo.monitors[] | select(.enabled == true) | .x' '0'

# Scale changes recompute the neighbour's x.
FAKE_ROFI_SCRIPT='row:0 submenu:Scale value:1.5 row:save text:Scaled' run_setup triple
assert_json_eq "$WORK/store.json" '.fingerprints[].layouts.Scaled.monitors[1].x' '1707'

# Row 2 stacks below.
FAKE_ROFI_SCRIPT='row:2 submenu:Row value:2 row:save text:Stacked' run_setup triple
assert_json_eq "$WORK/store.json" '.fingerprints[].layouts.Stacked.monitors[2].y' '1440'

# Mode choices come only from that monitor's real modes.
FAKE_ROFI_DUMP_ROWS=1 FAKE_ROFI_SCRIPT='row:0 submenu:Mode' run_setup triple \
  >"$WORK/modes.txt" 2>/dev/null || true
assert_contains     "the real mode is offered"                    "$WORK/modes.txt" '2560x1440@59.95'
assert_not_contains "a mode from another monitor is not offered"  "$WORK/modes.txt" '1920x1080@144.00'

# Cancel writes nothing.
before="$(cat "$WORK/store.json")"
FAKE_ROFI_SCRIPT='row:cancel' run_setup triple
assert_eq "cancel wrote nothing" "$before" "$(cat "$WORK/store.json")"

# An invalid name is rejected before any write.
FAKE_ROFI_SCRIPT='row:save text:bad/name' run_setup triple
assert_eq "an invalid name wrote nothing" "$before" "$(cat "$WORK/store.json")"

# A layout that disables everything cannot be saved.
FAKE_ROFI_SCRIPT='row:0 submenu:Off row:1 submenu:Off row:2 submenu:Off row:save text:Dark' run_setup triple
assert_eq "an all-off layout wrote nothing" "$before" "$(cat "$WORK/store.json")"
assert_contains "the user is told why" "$FAKE_LOG" 'notify-send'

# Review Focus 3 again, in the table: an option-like name is never a flag.
FAKE_ROFI_SCRIPT='row:save text:-n' run_setup triple
assert_contains "the table passes an option-like name after --" "$FAKE_LOG" 'DisplayProfile.sh -- -n'

check "no eval" bash -c '! sed "s/#.*$//" "'"$SCRIPT"'" | grep -nE "\beval\b"'
check "no hardcoded outputs" bash -c '! grep -nE "\b(eDP|DP|HDMI-A)-[0-9]" "'"$SCRIPT"'"'
check "uses -format i" bash -c 'grep -q -- "-format i" "'"$SCRIPT"'"'
```

`FAKE_ROFI_SCRIPT` is a space-separated program the fake `rofi` consumes one token per invocation: `row:<n|save|cancel>` returns that index, `submenu:<label>` returns the index of the row whose text starts with that label, `value:<v>` returns the index of the row equal to that value, `text:<s>` returns that string.

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-display-profile-setup.sh`
Expected: FAIL — `DisplayProfileSetup.sh` does not exist.

- [ ] **Step 3: Implement**

Write `$S/DisplayProfileSetup.sh` with the KooLDots header, the Global Constraints preamble, `source lib_display_store.sh`, and this shape:

```bash
draft="$STATE_DIR/setup-draft.json"

load_draft() {
  local name=${1-}
  if [[ -n $name ]] && ds_layout_names "$norm" "$DP_STORE_FILE" | grep -qxF -- "$name"; then
    ds_store_read "$DP_STORE_FILE" \
      | "$DS_JQ" -c --arg fp "$fp" --arg n "$name" '.fingerprints[$fp].layouts[$n]' >"$draft"
    ds_assign_ports "$draft" "$norm" >"$draft.n" && mv -- "$draft.n" "$draft"
  else
    ds_capture "$norm" >"$draft"
  fi
}

recompute() { ds_apply_grid "$draft" >"$draft.n" && mv -- "$draft.n" "$draft"; }

render_rows() {
  ROWS=(); ACTIONS=()
  local i=0 line
  while IFS=$'\t' read -r port make model enabled mode scale transform row order valign; do
    printf -v line '%s %s (%s)   %s · %s · x%s · row %s · order %s · %s' \
      "$make" "$model" "$port" "$([[ $enabled == true ]] && echo on || echo off)" \
      "$mode" "$scale" "$row" "$order" "$valign"
    ROWS+=("$line"); ACTIONS+=("monitor:$i"); i=$((i+1))
  done < <("$DS_JQ" -r '.monitors[] | [ .port, .make // "", .model // "", .enabled, .mode,
             .scale, (.transform // 0), (.grid.row // 1), (.grid.order // 1),
             (.grid.valign // "top") ] | @tsv' -- "$draft")
  ROWS+=("Primary                $(primary_label)");            ACTIONS+=("primary")
  ROWS+=("Waybar                 $(waybar_label)");             ACTIONS+=("waybar")
  ROWS+=("Position (computed)    $(position_label)");           ACTIONS+=("noop")
  ROWS+=("Save & apply");                                       ACTIONS+=("save")
  ROWS+=("Cancel");                                             ACTIONS+=("cancel")
}
```

The three label helpers, so the table has no undefined renderer:

```bash
primary_label() {   # the make+model of the draft's primary identity
  "$DS_JQ" -r --slurpfile live "$norm" '.primary as $p
    | ($live[0][] | select(.identity == $p) | ((.make // "") + " " + (.model // "")))
      // $p' -- "$draft" | head -n1
}

waybar_label() {    # "all" | "primary" | "selected -> A, B"
  "$DS_JQ" -r '(.waybar.mode // "all") as $m
    | if $m == "selected"
      then "selected -> " + (((.waybar.outputs // []) | join(", ")) | if . == "" then "(none)" else . end)
      else $m end' -- "$draft"
}

position_label() {  # "DP-1 0x0 · DP-2 1920x0", enabled monitors only
  "$DS_JQ" -r '[ .monitors[] | select(.enabled == true)
                 | .port + " " + (.x|tostring) + "x" + (.y|tostring) ]
               | join(" · ")' -- "$draft"
}
```

`make`/`model` come from the normalized live data joined on `identity`, so the draft need not store them. The main loop renders, reads one `-format i` choice, dispatches, calls `recompute`, and loops until `save` or `cancel`.

`save` must: prompt for a name when the layout is new, `ds_name_valid`, `recompute`, `ds_validate_layout` (notify and return to the table on failure), `ds_store_set`, then `exec "$DP_SCRIPT" -- "$name"`.

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-display-profile-setup.sh`
Expected: PASS, 0 failed.

- [ ] **Step 5: Verify**

Run all suites. Expected: 0 failed. Deploy and open the table from the menu; Cancel must leave the store byte-identical.

---

### Task 12: Upstream call sites

**Files:**
- Modify: `$S/Refresh.sh` (the `restart_waybar` function, around line 105)
- Modify: `$S/WaybarLayout.sh` (`apply_config`)
- Modify: `$S/Kool_Quick_Settings.sh` (around line 511)
- Create: `$T/test-upstream-fallbacks.sh`

**Interfaces:**
- Consumes: Task 8 (`DisplayProfile.sh waybar-config`).
- Produces: a shared snippet in each file, resolving the Waybar config path with a fallback. No new file is sourced, because these three scripts must keep working if `DisplayProfile.sh` is deleted.

- [ ] **Step 1: Back up the three files**

```bash
B="$HOME/.config/hypr/backups/multi-monitor-$(date +%Y%m%d-%H%M%S)"
mkdir -p -- "$B"
cp -- config/hypr/scripts/Refresh.sh config/hypr/scripts/WaybarLayout.sh \
      config/hypr/scripts/Kool_Quick_Settings.sh "$B/"
echo "$B"
```
Record the printed path in the README in Task 13.

- [ ] **Step 2: Write the failing tests**

Create `$T/test-upstream-fallbacks.sh`:

```bash
# With the controller present, the path comes from it.
printf '#!/usr/bin/env bash\necho /tmp/from-controller\n' >"$SANDBOX/bin/DisplayProfile.sh"
chmod +x "$SANDBOX/bin/DisplayProfile.sh"
out="$(resolve_in Refresh.sh)"
assert_eq "Refresh.sh uses the controller's path" '/tmp/from-controller' "$out"
out="$(resolve_in WaybarLayout.sh)"
assert_eq "WaybarLayout.sh uses the controller's path" '/tmp/from-controller' "$out"

# With the controller absent, each falls back to the plain config.
rm -f "$SANDBOX/bin/DisplayProfile.sh"
assert_eq "Refresh.sh falls back"      "$WORK/waybar/config" "$(resolve_in Refresh.sh)"
assert_eq "WaybarLayout.sh falls back" "$WORK/waybar/config" "$(resolve_in WaybarLayout.sh)"

# A controller that fails must not win.
printf '#!/usr/bin/env bash\nexit 1\n' >"$SANDBOX/bin/DisplayProfile.sh"; chmod +x "$SANDBOX/bin/DisplayProfile.sh"
assert_eq "a failing controller falls back" "$WORK/waybar/config" "$(resolve_in Refresh.sh)"

# A controller printing a non-existent path must not win.
printf '#!/usr/bin/env bash\necho /no/such/file\n' >"$SANDBOX/bin/DisplayProfile.sh"
assert_eq "a bogus path falls back" "$WORK/waybar/config" "$(resolve_in Refresh.sh)"

# Quick Settings routes the GUI through the pause protocol.
check "Quick Settings writes a pause file" bash -c 'grep -q "pause" "'"$S"'/Kool_Quick_Settings.sh"'
check "Quick Settings uses the discard path" bash -c 'grep -q "nwg-monitors.conf.discard" "'"$S"'/Kool_Quick_Settings.sh"'
check "Quick Settings still notifies when nwg-displays is missing" bash -c 'grep -q "Install nwg-displays" "'"$S"'/Kool_Quick_Settings.sh"'
check "Quick Settings closes fd 9 on the GUI" bash -c 'grep -n "nwg-displays" "'"$S"'/Kool_Quick_Settings.sh" | grep -q "9>&-"'
```

`resolve_in <file>` extracts the `kooldots_waybar_config` function from that file with `sed` and runs it in a subshell with `waybar_dir="$WORK/waybar"`.

- [ ] **Step 3: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-upstream-fallbacks.sh`
Expected: FAIL — no such function in any of the three files.

- [ ] **Step 4: Patch the three files**

Insert this function into `Refresh.sh` above `restart_waybar`, and into `WaybarLayout.sh` above `apply_config`:

```bash
# Ask the display-profile controller which Waybar config to use. Falls back to
# the plain config so this script keeps working without that feature installed.
kooldots_waybar_config() {
  local dp="${XDG_CONFIG_HOME:-$HOME/.config}/hypr/scripts/DisplayProfile.sh" p=""
  if [[ -x $dp ]]; then
    p="$("$dp" waybar-config 2>/dev/null | head -n1 || true)"
  fi
  if [[ -n $p && -f $p ]]; then printf '%s\n' "$p"; else printf '%s\n' "$waybar_dir/config"; fi
}
```

In `Refresh.sh`, replace the two `-c \"$waybar_dir/config\"` occurrences inside `restart_cmd` with `-c \"$(kooldots_waybar_config)\"`. In `WaybarLayout.sh`, `apply_config` switches the symlink first and then uses `kooldots_waybar_config` for the launch.

In `Kool_Quick_Settings.sh`, replace the bare `nwg-displays` call with:

```bash
    state_dir="${XDG_RUNTIME_DIR:-/tmp}/kooldots-display-profiles"
    mkdir -p -- "$state_dir"
    nwg-displays -m "$state_dir/nwg-monitors.conf.discard" 9>&- &
    nwg_pid=$!
    printf '%s\n' "$nwg_pid" >"$state_dir/pause"
    wait "$nwg_pid" || true
    rm -f -- "$state_dir/pause"
    notify-send -i "$iDIR/ja.png" "Monitors" "Press SUPER+ALT+D -> Save current state as... to keep this arrangement"
```

keeping the existing `command -v nwg-displays` guard and its "Install nwg-displays first" notification above it.

- [ ] **Step 5: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-upstream-fallbacks.sh`
Expected: PASS, 0 failed.

- [ ] **Step 6: Verify on the live session**

Deploy the three files, then run `~/.config/hypr/scripts/Refresh.sh` once.
Expected: `pgrep -c waybar` is `1` afterwards, the bars come back, and the Waybar process's `-c` argument matches `DisplayProfile.sh waybar-config`.

- [ ] **Step 7: Verify**

Run all suites. Expected: 0 failed.

---

### Task 13: Distribution, keybinds, docs, and the anti-regression scan

**Files:**
- Create: `config/hypr/UserConfigs/display-layouts.json`
- Create: `$T/test-no-hardcoded-outputs.sh`
- Modify: `config/hypr/UserConfigs/user_keybinds.lua`
- Modify: `copy.sh`
- Modify: `$S/README-display-profiles.md` (rewrite)
- Modify: `$T/` — a runner that executes every suite

**Interfaces:**
- Consumes: every earlier task.
- Produces: `$T/run-all.sh` printing one summary line per suite and exiting non-zero if any failed.

- [ ] **Step 1: Write the failing scan**

Create `$T/test-no-hardcoded-outputs.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
HERE="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
source "$HERE/lib_test.sh"
S="$HERE/.."

for f in DisplayProfile.sh DisplayProfileMenu.sh DisplayProfileSetup.sh \
         MonitorWatcher.sh lib_display_store.sh; do
  check "no output name in $f" bash -c \
    '! sed "s/#.*$//" "'"$S/$f"'" | grep -nE "\b(eDP|LVDS|DSI|DP|HDMI-A|DVI-D)-[0-9]+"'
done

check "no laptop/external vocabulary survives" bash -c \
  '! grep -rniE "LAPTOP_OUTPUT|EXTERNAL_OUTPUT|EXTERNAL_EXPECTED" "'"$S"'"/*.sh'

check "the shipped store is an empty v1 store" bash -c \
  'jq -e ".version == 1 and (.fingerprints | length) == 0" \
     "'"$S"'/../UserConfigs/display-layouts.json" >/dev/null'

test_summary
```

- [ ] **Step 2: Run to verify it fails**

Run: `bash config/hypr/scripts/tests/test-no-hardcoded-outputs.sh`
Expected: FAIL — the shipped store does not exist yet, and any leftover output name is named by file and line.

- [ ] **Step 3: Ship the store and fix whatever the scan found**

```bash
printf '{"version":1,"fingerprints":{}}\n' > config/hypr/UserConfigs/display-layouts.json
```

Remove every output name the scan reported. If one is genuinely needed in a comment, rewrite the comment — the scan strips comments before matching, so a comment cannot be the cause of a failure.

- [ ] **Step 4: Run to verify it passes**

Run: `bash config/hypr/scripts/tests/test-no-hardcoded-outputs.sh`
Expected: PASS, 7 checks, 0 failed.

- [ ] **Step 5: Update the keybind descriptions**

In `config/hypr/UserConfigs/user_keybinds.lua`, keep both binds and change only their descriptions:

```lua
bind("SUPER ALT", "D", exec_cmd("$HOME/.config/hypr/scripts/DisplayProfileMenu.sh"),
  { description = "Display layouts: menu for this monitor set" })
bind("SUPER ALT", "W", exec_cmd("$HOME/.config/hypr/scripts/DisplayProfile.sh -- Work"),
  { description = "Display layouts: apply the layout named Work" })
```

Run: `hyprctl configerrors`
Expected: empty output.

- [ ] **Step 6: Ship the store through copy.sh**

Confirm `copy.sh` already copies `config/hypr/UserConfigs/` wholesale (it does, via the hypr config copy). If it does, add nothing; if it enumerates files, add `display-layouts.json` beside `monitors.lua`. Verify with:

```bash
grep -n "UserConfigs" copy.sh | head -20
```
Expected: evidence of a directory-level copy, or the new entry present.

- [ ] **Step 7: Rewrite the README**

Rewrite `$S/README-display-profiles.md` covering: the model (identity, fingerprint, layout); the CLI table; the menu and the table; the drag-and-drop flow and why the pause file exists; the three Waybar modes; **the stand-down rule and that "it does nothing" is expected when `monitors.lua` has your own `hl.monitor` lines**; that disabling a monitor moves its workspaces and they do not come back; that an identity with no make/model is port-dependent; the state directory contents; the two migration steps (capture `Home`, author `Work` in the table); troubleshooting commands (`DisplayProfile.sh --dry-run auto`, `MonitorWatcher.sh --print-socket`, the two log files, `lsof` for a held lock); the test commands; and the exact rollback using the backup directory recorded in Task 12 Step 1.

- [ ] **Step 8: Add the runner and run everything**

Create `$T/run-all.sh` executing each `test-*.sh` under `timeout 300`, printing `<suite>: <n> run, <m> failed`, exiting 1 if any failed.

Run: `bash config/hypr/scripts/tests/run-all.sh`
Expected: eight suites, 0 failed in total.

- [ ] **Step 9: Final live verification**

```bash
diff -r --brief config/hypr/scripts ~/.config/hypr/scripts | grep -v '^Only in' || echo "scripts identical"
diff config/hypr/UserConfigs/display-layouts.json ~/.config/hypr/UserConfigs/display-layouts.json
hyprctl configerrors
pgrep -c waybar
pgrep -cf 'Monitor[W]atcher'
~/.config/hypr/scripts/DisplayProfile.sh --dry-run auto
```
Expected: scripts identical, store identical, no config errors, exactly one Waybar, exactly one watcher, and a dry-run plan naming the live monitors. Do not commit.

---

## Handover: what only the user can verify

1. **Apply on three or more monitors.** The development machine has two; the logic is proven on fixtures only.
2. **Drag-and-drop inside `nwg-displays`** — including that the pause file really stops the watcher from undoing the arrangement.
3. **Disabling a monitor that is currently in use**, including the workspace migration Hyprland performs and does not reverse.
4. **A monitor moved to a different port** — unplug the external monitor from one port and into another; the same layout must still apply.
