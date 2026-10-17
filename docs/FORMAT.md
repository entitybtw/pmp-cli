# PMP Container & Bitstream Format

Everything this document describes is either taken from the official
reference sources (`pmp_muxer_avc` 1.01, `pmp_mod_avc` 1.02) or was verified
byte-by-byte against the 11 ME-tested reference PMPs of the PSP port of
Minecraft: Story Mode (encoded by XviD4PSP 5) and reproduced by `pmp-cli`.

## 1. Container layout

```
+-------------------+  offset 0
| header (56 bytes) |  14 x uint32_t, little-endian
+-------------------+  offset 56
| frame index       |  n_vframes x uint32_t
+-------------------+  offset 56 + 4*n_vframes
| packet 0          |
| packet 1          |  one packet per video frame,
| ...               |  in playback order
+-------------------+
```

### 1.1 Header (offset 0, 14 x uint32 LE)

| # | Field | Value / rule |
|---|-------|--------------|
| 0 | `magic` | `0x6d706d70` — ASCII `pmpm` (bytes on disk: `70 6d 70 6d`) |
| 1 | `version` | `1` |
| 2 | `video.format` | `1` = H.264/AVC, `0` = MPEG-4 ASP (legacy) |
| 3 | `video.number_of_frames` | number of packets = index length |
| 4 | `width` | multiple of 16; `<= 480` for H.264 |
| 5 | `height` | multiple of 16; `<= 272` for H.264 |
| 6 | `video.scale` | fps = `vid_rate / vid_scale` |
| 7 | `video.rate` | e.g. `29970/1000` = 29.97, `25000/1000` = 25 |
| 8 | `audio.format` | `1` = AAC, `0` = MP3 |
| 9 | `audio.number_of_streams` | normally `1` |
| 10 | `audio.maximum_number_of_frames` (`maxaf`) | `>= 1` **and** `>= n_af` of every packet |
| 11 | `audio.scale` | samples per audio frame: **1024 = AAC**, 1152 = MP3 |
| 12 | `audio.rate` | `44100` (the reference player only accepts 44100) |
| 13 | `audio.stereo` | `1` = stereo |

### 1.2 Frame index (offset 56)

`n_vframes` x `uint32_t`, one entry per packet:

```
entry = (packet_total_size << 1) | keyframe_flag
```

`packet_total_size` counts **every byte of the packet** (the 1 + 4*(n_af+3)
byte length table plus all audio and video data). The low bit is the
keyframe flag — see §2.3 for the exact rule the reference muxer uses.

### 1.3 Packet

```
uint8   n_af                  audio frames in this packet
uint32  first_delay           µs, sceKernelDelayThread before the video frame
uint32  last_delay            µs, sceKernelDelayThread after the audio
uint32  video_length          bytes of video data
uint32  audio_length[n_af * n_astreams]
byte[]  video data            exactly one H.264 access unit (Annex B)
byte[]  audio data            raw AAC (or MP3) frames, no ADTS/headers
```

The reference player reads `n_af` first, allocates
`audio_frame_size * aud_max_frames` bytes, then copies `n_af` frames into
that buffer. If `maxaf` (header field 10) is smaller than the largest `n_af`
in the file, this heap-overflows and crashes on real hardware — emulators
usually shrug it off.

## 2. Timing, interleaving, keyframes

### 2.1 The delay invariant

`first_delay` and `last_delay` are passed to `sceKernelDelayThread()` in
**microseconds** and must satisfy the `time_math.pm` invariant of the
official muxer:

```
first_delay + audio_span + last_delay == video_frame_duration
```

where `audio_span` runs from the start of the *first* audio frame of the
packet to the start of the *last* one — i.e. `n_af - 1` audio periods.
`last_delay` is measured from the last audio frame's start and already
covers that frame's own playback time. Violating this makes A/V drift, or
makes the player look like it hangs while it is really just waiting out an
enormous delay.

### 2.2 Audio interleaving

