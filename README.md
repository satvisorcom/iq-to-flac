# flac-iq

Store raw IQ baseband recordings in FLAC - losslessly, with metadata.

FLAC's linear prediction model exploits sample-to-sample correlation in IQ data better than generic compressors like ZSTD or gzip. [RFC 9639](https://www.rfc-editor.org/rfc/rfc9639.html) (Section 8.2) formally supports non-audio data via `sample_rate=0`:

> A sample rate of 0 MAY be used when non-audio is represented. This is useful if data is encoded that is not along a time axis or when the sample rate of the data lies outside the range that FLAC can represent in the streaminfo metadata block.

IQ sample rate, center frequency, and bit depth are stored as Vorbis comments inside the FLAC file.

## Requirements

- `flac` and `metaflac` (libFLAC >= 1.3)
- bash

```
# Debian/Ubuntu
sudo apt install flac

# Fedora
sudo dnf install flac

# macOS
brew install flac
```

## Usage

### Encode

```bash
iq-to-flac --bps 16 --sample-rate 5000000 --frequency 145000000 recording.raw recording.flac
```

| Option | Description | Default |
|---|---|---|
| `--bps N` | Bits per sample (8, 16, or 32) | 16 |
| `--sample-rate HZ` | IQ sample rate (stored as metadata) | - |
| `--frequency HZ` | Center frequency (stored as metadata) | - |
| `--compression N` | FLAC compression level (0–8) | 5 |

Input format: raw interleaved IQ (I,Q,I,Q,...), signed, little-endian.

### SatDump compatibility

SatDump baseband recordings in `cs8` and `cs16` format work directly:

```bash
iq-to-flac --bps 8 --sample-rate 6000000 recording.cs8 recording.flac
iq-to-flac --bps 16 --sample-rate 6000000 recording.cs16 recording.flac
```

| SatDump format | Compatible | Notes |
|---|---|---|
| `cs8` | Yes | Use `--bps 8` |
| `cs16` | Yes | Use `--bps 16` |
| `cu8` | No | Unsigned - needs offset correction (-128) first |
| `cf32` | No | Float - FLAC only supports integer samples |
| `w16` | No | WAV container - strip header or use `flac` directly |
| `ziq` | No | Decompress with SatDump first |

### Decode

```bash
flac-to-iq recording.flac recording.raw
```

### Inspect metadata

```bash
flac-to-iq --info recording.flac
```

```
=== STREAMINFO ===
Sample rate (FLAC): 0 Hz
Channels: 2
Bits per sample: 16
Total samples: 5000000
=== IQ METADATA ===
IQ_SAMPLE_RATE=5000000
FREQUENCY=145000000
IQ_BITS_PER_SAMPLE=16
```

## Verification

Roundtrip is bit-perfect:

```
$ sha256sum input.raw output.raw
0471c111...  input.raw
0471c111...  output.raw
```

## Limitations

- **No float32 IQ** - FLAC only supports integer samples. 32-bit means int32, not IEEE float. Float inputs must be quantized first.
- **Compression depends on signal** - Noisy wideband captures compress poorly (~10%). Narrowband or cleaner signals do much better.
- **FLAC sample rate cap** - The STREAMINFO field maxes out at 655,350 Hz, which is why we set it to 0 and store the real rate as a Vorbis comment.

## How it works

I and Q map to FLAC channels 0 and 1 (like stereo audio). The FLAC encoder is invoked with `--sample-rate=0` to signal non-audio content per RFC 9639. Metadata (actual sample rate, center frequency, bit depth) is written as Vorbis comments via `metaflac`.

## See also

- [RFC 9639 - Free Lossless Audio Codec](https://www.rfc-editor.org/rfc/rfc9639.html)
- [ZIQ format (SatDump)](https://docs.satdump.org/baseband_formats.html) - ZSTD-compressed IQ container
- [SigMF](https://github.com/sigmf/SigMF) - signal metadata format
- [iq-compressor](https://github.com/zouppen/iq-compressor) - similar concept in C

## License

[Unlicense](UNLICENSE) - do whatever you want with it.
