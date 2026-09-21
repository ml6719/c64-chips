# C64 (chips-test / floooh/chips)

Fork of [floooh/chips-test](https://github.com/floooh/chips-test) (public — zlib licensed,
no GPL obligations, so no privacy caution needed here unlike hatari-wasm). Fork:
https://github.com/ml6719/c64-chips

## Why this core, not VICE

The project's own README originally slated C64 for VICE (matching Hatari's "compile the
real reference emulator" precedent). Investigated that first, tonight, and found upstream
VICE has **no native Emscripten build support at all** (unlike Hatari, which already had
an EMSCRIPTEN-guarded CMake path this project only needed to debug). Building VICE for the
web would mean architecting the entire SDL2/Emscripten integration layer from zero, across
a much bigger and more tangled autotools build than Hatari's — high risk of burning a whole
session on build-system archaeology with nothing playable to show for it.

Landed on [floooh/chips](https://github.com/floooh/chips) instead (via its `chips-test`
harness) after verifying independently (not just taking a suggestion at face value):
- Real, actively maintained (pushed within the last month at time of writing), 1284 stars,
  zlib license.
- `systems/c64.h` plus `c1541.h` (disk drive) and `c1530.h` (datasette) — storage isn't an
  afterthought.
- A dedicated `examples/common/webapi.c`/`.h` — the same kind of purpose-built JS↔WASM
  bridge pattern as Hatari's `web_api.c`, just cleaner (proper `emscripten_set_main_loop`
  callback model, not Hatari's Asyncify-coroutine danger zone — direct JS→C calls are safe
  at any time here, no pending-action-memory-write workaround needed).
- CI (`build.yml`) builds the WASM output on every push, so it's continuously verified to
  actually compile against current Emscripten — not a stale, bit-rotted port.
- This isn't "hand-rolled by us" or a toy JS reimplementation — a real independently-written
  native C core compiled to WASM, same category of choice as `vendor/zx84` (already proven
  out for CPC/Spectrum/ZX8x in this project). Not literally "the official reference
  emulator" (that's VICE), but neither is zx84 for CPC/Spectrum, and that choice worked out
  fine.
- One thing that could NOT be confirmed tonight: the live demo
  (floooh.github.io/tiny8bit/c64.html) rendered a genuinely black canvas in this session's
  sandboxed browser tool (WASM/JS loaded fine, audio initialized, frame-timing HUD updating,
  zero console errors, `readPixels` confirmed truly black framebuffer) — tried a fresh tab
  too, same result. Couldn't determine whether that's a real bug in floooh's demo or a
  WebGL/automation-environment quirk specific to this sandboxed tool. Doesn't block us
  either way since we're building our own shell from scratch, not using floooh's demo shell.

## Build tooling: `fibs`, not CMake/emcmake directly

`chips-test` uses its own Deno-based build orchestrator (`fibs`, fetched at runtime from
JSR — `jsr:@floooh/fibs`), not a hand-driven `emconfigure`/`cmake` invocation. Needed `deno`
installed (`winget install --id DenoLand.Deno`, not preinstalled on this box).

```bash
./fibs diag tools           # check git/cmake/ninja present
./fibs emsdk install        # installs its OWN emsdk copy under .fibs/sdks/emsdk
                             # (separate from the atari-st project's C:/LLM-Projects/emsdk —
                             # fibs manages its own, didn't try to make it share)
./fibs config emsc-ninja-release
./fibs build c64             # the plain (non-imgui-debug-UI) C64 target
```

**Same Python-2.7-on-PATH trap as atari-st's emsdk setup**: `emsdk.py` needs Python 3.10+,
but this box's default `python`/`cmd`-resolved python is `C:\Python27`. Fix: prepend a
working Python 3 to `PATH` before calling `fibs emsdk install`
(`/c/Users/Administrator/AppData/Local/Programs/Python/Python313`). Symptom without the fix:
`SyntaxError: invalid syntax` pointing at an f-string in `emsdk.py`, silently followed by
`fibs` reporting "Emscripten SDK already installed" on retry even though activation never
actually succeeded — had to manually `rm -rf .fibs/sdks/emsdk` before retrying with the
fixed PATH, `fibs emsdk uninstall` alone didn't clear the bad state.

## Critical fix: swapped the bundled copyrighted Commodore ROMs

