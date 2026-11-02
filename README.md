# FF Resonance (Switch → Android) — handoff cho port trên Anbernic RG477M

Repo này chỉ chứa **note bàn giao** (không chứa ROM, không chứa APK, không chứa dữ liệu game).

- Máy đích: Anbernic RG477M — Dimensity 8300 / Mali-G615 / Android 14 — 1280×960
- Game: FINAL FANTASY RESONANCE, TitleID `0100B8A026F14000`, package `org.nx2a.t0100b8a026f14000`
- Engine: nx2android (nxnative/dm) + bcn layer, dump bản Asia [AS] v0

## Đọc gì trước

1. **[`HANDOFF.md`](HANDOFF.md)** — toàn bộ trạng thái: đã fix gì (kèm bằng chứng), bug còn mở, cách build/đo, điều cấm.
2. Mục **5.1** trong HANDOFF là bug chính cần làm tiếp: **thế giới 3D đen tuyệt đối trong gameplay** (đo được 0.11/255), trong khi **title screen render hoàn hảo**.

## Tóm tắt 30 giây

- Đã fix: stuck splash (2 nguyên nhân), rò rỉ RAM 37 MB/s (fd DMA-BUF không close), log spam, zero-copy cache sync, barrier hẹp (Giai đoạn 2).
- Còn mở: world 3D đen trong gameplay (không phải thiếu shader — đã thử null-FS, skip 0 mà vẫn đen), text tiếng Anh (dump AS không có), FPS 10–21.
- Cấu hình đang chạy an toàn: xem HANDOFF mục 3.
