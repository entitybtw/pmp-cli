# Changelog

## 0.2.0 (2026-10-17)

Hardware-verified PSP bitstream + container-level compress pipeline.

### Fixed (real hardware)

- **PSP hang**: baseline/CAVLC encodes freeze the real Media Engine ~5 s in
  (first mid-GOP IDR; ffmpeg/PPSSPP still play them fine). Default is now
  Main/CABAC with HRD VUI + per-frame pic_timing SEIs, matching all
  ME-tested XviD4PSP reference PMPs. Verified on PSP Go (N1000, ARK-5):
  full playback past the former hang point, A/B vs reference identical.
- `--psp-compat` x264 patch: widen `log2_max_poc_lsb` to 10 (ME hangs on
  the 4-bit MSB wraparound), skip x264's 600-byte version SEI.
- Audio/video sync: delays recomputed from the reference
  `time_math.pm` invariant (`first_delay + audio_span + last_delay ==
  frame_duration`) instead of global positions.

### Added

- `decode` command: PMP → MP4 for checking on the PC.
- Container-level `compress`: no MP4 round trip; `--stream video|audio|both`,
  audio frames stay byte-identical unless re-encoded; atomic `--replace`
  in `batch` (temp file + swap).
- Native x264 with `--psp-compat` (`bin/x264`), wine no longer needed.
- `--profile`/`--preset` wired into the native x264 path (both were dead
  arguments); `--cabac` forced after any preset; `batch` honours
  `--preset` (`faster` ≈ 700 fps, `slow` = XviD4PSP HQ Extreme knobs).
- HRD defaults `--nal-hrd vbr --vbv 10000/10000` (values decoded from all
  11 reference PMPs); ffmpeg fallback matches.
- Reference-player warning suite: header fields, `aud_max_frames`, delays,
  AUD order, oversized AUs (> 256 KiB ME buffer).

### Changed

- Dropped the bogus `fps > 25` warning (references are 29.97).
- Docs: `docs/FORMAT.md` (container + ME bitstream rules + hardware
  debugging links), `docs/USAGE.md` (guide, CRF/preset ladders); README
  overhaul; "use baseline" troubleshooting advice removed (it hangs ME).

### Housekeeping

- Removed the bundled `output.mp4` demo from the repository history.

## 0.1 (2026-08-08)

Initial release: convert/compress/batch PMP tools, basic PSP checks.
