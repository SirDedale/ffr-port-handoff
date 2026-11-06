# HANDOFF — Final Fantasy Resonance (0100B8A026F14000) on Anbernic RG477M

> Handoff notes for the AI/person continuing this work. Updated: 2026-11-02 (this work session).
> Read section 0 first, then section 5 (open bugs) — that's what needs to be done next.

---

## 0. TL;DR status

| Item | Status |
|---|---|
| Boot → splash → title screen | **OK** (renders nicely, English on static UI) |
| Entering gameplay (press Continue/Load) | Loads fine, but the **3D world is almost completely black** |
| Crashes from code bugs | None left (2 crash/stuck-splash bugs fixed) |
| RAM leak | **Fixed** (before: +37 MB/s → killed by lowmemorykiller; now flat) |
| Gameplay FPS | 10–21 FPS (720p handheld) |
| English text | **Not available** in the [AS] dump — only Korean/Chinese |

**Next thing to do:** fix the "black 3D world" — details in **section 5.1**.

---

## 1. Device & environment

- Device: **Anbernic RG477M**, adb serial `RG477M01184776`
- SoC: MediaTek **Dimensity 8300** (mt6897), GPU **ARM Mali-G615 MC6**, driver `mali_avalon r44p1`
- OS: Android 14 ARM64; screen **1280×960 (4:3)**, 60 Hz; 12 GB RAM + 12 GB zram swap
- Package: `org.nx2a.t0100b8a026f14000`; TitleID `0100B8A026F14000`
- Game data on device: `/sdcard/Android/data/org.nx2a.t0100b8a026f14000/files/` (RomFS ~12.4 GB)
- Original dump: **Asia [AS] v0** (`Final Fantasy Resonance [0100B8A026F14000][v0][AS][Base].nsp`)

⚠️ **NEVER run `adb uninstall`** — it will wipe the entire 12.4 GB RomFS. Always use `adb install -r`.

## 2. Paths & build commands (Windows PC)

```text
Project root : c:\Users\levan\.gemini\antigravity\scratch\ff_resonance_port
Engine       : ...\repo\nxnative\dm
Main files to edit:
   repo\nxnative\dm\android\gfx_vk_present.c   (swapchain, blit/copy, profiling, stretch)
   repo\nxnative\dm\android\gfx_vk_core.c      (pools, DMA-BUF zero-copy, cache sync)
   repo\nxnative\dm\android\gfx_vk_exec.c      (replay loop, pipeline, null-FS)
   repo\nxnative\dm\host\hle_movie.c           (VP9/AVC/AAC movie decoding)
   repo\nxnative\dm\host\crash.c               (watchdog, DM_DUMP_AT)
NDK          : ...\home\toolchains\android-ndk-r27c
Output .so   : repo\nxnative\dm\android\out\0100B8A026F14000\libdmhost.so
APK          : ...\game\FinalFantasyResonance.apk
Signing      : C:\Users\levan\AppData\Roaming\nx2android\signing-key.json
BCn layer    : ...\bcn_prebuilt\libbcn_layer.so
```

Build + package + install (run from the project root):

```bash
python repo/nxnative/dm/android/build.py --game 0100B8A026F14000 \
  --ndk home/toolchains/android-ndk-r27c --lib

python repo/nxnative/dm/android/apk/mkapk.py \
  --package org.nx2a.t0100b8a026f14000 --landscape \
  --extra-so "…/bcn_prebuilt/libbcn_layer.so" \
  --version-code <INCREMENT EACH TIME> \
  "…/out/0100B8A026F14000/libdmhost.so" \
  "…/game/FinalFantasyResonance.apk" \
  "C:/Users/levan/AppData/Roaming/nx2android/signing-key.json"

adb install -r "…/game/FinalFantasyResonance.apk"
adb shell "am force-stop org.nx2a.t0100b8a026f14000; sleep 1; monkey -p org.nx2a.t0100b8a026f14000 1"
```

- Version-code used so far: **10034** (use 10035 or higher).
- Inside the APK, `libdmhost.so` is packaged under the name **`lib/arm64-v8a/libprobe.so`** (verify with `unzip -l`).
- The APK **contains no game data**; RomFS/mods/shaders live on the sdcard → swapping to a different game build just means swapping those 3 folders/files.

