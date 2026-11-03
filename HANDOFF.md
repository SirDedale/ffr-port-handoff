# HANDOFF — Final Fantasy Resonance (0100B8A026F14000) trên Anbernic RG477M

> Note bàn giao cho AI/người tiếp tục. Cập nhật: 2026-11-02 (phiên làm việc này).
> Đọc mục 0 trước, sau đó mục 5 (bug còn mở) — đó là việc cần làm tiếp.

---

## 0. TL;DR trạng thái

| Hạng mục | Trạng thái |
|---|---|
| Boot → splash → title screen | **OK** (render đẹp, tiếng Anh ở UI tĩnh) |
| Vào gameplay (ấn Continue/Load) | Vào được, nhưng **thế giới 3D gần như đen tuyền** |
| Crash do lỗi code | Không còn (2 lỗi crash/stuck splash đã fix) |
| Rò rỉ RAM | **Đã fix** (trước: +37 MB/s → bị lowmemorykiller giết; nay phẳng) |
| FPS gameplay | 10–21 FPS (720p handheld) |
| Tiếng Anh cho text | **Không có** trong bản dump [AS] — chỉ Hàn/Trung |

**Việc cần làm tiếp:** fix "thế giới 3D đen" — chi tiết ở **mục 5.1**.

---

## 1. Thiết bị & môi trường

- Thiết bị: **Anbernic RG477M**, serial adb `RG477M01184776`
- SoC: MediaTek **Dimensity 8300** (mt6897), GPU **ARM Mali-G615 MC6**, driver `mali_avalon r44p1`
- OS: Android 14 ARM64; màn hình **1280×960 (4:3)**, 60 Hz; RAM 12 GB + zram swap 12 GB
- Package: `org.nx2a.t0100b8a026f14000`; TitleID `0100B8A026F14000`
- Dữ liệu game trên máy: `/sdcard/Android/data/org.nx2a.t0100b8a026f14000/files/` (RomFS ~12.4 GB)
- Dump gốc: **bản Asia [AS] v0** (`Final Fantasy Resonance [0100B8A026F14000][v0][AS][Base].nsp`)

⚠️ **TUYỆT ĐỐI KHÔNG `adb uninstall`** — sẽ mất toàn bộ 12.4 GB RomFS. Luôn `adb install -r`.

## 2. Đường dẫn & lệnh build (PC Windows)

```text
Gốc dự án : c:\Users\levan\.gemini\antigravity\scratch\ff_resonance_port
Engine    : ...\repo\nxnative\dm
File cần sửa chính:
   repo\nxnative\dm\android\gfx_vk_present.c   (swapchain, blit/copy, profile, stretch)
   repo\nxnative\dm\android\gfx_vk_core.c      (pools, DMA-BUF zero-copy, cache sync)
   repo\nxnative\dm\android\gfx_vk_exec.c      (replay loop, pipeline, null-FS)
   repo\nxnative\dm\host\hle_movie.c           (VP9/AVC/AAC giải mã phim)
   repo\nxnative\dm\host\crash.c               (watchdog, DM_DUMP_AT)
NDK       : ...\home\toolchains\android-ndk-r27c
Output so : repo\nxnative\dm\android\out\0100B8A026F14000\libdmhost.so
APK       : ...\game\FinalFantasyResonance.apk
Signing   : C:\Users\levan\AppData\Roaming\nx2android\signing-key.json
BCn layer : ...\bcn_prebuilt\libbcn_layer.so
```

Build + đóng gói + cài (chạy từ gốc dự án):

