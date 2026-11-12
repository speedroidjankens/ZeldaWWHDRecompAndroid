# WWHD fork — merge notes

Merged source tree for the user's own fork:
`https://github.com/speedroidjankens/ZeldaWWHDRecompAndroid`
(`origin` in this repo points there; **nothing was pushed** — no GitHub auth
here. The user pushes from their side: `git push origin android`.)

- **Base (kept):** GreenNaugahyde `ZeldaWWHDRecompAndroid`, branch `android`,
  v0.9.9c (`f31b7e7`), Nov 6, 2026. Remote `upstream`.
- **Ported on top:** SSunnKing `ZeldaWWHDRecompAndroid---ENHANCED`,
  branch `enhanced` (`2f28a40`). Remote `enhanced`.
- **Common ancestor:** `1a4cf47e935f86b20b0fbe95809a4ab2718131a6`.
- Merge commit: see `git log` (merge of `enhanced/enhanced` into `android`).

Note: the `/tmp/wwhd-clash` clones used for the first clash check were wiped
by the environment (tmpfs cleared) during this merge — not deleted by hand.
Both histories survive in this repo's `.git` (`origin`/`upstream`/`enhanced`
refs), so nothing was lost.

## Conflict count and how they were resolved

Live `git merge` (not just `merge-tree`): **55 conflicted files, 396 hunks**.
239 files merged cleanly, including ~192 files Enhanced added
(IconForge/TouchIcons + touch icons, crash-log plumbing, LSFG, etc.).

Policy, cheapest first:
- **Union** for hunks that were pure additions on one side, supersets, or
  identical — keep both sides' lines.
