# pmp-cli Usage Guide

`pmp-cli` converts and compresses PMP video for the PSP. It requires
Python 3.7+ and ffmpeg; a native x264 with the `--psp-compat` patch is
bundled in `bin/x264` and used automatically when present.

```bash
./pmp-cli <command> [options]

commands:
  convert   MP4/MKV/anything -> .pmp
  compress  .pmp -> .pmp (container-level, no MP4 round trip)
  decode    .pmp -> .mp4 (for checking on the PC)
  batch     many files / whole directories
```

The exact container and bitstream rules are documented in
[docs/FORMAT.md](FORMAT.md); this file is about using the tool.

## 1. convert — make a PMP from any video

```bash
# straight convert, x264 defaults
./pmp-cli convert video.mp4 output.pmp

# reference-quality: same encoder settings the original XviD4PSP PMPs used
./pmp-cli convert video.mp4 output.pmp --crf 21 --preset slow --profile main

# small file, PSP-tested bitstream (Main/CABAC + HRD + psp-compat SPS)
./pmp-cli convert video.mp4 output.pmp --crf 27 --preset faster --audio-bitrate 48
```

Video is scaled down to multiples of 16 automatically; for the PSP the
ceiling is 480x272 and 29.97 fps (both clamped/checked).

## 2. compress — re-pack or shrink an existing PMP

`compress` works directly on the container: video packets are extracted as
a raw Annex B stream and handed to x264, audio frames stay byte-identical
unless you ask for a re-encode. Frame count, timing and A/V interleave are
preserved exactly (see FORMAT.md §2).

```bash
# re-encode video + audio
./pmp-cli compress in.pmp out.pmp --crf 27 --preset faster --audio-bitrate 48

# video only — every audio frame stays byte identical
./pmp-cli compress in.pmp out.pmp --stream video --crf 30

# audio only — video packets are not touched at all
./pmp-cli compress in.pmp out.pmp --stream audio --audio-bitrate 64
```

Every run prints `warning:` lines for anything the real PSP player would
reject (header rules from FORMAT.md §1). Treat warnings as errors before
shipping to hardware.

## 3. batch — directories

```bash
# dry run: see every file pair, encode nothing
./pmp-cli batch ./videos/ -r --crf 27 --preset faster --dry-run

# compress a whole tree, replace originals atomically
./pmp-cli batch ./videos/ -r --replace \
    --crf 27 --preset faster --audio-bitrate 48

# convert a tree of MP4s into a separate output dir
./pmp-cli batch ./raw/ -r --batch-mode convert --output-dir ./pmp/
```

`--replace` is atomic: output is written to a temp file and swapped in
only after a successful mux, so an interrupted run never corrupts the
library. Run long jobs under `nohup`/tmux and follow the log.

`batch` is sequential and one encode already uses all cores; to
parallelise, split the tree manually and run several `batch` instances on
disjoint subdirectories.

## 4. Choosing settings

### Video: CRF / preset

Speed ladder (i9-10850K, 20 threads, 480x272; measured):

| preset | fps | notes |
|--------|-----|-------|
| faster | ~700 | **recommended for libraries** |
| slow | ~190 | XviD4PSP "HQ Extreme" exact knobs (`me=umh subme=9 trellis=2 ...`) |

CRF ladder on real MCSM PMPs (`faster` preset, ratio vs original):

| crf | size vs original | SSIM | 8.15 GiB library becomes |
|-----|------------------|------|--------------------------|
| 24 | ~45% | — | ~3.7 GiB |
| 25 | 44–48% | 0.978 | ~3.7 GiB |
| 26 | 40–44% | 0.975 | ~3.4 GiB |
| **27** | **37–40%** | **0.972** | **~3.1 GiB** ← good balance |
| 28 | 32–36% | 0.968 | ~2.7 GiB |
| 30 | ~29% | — | ~2.3 GiB |

On the 480x272 PSP screen crf 27 is visually indistinguishable from the
originals; crf 30 is the "noticeably smaller" point.

### Audio

| setting | bitrate | sounds |
|---------|---------|--------|
| `--audio-bitrate 96` | 96k AAC-LC | closest to the original HE-AAC 64k |
| `--audio-bitrate 48` | 48k AAC-LC | audibly worse but fine for dialogue (**recommended with crf 27**) |
| `--audio-bitrate 32` | 32k AAC-LC | clearly muffled, last resort |

Sample rate stays 44100/stereo by default — do not change them
(the reference player only accepts 44100).

### Never change these

Everything in FORMAT.md §3 is enforced automatically; do not override it:

- `--profile baseline` — **hangs the real ME** (~5 s, full system freeze).
  The default `main` is correct.
- `--extra-x264` with `--no-cabac`, `--pic-struct`, `--nal-hrd` removed —
  the ME needs CABAC + HRD.
- fps above 29.97, resolution above 480x272 — clamped/checked anyway.

`--preset ultrafast`/`superfast` disable CABAC in x264; `pmp-cli` forces
`--cabac` back on, but these presets are still a poor size/quality trade
for PSP content — use `faster`.

## 5. Recipes

**Shrink a whole game-video library to ~target size** (MCSM example:
8.15 GiB -> ~3.1 GiB, PSP-tested bitstream):

```bash
./pmp-cli batch /path/to/assets/video -r --replace \
    --crf 27 --preset faster --audio-bitrate 48 \
    2>&1 | tee compress.log
```

**Convert an anime episode for on-the-go PSP viewing:**

```bash
./pmp-cli convert ep01.mkv ep01.pmp --crf 24 --preset slow --audio-bitrate 96
```

**Only fix the audio of a PMP** (video stays byte identical):

```bash
./pmp-cli compress in.pmp out.pmp --stream audio --audio-bitrate 64
```

**Verify a PMP on the PC before testing on hardware:**

```bash
./pmp-cli decode out.pmp check.mp4    # ffmpeg decodes every frame
ffmpeg -i out.pmp -f null -           # error-free decode = bitstream OK
```

Emulators (PPSSPP) will play almost anything — they are not a hardware
verification. The only real test is a PSP; see FORMAT.md §4 for the
debugging tools.

## 6. Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `warning:` lines on every run | read them — they are reference-player rules (FORMAT.md §1); fix before shipping |
| PSP freezes ~5 s into the video | baseline/CAVLC or missing HRD — the file was encoded without the bundled `bin/x264`; re-encode with defaults |
| PSP crashes instantly | oversized AU (> 256 KiB) or bad `maxaf` — warnings should have told you |
| `framerate is too high` from muxer logic | fps above the audio frame rate (43.07 AAC) — lower `--fps` |
| Encode too slow | `--preset faster` (~4x) or split the batch over several processes |
| Audio out of sync | only possible in hand-edited files; `compress` recomputes delays from the invariant |
