# Compression, Encodings & Data Formats

Pure-Rust codecs and format libraries with little or no dependency baggage. For compression there are two current crates, each for a different job. [`compcol`](#compcol) is the full-featured one: 40+ algorithms (deflate/gzip/zlib, zstd, brotli, xz/LZMA, bzip2, LZ4, RAR/StuffIt/LHA decoders, ...) behind one streaming trait, `no_std` + `alloc`, zero deps. Its `embed` mode builds the deflate family (gzip/zlib/raw deflate) for microcontrollers, with no heap, no panics and a few KB of code; it absorbed the former [`minizlib`](#minizlib) crate, which is now deprecated. [`minlz`](#minlz-rs) is the pick when you need byte-for-byte compatibility with Go's `klauspost/compress/s2` (Snappy-family) or MinIO's MinLZ, or very fast decoding. For data formats, [`emjson`](#emjson) fills the embedded JSON niche: it streams, parses, writes and edits in place in a few hundred bytes of RAM with no allocator, while `serde_json` needs a heap. [`tomlproc`](#tomlproc) is a zero-dependency TOML 1.1 parser/serializer. [`charcode`](#charcode) converts text encodings per WHATWG with no deps and no `unsafe`, [`anyd`](#anydcode) encodes and decodes 1D/2D barcodes, [`xuid-rs`](#xuid-rs) provides type-prefixed base32 UUIDs, and [`hdf5rs`](#hdf5rs) reads and writes HDF5 scientific data files without `libhdf5`, using `compcol` for its compression filters.

**Choosing a compressor:**
- **compcol** (standard build): you need a format other than deflate, a format picked at runtime (factory/magic-byte detection), `std::io`/tokio adapters, decompression of legacy archive codecs, or good gzip ratio (lazy matching + dynamic Huffman). Needs `alloc`.
- **compcol with `embed`**: gzip/zlib/deflate only, on a target with no heap and tight flash/RAM (MCU firmware, bootloaders, OTA images). It has a lower ratio when compressing (fixed Huffman only) and decodes at tens of MB/s. `compcol::embed::flate` keeps the old `minizlib` API. Do not use `minizlib` in new code.
- **minlz**: you exchange data with Go services that use S2/Snappy or MinLZ, or you need a very fast decoder (tens of GiB/s). Uses `crc` as a dependency and contains a few audited `unsafe` blocks.

## Quick pick
| Need | Use |
|------|-----|
| gzip/zlib/zstd/brotli/xz/bzip2/lz4 etc. through one API, `no_std` + alloc | [`compcol`](#compcol) |
| Choose a codec by name at runtime / sniff a compressed stream's format | [`compcol`](#compcol) (`factory`) |
| Decompress RAR 2/3/5, StuffIt, LHA, LZX/CAB, Quantum, PPMd; encode/decode LZFSE, LZF | [`compcol`](#compcol) |
| zlib-style tuning: levels, strategies, preset dictionaries, partial flush | [`compcol`](#compcol) (`deflate`/`zlib`/`gzip`) |
| HTTP/2 HPACK or HTTP/3 QPACK header compression | [`compcol`](#compcol) |
| gunzip/gzip on a microcontroller with no heap, a few KB of flash | [`compcol`](#compcol) (`embed` feature) |
| Stream-decompress an OTA/firmware image from UART to flash | [`compcol`](#compcol) (`embed::flate`) |
| S2 / Snappy wire-compatible with Go `klauspost/compress/s2` | [`minlz`](#minlz-rs) |
| MinLZ (`.mz`, minio/minlz) streams, seekable indexes | [`minlz`](#minlz-rs) |
| Parse / edit JSON on an MCU with no allocator, or a huge JSON file in constant RAM | [`emjson`](#emjson) |
| Parse/write TOML config with zero dependencies (optional serde) | [`tomlproc`](#tomlproc) |
| Convert Shift_JIS / Big5 / windows-1252 / EBCDIC ... to/from UTF-8 | [`charcode`](#charcode) |
| Generate or read QR / Data Matrix / EAN / Code 128 / PDF417 barcodes | [`anyd`](#anydcode) |
| Type-prefixed, sortable, base32 IDs compatible with Go `xuid` | [`xuid-rs`](#xuid-rs) |
| Read/write HDF5 (`.h5`) files without libhdf5: hyperslabs, compounds, chunked + compressed datasets | [`hdf5rs`](#hdf5rs) |

## compcol

**Repo:** https://github.com/KarpelesLab/compcol · **Crate:** `compcol` (crates.io `0.8.0`) · **License:** MIT · **Status:** usable (1,700+ tests, cross-validated against reference tools; fuzz targets)

A collection of compression algorithms in pure Rust behind one caller-buffered streaming trait (`Encoder`/`Decoder`/`Algorithm`). It is `#![no_std]` and `#![forbid(unsafe_code)]`, has zero runtime dependencies (tokio only with the `tokio` feature), and gates each algorithm behind its own cargo feature. MSRV 1.95, edition 2024.

**Use it when:** you need any mainstream or legacy compression format, want the format chosen from config or a CLI flag, need `Read`/`Write` or tokio `AsyncRead`/`AsyncWrite` adapters, or have to decompress untrusted input with an output cap.
**Don't use it when / limits:** in a no-allocator build only `rle` and the `embed` deflate family are available. zstd and brotli encoders produce conformant streams but lag the reference ratio. Some codecs are decode-only (`Error::Unsupported` on encode): RAR 1-5 (license), Quantum, PPMd, StuffIt methods. (LZFSE encodes since 0.7.7.) RAR1 does not decode either. LZX/Amiga LZX encoders emit uncompressed blocks only. This crate handles codec streams, not archive containers (no zip/rar/7z directory parsing).

**Add it:**
```toml
[dependencies]
compcol = "0.8"                                        # default: alloc, rle, deflate, zlib, gzip, factory
# compcol = { version = "0.8", features = ["std", "zstd", "brotli", "xz"] }
# compcol = { version = "0.8", features = ["all"] }    # every algorithm
# firmware, no heap: compcol = { version = "0.8", default-features = false, features = ["embed", "gzip"] }
```

**Key features / cargo features:**
- Capability features: `alloc`, `std` (`compcol::io`: `EncoderWriter`, `EncoderReader`, `DecoderWriter`, `DecoderReader`), `tokio` (same four names in `compcol::tokio_io`), `factory` (`encoder_by_name`, `decoder_by_name`, `names`, `extension`, `detect`), `checksum` (public Adler-32/CRC-32).
- Algorithm features: `deflate`, `deflate64`, `zlib`, `gzip`, `lzma`, `xz`, `lzma2`, `zstd`, `brotli`, `lz4`, `lz5`, `lzf` (liblzf raw format), `snappy`, `lzw`, `lzss`, `bzip2`, `lzo`, `lzx`, `amiga_lzx`, `quantum`, `lzfse`, `adc`, `ppmd`, `xpress`, `xpress_huffman`, `lznt1`, `lzham`, `lzs`, `packbits`, `lha`, `zip_implode`/`zip_shrink`/`zip_reduce`, `arc_*`, `sit13`/`lzah`/`arsenic`, `rar1`-`rar5`, filters `bcj`/`bcj2`/`delta`, primitives `huffman`/`rangecoder`/`mtf`/`bwt`, and `hpack`/`qpack`. The `all` feature enables everything.
- One-shot helpers in `compcol::vec`: `compress_to_vec`, `decompress_to_vec`, `*_with(config)`, and `decompress_to_vec_capped` (bomb-safe, returns `Error::OutputLimitExceeded`). `compcol::limit::LimitedDecoder` wraps any decoder with a cap.
- `Decoder::discard_output(input, n)` advances through decompressed output without emitting it.
- 0.8: `Decoder::set_output_limit` (`LimitedDecoder` passes its cap down, so buffering decoders such as snappy, huffman and bwt refuse before allocating), `ContiguousDecoder` (lzma, xz, lzma2, zstd decode into one caller buffer that doubles as the history) via `factory::contiguous_decoder_by_name`, streaming rar2/rar3, multi-frame zstd/lz4. CLI feature `mmap` (memmap2, binary only) decodes xz/zstd straight into the output file. 0.7.9 was a security-audit release (hangs, limit bypasses), so stay on 0.7.9+ / 0.8.
- zlib-level control for the deflate family (0.7.8): levels 0-9 (0 = stored), `deflate::Strategy` (`Default`, `Filtered`, `HuffmanOnly`, `Rle`, `Fixed`), memory level and window bits through `deflate::EncoderConfig` and `Encoder::with_deflate_config`, preset dictionaries (`zlib::Encoder::with_dictionary`; the decoder returns `Error::NeedDictionary` until you call `set_dictionary`), and partial/block flushes.
- Other recent additions: zstd accepts libzstd's negative "fast" levels and a configurable window limit (0.7.4); xz decodes concatenated streams and Delta/BCJ filter chains and verifies CRC64/SHA-256 checks (0.7.4); the bzip2 encoder output is byte-identical to libbzip2; an LZFSE encoder (0.7.7); and ZIP method 14 (LZMA) and 98 (PPMd variant I) entry codecs (0.7.8).
- `embed` is a mode, not an algorithm: it rebuilds the deflate family (`gzip`, `zlib`, `deflate`) as heap-free variants under the same paths and traits. It adds one-shot `gzip::decompress`, `gzip::decompressed_len` and `gzip::compress` (caller-provided table), `const`-constructible streaming codecs that can live in a `static` (`gzip::Decoder` with a 32 KiB window, `gzip::WindowedDecoder::<N>`, `gzip::Encoder`, `gzip::BlockEncoder::<B, T>`), and `compcol::embed::flate`, the former `minizlib` API (`gunzip`, `Reader`, `Stream`, `Decompressor`, `Compressor`). CI holds gzip streaming decode to about 4 KB of code and 352 B of stack on Cortex-M4.
- `compcol` CLI binary (`cargo install compcol --features all`), a gzip(1)-style filter with `-t ALGO`, `-d` and auto-detection.

**Example:**
```rust
use compcol::gzip::Gzip;
use compcol::vec::{compress_to_vec, compress_to_vec_with, decompress_to_vec_capped};

let plain = b"hello world hello world hello world";
let gz = compress_to_vec::<Gzip>(plain)?;
let gz9 = compress_to_vec_with::<Gzip>(plain, compcol::gzip::EncoderConfig { level: 9 })?;
let back = decompress_to_vec_capped::<Gzip>(&gz, 1 << 20)?; // refuse >1 MiB output
assert_eq!(back, plain);

// Runtime selection (feature "factory"):
assert_eq!(compcol::factory::detect(&gz), Some("gzip"));
let mut dec = compcol::factory::decoder_by_name("gzip").expect("compiled in");
# let _ = (gz9, &mut dec);
# Ok::<(), compcol::Error>(())
```

**Gotchas:**
- The trait methods return `Result<(Progress, Status), Error>`, a tuple. Loop until `Status::StreamEnd` on `finish()`.
- The README's install snippets say `version = "0.6"`, but crates.io is at 0.8.0. Use `0.8`.
- 0.8 added a provided method to `Decoder` (`set_output_limit`); custom `Decoder` implementations keep compiling, but code that matched on 0.7 trait shapes may need updating.
- Cargo features unify, so enabling `embed` anywhere in a dependency graph switches every user of compcol in that graph to the small deflate variants (lower ratio, no sync flush, no preset dictionaries). Enable it only in firmware binaries, never in a library. `--all-features` turns it on; `all` does not. Note that `outscript` 0.3's default `bbqr` feature enables compcol `embed`, so pulling in outscript with default features switches every compcol 0.8 user in the build (for example purecrypto's `cert-compression`) to the small variants; use `outscript` with `default-features = false` and without `bbqr` if that matters.
- `EncoderWriter` finishes on `Drop` on a best-effort basis. Call `.finish()` explicitly to see errors.
- `factory::detect` does not detect brotli or raw `.lzma` (they have no magic bytes). It only reports codecs that were compiled in.
- `decompress_to_vec` is unbounded. Use the `_capped` variants for untrusted input.

## minizlib

**Repo:** https://github.com/KarpelesLab/minizlib · **Crate:** `minizlib` (crates.io `0.1.2`) · **License:** MIT · **Status:** **deprecated**, superseded by `compcol::embed::flate`

The tiny `no_std`, no-alloc, no-panic gzip/zlib/deflate codec for firmware. Its code now lives in [`compcol`](#compcol) as `compcol::embed::flate`, with the same API. From 0.1.2 this crate only re-exports that module and gets no further work. Existing code keeps building. The `checksum`, `crc-table`, `concat`, `stored`, `fixed` and `dynamic` features are kept so existing feature lists still build, but they no longer change anything: checksums are always verified, and concatenated members and all block types are always accepted.

**Use it when:** never in new code. Use compcol's `embed` feature instead.

**Migrate:**
```toml
[dependencies]
compcol = { version = "0.8", default-features = false, features = ["embed", "gzip"] }
```
Then replace `use minizlib::*` with `use compcol::embed::flate::*`. Function and type names (`gunzip`, `gzip`, `Buffer`, `Reader`, `Stream`, `Decompressor`, `Compressor`, `Error`) are unchanged.

## minlz-rs

**Repo:** https://github.com/KarpelesLab/minlz-rs · **Crate:** `minlz` (crates.io `1.2.3`) · **License:** BSD-3-Clause · **Status:** production (1.x, Go byte-compat tests, proptest, libfuzzer, Miri)

It implements two Snappy-family codecs. **S2** has byte-for-byte identical output to Go `klauspost/compress/s2` in all four modes and decodes Snappy. **MinLZ** is the minio/minlz spec v1.0 `.mz` format; it decodes S2/Snappy, but its output cannot be read by them. Both come with block and stream formats (CRC32C framing), seek indexes and dictionaries. The block API is `no_std` + `alloc`. Decode is very fast (6-27x Go's asm, up to about 135 GiB/s in L1). Uses the `crc` crate and a few documented `unsafe` blocks in hot paths. MSRV 1.81.

**Use it when:** you interoperate with Go services or files using S2/Snappy/MinLZ, or you need the fastest possible decompression (caches, logs, RPC payloads) at a Snappy-like ratio.
**Don't use it when / limits:** you need gzip/zstd-class ratios (use `compcol`), or no allocator (use `compcol` with `embed`). The standard S2 encoder is 2-4x slower than Go's assembly. The MinLZ `Dict` format is crate-local and not interoperable with minio/minlz. The streaming API needs `std`.

**Add it:**
```toml
[dependencies]
minlz = "1"                          # default: std, s2, minlz
# minlz = { version = "1", default-features = false }                          # no_std + alloc, block APIs only
# minlz = { version = "1", features = ["concurrent"] }                         # rayon ConcurrentWriter
```

**Key features / cargo features:**
- `std` (streaming `Reader`/`Writer`/`ConcurrentWriter`), `s2`, `minlz`, `concurrent` (rayon), `cli` (`s2c`/`s2d`/`mzc`/`mzd` binaries: `cargo install minlz --features cli`).
- S2 (`minlz::s2`): `encode`, `encode_better`, `encode_best`, `encode_snappy`, `decode`, stateful `Encoder` (reuses hash tables), `make_dict`/`encode_with_dict`/`decode_with_dict`, `Index`.
- MinLZ (`minlz::minlz`): `compress`, `compress_level(Level::{Fastest,Balanced,Smallest})`, `decompress`, `decompress_into`, `decompressed_len`, `Writer` (`.with_index()`), `Reader`, `Index::load` + `seek_decompress`.

**Example:**
```rust
use std::io::{Read, Write};

// S2 block (Go s2.Encode-compatible)
let c = minlz::s2::encode(b"hello hello hello");
assert_eq!(minlz::s2::decode(&c).unwrap(), b"hello hello hello");

// MinLZ stream (.mz, CRC32C-checked)
let mut w = minlz::minlz::Writer::new(Vec::new());
w.write_all(b"hello hello hello").unwrap();
let stream = w.finish().unwrap();
let mut out = Vec::new();
minlz::minlz::Reader::new(&stream[..]).read_to_end(&mut out).unwrap();
```

**Gotchas:**
- The crate is named `minlz`, but the root re-exports (`minlz::encode`, `minlz::Reader`, ...) are **S2**, kept for backward compatibility. Write `minlz::s2::` or `minlz::minlz::` explicitly.
- S2 output cannot be read by plain Snappy decoders. Use `encode_snappy` if the consumer only speaks Snappy.
- For best throughput build with `RUSTFLAGS="-C target-cpu=native"`.

## emjson

**Repo:** https://github.com/KarpelesLab/emjson · **Crate:** `emjson` (crates.io `0.1.1`) · **License:** MIT · **Status:** usable (early 0.1; differential-tested against serde_json)

A streaming JSON parser, writer and **in-place editor** for embedded systems. It is `no_std`, uses no allocator and no `unsafe`, and has no required dependencies. The parser is 20 bytes plus a caller-sized buffer (16 bytes works), and nesting costs 1 bit per level. It reads from slices, from any stream, or from random-access storage (flash, SD, files). It can seek to a path, give the exact byte span of a value, and replace/insert/remove values in place, moving the tail of the document once. Flash cost is about 9-24 KB depending on the API used. Strict RFC 8259. MSRV 1.89.

**Use it when:** you parse or patch JSON config/state on a microcontroller, edit a multi-megabyte JSON file with a 512-byte buffer, extract one value from a large stream without materializing it, or serialize structs into a fixed buffer (`ToJson`, `encoded_len`).
**Don't use it when / limits:** you have a heap and want derive-based (de)serialization into Rust types (use `serde_json`). There is no DOM or `Value` tree and no serde integration. `ToJson` is implemented by hand. `f32` parse/format pulls in about 32 KB of `core` float code.

**Add it:**
```toml
[dependencies]
emjson = "0.1"
# emjson = { version = "0.1", features = ["std"] }          # std::io adapters + Storage for std::fs::File
# emjson = { version = "0.1", features = ["embedded-io"] }  # embedded-io 0.7 Read/Write adapters
```

**Key features / cargo features:**
- No default features. `std` enables `io::StdIo` and `Storage` for `File`. `embedded-io` enables `io::EmbeddedIo`.
- `Parser`: `seek(&["a","b"])` or `seek("/a/b")` (JSON Pointer), `peek`, `offset`, `read_str`, `read_num`, `read_bool`, `str_reader` (chunked), `value_span`, `skip_value`, cursor API (`begin_object`/`has_next`/`read_key`/`find_key`/`match_key`), pull events (`next_event`), `walk` callbacks with paths (`Flow::Skip`). Zero-copy `read_str_ref`/`raw_value` for slices.
- Writer: `JsonWriter`, `ToJson`, `to_slice`, `encoded_len`, `copy_value` (minify/pretty-print/extract in constant memory).
- Editing: `edit::Editor` over `Storage` (`MemStorage` or your own 4-method impl) with `replace`/`set`/`insert`/`push`/`remove`; `copy_edit`, `plan`/`plan_remove`/`apply_copy` for read-only sources.

**Example:**
```rust
use emjson::{Parser, Token};
use emjson::io::ReadSource;
use emjson::edit::{Editor, MemStorage};

// Seek into a stream with a 16-byte buffer
let stream: &[u8] = br#"{"foo": {"bar": "hello world"}}"#;
let mut buf = [0u8; 16];
let mut p = Parser::new(ReadSource::new(stream, &mut buf));
assert!(p.seek("/foo/bar").unwrap());
assert_eq!(p.peek().unwrap(), Token::String);
let mut s = [0u8; 32];
assert_eq!(p.read_str(&mut s).unwrap(), "hello world");

// Edit in place
let mut doc = [0u8; 128];
let src = br#"{"list": [1, 2]}"#;
doc[..src.len()].copy_from_slice(src);
let mut scratch = [0u8; 32];
let mut ed = Editor::new(MemStorage::new(&mut doc, src.len()), &mut scratch);
ed.push("/list", &3).unwrap();
ed.remove("/list/0").unwrap();
```

**Gotchas:**
- `Editor::set` inserts new members in sorted position by default. Call `set_placement(Placement::End)` to append instead.
- Unread values in the cursor API are skipped automatically. Strings longer than your buffer need `str_reader()`.
- Single-pass `copy_edit` appends new members at the end and cannot remove. Removal takes two passes (`plan_remove` + `apply_copy`).

## tomlproc

**Repo:** https://github.com/KarpelesLab/tomlproc · **Crate:** `tomlproc` (crates.io `0.1.2`) · **License:** MIT · **Status:** usable (full differential agreement with the `toml` crate on 401,684 inputs)

A full TOML 1.1.0 parser and serializer with **no dependencies**: no build scripts, no proc macros, no `unsafe`. It is `#![no_std]` with `alloc` (CI builds for thumbv7em and riscv32imc). Documents parse into an insertion-ordered `Table`/`Value` tree with type-strict accessors. Optional serde support maps documents onto your own types. MSRV 1.88.

**Use it when:** you read or write config files and want no dependency tree or proc-macro compile cost, need TOML 1.1 syntax (multi-line inline tables, `\e`/`\xHH`, optional seconds), need line/column errors or value spans for diagnostics, or run on `no_std` with a heap.
**Don't use it when / limits:** you need format-preserving edits (comments, layout and header-vs-inline choice are lost; values round-trip, text does not). For that, use `toml_edit`. There is no parser without an allocator (only `Datetime`/`Date`/`Time`/`Offset` work without `alloc`). The serializer emits TOML 1.0 syntax (times are normalized to `HH:MM:SS`).

**Add it:**
```toml
[dependencies]
tomlproc = "0.1"
# tomlproc = { version = "0.1", features = ["serde"] }
# tomlproc = { version = "0.1", default-features = false, features = ["alloc"] }  # no_std + alloc
```

**Key features / cargo features:**
- `std` (default; hash-indexed tables), `alloc` (value model, parser, serializer; ordered maps without std), `serde` (adds the only dependency).
- `parse`, `parse_bytes`, `parse_spans` (per-key line/column/byte ranges), `to_string`, `to_string_pretty`; `Table::insert`, `get`, `get_path("a.0.b")`; `as_str`/`as_integer`/`as_datetime` etc.
- `tomlproc::serde::{from_str, from_slice, from_value, from_table, to_value, to_string, to_string_pretty}`. Errors carry `key_path()`.
- Nesting is capped at 128, so deeply nested untrusted input cannot exhaust the stack.

**Example:**
```rust
let doc = tomlproc::parse(r#"
    title = "TOML Example"
    [[server]]
    ip = "10.0.0.1"
    ports = [8000, 8001]
"#)?;
assert_eq!(doc["title"].as_str(), Some("TOML Example"));
assert_eq!(doc["server"][0]["ports"][1].as_integer(), Some(8001));
assert_eq!(doc.get_path("server.0.ip").and_then(|v| v.as_str()), Some("10.0.0.1"));

let mut t = tomlproc::Table::new();
t.insert("name", "tomlproc");
assert_eq!(tomlproc::to_string(&t), "name = \"tomlproc\"\n");
# Ok::<(), tomlproc::Error>(())
```

**Gotchas:**
- Accessors are type-strict: `as_float` on an integer returns `None`.
- Indexing with `[]` panics on a missing key. Use `get`/`get_path` for `Option`.

## charcode

**Repo:** https://github.com/KarpelesLab/charcode · **Crate:** `charcode` (crates.io `0.1.5`) · **License:** MIT · **Status:** usable

Character encoding conversion implementing the WHATWG Encoding Standard (all 40 encodings, all 228 labels), plus 49 charsets from outside the standard: DOS/OEM, EBCDIC, Mac regional variants, UTF-32/UTF-7, ISO-2022-KR/CN/JP-2, SCSU, and real Big5/Shift_JIS/EUC-JP. It has no dependencies (serde is optional), is `#![forbid(unsafe_code)]`, and is `no_std`, working even without `alloc` via caller-supplied buffers. MSRV 1.88.

**Use it when:** you decode HTML/email/legacy files in Shift_JIS, Big5, GBK, EUC-KR, windows-125x and similar; you map Windows code page numbers; you want a dependency-free, `unsafe`-free alternative to `encoding_rs`; or you need an `iconv`-like CLI.
**Don't use it when / limits:** raw throughput on large documents matters most. `encoding_rs` is faster (SIMD + `unsafe`) and implements the same standard. Static tables range from about 4 KiB to about 750 KiB depending on features.

**Add it:**
```toml
[dependencies]
charcode = "0.1"                                           # default: std + whatwg
# charcode = { version = "0.1", default-features = false } # no alloc: buffer-to-buffer engine only
```

**Key features / cargo features:**
- `std` (default), `alloc`, `serde` (an encoding serializes as its name), `cli` (`cargo install charcode --features cli`), `translit` (`EncodeOptions::transliterate`, iconv `//TRANSLIT`, about 30 KiB).
- Tables: `whatwg` (default: `whatwg-aliases`, `single-byte`, `big5`, `euc-jp`, `euc-kr`, `gb18030`, `iso-2022-jp`, `shift-jis`), `iso-2022-kr`, `iso-2022-cn`, `iso-2022-jp-2`, `scsu`, `extras` (`dos`, `ebcdic`, `mac`, `misc`, `unicode-extras`).
- Lookups: `Encoding::for_label` (the charset the label actually names), `Encoding::for_whatwg_label` (what the browser standard resolves it to), `for_windows_code_page`, `for_cp`.
- Error policies: `Malformed::{Fail, Omit, Replace}`, `Unmappable::{Fail, Omit, Replace, Html, JsonEscape}`. Streaming `Decoder`/`Encoder` (`decode_to_utf8`/`encode_from_utf8` into `&mut [u8]`, minimum buffer 4 bytes).

**Example:**
```rust
use charcode::{EncodeOptions, Encoding, Unmappable, EUC_KR, WINDOWS_1252};

let (text, _enc, tally) = WINDOWS_1252.decode(b"caf\xE9");
assert_eq!(text, "café");
assert!(tally.is_lossless());

let sjis = Encoding::for_whatwg_label(b"Shift-JIS").unwrap(); // -> windows-31j
assert_eq!(sjis.decode(b"\x93\xFA\x96{").0, "日本");

assert!(EUC_KR.encode("한국어 😀").is_err()); // encode fails on unmappable by default
let opts = EncodeOptions::new().unmappable(Unmappable::Replace('?'));
let (bytes, _, _) = EUC_KR.encode_with("한국어 😀", opts).unwrap();
assert_eq!(&bytes[..], b"\xC7\xD1\xB1\xB9\xBE\xEE ?");
```

**Gotchas:**
- `for_label(b"iso-8859-1")` returns true ISO-8859-1. `for_whatwg_label` returns windows-1252. Use the WHATWG lookup for content from the web.
- A BOM overrides the encoding you pass to `decode`. Decoding substitutes U+FFFD by default. Encoding **fails** by default on unmappable characters.
- UTF-16BE/LE and `replacement` are decode-only. Asking them to encode yields UTF-8, per the standard.
- In streaming mode, pass `last = true` on the final chunk so a truncated sequence gets reported.

## anydcode

**Repo:** https://github.com/KarpelesLab/anydcode · **Crate:** `anyd` (crates.io `0.3.0`) · **License:** MIT · **Status:** usable / actively developed (broad symbology coverage; image detection for a subset)

Barcode encoding and decoding written from scratch for about 40 1D, stacked, 2D and postal symbologies: QR/Micro QR/rMQR, Data Matrix (square, rectangular and DMRE sizes, every encodation, GS1 mode since 0.3; `build_rectangular`, `build_gs1`), Aztec, PDF417/MicroPDF417, MaxiCode, Han Xin, DotCode, EAN/UPC, Code 128/GS1-128, Code 39/93, ITF, Codabar, DataBar, postal 4-state, Apple App Clip Codes and more. It is always `#![no_std]`. It has no dependencies (only the `cli` feature pulls `oxideav-png`) and a WASM demo. A decoded `Symbol` keeps every encoding decision (segments, version, EC level, mask), so re-encoding reproduces the symbol byte-for-byte. It includes a camera/live-video pipeline (`detect::locate` then decode) on grayscale frames. MSRV 1.89.

**Use it when:** you generate barcodes (including heap-free `encode_into` on MCUs), decode a module grid or pattern, or scan barcodes from still images or video frames in pure Rust or WASM.
**Don't use it when / limits:** image detection is implemented only for QR, Data Matrix, Aztec, Micro QR, rMQR, PDF417/MicroPDF417, App Clip and the main 1D codes. MaxiCode, Han Xin, DotCode, Grid Matrix, DataBar, Pharmacode, postal and stacked 1D codes decode structurally but cannot be located in images. PDF417 sampling is affine-only. (On master, after 0.3.0, MaxiCode and GS1 DataBar non-Limited can also be located in images.) Image scanning needs `std`. Default features embed about 1.7 MB of App Clip tables, so disable `appclip` if you don't need it.

**Add it:**
```toml
[dependencies]
anyd = "0.3"
# lean no_std: anyd = { version = "0.3", default-features = false, features = ["alloc", "encode", "decode", "qr", "code128"] }
```

**Key features / cargo features:**
- Tiers: `std`, `alloc`, `encode`, `decode` (needs alloc), `scan` (image samplers, `detect`, `pipeline`, `scan1d`; needs std). All are on by default with `all-codes`.
- Families `matrix`/`stacked`/`linear`/`postal`, or per-symbology (`qr`, `datamatrix`, `aztec`, `pdf417`, `ean`, `code128`, `appclip`, ...). Call `Symbology::is_implemented()` at runtime.
- `cli` builds the `anyd` binary (encode to PNG/SVG/terminal, decode PNG). `wasm` builds the raw FFI shim for the browser demo.

**Example:**
```rust
use anyd::codes::qr::{EcLevel, QrDecoder, QrEncoder};
use anyd::traits::{Decode, Encode};

let encoder = QrEncoder::new();
let symbol = encoder.build_text("HELLO WORLD", EcLevel::Q).unwrap();
let encoding = encoder.encode(&symbol).unwrap();

let decoded = QrDecoder::new().decode(&encoding).unwrap();
assert_eq!(decoded.text().as_deref(), Some("HELLO WORLD"));
assert_eq!(encoder.encode(&decoded).unwrap(), encoding); // lossless round-trip
```

**Gotchas:**
- The repo is `anydcode`, but the crate and import name is `anyd`.
- The README's `no_std` snippet still says `version = "0.1"`; use 0.3.
- 0.3.0 renamed `datamatrix::SquareSpec` to `SymbolSpec` (and added `all_specs()`).
- The caller converts camera frames to a luminance `GrayFrame`. The crate has no media dependencies.
- For live video, run cheap `detect::locate` on every frame and decode only the crops (`pipeline::scan_linear_at` for linear regions along the reported axis, `pipeline::scan_all` for matrix codes).

## xuid-rs

**Repo:** https://github.com/KarpelesLab/xuid-rs · **Crate:** `xuid-rs`, imported as `xuid` (crates.io `0.1.0`) · **License:** BSD-3-Clause · **Status:** usable (small; cross-compat vectors with the Go library)

A Rust port of Go `KarpelesLab/xuid`. It provides a 128-bit UUID with a short type prefix, rendered in base32 as `prefix-aaaaaa-aaaa-aaaa-aaaa-aaaaaaaa` (36 characters or fewer, lowercase, case-insensitive on parse). The string form is byte-for-byte compatible with the Go implementation. It defaults to UUIDv7 (time-ordered) and also supports v4 and deterministic v5. Depends on `uuid` and `data-encoding`. Requires `std` (uses `String`). MSRV 1.70.

**Use it when:** you need human-readable, type-tagged, sortable IDs, especially where Go services using `xuid` share them, or you store IDs as TEXT in SQL (sqlx or rusqlite).
**Don't use it when / limits:** you need a raw 16-byte binary column (it is stored as TEXT) or `no_std`. Only the first 5 prefix characters are rendered.

**Add it:**
```toml
[dependencies]
xuid-rs = "0.1"                                     # import as `xuid`
# xuid-rs = { version = "0.1", features = ["serde", "sqlx"] }
```

**Key features / cargo features:** `serde` (serializes as a string), `sqlx` (any backend; Encode/Decode as TEXT), `rusqlite` (ToSql/FromSql). API: `Xuid::new(prefix)` (v7), `new_random` (v4), `from_uuid`, `from_key_prefix` (v5), `parse_prefix(s, expected)`, `FromStr` (also accepts plain UUID strings), `prefix()`, `uuid()`, `to_uuid_string()`, `Display`.

**Example:**
```rust
use xuid::Xuid;

let id = Xuid::new("user");                    // e.g. user-h4nu2n-zu3f-dmnn-kguv-6f643nei
let parsed: Xuid = "user-h4nu2n-zu3f-dmnn-kguv-6f643nei".parse().unwrap();
assert_eq!(parsed.prefix(), "user");
assert_eq!(parsed.to_uuid_string(), "3f1b4d37-34d9-46c6-b546-a57c5f736d22");
assert!(Xuid::parse_prefix("user-h4nu2n-zu3f-dmnn-kguv-6f643nei", "user").is_ok());
let a = Xuid::from_key_prefix("resource-name", "res"); // deterministic
# let _ = (id, a);
```

**Gotchas:**
- The crate name is `xuid-rs`, but `use xuid::...`. The crate's `repository` metadata points at the Go repo `KarpelesLab/xuid`.
- The prefix is clipped to 5 characters and lowercased only when rendering. The full prefix is retained, so two IDs that differ only in prefix case compare unequal but print the same.
- `from_key` always uses the `utref` prefix and a fixed namespace.

## hdf5rs

**Repo:** https://github.com/KarpelesLab/hdf5rs · **Crate:** `hdf5rs` (crates.io `0.0.1`, MSRV 1.95) · **License:** MIT · **Status:** pre-1.0, the API may still change. Reading is broad (all datatypes, every layout and chunk index, common filters, virtual datasets); writing covers the common cases. Files it writes are checked against h5py in CI.

A clean-room reader and writer for [HDF5](https://www.hdfgroup.org/solutions/hdf5/) files with no `libhdf5`, no C toolchain and `#![forbid(unsafe_code)]`. It has zero required dependencies; the compression filters come from [`compcol`](#compcol) behind cargo features. It reads files produced by the reference library, h5py and other tools, and writes files they can read. It can also modify existing files in place. In the KLB stack it is the field-dump I/O for the [`ems`](https://github.com/KarpelesLab/ems) electromagnetic solver.

**Use it when:**
- You need to read or write `.h5` / `.hdf5` files (scientific data, ML weights and datasets, simulation dumps) from Rust without linking `libhdf5`, including for static or cross-compiled builds.
- You need hyperslab reads, compound or variable-length types, attributes, or chunked and compressed datasets.

**Don't use it when / limits:**
- You need parallel HDF5 / MPI-IO, or writer support for non-default file drivers (family, split, direct, ROS3); these are out of scope.
- You need byte-for-byte identical output to the reference library.
- The blosclz codec inside Blosc is not supported. There is no `ndarray` integration or derive macro for compound types yet, and the parsing core is not `no_std`.

**Add it:**
```toml
[dependencies]
hdf5rs = "0.0.1"                                            # default filters: deflate, lzf
# hdf5rs = { version = "0.0.1", features = ["all-filters"] }  # + zstd, lz4, bzip2, blosc, bitshuffle
# hdf5rs = { version = "0.0.1", default-features = false }    # zero dependencies, no compression filters
```

**Key features / cargo features:**
- Filters: `deflate` (default, id 1), `lzf` (default, id 32000, common in h5py output), `zstd`, `lz4`, `bzip2`, `blosc`, `bitshuffle`, or `all-filters`. Shuffle, Fletcher-32, N-bit and scale-offset are always built in. `parallel` decodes the chunks of one read on several threads.
- Reading: `File::open` / `File::from_bytes`, `file.root()?.members()`, `file.dataset(path)`, then `read::<T>()`, `read_nd::<T>()` (data plus shape), `read_selection(&Selection::from_ranges(..))` for hyperslabs, `read_values()` for compounds and variable-length data as `Value`, and `attribute(name)?.read_scalar()`.
- Writing: `write::FileWriter::create(path)`, groups via `create_group`, attributes via `set_attr`, and datasets via the `new_dataset::<T>(name)` builder (`.shape`, `.chunk`, `.shuffle`, `.deflate(level)`, `.write(&data)`), then `finish()`.
- Editing: `FileWriter::open_rw(path)` modifies an existing file: `unlink`, `delete_attr`, `write_selection` into a dataset, and `resize` on chunked datasets (within their maximum dimensions).
- Custom codecs: implement `filters::Filter` and register it in a `FilterRegistry` (or `FileWriter::register_filter`).
- Examples in the repo: `cargo run --example h5ls -- -v -d file.h5` prints a tree view of a file.

**Example** (write then read back; compiled and run against the current repo):
```rust
use hdf5rs::write::FileWriter;
use hdf5rs::{File, Selection};

fn main() -> hdf5rs::Result<()> {
    let mut f = FileWriter::create("out.h5")?;
    let mut root = f.root();
    root.set_attr("title", "example")?;
    let mut g = root.create_group("weather")?;
    let mut ds = g
        .new_dataset::<f64>("temperature")
        .shape(&[2, 3])
        .write(&[20.5, 21.0, 21.5, 22.0, 22.5, 23.0])?;
    ds.set_attr("units", "celsius")?;
    g.new_dataset::<i32>("counts")
        .shape(&[1000])
        .chunk(&[100])
        .shuffle()
        .deflate(6)
        .write(&(0..1000).collect::<Vec<i32>>())?;
    f.finish()?;

    let file = File::open("out.h5")?;
    let temps = file.dataset("/weather/temperature")?;
    let grid = temps.read_nd::<f64>()?; // grid.data, grid.shape == [2, 3]
    let row0: Vec<f64> = temps.read_selection(&Selection::from_ranges(&[0..1, 0..3]))?;
    let units: String = temps.attribute("units")?.read_scalar()?;
    println!("{:?} {row0:?} {units}", grid.shape);
    Ok(())
}
```

**Gotchas:**
- Version 0.0.1 is the only release so far, and the repo has had no commits since 2026-10-19. Pin the exact version; pre-1.0 releases may break the API.
- Filters are compiled in by feature. Reading a dataset whose filter you didn't enable (for example zstd from h5py's `hdf5plugin`) fails with "filter id N is not available" (unless the filter was marked optional in the file), so enable `all-filters` when you read files from unknown sources.
- The writer does not produce SWMR-compatible files yet.
