# Cryptography, TLS & Networking

A foreign-code-free security and networking stack. It is rooted in **`purecrypto`**, a single crate that provides everything from constant-time primitives through post-quantum crypto, X.509, TLS 1.2/1.3, DTLS and QUIC. The TLS engine is sans-I/O. Every other package here builds on it, and none of them links OpenSSL, ring or rustls by default:

- **`cacrt`** is the curated root-CA store. `purecrypto`'s `embedded-roots` feature (on by default) feeds it into `RootCertStore::with_embedded_roots()`, so anything using `purecrypto` TLS gets portable roots without reading the OS trust store.
- **`puressh`** (SSH), **`purecrypto-tpm`** (TPM 2.0) and **`pktkit`** (WireGuard/OpenVPN) take their crypto from `purecrypto`.
- **`rsurl`** (a curl clone) uses `purecrypto` for TLS/QUIC, the embedded `cacrt` roots by default, `puressh` for `sftp://` and `scp://`, and `psl2` for cookie domain checks.
- **`httpsd`** (an HTTP/1.1, /2 and /3 server) uses `purecrypto` TLS/QUIC and uses `rsurl` as its ACME client.
- **`spotlib`** (E2EE messaging) and **`rsupd`** (a signed auto-updater) sit on top. They use `purecrypto` plus `bottlers` (Bottle signing/encryption), with `rsurl` as the transport.

HTTP/2 HPACK, HTTP/3 QPACK and all compression come from the sibling crate `compcol`.

