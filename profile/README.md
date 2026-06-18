# Karpeles Lab Inc.

**IT Development & R&D based in Tokyo, Japan**

We develop and maintain a wide range of open-source tools and libraries, primarily in **Go** and increasingly in **Rust**, with a focus on cloud infrastructure, networking, cryptography, language runtimes, and system-level utilities — often with an emphasis on pure, dependency-free implementations (no CGO, no C, no FFI).

[![Website](https://img.shields.io/badge/Website-klb.jp-blue)](https://klb.jp)

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
| [cterm](https://github.com/KarpelesLab/cterm) | Terminal emulator optimized for AI coding tools | ![Stars](https://img.shields.io/github/stars/KarpelesLab/cterm) |

---

## Repository Categories

### Languages & Runtimes
Interpreters, shells, and runtime environments.

- **[goro](https://github.com/KarpelesLab/goro)** - PHP interpreter implemented in Go
- **[goro-rs](https://github.com/KarpelesLab/goro-rs)** - PHP interpreter in Rust
- **[gsh](https://github.com/KarpelesLab/gsh)** - Go native shell replacement
- **[nodejs](https://github.com/KarpelesLab/nodejs)** - Node.js factory and pool for Go

### AI & Developer Tools
Tooling built around AI-assisted development.

- **[teamclaude](https://github.com/KarpelesLab/teamclaude)** - Multi-account Claude proxy with automatic quota-based rotation
- **[cterm](https://github.com/KarpelesLab/cterm)** - Terminal emulator optimized for AI coding tools
- **[mcprun](https://github.com/KarpelesLab/mcprun)** - MCP server runner
- **[bnpm](https://github.com/KarpelesLab/bnpm)** - Sandboxed package manager using Linux namespaces

### Pure Rust Ecosystem
A growing family of pure-Rust implementations — no C, no FFI, often `no_std`.

- **[argus](https://github.com/KarpelesLab/argus)** - Web browser written in pure Rust (in-house engine, GUI + headless)
- **[kataan](https://github.com/KarpelesLab/kataan)** - High-performance JavaScript engine in pure Rust (interpreter, bytecode VM, x86-64 JIT)
- **[purecrypto](https://github.com/KarpelesLab/purecrypto)** - Pure-Rust crypto toolkit: classical & post-quantum, X.509, TLS/DTLS/QUIC
- **[rsurl](https://github.com/KarpelesLab/rsurl)** - Pure-Rust curl — HTTP/1-3, FTP, SFTP, WebSocket, and many more protocols
- **[puressh](https://github.com/KarpelesLab/puressh)** - Pure-Rust SSH library and CLI suite (ssh/sftp/scp/sshd)
- **[compcol](https://github.com/KarpelesLab/compcol)** - 30+ compression/decompression codecs behind one uniform streaming API
- **[fstool](https://github.com/KarpelesLab/fstool)** - Build, inspect, convert, and repack disk images and filesystems
- **[univdreams](https://github.com/KarpelesLab/univdreams)** - Universal decompiler + compiler (ELF, PE, Mach-O round-trip)
- **[cacrt](https://github.com/KarpelesLab/cacrt)** - `no_std` curated CA root certificates by OpenSSL subject hash
- **[psl2](https://github.com/KarpelesLab/psl2)** - Fast `no_std` Public Suffix List with built-in IDNA
- **[origami](https://github.com/KarpelesLab/origami)** - Experimental first-principles protein folder (all-atom MD, GPU-accelerated via wgpu)
- **[fullrust](https://github.com/KarpelesLab/fullrust)** - 100% Rust binaries

### Cloud & Infrastructure
Building blocks for cloud-native applications and distributed systems.

- **[fleet](https://github.com/KarpelesLab/fleet)** - Fleet cloud system
- **[clouddb](https://github.com/KarpelesLab/clouddb)** - Decentralized indexed database using LevelDB
- **[cloudhttp](https://github.com/KarpelesLab/cloudhttp)** - Easy SSL HTTP server for AWS, GCP, etc.
- **[cloudinfo](https://github.com/KarpelesLab/cloudinfo)** - Fetch info on current cloud environment
- **[seidan](https://github.com/KarpelesLab/seidan)** - 星団 server cluster project
- **[lambda](https://github.com/KarpelesLab/lambda)** - Lambda utilities

### Networking & Protocols
Low-level networking tools and protocol implementations.

- **[magictls](https://github.com/KarpelesLab/magictls)** - Auto PROXY/PROXYv2/TLS detection on TCP streams
- **[dns](https://github.com/KarpelesLab/dns)** - Modular DNS tools
- **[pktkit](https://github.com/KarpelesLab/pktkit)** - Zero-copy L2/L3 packet handling — virtual switches, NAT, DHCP, ARP, NDP
- **[slirp](https://github.com/KarpelesLab/slirp)** - SLiRP networking stack in Go
- **[pppoeproxy](https://github.com/KarpelesLab/pppoeproxy)** - Simple PPPoE client/server proxy
- **[smartremote](https://github.com/KarpelesLab/smartremote)** - Transparent partial HTTP file access with caching and resume

### Cryptography & Security
Secure implementations for authentication, HSM, and post-quantum cryptography.

- **[hsm](https://github.com/KarpelesLab/hsm)** - Go support for Hardware Security Modules
- **[jwt](https://github.com/KarpelesLab/jwt)** - JWT tokens without external dependencies
- **[jwttool](https://github.com/KarpelesLab/jwttool)** - Command-line JWT generation tool
- **[mldsa](https://github.com/KarpelesLab/mldsa)** / **[slhdsa](https://github.com/KarpelesLab/slhdsa)** - Post-quantum signature schemes
- **[authenticode](https://github.com/KarpelesLab/authenticode)** - Pure-Go Authenticode signing for Windows PE files (no CGO)
- **[tpmlib](https://github.com/KarpelesLab/tpmlib)** - TPM (Trusted Platform Module) support
- **[vncpasswd](https://github.com/KarpelesLab/vncpasswd)** - VNC password encryption/decryption

### Blockchain & Wallets
Multi-chain wallet infrastructure, signing, and cryptographic primitives.

- **[libwallet](https://github.com/KarpelesLab/libwallet)** - Multi-chain mobile wallet library with TSS — Ethereum, Bitcoin, Solana, NFTs
- **[modchain](https://github.com/KarpelesLab/modchain)** - Modular blockchain system
- **[tss-lib](https://github.com/KarpelesLab/tss-lib)** - Threshold signature scheme library
- **[ethrpc](https://github.com/KarpelesLab/ethrpc)** - Ethereum RPC client
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
Multimedia processing libraries, often pure Go with no CGO.

- **[avgo](https://github.com/KarpelesLab/avgo)** - AV library in Go
- **[ffprobe](https://github.com/KarpelesLab/ffprobe)** - FFprobe tools in Go
- **[hlsmaker](https://github.com/KarpelesLab/hlsmaker)** - HLS stream creation
- **[goavif](https://github.com/KarpelesLab/goavif)** - AVIF codec in pure Go (no CGO)
- **[gowebp](https://github.com/KarpelesLab/gowebp)** - Pure-Go WebP encoder (no CGO)
- **[static-opus](https://github.com/KarpelesLab/static-opus)** - Statically linked libopus for Go
- **[static-portaudio](https://github.com/KarpelesLab/static-portaudio)** - PortAudio static for Go
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
- **[strtotime](https://github.com/KarpelesLab/strtotime)** - PHP-compatible strtotime() for Go (natural-language date parsing)
- **[gotz](https://github.com/KarpelesLab/gotz)** - Raw IANA timezone data with embedded tzdata
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
- **[intel-dcapd](https://github.com/KarpelesLab/intel-dcapd)** - Intel DCAP daemon

### Frontend & Web
JavaScript/TypeScript libraries for web development.

- **[klbfw](https://github.com/KarpelesLab/klbfw)** - KLB framework module for frontend code
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
- **[unison](https://github.com/KarpelesLab/unison)** - Coalescing duplicate function calls with generics and caching
- **[bitmap](https://github.com/KarpelesLab/bitmap)** - Bitmap manipulation
- **[ringbuf](https://github.com/KarpelesLab/ringbuf)** - Ring buffer with readers
- **[weak](https://github.com/KarpelesLab/weak)** - Weak reference map (Go 1.18+)
- **[textutil](https://github.com/KarpelesLab/textutil)** - Text processing (word wrapping, etc.)
- **[typutil](https://github.com/KarpelesLab/typutil)** - Type conversion utilities
- **[rndstr](https://github.com/KarpelesLab/rndstr)** - Random string generation

---

## Technology Focus

Our 200+ repositories are predominantly written in:

- **Go** - Cloud infrastructure, networking, crypto, language runtimes, and utilities
- **Rust** - A growing pure-Rust ecosystem (browser, crypto, networking, SSH, compression)
- **JavaScript/TypeScript** - Frontend frameworks and React/Vue components
- **C/C++** - Static library bindings and low-level tools
- **Python** - USD tools and analysis utilities

---

## Contributing

We welcome contributions! Each repository has its own contribution guidelines. Feel free to open issues or submit pull requests.

---

*Tokyo, Japan*