`chips-test` bundles real Commodore ROM dumps directly in its repo at `examples/roms/`:
`c64_basic.bin` (8192 bytes), `c64_char.bin` (4096 bytes), `c64_kernalv3.bin` (8192 bytes)
— confirmed by exact chip size match and byte content (visibly a real 6502 BASIC ROM
disassembly, "COMMODORE BASIC" string literally present in the bytes). Same copyright
situation as Atari TOS, which this project has been careful never to bundle itself (see
`machines/atari-st/CLAUDE.md`'s Licensing note) — so applied the same rule here.

**Fix**: replaced the three files' *contents* with the equivalent binaries from
[MEGA65/open-roms](https://github.com/MEGA65/open-roms) (LGPL-3.0+, genuinely open,
confirmed by reading the actual `LICENSE` file text, not just trusting the GitHub license
badge which shows "Other/NOASSERTION") — `basic_generic.rom`, `kernal_generic.rom`,
`chargen_openroms.rom`, all exact byte-size matches for their C64 chip slots (8K/8K/4K).
Kept the **original filenames** so the generated header (`c64-roms.h`, produced by fibs'
built-in `embedfiles` job from `fibs.ts`'s `addRoms()`) keeps identical C symbol names
(`dump_c64_basic_bin` etc.) — zero changes needed to `examples/emus/c64.c`.

This is the same EmuTOS-equivalent default-boot strategy as Atari ST: boots to a compatible-
but-not-bit-identical open ROM set by default. `open-roms` itself documents that some
software relying on undocumented/hardcoded KERNAL routine addresses may not behave
identically to a real KERNAL — same caveat class as EmuTOS vs real TOS. **Future work**:
add a "Load ROM" upload feature mirroring the "Load TOS" pattern just built for Atari ST,
for users who have legal rights to the original Commodore ROMs and want full compatibility.

**Fixed too**: `c64.c` also `#include`s `c1541-roms.h` (the disk-drive ROMs get compiled
into `c64.wasm` regardless of whether the drive is runtime-enabled — `c1541_enabled =
sargs_exists("c1541")`, off by default, but the bytes are still shipped in the binary
either way). Since no clean open-source 1541 firmware replacement exists (checked:
open1541/Pi1541/1541 Ultimate are all real-hardware SD-card-replacement projects, not
drop-in ROM binaries; DolphinDOS 2 is a kernal+1541 speed-hack, not a clean-room rewrite)
and our shell doesn't expose real disk-drive emulation anyway (PRG quickload only, via the
`mx_quickload_prg` bridge below — see `web_load()`'s SYS-call-only behavior in c64.c for
why we didn't just wire up `webapi.c`'s existing 16-byte-header `load()` path instead),
**zeroed out both 1541 ROM files' contents** (same filenames, all-zero bytes, correct
sizes) rather than ship the real firmware unnecessarily. If real `.d64` disk emulation is
ever added, this needs a real fix first (either find/wait for an open 1541 replacement, or
drop `c1541-roms.h`/the drive object from `c64.c` entirely instead of just stubbing it).

## Did NOT need to trim the `roms` targets

`fibs.ts`'s `addRoms()` unconditionally defines an `embedfiles` job for every catalogued
system (atom, bombjack, pacman, pengo, cp4, cpc, kc85, lc80, z9001, zx, c1541, c64) inside
one `roms` interface target. Confirmed this does NOT block building just the `c64` target —
cmake/ninja only actually processes/embeds the ROM sets a given target's source
`#include`s (`c64.c` only pulls in `c64-roms.h` + `c1541-roms.h`), so the other systems'
files being present-but-untouched in the fork's working tree isn't a build problem. Also
not a *new* distribution concern: those files were already public in the upstream repo we
forked (chips-test is public), and our own shipped artifact (`c64.wasm`) only ever embeds
the two headers `c64.c` actually includes — verified by grep, not assumption.

## The web bridge: `mx_reset`/`mx_key_down`/`mx_key_up`/`mx_quickload_prg`

The plain `c64` fibs target (the one we actually build - no debug UI) has **no `webapi.c`
wired in at all** — that file, and the `webapi_init()` call wiring it up, only exist inside
`#ifdef CHIPS_USE_UI`, which is only defined for the heavier `c64-ui` target (floooh's own
imgui debug overlay). Rather than pull in that whole UI just to get a JS bridge, added four
small `EMSCRIPTEN_KEEPALIVE` functions directly in `c64.c` (outside any `#ifdef`, right
after the `state` struct) — same "small, project-owned exported bridge" pattern as
`machines/atari-st`'s `web_api.c`, just four one-line functions instead of a whole file:
- `mx_reset()` → `c64_reset(&state.c64)`
- `mx_key_down(int c)` / `mx_key_up(int c)` → `c64_key_down`/`c64_key_up`. All safe to call
  at any time — `chips-test`'s main loop is a plain per-frame `emscripten_request_animation_frame`
  callback, nothing like Hatari's Asyncify-coroutine danger zone, so no pending-action/
  memory-write workaround is needed here.
- `mx_quickload_prg(void* ptr, int size)` → `c64_quickload()` + `c64_basic_run()` (mirrors
  `handle_file_loading()`'s own drag-drop path — load then simulate typing RUN — rather than
  `webapi.c`'s `web_load()`, which always does a SYS call to a machine-code entry point;
  wrong default for a plain BASIC program).

**One extra link-option patch needed**: the shell calls `mx_quickload_prg` via
`Module.ccall(..., ['array', 'number'], ...)` (marshals the PRG byte array via stack
allocation) rather than raw `Module._malloc`/`_free`, because neither `malloc`/`free` nor
`ccall` are exported by default. Added one link option
(`-sEXPORTED_RUNTIME_METHODS=['ccall']`), scoped to just the `c64` target inside the `emus`
loop in `fibs.ts` (an `if (emu === 'c64')` guard) so it doesn't change any other system's
build.

