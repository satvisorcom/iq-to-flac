# iq-to-flac

Store raw IQ baseband recordings in FLAC - losslessly, with metadata.

FLAC's linear prediction model exploits sample-to-sample correlation in IQ data better than generic compressors like ZSTD or gzip. [RFC 9639](https://www.rfc-editor.org/rfc/rfc9639.html) (Section 8.2) formally supports non-audio data via `sample_rate=0`:

> A sample rate of 0 MAY be used when non-audio is represented. This is useful if data is encoded that is not along a time axis or when the sample rate of the data lies outside the range that FLAC can represent in the streaminfo metadata block.

## Requirements

- `flac` and `metaflac` (libFLAC >= 1.3)
- `zstd` (only for compressed ZIQ input)
- bash and standard coreutils (`tr`, `od`, `dd`, `tail`)

```
# Debian/Ubuntu
sudo apt install flac zstd

# macOS
brew install flac zstd
```

## Usage

### Encode

```bash
iq-to-flac --format cs16 --sample-rate 5M --frequency 145M recording.cs16 recording.flac
```

| Option | Description | Default |
|---|---|---|
| `--format FMT` | Input format (see below) | cs16 |
| `--bps N` | Bits per sample: 8, 16, or 32 (auto-set by `--format`) | 16 |
| `--sample-rate HZ` | IQ sample rate (supports k/M/G suffixes) | - |
| `--frequency HZ` | Center frequency (supports k/M/G suffixes) | - |
| `--compression N` | FLAC compression level (0-8) | 5 |

### Decode

```bash
flac-to-iq recording.flac recording.raw
flac-to-iq --format cu8 recording.flac recording.cu8
flac-to-iq --format ziq --zstd-level 3 recording.flac recording.ziq
```

| Option | Description | Default |
|---|---|---|
| `--format FMT` | Output format (see below) | signed, matching FLAC bps |
| `--zstd-level N` | ZSTD compression level for ZIQ output (0 = none, 1-19) | 1 |
| `-i, --info` | Show metadata and exit | - |

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
IQ_FREQUENCY=145000000
IQ_SAMPLE_RATE=5000000
```

## Supported formats

| Format | Description | Notes |
|---|---|---|
| `cs8` | Signed 8-bit interleaved IQ | Direct, no conversion needed |
| `cs16` | Signed 16-bit LE interleaved IQ | Direct, no conversion needed |
| `cs32` | Signed 32-bit LE interleaved IQ | Direct, no conversion needed |
| `cu8` | Unsigned 8-bit interleaved IQ | Native output format of RTL-SDR. FLAC only supports signed samples, so each byte is XOR'd with 0x80 to convert unsigned (0-255, center 128) to signed (-128 to 127, center 0). The operation is its own inverse, so decoding applies the same XOR to recover the original bytes |
| `w16` | WAV 16-bit | Encode: header is stripped, PCM data is encoded as cs16. Decode: WAV header is reconstructed with sample rate from `IQ_SAMPLE_RATE` metadata |
| `ziq` | SatDump ZIQ (8/16-bit) | Encode: header is parsed to auto-set bps, sample rate, and frequency. Decode: writes ZIQ with ZSTD compression (use `--no-compress` to disable). Requires `zstd`. 32-bit ZIQ is IEEE float and not supported |

## Verification

IQ samples are always bit-perfect through encode/decode. For raw formats (cs8, cs16, cs32, cu8) the files are byte-identical:

```
$ sha256sum input.cs16 output.cs16
0471c111...  input.cs16
0471c111...  output.cs16
```

For container formats (w16, ziq), the sample data is preserved but headers may differ. WAV files from SDR software often include extra metadata chunks (e.g. SDRuno adds `LIST`/`INFO`) that are not preserved  - our encoder strips the header down to raw PCM, and our decoder writes a minimal 44-byte WAV header. ZIQ files may differ in ZSTD compression level. To verify sample-level integrity, compare the decoded raw output rather than the container files.

## Compression

FLAC's linear predictor exploits sample-to-sample correlation in IQ data, which generic compressors like ZSTD can't. Benchmarks against ZIQ (SatDump's ZSTD-1 compressed format) on real recordings:

| Signal | Raw | FLAC | ZIQ | FLAC advantage |
|---|---|---|---|---|
| NOAA-15 SARSAT (L-band, 250 kSPS) | 876 MB | 58% | 68% | 10% smaller |
| FengYun-4A LRIT (L-band, 1 MSPS) | 471 MB | 93% | 97% | 4% smaller |
| AIS (VHF, 62.5 kSPS) | 19 MB | 60% | 73% | 13% smaller |
| FT8 (HF, 62.5 kSPS) | 40 MB | 58% | 74% | 16% smaller |
| DMR (VHF, narrowband) | 346 MB | 52% | 78% | 26% smaller |
| WEFAX (HF, 62.5 kSPS) | 153 MB | 55% | 70% | 15% smaller |

FLAC beats ZIQ by 4-26% depending on the signal. Narrowband signals with gaps (DMR, FT8) compress best. Wideband digital downlinks with high SNR (FengYun-4A) compress poorly in both  - a clean BPSK signal carrying compressed data is essentially random.

This format is primarily useful for **archival**. No SDR software reads FLAC IQ natively, so recordings must be decoded back to a supported format (cs16, cu8, w16, ziq) before processing. The tradeoff is smaller files on disk at the cost of a conversion step.

## Limitations

- **No float32 IQ** - FLAC only supports integer samples. `cf32` and 32-bit ZIQ (IEEE float) must be quantized to integer first.
- **Not real-time** - FLAC encodes at ~115 MB/s on the test above. Fine for archival, but ZIQ's ZSTD-1 is ~12x faster, which matters for real-time recording to disk.

## File format

The output is a standard FLAC file. I and Q map to channels 0 and 1, encoded with `--sample-rate=0` per RFC 9639. Any FLAC decoder can decompress it, but the IQ metadata stored as Vorbis comments is needed to interpret the result correctly. A generic audio player will see a stereo file with a 0 Hz sample rate and produce silence or an error. Use `flac-to-iq` (or `metaflac`) to read the metadata and recover the original IQ format.

### Vorbis comment tags

| Tag | Example | Notes |
|---|---|---|
| `IQ_FREQUENCY` | `145000000` | RF tuner frequency in Hz. Cannot be derived from IQ data since baseband is always centered at DC |
| `IQ_SAMPLE_RATE` | `5000000` | Complex sample rate in Hz. FLAC's STREAMINFO sample rate is set to 0 (non-audio mode), so the real rate must be stored here |

All tags are readable with standard tools:

```bash
metaflac --export-tags-to=- recording.flac
```

## See also

- [RFC 9639 - Free Lossless Audio Codec](https://www.rfc-editor.org/rfc/rfc9639.html) - Section 8.2 defines the non-audio mode this project relies on
- [ZIQ format (SatDump)](https://docs.satdump.org/baseband_formats.html) - ZSTD-compressed IQ container used by SatDump

## License

[Unlicense](UNLICENSE) - do whatever you want with it.
