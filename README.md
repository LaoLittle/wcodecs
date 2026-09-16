# wcodecs

`wcodecs` is a cross-platform audio and video decoding API for Rust. It runs
natively on VideoToolbox, Media Foundation, MediaCodec, VA-API, and software
decoders, and uses browser WebCodecs on WebAssembly.

The API is based on the [WebCodecs decoder model][webcodecs], adapted to Rust
with owned output and polling-based event delivery. Applications provide encoded
chunks and receive decoded video frames or interleaved PCM audio. No game
engine, rendering API, or particular async runtime is required for native
decoding.

Container demuxing, network transport, playback clocks, seeking, audio output,
and rendering belong to the application. Encoding and image decoding are not
implemented yet.

[Getting started](#getting-started) · [Platforms](#supported-platforms) ·
[Codecs](#codec-support) · [Features](#cargo-features) ·
[WebCodecs compatibility](#webcodecs-compatibility)

## Getting started

`VideoDecoder` and `AudioDecoder` share the same control model: configure a
decoder, submit timestamped chunks, and poll for output or errors. A flush
barrier marks completion of previously submitted work without blocking.

This example submits one raw Opus packet. The packet and timestamp come from
your demuxer or transport, not a complete Ogg or WebM file.

```rust
use std::sync::Arc;
use wcodecs::{
    AudioData, AudioDecoder, AudioDecoderConfig, ChunkType, CodecError,
    DecoderEvent, EncodedAudioChunk, EncodedChunk, FlushId,
};

fn start_packet(
    packet: Arc<[u8]>,
    timestamp_us: i64,
) -> Result<(AudioDecoder, FlushId), CodecError> {
    let mut decoder = AudioDecoder::new()?;
    decoder.configure(AudioDecoderConfig::new("opus", 48_000, 2))?;
    decoder.decode(EncodedAudioChunk(EncodedChunk {
        kind: ChunkType::Key,
        timestamp: timestamp_us,
        duration: None,
        data: packet,
    }))?;
    let barrier = decoder.flush()?;
    Ok((decoder, barrier))
}

// Call from your update/event loop until this returns true.
fn poll_packet(
    decoder: &mut AudioDecoder,
    barrier: FlushId,
    mut on_output: impl FnMut(AudioData),
) -> Result<bool, CodecError> {
    while let Some(event) = decoder.poll() {
        match event {
            DecoderEvent::Output(data) => on_output(data),
            DecoderEvent::Flushed(id) if id == barrier => return Ok(true),
            DecoderEvent::Flushed(_) => {}
            DecoderEvent::Error(error) => return Err(error),
        }
    }
    Ok(false)
}
```

Keep the decoder and barrier across updates. `poll()` returning `None` means
that no event is currently available, not that decoding has finished. Yield to
the browser between updates on Wasm; do not spin or block its event loop.
For continuous streams, interleave submission and polling and flush only at a
drain boundary. Opus chunks should be marked `Key`.

For video, use `VideoDecoderConfig::new("av01.0.04M.08", width, height)` and
`EncodedVideoChunk`. Call `VideoDecoder::is_config_supported(&config).await` or
`AudioDecoder::is_config_supported(&config).await` to query support before
configuration. A positive result is not a hardware or playback guarantee.

## Supported platforms

| Backend | Windows | Linux | macOS | Android API 28+ | Web / Wasm |
| --- | :---: | :---: | :---: | :---: | :---: |
| Software: rav1d / opus-rs | ✅ | ✅ | ✅ | ✅ | — |
| VideoToolbox | — | — | ✅ | — | — |
| Media Foundation | ✅ | — | — | — | — |
| MediaCodec | — | — | — | 🧪 | — |
| VA-API | — | 🧪 | — | — | — |
| Browser WebCodecs | — | — | — | — | ✅ |

✅ = Implemented backend path, not a tested-device certification.  
🧪 = Experimental; full target builds and real-device playback have not yet
been verified.

Native platform backends currently decode **AV1 video only**. Native audio uses
the software Opus decoder, including with `hardware` enabled. Other Apple
targets are not yet documented as validated platforms.

On Wasm, decoding is delegated to the browser. There is no bundled rav1d/Opus
fallback when browser WebCodecs is unavailable.

## Codec support

Codec identifiers are open strings following the [WebCodecs Codec Registry][registry],
not a closed Rust enum. The API can carry new identifiers without changing its
types; that does not add a native decoder implementation.

| Codec | Identifier / family | Native decoding | Browser backend |
| --- | --- | --- | --- |
| AV1 | `av01.*` | Profile 0, 8-bit 4:2:0, SDR | Browser-dependent |
| AVC / H.264 | `avc1.*`, `avc3.*` | Not implemented | Browser-dependent |
| HEVC / H.265 | `hvc1.*`, `hev1.*` | Not implemented | Browser-dependent |
| VP8 / VP9 | `vp8`, `vp09.*` | Not implemented | Browser-dependent |
| Opus | `opus` | Mono / stereo; restrictions below | Browser-dependent |
| AAC | `mp4a.*` | Not implemented | Browser-dependent |
| MP3 / FLAC / Vorbis | `mp3`, `flac`, `vorbis` | Not implemented | Browser-dependent |
| G.711 / linear PCM | `ulaw`, `alaw`, `pcm-*` | Not implemented | Browser-dependent |

`*` denotes a codec-string family, not a literal configuration value.
**Browser-dependent means forwarded, not verified codec support.** Both the
browser configuration and this crate's output conversion must work. Native
support is not inferred from codecs supported by the operating-system API.

Native AV1 uses rav1d as its software implementation and fallback. The current
bridge excludes monochrome, 4:2:2, 4:4:4, higher-bit-depth, and HDR output.
The codec-string parser accepts the four-field short form, such as
`av01.0.04M.08`, or the complete ten-field form.

Native Opus accepts 1 or 2 channels at **8, 12, 16, 24, or 48 kHz** and requires
`description` to be absent or empty. Identification-header parsing, multistream
channel mappings, and automatic container pre-skip/discard-padding handling
are not implemented. Output is interleaved `f32` PCM.

## Cargo features

The default **`hardware`** feature enables all four native backend features,
**including Linux VA-API**. Dependencies are target-specific.

| Feature | Platform | Enables |
| --- | --- | --- |
| `hardware` | Native | All four native platform features below. |
| `video-toolbox` | Apple | VideoToolbox AV1 decoding. |
| `media-foundation` | Windows | Media Foundation AV1 decoding. |
| `media-codec` | Android | NDK MediaCodec AV1 decoding. |
| `vaapi` | Linux | VA-API through cros-codecs and cros-libva. |

For a local checkout, native software-only decoding can be selected with:

```toml
[dependencies]
wcodecs = { path = "../wcodecs", default-features = false }
```

Add `features = ["vaapi"]` to select only VA-API, or omit
`default-features = false` for default platform selection. On Wasm, feature
selection does not replace browser WebCodecs with a native software decoder.

`HardwareAcceleration::PreferSoftware` bypasses native platform video backends.
`NoPreference` and `PreferHardware` try the applicable native backend, with
rav1d fallback on initialization failure. The browser receives the equivalent
WebCodecs hint. Android and Windows can select an OS software decoder, so
`hardware` is not a hardware-acceleration guarantee.

After a backend accepts stream input, errors are reported rather than silently
switching to a software decoder without the required reference frames.

## Execution and output

`state()` exposes `Unconfigured`, `Configured`, and `Closed`. `configure()` and
`decode()` submit work; `flush()` returns a `FlushId` observed through
`DecoderEvent::Flushed`. Both decoders require a key chunk after configuration
or flush. `reset()` discards pending work and barriers without reusing flush
IDs. Backend errors received through `poll()` close the decoder.

Native work runs on a dedicated worker with a **32-event output channel** and
an **unbounded input queue**. Browser event/frame-copy queues are also unbounded.
Use `decode_queue_size()` and `pending_output()` to throttle submissions and
keep polling; these counters are not a global memory limit.

Native `close()`, `reset()`, and `Drop` signal cancellation without joining the
worker on the caller's thread. Resources are released as the worker exits;
a hung vendor codec or driver call cannot be interrupted by Rust.

Video output is owned, CPU-readable `I420Strided`, `I420Planar`, `Nv12Strided`, or
`Rgba` data, depending on the backend. Audio output is reference-counted,
interleaved `f32` PCM with sample rate and channel count. Timestamps use signed
microseconds, including negative preroll, subject to backend range limits.

Frames include a YUV-to-RGB transform and simplified transfer-function metadata.
Full color management, HDR tone mapping, higher-bit-depth output, and GPU-native
frame sharing are not implemented. Codec/browser output is copied into
Rust-owned storage: this is **not a zero-copy API**.

## WebCodecs compatibility

The API follows the decoder model of the
[27 August 2026 WebCodecs Working Draft][webcodecs], not the entire W3C surface.

| Area | Status |
| --- | --- |
| `VideoDecoder` / `AudioDecoder` | Implemented with Rust states, polling, and drain barriers. |
| Encoded audio/video chunks | Owned bytes, key/delta type, timestamp, and optional duration. |
| Decoder configurations | Core fields are exposed; video configuration is a subset. |
| `VideoFrame` / `AudioData` | Simplified Rust-owned output, not full browser-object wrappers. |
| `VideoEncoder` / `AudioEncoder` | Not implemented. |
| `ImageDecoder` / image track interfaces | Not implemented. |
| Browser/GPU frame import and export | Not implemented. |

Polling replaces callbacks and `dequeue` events; flush tokens replace promises.
Rust ownership replaces explicit release of returned media objects. Native
`DecodeSettings` adds software decoder tuning rather than a browser API mapping.

Important gaps include video display-aspect/color overrides, rotation/flip,
complete frame geometry and metadata, sample-format negotiation, and a complete
DedicatedWorker/OffscreenCanvas path. Native support queries do not probe
hardware, and browser support queries do not validate the Rust output bridge.

See [WebCodecs compatibility details](WEBCODECS.md) for field mappings,
codec-description differences, output limitations, and query semantics.

## Backend notes

| Backend | Current constraints |
| --- | --- |
| VideoToolbox | Requires AV1 hardware support and av1C in `description`; negotiates CPU-readable NV12/I420. Other configurations fall back during initialization. |
| Media Foundation | Synchronous/asynchronous AV1 MFTs with CPU-readable NV12. No D3D-only transform/device-manager path. `description` is not consumed. |
| MediaCodec | NDK byte buffers; no JNI context or rendering surface. Linear I420/NV12 and even-origin crops only; flexible/opaque/tiled output is rejected. Copies to owned I420. |
| VA-API | Probes DRM render nodes, AV1 Profile 0/VLD, and NV12 GBM allocation/import. Extracts OBUs from av1C; copies NV12 surfaces to owned I420. |
| Browser WebCodecs | Copies I420 when possible, otherwise RGBA with a DOM canvas fallback. PQ/HLG is rejected. Bindings remain private; no `web_sys_unstable_apis` flag is required. |

AV1 configuration is not normalized across native paths. VideoToolbox needs
av1C, while rav1d and Media Foundation expect required sequence information in
the encoded chunks. Portable fallback must not depend on out-of-band
configuration alone. The [WebCodecs AV1 registration][av1-registration] does not
use `description`; the current web bridge nevertheless forwards it when supplied.

## Building and validation

Linux VA-API requires libva, GBM, and DRM development libraries, pkg-config,
and Clang/libclang. These also apply to default-feature Linux builds.
Native x86 rav1d assembly builds require NASM. Android needs a configured NDK
with the API 28+ bindings used by this crate.

The extracted manifest still has `futures-lite.workspace = true` as a dev
dependency. Replace it with an explicit version for a standalone checkout
before running these commands. No minimum supported Rust version is declared.

```sh
# Native software path.
cargo test -p wcodecs --no-default-features --lib

# Default platform backends, with native prerequisites installed.
cargo test -p wcodecs --lib

# Linux VA-API without the hardware umbrella feature.
cargo test -p wcodecs --no-default-features --features vaapi --lib
```

Tests cover lifecycle, flush ordering, signed timestamps, delayed AV1 output,
and checked frame/progress helpers. Hardware playback requires separate tests
on supported devices and drivers. Android and Linux platform builds and
real-device validation remain outstanding.

## License

Licensed under either [Apache License 2.0](LICENSE-APACHE) or
[MIT License](LICENSE-MIT), at your option.

Native decoding uses the rav1d/opus-rs forks and cros-codecs/cros-libva dependencies
listed in [`Cargo.toml`](Cargo.toml). No FFmpeg/libavcodec backend is included.
Dependencies and separately packaged system drivers retain their own licenses.

[webcodecs]: https://www.w3.org/TR/webcodecs/
[registry]: https://www.w3.org/TR/webcodecs-codec-registry/
[av1-registration]: https://www.w3.org/TR/webcodecs-av1-codec-registration/