```bash
python repo/nxnative/dm/android/build.py --game 0100B8A026F14000 \
  --ndk home/toolchains/android-ndk-r27c --lib

python repo/nxnative/dm/android/apk/mkapk.py \
  --package org.nx2a.t0100b8a026f14000 --landscape \
  --extra-so "…/bcn_prebuilt/libbcn_layer.so" \
  --version-code <TĂNG MỖI LẦN> \
  "…/out/0100B8A026F14000/libdmhost.so" \
  "…/game/FinalFantasyResonance.apk" \
  "C:/Users/levan/AppData/Roaming/nx2android/signing-key.json"

adb install -r "…/game/FinalFantasyResonance.apk"
adb shell "am force-stop org.nx2a.t0100b8a026f14000; sleep 1; monkey -p org.nx2a.t0100b8a026f14000 1"
```

- Version-code đã dùng tới: **10034** (dùng 10035 trở lên).
- Trong APK, `libdmhost.so` được đóng tên là **`lib/arm64-v8a/libprobe.so`** (kiểm tra bằng `unzip -l`).
- APK **không chứa dữ liệu game**; RomFS/mods/shaders nằm trên sdcard → thay bản game khác = thay 3 thư mục/tệp đó.

## 3. Cấu hình ĐANG DÙNG (ổn định, đã test)

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

Ghi chú:
- `DM_OPMODE=0` **một mình không đủ**: `settings.txt` có `docked=1` sẽ ghi đè → game render 1920×1080 thay vì 1280×720. Đã sửa `docked=0` → nguồn present 1280×720.
- `DM_VK_STRETCH=1` = kéo giãn đầy màn (16:9 → 4:3 ngang, méo dọc). Mặc định nên **TẮT** (giữ letterbox 120 px trên/dưới, đúng tỉ lệ).
- Muốn chắc chắn không OOM: thêm `DM_VK_NO_ZEROCOPY=1` (chậm hơn nhưng an toàn).

## 4. Bug ĐÃ FIX (kèm bằng chứng)

### 4.1 Stuck ở splash screen (Square Enix logo đứng yên, 30 FPS "ảo")
- **Nguyên nhân 1 — `gfx_vk_exec.c` replay loop**: gặp record lạ/`h.size < 8` thì `return;` → **bỏ toàn bộ draw còn lại của frame**. Đã đổi thành log 1 lần + `p += h.size; continue;`, `default:` trong dispatch không còn `return;`.
- **Nguyên nhân 2 — DMA-BUF zero-copy `gfx_vk_core.c`**: MediaTek trả 3 fd:
  - `fd[0]` = `anon_inode:gralloc_extra` → mmap RW = EACCES, RO = ENODEV (đây là lý do bản gốc fail → rơi vào shadow copy 16–25 MB/frame)
  - `fd[1]` = `/dmabuf:G:pid` → map được, là buffer thật
  - `fd[2]` = `/memfd:gralloc_shared_memory`
  - Ép `fds[1]` **không sync cache** → dữ liệu stale → màn hình đen sau splash.
  - **Fix đúng**: giữ fd (`dup`) + `DMA_BUF_IOCTL_SYNC` — `END|WRITE` trước mỗi submit, `START|READ` khi fence xong. Cờ: `DM_VK_ZC_SYNC=1`.
- **A/B đã chứng minh**: đóng gói `.so` cũ (trước khi sửa) → cũng kẹt y hệt ⇒ không phải do phần barrier.

### 4.2 Rò rỉ RAM khổng lồ (EGL mtrack +37 MB/s → lowmemorykiller giết app)
- Triệu chứng logcat: `Zygote: Process <pid> exited due to signal 9 (Killed)` + `lowmemorykiller: ... device is low on swap ... thrashing (200%)`.
- Số đo: `EGL mtrack` 5.75 → 6.68 GB trong 25 s; `MemAvailable` 1.6 → 0.72 GB.
- **Nguyên nhân**: fd của DMA-BUF được `dup()` để sync nhưng **không `close()` khi huỷ pool** → dmabuf giữ reference → buffer gralloc không bao giờ được trả.
- **Fix**: `close(p->zc_fd)` trong `mtl_pool_del()` (kèm đếm `AHardwareBuffer` made/freed).
- Kết quả: `EGL mtrack` phẳng 409 MB, `MemAvailable` phẳng ~7 GB trong nhiều phút.