## 3. Configuration CURRENTLY IN USE (stable, tested)

`/sdcard/Android/data/org.nx2a.t0100b8a026f14000/files/env.txt`
```
DM_LANG=en-US
DM_VK_ZC_SYNC=1
DM_VK_ZC_BUDGET_MB=1024
DM_OPMODE=0
```

`.../files/settings.txt`
```
overlay=1
rumble=1
docked=0
```

Notes:
- `DM_OPMODE=0` **alone is not enough**: if `settings.txt` has `docked=1`, it overrides this → the game renders at 1920×1080 instead of 1280×720. Fixed by setting `docked=0` → the present source becomes 1280×720.
- `DM_VK_STRETCH=1` = stretch to fill the whole screen (16:9 → 4:3 horizontally, vertical distortion). By default this should be **OFF** (keep the 120 px letterbox top/bottom, correct aspect ratio).
- To be absolutely sure there's no OOM: add `DM_VK_NO_ZEROCOPY=1` (slower but safe).

## 4. Bugs ALREADY FIXED (with evidence)

### 4.1 Stuck on splash screen (Square Enix logo frozen, "fake" 30 FPS)
- **Cause 1 — `gfx_vk_exec.c` replay loop**: on encountering an unknown record / `h.size < 8`, it would `return;` → **dropping all remaining draws for that frame**. Changed to log once + `p += h.size; continue;`; the `default:` case in the dispatch no longer `return;`s.
- **Cause 2 — DMA-BUF zero-copy `gfx_vk_core.c`**: MediaTek returns 3 fds:
  - `fd[0]` = `anon_inode:gralloc_extra` → mmap RW = EACCES, RO = ENODEV (this is why the original build failed → fell back to a shadow copy of 16–25 MB/frame)
  - `fd[1]` = `/dmabuf:G:pid` → mappable, the real buffer
  - `fd[2]` = `/memfd:gralloc_shared_memory`
  - Forcing use of `fds[1]` **without cache sync** → stale data → black screen after splash.
  - **Correct fix**: keep the fd (`dup`) + `DMA_BUF_IOCTL_SYNC` — `END|WRITE` before each submit, `START|READ` once the fence completes. Flag: `DM_VK_ZC_SYNC=1`.
- **Proven by A/B test**: repackaging the old `.so` (before this fix) → got stuck in exactly the same way ⇒ not caused by the barrier change.

### 4.2 Massive RAM leak (EGL mtrack +37 MB/s → app killed by lowmemorykiller)
- Logcat symptoms: `Zygote: Process <pid> exited due to signal 9 (Killed)` + `lowmemorykiller: ... device is low on swap ... thrashing (200%)`.
- Measurements: `EGL mtrack` 5.75 → 6.68 GB over 25 s; `MemAvailable` 1.6 → 0.72 GB.
- **Cause**: the DMA-BUF fd was `dup()`'d for syncing but **never `close()`'d when the pool was destroyed** → dmabuf held a reference → the gralloc buffer was never returned.
- **Fix**: `close(p->zc_fd)` in `mtl_pool_del()` (along with counting `AHardwareBuffer` made/freed).
- Result: `EGL mtrack` flat at 409 MB, `MemAvailable` flat at ~7 GB over several minutes.

### 4.3 Debug logging stalling the game thread
- `hle_nvn.c` had 3 unconditional `LOG()` calls in the hot path (`BeginRecording`, `AddCommandMemory`, `submit_now`) → **11 MB of log / 2 minutes**.
- Removed. Logging is now ~60 KB/session.

### 4.4 "Fake" 5 FPS stall caused by shadow copy
- Log: `submit+present 378 ms` while the GPU itself only took 47 ms ⇒ bottleneck on the host side.
- Measured `skipped (no program)` + `copied 16–25 MB`/submit ⇒ zero-copy was broken ⇒ memcpy of the whole pool on every submit.
- After zero-copy + budget were working: `submit+present` down to **36–70 ms**.