`pmp-cli` reimplements the reference algorithm (`interleaving.pm`): audio
frames are dealt out to video packets strictly in order — a packet takes as
many audio frames as fit before the next video frame's presentation time.
At 29.97 fps video / 43.07 fps audio (AAC 44100/1024) most packets get 1
audio frame and every other packet gets 2 (`n_af` distribution of the
reference files: `{1: 1462, 2: 1136}` for a typical episode clip).

### 2.3 Keyframe flag

The index keyframe bit is set only when the access unit is
`AUD(primary_pic_type == 0) → SPS`. Mid-stream IDRs (scene cuts) written
together with SPS/PPS therefore also count as keyframes.

### 2.4 Framerate ceiling

- Hard muxer rule: every video frame must get at least one audio frame, so
  fps may not exceed the audio frame rate (43.07 fps for AAC 44100/1024,
  38.28 for MP3 44100/1152). The official muxer refuses higher with
  `framerate is too high`.
- Practical ceiling: **29.97 fps** — every known-good XviD4PSP reference
  encode is 29.97 and plays fine; nothing above it was ever ME-tested.

## 3. H.264 bitstream requirements (real Media Engine)

This is the part emulators cannot tell you about. PPSSPP does not run the
hardware path at all: it detects PMP and decodes the access units with
ffmpeg (`decodePmpVideo()` in `Core/HLE/sceMpeg.cpp`), hardcoded to
480x272 and tolerant of anything. On a real PSP the reference player DMAs
the access unit into the fixed ME buffer at `0x4a000` (`avc.c: MEAVCBUF`)
and feeds it to `sceMpegAvcDecode`. Everything below was proven by hang
captures (psplink + a thread-state watchdog) and by A/B against the
reference files.

### 3.1 Profile: Main + CABAC — mandatory

Every ME-tested reference PMP is `profile_idc = 77` (Main) with
`entropy_coding_mode_flag = 1` (CABAC). A Baseline/CAVLC encode decodes
cleanly in ffmpeg and PPSSPP but **hangs the real ME ~5 s in, at the first
mid-GOP IDR**: decode dies inside `sceMpegAvcDecode`/ME, all player
threads go to SLEEP forever, and the whole system freezes 13 s later.
Never ship baseline to hardware.

Note: x264's `ultrafast`/`superfast` presets disable CABAC.
`pmp-cli` forces `--cabac` back on after any preset.

### 3.2 SPS fields that must match the reference (`--psp-compat`)

| Field | Required | Why |
|-------|----------|-----|
| `log2_max_frame_num_minus4` | `5` (log2 = 9) | XviD4PSP reference value; x264's default 4+4=8 works but does not match |
| `pic_order_cnt_type` | `0` | x264 would emit type 2 on bframes=0 paths |
| `log2_max_poc_lsb_minus4` | `6` (log2 = 10) | x264's minimal 4-bit window wraps `pic_order_cnt_lsb` every 8 frames; the ME hangs on the MSB wraparound |
| version/user-data SEI | absent | x264 writes a 600+ byte "info" SEI into frame 0; the reference bitstream has none |

`bin/x264` is a native build patched with `--psp-compat` which enforces
all of the above (see `bin/README.md`).

### 3.3 HRD VUI + per-frame pic_timing SEI — mandatory

All reference PMPs carry a VUI with:

- `nal_hrd_parameters_present_flag = 1`, `vcl_hrd = 0`
- HRD values decode to exactly `vbv-maxrate 10000 kbps / vbv-bufsize
  10000 kbit` (bit_rate ≈ 10 000 128 bps, cpb ≈ 10 000 128 bits in all 11
  files) → x264: `--nal-hrd vbr --vbv-maxrate 10000 --vbv-bufsize 10000`
- `pic_struct_present_flag = 0` — do **not** pass `--pic-struct`
- `bitstream_restriction_flag = 0`

With `nal_hrd` on, x264 automatically emits the SEI layout the ME expects:

```
IDR access unit: [AUD(9), SPS(7), PPS(8), SEI buffering_period(6),
                  SEI pic_timing(6), IDR slice(5)]
P access unit:   [AUD(9), SEI pic_timing(6), non-IDR slice(1)]
```