## Quick pick
| Need | Use |
|------|-----|
| Hashes, AEADs, KDFs, RSA/ECDSA/Ed25519, ML-KEM/ML-DSA/SLH-DSA, pure Rust or `no_std` | [`purecrypto`](#purecrypto) |
| TLS 1.2/1.3, DTLS or QUIC client/server engine (sans-I/O) | [`purecrypto`](#purecrypto) (`tls`, `dtls`, `quic`) |
| X.509 certs, CSRs, a small CA, PKCS#12, JOSE/JWT signing | [`purecrypto`](#purecrypto) |
| Talk to a TPM 2.0 (seal/unseal, PCR read, random) without tpm2-tss | [`purecrypto-tpm`](#purecrypto-tpm) |
| Trusted root CA certificates as static DER, `no_std`/no-alloc | [`cacrt`](#cacrt) |
| Registrable domain / public suffix (eTLD+1, cookie domain) | [`psl2`](#psl2) |
| SSH client/server, SFTP, SCP, port forwarding | [`puressh`](#puressh) |
| Embeddable HTTP/HTTPS server (h1/h2/h3, ACME), or a static file server CLI | [`httpsd`](#httpsd) |
| HTTP(S) client, curl-like CLI, many protocols (FTP, SFTP, WS, ...) | [`rsurl`](#rsurl) |
| Userspace packet plumbing: virtual L2/L3, NAT, TCP stack, WireGuard, TUN/TAP, AF_XDP | [`pktkit-rs`](#pktkit-rs) |
| End-to-end encrypted messaging over the Spot network | [`spotlib-rs`](#spotlib-rs) |
| Signed releases plus in-place self-update for a Rust binary | [`rsupd`](#rsupd) |

## purecrypto

**Repo:** https://github.com/KarpelesLab/purecrypto · **Crate:** `purecrypto` (crates.io `0.9.3`) · **License:** MIT · **Status:** usable, pre-1.0, broad test coverage (the full Wycheproof suite, NIST ACVP vectors, OpenSSL 3.5 interop). No third-party human audit, not FIPS validated.

A cryptography toolkit written entirely in Rust, with no C, no assembly and no third-party crypto crates. It covers constant-time primitives, bignum, RSA/EC/PQC, ASN.1/DER, X.509, TLS 1.2/1.3, DTLS 1.2/1.3 and QUIC v1. The core is `#![no_std]`, every module has its own feature gate, and library code does not use `unsafe` (it is allowed only in the `ffi` feature). The TLS engine is **sans-I/O**: you feed and pop bytes yourself. It also ships a C ABI (which also builds for WASM) and an OpenSSL-style `purecrypto` CLI.

**Use it when:**
- You need crypto without C or `*-sys` crates, including on embedded, WASM, or libc-free targets.
- You need post-quantum algorithms: ML-KEM, ML-DSA, SLH-DSA, Falcon, LMS/XMSS, or the X25519MLKEM768 TLS group.
- You want one dependency that covers primitives, X.509 and TLS together.

**Don't use it when / limits:**
- You need an audited or FIPS-validated library.
- You need a `rustls`/`tokio-rustls` drop-in. The API is its own (`tls::Config` + `tls::Connection`).
- Every `zkp-*` and `hazmat-*` feature has no semver guarantee.

**Add it:**
```toml
[dependencies]
purecrypto = "0.9"
# lean / no_std examples:
# purecrypto = { version = "0.9", default-features = false, features = ["hash", "cipher"] }
# purecrypto = { version = "0.9", default-features = false, features = ["mlkem"] }   # no alloc needed
```

**Key features / cargo features:**
- The default set is `std` plus most modules plus the `cli` binary plus `embedded-roots` (which pulls in `cacrt`). Use `default-features = false` for lean builds.
- Module features:
  - Hashes and ciphers: `hash`, `cipher`, `mac`, `kdf`, `rng`.
  - Asymmetric: `bignum`, `rsa`, `ec`, `dh`, `key` (an `EVP_PKEY`-style `PrivateKey`/`PublicKey` facade).
  - Encodings and certificates: `der`, `x509`, `pkcs12`, `jose`.
  - Protocols: `tls`, `dtls`.
  - Post-quantum and newer schemes: `mlkem`, `mldsa`, `slhdsa`, `lms`, `xmss`, `ascon`, `aez`.
- Opt-in features: `quic`, `hpke`, `ech`, `falcon`, `bls`, `bip340`, `ristretto255`, `zkp-*`, `hazmat-*`, `dsa`, `legacy-ec`, `legacy-ciphers`, `tls-legacy` (SSLv3/TLS 1.0/1.1, insecure), `ffi` (C ABI), `tokio` / `mio` (async TLS adapters, `tls::tokio::TlsStream`), `wasi-getrandom`.
- The `*-table` features trade flash for speed (`ed25519-table` is about 115 KB, `p256-table` about 61 KB). Turn them off on flash-constrained targets.
- `ct` mirrors the `subtle` crate and `zeroize` mirrors the `zeroize` crate, so you can drop both dependencies.
- The only non-std dependencies are `compcol` (under `cert-compression`), `cacrt` (under `embedded-roots`), and `tokio`/`mio` when those features are enabled.

**Example:**
```rust
use purecrypto::hash::{Digest, Sha256};
use purecrypto::ec::Ed25519PrivateKey;
use purecrypto::mlkem::MlKem768DecapsKey;
use purecrypto::rng::OsRng;

let digest = Sha256::digest(b"abc");

let sk = Ed25519PrivateKey::generate(&mut OsRng);
let sig = sk.sign(b"hello");
sk.public_key().verify(b"hello", &sig).unwrap();

let (dk, ek) = MlKem768DecapsKey::generate(&mut OsRng);
let (ct, ss_a) = ek.encapsulate(&mut OsRng);
assert_eq!(dk.decapsulate(&ct), ss_a);
```

TLS client. The engine is sans-I/O; this example drives it over a `TcpStream` and is condensed from `examples/tls_get.rs`:
```rust
use purecrypto::tls::{Config, Connection, HandshakeStatus, RootCertStore};
use std::{io::{Read, Write}, net::TcpStream, sync::Arc};

let cfg = Config::builder()
    .tls_only()
    .rng(Arc::new(purecrypto::rng::OsRng))        // required: no implicit RNG
    .roots(RootCertStore::with_embedded_roots())  // cacrt bundle
    .server_name("example.org")
    .build();
let mut conn = Connection::client(&cfg).unwrap();
let mut sock = TcpStream::connect(("example.org", 443)).unwrap();
let mut buf = [0u8; 8192];
loop {
    let out = conn.pop().unwrap_or_default();
    if !out.is_empty() { sock.write_all(&out).unwrap(); }
    match conn.handshake().unwrap() {
        HandshakeStatus::Complete => break,
        HandshakeStatus::WantWrite => continue,
        HandshakeStatus::WantRead => { let n = sock.read(&mut buf).unwrap(); conn.feed(&buf[..n]).unwrap(); }
    }
}
conn.send(b"GET / HTTP/1.1\r\nHost: example.org\r\n\r\n").unwrap();
sock.write_all(&conn.pop().unwrap_or_default()).unwrap();
```

**Gotchas:**
- `Connection::client`/`server` fails with `MissingEntropySource` unless `.rng(...)` was set. This is intentional: it lets a TPM or HSM supply entropy.
- Use `try_identity` rather than `identity`. It checks that the private key matches the leaf certificate.
- On TCP EOF, check `received_close_notify()`. Without it, the response may have been truncated.
- Signing keys held on external devices (TPM/HSM) use the suspend/resume seam (`SigningKey::External`). See `examples/tls_external_signing.rs`.
- `ct::ConditionallySelectable::conditional_select(a, b, choice)` has the argument order reversed relative to `subtle`. Use `conditional_select_b_if_true` for `subtle` semantics.
- The CLI installs with `cargo install purecrypto`. Examples: `purecrypto s_client -connect host:443`, `genpkey`, `x509`, `ca`, `kem`. See `docs/cli.md` in the repo.

## purecrypto-tpm

**Repo:** https://github.com/KarpelesLab/purecrypto-tpm · **Crate:** `purecrypto-tpm` (crates.io `0.1.0`) · **License:** MIT · **Status:** early.

A pure-Rust TPM 2.0 stack. It speaks the TCG command/response wire protocol directly over `/dev/tpm0`/`/dev/tpmrm0` or the MS/swtpm simulator socket. It uses neither `tpm2-tss` nor C. Session crypto (object Names, HMAC sessions, KDFa, AES-CFB) comes from `purecrypto`. The marshalling and session core is `no_std` + `alloc`, and it can drive any custom `Transport`.

**Use it when:**
- You need basic TPM operations from pure Rust: startup, `get_random`, `get_capability`, `pcr_read`, `create_primary`, `create`/`load`/`unseal`.
- You need HMAC-authorized sessions.

**Don't use it when / limits:**
- Salted sessions, parameter encryption, policy sessions (e.g. `PolicyPCR`), NV storage, persistent objects, and quote/attestation are **not implemented yet**.
- It is not yet wired to `purecrypto`'s TLS external-signer seam. You would glue that yourself.

**Add it:**
```toml
[dependencies]
purecrypto-tpm = "0.1"
```

**Key features / cargo features:**
- `std` (default): the OS transports plus `std::error::Error`.
- `device` (default): the Linux character device transport.
- `simulator` (default): the TCP transport to swtpm or ms-tpm-20-ref.
- Integration tests run against swtpm when `PURECRYPTO_TPM_SIM=127.0.0.1:2321` is set.

**Example:**
```rust
use purecrypto_tpm::Tpm;
use purecrypto_tpm::transport::{DeviceTransport, SimulatorTransport};

let mut tpm = Tpm::new(DeviceTransport::open_default()?);          // real TPM (usually root-only)
let mut tpm = Tpm::new(SimulatorTransport::connect_default()?);    // swtpm on :2321
# Ok::<(), purecrypto_tpm::Error>(())
```
The README contains a full seal/unseal flow using `Public::ecc_storage_parent`, `Public::sealed_data`, `start_hmac_session` and `Auth::Session`.

**Gotchas:**
- 0.1.0 depends on `purecrypto = "0.6"` (hash/cipher/ec/rng only). Next to a current `purecrypto` 0.9, you get two copies of `purecrypto` in your dependency tree.
- Start swtpm with `--flags startup-clear`, or call `tpm.startup(su::CLEAR)` and ignore the "already initialised" error.

## cacrt

**Repo:** https://github.com/KarpelesLab/cacrt · **Crate:** `cacrt` (crates.io `0.1.2`) · **License:** MIT OR Apache-2.0 (the root data is under MPL-2.0) · **Status:** usable. It is the default trust store for `purecrypto` and `rsurl`.

A curated set of trusted TLS root CAs, embedded as static DER. Roots are addressable by their OpenSSL subject hash name (e.g. `062cdee6.0`) or by the raw subject DER. The crate is `#![no_std]`, never uses `alloc`, is `#![forbid(unsafe_code)]`, and has **zero runtime dependencies**. All parsing and hashing happen in `build.rs`. The set was seeded from Mozilla NSS and is now curated by hand against CA/Browser Forum rules; see `CURATION.md` in the repo.

**Use it when:**
- You need trust anchors on embedded, WASM, or cross-platform targets without reading `/etc/ssl`.
- You are building your own chain validator and need issuer lookup by subject DER.

**Don't use it when / limits:**
- You need the OS trust store, including enterprise-installed roots.
- You need to parse certificates. It only hands out DER; parse with `purecrypto::x509`.
- If you already use `purecrypto` TLS, `RootCertStore::with_embedded_roots()` wraps it for you, so you don't need `cacrt` directly.

**Add it:**
```toml
[dependencies]
cacrt = "0.1"
```

**Key features / cargo features:** it has no features. The API is `all()`, `len()`, `lookup(name)`, `lookup_by_hash(u32)` and `find_by_subject(&[u8])`. Each `Cert` exposes `der()`, `subject_der()`, `subject_hash()`, `seq()`, `label()` and `hash_name()`.

**Example:**
```rust
if let Some(ca) = cacrt::lookup("062cdee6.0") {
    let der: &[u8] = ca.der();
    println!("{} {}", ca.hash_name(), ca.label());
}
for ca in cacrt::all() { let _ = ca.subject_der(); }
```

## psl2

**Repo:** https://github.com/KarpelesLab/psl2 · **Crate:** `psl2` (crates.io `0.1.30`) · **License:** MIT OR Apache-2.0 (the PSL data is under MPL-2.0) · **Status:** production. CI republishes the crate automatically when the upstream Public Suffix List changes.

Mozilla's Public Suffix List as a data trie. It gives you the public suffix, the registrable domain (eTLD+1, i.e. the cookie domain), the subdomain, and whether a suffix is ICANN or PRIVATE. The core is `no_std` and **allocation-free**. There is no `build.rs` or codegen, so it compiles quickly. IDN support is built in through the `idna` crate. It is used by `rsurl`'s cookie jar.

**Use it when:**
- You need cookie scoping, "same site" checks, or domain grouping.
- You want a replacement for the `psl` or `publicsuffix` crates. The `compat` module mirrors the `psl` API.

**Don't use it when / limits:**
- You need to exclude PRIVATE-section suffixes. Both sections are always honored (e.g. `registrable_domain("blogspot.com")` is `None`); use `is_icann()`/`is_private()` to tell them apart.
- `lookup` requires lowercase ASCII/punycode input.

**Add it:**
```toml
[dependencies]
psl2 = "0.1"
# no_std, no alloc: psl2 = { version = "0.1", default-features = false }
```

**Key features / cargo features:**
- `std` (default).
- `alloc` (default): the owned `analyze`, `suffix`, `registrable_domain`, `subdomain` and `is_public_suffix` functions.
- `idna` (default): Unicode input. It is the only external dependency.
- `fast-lookup` (default): about 42 KB of index for roughly 2x faster lookups.
- `psl_version()` returns the bundled list version.

**Example:**
```rust
assert_eq!(psl2::registrable_domain("www.example.co.uk").as_deref(), Some("example.co.uk"));
assert!(psl2::is_public_suffix("co.uk"));

// zero-alloc path, pre-normalized input
let d = psl2::lookup("www.example.co.uk").unwrap();
assert_eq!(d.suffix(), "co.uk");
assert_eq!(d.subdomain(), Some("www"));
```

**Gotchas:** the MSRV is 1.86 because of `idna`. The core without features builds on much older Rust.

## puressh

**Repo:** https://github.com/KarpelesLab/puressh · **Crate:** `puressh` (crates.io `0.1.8`) · **License:** MIT OR Apache-2.0 · **Status:** functional, pre-1.0. It is tested against OpenSSH but has not had an independent audit.

An SSH library and CLI suite in the spirit of libssh. It has a sans-I/O core (`ClientDriver`/`ServerDriver`) with blocking, `futures`, tokio and mio frontends layered on top. **All crypto comes from `purecrypto`**, using `hash, cipher, kdf, rng, rsa, dh, ec, der, mlkem` with no TLS. It has no C dependencies and no `unsafe` outside `ffi`. The protocol core builds for `no_std` + `alloc`.

**Use it when:**
- You need a programmatic SSH client: exec, shell, SFTP, SCP, `-L`/`-R` forwarding, agent/X11 forwarding.
- You need an embedded SSH server.
- You want PQ hybrid key exchange (`mlkem768x25519-sha256`).

**Don't use it when / limits:**
- `hostbased` and `gssapi-with-mic` auth are not supported.
- `PermitTunnel` and external-command `Subsystem` entries are not supported.
- The API may still change before 1.0.

**Add it:**
```toml
[dependencies]
puressh = "0.1"
# async: puressh = { version = "0.1", features = ["tokio"] }
```

**Key features / cargo features:**
- On by default:
  - `std` and `alloc`.
  - `client` and `server`.
  - `compress`: zlib via `compcol`.
  - `pam`: a Linux-only libpam dependency used by `sshd`. Disable default features to drop it.
  - `multichannel`: `SharedClient`, `SftpSession`, and concurrent channels.
- Opt-in: `async` (`AsyncClient` over `futures_io`), `tokio` (`connect_tokio`/`accept_tokio`), `mio` (`MioClient`), `ffi` (a `pcssh_*` C ABI).
- Binaries: `ssh`, `sftp`, `scp`, `sshd` and `ssh-keygen`. They understand `ssh_config` (including `Match`/`Include`) and `known_hosts`.
- `hazmat` exposes the raw protocol layers, with no stability guarantee.

**Example:**
```rust
use puressh::client::{Client, Config};

fn main() -> Result<(), puressh::Error> {
    // Config::insecure() trusts any host key; use Config::with_known_hosts(store) for real use.
    let mut c = Client::connect("example.com:22", Config::insecure())?;
    c.authenticate_password("alice", "hunter2")?;
    let out = c.exec("uname -a")?;
    println!("{} exit={:?}", String::from_utf8_lossy(&out.stdout), out.exit_status);
    Ok(())
}
```

**Gotchas:**
- `Config::insecure()` accepts any host key. Use `Config::with_known_hosts(Arc<Mutex<KnownHosts>>)` in anything real.
- `Client::sftp()` borrows the client. For concurrent channels, use `SharedClient` (the `multichannel` feature).

## httpsd

**Repo:** https://github.com/KarpelesLab/httpsd · **Crate:** `httpsd` (crates.io `0.1.3`) · **License:** MIT · **Status:** usable, early 0.1.x. It is verified against curl for h1, h2 and h3.

An HTTP/1.1, HTTP/2 and HTTP/3 server. It has a sans-I/O protocol core (`proto::H1Conn`) and pluggable runtimes: a thread pool, tokio, or mio. It works as a library (a synchronous `Handler` or `Router`) or as a CLI that serves a directory or a TOML config. Components come from sibling crates:
- **HTTPS and QUIC:** `purecrypto::tls` and `purecrypto` QUIC.
- **Compression, HPACK and QPACK:** `compcol`.
- **ACME:** `rsurl` is used as the ACME HTTP client.

**Use it when:**
- You want an embeddable HTTPS server with no OpenSSL or rustls.
- You want automatic Let's Encrypt certificates (TLS-ALPN-01 or HTTP-01, issued on demand per SNI).
- You want a quick static file server.

**Don't use it when / limits:**
- Handlers are synchronous; there are no async handler traits.
- Request and response bodies are buffered; there is no streaming.
- There is no HTTP/2 server push.
- HTTP/3 needs a static certificate; ACME certificates don't work over h3 yet.
- ACME runs only under the thread-pool runtime.

**Add it:**
```toml
[dependencies]
httpsd = "0.1"
# lean library: httpsd = { version = "0.1", default-features = false, features = ["rt-tokio", "tls", "compress"] }
```
CLI: `cargo install httpsd`. Example invocations:
- `httpsd ./public -l 0.0.0.0:8443 --self-signed`
- `httpsd /srv/www -l 0.0.0.0:443 --http 0.0.0.0:80 --acme-accept-tos --acme-email you@example.com`

**Key features / cargo features:**
- Default: `cli`, `rt-threadpool`, `tls`, `compress`, `h2`, `h3`, `router`, `hardened-fs` (`openat2` `RESOLVE_BENEATH` on Linux).
- `cli` implies `config`, `acme` and `privdrop`.
- Opt-in: `rt-tokio`, `rt-mio`, `acme`, `http` (conversions to and from the `http` crate), `config`, `privdrop`.
- Runtimes: `server.run()`, `run_tokio().await`, `run_mio()`, and `run_h3()` for UDP/QUIC. To serve TCP and h3 together, run them on separate threads.

**Example:**
```rust
use httpsd::{Server, Request, Response, StatusCode, tls::TlsAcceptor};
use httpsd::router::Router;

fn main() -> httpsd::Result<()> {
    let app = Router::new()
        .get("/", |_req: &Request| "hello world")
        .get("/users/:id", |req: &Request| Response::text(format!("user {}", req.param("id").unwrap_or("?"))))
        .post("/users", |_req: &Request| (StatusCode::CREATED, "created"));
    let acceptor = TlsAcceptor::from_pem_files("cert.pem", "key.pem")?; // or TlsAcceptor::self_signed(&["localhost"])?
    Server::bind("0.0.0.0:8443")?.handler(app).tls(acceptor).run()
}
```

**Gotchas:**
- With `default-features = false` you must select a runtime (`rt-*`) and `tls` yourself.
- It depends on full-default `purecrypto`, which includes its CLI features.

## rsurl

**Repo:** https://github.com/KarpelesLab/rsurl · **Crate:** `rsurl` (crates.io `0.1.15`) · **License:** MIT · **Status:** functional across a broad protocol surface, in active development. The API may shift before 1.0.

A pure-Rust implementation of curl. It ships as a Rust library, as a C ABI (`rsurl_*`, with the `ffi` feature), and as the `rsurl` CLI. The repo also contains an unpublished `curl-compat` crate that exposes the libcurl ABI (`libcurl.so.4`).

Protocols: HTTP/1.1, HTTP/2 (including `send_multiplexed`), HTTP/3, FTP(S), SFTP/SCP, WS(S), IMAP/POP3/SMTP, LDAP, MQTT, RTSP, TFTP, DICT, Gopher, `file:`, and BitTorrent. It also supports proxies (HTTP CONNECT, SOCKS4/5), cookies, resumable and segmented downloads, and AWS SigV4.

How it composes with the sibling crates:
- TLS and QUIC come from `purecrypto`.
- The default trust store is the embedded `cacrt` bundle. `--cacert` replaces it; `--capath` adds to it.
- SSH comes from `puressh`, cookie PSL checks from `psl2`, and decompression (gzip, zstd, br, and others) from `compcol`.

**Use it when:**
- You want a synchronous HTTP(S) client with no C dependencies.
- You need non-HTTP schemes through one API (`rsurl::transfer`).
- You need resumable downloads (`rsurl::download`).
- You need a curl-compatible CLI or a libcurl ABI replacement.
- You need a single async API across native and browser WASM (`rsurl::aio`, which uses the Fetch API in the browser).

**Don't use it when / limits:**
- On `wasm32` only `aio` exists; the blocking API is not available.
- HTTP/3 does not interoperate with Cloudflare's QUIC endpoints yet; `--http3` falls back to HTTP/2.
- The CLI does not multiplex multiple URLs.

**Add it:**
```toml
[dependencies]
rsurl = "0.1"
# HTTP-only, no SSH/BitTorrent: rsurl = { version = "0.1", default-features = false, features = ["purecrypto-tls", "idn"] }
```
CLI: `cargo install rsurl`. Its flags are curl-style: `-L`, `-o`, `-O`, `-d`, `-F`, `-x`, `-k`, `--cacert`, `--http2`, `--http3`, and so on.

**Key features / cargo features:**
- Default: `purecrypto-tls`, `idn` (via `intl`), `bittorrent`, `ssh` (pulls in `puressh`).
- Opt-in: `rustls-tls` (rustls + ring + `cacrt`; HTTP/3 still uses purecrypto), `json`, `tokio-rt`, `ffi`.
- Builds for `fullrust`, producing fully static, libc-free Linux binaries.

**Example:**
```rust
let resp = rsurl::get("https://example.com")?;
println!("{} {}", resp.status, String::from_utf8_lossy(&resp.body));

let resp = rsurl::Request::get("https://example.com/api")?
    .header("Accept", "application/json")
    .send()?;

let client = rsurl::Client::new().proxy("socks5h://127.0.0.1:1080")?;
let bytes = client.transfer("ftp://ftp.example.com/pub/file")?;
```

**Gotchas:**
- The README lists system CA bundle paths, but the purecrypto backend defaults to the embedded `cacrt` roots (`RootCertStore::with_embedded_roots`). Pass `--cacert` or `--capath` to use system roots.
- SSH host keys are checked strictly against `known_hosts`. Trust-on-first-use is opt-in via `--ssh-accept-new` or `SshOptions::accept_new`.
- The MSRV is 1.89 because `purecrypto` requires it.

## pktkit-rs

**Repo:** https://github.com/KarpelesLab/pktkit-rs · **Crate:** `pktkit` (crates.io `0.1.7`) · **License:** MIT · **Status:** active development; the API is not yet stable. Most features are complete, with documented TODOs for `ovpn`, `xdp`/`afxdp` and IPv6 accept in `slirp`.

A zero-copy L2/L3 packet toolkit and a Rust port of Go `pktkit`. `Frame` and `Packet` are `#[repr(transparent)]` wrappers over `[u8]`. It also provides typed L4 views, builders that fill in checksums, L2/L3 hubs, pipes, and synchronous callback forwarding with no async runtime.

The default build has **zero dependencies**. `libc` is added only for the OS interfaces. **`purecrypto` is used for all crypto** under `wg` and `ovpn`; the OpenVPN control channel uses `purecrypto` TLS 1.2 and X.509. `cargo-deny` bans ring, aws-lc-rs, openssl-sys and rustls. `full` builds for `wasm32` as a sans-I/O stack.

**Use it when:**
- You are building virtual networks, VPN servers, userspace NAT, or VM networking (QEMU).
- You need packet parsing and building.
- You need a userspace TCP/IP stack (`vtcp`, `slirp`, `vclient`).
- You need AF_XDP capture.

**Don't use it when / limits:**
- `ovpn` lacks tls-crypt/tls-auth and control retransmit timers.
- XDP kernel paths are only tested manually (the tests need root).
- macOS `utun` has not been exercised on real hardware.

**Add it:**
```toml
[dependencies]
pktkit = { version = "0.1", features = ["wg", "slirp"] }   # or "full"
```

**Key features / cargo features:** every feature is off by default.
- Link layer and addressing: `l2adapter` (ARP/NDP), `dhcp`.
- Testing and debugging: `impair` (link impairment), `pcap`.
- Host and VM interfaces: `qemu`, `tuntap`, `afpacket`, `xdp`, `afxdp`.
- TCP/IP stacks: `vtcp` (TCP engine), `slirp` (NAT to real sockets), `vclient` (dial, listen, DNS, HTTP), `nat` (NAT44/NAT64 plus ALGs).
- Tunnels: `wg` (WireGuard), `ovpn` (OpenVPN server).
- `full` enables all of the above.

**Example:**
```rust
use pktkit::build::{build_ipv4, build_udp};
use pktkit::{Packet, Protocol};
use std::net::Ipv4Addr;

let (src, dst) = (Ipv4Addr::new(10, 0, 0, 1), Ipv4Addr::new(10, 0, 0, 2));
let udp = build_udp(src.into(), dst.into(), 5000, 53, b"query");
let buf = build_ipv4(src, dst, Protocol::UDP, 64, &udp);
let pkt = Packet::from_slice(&buf);
assert!(pkt.verify_ipv4_checksum());
assert_eq!(pkt.udp().unwrap().dst_port(), 53);
```

**Gotchas:**
- On `wasm32` you must drive the timers yourself (`vclient::Client::tick`, `wg::Handler::maintenance`, `nat::Nat::sweep`, and others).
- On `wasm32-unknown-unknown` the host must provide the `pktkit.now_ms`, `pktkit.unix_ms` and `purecrypto.random_get` imports.

## spotlib-rs

**Repo:** https://github.com/KarpelesLab/spotlib-rs · **Crates:** `spotlib` (crates.io `0.1.2`) and `spotproto` (crates.io `0.1.2`). `spot-web` holds wasm-bindgen bindings and is unpublished. · **License:** MIT · **Status:** functional and verified against the live Spot network; the API may change.

A pure-Rust client for the **Spot** end-to-end encrypted messaging network, wire-compatible with Go `spotlib`/`spotproto`. Each client has a keychain and a signed Bottle ID card, and its address is `k.<base64url sha256(pubkey)>`. Messages to `k.` addresses are encrypted for the recipient and signed by the sender, so relays only see ciphertext.

It builds on three sibling crates:
- **`bottlers`** (Bottle format) handles encryption and signatures.
- **`purecrypto`** provides the primitives.
- **`rsurl`** handles REST host discovery and the `wss://` split WebSocket transport.

It uses no async runtime; connections run on background threads. It also works in the browser via WASM.

**Use it when:** you need to talk to Spot endpoints, or you want a ready-made E2EE request/response and fire-and-forget messaging layer on top of KarpelesLab infrastructure.

**Don't use it when / limits:**
- It only works with the Spot network. It is not a general messaging protocol.
- The ID card is ephemeral unless you use `spotlib::DiskStore`.

**Add it:**
```toml
[dependencies]
spotlib = "0.1"
# wire format only: spotproto = "0.1"
```

**Key features / cargo features:**
- `spotlib`: the `native` feature (default).
- `spotproto`: packet framing, the CBOR handshake, and the instant-message format. Its only dependencies are `bottlers` and `ciborium`.
- The client API includes `query`, `send_to`, `listen_packet`, blob storage, group membership, ID-card lookup and events.

**Example:**
```rust
use std::time::Duration;

fn main() -> Result<(), spotlib::Error> {
    let t = Duration::from_secs(30);
    let client = spotlib::Client::builder()
        .handler("myendpoint", |msg| Ok(Some(msg.body.clone())))
        .build()?;
    client.wait_online(t)?;
    println!("{}", client.target_id());                     // k.<hash>
    let resp = client.query("k.<target>/endpoint", b"payload", t)?;
    println!("{} bytes", resp.len());
    Ok(())
}
```

**Gotchas:**
- Set `SPOTLIB_DEBUG=1` for connection logs.
- `g-dns.net` hostnames (base32-encoded IPs) are resolved locally by a custom connector.

## rsupd

**Repo:** https://github.com/KarpelesLab/rsupd · **Crate:** `rsupd` (crates.io `0.3.5`) · **License:** MIT · **Status:** complete and in use. It is the Rust successor to Go `goupd`.

Signed release distribution plus in-place auto-updating for Rust binaries. A producer CLI signs a CBOR manifest with an Ed25519 Bottle identity (`bottlers`). The consumer library trusts only a **32-byte fingerprint compiled into the binary**. It verifies the manifest signature, downloads the artifact (zstd via `compcol`), checks its SHA-256 (via `purecrypto`), then atomically swaps the binary and restarts. The HTTP transport is `rsurl`. The crate is `forbid(unsafe_code)` and builds for `fullrust`.

**Use it when:** you want self-updating CLIs or daemons with signature-based trust that doesn't depend on the hosting server or TLS.

**Don't use it when / limits:**
- `HttpTransport` is hardwired to the `dist-go.tristandev.net` host. Implement `rsupd::Transport` for other hosts.
- Manifests have no freshness or expiry, so a network attacker can suppress updates, though not forge them.
- Compromise of the signing key is total.
- The MSRV is **1.95**.

**Add it:**
```toml
[dependencies]
rsupd = "0.3"      # consumer updater (default, no features)
```
Producer CLI: `cargo install rsupd --features _cli`, then:
1. `rsupd id init --project myapp`
2. `rsupd id export --project myapp` (prints the fingerprint hex)
3. `rsupd publish --setup-ci --full`
4. `rsupd check` (a CI gate)

**Key features / cargo features:**
- The default is the consumer library only; `_cli` builds the producer binary.
- Transports: `HttpTransport` and `ZipPackageTransport` (offline/sideload).
- Updater methods: `check()`, `install(&available)`, `update()`, and `spawn_auto_update(immediate)`, which checks hourly.
- It never downgrades. Newer means a higher semver, or an equal version with a newer `date_tag`.

**Example:**
```rust
fn updater() -> rsupd::Result<rsupd::Updater> {
    rsupd::Updater::builder(env!("CARGO_PKG_NAME"), env!("CARGO_PKG_VERSION"))
        .fingerprint_hex("925804220841644e23b6c756b2dc3e611374d08eeb24918fcff0161401da8334")
        .build()
}

fn main() {
    rsupd::honor_startup_delay();
    if let Ok(u) = updater() {
        u.spawn_auto_update(false); // hourly check; installs + restarts
    }
    // ... service ...
}
```

**Gotchas:**
- The channel defaults to `master`, and it must match how you publish.
- To detect rebuilds of the same version, add the `build.rs` that `--setup-ci` generates, and pass `.git_tag(env!("RSUPD_GIT_TAG"))` and `.date_tag(rsupd::date_tag_from_unix(env!("RSUPD_BUILD_UNIX")))`.
- Pass `.auto_restart(false)` to handle the restart yourself.