### 4.5 Phase 2 optimization (narrow barriers) — DONE
- `gfx_vk_present.c`: replaced `VK_PIPELINE_STAGE_ALL_COMMANDS_BIT` + `VK_ACCESS_MEMORY_READ|WRITE` with narrow masks in 3 places:
  - `UNDEFINED → TRANSFER_DST_OPTIMAL`: `TOP_OF_PIPE,0 → TRANSFER,TRANSFER_WRITE`
  - `TRANSFER_DST → COLOR_ATTACHMENT_OPTIMAL`: `TRANSFER,TRANSFER_WRITE → COLOR_ATTACHMENT_OUTPUT,COLOR_ATTACHMENT_READ|WRITE`
  - `→ PRESENT_SRC_KHR`: producer is TRANSFER or COLOR_ATTACHMENT_OUTPUT → `BOTTOM_OF_PIPE,0`
- Measured (1280×960 present, 300-frame window, 30 FPS): GPU present **224.7 → ~160 µs**, CPU present 1.18 → 0.66 ms, frametime held at 33.3 ms.

## 5. OPEN BUGS (top to bottom priority)

### 5.1 ⚠️ Fully black 3D world during gameplay (MAIN BUG)

**UPDATE 2026-11-03 (narrowed down, important):**
Using `DM_VK_PASSDUMP=2000` (+`DM_VK_PASSDUMP_DEPTH=1`) to dump every render target of a single gameplay frame:

| Pass (dump) | Size | Format | Mean luminance | Meaning |
|---|---|---|---|---|
| 01–04, 06, 08… | 768×436 | depth (129) | — | depth passes |
| **07** (and 05/09/11/18/20/22/28…) | 768×436 | **122 = B10G11R11_UFLOAT** (HDR scene color) | **20.2/255** | **SCENE RENDERED CORRECTLY**: cave, rocks, stairs, character, glowing orb all clearly visible |
| 13, 15 | 768×436 | 9 = R8_UNORM | 253 / 238 | mask/white texture |
| 23–26 | 32×32 | 97 = R16G16B16A16_SFLOAT | 82.6 / **255 255 255** | luminance/bloom chain — **saturated white** |
| 37 → 44 | 384×218 → 6×7 (mip down) | 97 | **14.35 → 1.5 → 1.0** | bloom downsample: **~10× brightness lost here** |
| 45 → 57 | 12×7 → 768×436 (mip up) | 97 | 0.0 → 1.85 → 0.44 | upsample |
| 58 | 1280×720 | 37 = R8G8B8A8_UNORM | **0.20** | composite: **crushed ~100×** |
| 59 | 1280×720 | 37 | 0.78 | composite + UI (UI still looks normally bright) |

Evidence images: pass 07 renders the scene correctly (`pd_07.png`), pass 58 is nearly black (`pd_58.png`).

**Ruled out:**
- Not the present path: forcing the swapchain to `R8G8B8A8_SRGB` (`DM_VK_PRESENT_SRGB=1`) makes the title screen washed-out/over-bright ⇒ the guest pixels are already sRGB, the present path is correct.
- Not a missing shader: `skipped (no program) = 0` in this scene, **no** `dispatch with no compute shader` lines.
- Not zero-copy/cache sync: tried `DM_VK_NO_ZEROCOPY=1`, the black screen behavior is unchanged.
- Not brightness: in-game Brightness = 10/10 + screen brightness 255/255, output is still 0.2/255.

**Conclusion:** the scene renders correctly at the HDR scene color stage (B10G11R11), but the **post-process chain (luminance/eye-adaptation → tonemap)** crushes it by ~100×. The 32×32 luminance chain is saturated at **white = 1.0** ⇒ auto-exposure thinks the scene is extremely bright and picks an extremely small exposure → world goes black, while the UI (drawn after tonemap) is unaffected.

**UPDATE 2 (misaligned texel buffer — strong lead):**
- Log: `[RED] buffer texture not in a pool or misaligned (format 0xa, 44 elements, inpool 1 off 0x1000c8 align 0x40 bpb 4)`
  ⇒ the texel buffer **is in the pool but its offset is 8 bytes off** from `minTexelBufferOffsetAlignment = 64`.
  The old port binds a **dummy view** for it ⇒ the shader reads incorrect data (suspected to be a post-process/brightness parameter table, 44 elements).
