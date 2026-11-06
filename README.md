# FF Resonance (Switch → Android) — handoff for the port on Anbernic RG477M

This repo contains only **handoff notes** (no ROM, no APK, no game data).

- Target device: Anbernic RG477M — Dimensity 8300 / Mali-G615 / Android 14 — 1280×960
- Game: FINAL FANTASY RESONANCE, TitleID `0100B8A026F14000`, package `org.nx2a.t0100b8a026f14000`
- Engine: nx2android (nxnative/dm) + bcn layer, Asia [AS] v0 dump

## What to read first

1. **[`HANDOFF.md`](HANDOFF.md)** — full status: what's been fixed (with evidence), open bugs, how to build/measure, things not to do.
2. Section **5.1** in HANDOFF is the main bug to work on next: **absolute black 3D world during gameplay** (measured 0.11/255), while the **title screen renders perfectly**.

## 30-second summary

- Fixed: stuck splash (2 causes), 37 MB/s RAM leak (unclosed DMA-BUF fd), log spam, zero-copy cache sync, narrow barrier (Phase 2).
- Still open: black 3D world during gameplay (not a missing shader — null-FS and skip=0 were tried, still black), English text (not present in the AS dump), FPS 10–21.
- Currently running safe configuration: see HANDOFF section 3.