- **Keep Modern (ours)** for hunks where both sides changed the same lines.
  Modern is the newer base (0.9.9c vs Enhanced's 0.8/0.9-era fork); Enhanced's
  version of the same code is the older one.
- **Data files** (`tools/recomp/hooks.txt`, `hooks_aspect.txt`,
  `release_jpn.txt`): union of lines, deduped.
- **CrashLogs / CrashLogProvider (add/add):** kept Modern's version.

Per-file summary (hunks resolved to Modern unless noted):
- `runtime/src/interp.cpp` (47), `interp_fx.cpp` (12), `interp.h` (2),
  `interp_pacing.h` (1), `true60.cpp` (2), `true60.h` (1): **all Modern**.
  Enhanced's interp is the older version of the same system with
  incompatible hook names (`camera_draw_orig` vs `f_024FFC40_orig`, etc.).
  Both offer 40 fps and true-60; Modern's is the rewritten one. See
  "40 fps mode" below.
- `runtime/src/vk/vk_draw.cpp` (48): Modern, except one 282-line
  Enhanced-only `Prepared` struct (hunk 45) was **dropped** — Modern's
  equivalent (shader/pipeline cache rewrite) lives elsewhere in the file.
- `runtime/src/vk/vk_device.cpp` (24), `vk_surfaces.cpp` (23),
  `gx2/gx2_core.cpp` (20), `vk_record.cpp` (10), `area_sample.h` (6):
  Modern for changed lines; Enhanced's additive lines kept where they fit.
  One dangling call this produced (`interp::mode40()` in vk_device) was
  fixed by adding the flag — see below.
- Java UI — `MainActivity.java` (22), `ControlsView.java` (18),
  `OptionsMenu.java` (6), `Native.java` (3), `GameDialog.java` (1),
  `CrashLogs.java` (12), `CrashLogProvider.java` (3), `strings.xml` (5),
  `build.gradle` (1), `AndroidManifest.xml` (1): Modern. Enhanced-only
  Java files (`IconForge.java`, `TouchIcons.java`) came in cleanly.
- `runtime/src/aspect.cpp` (20), `mods/cheats.cpp` (12), `jni_main.cpp` (11),
  `main.cpp` (7), `mods/turbo.cpp` (7), `mods/mods.h` (5), `ui_icons.cpp` (5),
  `platform.cpp` (4), `recomp/loader.cpp` (5), `recomp/locals.cpp` (4),
  `hle/ax.cpp` (3), `threads.cpp` (3), `recomp/compile.cpp` (2),
  `platform.h` (2), `vk.h` (2), `vk_record.h` (2), `hle/fs.cpp` (2),
  `CMakeLists.txt` (5), `README.md` (5): Modern for changed lines.
- `runtime/tools/recomp_llvm.cpp`, `gx2.h`, `gx2_resources.cpp`,
  `hle/padscore.cpp`, `mods/mods.cpp`, `recomp/emit.cpp`, `locals.h`,
  `savestate.cpp`, `LatteDecompiler.h`: Modern (1-2 hunks each).

## Big finding: v1.7's headline features have no published source

SSunnKing's v1.7 release notes list **HDR10 with in-game calibration,
GameCube look mode, and colour saturation 100–200%**. None of those exist
in any source this repo can see:
- Tags `1.4`, `1.5`, `1.7` in his repo **all point at the same commit**
  (`2f28a40`, "Release notes: version 1.4-enhanced").
- His branches are only `android` (= the common ancestor), `enhanced`
  (= `2f28a40`), and `optimizaciones` (older, v0.6/v0.7 era) — checked
  with `git ls-remote`, no HDR/saturation/GameCube-look strings in any of them.
- Neither does Modern's tree have them.

So they **could not be merged from source**. What Enhanced's source
actually brought in (and is in this tree): his frame-rate modes
(fps60.cpp: 30/60/true-60/40/120 + LSFG frame generation), bloom strength
setting, AO modes, aniso, console-accurate blur, his touch-icon set,
crash logs, turbo/peekz/fast-scene work, and the `fps_mode` setting.
If the HDR / GameCube-look / saturation code exists anywhere, it was
never pushed to that repo — it needs the source from SSunnKing, or a
fresh implementation. Do not claim this fork has them.

Correction to the earlier working notes: the **40 fps mode lives in
Enhanced**, not Modern. Modern's interp offers 60/120/240 fps
interpolation + true 60 + generated 60; Enhanced's fps60 offers
30/60/true-60/**40**/120. (Earlier messages had this flipped.)

## Modes — what controls what

Frame rate / look are **modes, not a merged system**. Only one runs at a time.

| Setting (in-game) | Native option | What runs |
|---|---|---|
| Graphics › Frame rate (fps60 modes) | `fps_mode` (0/1/3/4/5) | 0 = 30 fps, 1 = 60 fps interpolation (Modern's interp), 3 = true 60, **4 = 40 fps**, 5 = 120 fps. 40/120 only offered on 120 Hz panels. Wired in `jni_main.cpp` → `fps60::set_mode`. |
| Graphics › Frame rate (interp) | `interp_fps` (0/60/120/240) + `fps_generated` | Modern's own interpolation picker. Overlaps with the fps60 picker above — both are offered in OptionsMenu (see "still to do"). |
| Tick speed | `tick_speed` (1–9) | Whole-game fast-forward, see below. |

### 40 fps mode — flag only, core port pending (known limitation)

`interp::mode40()` / `set_mode40(bool)` now exist (`runtime/src/interp.h`,
`interp.cpp`) so Enhanced's `fps60.cpp` and `vk_device.cpp` link, and the
display-rate request (`vk_device.cpp`, 120 Hz for mode 4) works. But
**Modern's interpolation core has no 4-frames-per-3-steps path**, so mode 4
currently runs as 60 fps interpolation and logs
`[interp] 40 fps mode: core port pending, running 60 fps interpolation`.
Port task, if wanted: Enhanced's `interp.cpp` phase40 cycle (`g_phase40`,
`pass_t()` kW40 weights, hold-pass rules, `vsync_rate()`) into Modern's
`interp.cpp` pass structure. Not guessed at here.

## Tick speed — new feature (user's order)

Whole-game fast-forward as a setting, **1x–9x, single digit only** (no 99x
by accident). The game clock runs N times as fast, so the game plays N
logic ticks per real second. If the hardware can't keep up, frames just
come as fast as the device can draw them — the game runs slower than the
number asked, nothing breaks.

- `runtime/src/mods/turbo.cpp`: `mods::tick_speed()` / `set_tick_speed(v)`
  (clamped 1–9), env `WWHD_TICK_SPEED`. Wired into `after_execute()`: the
  clock rate is tick speed when above 1x, otherwise the existing event
  fast-forward rate (ZR). Reuses the proven `gx2::set_clock_rate` +
  `gfx::set_fast_forward` mechanism.
- `runtime/src/mods/mods.h`: declarations.
- `runtime/src/android/jni_main.cpp`: `tick_speed` in setOption/getOption.
- `android/.../OptionsMenu.java`: Graphics › "Tick speed" choice, 1x–9x list.
- `android/.../res/values/strings.xml`: `opt_tick_speed` (+ hint).
- `android/.../MainActivity.java`: saved value applied at startup via
  `WWHD_TICK_SPEED` env (same pattern as `interp_fps`).
- This is **not** the cutscene/dialogue fast-forward both ports already
  have (`mod_ff_cutscenes` / `mod_ff_dialogues`, ZR-held) — that is
  untouched and separate.

## Verification (done after the merge commit)

Per-file syntax check with the NDK Clang (arm64, `-fsyntax-only`, with the
project's own `cemu_shim.h` force-include and a stub `volk.h`): **89 of 110
runtime `.cpp` files pass, 0 fail, 21 skipped** (full log:
`~/workspace/wwhd-merge-check.log`). The 21 skipped files need third-party
headers CMake normally fetches (VMA `vk_mem_alloc.h`, zstd, LLVM, glslang,
libarchive) or macOS-only headers — they could not be checked here, and a
full `libwwhd.so` build also needs the user's own `game/code/cking.rpx`
to generate code (`tools/recomp/recomp.py`), which is not in this VM.
No APK build was attempted.

The check caught **4 latent bugs — all pre-existing in Modern 0.9.9c's HEAD,
not caused by the merge** (verified by checking the same files at HEAD) —
and they are fixed in this tree:

1. `runtime/src/mods/turbo.cpp`: `kStep` used but never defined (Modern).
   Fixed by defining `const StepParts& kStep = kDoorStep;` at the top of
   the quick-doors loop — behaviour unchanged, since `kDoorStep` has every
   called part set (only deletion is off, and that was already skipped).
2. `runtime/src/interp_fx.cpp`: `blend_pass()` called at 2 weather-effect
   sites but never defined (Modern renamed it `halfway()` and missed two
   call sites). Fixed by calling `halfway()`, the same guard the file
   already uses at 6+ similar sites.
3. `runtime/src/true60.h` / `true60.cpp`: the header declared `pass_begin`
   and `camera_draw_preview` as no-op inlines while the `.cpp` defined the
   real ones (redefinition errors), and `true60_link.cpp` used
   `true60::preview()`, which the header never declared (the `.cpp` does
   define it). Fixed by declaring the three real functions in the header.
4. `runtime/src/gx2/gx2_core.cpp`: used `g_vsync_epoch`, `render_sync()`,
   `display_vsync::start()`, `fps60::on_swap()`, `perf_hint::on_swap()` and
   `platform::apply_thread_cores()` with none of them defined or declared.
   Fixed by porting Enhanced's Android includes, its `render_sync()`
   definition, and the `set_core_mode` / `core_mode` / `apply_thread_cores`
   declarations in `platform.h` (the implementations were already in
   Modern's `platform.cpp`); `g_vsync_epoch` is defined locally as the
   startup time (the clock is re-based on every rate change, so that is
   the right base).

## Still to do / not verified

1. **No full build done.** Syntax check only (see above): 89 pass, 0 fail,
   21 skipped for missing fetched deps, Java files unchecked (no javac
   here). A real CMake build (which fetches VMA/zstd/LLVM/glslang) + the
   user's game dump is the next verification step — do not assume it links.
2. **40 fps core port** (above) and **HDR / GameCube look / saturation**
   (no source — see above).
3. **Two frame-rate settings overlap:** OptionsMenu offers both the fps60
   picker (`opt_fps`) and Modern's interpolation picker (`opt_interp`).
   They both drive the same interp core and can disagree (e.g. true 60
   shows as "off" in the interp picker). Unify into one mode list next.
4. Enhanced's 40 fps menu-animation fix and his `g_menu30` handling were
   dropped with his interp.cpp.
5. `README.md` still describes Modern 0.9.9c only; update it for the fork
   (tick speed, fps modes) before release.