- Added an experimental fix `DM_VK_TBUF_FIX=1` (copy 176 bytes to an aligned scratch buffer then create a BufferView) — **the cave scene did NOT get brighter**, but the title screen's brightness changed ⇒ this buffer IS part of the post-process chain.
  ⇒ Correct direction: the buffer needs to be **updated every frame** (the guest rewrites it every frame; the current scratch copy is a one-time snapshot ⇒ stale). Need a persistent buffer with a copy on every submission, or a direct mapping of the aligned memory region.
- Side observation: **toggling in-game Brightness (0↔10) doesn't change image brightness at all** ⇒ the tonemap gain value is currently a wrong constant (consistent with exposure being crushed).

**Next steps (concrete):**
1. `DM_VK_TRACE_PASS=<pass>` / `DM_VK_TRACE_PASS=-2` to get the **shader key** for passes 23–26 (luminance) and 37/38 (where the 10× loss happens) → cross-reference against the UE shader list.
2. Dump the luminance chain specifically: `DM_VK_PASSDUMP_SIZE=32x32,384x218` and compare values against the scene color (B10G11R11). Suspicion: the luminance pass **samples the scene color incorrectly** (B10G11R11 packed format or dummy view) → returns 1.0.
3. Compare this same chain at the **title screen** (renders correctly) to see which pass differs.
4. If it's a texture binding issue → check `vkshader_t`/`stage_set` for the B10G11R11 target and the dummy views.

Measured evidence:
- Average frame brightness in-game: **0.11 / 255** (absolute black, not just "dark") — measured twice, 25 s apart; when exiting the game screen, measured 182/255.
- Title screen: renders **perfectly**, `skipped (no program) = 0` across 129,570 draws, correct colors.
- Gameplay: `skipped (no program) = 1.96 M / 3.27 M` (**55–58% of draws skipped**).
- Newly added diagnostic log: `[vk noprog] prog 0x0 nstage -1 rast_discard 0 | vs 0x… | fs 0x0` → the skipped draws are programs with **no fragment shader** (depth-only: depth prepass / shadow map).

Already tried (did not fix it):
- **Null fragment shader**: added `android/null_frag_spv.h` (glslc `void main(){}`) + `pkey.null_fs`, enabled via `DM_VK_NULLFS=1`.
  - Result: `skipped (no program)` → **0** across 43,608 draws, **but the scene is still black** ⇒ the black screen cause is NOT the skipped draws.
  - Note: once saw broken glyphs in a menu text with this flag on (possibly caught mid-redraw, not confirmed).
  - Default: **OFF**.
- Increased in-game brightness (Options → 그래픽/Graphics → 밝기 설정 = 10/10) and screen brightness 255/255: **didn't help** (since the output is absolute black).

Next directions (not yet done):
1. **Dump render targets per pass**: add a mode that writes each attachment to a PPM after every pass (or use the existing `DM_VK_SHOTS`/`shot()`) → find the first pass that turns black (suspects: lighting/composite/tonemap).
2. Compare the pass list between **title (correct)** and **gameplay (black)**: pass count, compute dispatch count, target format, blend state.
3. Check the **compute** passes (~148,000 dispatches) — UE5 uses compute for lighting/exposure.
4. Check the `DM_VK_X_*` flags already available in `gfx_vk_exec.c` to bisect (skip pass, disable variant…).
5. Look into `DM_VK_FDZ`, `DM_VK_FDZ=0`, `DM_VK_X_NOPRERESET`, `DM_VK_X_RDKEEP`… (read the code comments to see what they do).

### 5.2 English text: NOT feasible with the current dump
- Options → 텍스트 언어 only offers: `한국어`, `중국어(간체자)`, `중국어(번체자)` — **no English**.
- The static UI (New Game/Option/Auto/Log/Skip/SYSTEM MENU) is in English because it's hardcoded in the atlas, not part of localization.
- To get English text, we'd need a **US/EU dump** (different pak) to port from; or do a translation ourselves (very large effort: IoStore .ucas/.utoc 5.9 GB, would need `retoc`/ZenTools to export locres → translate → rebuild utoc + hashes + a font that supports the needed characters).

