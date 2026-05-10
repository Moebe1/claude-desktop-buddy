# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Behavioral guidelines

Tradeoff: bias toward caution over speed. For trivial tasks, use judgment.

### 1. Think before coding

Don't assume. Don't hide confusion. Surface tradeoffs.

- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity first

Minimum code that solves the problem. Nothing speculative.

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical changes

Touch only what you must. Clean up only your own mess.

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.
- Remove imports/variables/functions that *your* changes orphan; don't remove pre-existing dead code unless asked.

Test: every changed line should trace directly to the user's request.

### 4. Goal-driven execution

Define success criteria. Loop until verified.

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan with a verification step per item.

## Project nature

This is an ESP32/Arduino **reference implementation** of the BLE protocol the
Claude desktop apps speak to maker hardware. The protocol — not this firmware
— is the stable surface. `REFERENCE.md` is the canonical wire-protocol spec;
treat it as source of truth and update it when protocol behavior changes.

`CONTRIBUTING.md` declares this firmware is not actively maintained: no new
features, pets, screens, board ports, refactors, style changes, or dependency
bumps. Acceptable changes are limited to (a) corrections to `REFERENCE.md`
and (b) bugs that prevent the reference from working as a reference (won't
pair, won't render, crashes on boot). Default to declining feature work and
suggesting a fork unless the user explicitly overrides this.

## Build, flash, develop

PlatformIO Core is required. Single env: `m5stickc-plus` (see `platformio.ini`).

```bash
pio run -t upload       # compile + flash firmware over USB
pio run -t erase        # full chip erase (NVS + filesystem + bonds)
pio run -t uploadfs     # flash LittleFS image from data/ (used for GIF packs)
pio device monitor      # serial console at 115200
```

There is no test framework wired up. The "tests" are interactive scripts under `tools/`:

- `tools/test_serial.py` — drive the device with fake heartbeat JSON over USB
- `tools/test_xfer.py` — exercise the folder-push receiver over USB
- `tools/flash_character.py characters/<name>` — stage a pack into `data/` and `uploadfs` it (skips the BLE round-trip when iterating on art)
- `tools/prep_character.py <dir-or-zip>` — downscale GIFs to a consistent 96px-wide pack

`data/` is gitignored — it's a transient staging area for `uploadfs`. Don't commit it.

## Architecture

**Top-level flow.** Desktop apps speak Nordic UART Service (BLE). Each line
is one UTF-8 JSON object terminated by `\n`. Both directions reassemble
fragmented notifications until they see `\n`. The same JSON parser handles
USB serial and BLE — see `_applyJson` in `src/data.h`.

**Operating modes** (priority order, in `dataPoll()`):
1. **demo** — auto-cycle fake scenarios every 8s, ignore live data
2. **live** — JSON arrived in the last 10s over USB or BLE
3. **asleep** — no data, all zeros

**Render pipeline.** `main.cpp` runs the loop, state machine, menus, and
screens, drawing into a single shared `TFT_eSprite spr`. Two render
backends draw the buddy:

- **ASCII species** (`src/buddies/<species>.cpp` + `src/buddy.cpp`) — 18
  species, each exposing 7 state functions (sleep, idle, busy, attention,
  celebrate, dizzy, heart). Species are statically linked: every species
  has an `extern const Species` declaration and an entry in `SPECIES_TABLE`
  inside `buddy.cpp`. PlatformIO's `build_src_filter` pulls in `buddies/`.
- **GIF character** (`src/character.cpp`) — a single character pack lives at
  `/characters/<name>/` on LittleFS with a `manifest.json` and 7 state GIFs
  (or arrays of GIFs, which rotate).

`buddy.cpp` and `character.cpp` both have a `setPeek(bool)` that switches
between full-size (home) and half-size (info/pet pages) rendering. They use
the same persona-state enum order — keep `PersonaState` in `main.cpp`,
`buddy.cpp`'s `B_*` enum, and `character.cpp`'s state index aligned.

**Persistence.** Two layers, not interchangeable:

- **NVS / `Preferences`** for small structured state — stats, settings,
  owner name, pet name, species choice, BLE bonds. `stats.h` and
  `main.cpp` open the `"buddy"` namespace. Save sparingly (sectors wear at
  ~100K writes); save on events, never on a timer.
- **LittleFS** for files — GIF character packs under `/characters/`, written
  by `xfer.h`'s folder-push receiver. The partition is sized for one
  character at a time; `_xWipeAllChars` wipes before installing a new pack.
  Cap is 1.8 MB.

**Header-only state warning.** `src/stats.h` and `src/xfer.h` use
file-static state in headers and must be included from **exactly one
translation unit (`main.cpp`)**. Including from a second `.cpp` produces
duplicate-symbol link errors and silently splits state across TUs. If
sharing them more broadly, refactor to a `.cpp` first.

**BLE security.** NUS characteristics are encrypted-only with LE Secure
Connections bonding (DisplayOnly IO capability → 6-digit passkey on the
LCD). `bleClearBonds()` is called from the `unpair` command and from
factory reset; the desktop's "Forget" button drives that path.

## Adding an ASCII species

1. Add `src/buddies/<name>.cpp` exposing `const Species <NAME>_SPECIES = { ... }` with 7 state functions.
2. Add the matching `extern const Species` and append to `SPECIES_TABLE[]` in `src/buddy.cpp`.

(But note the contributing policy above — usually decline this and recommend forking.)

## Things easy to break

- **Persona-state enum order is load-bearing.** `PersonaState` (main.cpp), the `B_*` enum (buddy.cpp), and the GIF state index (character.cpp/manifest.json) all index the same arrays. Reordering one without the others silently swaps animations.
- **Wire protocol must echo `prompt.id` exactly** in permission decisions, and every `cmd` expects a matching `{"ack":"<cmd>","ok":...}`. See `REFERENCE.md` for the full table.
- **30-second snapshot timeout** is the desktop's deadline; if you change heartbeat cadence, update `dataConnected()`'s 30s window in `data.h` and the spec in `REFERENCE.md` together.
- **Folder-push paths come from the desktop's filesystem.** `xfer.h` should reject `..` and absolute paths before opening the file.
