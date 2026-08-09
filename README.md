# pmp-cli

**WARNING: THIS TOOL IS COMPLETELY VIBECODED**

PSP PMP video converter and compressor.

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

# Compress existing PMP
pmp-cli compress input.pmp output.pmp --crf 30

# Batch convert directory
pmp-cli batch ./videos/ --batch-mode convert --output-dir ./pmp-files/
```

## Common Commands

```bash
# Maximum quality
pmp-cli convert movie.mp4 movie.pmp --crf 18 --preset veryslow

# Fast conversion
pmp-cli convert test.mp4 test.pmp --preset ultrafast

# Maximum compatibility
pmp-cli convert video.mp4 output.pmp --profile baseline --fps 24

# Batch compress, replace originals
pmp-cli batch ./library/ --batch-mode compress --recursive --replace --crf 30
```

## Key Options

### Video
- `--crf 0-51` — Quality (lower = better, default 28)
- `--fps 1-25` — Frame rate (default 24, max 25)
- `--preset` — Speed: ultrafast, fast, slow, veryslow
- `--profile` — baseline, main, high
- `--width/--height` — Resolution (default 480x272, must be divisible by 8)
- `--deinterlace` — Apply deinterlacing filter

### Audio
- `--audio-bitrate` — CBR bitrate (default 96)
- `--audio-quality 0.1-5.0` — VBR quality (lower = better)
- `--audio-rate` — Sample rate: 32000, 44100, 48000
- `--no-audio` — Strip audio

### Batch
- `-r, --recursive` — Process subdirectories
- `--replace` — Replace originals
- `--dry-run` — Show what would be done

Full options: run `pmp-cli convert --help`

## PMP Format

PMP (Portable Media Player) is a PSP multimedia container with H.264 video + AAC audio.

**Container Structure:**
```
Header (56 bytes) → 14 × uint32_t (little-endian)
Frame Index       → n_vframes × uint32_t (size << 1 | keyframe_flag)
Packet Data       → Video + Audio packets
```

**Constraints:**
- Video: H.264 Annex B, baseline/main profile, level 3.0
- Audio: AAC-LC 44.1kHz stereo, raw frames (no ADTS headers)
- Resolution: up to 720×512 (divisible by 8), 480×272 recommended
- FPS: 1-25 (>25 causes performance issues on PSP)
- **Critical:** `aud_scale` must be 1024 (PSP crashes if set to 1)
- **Critical:** NAL structure must be `AUD → SPS → PPS → IDR` (PSP hardware decoder requirement)

## Building Binary

```bash
pip install pyinstaller
pyinstaller --onefile --name pmp-cli-bin pmp-cli
# Output: dist/pmp-cli-bin
```

## Troubleshooting

**Player crashes on PSP:**
- Use `--profile baseline --level 3.0`

**No audio:**
- Check `--audio-rate 44100` and `--audio-channels stereo`

**Choppy playback:**
- Lower FPS: `--fps 20`
- Lower resolution: `--width 368 --height 208`
- Lower bitrate: `--crf 30`

## References

- [PMP-Mod GitHub](https://github.com/PSP-Archive/PMP-Mod) — Original player source
- [PMP Container Spec](https://web.archive.org/web/20080410063712fw_/http://jonny.leffe.dnsalias.com/pmp_mod/container.htm) — Format docs

## License

MIT
