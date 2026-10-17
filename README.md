# pmp-cli

**WARNING: THIS TOOL IS COMPLETELY VIBECODED**

PSP PMP video converter and compressor.

Docs: [docs/USAGE.md](docs/USAGE.md) — full command guide, presets, size
ladders. [docs/FORMAT.md](docs/FORMAT.md) — container + bitstream format and
the exact rules the real PSP Media Engine enforces.

## Installation

```bash
# Binary (no dependencies)
./dist/pmp-cli-bin convert video.mp4 output.pmp

# From source (requires Python 3.7+ and ffmpeg)
./pmp-cli convert video.mp4 output.pmp
```

## Quick Start

```bash
# Convert video to PMP
pmp-cli convert video.mp4 output.pmp

# Compress existing PMP (works directly on the container, see below)
pmp-cli compress input.pmp output.pmp --crf 30

# Batch convert directory
pmp-cli batch ./videos/ --batch-mode convert --output-dir ./pmp-files/
```

## Common Commands

```bash
# Maximum quality
pmp-cli convert movie.mp4 movie.pmp --crf 18 --preset veryslow

# PSP reference-compatible encode (uses bundled bin/x264)
pmp-cli convert video.mp4 output.pmp --x264-exe bin/x264

# Re-encode video only, keep every audio frame byte identical
pmp-cli compress input.pmp output.pmp --stream video --crf 30

# Re-encode audio only, video packets stay byte identical (no decode at all)
pmp-cli compress input.pmp output.pmp --stream audio --audio-bitrate 64

# Batch compress, replace originals
pmp-cli batch ./library/ --batch-mode compress --recursive --replace --crf 30
```

### How `compress` works

`compress` never builds an intermediate MP4. It reads the container and:

1. pulls the video packets out as a raw Annex B elementary stream and hands it
   straight to x264 (`--stream video/both`), or copies them byte for byte;
2. keeps the raw AAC/MP3 frames untouched unless `--stream audio/both` asks for
   a re-encode (the codec is taken from the header, so an AAC PMP stays AAC and
   an MP3 PMP stays MP3);
3. inherits the whole header (resolution, `vid_scale`/`vid_rate`,
   `aud_format`/`aud_scale`/`aud_rate`/`aud_stereo`) from the source file;
4. re-interleaves audio with the algorithm of the official `pmp_muxer_avc`.

The frame count of the source is preserved (no `PMP -> MP4 -> PMP` round trip),
and `first_delay`/`last_delay` satisfy the exact `time_math.pm` invariant of
the official `pmp_muxer_avc` (values in microseconds; `audio_span` runs from
the start of the *first* audio frame to the start of the *last* one, i.e.
`n_af - 1` audio periods — `last_delay` is measured from the last audio
frame's start and already covers that frame's playback):

```
first_delay + audio_span + last_delay == video_frame_duration
```

Every run prints `warning:` lines for anything a PSP player would reject
(see *PMP Format* below).

### PSP-compatible video (XviD4PSP 5 exact preset)

`pmp-cli` bundles a native x264 built with `--psp-compat`, which emits the exact
PSP reference SPS (`log2_max_frame_num_minus4=5`, `pic_order_cnt_type=0`,
`log2_max_poc_lsb_minus4=6`) — the same stream params XviD4PSP 5 produced.

Every encode is Main/CABAC with HRD VUI and per-frame pic_timing SEIs
(`--nal-hrd vbr --vbv 10000/10000`), matching the AU structure of all
ME-tested reference PMPs. Baseline/CAVLC hangs the real Media Engine ~5 s in
(ffmpeg/PPSSPP still play it fine — see docs/FORMAT.md §3).

- When `bin/x264` exists next to `pmp-cli` it is used automatically.
- `--x264-exe <path>` overrides the binary. A non-`.exe` build runs directly
  (native); a `.exe` (e.g. the old XviD4PSP 2009 build) runs via `wine`.

## Key Options

### Video
- `--crf 0-51` — Quality (lower = better, default 21; 27 ≈ good PSP balance)
- `--video-mode vbr|cbr` — CRF or constant bitrate (`--video-bitrate`)
- `--fps 1-29.97` — Frame rate (default: source)
- `--preset` — hyperfast..veryslow; `faster` ≈ 4x speed, `slow` = XviD4PSP HQ Extreme
- `--profile` — baseline, main (default), high — keep `main` for hardware
- `--vbv-maxrate/--vbv-bufsize` — HRD values, default 10000/10000 (= reference)
- `--width/--height` — Resolution (default: source, must be divisible by 16)
- `--stream video|audio|both` — What `compress` re-encodes (default: both)
- `--deinterlace` — Apply deinterlacing filter