### 4.3 Debug log làm nghẽn game thread
- `hle_nvn.c` có 3 dòng `LOG()` không điều kiện trong hot path (`BeginRecording`, `AddCommandMemory`, `submit_now`) → **11 MB log / 2 phút**.
- Đã bỏ. Log còn ~60 KB/phiên.

### 4.4 Kẹt FPS "ảo" 5 FPS do shadow copy
- Log: `submit+present 378 ms` trong khi GPU chỉ 47 ms ⇒ nghẽn ở host.
- Đo được `skipped (no program)` + `copied 16–25 MB`/submit ⇒ zero-copy hỏng ⇒ memcpy toàn pool mỗi submit.
- Sau khi zero-copy chạy + budget: `submit+present` còn **36–70 ms**.

### 4.5 Tối ưu Giai đoạn 2 (barrier hẹp) — ĐÃ XONG
- `gfx_vk_present.c`: thay `VK_PIPELINE_STAGE_ALL_COMMANDS_BIT` + `VK_ACCESS_MEMORY_READ|WRITE` bằng mask hẹp cho 3 chỗ:
  - `UNDEFINED → TRANSFER_DST_OPTIMAL`: `TOP_OF_PIPE,0 → TRANSFER,TRANSFER_WRITE`
  - `TRANSFER_DST → COLOR_ATTACHMENT_OPTIMAL`: `TRANSFER,TRANSFER_WRITE → COLOR_ATTACHMENT_OUTPUT,COLOR_ATTACHMENT_READ|WRITE`
  - `→ PRESENT_SRC_KHR`: producer là TRANSFER hoặc COLOR_ATTACHMENT_OUTPUT → `BOTTOM_OF_PIPE,0`
- Đo (1280×960 present, 300 frame/window, 30 FPS): GPU present **224.7 → ~160 µs**, CPU present 1.18 → 0.66 ms, frametime giữ 33.3 ms.

## 5. BUG CÒN MỞ (ưu tiên từ trên xuống)

### 5.1 ⚠️ Thế giới 3D đen tuyền trong gameplay (BUG CHÍNH)

**CẬP NHẬT 2026-11-03 (đã khoanh vùng được, quan trọng):**
Dùng `DM_VK_PASSDUMP=2000` (+`DM_VK_PASSDUMP_DEPTH=1`) để dump từng render target của 1 frame trong gameplay:

| Pass (dump) | Size | Format | Mean luminance | Ý nghĩa |
|---|---|---|---|---|
| 01–04, 06, 08… | 768×436 | depth (129) | — | các pass depth |
| **07** (và 05/09/11/18/20/22/28…) | 768×436 | **122 = B10G11R11_UFLOAT** (HDR scene color) | **20.2/255** | **CẢNH ĐƯỢC RENDER ĐÚNG**: thấy rõ hang, đá, bậc thang, nhân vật, quả cầu sáng |
| 13, 15 | 768×436 | 9 = R8_UNORM | 253 / 238 | mask/white texture |
| 23–26 | 32×32 | 97 = R16G16B16A16_SFLOAT | 82.6 / **255 255 255** | chuỗi luminance/bloom — **bão hoà trắng** |
| 37 → 44 | 384×218 → 6×7 (mip down) | 97 | **14.35 → 1.5 → 1.0** | bloom down-sample: **mất ~10× độ sáng ở đây** |
| 45 → 57 | 12×7 → 768×436 (mip up) | 97 | 0.0 → 1.85 → 0.44 | up-sample |
| 58 | 1280×720 | 37 = R8G8B8A8_UNORM | **0.20** | composite: **bị crush ~100×** |
| 59 | 1280×720 | 37 | 0.78 | composite + UI (UI vẫn sáng bình thường) |