A stream without HRD (`--nal-hrd` off) loses the per-frame pic_timing SEIs
and hangs the real ME at the first mid-GOP IDR even when everything else
is correct.

### 3.4 NAL-level rules

- All start codes are 4 bytes (`00 00 00 01`); `pmp-cli` normalises them.
- Access units are ordered `AUD → SPS → PPS → SEI → slice`; `pmp-cli`
  reorders defensively before muxing.
- IDRs carry SPS/PPS in band (every keyframe repeats them).
- One slice per frame (no FMO/ASO); `frame_mbs_only_flag = 1`.
- Access unit size must stay under the ME buffer: `pmp-cli` enforces
  `MAX_AU_SIZE = 0x40000` (256 KiB), matching the `0x4a000` DMA region.

### 3.5 Reference encoder settings (XviD4PSP 5 "x264 HQ Extreme" for PMP)

Recovered from the shipped preset + the PMP-encoder window defaults (all
numbers verified against the reference bitstreams):

```
--profile main --level 3.0 --ref 2 --deblock -2:-1 --direct none
--no-b-adapt --no-8x8dct --no-weightb --weightp 0 --bframes 0
--trellis 2 --no-fast-pskip --no-dct-decimate --aud
--partitions p8x8,b8x8,i4x4,p4x4 --me umh --subme 9
--crf 21                       (reference PPS pic_init_qp = 21)
--nal-hrd vbr --vbv-maxrate 10000 --vbv-bufsize 10000   (PMP window)
keyint 250 (reference: 31 keyframes / 2598 frames = scenecut + keyint)
```

`pmp-cli`'s native path reproduces this exactly for `--preset slow` and
keeps all of §3.1–3.3 for every faster preset.

### 3.6 Known cosmetic differences (safe)

Modern x264 vs the 2009 XviD4PSP build writes `num_ref_idx_l0_default_active_minus1 = ref-1`
in the PPS (reference always wrote 0), and slightly different HRD scale
selection / SEI field widths. Both are self-consistent inside the
bitstream; the ME reads per-slice overrides and declared field widths.
No effect on hardware.

## 4. Debugging on real hardware

- **`pmpwatch.prx`** (this repo, `pmpwatch/`) — kernel plugin that dumps
  all thread states to `ms0:/pmp_watch.log` (or `ef0:` on PSP Go) every
  250 ms. Healthy decode: `decode`/`output` threads cycle wait states.
  Freeze signature: all player threads SLEEP (`wt=0x1`) forever inside
  firmware calls.
- **psplink** — attach over USB (`start-host.sh` + `attach.sh`) for an
  interactive shell and kprintf.
- The Lua test harness used for hardware verification lives in
  `pmp_test/` (`script.lua` plays a PMP through the openTri player inside
  LuaPlayerYT).

## 5. References

- [PMP-Mod GitHub](https://github.com/PSP-Archive/PMP-Mod) — original player source
- [PMP Container Spec](https://web.archive.org/web/20080410063712fw_/http://jonny.leffe.dnsalias.com/pmp_mod/container.htm)
- [pmp_muxer_avc 1.01 source](https://web.archive.org/web/20060711092320/http://jonny.leffe.dnsalias.com:80/pmp_mod_avc/rar/pmp_muxer_avc_1_01_source.tar.gz)
  — official H.264 muxer; `time_math.pm` / `interleaving.pm` define the
  delay invariant and interleaving used by `pmp-cli`
- [pmp_mod_avc 1.02 source](https://web.archive.org/web/20060711092609/http://jonny.leffe.dnsalias.com:80/pmp_mod_avc/rar/pmp_mod_avc_1_02_source.tar.gz)
  — reference PSP player (`pmp_file.c` header checks, `avc.c` DMA into
  `MEAVCBUF = 0x4a000`, `pmp_read.c` packet parsing)
- [PPSSPP sceMpeg.cpp](https://github.com/hrydgard/ppsspp/blob/master/Core/HLE/sceMpeg.cpp)
  — emulator PMP path (`decodePmpVideo()`), explains why the emulator is lenient
