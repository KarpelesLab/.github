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
| 3D assets (STL/OBJ/glTF/USDZ/FBX/VRML/X3D) and CAD/BIM (STEP, IFC) | `oxideav-mesh3d` + format crates ([3D table](#3d-scenes)) |
| Render a 3D scene to pixels (CPU, or GPU via wgpu) | `oxideav-render`; `oxideav-render-vulkan` for GPU raster/path tracing |
| Read/write one still-image format without the framework | the format crate alone, `default-features = false` ([still images](#still-images-without-the-framework)) |
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

crates.io status was last checked on 2026-10-04. The code examples below were compiled and run against the crates.io versions shown.

| Approach | Cargo | Notes |
|---|---|---|
| **Individual crates (recommended for libraries)** | `oxideav-core = "0.1"` + e.g. `oxideav-flac = "0.0"`, `oxideav-basic = "0.0"`, `oxideav-pipeline = "0.1"`, `oxideav-mkv = "0.0"`, `oxideav-h264 = "0.1"`, `oxideav-png = "0.1"`, `oxideav-pixfmt = "0.1"` | Call each crate's `register(&mut ctx)`. Compiles cleanly against current `oxideav-core 0.1.x`. |
| **Everything via `oxideav-meta`** | `oxideav-meta = { git = "https://github.com/OxideAV/oxideav-meta", default-features = false, features = ["pure-rust"] }` | The git version wires 91+ siblings (131 decoders / 109 encoders / 61 demuxers / 51 muxers with `pure-rust`), and its siblings resolve from crates.io. **Do not use crates.io `oxideav-meta 0.0.1`** (still the latest release): its generated `register_all` is empty, so it registers nothing. **Needs a patch today**, see below. |
| Facade `oxideav` | git/path only | crates.io `oxideav 0.0.3` (still the latest release) **fails to compile** against `oxideav-core 0.1.37` (E0599: it calls the removed `RuntimeContext::with_all_features_traced` / `_filtered`). Not needed: use `oxideav-core` and `oxideav-pipeline` directly. |
| `oxideav-io` (one-call open) | `oxideav-io = { version = "0.1", default-features = false, features = ["registry"] }` | Its default `full` feature pulls `oxideav-meta` from crates.io, which is the broken 0.0.1. Use `registry` and pass your own context to the `*_with` functions. |
| Hacking on the framework | `git clone https://github.com/OxideAV/oxideav-workspace && ./scripts/update-crates.sh && cargo build --workspace` | `update-crates.sh` uses `gh` to clone every sibling into `crates/`, and `[patch.crates-io]` points all `oxideav-*` deps at those local clones. A bare checkout does not build. |

**git `oxideav-meta` does not compile without a patch (checked 2026-10-04).** Any feature set that includes `3d` (so also the default `all` and `pure-rust`) fails with `E0425: cannot find function register_mesh3d in crate oxideav_step`, because crates.io `oxideav-step` is still an empty `0.0.0` name placeholder. Either list the features you need without `3d`/`step`, or point `oxideav-step` at git:

```toml
[patch.crates-io]
oxideav-step = { git = "https://github.com/OxideAV/oxideav-step" }
```

With this patch, `features = ["3d"]` builds (verified; `cargo check` resolves `oxideav-ifc 0.0.3`, `oxideav-render 0.0.5`, `oxideav-vrml 0.0.1` and `oxideav-x3d 0.0.1` from crates.io). Fetching the git dependency may need `CARGO_NET_GIT_FETCH_WITH_CLI=true` if cargo's built-in git client cannot authenticate.

`oxideav-meta` presets:
- `all` (default)
- `pure-rust` (`all` minus hwaccel)
- `audio`, `video`, `image`, `subtitles`, `3d`, `hwaccel`, `source-drivers`
- `3d` is `mesh3d, stl, obj, gltf, usdz, fbx, ifc, vrml, x3d, step, render`. `oxideav-render-vulkan` is deliberately not in meta.
- Per-crate features named after the crate's short name (`aac`, `h264`, `mp4`, ...). `oxideav-mod` is `amiga-mod`.
- `vfw` is opt-in only.

Introspection constants (git version): `ENABLED_SIBLINGS`, `ENABLED_SIBLINGS_BY_CATEGORY` and `category_of("aac")`. `populate_mesh3d_registry(&mut Mesh3DRegistry)` sits behind the `mesh3d` feature.

These crates are **not** wired into meta; call their `register` yourself:
- `oxideav-bmp`, `-ico`, `-tiff`
- `-evc`, `-vc2`
- `-midi`, `-nsf`
- `-rtmp` (source registry: `oxideav_rtmp::register(&mut ctx.sources)`)

Yanked or unpublished (git only): `oxideav-dts`, `-cook`, `-wavpack`, `-lagarith`, `-indeo`, `-aptx`, and the archived `-aiff`.

**Version-skew warning:** each crate is released independently, so crates.io versions can lag each other and git. As of 2026-10-04 the latest crates.io releases of core, pipeline, mp4, mkv, aac, h264, png, opus and io all compile together on `oxideav-core 0.1.37`, but some combinations still fail at runtime: with `oxideav-mp4 0.0.10` + `oxideav-aac 0.1.7`, every AAC packet from an MP4 is rejected with `packet has neither an ADTS nor a LOAS syncword` (H.264 from the same file decodes). For the widest format coverage, build from the workspace. For a library, pin exact versions that you have tested together.

## CLI: `oxideav`

**Install:**
- Prebuilt tarballs are on https://github.com/OxideAV/oxideav-workspace/releases (latest is still `v0.0.5`, 2026-05-27, for linux-x86_64, macos-universal and windows-x86_64). They bundle `oxideav` and `oxideplay`. The release is much older than the code: it lacks e.g. an AAC decoder.
- Otherwise build it: `cargo build --release -p oxideav-cli` in the workspace. The crate has `publish = false`, so there is no `cargo install`.
- The engine behind `oxideav convert` is published separately as `oxideav-cli-convert` (crates.io `0.0.7`), which now takes `-threads N`.

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

The following commands were checked against a workspace build at tip (2026-09-28; the failures listed under gotchas were re-tested at `d02173e`, 2026-10-04):

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

Re-tested at workspace tip `d02173e` (2026-10-04) with small ffmpeg-generated files. Re-test before relying on any of them.
- **Extracting frames from video to images does not work from the CLI.** It now fails with an error instead of silently writing nothing:
  - `transcode in.mp4 frame-%03d.png` fails with `PNG muxer: codec_id must be png (got h264)` (or `ffv1`/`mjpeg`).
  - `convert in.mp4 frame-%d.png` fails with "does not declare the pixel layout of video stream #0"; a single `out.png` fails with `PNG encoder: stride 160 is shorter than the 640-byte row`.
  - `transcode ... --codec-video png` into `.mkv` no longer panics, but fails with the same stride error.
  - Use the [library route](#extract-frames-to-png) instead; it works.
- **H.264 input can decode zero frames in the CLI.** A 160x120 baseline H.264 file reported "10 pkts in, 0 frames decoded" when transcoded with `--codec-video png` or `ffv1`, although the same file decodes through the crates directly.
- **AAC inside MP4/MKV does not decode** (`.m4a`, `.mka`, `.mp4`, `.mkv` → wav/flac): `oxideav-aac: packet has neither an ADTS nor a LOAS syncword`. So `--codec-audio X` on a typical h264+aac MP4 fails. Remux (copy) of the same file works.
- **Encoding AAC into MP4/M4A fails:** `mp4 muxer: aac stream missing extradata (AudioSpecificConfig)`.
- **Other encoders reject inputs:**
  - Opus only accepts 48 kHz: `unsupported input sample rate 44100 Hz (48000 required)`; resample first (see the `run` example above).
  - VP9 into `.mkv`/`.webm` fails with `vp9 encoder: pixel_format is required`.
  - There is no MP3 output: `transcode in.wav out.mp3` fails with `format not found: mp3`, with or without `--codec-audio mp3`.
- **WAV output accepts exactly one audio stream,** and an A/V input is refused ("WAV supports exactly one audio stream") rather than having its video dropped. Use a `run` job with only an `audio` track.
- `generate://testsrc` is rejected by `transcode`/`probe`; use it from an `oxideav run` JSON job.

## Player: `oxideplay`

- Usage: `oxideplay <file-or-url>`, or `oxideplay --job job.json`, where `@display` / `@out` bind to the player.
- Video output:
  - SDL2 is the default, and `libSDL2` is loaded at runtime. There is no build-time link; if SDL2 is missing the player exits cleanly.
  - `--vo winit` uses winit+wgpu and adds an egui overlay.
- Audio goes through `oxideav-sysaudio`, which runtime-loads ALSA, PulseAudio, WASAPI, CoreAudio or OSS.
- Keys: `q` quit, `space` pause, `←/→` ±10 s, `↑/↓` ±1 min, `PgUp/PgDn` ±10 min, `*` and `/` for volume.
- 3D model viewer (new in git, not in the v0.0.5 release): opening a 3D file shows it interactively. Built with the `viewer-gpu` feature (winit output) it draws on the GPU through `oxideav-render-vulkan`; otherwise it uses the `oxideav-render` software backends, including the path tracer.

## Library usage

The examples below compiled and ran against crates.io `oxideav-core 0.1.37`, `oxideav-flac 0.0.11`, `oxideav-basic 0.0.10`, `oxideav-pipeline 0.1.12`, `oxideav-source 0.1.5`, `oxideav-mkv 0.0.10` and `oxideav-h264 0.1.8`. The frame-extraction and still-image examples were re-verified on 2026-10-04 with `oxideav-png 0.1.11` and `oxideav-pixfmt 0.1.9`.

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

This decodes H.264 from MKV, converts each frame to RGBA and encodes one PNG per frame through the codec registry. Build it with `--release`: on a 5-frame 160x120 clip the release build finished instantly, while an unoptimised debug build had not finished after 5 minutes.

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

### Still images without the framework

Since 2026-10-03 every image crate (`oxideav-png`, `-mjpeg`, `-webp`, `-gif`, `-bmp`, `-tiff`, `-heif`, `-avif`, `-qoi`, `-tga`, ...) follows one API contract (`IMAGE_CRATE_API.md` in oxideav-workspace). The same root functions exist in each crate and work with `default-features = false`, without `oxideav-core`: `probe`, `info`, `decode` (native layout), `decode_rgb8` / `decode_rgba8`, `decode_all` (animation frames), `encode`, `encode_rgb8` / `encode_rgba8`, `encode_to`, and an `EncodeOptions` builder. Older per-format names such as `decode_png_to_rgba` or `encode_png_image` are deprecated aliases. Releases on the new contract include `oxideav-png 0.1.11`, `oxideav-bmp 0.1.7` and `oxideav-webp 0.3.0`. A framework-side gateway crate, `oxideav-image`, is described in the contract but does not exist yet.

```rust
// oxideav-png = { version = "0.1.11", default-features = false }
let bytes = std::fs::read("in.png")?;
if oxideav_png::probe(&bytes) {
    let info = oxideav_png::info(&bytes)?;            // header only: width, height, format, frames
    let img = oxideav_png::decode(&bytes)?;           // PngImage, native layout
    let rgba: Vec<u8> = img.to_rgba8();               // tightly packed, 4 * width bytes per row
    let (w, h) = (img.width(), img.height());
    let opts = oxideav_png::EncodeOptions::default().with_level(2);
    let out: Vec<u8> = oxideav_png::encode_rgba8(w, h, &rgba, &opts)?;
    std::fs::write("out.png", out)?;
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
- Per-crate README snippets are often stale: as of 2026-10-04, 68 per-crate READMEs still call `codecs.make_decoder(...)` or `containers.open(...)`, which are **not** methods on the current core registries. Use `CodecRegistry::first_decoder` / `first_encoder` and `ContainerRegistry::open_demuxer`. `make_decoder` only exists as the free function `oxideav_pipeline::make_decoder(&reg, &params)` and as each codec crate's own `decoder::make_decoder(&params)` factory.
- When a hardware bridge also claims a codec id, `first_decoder` may choose it. Pin the software implementation with `decoder_by_impl("<id>_sw", &params)`, or use `CodecPreferences { no_hardware: true, .. }`.
- The PCM codec ids are `pcm_s16le`, `pcm_f32le`, etc. (from `oxideav-basic`). FLAC decode outputs `U8`, `S16`, `S24` or `S32` depending on the stream's bit depth.
- Repository descriptions on GitHub are often outdated. For example, "H.264 decoder: I-slice only" is wrong; H.264 now decodes and encodes. The workspace README's status tables are the current source, and `oxideav list` / `oxideav info <codec>` are authoritative for any given build.

## Format support

The status column is condensed from the oxideav-workspace README current-status tables (2026-09-28); crates.io versions were refreshed on 2026-10-04 for the crates that had new releases. ✅ means working end-to-end, with a percentage where the README gives one. 🚧 means partial or scaffold. — means not implemented. "crates.io" is the latest published version; "git" means yanked or unpublished. "meta" is the feature name in `oxideav-meta` (✗ means you must register the crate manually).

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
| oxideav-h265 | H.265/HEVC | ✅ ~99% | 🟢 ~90% | 0.0.14 | h265 |
| oxideav-h266 | H.266/VVC | ✅ 100% | 🚧 ~95% | 0.0.9 | h266 |
| oxideav-av1 | AV1 | ✅ 100% | 🟢 ~99% | 0.1.20 | av1 |
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
| oxideav-png | PNG + APNG | ✅ 100% | 0.1.11 | png |
| oxideav-gif | GIF 87a/89a + animation | ✅ 100% | 0.0.11 | gif |
| oxideav-webp | WebP lossy/lossless/animated | ✅ 100% | 0.3.0 | webp |
| oxideav-mjpeg | JPEG (baseline/progressive/hierarchical) | ✅ ~95% dec / ~90% enc | 0.1.9 | mjpeg |
| oxideav-tiff | TIFF 6.0 + BigTIFF, CCITT, JPEG-in-TIFF | ✅ 100% / ~97% | 0.0.6 | ✗ |
| oxideav-bmp / -ico | BMP / ICO, CUR, ANI | ✅ ~97–98% | 0.1.7 / 0.0.7 | ✗ |
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

These crates use `oxideav-mesh3d`'s `Mesh3DRegistry` rather than the codec registry. Populate it with `oxideav_meta::populate_mesh3d_registry` (meta features `stl`, `obj`, `gltf`, `usdz`, `fbx`, `ifc`, `vrml`, `x3d`, `step`, all in the `3d` preset), or call each crate's `register` (`register_mesh3d` for `oxideav-ifc` and `oxideav-step`). `oxideav-step` is git-only: crates.io `0.0.0` is an empty placeholder, so use the `[patch.crates-io]` entry from [Depending on it](#depending-on-it). `oxideav-step`, `-vrml` and `-x3d` also build without `oxideav-core` when you set `default-features = false`; `oxideav-step` then exposes only `read_step` and `StepModel`.

`oxideav-render-vulkan` (MSRV 1.87, wgpu 29) is not wired into meta. `GpuRenderer::new()` returns `Error::Backend` when there is no usable GPU adapter, so fall back to `oxideav-render`. `register_into(&mut RenderRegistry)` adds the backends `"gpu"` and `"gpu-pathtrace"`.

```rust
// STEP, std-only model (works with default-features = false)
let bytes = std::fs::read("part.stp")?;
let model = oxideav_step::read_step(&bytes)?;
for part in &model.parts { println!("{:?}: {} shapes", part.name, part.shapes.len()); }

// Any of the three through the registry (default `registry` feature)
let mut reg = oxideav_mesh3d::Mesh3DRegistry::new();
oxideav_step::register_mesh3d(&mut reg);
oxideav_vrml::register(&mut reg);
oxideav_x3d::register(&mut reg);
let scene = reg.decoder_for_extension("stp").unwrap().decode(&bytes)?;
```

| Crate | Format | Decode | Encode | crates.io |
|---|---|---|---|---|
| oxideav-stl | STL ASCII/binary | ✅ ~99% | ✅ ~99% | 0.0.4 |
| oxideav-obj | OBJ + MTL | ✅ ~99% | ✅ ~99% | 0.0.4 |
| oxideav-gltf | glTF 2.0 / .glb | ✅ ~98% | ✅ ~95% | 0.0.4 |
| oxideav-usdz | USDZ/USDA | ✅ ~97% | ✅ ~92% | 0.0.4 |
| oxideav-fbx | FBX | 🚧 ~95% | ✅ ~95% | 0.0.3 |
| oxideav-ifc | IFC (BIM, STEP) | ✅ | — | 0.0.3 |
| oxideav-step | STEP CAD (ISO 10303-21; AP242/AP214/AP203 B-rep + AP242 tessellated), `.step`/`.stp`/`.p21` | ✅ | ✅ AP242 tessellated only | git (0.0.0 is a placeholder) |
| oxideav-vrml | VRML97 (ISO/IEC 14772), `.wrl`/`.wrz` | ✅ | ✅ | 0.0.1 |
| oxideav-x3d | X3D 4.0 (ISO/IEC 19775-1): XML `.x3d`/`.x3dz`, ClassicVRML `.x3dv`/`.x3dvz`, JSON `.x3dj` | ✅ | ✅ (no JSON write) | 0.0.1 |
| oxideav-render | Scene3D → raster (scanline / Whitted raycast / path tracer) | ✅ | | 0.0.5 |
| oxideav-render-vulkan | Scene3D → raster on the GPU via wgpu (Vulkan/Metal/DX12/GL, loaded at runtime): PBR rasteriser + path tracer | ✅ | | 0.0.1 |

### Filters, conversion, text, sources, output

| Crate | Role | Status | crates.io | meta |
|---|---|---|---|---|
| oxideav-pixfmt | 70 pixel formats, full conversion matrix, palette generation, dithering (`convert(&frame, FrameInfo, dst, &ConvertOptions)`) | ✅ | 0.1.9 | (dependency) |
| oxideav-image-filter | 136 filter types (resize, blur, edge, crop, rotate, ...), JSON names `video.*` | ✅ | 0.1.2 | image-filter |
| oxideav-audio-filter | about 50 filters (volume, resample, echo, EQ, compressor, reverb, loudness, spectrogram, ...) | ✅ | 0.1.2 | audio-filter |
| oxideav-source | `SourceRegistry` drivers: file, mem, data:, slice, concat; `BufferedSource` prefetch | ✅ | 0.1.5 | source |
| oxideav-http | `http(s)://` via ureq + rustls, Range seek | ✅ | 0.0.8 | http |
| oxideav-rtmp | RTMP server/client, `rtmp://` packet source, Enhanced-RTMP | ✅ (no RTMPS) | 0.0.6 | ✗ (`register(&mut ctx.sources)`) |
| oxideav-generator | `generate://` synthetic audio/image/video, `xc:`/`gradient:`/`label:` | ✅ | 0.1.4 | generator |
| oxideav-sysaudio | native audio output (ALSA/Pulse/WASAPI/CoreAudio/OSS, runtime-loaded) | ✅ | 0.1.1 | ✗ |
| oxideav-ttf / -otf / -scribe / -raster | TrueType/OpenType parsing, shaping (GSUB/GPOS, bidi), vector→raster | ✅ | 0.1.8 / 0.1.4 / 0.1.10 / 0.1.3 | ✗ |
| oxideav-scene | time-based scene model (PDF pages, compositor, NLE) | 🚧 | 0.1.4 | ✗ |
| oxideav-mesh3d | typed Scene3D model + registry | ✅ | 0.0.7 | mesh3d |
| oxideav-bitstream | H.264/HEVC/AV1 header parse helpers for HW bridges | ✅ | 0.0.2 | ✗ |
| oxideav-io | open/probe/save/transcode facade | ✅ still images; A/V transcode pending | 0.1.0 | ✗ |
| oxideav-cli-convert | engine behind `oxideav convert` (`-threads N`) | ✅ | 0.0.7 | ✗ |
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
