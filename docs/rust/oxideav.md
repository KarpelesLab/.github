# OxideAV: pure-Rust media framework

OxideAV (https://github.com/OxideAV, about 149 repos) is a media transcoding and streaming framework written entirely in Rust, in the same space as FFmpeg. Every codec, container and filter is implemented clean-room from the specs. It wraps no C codec libraries and uses no `*-sys` crates. The only FFI is in the optional hardware-acceleration bridges, which load OS libraries at runtime through `libloading`. The framework is one ecosystem: `oxideav-core` defines the types, traits and registries, each format lives in its own `oxideav-<name>` crate that plugs into those registries, `oxideav-meta` wires everything together behind cargo features, and `oxideav-workspace` builds the `oxideav` CLI and the `oxideplay` player on top. License is MIT across the org, with edition 2021 and MSRV 1.80.

**Maturity:** active, experimental (versions 0.0.x/0.1.x, commits daily). The format coverage is very broad and many codecs are validated bit-exact against conformance corpora. However, the published crates.io snapshots of individual crates are out of step with one another, and some end-to-end CLI paths fail today (see [CLI gotchas](#cli-gotchas)). Pin exact versions and test your specific path.

## Quick pick

| Need | Use |
|------|-----|
| Decode/encode one format in-process with minimal deps | `oxideav-core` + the one format crate, e.g. `oxideav-flac` ([library usage](#library-usage)) |
| Register "everything" into one context | `oxideav-meta` **from git**, not crates.io ([depending](#depending-on-it)) |
| Open any image/audio/video/PDF/3D file with one call | [`oxideav-io`](#oxideav-io-one-call-openprobe) |
| Multi-output, multithreaded transcode graph from JSON | `oxideav-pipeline` (`Job` + `Executor`) or `oxideav run` |
| Command-line probe/remux/transcode | [`oxideav` CLI](#cli-oxideav) (binary from GitHub Releases or built from the workspace) |
| Playback (SDL2 or winit+wgpu, TUI) | [`oxideplay`](#player-oxideplay) |
| Pixel-format conversion, scaling, palette/dither | `oxideav-pixfmt`, `oxideav-image-filter` |
| Audio effects/resampling | `oxideav-audio-filter` |
| Font parsing/shaping/rasterising, SVG/PDF | `oxideav-ttf`/`-otf`/`-scribe`, `oxideav-svg`, `oxideav-pdf`, `oxideav-raster` |
| 3D assets (STL/OBJ/glTF/USDZ/FBX) | `oxideav-mesh3d` + format crates |
| RTMP ingest/push | `oxideav-rtmp` |
| Look up format support | [format tables](#format-support) |

## Architecture

```
            SourceRegistry (file/mem/data/http/rtmp/generate/bluray/dvd URIs)
                 | SourceOutput::{Bytes, Packets, Frames, MultiTitle}
Bytes -> ContainerRegistry.probe_input -> Demuxer --Packet--> Decoder --Frame--> [StreamFilter / pixfmt convert]
                                                                                     |
                                     Muxer <--Packet-- Encoder <---------Frame-------+
        (remux = Demuxer -> Muxer directly, no decode)
```

**`oxideav-core`** (crates.io `0.1.37`) holds every shared type and trait. Its docs are complete (`missing_docs` is enforced) and it has zero FFI.
- Data types:
  - `Packet` holds compressed data plus `stream_index`, `time_base`, `pts`/`dts`/`duration` and `flags.keyframe`.
  - `Frame` is an enum: `Audio(AudioFrame)`, `Video(VideoFrame)`, `Subtitle(SubtitleCue)` or `Vector(VectorFrame)`.
  - `AudioFrame { samples, pts, data: Vec<Vec<u8>> }` holds interleaved or planar bytes.
  - `VideoFrame { pts, planes: Vec<VideoPlane{stride,data}> }`. Width, height and pixel format are **not** stored in the frame; read them from the stream's `CodecParameters`.
- Stream description:
  - `StreamInfo { index, time_base, duration, start_time, params }`.
  - `CodecParameters` is built with `CodecParameters::audio(CodecId::new("flac"))` or `::video(..)`. Its public fields are `sample_rate`, `channels`, `sample_format`, `width`, `height`, `pixel_format`, `frame_rate`, `extradata`, `color_signal`, `options` and others.
- Time and formats:
  - `TimeBase`/`Rational` arithmetic never panics.
  - `PixelFormat` has 70 variants and `SampleFormat` covers the common layouts.
- Traits:
  - `Decoder`: `send_packet`, `receive_frame`, `flush`, `reset`.
  - `Encoder`: `send_frame`, `receive_packet`, `flush`, `output_params`.
  - `Demuxer`: `streams`, `next_packet`, `seek_to`, `metadata`, `attached_pictures`, `chapters`, `duration_micros`.
  - `Muxer`: `write_header`, `write_packet`, `write_trailer`.
  - `StreamFilter` is the filter trait.
  - Demuxers read from a `Box<dyn ReadSeek>`, and muxers write to a `Box<dyn WriteSeek>`.
- Errors:
  - `Error` is a single enum for the whole framework.
  - Decoders and encoders return `Error::NeedMore` when they need more input and `Error::Eof` when drained. Loop on `receive_*` until you get one of these.
- `RuntimeContext { codecs, containers, sources, filters }` bundles four registries:
  - **`CodecRegistry`**: `register(CodecInfo)`, `first_decoder(&params)`, `first_encoder(&params)`, `decoder_by_impl("flac_sw", &params)`, `implementations(id)`, `decoder_ids()`, `encoder_ids()`, plus tag and payload-magic resolution (FourCC, `wFormatTag`, Matroska CodecID, Ogg BOS magic). One codec id can have several implementations, e.g. software and hardware, ranked by `CodecCapabilities::priority`.
  - **`ContainerRegistry`**: `register_demuxer`, `register_muxer`, `register_extension`, `register_probe`, `probe_input(&mut reader, ext_hint)`, `open_demuxer(name, reader, &codecs)`, `open_muxer(name, writer, &streams)`, `container_for_extension(ext)`. Format detection is content-based: probes score the first 256 KiB, and the file extension only breaks ties.
  - **`SourceRegistry`**: maps URI schemes to drivers. `open(uri)` returns `SourceOutput::Bytes`, `Packets` (e.g. rtmp), `Frames` (e.g. `generate://`) or `MultiTitle` (DVD/Blu-ray).
  - **`FilterRegistry`**: `register(name, factory)` and `make(name, &json_params, &ports)`.
- How sibling crates plug in: each crate exposes `pub fn register(ctx: &mut RuntimeContext)` and usually also `register_codecs(&mut CodecRegistry)` and `register_containers(&mut ContainerRegistry)`. It also invokes `oxideav_core::register!("name", register)`, which generates a hidden `__oxideav_entry`.
- `oxideav-meta` has a `build.rs` that reads the meta crate's own `Cargo.toml` and the active features, then generates an explicit `register_all(ctx)` that calls each enabled sibling. There is no linkme or ctor, so it works on wasm. The older linkme-based `oxideav-format-all` is archived.
- Resolution order is deterministic. The highest probe score or tag confidence wins; ties go to the lowest resolution priority, then to registration order. `register_all` registers crates alphabetically.

**`oxideav-pipeline`** (`0.1.12`) is the executor:
- **Helpers:** `remux(&mut dyn Demuxer, &mut dyn Muxer)`, `transcode_simple(demuxer, muxer_open, &codecs, plan_for)` with `StreamPlan::{Copy, Reencode{output_codec}, Drop}`, and `make_decoder_with` / `make_encoder_with` with `CodecPreferences { no_hardware, prefer, exclude, .. }`.
- **JSON job graph:** `Job::from_json(&str)` → `Executor::new(&job, &ctx).with_threads(n).run()` returns `ExecutorStats`.
  - With `threads >= 2` it runs one thread per stage per track, connected by bounded channels.
  - `Executor::spawn` gives a handle with seek, progress and stop for playback.
  - A pixel-format conversion is inserted automatically when an encoder's `accepted_pixel_formats` excludes the decoder's output.
  - A decode error on one packet is logged and skipped rather than failing the stream.

**`oxideav`** (facade) re-exports `oxideav_core as core`, `oxideav_pipeline as pipeline`, `oxideav_source as source` and `RuntimeContext` (alias `Registries`). It contains no codecs.

**Hardware acceleration:** `oxideav-vaapi`, `-vdpau`, `-nvidia` and `-vulkan-video` (Linux), and `-videotoolbox` and `-audiotoolbox` (macOS). They register as extra implementations of existing codec ids. All are runtime-loaded and partial (🚧). Use the `pure-rust` meta feature, or `--no-hwaccel` / `CodecPreferences{no_hardware:true}`, to avoid them.

## Depending on it

crates.io status was checked on 2026-09-28. The code examples below were compiled and run against the crates.io versions shown.

| Approach | Cargo | Notes |
|---|---|---|
| **Individual crates (recommended for libraries)** | `oxideav-core = "0.1"` + e.g. `oxideav-flac = "0.0"`, `oxideav-basic = "0.0"`, `oxideav-pipeline = "0.1"`, `oxideav-mkv = "0.0"`, `oxideav-h264 = "0.1"`, `oxideav-png = "0.1"`, `oxideav-pixfmt = "0.1"` | Call each crate's `register(&mut ctx)`. Compiles cleanly against current `oxideav-core 0.1.x`. |
| **Everything via `oxideav-meta`** | `oxideav-meta = { git = "https://github.com/OxideAV/oxideav-meta", default-features = false, features = ["pure-rust"] }` | The git version wires 91 siblings (131 decoders / 109 encoders / 61 demuxers / 51 muxers with `pure-rust`), and its siblings resolve from crates.io. **Do not use crates.io `oxideav-meta 0.0.1`**: its generated `register_all` is empty, so it registers nothing. |
| Facade `oxideav` | git/path only | crates.io `oxideav 0.0.3` **fails to compile** against current `oxideav-core` (it calls a removed `RuntimeContext::with_all_features_*`). Not needed: use `oxideav-core` and `oxideav-pipeline` directly. |
| `oxideav-io` (one-call open) | `oxideav-io = { version = "0.1", default-features = false, features = ["registry"] }` | Its default `full` feature pulls `oxideav-meta` from crates.io, which is the broken 0.0.1. Use `registry` and pass your own context to the `*_with` functions. |
| Hacking on the framework | `git clone https://github.com/OxideAV/oxideav-workspace && ./scripts/update-crates.sh && cargo build --workspace` | `update-crates.sh` uses `gh` to clone every sibling into `crates/`, and `[patch.crates-io]` points all `oxideav-*` deps at those local clones. A bare checkout does not build. |

`oxideav-meta` presets:
- `all` (default)
- `pure-rust` (`all` minus hwaccel)
- `audio`, `video`, `image`, `subtitles`, `3d`, `hwaccel`, `source-drivers`
- Per-crate features named after the crate's short name (`aac`, `h264`, `mp4`, ...). `oxideav-mod` is `amiga-mod`.
- `vfw` is opt-in only.

Introspection constants (git version): `ENABLED_SIBLINGS`, `ENABLED_SIBLINGS_BY_CATEGORY` and `category_of("aac")`. `populate_mesh3d_registry(&mut Mesh3DRegistry)` sits behind the `mesh3d` feature.

These crates are **not** wired into meta; call their `register` yourself:
- `oxideav-bmp`, `-ico`, `-tiff`
- `-evc`, `-vc2`
- `-midi`, `-nsf`
- `-rtmp` (source registry: `oxideav_rtmp::register(&mut ctx.sources)`)

Yanked or unpublished (git only): `oxideav-dts`, `-cook`, `-wavpack`, `-lagarith`, `-indeo`, `-aptx`, and the archived `-aiff`.

**Version-skew warning:** each crate is released independently, so crates.io versions can lag each other and git. Some published combinations mismatch at runtime; the CLI tests below turned up a pipeline/mp4 option-name mismatch (`unknown option 'elst_entry_count'`). For the widest format coverage, build from the workspace. For a library, pin exact versions that you have tested together.

## CLI: `oxideav`

**Install:**
- Prebuilt tarballs are on https://github.com/OxideAV/oxideav-workspace/releases (latest `v0.0.5`, 2026-05-27, for linux-x86_64, macos-universal and windows-x86_64). They bundle `oxideav` and `oxideplay`. The release is older than the code: it lacks e.g. an AAC decoder.
- Otherwise build it: `cargo build --release -p oxideav-cli` in the workspace. The crate has `publish = false`, so there is no `cargo install`.

Global flags: `--no-hwaccel`, `--debug`, `--debug-output FILE` and `--buffer-mib N` (prefetch buffer size in MiB). Inputs can be paths or `file://`, `http(s)://` or `rtmp://` URIs.

| Subcommand | Purpose |
|---|---|
| `list` | compiled-in containers (D = demux, M = mux) and codec implementations |
| `info <codec>` | backends, capabilities and encoder option schema for a codec |
| `probe <in>` | container, metadata, streams, attached pictures |
| `remux <in> <out> [--format F]` | stream copy (no re-encode); `%s` in the output fans out DVD/Blu-ray titles |
| `transcode <in> <out> [--codec X] [--codec-audio/--c:a X] [--codec-video/--c:v X] [--codec-subtitle X] [-o KEY=VAL]... [--format F]` | decode and re-encode per stream. With no codec given, audio becomes PCM and video/subtitles are stream-copied. |
| `convert <in> [-resize WxH -blur .. -quality ..] <out>` | ImageMagick-style conversion. Also accepts generators (`xc:red`, `label:Hello`, `gradient:red-blue`), PDF page selectors (`in.pdf[0]`, `in.pdf[2-5]`, `page-%03d.png`) and 3D (`in.obj out.gltf`). |
| `run <job.json \| -> [--inline JSON] [--threads N]`, `validate`, `dry-run` | JSON job graph |
| `bench <codec> [--all]` | encode/decode throughput per backend |

The following commands were checked against a workspace build at tip (2026-09-28):

```sh
oxideav probe in.mp4                                     # Format: mp4, per-stream codec/size/rate/duration
oxideav remux in.mp4 out.mkv                             # stream copy, works for h264+aac
oxideav transcode in.wav out.flac --codec-audio flac     # works (without --codec-audio the audio stays PCM and the FLAC muxer refuses it)
oxideav transcode in.ogg out.flac --codec-audio flac     # Vorbis -> FLAC works
oxideav transcode video_only.mp4 out.mkv --codec-video mjpeg   # h264 decode -> MJPEG encode works
oxideav transcode video_only.mp4 out.mkv --codec-video h264    # h264 re-encode works (ffmpeg decodes the output)
# filter + resample + encode through a JSON job (Opus requires 48 kHz input):
oxideav run --inline '{"out.ogg":{"audio":[{"filter":"resample","params":{"rate":48000},"input":{"from":"in.flac"},"codec":"opus"}]}}'
```

JSON job schema:
- Top-level keys are output paths, or the reserved sinks `@null`, `@display` and `@out`. Other `@name` keys are aliases, and a `threads` key sets the thread budget.
- Each value groups tracks under `audio`, `video`, `subtitle` or `all`.
- A track is a recursive input tree:
  - `{"from": "uri"}` is a source.
  - `{"filter": "name", "params": {...}, "input": <node>}` applies a filter.
  - `{"convert": "rgba", "input": <node>}` converts pixel format.
- A track can also carry `codec`, `codec_params` and `stream_selector: {"kind": "video", "index": 0}`.

Filter names:
- Video filters use the `video.` prefix, e.g. `video.resize` with `{width, height, interpolation}` or `video.blur`.
- Audio filters include `volume` (`{"gain_db": -3}` or `{"gain": 0.5}`), `resample` (`{"rate": 48000}`), `echo`, `compressor`, `limiter`, `equalizer`, `reverb`, `loudness_itu`, `spectrogram` and about 50 more.

### CLI gotchas

These were observed at tip on 2026-09-28. Re-test before relying on any of them.
- **Extracting frames to images does not work from the CLI today:**
  - `convert movie.mp4 frame-%03d.png` is refused, because `%d` templates only apply to PDF inputs.
  - A `run` job writing `out.png` from a video fails with "PNG muxer: no packets written" (MKV input) or `unknown option 'elst_entry_count'` (MP4 input).
  - `transcode ... --codec-video png` panics.
  - Use the [library route](#extract-frames-to-png) instead.
- AAC inside MP4/MKV fails to decode in `transcode` ("packet has neither an ADTS nor a LOAS syncword"), so `--codec-audio X` on a typical h264+aac MP4 fails. Remux (copy) of the same file works.
- Encoding AAC into MP4 fails ("aac stream missing extradata").
- Other encoders reject inputs:
  - The Opus encoder only accepts 48 kHz; resample first.
  - The VP9 encoder needs `pixel_format` set.
  - No `.mp3` muxer is registered for output.
- WAV output accepts exactly one audio stream. Drop the video by using a job with only an `audio` track.

## Player: `oxideplay`

- Usage: `oxideplay <file-or-url>`, or `oxideplay --job job.json`, where `@display` / `@out` bind to the player.
- Video output:
  - SDL2 is the default, and `libSDL2` is loaded at runtime. There is no build-time link; if SDL2 is missing the player exits cleanly.
  - `--vo winit` uses winit+wgpu and adds an egui overlay.
- Audio goes through `oxideav-sysaudio`, which runtime-loads ALSA, PulseAudio, WASAPI, CoreAudio or OSS.
- Keys: `q` quit, `space` pause, `←/→` ±10 s, `↑/↓` ±1 min, `PgUp/PgDn` ±10 min, `*` and `/` for volume.

## Library usage

All three examples below compiled and ran against crates.io `oxideav-core 0.1.37`, `oxideav-flac 0.0.11`, `oxideav-basic 0.0.10`, `oxideav-pipeline 0.1.12`, `oxideav-source 0.1.5`, `oxideav-mkv 0.0.10`, `oxideav-h264 0.1.8`, `oxideav-png 0.1.8` and `oxideav-pixfmt 0.1.8`.

### Encode PCM, mux, probe, demux, decode (FLAC)

```rust
use oxideav_core::{
    AudioFrame, CodecId, CodecParameters, Error, Frame, ReadSeek, RuntimeContext, SampleFormat,
    StreamInfo, TimeBase, WriteSeek,
};

fn main() -> oxideav_core::Result<()> {
    let mut ctx = RuntimeContext::new();
    oxideav_flac::register(&mut ctx);   // flac codec + container
    oxideav_basic::register(&mut ctx);  // wav/slin/y4m containers + pcm_* codecs
    oxideav_source::register(&mut ctx); // file:// etc. (needed by the pipeline Executor)

    // --- encode 1 s of stereo S16 PCM ---
    let rate = 44_100u32;
    let mut params = CodecParameters::audio(CodecId::new("flac"));
    params.sample_rate = Some(rate);
    params.channels = Some(2);
    params.sample_format = Some(SampleFormat::S16);
    let mut enc = ctx.codecs.first_encoder(&params)?;
    let mut pcm = Vec::new(); // interleaved little-endian S16
    for i in 0..rate {
        let s = ((i as f32 * 440.0 * std::f32::consts::TAU / rate as f32).sin() * 8000.0) as i16;
        pcm.extend_from_slice(&s.to_le_bytes());
        pcm.extend_from_slice(&s.to_le_bytes());
    }
    enc.send_frame(&Frame::Audio(AudioFrame { samples: rate, pts: Some(0), data: vec![pcm] }))?;
    enc.flush()?;

    // --- mux ---
    let stream = StreamInfo {
        index: 0,
        time_base: TimeBase::new(1, rate as i64),
        duration: None,
        start_time: Some(0),
        params: enc.output_params().clone(), // carries STREAMINFO in extradata
    };
    let out: Box<dyn WriteSeek> = Box::new(std::fs::File::create("tone.flac")?);
    let mut mux = ctx.containers.open_muxer("flac", out, &[stream])?;
    mux.write_header()?;
    loop {
        match enc.receive_packet() {
            Ok(pkt) => mux.write_packet(&pkt)?,
            Err(Error::NeedMore) | Err(Error::Eof) => break,
            Err(e) => return Err(e),
        }
    }
    mux.write_trailer()?;
    drop(mux);

    // --- probe + demux + decode ---
    let mut input: Box<dyn ReadSeek> = Box::new(std::fs::File::open("tone.flac")?);
    let format = ctx.containers.probe_input(&mut *input, Some("flac"))?; // "flac"
    let mut dmx = ctx.containers.open_demuxer(&format, input, &ctx.codecs)?;
    let mut dec = ctx.codecs.first_decoder(&dmx.streams()[0].params)?;
    let mut total = 0u64;
    loop {
        match dmx.next_packet() {
            Ok(pkt) => {
                dec.send_packet(&pkt)?;
                loop {
                    match dec.receive_frame() {
                        Ok(Frame::Audio(a)) => total += a.samples as u64, // a.data[0] = interleaved PCM
                        Ok(_) => {}
                        Err(Error::NeedMore) | Err(Error::Eof) => break,
                        Err(e) => return Err(e),
                    }
                }
            }
            Err(Error::Eof) => break,
            Err(e) => return Err(e),
        }
    }
    assert_eq!(total, 44_100);
    Ok(())
}
```

### Transcode, remux and JSON jobs through `oxideav-pipeline`

This continues with the same `ctx` as above.

```rust
// FLAC -> WAV (pcm_s16le) with the multi-stream helper
let input: Box<dyn oxideav_core::ReadSeek> = Box::new(std::fs::File::open("tone.flac")?);
let mut dmx = ctx.containers.open_demuxer("flac", input, &ctx.codecs)?;
let stats = oxideav_pipeline::transcode_simple(
    &mut *dmx,
    |streams| {
        let out: Box<dyn oxideav_core::WriteSeek> = Box::new(std::fs::File::create("tone.wav")?);
        ctx.containers.open_muxer("wav", out, streams)
    },
    &ctx.codecs,
    |_stream| Ok(oxideav_pipeline::StreamPlan::Reencode { output_codec: "pcm_s16le".into() }),
    // or StreamPlan::Copy / StreamPlan::Drop per stream
)?;

// Pure stream copy: oxideav_pipeline::remux(&mut *demuxer, &mut *muxer) -> packet count

// Declarative job (same schema as `oxideav run`)
let job = oxideav_pipeline::Job::from_json(
    r#"{"tone2.wav": {"audio": [{"from": "tone.flac", "codec": "pcm_s16le"}]}}"#)?;
let st = oxideav_pipeline::Executor::new(&job, &ctx).with_threads(1).run()?;
println!("{} packets read, {} frames decoded", st.packets_read, st.frames_decoded);
```

### Extract frames to PNG

This decodes H.264 from MKV, converts each frame to RGBA and encodes one PNG per frame.

```rust
use oxideav_core::{CodecId, CodecParameters, Error, Frame, PixelFormat, ReadSeek, RuntimeContext};
use oxideav_pixfmt::{convert, ConvertOptions, FrameInfo};

fn main() -> oxideav_core::Result<()> {
    let mut ctx = RuntimeContext::new();
    oxideav_mkv::register(&mut ctx);
    oxideav_h264::register(&mut ctx);
    oxideav_png::register(&mut ctx);

    let mut input: Box<dyn ReadSeek> = Box::new(std::fs::File::open("in.mkv")?);
    let fmt = ctx.containers.probe_input(&mut *input, Some("mkv"))?;
    let mut dmx = ctx.containers.open_demuxer(&fmt, input, &ctx.codecs)?;
    let vs = dmx.streams().iter().find(|s| s.params.width.is_some()).cloned()
        .ok_or_else(|| Error::invalid("no video stream"))?;
    let (w, h) = (vs.params.width.unwrap(), vs.params.height.unwrap());
    // the demuxer may leave pixel_format unset (MKV/H.264 did); 4:2:0 is the H.264 norm
    let src_fmt = vs.params.pixel_format.unwrap_or(PixelFormat::Yuv420P);
    let mut dec = ctx.codecs.first_decoder(&vs.params)?;

    let mut n = 0;
    loop {
        let pkt = match dmx.next_packet() {
            Ok(p) => p,
            Err(Error::Eof) => break, // (call dec.flush() and drain again to get trailing frames)
            Err(e) => return Err(e),
        };
        if pkt.stream_index != vs.index { continue; }
        dec.send_packet(&pkt)?;
        loop {
            match dec.receive_frame() {
                Ok(Frame::Video(vf)) => {
                    let rgba = convert(&vf, FrameInfo::new(src_fmt, w, h), PixelFormat::Rgba,
                                       &ConvertOptions::default())?;
                    let mut p = CodecParameters::video(CodecId::new("png"));
                    p.width = Some(w);
                    p.height = Some(h);
                    p.pixel_format = Some(PixelFormat::Rgba);
                    let mut enc = ctx.codecs.first_encoder(&p)?;
                    enc.send_frame(&Frame::Video(rgba))?;
                    enc.flush()?;
                    let png = enc.receive_packet()?; // packet bytes = a complete .png file
                    std::fs::write(format!("frame-{n:03}.png"), &png.data)?;
                    n += 1;
                }
                Ok(_) => {}
                Err(Error::NeedMore) | Err(Error::Eof) => break,
                Err(e) => return Err(e),
            }
        }
    }
    Ok(())
}
```

### `oxideav-io`: one-call open/probe

Taken from the `oxideav-io` README; this was not compiled here. The API is `open(path) -> Opened::{Image(RgbaImage{width,height,pixels,stride}), Vector, Scene, Mesh, Media(MediaReader)}`, together with `open_rgba`, `open_rgb`, `open_media`, `ping_format` (reads at most 257 KiB), `probe(path) -> Probe { kind, container, duration_secs, metadata, streams }`, `save(&opened, "out.jpg")` and `transcode_with`. Each has a `*_with(&ctx, Source::Path/Uri/Bytes/Reader, &OpenOptions)` variant. `OpenOptions` can allow or deny containers and codecs, which is useful for sandboxing untrusted input. With crates.io, use `default-features = false, features = ["registry"]` and the `*_with` functions. The zero-config variants depend on `oxideav-meta`.

### Writing your own codec/container crate

The sketch below follows the pattern every sibling uses. The builder and registry method names come from `oxideav-core`, but you supply the factory functions.

```rust
pub fn register(ctx: &mut oxideav_core::RuntimeContext) {
    ctx.codecs.register(oxideav_core::CodecInfo::new(oxideav_core::CodecId::new("mycodec"))
        .decoder(make_decoder)       // fn(&CodecParameters) -> Result<Box<dyn Decoder>>
        .encoder(make_encoder));     // optionally .tag(CodecTag) / .payload_magic(b"...") / .capabilities(..)
    ctx.containers.register_demuxer("myfmt", open_demuxer); // fn(Box<dyn ReadSeek>, &dyn CodecResolver)
    ctx.containers.register_muxer("myfmt", open_muxer);     // fn(Box<dyn WriteSeek>, &[StreamInfo])
    ctx.containers.register_extension("myf", "myfmt");
    ctx.containers.register_probe("myfmt", probe);          // fn(&ProbeData) -> u8 score
}
oxideav_core::register!("mycodec", register); // must be reachable at the crate root for oxideav-meta
```

**Library gotchas:**
- Per-crate README snippets are sometimes stale. Some call `codecs.make_decoder(...)` or `containers.open(...)`, which are **not** methods on the current core registries. Use `first_decoder` / `first_encoder` / `open_demuxer`, or the free functions `oxideav_pipeline::make_decoder(&reg, &params)`. Each codec crate also exposes its own `decoder::make_decoder(&params)`.
- When a hardware bridge also claims a codec id, `first_decoder` may choose it. Pin the software implementation with `decoder_by_impl("<id>_sw", &params)`, or use `CodecPreferences { no_hardware: true, .. }`.
- The PCM codec ids are `pcm_s16le`, `pcm_f32le`, etc. (from `oxideav-basic`). FLAC decode outputs `U8`, `S16`, `S24` or `S32` depending on the stream's bit depth.
- Repository descriptions on GitHub are often outdated. For example, "H.264 decoder: I-slice only" is wrong; H.264 now decodes and encodes. The workspace README's status tables are the current source, and `oxideav list` / `oxideav info <codec>` are authoritative for any given build.

## Format support

The status column is condensed from the oxideav-workspace README current-status tables (2026-09-28). ✅ means working end-to-end, with a percentage where the README gives one. 🚧 means partial or scaffold. — means not implemented. "crates.io" is the latest published version; "git" means yanked or unpublished. "meta" is the feature name in `oxideav-meta` (✗ means you must register the crate manually).

### Containers

| Crate | Formats | Demux | Mux | Seek | crates.io | meta |
|---|---|:-:|:-:|:-:|---|---|
| oxideav-basic | WAV (+RF64/BW64, BWF), raw PCM, slin, Y4M | ✅ | ✅ | ✅ | 0.0.10 | basic |
| oxideav-mp4 | MP4/ISMV/CMAF/DASH fragments, CENC decrypt+encrypt | ✅ | ✅ | ✅ | 0.0.10 | mp4 |
| oxideav-mov | QuickTime (QTFF) | ✅ | ✅ | ✅ | 0.0.5 | mov |
| oxideav-mkv | Matroska/WebM (full RFC 9559) | ✅ | ✅ | ✅ | 0.0.10 | mkv |
| oxideav-ogg | Ogg (Vorbis/Opus/Theora/Speex/FLAC, Skeleton) | ✅ | ✅ | ✅ | 0.1.8 | ogg |
| oxideav-avi | AVI 1.0 + OpenDML | ✅ | ✅ | ✅ | 0.0.10 | avi |
| oxideav-mpegts | MPEG-TS (DVB/ATSC SI, T-STD) | ✅ | ✅ | ✅ | 0.0.3 | mpegts |
| oxideav-flv | FLV + Enhanced-RTMP | ✅ | ✅ | — | 0.0.5 | flv |
| oxideav-iff | IFF 85: 8SVX, ILBM, ANIM, AIFF/AIFF-C | ✅ | ✅ | — | 0.0.10 | iff |
| oxideav-amv | AMV (incl. its video + IMA-ADPCM) | ✅ | ✅ | — | 0.0.10 | amv |
| oxideav-heif | HEIF/HEIC/AVIF/MIAF items and sequences | ✅ | ✅ | — | 0.0.4 | heif |
| oxideav-bluray | BD-ROM (UDF 2.50, BDMV, m2ts, `bluray://`) | ✅ | — | — | 0.0.4 | bluray |
| oxideav-dvd | DVD-Video (IFO/VOB, `dvd://`, nav VM) | ✅ | — | — | 0.0.4 | dvd |
| oxideav-aacs | AACS decryption (Common/BD-Prerecorded 0.953) | lib | — | — | 0.1.3 | ✗ |
| oxideav-riff | RIFF chunk walker (library only) | lib | — | — | 0.0.2 | riff |
| oxideav-id3 | ID3v1/v2.2–2.4 tags + chapters | lib | lib | — | 0.0.6 | ✗ |

MP3, FLAC, GIF, PNG, WebP, TIFF, JPEG, BMP, ICO and the trackers also ship their own containers inside their codec crates. `IVF` is demux-only. Remuxing works between any containers whose codecs need no rewriting.

### Audio codecs

| Crate | Codec | Decode | Encode | crates.io | meta |
|---|---|---|---|---|---|
| oxideav-basic | PCM s8/16/24/32/f32/f64, slin | ✅ | ✅ | 0.0.10 | basic |
| oxideav-flac | FLAC | ✅ 100% | ✅ 100% | 0.0.11 | flac |
| oxideav-vorbis | Vorbis | ✅ ~98% | 🟡 ~93% | 0.0.12 | vorbis |
| oxideav-opus | Opus (SILK/CELT/hybrid) | ✅ ~96% | ✅ ~98% (48 kHz in) | 0.0.14 | opus |
| oxideav-celt | CELT | ✅ ~98% | ✅ ~98% | 0.1.13 | celt |
| oxideav-mp1 / -mp2 / -mp3 | MPEG-1/2 Layer I/II/III | ✅ ~97–99% | ✅ ~95–100% | 0.0.7 / 0.0.10 / 0.1.3 | mp1/mp2/mp3 |
| oxideav-aac | AAC LC/HE-v1/v2/LD/SSR/BSAC | 🚧 ~96% | ✅ ~99% | 0.1.7 | aac |
| oxideav-ac3 | AC-3 / E-AC-3 | ✅ ~97% | ✅ ~98% | 0.0.11 | ac3 |
| oxideav-ac4 | Dolby AC-4 | 🚧 ~99% | 🚧 ~96% | 0.0.8 | ac4 |
| oxideav-dts | DTS Core (+ext) | ✅ ~98% | 🟢 ~85% | git | ✗ |
| oxideav-speex | Speex NB/WB | 🚧 ~85% | ✅ | 0.0.10 | speex |
| oxideav-gsm | GSM 06.10 FR (+HR) | ✅ | ✅ | 0.0.10 | gsm |
| oxideav-g711 / -g722 / -g7231 / -g728 / -g729 | ITU G.711/722/723.1/728/729 | ✅ (G.729 ~92%) | ✅ (G.729 ~78%) | 0.0.7–0.0.8 | g711… |
| oxideav-adpcm | MS/IMA/Yamaha ADPCM, G.726, OKI/VOX | ✅ | ✅ | 0.0.7 | adpcm |
| oxideav-ilbc | iLBC (RFC 3951) | ✅ | ✅ | 0.0.7 | ilbc |
| oxideav-wma | WMA v1/v2 | ✅ ~95% | 🟢 ~90% | 0.0.4 | wma |
| oxideav-ape | Monkey's Audio | ✅ ~95% | ✅ ~90% | 0.0.4 | ape |
| oxideav-wavpack | WavPack | ✅ | ✅ ~97% | git | ✗ |
| oxideav-tta | True Audio | ✅ ~99% | ✅ ~97% | 0.0.4 | tta |
| oxideav-shorten | Shorten | ✅ ~95% | ✅ ~90% | 0.0.4 | shorten |
| oxideav-musepack | Musepack SV7/SV8 | ✅ ~98% | ✅ ~95% | 0.0.4 | musepack |
| oxideav-cook | RealAudio Cook | 🚧 ~70% | — | git | ✗ |
| oxideav-aptx | aptX / aptX HD | 🚧 stub | — | git | ✗ |
| oxideav-midi | Standard MIDI File → PCM (soundfonts) | ✅ ~99% | ✅ SMF write | 0.0.5 | ✗ |
| oxideav-nsf | NES Sound Format (6502 + APU emu) | 🚧 ~98% | — | 0.0.5 | ✗ |
| oxideav-mod / -s3m | MOD, STM, XM, IT / S3M trackers | ✅ ~85–98% | — (by design) | 0.0.10 / 0.0.9 | amiga-mod / s3m |

### Video codecs

| Crate | Codec | Decode | Encode | crates.io | meta |
|---|---|---|---|---|---|
| oxideav-h264 | H.264/AVC | ✅ ~97% | ✅ ~98% | 0.1.8 | h264 |
| oxideav-h265 | H.265/HEVC | ✅ ~99% | 🟢 ~90% | 0.0.12 | h265 |
| oxideav-h266 | H.266/VVC | ✅ 100% | 🚧 ~95% | 0.0.9 | h266 |
| oxideav-av1 | AV1 | ✅ 100% | 🟢 ~99% | 0.1.19 | av1 |
| oxideav-vp8 / -vp9 | VP8 / VP9 | ✅ 100% / ~97% | ✅ 100% / ~99% | 0.2.7 / 0.0.13 | vp8/vp9 |
| oxideav-vp6 | VP6 (vp6f/vp6a) | 🚧 ~92% | 🚧 ~70% | 0.0.9 | vp6 |
| oxideav-evc | MPEG-5 EVC | 🟢 ~90% | 🟢 ~88% | 0.0.4 | ✗ |
| oxideav-mpeg12video | MPEG-1 / MPEG-2 video | ✅ ~96–97% | 🚧 ~90% / ✅ ~95% | 0.0.13 | mpeg12video |
| oxideav-mpeg4video | MPEG-4 Part 2 (ASP) | ✅ ~97% | 🟢 ~93% | 0.1.8 | mpeg4video |
| oxideav-msmpeg4 | MS MPEG-4 v1/v2/v3 (DIV3) | 🚧 ~95% | ✅ ~92% | 0.0.10 | msmpeg4 |
| oxideav-h261 / -h263 | H.261 / H.263 | ✅ ~99% | ✅ ~98% / 🟡 ~91% | 0.0.7 / 0.0.10 | h261/h263 |
| oxideav-theora | Theora | ✅ 100% | ✅ ~97% | 0.0.12 | theora |
| oxideav-mjpeg | Motion JPEG + still JPEG | ✅ 100% | ✅ 100% | 0.1.9 | mjpeg |
| oxideav-prores | Apple ProRes 422 | ✅ 100% | ✅ 100% | 0.1.1 | prores |
| oxideav-ffv1 | FFV1 | ✅ 100% | ✅ 100% | 0.0.9 | ffv1 |
| oxideav-huffyuv | HuffYUV / FFVHuff | ✅ ~97% | ✅ ~97% | 0.0.3 | huffyuv |
| oxideav-utvideo | Ut Video | ✅ ~98% | ✅ ~98% | 0.0.3 | utvideo |
| oxideav-magicyuv | MagicYUV | ✅ 100% | ✅ 100% | 0.0.6 | magicyuv |
| oxideav-lagarith | Lagarith | ✅ 100% | ✅ ~95% | git | ✗ |
| oxideav-dirac / -vc2 | Dirac + VC-2 / standalone SMPTE VC-2 | ✅ ~99% | ✅ ~99% | 0.0.9 / 0.0.2 | dirac / ✗ |
| oxideav-cinepak | Cinepak | ✅ ~98% | ✅ ~98% | 0.0.3 | cinepak |
| oxideav-svq | Sorenson SVQ1 / SVQ3 | 🚧 ~97% | ✅ SVQ1 | 0.0.3 | svq |
| oxideav-indeo | Intel Indeo 2/3/4/5 | ✅ IV3 ~95%, 🚧 2/4/5 ~75% | — | git | ✗ |
| oxideav-amv | AMV video | ✅ ~92% | 🚧 ~85% | 0.0.10 | amv |
| oxideav-vfw | x86 emulator hosting legacy Win32 VfW codec DLLs | delegation | | 0.1.1 | vfw (opt-in) |

### Image codecs

All image crates decode and encode unless noted.

| Crate | Format | Status | crates.io | meta |
|---|---|---|---|---|
| oxideav-png | PNG + APNG | ✅ 100% | 0.1.8 | png |
| oxideav-gif | GIF 87a/89a + animation | ✅ 100% | 0.0.11 | gif |
| oxideav-webp | WebP lossy/lossless/animated | ✅ 100% | 0.2.3 | webp |
| oxideav-mjpeg | JPEG (baseline/progressive/hierarchical) | ✅ ~95% dec / ~90% enc | 0.1.9 | mjpeg |
| oxideav-tiff | TIFF 6.0 + BigTIFF, CCITT, JPEG-in-TIFF | ✅ 100% / ~97% | 0.0.6 | ✗ |
| oxideav-bmp / -ico | BMP / ICO, CUR, ANI | ✅ ~97–98% | 0.1.6 / 0.0.7 | ✗ |
| oxideav-jpeg2000 | JPEG 2000 Part 1 + HTJ2K | ✅ ~98% | 0.0.16 | jpeg2000 |
| oxideav-jpegxl | JPEG XL | ✅ ~99% decode, encode retired | 0.0.13 | jpegxl |
| oxideav-jpegxs | JPEG XS | ✅ 100% | 0.0.7 | jpegxs |
| oxideav-avif / oxideav-heif | AVIF / HEIC (via av1 / h265) | ✅ ~98–99% dec, ~97% enc | 0.0.11 / 0.0.4 | avif / heif |
| oxideav-openexr | OpenEXR | ✅ 100% / ~99% | 0.0.6 | openexr |
| oxideav-hdr | Radiance RGBE | ✅ ~99% | 0.0.5 | hdr |
| oxideav-dds | DDS (BC1–7) | ✅ ~99% | 0.0.5 | dds |
| oxideav-pbm | Netpbm PBM/PGM/PPM/PAM | ✅ ~95% | 0.0.4 | pbm |
| oxideav-qoi / -farbfeld / -tga / -pcx / -wbmp | QOI, farbfeld, TGA, PCX, WBMP | ✅ 100% | 0.1.4 / 0.0.4 / 0.0.3 / 0.1.1 / 0.0.3 | qoi/farbfeld/tga/pcx/wbmp |
| oxideav-pict | Apple PICT | ✅ ~99% / ~97% | 0.0.4 | pict |
| oxideav-icer | JPL ICER | ✅ ~98% / ~95% | 0.0.5 | icer |
| oxideav-svg | SVG 1.1/2 (vector frames) | ✅ ~99% / ~97% | 0.1.8 | svg |
| oxideav-pdf | PDF read (to Scene) + write | ✅ ~99% | 0.2.0 | pdf |
| oxideav-embroidery | DST, PES/PEC, EXP, JEF, HUS/VIP, PHC | ✅ | 0.0.3 | embroidery |

### Subtitles

| Crate | Formats | Decode | Encode | crates.io | meta |
|---|---|---|---|---|---|
| oxideav-subtitle | SRT, WebVTT, TTML, SAMI, EBU STL, MicroDVD, MPL2, MPsub, VPlayer, PJS, AQTitle, JACOsub, RealText, SubViewer 1/2 | ✅ | ✅ | 0.1.2 | subtitle |
| oxideav-ass | ASS/SSA with override tags | ✅ | ✅ | 0.0.9 | ass |
| oxideav-sub-image | PGS/HDMV `.sup`, DVB subtitles, VobSub | ✅ | ✅ (VobSub decode only) | 0.0.7 | sub-image |

### 3D scenes

These crates use `oxideav-mesh3d`'s `Mesh3DRegistry` rather than the codec registry. Populate it with `oxideav_meta::populate_mesh3d_registry`.

| Crate | Format | Decode | Encode | crates.io |
|---|---|---|---|---|
| oxideav-stl | STL ASCII/binary | ✅ ~99% | ✅ ~99% | 0.0.4 |
| oxideav-obj | OBJ + MTL | ✅ ~99% | ✅ ~99% | 0.0.4 |
| oxideav-gltf | glTF 2.0 / .glb | ✅ ~98% | ✅ ~95% | 0.0.4 |
| oxideav-usdz | USDZ/USDA | ✅ ~97% | ✅ ~92% | 0.0.4 |
| oxideav-fbx | FBX | 🚧 ~95% | ✅ ~95% | 0.0.3 |
| oxideav-ifc | IFC (BIM, STEP) | ✅ | — | 0.0.2 |
| oxideav-render | Scene3D → raster (scanline/raycast) | 🚧 | | 0.0.4 |

### Filters, conversion, text, sources, output

| Crate | Role | Status | crates.io | meta |
|---|---|---|---|---|
| oxideav-pixfmt | 70 pixel formats, full conversion matrix, palette generation, dithering (`convert(&frame, FrameInfo, dst, &ConvertOptions)`) | ✅ | 0.1.8 | (dependency) |
| oxideav-image-filter | 136 filter types (resize, blur, edge, crop, rotate, ...), JSON names `video.*` | ✅ | 0.1.2 | image-filter |
| oxideav-audio-filter | about 50 filters (volume, resample, echo, EQ, compressor, reverb, loudness, spectrogram, ...) | ✅ | 0.1.2 | audio-filter |
| oxideav-source | `SourceRegistry` drivers: file, mem, data:, slice, concat; `BufferedSource` prefetch | ✅ | 0.1.5 | source |
| oxideav-http | `http(s)://` via ureq + rustls, Range seek | ✅ | 0.0.8 | http |
| oxideav-rtmp | RTMP server/client, `rtmp://` packet source, Enhanced-RTMP | ✅ (no RTMPS) | 0.0.6 | ✗ (`register(&mut ctx.sources)`) |
| oxideav-generator | `generate://` synthetic audio/image/video, `xc:`/`gradient:`/`label:` | ✅ | 0.1.4 | generator |
| oxideav-sysaudio | native audio output (ALSA/Pulse/WASAPI/CoreAudio/OSS, runtime-loaded) | ✅ | 0.1.1 | ✗ |
| oxideav-ttf / -otf / -scribe / -raster | TrueType/OpenType parsing, shaping (GSUB/GPOS, bidi), vector→raster | ✅ | 0.1.8 / 0.1.4 / 0.1.10 / 0.1.3 | ✗ |
| oxideav-scene | time-based scene model (PDF pages, compositor, NLE) | 🚧 | 0.1.4 | ✗ |
| oxideav-mesh3d | typed Scene3D model + registry | ✅ | 0.0.6 | mesh3d |
| oxideav-bitstream | H.264/HEVC/AV1 header parse helpers for HW bridges | ✅ | 0.0.2 | ✗ |
| oxideav-io | open/probe/save/transcode facade | ✅ still images; A/V transcode pending | 0.1.0 | ✗ |
| oxideav-cli-convert | engine behind `oxideav convert` | ✅ | 0.0.5 | ✗ |
| oxideav-videotoolbox / -audiotoolbox | macOS hardware decode/encode | 🚧 | 0.0.3 | hwaccel |
| oxideav-vaapi / -vdpau / -nvidia / -vulkan-video | Linux (+Windows for Vulkan) hardware decode/encode | 🚧 | 0.0.2–0.0.3 | hwaccel |

### Other repos in the org

- `oxideav-workspace` builds the CLI, the player and cross-crate tests.
- `oxideav` is the facade, and `oxideav-meta` is the aggregator.
- `oxidepbx` is a pure-Rust PBX (SIP/IAX2/RTP) built on OxideAV codecs; it is not covered here.
- `opendocs` holds public format documentation. `docs` is private.
- These are archived and should not be used:
  - `oxideav-codec` and `oxideav-container`: their traits moved into `oxideav-core`.
  - `oxideav-job`: moved into `oxideav-pipeline`.
  - `oxideav-format-all`: replaced by `oxideav-meta`.
  - `oxideav-tracevfw`: moved to KarpelesLab/univdreams.
  - `oxideav-aiff`: moved into `oxideav-iff`.
