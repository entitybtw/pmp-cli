# bin/x264 — native x264 with PSP-compat patch

`bin/x264` is a native (Linux, x86-64) x264 CLI build patched with `--psp-compat`,
used by `pmp-cli` to encode video with the exact PSP reference SPS
(`log2_max_frame_num_minus4=5`, `pic_order_cnt_type=0`) that XviD4PSP 5 produced.

No wine / Windows x264.exe is required.

## Rebuild

The patch adds a single CLI flag `--psp-compat` that forces:
- `sps->i_log2_max_frame_num = 9`  → log2_max_frame_num_minus4 = 5
- `pic_order_cnt_type = 0` when x264 would otherwise emit type 2
  (bframes=0 path)

```bash
git clone --filter=blob:none https://code.videolan.org/videolan/x264.git xu
cd xu
# apply encoder/set.c + x264.h + common/base.c + x264.c hunks shown in ../psp-compat.patch
./configure --disable-asm --disable-opencl --disable-lavf --enable-static
make -j$(nproc)
cp x264 ../bin/x264
```

A ready-to-apply patch is in `psp-compat.patch`.