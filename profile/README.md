# Karpeles Lab Inc.

**IT Development & R&D based in Tokyo, Japan**

We develop and maintain a wide range of open-source tools and libraries, primarily in **Go** and increasingly in **Rust**, with a focus on cloud infrastructure, networking, cryptography, language runtimes, and system-level utilities — often with an emphasis on pure, dependency-free implementations (no CGO, no C, no FFI).

[![Website](https://img.shields.io/badge/Website-klb.jp-blue)](https://klb.jp)
[![OxideAV](https://img.shields.io/badge/GitHub-OxideAV-orange)](https://github.com/OxideAV)
[![portablesql](https://img.shields.io/badge/GitHub-portablesql-00ADD8)](https://github.com/portablesql)

---

## Featured Projects

| Project | Description | Stars |
|---------|-------------|-------|
| [goro](https://github.com/KarpelesLab/goro) | PHP interpreter implemented in Go | ![Stars](https://img.shields.io/github/stars/KarpelesLab/goro) |
| [usdpython](https://github.com/KarpelesLab/usdpython) | Apple's usdzconvert and USD-related tools | ![Stars](https://img.shields.io/github/stars/KarpelesLab/usdpython) |
| [teamclaude](https://github.com/KarpelesLab/teamclaude) | Multi-account Claude proxy with quota-based rotation | ![Stars](https://img.shields.io/github/stars/KarpelesLab/teamclaude) |
| [reflink](https://github.com/KarpelesLab/reflink) | Reflink (copy-on-write) file copy in Go | ![Stars](https://img.shields.io/github/stars/KarpelesLab/reflink) |
| [squashfs](https://github.com/KarpelesLab/squashfs) | SquashFS read/write implementation in pure Go | ![Stars](https://img.shields.io/github/stars/KarpelesLab/squashfs) |
| [magictls](https://github.com/KarpelesLab/magictls) | Automatic PROXY, PROXYv2 and TLS support on TCP streams | ![Stars](https://img.shields.io/github/stars/KarpelesLab/magictls) |
| [compcol](https://github.com/KarpelesLab/compcol) | 30+ compression codecs behind one streaming API, pure Rust | ![Stars](https://img.shields.io/github/stars/KarpelesLab/compcol) |
| [cterm](https://github.com/KarpelesLab/cterm) | Terminal emulator optimized for AI coding tools | ![Stars](https://img.shields.io/github/stars/KarpelesLab/cterm) |
| [graphitesql](https://github.com/KarpelesLab/graphitesql) | Pure, safe, `no_std` Rust re-implementation of SQLite | ![Stars](https://img.shields.io/github/stars/KarpelesLab/graphitesql) |
| [OxideAV](https://github.com/OxideAV) | Pure-Rust media transcoding and streaming framework — 140+ codec, container and filter crates | ![Stars](https://img.shields.io/github/stars/OxideAV/oxideav-workspace) |
| [portablesql](https://github.com/portablesql) | Portable SQL toolkit for Go — one codebase for MySQL, PostgreSQL and SQLite | ![Stars](https://img.shields.io/github/stars/portablesql/psql) |

---

## Repository Categories

### Languages & Runtimes
Interpreters, shells, and runtime environments.

- **[goro](https://github.com/KarpelesLab/goro)** - PHP interpreter implemented in Go
- **[goro-rs](https://github.com/KarpelesLab/goro-rs)** - PHP interpreter in Rust
- **[rustygo](https://github.com/KarpelesLab/rustygo)** - Go → Rust compiler with a Rust runtime (GC, scheduler, channels, reflect) — design stage
- **[gsh](https://github.com/KarpelesLab/gsh)** - Go native shell replacement
- **[nodejs](https://github.com/KarpelesLab/nodejs)** - Node.js factory and pool for Go

### AI & Developer Tools
Tooling built around AI-assisted development.

- **[teamclaude](https://github.com/KarpelesLab/teamclaude)** - Multi-account Claude proxy with automatic quota-based rotation
- **[cterm](https://github.com/KarpelesLab/cterm)** - Terminal emulator optimized for AI coding tools
- **[mcprun](https://github.com/KarpelesLab/mcprun)** - MCP server runner
- **[manu](https://github.com/KarpelesLab/manu)** - Multipurpose MCP server that gives AI agents hands
- **[atelier](https://github.com/KarpelesLab/atelier)** - Minimal-TUI AI coding harness in Rust for OpenAI-compatible APIs
- **[aipencil](https://github.com/KarpelesLab/aipencil)** - Render structured JSON scene descriptions to deterministic SVG/PNG
- **[nixvm](https://github.com/KarpelesLab/nixvm)** - Portable sandbox running a real Linux userland by emulating syscalls, not hardware
- **[bnpm](https://github.com/KarpelesLab/bnpm)** - Sandboxed package manager using Linux namespaces

### Pure Rust Ecosystem
A growing family of pure-Rust implementations — no C, no FFI, often `no_std`. See also [OxideAV](#media--audio) and [graphitesql](#databases--sql).

**Applications & engines**
- **[argus](https://github.com/KarpelesLab/argus)** - Web browser written in pure Rust (in-house engine, GUI + headless)
- **[kataan](https://github.com/KarpelesLab/kataan)** - High-performance JavaScript engine in pure Rust (interpreter, bytecode VM, x86-64 JIT, WebAssembly)
- **[z3rs](https://github.com/KarpelesLab/z3rs)** - `no_std` port of the Z3 theorem prover, no GMP or native deps
- **[mathesis](https://github.com/KarpelesLab/mathesis)** - Mathematica-style computational notebook running entirely in the browser ([live](https://karpeleslab.github.io/mathesis/))
- **[rsurl](https://github.com/KarpelesLab/rsurl)** - Pure-Rust curl — HTTP/1-3, FTP, SFTP, WebSocket, and many more protocols
- **[httpsd](https://github.com/KarpelesLab/httpsd)** - HTTP/1.1, HTTP/2 and HTTP/3 server with a sans-I/O core and pluggable runtimes
- **[puressh](https://github.com/KarpelesLab/puressh)** - SSH library and CLI suite (ssh/sftp/scp/sshd)
- **[puregit](https://github.com/KarpelesLab/puregit)** - Git from scratch: object model, packfiles, smart protocol client + server over HTTP and SSH
- **[fstool](https://github.com/KarpelesLab/fstool)** - Build, inspect, convert, and repack disk images and filesystems
- **[origami](https://github.com/KarpelesLab/origami)** - Experimental first-principles protein folder (all-atom MD, GPU-accelerated via wgpu)

**Systems & toolchain**
- **[kintane](https://github.com/KarpelesLab/kintane)** - Modular OS kernel, from no-MMU microcontrollers to multi-socket SMP
- **[purestd](https://github.com/KarpelesLab/purestd)** - A Rust `std` that never calls libc — direct kernel syscalls all the way down
- **[rustlibc](https://github.com/KarpelesLab/rustlibc)** - Full libc in Rust with C bindings, a drop-in for glibc or musl
- **[latticefoundry](https://github.com/KarpelesLab/latticefoundry)** - Clean-room compiler back-end framework (SSA IR, codegen, object files, linker core)
- **[qld](https://github.com/KarpelesLab/qld)** - Fast parallel linker, drop-in for GNU ld / gold / lld / mold
- **[rsasm](https://github.com/KarpelesLab/rsasm)** - Multi-target assembler accepting real-world asm syntaxes
- **[reticle](https://github.com/KarpelesLab/reticle)** - VHDL and Verilog compiler from scratch: simulation, synthesis, formal verification, FPGA place-and-route, bitstreams and JTAG programming
- **[univdreams](https://github.com/KarpelesLab/univdreams)** - Universal decompiler + compiler (ELF, PE, Mach-O round-trip)
- **[rsemu](https://github.com/KarpelesLab/rsemu)** - Multiplatform emulator built on a generic framework, machines described by config files
- **[x11anywhere](https://github.com/KarpelesLab/x11anywhere)** - Portable X11 server with native backends for Linux, macOS and Windows
- **[fullrust](https://github.com/KarpelesLab/fullrust)** - 100% Rust binaries

**Libraries**
- **[purecrypto](https://github.com/KarpelesLab/purecrypto)** - Crypto toolkit: classical & post-quantum, X.509, TLS/DTLS/QUIC
- **[purecrypto-tpm](https://github.com/KarpelesLab/purecrypto-tpm)** - TPM 2.0 stack speaking the wire protocol directly, no tpm2-tss
- **[puremp](https://github.com/KarpelesLab/puremp)** - Arbitrary-precision arithmetic: integers, rationals, MPFR-class floats, polynomials, matrices
- **[compcol](https://github.com/KarpelesLab/compcol)** - 30+ compression/decompression codecs behind one uniform streaming API
- **[minizlib](https://github.com/KarpelesLab/minizlib)** - Tiny gzip/zlib/deflate codec: `no_std`, no allocation, no `unsafe`, ~2.5 KB decompressor
- **[minlz-rs](https://github.com/KarpelesLab/minlz-rs)** - S2 compression, binary-compatible with Go's klauspost/compress/s2
- **[anydcode](https://github.com/KarpelesLab/anydcode)** - Encode/decode 50+ 1D/2D barcode symbologies (QR, Data Matrix, PDF417, Aztec, App Clip Codes…)
- **[charcode](https://github.com/KarpelesLab/charcode)** - WHATWG Encoding Standard character conversion, zero deps, no `unsafe`
- **[tomlproc](https://github.com/KarpelesLab/tomlproc)** - Complete TOML 1.1.0 parser and serializer, zero deps, `no_std`
- **[emjson](https://github.com/KarpelesLab/emjson)** - Streaming JSON parser, writer and in-place editor for embedded systems, in a few hundred bytes of RAM
- **[noroi](https://github.com/KarpelesLab/noroi)** - Rich curses-style terminal UI with zero external crates
- **[stipple](https://github.com/KarpelesLab/stipple)** - Self-drawn, themeable cross-platform UI toolkit (desktop, mobile, web)
- **[ldtray](https://github.com/KarpelesLab/ldtray)** - Cross-platform tray icons with no compile-time GUI linkage
- **[rawusb](https://github.com/KarpelesLab/rawusb)** - Dependency-free cross-platform USB access in the spirit of libusb (Linux, macOS, Windows)
- **[rsupd](https://github.com/KarpelesLab/rsupd)** - Signed release distribution and in-place auto-updates (successor to goupd)
- **[cacrt](https://github.com/KarpelesLab/cacrt)** - `no_std` curated CA root certificates by OpenSSL subject hash
- **[psl2](https://github.com/KarpelesLab/psl2)** - Fast `no_std` Public Suffix List with built-in IDNA

### Databases & SQL
Database toolkits and storage engines.

- **[portablesql](https://github.com/portablesql)** - Portable SQL toolkit for Go: write database code once, run it on MySQL / MariaDB, PostgreSQL / CockroachDB and SQLite
  - [psql](https://github.com/portablesql/psql) core (struct binding, query builder, transactions, associations) with [MySQL](https://github.com/portablesql/psql-mysql), [PostgreSQL](https://github.com/portablesql/psql-pgsql) and [SQLite](https://github.com/portablesql/psql-sqlite) drivers — [docs](https://portablesql.github.io)
- **[graphitesql](https://github.com/KarpelesLab/graphitesql)** - Pure-Rust, `no_std` re-implementation of SQLite, file-format compatible, targets WebAssembly
- **[pebbledb](https://github.com/KarpelesLab/pebbledb)** - Rust port of CockroachDB's Pebble LSM key-value storage engine

### Cloud & Infrastructure
Building blocks for cloud-native applications and distributed systems.

- **[fleet](https://github.com/KarpelesLab/fleet)** - Fleet cloud system
- **[clouddb](https://github.com/KarpelesLab/clouddb)** - Decentralized indexed database using LevelDB
- **[cloudhttp](https://github.com/KarpelesLab/cloudhttp)** - Easy SSL HTTP server for AWS, GCP, etc.
- **[cloudinfo](https://github.com/KarpelesLab/cloudinfo)** - Fetch info on current cloud environment
- **[lambda](https://github.com/KarpelesLab/lambda)** - Lambda utilities

### Networking & Protocols
Low-level networking tools and protocol implementations.

- **[magictls](https://github.com/KarpelesLab/magictls)** - Auto PROXY/PROXYv2/TLS detection on TCP streams
- **[dns](https://github.com/KarpelesLab/dns)** - Modular DNS tools
- **[pktkit](https://github.com/KarpelesLab/pktkit)** / **[pktkit-rs](https://github.com/KarpelesLab/pktkit-rs)** - Zero-copy packet toolkit for virtual network topologies — switches, NAT, virtual TCP/IP, WireGuard (Go / Rust)
- **[slirp](https://github.com/KarpelesLab/slirp)** - SLiRP networking stack in Go
- **[pppoeproxy](https://github.com/KarpelesLab/pppoeproxy)** - Simple PPPoE client/server proxy
- **[spotlib](https://github.com/KarpelesLab/spotlib)** / **[spotlib-rs](https://github.com/KarpelesLab/spotlib-rs)** - Spot end-to-end encrypted messaging protocol client (Go / Rust)
- **[smartremote](https://github.com/KarpelesLab/smartremote)** - Transparent partial HTTP file access with caching and resume

### Cryptography & Security
Secure implementations for authentication, HSM, and post-quantum cryptography.

- **[hsm](https://github.com/KarpelesLab/hsm)** - Go support for Hardware Security Modules
- **[jwt](https://github.com/KarpelesLab/jwt)** - JWT tokens without external dependencies
- **[jwttool](https://github.com/KarpelesLab/jwttool)** - Command-line JWT generation tool
- **[mldsa](https://github.com/KarpelesLab/mldsa)** / **[slhdsa](https://github.com/KarpelesLab/slhdsa)** - Post-quantum signature schemes
- **[authenticode](https://github.com/KarpelesLab/authenticode)** - Pure-Go Authenticode signing for Windows PE files (no CGO)
- **[gosigner](https://github.com/KarpelesLab/gosigner)** - Remote Authenticode signing with a USB token over the Spot network
- **[lnp22](https://github.com/KarpelesLab/lnp22)** - Lattice-based NIZK proof framework (LNP22, CRYPTO 2022)
- **[anyhash](https://github.com/KarpelesLab/anyhash)** - 60 hash algorithms selected by name, with cloneable state
- **[tmpsecfile](https://github.com/KarpelesLab/tmpsecfile)** - Anonymous, encrypted-at-rest temporary files for Go
- **[tpmlib](https://github.com/KarpelesLab/tpmlib)** - TPM (Trusted Platform Module) support
- **[vncpasswd](https://github.com/KarpelesLab/vncpasswd)** - VNC password encryption/decryption

### Blockchain & Wallets
Multi-chain wallet infrastructure, signing, and cryptographic primitives.

- **[libwallet](https://github.com/KarpelesLab/libwallet)** - Multi-chain mobile wallet library with TSS — Ethereum, Bitcoin, Solana, NFTs
- **[tss-lib](https://github.com/KarpelesLab/tss-lib)** / **[tsslib-rs](https://github.com/KarpelesLab/tsslib-rs)** - Threshold signature schemes, wire-compatible (Go / Rust)
- **[outscript](https://github.com/KarpelesLab/outscript)** / **[outscript-rs](https://github.com/KarpelesLab/outscript-rs)** - Output scripts, addresses and transaction signing across many networks (Go / Rust)
- **[ethrpc](https://github.com/KarpelesLab/ethrpc)** / **[ethrpc-rs](https://github.com/KarpelesLab/ethrpc-rs)** - Ethereum JSON-RPC client (Go / Rust)
- **[erigon-seg](https://github.com/KarpelesLab/erigon-seg)** - Read, query, write and merge Erigon 3 seg state files
- **[chiefsplitter](https://github.com/KarpelesLab/chiefsplitter)** - Solana program for configurable fee splitting
- **[secp256k1](https://github.com/KarpelesLab/secp256k1)** / **[edwards25519](https://github.com/KarpelesLab/edwards25519)** - Elliptic curve primitives
- **[base58](https://github.com/KarpelesLab/base58)** / **[bech32m](https://github.com/KarpelesLab/bech32m)** - Address encodings
- **[blindsig](https://github.com/KarpelesLab/blindsig)** - Blind signatures

### File Systems & Storage
Pure Go implementations for various file system and archive formats.

- **[squashfs](https://github.com/KarpelesLab/squashfs)** - SquashFS read/write implementation
- **[iso9660](https://github.com/KarpelesLab/iso9660)** - ISO9660 image reading and creation
- **[vfs](https://github.com/KarpelesLab/vfs)** - Virtual filesystem abstraction
- **[reflink](https://github.com/KarpelesLab/reflink)** - Copy-on-write file copy
- **[gobzip2](https://github.com/KarpelesLab/gobzip2)** - Pure Go bzip2 compression/decompression with parallel support
- **[gzscan](https://github.com/KarpelesLab/gzscan)** - Scanner for gzip files in disk images

### Media & Audio
Multimedia processing libraries — pure Rust and pure Go, no C dependencies where possible.

- **[OxideAV](https://github.com/OxideAV)** - Pure-Rust media transcoding and streaming — no C libraries, no FFI, every format implemented clean-room from the spec
  - 140+ crates: video (H.264/H.265/H.266, AV1, VP8/VP9, ProRes…), audio (AAC, Opus, FLAC, MP3, AC-3, DTS…), images (PNG, WebP, JPEG XL, AVIF, HEIF…), containers (MP4, Matroska, Ogg…) and subtitles
  - The `oxideav` CLI and `oxideplay` player ship from [oxideav-workspace](https://github.com/OxideAV/oxideav-workspace); use crates individually or all at once via [oxideav-meta](https://github.com/OxideAV/oxideav-meta)
- **[avgo](https://github.com/KarpelesLab/avgo)** - AV library in Go
- **[ffprobe](https://github.com/KarpelesLab/ffprobe)** - FFprobe tools in Go
- **[hlsmaker](https://github.com/KarpelesLab/hlsmaker)** - HLS stream creation
- **[goavif](https://github.com/KarpelesLab/goavif)** - AVIF codec in pure Go (no CGO)
- **[gowebp](https://github.com/KarpelesLab/gowebp)** - Pure-Go WebP encoder (no CGO)
- **[static-opus](https://github.com/KarpelesLab/static-opus)** - Statically linked libopus for Go
- **[static-portaudio](https://github.com/KarpelesLab/static-portaudio)** - PortAudio static for Go
- **[gogd](https://github.com/KarpelesLab/gogd)** - PHP gd image operations in native Go (no cgo, no libgd)
- **[mblur](https://github.com/KarpelesLab/mblur)** - ImageMagick's motionBlur ported to Go

### Data Formats & Parsing
Parsers and serializers for various data formats.

- **[pjson](https://github.com/KarpelesLab/pjson)** - Enhanced JSON with contexts and group resolution
- **[graphql](https://github.com/KarpelesLab/graphql)** - GraphQL query parser
- **[gopcre2](https://github.com/KarpelesLab/gopcre2)** - Pure Go PCRE2 (Perl-compatible regex), no CGO
- **[gomailparse](https://github.com/KarpelesLab/gomailparse)** - Streaming MIME email parser for Go
- **[ini](https://github.com/KarpelesLab/ini)** - Simple INI file handling
- **[csscolor](https://github.com/KarpelesLab/csscolor)** - CSS color parser
- **[tpl](https://github.com/KarpelesLab/tpl)** - Template engine for Go

### Internationalization & Localization
Tools for building multilingual applications.

- **[strftime](https://github.com/KarpelesLab/strftime)** - strftime with BCP 47 language tags
- **[strtotime](https://github.com/KarpelesLab/strtotime)** / **[strtotime-rs](https://github.com/KarpelesLab/strtotime-rs)** - PHP-compatible strtotime() (Go / `no_std` Rust)
- **[gotz](https://github.com/KarpelesLab/gotz)** / **[timezone-data-rs](https://github.com/KarpelesLab/timezone-data-rs)** - Raw IANA timezone data, embedded (Go / Rust)
- **[goicu](https://github.com/KarpelesLab/goicu)** / **[intlrs](https://github.com/KarpelesLab/intlrs)** - ICU-compatible features: transliteration, collation, normalization (Go / `no_std` Rust)
- **[lngdb](https://github.com/KarpelesLab/lngdb)** - Language database for Go
- **[countrydb](https://github.com/KarpelesLab/countrydb)** - Country database
- **[currencydb](https://github.com/KarpelesLab/currencydb)** - Currency database
- **[flagemoji](https://github.com/KarpelesLab/flagemoji)** - Flag emoji utilities
- **[ibanlib](https://github.com/KarpelesLab/ibanlib)** - IBAN parsing and generation

### Hardware Integration
Libraries for interacting with hardware devices.

- **[streamdeck](https://github.com/KarpelesLab/streamdeck)** - Elgato StreamDeck API
- **[hid](https://github.com/KarpelesLab/hid)** - Pure Go HID driver (no CGO)
- **[pixoo64](https://github.com/KarpelesLab/pixoo64)** - Pixoo64 LED display library
- **[usbmagic](https://github.com/KarpelesLab/usbmagic)** - Library and CLI for programmable USB test instruments (Cynthion), with [FPGA gateware](https://github.com/KarpelesLab/usbmagic-gateware)
- **[selecard](https://github.com/KarpelesLab/selecard)** - Reverse-engineered protocol and tool for the SeleCard III 426 MHz garage remote
- **[intel-dcapd](https://github.com/KarpelesLab/intel-dcapd)** - Intel DCAP daemon

### Frontend & Web
JavaScript/TypeScript libraries for web development.

- **[klbfw](https://github.com/KarpelesLab/klbfw)** - KLB framework module for frontend code (also in [Rust](https://github.com/KarpelesLab/klbfw-rs) and [Swift](https://github.com/KarpelesLab/swiftrest))
- **[fyvue](https://github.com/KarpelesLab/fyvue)** - Vue library for KLB systems
- **[react-klbfw-hooks](https://github.com/KarpelesLab/react-klbfw-hooks)** - React hooks for KLB framework
- **[react-autoruby](https://github.com/KarpelesLab/react-autoruby)** - Auto ruby tag generation for React
- **[i18next-klb-backend](https://github.com/KarpelesLab/i18next-klb-backend)** - i18next backend for KLB systems

### 3D & Apple USD
Tools for 3D content and Apple's Universal Scene Description format.

- **[usdpython](https://github.com/KarpelesLab/usdpython)** - Apple's usdzconvert and USD tools (our most starred Python repo!)

### Go Utilities
General-purpose Go libraries and tools.

- **[goupd](https://github.com/KarpelesLab/goupd)** - Auto updater for Go applications
- **[uhash](https://github.com/KarpelesLab/uhash)** - Multi-algorithm hashing tool
- **[xuid](https://github.com/KarpelesLab/xuid)** / **[xuid-rs](https://github.com/KarpelesLab/xuid-rs)** - Type-prefixed, base32-encoded UUIDs (Go / Rust)
- **[unison](https://github.com/KarpelesLab/unison)** - Coalescing duplicate function calls with generics and caching
- **[bitmap](https://github.com/KarpelesLab/bitmap)** - Bitmap manipulation
- **[ringbuf](https://github.com/KarpelesLab/ringbuf)** - Ring buffer with readers
- **[weak](https://github.com/KarpelesLab/weak)** - Weak reference map (Go 1.18+)
- **[textutil](https://github.com/KarpelesLab/textutil)** - Text processing (word wrapping, etc.)
- **[typutil](https://github.com/KarpelesLab/typutil)** - Type conversion utilities
- **[rndstr](https://github.com/KarpelesLab/rndstr)** - Random string generation

---

## Technology Focus

Our 240+ repositories are predominantly written in:

- **Go** - Cloud infrastructure, networking, crypto, language runtimes, and utilities, including [portablesql](https://github.com/portablesql)
- **Rust** - A fast-growing pure-Rust ecosystem (browser, JS engine, SQLite, kernel, toolchain, crypto, networking), including the [OxideAV](https://github.com/OxideAV) media stack
- **JavaScript/TypeScript** - Frontend frameworks and React/Vue components
- **C/C++** - Static library bindings and low-level tools
- **Python** - USD tools and analysis utilities

---

## Contributing

We welcome contributions! Each repository has its own contribution guidelines. Feel free to open issues or submit pull requests.

---

*Tokyo, Japan*
