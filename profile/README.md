# Karpeles Lab Inc.

**IT Development & R&D based in Tokyo, Japan**

We develop and maintain a wide range of open-source tools and libraries, primarily in **Go**, with a focus on cloud infrastructure, networking, cryptography, and system-level utilities.

[![Website](https://img.shields.io/badge/Website-klb.jp-blue)](https://klb.jp)

---

## Featured Projects

| Project | Description | Stars |
|---------|-------------|-------|
| [usdpython](https://github.com/KarpelesLab/usdpython) | Apple's usdzconvert and USD-related tools | ![Stars](https://img.shields.io/github/stars/KarpelesLab/usdpython) |
| [reflink](https://github.com/KarpelesLab/reflink) | Reflink (copy-on-write) file copy in Go | ![Stars](https://img.shields.io/github/stars/KarpelesLab/reflink) |
| [squashfs](https://github.com/KarpelesLab/squashfs) | SquashFS read-only implementation in pure Go | ![Stars](https://img.shields.io/github/stars/KarpelesLab/squashfs) |
| [magictls](https://github.com/KarpelesLab/magictls) | Automatic PROXY, PROXYv2 and TLS support on TCP streams | ![Stars](https://img.shields.io/github/stars/KarpelesLab/magictls) |
| [strftime](https://github.com/KarpelesLab/strftime) | strftime with BCP 47 language tag support | ![Stars](https://img.shields.io/github/stars/KarpelesLab/strftime) |
| [weak](https://github.com/KarpelesLab/weak) | Weak reference map for Go 1.18+ | ![Stars](https://img.shields.io/github/stars/KarpelesLab/weak) |

---

## Repository Categories

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
- **[slirp](https://github.com/KarpelesLab/slirp)** - SLiRP networking stack in Go
- **[pppoeproxy](https://github.com/KarpelesLab/pppoeproxy)** - Simple PPPoE client/server proxy
- **[rawnet](https://github.com/KarpelesLab/rawnet)** - Raw networking utilities

### Cryptography & Security
Secure implementations for authentication, HSM, and post-quantum cryptography.

- **[hsm](https://github.com/KarpelesLab/hsm)** - Go support for Hardware Security Modules
- **[jwt](https://github.com/KarpelesLab/jwt)** - JWT tokens without external dependencies
- **[jwttool](https://github.com/KarpelesLab/jwttool)** - Command-line JWT generation tool
- **[mldsa](https://github.com/KarpelesLab/mldsa)** / **[slhdsa](https://github.com/KarpelesLab/slhdsa)** - Post-quantum signature schemes
- **[tpmlib](https://github.com/KarpelesLab/tpmlib)** - TPM (Trusted Platform Module) support
- **[vncpasswd](https://github.com/KarpelesLab/vncpasswd)** - VNC password encryption/decryption

### File Systems & Storage
Pure Go implementations for various file system formats.

- **[squashfs](https://github.com/KarpelesLab/squashfs)** - SquashFS read-only implementation
- **[iso9660](https://github.com/KarpelesLab/iso9660)** - ISO9660 image reading and creation
- **[vfs](https://github.com/KarpelesLab/vfs)** - Virtual filesystem abstraction
- **[reflink](https://github.com/KarpelesLab/reflink)** - Copy-on-write file copy
- **[gzscan](https://github.com/KarpelesLab/gzscan)** - Scanner for gzip files in disk images

### Internationalization & Localization
Tools for building multilingual applications.

- **[strftime](https://github.com/KarpelesLab/strftime)** - strftime with BCP 47 language tags
- **[lngdb](https://github.com/KarpelesLab/lngdb)** - Language database for Go
- **[countrydb](https://github.com/KarpelesLab/countrydb)** - Country database
- **[currencydb](https://github.com/KarpelesLab/currencydb)** - Currency database
- **[flagemoji](https://github.com/KarpelesLab/flagemoji)** - Flag emoji utilities
- **[ibanlib](https://github.com/KarpelesLab/ibanlib)** - IBAN parsing and generation

### Media & Audio
Multimedia processing libraries with static linking options.

- **[avgo](https://github.com/KarpelesLab/avgo)** - AV library in Go
- **[ffprobe](https://github.com/KarpelesLab/ffprobe)** - FFprobe tools in Go
- **[hlsmaker](https://github.com/KarpelesLab/hlsmaker)** - HLS stream creation
- **[static-opus](https://github.com/KarpelesLab/static-opus)** - Statically linked libopus for Go
- **[static-portaudio](https://github.com/KarpelesLab/static-portaudio)** - PortAudio static for Go
- **[mblur](https://github.com/KarpelesLab/mblur)** - ImageMagick's motionBlur ported to Go

### Data Formats & Parsing
Parsers and serializers for various data formats.

- **[pjson](https://github.com/KarpelesLab/pjson)** - Enhanced JSON with contexts and group resolution
- **[graphql](https://github.com/KarpelesLab/graphql)** - GraphQL query parser
- **[ini](https://github.com/KarpelesLab/ini)** - Simple INI file handling
- **[csscolor](https://github.com/KarpelesLab/csscolor)** - CSS color parser
- **[tpl](https://github.com/KarpelesLab/tpl)** - Template engine for Go

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

- **[usdpython](https://github.com/KarpelesLab/usdpython)** - Apple's usdzconvert and USD tools (our most starred repo!)

### Go Utilities
General-purpose Go libraries and tools.

- **[gsh](https://github.com/KarpelesLab/gsh)** - Go native shell replacement
- **[goupd](https://github.com/KarpelesLab/goupd)** - Auto updater for Go applications
- **[uhash](https://github.com/KarpelesLab/uhash)** - Multi-algorithm hashing tool
- **[bitmap](https://github.com/KarpelesLab/bitmap)** - Bitmap manipulation
- **[ringbuf](https://github.com/KarpelesLab/ringbuf)** - Ring buffer with readers
- **[weak](https://github.com/KarpelesLab/weak)** - Weak reference map (Go 1.18+)
- **[textutil](https://github.com/KarpelesLab/textutil)** - Text processing (word wrapping, etc.)
- **[typutil](https://github.com/KarpelesLab/typutil)** - Type conversion utilities
- **[rndstr](https://github.com/KarpelesLab/rndstr)** - Random string generation

---

## Technology Focus

Our repositories are predominantly written in:

- **Go** - ~100 repositories (cloud infrastructure, networking, crypto, utilities)
- **JavaScript/TypeScript** - Frontend frameworks and React/Vue components
- **Rust** - Emerging projects (klbfw-rs, minlz-rs, x11anywhere)
- **C/C++** - Static library bindings and low-level tools
- **Python** - USD tools and utilities

---

## Contributing

We welcome contributions! Each repository has its own contribution guidelines. Feel free to open issues or submit pull requests.

---

*Tokyo, Japan*