**Verified working end to end from the browser console** (not just "it compiled"):
`Module._mx_reset()`, `Module._mx_key_down/up(...)`, and
`Module.ccall('mx_quickload_prg', ...)` with a synthetic 5-byte PRG (2-byte load address +
3 bytes) all ran with zero exceptions and zero console errors/warnings.

## Keyboard: verified against source, not guessed

Every key code and matrix position in `machines/c64/web/index.html`'s `KEYBOARD_ROWS` was
read directly out of `systems/c64.h`'s `_c64_init_key_map()` (the `C64_KEY_*` `#define`s and
the ASCII `keymap` table + `kbd_register_key()` calls), not recalled from memory or
guessed — same discipline as the ZX80/81 keyboard build, which explicitly asked for a
reference re-paste rather than risk a wrong transcription.

**Real hold-Shift isn't possible with this API**: `c64_key_down()`/`c64_key_up()` take a
resolved *character code*, not a raw matrix cell. Shift is registered only as a
`kbd_register_modifier()` (asserted automatically based on which layer a character's code
belongs to), with no dedicated `C64_KEY_SHIFT` constant the way `CTRL`/`C=`/`RESTORE`/etc.
have — so there's no legal code to pass `kbd_key_down()` for "just Shift, held, on its
own". Implemented on-screen Shift as a **UI-level lock** instead (tracked in JS, flips which
code — `k.code` vs `k.shiftCode` — a subsequent tap sends), which reproduces the same
user-facing double-click-to-lock behavior as every other machine's Shift key, just without
a real matrix cell underneath. CTRL and C= genuinely do hold via mousedown/mouseup, since
both have real dedicated key-codes.

**Dropped from the on-screen layout**: the "↑" (up-arrow/exponent) key at the end of the
QWERTY row. Accounted for every other key in the physical layout against the extracted
table with high confidence, but couldn't pin down ↑'s exact matrix position (it isn't in
the `C64_KEY_*` special-keys list, and every printable-ASCII-keymap cell is otherwise
accounted for) — left it off rather than guess a matrix cell, same principle as everything
else in this section. Physical/real keyboard passthrough is unaffected either way (this is
purely an on-screen-keyboard gap).

## Status

- [x] Forked `chips-test`, `deno`/`cmake`/`ninja` toolchain confirmed present, fibs' own
      emsdk installed successfully (`6.0.10`).
- [x] Copyrighted C64 KERNAL/BASIC/char ROMs replaced with open-roms equivalents; 1541
      disk-drive ROMs zeroed out (no open replacement exists).
- [x] `emsc-ninja-release` build of the plain `c64` target — clean, no errors.
- [x] Custom web shell (`web/index.html`) matching every other machine's shared-toolbar
      convention (hamburger/sidebar responsive layout, canvas-wrap aspect-ratio-preserving
      resize, fullscreen, keyboard toggle) — hand-rolled directly in the shell (not the
      shared matrix-based VKeyboard component), same as Atari ST, since this core's
      character-code keyboard API doesn't fit that component's (row,bit) model either.
- [x] Virtual keyboard, matrix/key-code-accurate against `systems/c64.h` source.
- [x] PRG loading (with zip support, same dependency-free `DecompressionStream` approach as
      every other machine) via the new `mx_quickload_prg` bridge.
- [x] `build-site.mjs` / landing page wiring (`site/index.html`'s C64 tile enabled, marked
      "early build" rather than claiming full parity with the other four).
- [ ] **Not yet visually confirmed on screen** — the session that built this had the
      Browser pane hidden throughout (`window.innerWidth`/`innerHeight` read `0`, which
      genuinely prevents a WASM canvas from sizing/rendering at all — confirmed this isn't
      a bug in the build by testing `Module._mx_reset()` etc. directly via the JS console
      instead, all of which worked cleanly). Not an issue for a normal user with the pane
      visible — just something this session couldn't check itself. **First thing to check
      in the morning.**
- [ ] Never tested against a real game/program — only a synthetic empty PRG (load-address
      header only, no real code) to confirm the loading pipeline doesn't crash.
- [ ] No real hold-Shift, no ↑ key (see above), no volume control (didn't find sokol_audio's
      Web Audio context exposed anywhere the way Hatari's SDL2 audio graph was — didn't
      want to guess at unfamiliar internals this late), no focus-pause (chips-test has no
      Hatari-style `pause()`/`resume()` API — the browser's own animation-frame throttling
      on a backgrounded tab is the only thing currently doing this, which is *something*
      but not the same guaranteed behavior as the other machines), no model select (only
      plain "C64", not C64C or other variants — chips-test may not even support variants).
- [ ] `site/c64` is built locally but not yet deployed — needs `node build-site.mjs && npx
      wrangler pages deploy site` to go live at themultitude.pages.dev / gingerspice.com.
