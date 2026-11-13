# KarpelesLab pure-Rust ecosystem

Karpeles Lab maintains a family of Rust crates and tools written from scratch in pure Rust: no C libraries, no FFI bindings to system libraries, and usually zero third-party crates. Many are `no_std`, many forbid `unsafe`, and several build for WebAssembly. This page describes the shared conventions and how the pieces fit together. Each area page lists every package with its crate name, publication status, maturity, when to use it, and verified usage examples.

## Area pages

| Page | Contents |
|------|----------|
| [crypto-networking.md](crypto-networking.md) | purecrypto (crypto, X.509, TLS/DTLS/QUIC, post-quantum), purecrypto-tpm, cacrt, psl2, dnsbox, puressh, httpsd, rsurl, pktkit-rs, spotlib-rs, rsupd |
| [data-formats.md](data-formats.md) | compcol, minizlib (deprecated), minlz-rs, emjson, tomlproc, charcode, anydcode (`anyd`), xuid-rs, hdf5rs |
| [i18n-math.md](i18n-math.md) | intlrs (`intl`), strtotime-rs, timezone-data-rs, puremp, z3rs, polyclip |
| [systems-toolchain.md](systems-toolchain.md) | kintane, purestd, fullrust, rustlibc, latticefoundry, qld, rsasm, reticle, univdreams, rsemu, algoc, lode, dlsym, dldota |
| [apps-engines.md](apps-engines.md) | argus, kataan, goro-rs, mathesis, puregit, fstool, origami, x11anywhere, cterm, cadlab, guide, ems, csx, nanami |
| [ui-hardware.md](ui-hardware.md) | noroi, stipple, ldtray, rawusb, usbmagic, visgrok, scopecap |
| [ai-tools-databases.md](ai-tools-databases.md) | atelier, carl (formerly manu), mcptk, oxinfer, nixvm, graphitesql, pebbledb |
| [blockchain.md](blockchain.md) | tsslib-rs, outscript-rs, ethrpc-rs, erigon-seg, libwallet, lsdk, btcwallet, chiefsplitter (now TibaneLabs), zanolib (archived), evmabiless |
| [oxideav.md](oxideav.md) | OxideAV media framework (codecs, containers, filters, `oxideav` CLI) |

Non-Rust packages: [../go.md](../go.md) (Go modules, portablesql) and [../frontend.md](../frontend.md) (KLB platform client libraries).

## Conventions for agents

- **Check the crate name, not the repo name.** Several repos publish under a different crate name (`anydcode` is `anyd`, `intlrs` is `intl`, `tsslib-rs` is `tsslib`, `outscript-rs` is `outscript`, `pktkit-rs` is `pktkit`). Some crates.io names that look like ours belong to unrelated projects (`goro`, `atelier`, `ethrpc`, `z3`, `lode`, `origami`, `xuid`). Each package section states the exact name to put in `Cargo.toml`.
- **crates.io vs git.** When a crate is published, depend on the crates.io version. When a section says "git only", use `{ git = "https://github.com/KarpelesLab/<repo>" }`. Some git-only crates reference siblings by relative path and need the sibling repo checked out next to them; the gotchas say so where it applies.
- **Cargo features.** Most crates keep the default feature set small and gate each algorithm, format or backend behind a feature. Enable what you need rather than `all`.
- **Maturity varies.** Each section has a Status field (experimental, usable, production). Several repos have READMEs that lag behind the code; where they disagree, these docs follow the source and say so.
- **Toolchain.** Most crates use edition 2024 and a recent stable Rust. Check `rust-version` in `Cargo.toml` if a build fails on an older compiler.

## How the pieces fit together

- **purecrypto** is the cryptographic root: primitives, X.509, sans-I/O TLS 1.2/1.3, DTLS and QUIC. **cacrt** supplies its embedded root certificates.
- **rsurl** (HTTP/1-3 and many protocols) and **httpsd** (HTTP server) use purecrypto for TLS/QUIC. httpsd uses rsurl as its ACME client. **puressh** provides SSH, and rsurl uses it for `sftp://`/`scp://`.
- **compcol** provides compression everywhere, including HPACK/QPACK for HTTP/2 and HTTP/3.
- **argus** (browser) combines **kataan** (JavaScript engine) with rsurl networking.
- **z3rs** builds on **puremp** for arbitrary precision; **mathesis** combines both in a browser notebook.
- **strtotime-rs** and **intlrs** use **timezone-data-rs** for IANA zones.
- **libwallet** builds on **tsslib-rs** for threshold signatures, and **outscript-rs** covers addresses and transaction signing across chains, including Zano (replacing the archived **zanolib**), Monero and Zcash shielded. **lsdk** (Ledger/BOLOS hardware wallets) and **btcwallet** build on outscript and purecrypto.
- **purestd** replaces `std` without libc; **fullrust** builds unmodified crates into libc-free static binaries; **rustlibc** goes the other way and provides a libc for C programs.
- **lode** compiles through **latticefoundry**; **cadlab** and **guide** use **polyclip** for board geometry, and cadlab uses OxideAV's 3D crates for models.
- **ems** (FDTD solver) builds on **csx** (geometry/materials) and **hdf5rs** (field dumps).
- **dlsym** loads shared libraries safely (used by oxinfer, puressh, OxideAV); on libc-free **fullrust** binaries it uses **dldota** to boot the system's ld.so.
- **rawusb** underpins the instrument tools **usbmagic**, **visgrok** and **scopecap**, and lsdk's Linux USB fallback.
- **oxinfer** (local LLM/diffusion inference) uses OxideAV for media I/O.
- **purecrypto 0.10** is the current line; tsslib, outscript and libwallet require it graph-wide (kataan's default `crypto` feature, purecrypto-tpm and ethrpc-rs `abi` still pull purecrypto 0.6 copies).
- Sister org [Tibane Labs](https://github.com/TibaneLabs) builds wallets and Solana programs on this stack (TibaneApp on libwallet, chiefsplitter, catcard firmware).
- Several crates have Go counterparts (outscript, ethrpc, xuid, spotlib, pktkit, strtotime, dns/dnsbox, minlz/S2); check each section for the exact compatibility claim. See [../go.md](../go.md).