### 5.3 Low FPS during gameplay (10–21 FPS)
- Source: heavy guest-side rendering (GPU ~40–60 ms/frame) + host ~36 ms; no longer bottlenecked on copies.
- Tried `DM_RENDER_SCALE=0.75`: measured **slower** (~10.6 vs ~19 FPS) in a cutscene scene ⇒ **do not use**.
- Phase 3 (not yet done): 1280×720 handheld, compare `vkCmdCopyImage` (1:1) vs `vkCmdBlitImage` — code already exists: `DM_VK_PRESENT_COPY=1` + log `[PRESENT-SRC]`; the 1:1 condition has been confirmed to occur (`src 1280x720 → dst (0,120), scale 1.000, fmt match`).

## 6. Measurement & debug tools available in the build

| Flag / mechanism | Effect |
|---|---|
| `[PROFILE-300]` in `log.txt` | FPS avg/min/max, frametime p50/p95/p99, CPU (fence/acquire/clear/blit/submit/present), GPU (clear/blit/total), `path: copy/blit N frames` |
| `[PRESENT-SRC]` | source size, crop, destination, scale, whether it's 1:1 + format match |
| `DM_VK_ZC_PROBE=1` | prints link/flag/mmap info for each DMA-BUF fd |
| `[zc stats]` (every 200 submits) | pool count, live zero-copy MB, AHB made/freed |
| `DM_MOVIE_TRACE=1` | full movie trace + `[movie] vp9 cost` (ms/frame decoded) |
| `DM_MOVIE_THREADS=N` | number of libvpx threads (default 4) |
| `DM_DUMP_AT=<seconds>` | dumps thread state once after N seconds (GameThread/RenderThread/CRI…) |
| `DM_WATCH_STALL=<seconds>` | watchdog: dumps when presents stop, kills after threshold |
| `DM_VK_NO_ZEROCOPY=1` | disables zero-copy (safe but slow fallback) |
| `DM_VK_ZC_BUDGET_MB` | GPU memory cap for zero-copy (default 1024) |
| `DM_VK_STRETCH=1` | stretch to fill screen (removes letterbox) |
| `DM_VK_NULLFS=1` | enables null fragment shader for depth-only draws (currently OFF) |
| `DM_VK_ZC_SYNC=1` | cache maintenance for zero-copy (**required** when using zero-copy) |

Logs: `/sdcard/Android/data/org.nx2a.t0100b8a026f14000/files/log.txt` (rotated to `log.prev.txt` on every boot).

## 7. Lessons learned / things NOT to do

1. **Never run `adb uninstall`** (wipes 12.4 GB of data).
2. **Don't force FPS flags** (`DM_FPS_CAP`, `DM_FPS_PIN`, `DM_PRESENT_INTERVAL`, `DM_ADAPTIVE_FPS`, `DM_VK_PRESENT_MODE=mailbox`) — causes render/streaming desync during scene transitions.
3. **Never `return;` in the middle of the replay loop** — only log + `p += h.size; continue;`.
4. **Don't set `docked=1` in `settings.txt`** if you want 720p (it overrides `DM_OPMODE`).
5. When changing one variable, **A/B test the same scene**; measure with `[PROFILE-300]`, not by feel.
6. `adb input keyevent` only works on the title screen; in-game you must use physical buttons (or a pad). Port's key mapping: `KEYCODE_BUTTON_A(96) → DM_BTN_B`, `KEYCODE_BUTTON_B(97) → DM_BTN_A` (A/B are swapped), `ENTER(66)` = confirm on title screen, `BUTTON_START(108)` = port's menu.
7. Before concluding "it's the new build's fault," repackage the old `.so` into an APK (`mkapk.py --version-code <new>`) and test it — A/B on the device.

## 8. Existing artifacts

- Currently installed APK: versionCode **10034** (`.../game/FinalFantasyResonance.apk`)
- Phase-1 `.so` (before the barrier fix) kept at: `.../out/0100B8A026F14000/libdmhost.so.p1`
- Diagnostic screenshots/logs: `C:\Users\levan\AppData\Local\hermes\cache\scratch\ff_*.png`, `ff_mem.csv`

## 9. Things to do right away in the next session (in order)

1. Dump each pass/render target of a gameplay scene → identify the first pass that goes black.
2. Compare state between title (correct) and gameplay (black): HDR/RT format, blend, compute lighting.
3. If the lighting pass uses a shader that's `is not in bundle` → capture the shader (`captured_shaders/`) and re-port it.
4. Once the black screen is fixed → Phase 3 (copy vs 1:1 blit) + re-measure the whole table.