Ảnh chứng cứ: pass 07 render đúng cảnh (`pd_07.png`), pass 58 gần như đen (`pd_58.png`).

**Đã loại trừ:**
- Không phải present: ép swapchain sang `R8G8B8A8_SRGB` (`DM_VK_PRESENT_SRGB=1`) làm title **bạt màu/over-bright** ⇒ pixel guest đã là sRGB, đường present đúng.
- Không phải thiếu shader: `skipped (no program) = 0` ở cảnh này, **không** có dòng `dispatch with no compute shader`.
- Không phải zero-copy/cache sync: đã thử `DM_VK_NO_ZEROCOPY=1`, hành vi đen không đổi.
- Không phải brightness: game Brightness = 10/10 + màn hình 255/255, output vẫn 0.2/255.

**Kết luận:** cảnh render đúng ở HDR scene color (B10G11R11), nhưng **chuỗi post-process (luminance/eye-adaptation → tonemap)** kéo tụt ~100×. Chuỗi luminance 32×32 bão hoà **trắng = 1.0** ⇒ auto-exposure tưởng cảnh rất sáng nên chọn exposure cực nhỏ → world đen, UI (vẽ sau tonemap) không bị ảnh hưởng.

**CẬP NHẬT 2 (texel buffer misaligned — manh mối mạnh):**
- Log: `[RED] buffer texture not in a pool or misaligned (format 0xa, 44 elements, inpool 1 off 0x1000c8 align 0x40 bpb 4)`
  ⇒ texel buffer **nằm trong pool nhưng offset lệch 8 byte** so với `minTexelBufferOffsetAlignment = 64`.
  Port cũ bind **dummy view** cho nó ⇒ shader đọc dữ liệu sai (nghi là bảng thông số post-process/brightness, 44 phần tử).
- Đã thêm fix thử nghiệm `DM_VK_TBUF_FIX=1` (copy 176 byte sang scratch căn lề rồi tạo BufferView) — **cảnh hang KHÔNG sáng lên**, nhưng title đổi độ sáng ⇒ buffer này có tham gia chuỗi post.
  ⇒ Hướng đúng: buffer phải được **cập nhật mỗi frame** (guest ghi lại mỗi frame; bản scratch hiện tại là ảnh chụp một lần ⇒ stale). Cần buffer persistent + copy mỗi submission, hoặc map thẳng vùng nhớ căn lề.
- Quan sát phụ: **đổi Brightness trong game (0↔10) không làm thay đổi độ sáng ảnh** ⇒ giá trị gain của tonemap đang là hằng số sai (khớp với việc exposure bị crush).

**Bước tiếp theo (cụ thể):**
1. `DM_VK_TRACE_PASS=<pass>` / `DM_VK_TRACE_PASS=-2` để lấy **shader key** của các pass 23–26 (luminance) và 37/38 (nơi mất 10×) → đối chiếu với danh sách shader UE.
2. Dump riêng chuỗi luminance: `DM_VK_PASSDUMP_SIZE=32x32,384x218` và so giá trị với scene color (B10G11R11). Nghi vấn: pass luminance **sample scene color sai** (format pack B10G11R11 hoặc dummy view) → trả 1.0.
3. So sánh cùng chuỗi này ở **title screen** (render đúng) để thấy pass nào khác biệt.
4. Nếu là texture bind sai → kiểm tra `vkshader_t`/`stage_set` cho target B10G11R11 và các view dummy.

Bằng chứng đo được:
- Độ sáng trung bình khung hình trong game: **0.11 / 255** (đen tuyệt đối, không phải "tối") — đo 2 lần cách 25 s; khi thoát khỏi màn game đo được 182/255.
- Title screen: render **hoàn hảo**, `skipped (no program) = 0` trên 129.570 draw, màu đúng.
- Gameplay: `skipped (no program) = 1.96 M / 3.27 M` (**55–58% draw bị bỏ**).
- Log chẩn đoán mới thêm: `[vk noprog] prog 0x0 nstage -1 rast_discard 0 | vs 0x… | fs 0x0` → các draw bị bỏ là **program không có fragment shader** (depth-only: depth prepass / shadow map).

