# bin/x264 — native x264 with PSP-compat patch

`bin/x264` is a native (Linux, x86-64) x264 CLI build patched with `--psp-compat`,
used by `pmp-cli` to encode video with the exact PSP reference SPS
(`log2_max_frame_num_minus4=5`, `pic_order_cnt_type=0`) that XviD4PSP 5 produced.

No wine / Windows x264.exe is required.

`pmp-cli` pairs it with `--profile main --nal-hrd vbr --vbv-*`: every
ME-tested reference PMP is Main/CABAC with HRD VUI and per-frame pic_timing
SEIs (`AU = [AUD,SPS,PPS,SEI,SEI,slice]` on IDR, `[AUD,SEI,slice]` on P).
Baseline/CAVLC encodes decode cleanly in ffmpeg/PPSSPP but hang the real
Media Engine at the first mid-GOP IDR — never ship baseline to hardware.

## Rebuild

The patch adds a single CLI flag `--psp-compat` that forces:
- `sps->i_log2_max_frame_num = 9`  → log2_max_frame_num_minus4 = 5
- `pic_order_cnt_type = 0` when x264 would otherwise emit type 2
  (bframes=0 path)
- `sps->i_log2_max_poc_lsb = 10` → log2_max_poc_lsb_minus4 = 6.
  x264's minimal window (4 bits) makes pic_order_cnt_lsb wrap every
  8 frames; the PSP ME decoder hangs on the MSB wraparound, so the
  field is widened to 10 bits (matches the XviD4PSP reference).
- skips the x264 "info" version SEI (600+ byte user-data SEI in
  frame 0); the reference bitstream has no such SEI.

```bash
git clone --filter=blob:none https://code.videolan.org/videolan/x264.git xu
cd xu
# apply encoder/set.c + x264.h + common/base.c + x264.c hunks shown in ../psp-compat.patch
./configure --disable-asm --disable-opencl --disable-lavf --enable-static
make -j$(nproc)
cp x264 ../bin/x264
```

A ready-to-apply patch is in `psp-compat.patch`.