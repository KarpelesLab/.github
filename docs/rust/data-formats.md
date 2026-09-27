# Compression, Encodings & Data Formats

Pure-Rust codecs and format libraries with little or no dependency baggage. For compression there are three crates, each for a different job. [`compcol`](#compcol) is the full-featured one: 40+ algorithms (deflate/gzip/zlib, zstd, brotli, xz/LZMA, bzip2, LZ4, RAR/StuffIt/LHA decoders, ...) behind one streaming trait, `no_std` + `alloc`, zero deps. [`minizlib`](#minizlib) is a tiny gzip/zlib/deflate codec for microcontrollers, with no allocator and no panics, about 2.5 KB of code for decompression and about 1 KB for compression. [`minlz`](#minlz-rs) is the pick when you need byte-for-byte compatibility with Go's `klauspost/compress/s2` (Snappy-family) or MinIO's MinLZ, or very fast decoding. For data formats, [`emjson`](#emjson) fills the embedded JSON niche: it streams, parses, writes and edits in place in a few hundred bytes of RAM with no allocator, while `serde_json` needs a heap. [`tomlproc`](#tomlproc) is a zero-dependency TOML 1.1 parser/serializer. [`charcode`](#charcode) converts text encodings per WHATWG with no deps and no `unsafe`, [`anyd`](#anydcode) encodes and decodes 1D/2D barcodes, and [`xuid-rs`](#xuid-rs) provides type-prefixed base32 UUIDs.

**Choosing a compressor:**
- **compcol**: you need a format other than deflate, a format picked at runtime (factory/magic-byte detection), `std::io`/tokio adapters, decompression of legacy archive codecs, or good gzip ratio (lazy matching + dynamic Huffman). Needs `alloc`.
- **minizlib**: gzip/zlib/deflate only, on a target with no heap and tight flash/RAM (MCU firmware, bootloaders, OTA images). It has a smaller ratio when compressing (fixed Huffman only) and a slower decode speed (~80 MB/s).
- **minlz**: you exchange data with Go services that use S2/Snappy or MinLZ, or you need a very fast decoder (tens of GiB/s). Uses `crc` as a dependency and contains a few audited `unsafe` blocks.

## Quick pick
| Need | Use |
|------|-----|
| gzip/zlib/zstd/brotli/xz/bzip2/lz4 etc. through one API, `no_std` + alloc | [`compcol`](#compcol) |
| Choose a codec by name at runtime / sniff a compressed stream's format | [`compcol`](#compcol) (`factory`) |
| Decompress RAR 2/3/5, StuffIt, LHA, LZX/CAB, Quantum, PPMd, LZFSE | [`compcol`](#compcol) |
| HTTP/2 HPACK or HTTP/3 QPACK header compression | [`compcol`](#compcol) |
| gunzip/gzip on a microcontroller with no heap, a few KB of flash | [`minizlib`](#minizlib) |
| Stream-decompress an OTA/firmware image from UART to flash | [`minizlib`](#minizlib) |
| S2 / Snappy wire-compatible with Go `klauspost/compress/s2` | [`minlz`](#minlz-rs) |
| MinLZ (`.mz`, minio/minlz) streams, seekable indexes | [`minlz`](#minlz-rs) |
| Parse / edit JSON on an MCU with no allocator, or a huge JSON file in constant RAM | [`emjson`](#emjson) |
| Parse/write TOML config with zero dependencies (optional serde) | [`tomlproc`](#tomlproc) |
| Convert Shift_JIS / Big5 / windows-1252 / EBCDIC ... to/from UTF-8 | [`charcode`](#charcode) |
| Generate or read QR / Data Matrix / EAN / Code 128 / PDF417 barcodes | [`anyd`](#anydcode) |
| Type-prefixed, sortable, base32 IDs compatible with Go `xuid` | [`xuid-rs`](#xuid-rs) |

## compcol

**Repo:** https://github.com/KarpelesLab/compcol · **Crate:** `compcol` (crates.io `0.6.11`) · **License:** MIT · **Status:** usable (566+ tests, cross-validated against reference tools; fuzz targets)

A collection of compression algorithms in pure Rust behind one caller-buffered streaming trait (`Encoder`/`Decoder`/`Algorithm`). It is `#![no_std]` and `#![forbid(unsafe_code)]`, has zero runtime dependencies (tokio only with the `tokio` feature), and gates each algorithm behind its own cargo feature. MSRV 1.88, edition 2024.

**Use it when:** you need any mainstream or legacy compression format, want the format chosen from config or a CLI flag, need `Read`/`Write` or tokio `AsyncRead`/`AsyncWrite` adapters, or have to decompress untrusted input with an output cap.
**Don't use it when / limits:** you have no allocator (only `rle` works without `alloc`; use `minizlib` instead). zstd and brotli encoders produce conformant streams but lag the reference ratio. Some codecs are decode-only (`Error::Unsupported` on encode): RAR 1-5 (license), Quantum, LZFSE, PPMd, StuffIt methods. RAR1 does not decode either. LZX/Amiga LZX encoders emit uncompressed blocks only. This crate handles codec streams, not archive containers (no zip/rar/7z directory parsing).

**Add it:**
```toml
[dependencies]
compcol = "0.6"                                        # default: alloc, rle, deflate, zlib, gzip, factory
# compcol = { version = "0.6", features = ["std", "zstd", "brotli", "xz"] }
# compcol = { version = "0.6", features = ["all"] }    # every algorithm
```

**Key features / cargo features:**
- Capability features: `alloc`, `std` (`compcol::io`: `EncoderWriter`, `EncoderReader`, `DecoderWriter`, `DecoderReader`), `tokio` (same four names in `compcol::tokio_io`), `factory` (`encoder_by_name`, `decoder_by_name`, `names`, `extension`, `detect`), `checksum` (public Adler-32/CRC-32).
- Algorithm features: `deflate`, `deflate64`, `zlib`, `gzip`, `lzma`, `xz`, `lzma2`, `zstd`, `brotli`, `lz4`, `lz5`, `snappy`, `lzw`, `lzss`, `bzip2`, `lzo`, `lzx`, `amiga_lzx`, `quantum`, `lzfse`, `adc`, `ppmd`, `xpress`, `xpress_huffman`, `lznt1`, `lzham`, `lzs`, `packbits`, `lha`, `zip_implode`/`zip_shrink`/`zip_reduce`, `arc_*`, `sit13`/`lzah`/`arsenic`, `rar1`-`rar5`, filters `bcj`/`bcj2`/`delta`, primitives `huffman`/`rangecoder`/`mtf`/`bwt`, and `hpack`/`qpack`. The `all` feature enables everything.
- One-shot helpers in `compcol::vec`: `compress_to_vec`, `decompress_to_vec`, `*_with(config)`, and `decompress_to_vec_capped` (bomb-safe, returns `Error::OutputLimitExceeded`). `compcol::limit::LimitedDecoder` wraps any decoder with a cap.
- `Decoder::skip(input, n)` advances through decompressed output without emitting it.
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
- The trait methods return `Result<(Progress, Status), Error>`, a tuple. The README's trait sketch shows `Result<Progress, Error>`, which is stale. Loop until `Status::StreamEnd` on `finish()`.
- The README's install snippet says `version = "0.4"`, which is stale. Use `0.6`.
- `EncoderWriter` finishes on `Drop` on a best-effort basis. Call `.finish()` explicitly to see errors.
- `factory::detect` does not detect brotli or raw `.lzma` (they have no magic bytes). It only reports codecs that were compiled in.
- `decompress_to_vec` is unbounded. Use the `_capped` variants for untrusted input.

## minizlib

**Repo:** https://github.com/KarpelesLab/minizlib · **Crate:** `minizlib` (crates.io `0.1.1`) · **License:** MIT · **Status:** usable (tested against flate2, corruption and truncation fuzzing, CI footprint checks)

A tiny gzip/zlib/raw-deflate compressor and decompressor built for firmware. It is `no_std` and uses no allocator, no `unsafe`, no dependencies and no panics: no configuration links panic machinery. Full gzip decompression is about 2.4 KB of Thumb-2 code and compression about 1 KB. Decompression needs about 1.5 KiB of stack. All memory (output buffer, 32 KiB window, match table) comes from the caller. MSRV 1.89.

**Use it when:** you need gzip/zlib on a microcontroller or bootloader, you stream from UART/socket/flash to flash, you need a bounded decompressor that is safe against bombs by construction, or you want only the decompressed length of a stream (`gunzip_len` needs no window).
**Don't use it when / limits:** you need any other format or a good compression ratio. The compressor is greedy LZ77 with fixed Huffman only (about 32% vs `gzip -6` at 19% on source code), and matches are found only within a chunk. Decoding runs at about 80 MB/s (bitwise canonical Huffman). There are no preset dictionaries and no access to gzip header fields. On a desktop or server, use `compcol`.

**Add it:**
```toml
[dependencies]
minizlib = "0.1"
# decompress-only, gzip, dynamic blocks:
# minizlib = { version = "0.1", default-features = false, features = ["decompress", "gzip", "dynamic"] }
```

**Key features / cargo features:** default is all except `crc-table`.
- `decompress`, `compress`: the two halves. `gzip`, `zlib`: containers (raw deflate is always present).
- `checksum` (verify CRC-32/ISIZE/Adler-32), `crc-table` (1 KiB table instead of 64 B; faster), `concat` (multi-member gzip).
- `stored`, `fixed`, `dynamic`: which deflate block types the decoder accepts. Disabled types produce `Error::Unsupported`.
- API matrix: `gunzip`/`unzlib`/`inflate`/`decompress` (auto-detect), `*_len` (length only), `gzip`/`zlib`/`deflate`. Inputs: `&[u8]`, `Reader` (callback), `Bytes(iter)`, `Decompressor` (push, about 1.1 KiB state). Outputs: `Buffer`, `Stream` (window + callback), `Counter`. Streaming compression: `Compressor` and `BufferedCompressor`.

**Example:**
```rust
use minizlib::{gunzip, gzip, Buffer};

let mut table = [0u16; 4096];            // match table, need not be cleared
let mut gz = [0u8; 4096];
let gz_len = gzip(b"hello hello hello", &mut table, Buffer::new(&mut gz))? as usize;

let mut out = [0u8; 4096];
let len = gunzip(&gz[..gz_len], Buffer::new(&mut out))? as usize;
assert_eq!(&out[..len], b"hello hello hello");
# Ok::<(), minizlib::Error>(())
```

**Gotchas:**
- A `Stream` window must be at least as large as the compressor's window (32 KiB covers everything), otherwise decoding fails with `Error::WindowTooSmall`.
- `Stream`/`Counter`/`*_len` take a mandatory `max_len`. Exceeding it gives `Error::OutputFull`. Pass `NO_LIMIT` explicitly to opt out.
- Only disable decoder block types if you control the compressor. General-purpose gzip emits all three.
- `gzip_size_hint` reads the trailer ISIZE: a last-member-only, mod 2^32, unverified hint.

## minlz-rs

**Repo:** https://github.com/KarpelesLab/minlz-rs · **Crate:** `minlz` (crates.io `1.2.3`) · **License:** BSD-3-Clause · **Status:** production (1.x, Go byte-compat tests, proptest, libfuzzer, Miri)

It implements two Snappy-family codecs. **S2** has byte-for-byte identical output to Go `klauspost/compress/s2` in all four modes and decodes Snappy. **MinLZ** is the minio/minlz spec v1.0 `.mz` format; it decodes S2/Snappy, but its output cannot be read by them. Both come with block and stream formats (CRC32C framing), seek indexes and dictionaries. The block API is `no_std` + `alloc`. Decode is very fast (6-27x Go's asm, up to about 135 GiB/s in L1). Uses the `crc` crate and a few documented `unsafe` blocks in hot paths. MSRV 1.81.

**Use it when:** you interoperate with Go services or files using S2/Snappy/MinLZ, or you need the fastest possible decompression (caches, logs, RPC payloads) at a Snappy-like ratio.
**Don't use it when / limits:** you need gzip/zstd-class ratios (use `compcol`), or no allocator (use `minizlib`). The standard S2 encoder is 2-4x slower than Go's assembly. The MinLZ `Dict` format is crate-local and not interoperable with minio/minlz. The streaming API needs `std`.

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

**Repo:** https://github.com/KarpelesLab/anydcode · **Crate:** `anyd` (crates.io `0.2.0`) · **License:** MIT · **Status:** usable / actively developed (broad symbology coverage; image detection for a subset)

Barcode encoding and decoding written from scratch for about 40 1D, stacked, 2D and postal symbologies: QR/Micro QR/rMQR, Data Matrix, Aztec, PDF417/MicroPDF417, MaxiCode, Han Xin, DotCode, EAN/UPC, Code 128/GS1-128, Code 39/93, ITF, Codabar, DataBar, postal 4-state, Apple App Clip Codes and more. It is always `#![no_std]`. It has no dependencies (only the `cli` feature pulls `oxideav-png`) and a WASM demo. A decoded `Symbol` keeps every encoding decision (segments, version, EC level, mask), so re-encoding reproduces the symbol byte-for-byte. It includes a camera/live-video pipeline (`detect::locate` then decode) on grayscale frames. MSRV 1.89.

**Use it when:** you generate barcodes (including heap-free `encode_into` on MCUs), decode a module grid or pattern, or scan barcodes from still images or video frames in pure Rust or WASM.
**Don't use it when / limits:** image detection is implemented only for QR, Data Matrix, Aztec, Micro QR, rMQR, PDF417/MicroPDF417, App Clip and the main 1D codes. MaxiCode, Han Xin, DotCode, Grid Matrix, DataBar, Pharmacode, postal and stacked 1D codes decode structurally but cannot be located in images. PDF417 sampling is affine-only. Image scanning needs `std`. Default features embed about 1.7 MB of App Clip tables, so disable `appclip` if you don't need it.

**Add it:**
```toml
[dependencies]
anyd = "0.2"
# lean no_std: anyd = { version = "0.2", default-features = false, features = ["alloc", "encode", "decode", "qr", "code128"] }
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