Đã thử (không giải quyết được):
- **Null fragment shader**: thêm `android/null_frag_spv.h` (glslc `void main(){}`) + `pkey.null_fs`, bật bằng `DM_VK_NULLFS=1`.
  - Kết quả: `skipped (no program)` → **0** trên 43.608 draw, **nhưng cảnh vẫn đen** ⇒ nguyên nhân đen KHÔNG phải mấy draw bị bỏ.
  - Lưu ý: có 1 lần thấy text menu bị vỡ glyph khi bật cờ này (có thể do chụp giữa lúc redraw, chưa khẳng định).
  - Mặc định để **TẮT**.
- Tăng brightness trong game (Options → 그래픽/Graphics → 밝기 설정 = 10/10) và độ sáng màn hình 255/255: **không cứu được** (vì output là đen tuyệt đối).

Hướng đi tiếp (chưa làm):
1. **Dump render target theo pass**: thêm chế độ ghi từng attachment ra PPM sau mỗi pass (hoặc dùng `DM_VK_SHOTS`/`shot()` có sẵn) → tìm pass đầu tiên ra đen (nghi: lighting/composite/tone-map).
2. So sánh pass list giữa **title (đẹp)** và **gameplay (đen)**: số pass, số dispatch compute, target format, blend state.
3. Kiểm tra các pass **compute** (dispatches ~148.000) — UE5 dùng compute cho lighting/exposure.
4. Kiểm tra `DM_VK_X_*` flags có sẵn trong `gfx_vk_exec.c` để bisect (skip pass, tắt variant…).
5. Xem xét `DM_VK_FDZ`, `DM_VK_FDZ=0`, `DM_VK_X_NOPRERESET`, `DM_VK_X_RDKEEP`… (đọc comment trong code để biết tác dụng).

### 5.2 Text tiếng Anh: KHÔNG khả thi với dump hiện tại
- Options → 텍스트 언어 chỉ có: `한국어`, `중국어(간체자)`, `중국어(번체자)` — **không có English**.
- UI tĩnh (New Game/Option/Auto/Log/Skip/SYSTEM MENU) là tiếng Anh vì hardcode trong atlas, không thuộc localization.
- Muốn text tiếng Anh ⇒ phải có **dump US/EU** (pak khác) rồi port lại; hoặc Việt hoá (rất lớn: IoStore .ucas/.utoc 5.9 GB, cần `retoc`/ZenTools export locres → dịch → rebuild utoc + hash + font hỗ trợ dấu).

### 5.3 FPS thấp trong gameplay (10–21 FPS)
- Nguồn: guest render nặng (GPU ~40–60 ms/frame) + host ~36 ms; không còn nghẽn copy.
- Đã thử `DM_RENDER_SCALE=0.75`: đo được **chậm hơn** (~10.6 vs ~19 FPS) ở cảnh cutscene ⇒ **không dùng**.
- Giai đoạn 3 (chưa làm): 1280×720 handheld, so `vkCmdCopyImage` (1:1) vs `vkCmdBlitImage` — code đã có sẵn: `DM_VK_PRESENT_COPY=1` + log `[PRESENT-SRC]`; điều kiện 1:1 đã xác nhận có xảy ra (`src 1280x720 → dst (0,120), scale 1.000, fmt match`).

## 6. Công cụ đo & debug có sẵn trong build