### Audio
- `--audio-bitrate` — CBR bitrate (default 96; 48 = audible but fine trade)
- `--audio-quality 0.1-5.0` — VBR quality (lower = better)
- `--audio-rate` — Sample rate: 32000, 44100 (keep), 48000
- `--no-audio` — Strip audio

### Batch
- `-r, --recursive` — Process subdirectories
- `--replace` — Replace originals (atomic: temp file + swap)
- `--dry-run` — Show what would be done

Full options: run `pmp-cli convert --help`; usage recipes in
[docs/USAGE.md](docs/USAGE.md).

## PMP Format

PMP (Portable Media Player) is a PSP multimedia container
(H.264 video + AAC/MP3 audio). The full spec — header fields, packet
layout, A/V delay invariant, interleaving, and the bitstream rules the
real Media Engine enforces — is in [docs/FORMAT.md](docs/FORMAT.md).

Quick view:
```
Header (56 bytes) → 14 × uint32_t (little-endian)
Frame Index       → n_vframes × uint32_t (packet_size << 1 | keyframe_flag)
Packet Data       → one packet per video frame
                    [n_af][first_delay][last_delay][video_length]
                    [audio_lengths...][video AU][raw AAC frames]
```

Every run prints `warning:` lines for anything the reference PSP player
would reject (header fields, `aud_max_frames`, delays, AU size).

**Emulator vs real PSP:** PPSSPP does not run the hardware path at all — it
decodes PMP access units with ffmpeg and tolerates almost anything. On a
real PSP the ME decodes from a fixed DMA buffer; wrong profiles (baseline),
missing HRD SEIs or oversized AUs freeze or crash the console. That is why
a file can play in the emulator and die on the console.

## Building Binary

```bash
pip install pyinstaller
pyinstaller --onefile --name pmp-cli-bin pmp-cli
# Output: dist/pmp-cli-bin
```

## Troubleshooting

**Player crashes on PSP:**
- Read the `warning:` lines the tool prints - they are checks against the
  reference player source (header fields, `aud_max_frames`, delays, AUD order,
  oversized access units)
- Re-encode with the bundled `bin/x264` (default when present) — do **not**
  use `--profile baseline`: baseline/CAVLC freezes the real ME ~5 s in
- Keep the resolution at `480x272` or below and fps at `29.97` or below

**Freezes ~5 seconds into the video (system hard-lock):**
- The file is baseline/CAVLC or has no HRD VUI — re-encode with the default
  settings (Main + `--nal-hrd vbr`, see docs/FORMAT.md §3)

**No audio:**
- Check `--audio-rate 44100` and `--audio-channels stereo`

**Choppy playback:**
- Lower FPS: `--fps 25`
- Lower resolution: `--width 368 --height 208`
- Lower bitrate: `--crf 30`

## References

- [PMP-Mod GitHub](https://github.com/PSP-Archive/PMP-Mod) — Original player source
- [PMP Container Spec](https://web.archive.org/web/20080410063712fw_/http://jonny.leffe.dnsalias.com/pmp_mod/container.htm) — Format docs
- [pmp_muxer_avc 1.01 source](https://web.archive.org/web/20060711092320/http://jonny.leffe.dnsalias.com:80/pmp_mod_avc/rar/pmp_muxer_avc_1_01_source.rar)
  — the official H.264 muxer; `time_math.pm`/`interleaving.pm` define
  `first_delay`/`last_delay` and the keyframe rule used by this tool
- [pmp_mod_avc 1.02 source](https://web.archive.org/web/20060711092609/http://jonny.leffe.dnsalias.com:80/pmp_mod_avc/rar/pmp_mod_avc_1_02_source.rar)
  — the reference PSP player (`pmp_file.c` header checks, `avc.c` DMA into
  `MEAVCBUF = 0x4a000`, `pmp_read.c` packet parsing)
- [PPSSPP sceMpeg.cpp](https://github.com/hrydgard/ppsspp/blob/master/Core/HLE/sceMpeg.cpp)
  — emulator PMP path (`decodePmpVideo()`), explains why the emulator is lenient

## License

MIT