| Cờ / cơ chế | Tác dụng |
|---|---|
| `[PROFILE-300]` trong `log.txt` | FPS avg/min/max, frametime p50/p95/p99, CPU (fence/acquire/clear/blit/submit/present), GPU (clear/blit/total), `path: copy/blit N frames` |
| `[PRESENT-SRC]` | kích thước nguồn, crop, đích, scale, có 1:1 + format match không |
| `DM_VK_ZC_PROBE=1` | in link/flag/mmap của từng fd DMA-BUF |
| `[zc stats]` (mỗi 200 submit) | số pool, zero-copy live MB, AHB made/freed |
| `DM_MOVIE_TRACE=1` | trace đầy đủ movie + `[movie] vp9 cost` (ms/frame giải mã) |
| `DM_MOVIE_THREADS=N` | số thread libvpx (mặc định 4) |
| `DM_DUMP_AT=<giây>` | dump thread state 1 lần sau N giây (GameThread/RenderThread/CRI…) |
| `DM_WATCH_STALL=<giây>` | watchdog: dump khi presents ngừng, kill sau ngưỡng |
| `DM_VK_NO_ZEROCOPY=1` | tắt zero-copy (fallback an toàn, chậm) |
| `DM_VK_ZC_BUDGET_MB` | trần GPU memory cho zero-copy (mặc định 1024) |
| `DM_VK_STRETCH=1` | kéo giãn đầy màn (bỏ letterbox) |
| `DM_VK_NULLFS=1` | bật null fragment shader cho draw depth-only (đang TẮT) |
| `DM_VK_ZC_SYNC=1` | cache maintenance cho zero-copy (**bắt buộc** nếu dùng zero-copy) |

Lấy log: `/sdcard/Android/data/org.nx2a.t0100b8a026f14000/files/log.txt` (mỗi lần boot xoay thành `log.prev.txt`).

## 7. Bài học / điều KHÔNG được làm

1. **Không `adb uninstall`** (mất 12.4 GB dữ liệu).
2. **Không ép cờ FPS** (`DM_FPS_CAP`, `DM_FPS_PIN`, `DM_PRESENT_INTERVAL`, `DM_ADAPTIVE_FPS`, `DM_VK_PRESENT_MODE=mailbox`) — gây lệch nhịp render/streaming khi chuyển cảnh.
3. **Không `return;` giữa replay loop** — chỉ log + `p += h.size; continue;`.
4. **Không đặt `docked=1` trong `settings.txt`** nếu muốn 720p (nó ghi đè `DM_OPMODE`).
5. Khi thay đổi 1 biến, **A/B cùng cảnh**; đo bằng `[PROFILE-300]` chứ không cảm giác.
6. `adb input keyevent` chỉ ăn ở title screen; trong game phải bấm tay (hoặc pad). Key map của port: `KEYCODE_BUTTON_A(96) → DM_BTN_B`, `KEYCODE_BUTTON_B(97) → DM_BTN_A` (A/B bị hoán đổi), `ENTER(66)` = confirm ở title, `BUTTON_START(108)` = menu của port.
7. Trước khi kết luận "do bản mới", hãy đóng gói `.so` cũ thành APK (`mkapk.py --version-code <mới>`) và cài thử — A/B trên máy.

## 8. Artifact hiện có

- APK đang cài: versionCode **10034** (`.../game/FinalFantasyResonance.apk`)
- `.so` bản Giai đoạn-1 (trước khi sửa barrier) giữ tại: `.../out/0100B8A026F14000/libdmhost.so.p1`
- Screenshot/log chẩn đoán: `C:\Users\levan\AppData\Local\hermes\cache\scratch\ff_*.png`, `ff_mem.csv`

## 9. Việc nên làm ngay ở phiên sau (thứ tự)

1. Dump từng pass/render target của cảnh gameplay → xác định pass đầu tiên đen.
2. So sánh state giữa title (đẹp) và gameplay (đen): format HDR/RT, blend, compute lighting.
3. Nếu pass lighting dùng shader có `is not in bundle` → bắt shader (`captured_shaders/`) rồi re-port.
4. Sau khi hết đen → Giai đoạn 3 (copy vs blit 1:1) + đo lại toàn bộ bảng số.
