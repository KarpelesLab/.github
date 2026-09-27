# Go packages

KarpelesLab publishes a large set of Go modules, most of them pure Go (no cgo) with few or no third-party dependencies. They cover language runtimes (a PHP engine), cloud and peer-to-peer infrastructure, networking stacks, cryptography (including post-quantum), blockchain primitives, filesystems, media codecs, parsers, i18n data and small utilities. The sister org [portablesql](https://github.com/portablesql) holds the SQL toolkit `psql`. Many packages build on each other: `fleet` uses `spotlib`, `gosigner` combines `spotlib` + `hsm` + `authenticode`, `goro` uses `gopcre2` + `gotz`, and `psql` uses `pjson` + `typutil`.

Conventions for this file: module path is `github.com/KarpelesLab/<repo>` unless stated otherwise (checked against each `go.mod`). Add a library with `go get <module>@latest`. "No license file" means the repo has no LICENSE; ask before depending on it. For Rust ports of some of these (strtotime-rs, xuid-rs, pktkit-rs, ...) see the Rust docs.

## Quick pick

| Need | Use |
|------|-----|
| SQL ORM / query builder for MySQL, PostgreSQL and SQLite from one codebase | [`portablesql/psql`](#portablesql-psql) |
| Run PHP code from Go, or a PHP CLI/FPM written in Go | [`goro`](#goro) |
| Drive Node.js processes from Go | [`nodejs`](#nodejs) |
| HTTP + HTTPS + PROXY protocol on one port | [`magictls`](#magictls) |
| Automatic HTTPS on a cloud VM with no setup | [`cloudhttp`](#cloudhttp) |
| Peer-to-peer cluster: discovery, RPC, replicated KV, locks | [`fleet`](#fleet) |
| E2E-encrypted messaging between Go programs | [`spotlib`](#spotlib) |
| Virtual networks, userspace TCP/IP, NAT, WireGuard | [`pktkit`](#pktkit) |
| DNS message parsing/encoding, DNSSEC | [`dns`](#dns) |
| JWT / JWS / JWE (incl. post-quantum) | [`jwt`](#jwt) |
| ML-DSA / SLH-DSA signatures | [`mldsa`](#mldsa), [`slhdsa`](#slhdsa) |
| Sign Windows .exe/.dll in pure Go | [`authenticode`](#authenticode), [`gosigner`](#gosigner) |
| HSM (YubiHSM2, IDPrime smart cards) or TPM keys | [`hsm`](#hsm), [`tpmlib`](#tpmlib) |
| Hash algorithm chosen by name (PHP `hash()` compatible) | [`anyhash`](#anyhash) |
| Threshold ECDSA/EdDSA signatures | [`tss-lib`](#tss-lib) |
| Crypto addresses / tx signing for many chains | [`outscript`](#outscript) |
| Ethereum JSON-RPC | [`ethrpc`](#ethrpc) |
| Read/write SquashFS images | [`squashfs`](#squashfs) |
| Read/write ISO9660 images | [`iso9660`](#iso9660) |
| Copy-on-write file copy (btrfs/xfs) | [`reflink`](#reflink) |
| bzip2 compression (stdlib only decompresses) | [`gobzip2`](#gobzip2) |
| AVIF / WebP encode in pure Go | [`goavif`](#goavif), [`gowebp`](#gowebp) |
| PHP-gd-style image operations | [`gogd`](#gogd) |
| Regex with backreferences / lookaround | [`gopcre2`](#gopcre2) |
| Parse "next friday", "+2 days" | [`strtotime`](#strtotime) |
| Localized strftime | [`strftime`](#strftime) |
| Raw IANA tz transitions | [`gotz`](#gotz) |
| Country / currency / language data | [`countrydb`, `currencydb`, `lngdb`](#reference-data) |
| Prefixed, base32 UUIDs (`user-xxxx-...`) | [`xuid`](#xuid) |
| Collapse duplicate concurrent calls (generic singleflight) | [`unison`](#unison) |
| Loose type conversion (`any` to struct/int/...) | [`typutil`](#typutil) |
| Self-updating Go binaries | [`goupd`](#goupd) |
| Sandbox `npm install` and similar | [`bnpm`](#bnpm) |

---

## portablesql (psql)

**Org:** https://github.com/portablesql · **Docs:** https://portablesql.github.io and https://pkg.go.dev/github.com/portablesql/psql · **License:** MIT (core) · **Status:** active, pre-1.0 (psql v0.5.x)

Portable SQL library for Go: struct-tag object binding, a query builder that renders engine-specific SQL, lifecycle hooks, associations, soft delete, transactions with savepoints and automatic retries, and pgvector search. The same code runs on MySQL/MariaDB, PostgreSQL/CockroachDB and SQLite. Similar scope to GORM, but built on generics and range iterators. Tables are created, and missing columns added, on first use by default.

| Module | Role | Go | Built on |
|--------|------|----|----------|
| `github.com/portablesql/psql` | Core: binding, query builder, tx | 1.24+ | - |
| `github.com/portablesql/psql-mysql` | MySQL / MariaDB driver | 1.24+ | go-sql-driver/mysql (pure Go) |
| `github.com/portablesql/psql-pgsql` | PostgreSQL / CockroachDB driver | 1.25+ | pgx v5 |
| `github.com/portablesql/psql-sqlite` | SQLite driver | 1.25+ | modernc.org/sqlite (pure Go, no cgo) |

**Use it when:** you want one data layer that runs on several SQL engines, for example SQLite in tests and PostgreSQL or MySQL in production.
**Limits:** a feature the engine lacks (for example `RETURNING` or `DISTINCT ON` on MySQL) fails with `psql.ErrNotSupported`. Check it first with `be.Supports(feature)`.

```bash
go get github.com/portablesql/psql github.com/portablesql/psql-sqlite
```

Drivers register themselves in `init()`, so import them with a blank identifier. `psql.New(dsn)` picks the engine from the DSN:
- SQLite: `:memory:`, `file:...`, `sqlite:...`, or a path ending in `.db`, `.sqlite` or `.sqlite3`.
- PostgreSQL: `postgres://...`, `postgresql://...`, or libpq `host=... dbname=...`.
- MySQL: the go-sql-driver format, such as `user:pw@tcp(host:3306)/db?parseTime=true`.

The backend is carried in the `context.Context`.

```go
package main

import (
	"context"
	"fmt"

	"github.com/portablesql/psql"
	_ "github.com/portablesql/psql-sqlite" // or psql-mysql, psql-pgsql
)

type User struct {
	psql.Name `sql:"users"`
	ID        uint64 `sql:",key=PRIMARY"`
	Email     string `sql:",type=VARCHAR,size=255,key=UNIQUE:email"`
	Login     string `sql:",type=VARCHAR,size=128"`
}

func main() {
	be, err := psql.New(":memory:")
	if err != nil {
		panic(err)
	}
	ctx := be.Plug(context.Background()) // all psql calls take this ctx

	// table is created on first use
	if err := psql.Insert(ctx, &User{ID: 1, Login: "Alice", Email: "alice@example.com"}); err != nil {
		panic(err)
	}
	user, err := psql.Get[User](ctx, map[string]any{"ID": uint64(1)})
	if err != nil {
		panic(err)
	}
	user.Login = "Alice Smith"
	_ = psql.Update(ctx, user) // writes only changed columns

	// transaction: ctx inside the callback is bound to the tx
	err = psql.Tx(ctx, func(ctx context.Context) error {
		return psql.Insert(ctx, &User{ID: 2, Login: "Bob", Email: "bob@example.com"})
	})
	if err != nil {
		panic(err)
	}

	users, _ := psql.Fetch[User](ctx, nil, psql.Sort(psql.S("Login", "ASC")), psql.Limit(10))
	q := psql.B().Select().From("users").Where(map[string]any{"Login": "Bob"})
	bobs, _ := psql.RunQueryT[User](ctx, q)
	n, _ := psql.Count[User](ctx, nil)
	fmt.Println(len(users), bobs[0].Email, n) // 2 bob@example.com 2
}
```

**Key API:**
- CRUD: `Insert`, `Get[T]`, `Fetch[T]`, `Update`, `Replace`, `Delete`, `Count[T]`.
- Query builder: `psql.B()`, with `Select`/`Update`/`Insert`/`Delete`, joins, CTEs, upserts and `Returning`. Run it with `.RunQuery(ctx)`, `.ExecQuery(ctx)` or `psql.RunQueryT[T]`, and debug it with `.RenderArgs(ctx)`.
- Transactions: `psql.Tx`, `TxWithOptions`, `BeginTx`.
- Other features: hooks (`BeforeSave`, `AfterInsert`, ...), associations (`belongs_to`, `has_many`, ...), soft delete through a `DeletedAt *time.Time` field, and `psql.Vector`.

**Gotchas:**
- Pass the plugged `ctx` everywhere. A bare `context.Background()` has no backend.
- A `Where` map is keyed by Go field name or column name.
- SQLite paths with other extensions need the `sqlite:` prefix.
- The `docs/` folder in the psql repo has full guides for each feature.

---

## Languages & runtimes

### goro
**Repo:** https://github.com/KarpelesLab/goro · **Module:** `github.com/KarpelesLab/goro` · **License:** BSD-3-Clause · **Status:** active; passes ~98% of the PHP 8.5 test suite

A PHP 8.5 engine in pure Go, with a bytecode VM. It ships SAPIs under `sapi/`: `php-cli`, `php-cgi`, `php-fpm`, `php-httpd` (an HTTP handler) and `php-test`. Uses `gopcre2` for PCRE and `gotz` for timezones.
**Use it when:** you need to run PHP (legacy or user-provided) inside a Go process, sandboxed through `fs.FS`, or want a PHP binary with no C.
Install the CLI with `go install github.com/KarpelesLab/goro/sapi/php-cli@latest`.
**Gotcha:** code that uses `core/compiler` directly must blank-import `core/vm/vmcompiler`.

### gsh
**Repo:** https://github.com/KarpelesLab/gsh · **Module:** `github.com/KarpelesLab/gsh` · **License:** no license file · **Status:** early/minimal

A POSIX-ish shell in Go, meant for running shell scripts inside Go programs without depending on bash or dash.

### nodejs
**Repo:** https://github.com/KarpelesLab/nodejs · **Module:** `github.com/KarpelesLab/nodejs` · **License:** MIT

Finds and runs a system Node.js from Go. `nodejs.New()` returns a factory. The package also provides a process pool with auto-scaling, JS execution, bidirectional IPC, health checks, isolated contexts, and HTTP requests served by JS handlers.
**Use it when:** Go code must run JavaScript, for example for SSR. Requires a Node.js install on the host.

## AI & developer tools

### bnpm
**Repo:** https://github.com/KarpelesLab/bnpm · **Module:** `github.com/KarpelesLab/bnpm` (CLI) · **License:** MIT

A CLI that sandboxes package-manager commands in unprivileged Linux namespaces, applying TOML profiles by command:
- The filesystem is limited to the project directory, plus read-only system paths.
- Install commands can reach only allowlisted registries.
- Build and test commands get no network at all.

`go install github.com/KarpelesLab/bnpm@latest`, then run `bnpm -- npm install`. Linux only.

### aipencil
**Repo:** https://github.com/KarpelesLab/aipencil · **Module:** `github.com/KarpelesLab/aipencil` (CLI) · **License:** MIT

Renders a structured JSON scene description (groups, shapes, text, arrows, and layouts such as `ranked` and `stack`) to deterministic SVG or PNG. The layout is computed automatically. It is a cheap, reproducible way for an AI to produce diagrams. `go install github.com/KarpelesLab/aipencil@latest`.

### teamclaude
**Repo:** https://github.com/KarpelesLab/teamclaude · **Language:** JavaScript (Node.js 20+, no dependencies), **not Go** · **npm:** `@karpeleslab/teamclaude` (1.1.x) · **License:** MIT

A proxy for Claude Code and Codex that pools several accounts: Claude Max, ChatGPT/Codex, API keys and Anthropic-compatible fallbacks.
- Rotates to another account when the session or weekly quota bucket nears its limit, but does not rotate on per-minute rate limits.
- Has a TUI, an opt-in MCP control endpoint, and a MITM forward proxy for hard-coded endpoints.

```bash
npm install -g @karpeleslab/teamclaude
teamclaude login && teamclaude server   # then in another terminal: teamclaude run
```

### mcprun
**Repo:** https://github.com/KarpelesLab/mcprun · **Language:** JavaScript, **not Go** · **npm:** not published (the npm name `mcp-http-client` belongs to an unrelated project) · **License:** no license file

A self-contained Node.js client for MCP servers over HTTP. `npm run build` produces `dist/mcp-client.js`, which exports `MCPClient` and `createClients`. Main calls: `client.initialize(...)`, `listTools()` and `callTool(name, args)`. You can also call tools as `client.tools.<name>(args)`.

## Cloud & infrastructure

### fleet
**Repo:** https://github.com/KarpelesLab/fleet · **Module:** `github.com/KarpelesLab/fleet` · **License:** MIT · Go 1.24+

A peer-to-peer cluster framework:
- Peers discover each other through the Spot protocol and connect over TLS/QUIC, optionally with TPM-backed keys.
- A replicated BoltDB key-value store.
- gob and binary RPC with broadcast patterns.
- Distributed locks.

Entry point: `fleet.New()`, then `agent.WaitReady()`.

### clouddb
**Repo:** https://github.com/KarpelesLab/clouddb · **Module:** `github.com/KarpelesLab/clouddb` · **License:** MIT · **Status:** experimental

A decentralized store for JSON objects, kept locally in LevelDB. It replicates a timestamped journal and globally indexed keys. There are no ACID guarantees and no transaction isolation.

### cloudhttp
**Repo:** https://github.com/KarpelesLab/cloudhttp · **Module:** `github.com/KarpelesLab/cloudhttp` · **License:** MIT

`cloudhttp.Serve(mux)` starts HTTP and HTTPS servers with a valid certificate on an automatic `.g-dns.net` hostname. It needs a public IP and port 443 open. Intended for cloud VMs (AWS, GCP, ...).

### cloudinfo
**Repo:** https://github.com/KarpelesLab/cloudinfo · **Module:** `github.com/KarpelesLab/cloudinfo` · **License:** MIT

Detects the current cloud provider and fetches instance information, independent of the provider.

### lambda
**Repo:** https://github.com/KarpelesLab/lambda · **Module:** `github.com/KarpelesLab/lambda` · **License:** MIT

This is a **lambda calculus** library, not AWS Lambda. It provides Church encodings, reduction, and Tromp diagrams as Unicode text, SVG, or animated SVG (`lambda.Diagram`, `lambda.DiagramSVG`).

## Networking & protocols

### magictls
**Repo:** https://github.com/KarpelesLab/magictls · **Module:** `github.com/KarpelesLab/magictls` · **License:** MIT · stdlib only

A drop-in replacement for `tls.Listen` that detects TLS or plaintext and PROXY v1/v2 headers on the same port. It can route connections by ALPN and accepts custom filters. It only works for protocols where the client speaks first and sends at least 16 bytes, so HTTP, TLS and gRPC work; SMTP and IMAP work only with the `ForceTLS` filter.

```go
ln, err := magictls.Listen("tcp", ":8080", tlsConfig) // HTTP and HTTPS on one port
http.Serve(ln, handler)
```

### dns
**Repo:** https://github.com/KarpelesLab/dns · **Module:** `github.com/KarpelesLab/dns` · **License:** no license file

Parses and encodes DNS wire-format messages in pure Go, with EDNS0 and hardening against malformed packets. It has two subpackages: `dnsmsg` for messages and `dnssec` for signing and verifying RRsets and computing DS records. Parsing has no dependencies.

### pktkit
**Repo:** https://github.com/KarpelesLab/pktkit · **Module:** `github.com/KarpelesLab/pktkit` · **License:** MIT

A zero-copy L2/L3 packet toolkit for building virtual networks: L2 and L3 hubs, an L2 adapter with ARP/NDP/DHCP, and a DHCP server. Subpackages:
- `slirp`: userspace NAT to the real network.
- `vclient`: `Dial`, `Listen`, DNS and `http.Client` over a virtual network.
- `vtcp`: an RFC-compliant TCP engine.
- `nat`: NAT with ALGs and NAT64.
- `wg`: WireGuard.

**Use it when:** you need VM or container networking, test harnesses, or multi-tenant virtual networks without root.

### slirp
**Repo:** https://github.com/KarpelesLab/slirp · **Module:** `github.com/KarpelesLab/slirp` · **Status:** **deprecated**. Use `pktkit` (its `slirp` subpackage) instead.

### pppoeproxy
**Repo:** https://github.com/KarpelesLab/pppoeproxy · **Module:** `github.com/KarpelesLab/pppoeproxy` (CLI) · **License:** MIT

Tunnels PPPoE discovery and session frames between networks over TCP. It has three modes:
- Client mode captures PPPoE frames on a local interface and forwards them.
- Server mode forwards them to the real PPPoE server, with IP-based access control.
- Tunnel mode terminates PPPoE and PPP in-process and exposes a tun interface, so `pppd` is not needed.

Linux, raw sockets.

### spotlib
**Repo:** https://github.com/KarpelesLab/spotlib · **Module:** `github.com/KarpelesLab/spotlib` · **License:** MIT

Client for the Spot network, which carries end-to-end encrypted messages between identities (ID cards). It supports request/response and fire-and-forget messages, and offers a `net.PacketConn` interface and status events. `fleet` and `gosigner` both use it.

### smartremote
**Repo:** https://github.com/KarpelesLab/smartremote · **Module:** `github.com/KarpelesLab/smartremote` · **License:** MIT

Opens a remote HTTP file as a local `io.ReaderAt`/`io.Seeker`. It downloads only the 64 KB blocks you read, using Range requests, caches them locally, and can resume through `.part` files. It fills remaining gaps in the background when idle. Good for reading, for example, a ZIP central directory without downloading the whole file.

## Cryptography & security

### jwt
**Repo:** https://github.com/KarpelesLab/jwt · **Module:** `github.com/KarpelesLab/jwt` · **License:** MIT · Go 1.24+

JWS and JWE with no external dependencies.
- Signing: HMAC, RSA, RSA-PSS, ECDSA, Ed25519, ML-DSA and SLH-DSA.
- Encryption: RSA-OAEP, ECDH-ES, AES-KW and ML-KEM.
- JWK support.
- Keys can be any `crypto.Signer` or `crypto.Decrypter`, such as an HSM or KMS.

```go
import _ "crypto/sha256" // hash implementations must be linked in

tok := jwt.New(jwt.HS256)
tok.Payload().Set("iss", "myself")
tok.Payload().Set("exp", time.Now().Add(time.Hour).Unix())
signed, err := tok.Sign(rand.Reader, key)

parsed, err := jwt.ParseString(signed)
err = parsed.Verify(jwt.VerifyAlgo(jwt.HS256), jwt.VerifySignature(key), jwt.VerifyExpiresAt(time.Now(), false))
```

### jwttool
**Repo:** https://github.com/KarpelesLab/jwttool · **Module:** `github.com/KarpelesLab/jwttool` (CLI) · **License:** no license file

An internal CLI that issues node JWTs signed with an HSM key (`HSM=yubihsm2 CLUSTER=... jwttool gen <name> <pubkey> <days>`).

### mldsa
**Repo:** https://github.com/KarpelesLab/mldsa · **Module:** `github.com/KarpelesLab/mldsa` · **License:** MIT

Pure-Go ML-DSA (FIPS 204) at levels 44, 65 and 87, validated against the NIST ACVP vectors. It implements `crypto.Signer` and `crypto.MessageSigner`. Stdlib only.

### slhdsa
**Repo:** https://github.com/KarpelesLab/slhdsa · **Module:** `github.com/KarpelesLab/slhdsa` · **License:** MIT · Go 1.24+

Pure-Go SLH-DSA (FIPS 205), validated against the NIST vectors. The API is `GenerateKey(rand, slhdsa.SHA2_128f)`, `Sign` and `Verify`.

### authenticode
**Repo:** https://github.com/KarpelesLab/authenticode · **Module:** `github.com/KarpelesLab/authenticode` · **License:** MIT

Authenticode signing for PE32/PE32+ files in pure Go: no cgo and no osslsigncode. Optional RFC 3161 timestamping. The signer is any `crypto.Signer` that also provides `Certificate()` and `CertificateChain()`; `hsm` keys qualify. Call: `authenticode.Sign(peBytes, signer, authenticode.SignOptions{Hash: crypto.SHA384, TSAURL: "..."})`.

### gosigner
**Repo:** https://github.com/KarpelesLab/gosigner · **Module:** `github.com/KarpelesLab/gosigner` · **License:** MIT

Signs Windows binaries remotely using a USB code-signing token.
- A daemon on the host that holds the token signs with `hsm` + `authenticode`, talking to the token through pcscd.
- Clients send it AES-GCM-encrypted requests over the Spot network.

Client usage: `go run github.com/KarpelesLab/gosigner/cli/signreq@latest ... -o out.signed.exe in.exe`.

### hsm
**Repo:** https://github.com/KarpelesLab/hsm · **Module:** `github.com/KarpelesLab/hsm` · **License:** no license file

A single `HSM` interface over several backends:
- YubiHSM2, through its HTTP connector (SCP03).
- Thales/Gemalto IDPrime smart cards, such as the SafeNet eToken 5110+, in pure Go with no PKCS#11.
- A BoltDB-backed software HSM for development.

It signs with ECDSA, EdDSA or RSA, and stores certificates.

### tpmlib
**Repo:** https://github.com/KarpelesLab/tpmlib · **Module:** `github.com/KarpelesLab/tpmlib` · **License:** MIT

Uses a local TPM for ECDSA signing, ECDH, hardware RNG and attestation, and creates BottleFmt ID cards and keychains.

### anyhash
**Repo:** https://github.com/KarpelesLab/anyhash · **Module:** `github.com/KarpelesLab/anyhash` · **License:** MIT · stdlib only

60 hash algorithms selected by name, such as `anyhash.New("SHA-256")`, with names and output matching PHP `hash()`. Hash state can be cloned with `Clone()`, and writing may continue after `Sum()`. Also provides `NewHMAC` and `NewHKDF`.

### lnp22
**Repo:** https://github.com/KarpelesLab/lnp22 · **Module:** `github.com/KarpelesLab/lnp22` · **License:** MIT · **Status:** research

An implementation of the LNP22 lattice-based NIZK framework (CRYPTO 2022). It includes BDLOP commitments, linear and range proofs, proof composition, NTRU trapdoors and Gaussian sampling.

### blindsig
**Repo:** https://github.com/KarpelesLab/blindsig · **Module:** `github.com/KarpelesLab/blindsig` · **License:** MIT

Five blind signature schemes:
- RSA (Chaum, with full-domain hashing).
- Ed25519 Schnorr.
- secp256k1 Schnorr (BIP-340).
- BDHKE, as used for ecash mints.
- BLNS23, post-quantum.

### tmpsecfile
**Repo:** https://github.com/KarpelesLab/tmpsecfile · **Module:** `github.com/KarpelesLab/tmpsecfile` · **License:** MIT · Go 1.25+

A temporary file that has no name on disk (`O_TMPFILE` on Linux) and is AES-256-CTR encrypted with a per-file key held only in memory. It supports random access and sparse regions. Use it to stage sensitive data that must not survive the process.

### vncpasswd
**Repo:** https://github.com/KarpelesLab/vncpasswd · **Module:** `github.com/KarpelesLab/vncpasswd`

Encrypts and decrypts VNC `~/.vnc/passwd` files, which use a DES-based obfuscation.

## Blockchain & wallets

### tss-lib
**Repo:** https://github.com/KarpelesLab/tss-lib · **Module:** `github.com/KarpelesLab/tss-lib/v2` (note the `/v2`) · **License:** MIT

A fork of Binance's tss-lib: {t,n}-threshold ECDSA (GG18) and EdDSA, covering key generation, signing and resharing. Adds an experimental `mldsatss` package for threshold ML-DSA-44. It is wire-compatible with `tsslib-rs`. Read the security notes in the README before using it.

### outscript
**Repo:** https://github.com/KarpelesLab/outscript · **Module:** `github.com/KarpelesLab/outscript` · **License:** MIT

Builds output scripts and addresses, and builds and signs transactions, for Bitcoin-family chains, EVM, Solana, Massa and more. Example: `outscript.New(pubKey).Address("p2wpkh", "bitcoin")`.

### ethrpc
**Repo:** https://github.com/KarpelesLab/ethrpc · **Module:** `github.com/KarpelesLab/ethrpc` · **License:** MIT

A lightweight Ethereum JSON-RPC client: `rpc := ethrpc.New(url)`, then `ethrpc.ReadUint64(rpc.Do("eth_blockNumber"))`, with positional or named arguments.

### secp256k1
**Repo:** https://github.com/KarpelesLab/secp256k1 · **Module:** `github.com/KarpelesLab/secp256k1` · **License:** ISC

Optimized, constant-time secp256k1 arithmetic and key handling, extracted from decred.

### edwards25519
**Repo:** https://github.com/KarpelesLab/edwards25519 · **Status:** **decommissioned / unmaintained**. Merged Ed25519 internals; do not use in new code.

### base58
**Repo:** https://github.com/KarpelesLab/base58 · **Module:** `github.com/KarpelesLab/base58` · **License:** MIT

Base58 with no dependencies. Provides the Bitcoin and Flickr alphabets (`base58.Bitcoin.Encode`), custom alphabets, and an O(n) chunked mode for large payloads.

### bech32m
**Repo:** https://github.com/KarpelesLab/bech32m · **Module:** `github.com/KarpelesLab/bech32m` · **License:** MIT

Bech32 (BIP-173), Bech32m (BIP-350) and CashAddr, plus segwit helpers (`SegwitAddrDecode`).

Rust-only blockchain repos listed in the profile: `libwallet`, `erigon-seg`, `chiefsplitter`.

## Filesystems & storage

### squashfs
**Repo:** https://github.com/KarpelesLab/squashfs · **Module:** `github.com/KarpelesLab/squashfs` · **License:** MIT

Reads and writes SquashFS images in pure Go.
- Reading implements `fs.FS`, `ReadDirFS` and `StatFS`; FUSE is available with the `fuse` build tag.
- Writing works from an `fs.FS` or programmatically.
- Only gzip is built in. Build tags `xz` and `zstd` add those codecs, or you can call `RegisterCompHandler`.
- CLI: `go install github.com/KarpelesLab/squashfs/cmd/sqfs@latest`.

```go
out, _ := os.Create("out.squashfs")
w, err := squashfs.NewWriter(out)
err = w.AddFS(os.DirFS("src"))
err = w.Finalize()
out.Close()

sqfs, err := squashfs.Open("out.squashfs")
defer sqfs.Close()
data, err := fs.ReadFile(sqfs, "dir/file.txt")
```

### iso9660
**Repo:** https://github.com/KarpelesLab/iso9660 · **Module:** `github.com/KarpelesLab/iso9660` · **License:** Apache-2.0

Reads and creates ISO9660 images, forked from kdomanski/iso9660. Reading supports Rock Ridge and SUSP. Writing supports El Torito boot, hard-link deduplication and streaming output. Joliet is not supported.

### vfs
**Repo:** https://github.com/KarpelesLab/vfs · **Module:** `github.com/KarpelesLab/vfs` · **License:** MIT

A virtual filesystem abstraction that exposes key/value stores such as S3 as real filesystems that support partial writes.

### reflink
**Repo:** https://github.com/KarpelesLab/reflink · **Module:** `github.com/KarpelesLab/reflink` · **License:** MIT

Copy-on-write file copy. Supported: btrfs, and xfs mounted with `reflink=1`, both on Linux. macOS, Windows and Solaris are not implemented.
- `reflink.Always(src, dst)` fails if the filesystem does not support reflinks.
- `reflink.Auto(src, dst)` falls back to a regular copy.
- `reflink.Reflink(dstFile, srcFile, fallback)` works on open file handles.

### gobzip2
**Repo:** https://github.com/KarpelesLab/gobzip2 · **Module:** `github.com/KarpelesLab/gobzip2`

bzip2 compression and decompression in pure Go, with parallel support, ported from bzip2 1.0.8. Use `gobzip2.NewWriter(w)` and the matching reader.

### gzscan
**Repo:** https://github.com/KarpelesLab/gzscan · **Module:** `github.com/KarpelesLab/gzscan` (CLI) · **License:** MIT

A multi-threaded scanner that finds gzip streams inside files or block devices, for data recovery. `go install github.com/KarpelesLab/gzscan@latest`.

## Media & audio

### goavif
**Repo:** https://github.com/KarpelesLab/goavif · **Module:** `github.com/KarpelesLab/goavif` · **License:** MIT

A pure-Go AVIF/AVIS codec, with its own AV1 implementation and no cgo. It encodes and decodes still images and animations at 8, 10 and 12 bits, with alpha, 4:2:0, 4:2:2 and 4:4:4 chroma, and HDR.

### gowebp
**Repo:** https://github.com/KarpelesLab/gowebp · **Module:** `github.com/KarpelesLab/gowebp` · **License:** MIT

A pure-Go WebP encoder, lossless (VP8L) by default and lossy (VP8) with `Options.Lossy`. It supports animation and alpha. Decoding is delegated to `golang.org/x/image`.

### gogd
**Repo:** https://github.com/KarpelesLab/gogd · **Module:** `github.com/KarpelesLab/gogd` · **License:** MIT

Implements the image operations of PHP's gd extension in native Go, on top of `image/draw`. No cgo and no libgd.

### mblur
**Repo:** https://github.com/KarpelesLab/mblur · **Module:** `github.com/KarpelesLab/mblur` · **License:** GPL-3.0 (note the copyleft)

A multi-threaded port of ImageMagick's MotionBlurImage.

### avgo / ffprobe / hlsmaker
- **avgo** (https://github.com/KarpelesLab/avgo, no license file): a fork of joy4/joy5 audio/video container code. Minimal docs.
- **ffprobe** (https://github.com/KarpelesLab/ffprobe, MIT): probes media files using the ffmpeg tools, which must be installed.
- **hlsmaker** (https://github.com/KarpelesLab/hlsmaker, MIT): converts video to multi-resolution HLS via ffmpeg. It uses a custom layout served by KLB's own server.

### static-opus / static-portaudio
cgo bindings that bundle their C library, so no system library is needed.

**Their `go.mod` keeps the upstream module path:** `github.com/hraban/opus` and `github.com/gordonklaus/portaudio`. Use them through a `replace` directive, for example:

```
replace github.com/hraban/opus => github.com/KarpelesLab/static-opus <version>
```

They are the only cgo packages in this list.

## Data formats & parsing

### gopcre2
**Repo:** https://github.com/KarpelesLab/gopcre2 · **Module:** `github.com/KarpelesLab/gopcre2` · **License:** MIT

PCRE2 in pure Go: backreferences, lookaround, atomic groups, possessive quantifiers, recursion and Unicode properties. The API mirrors `regexp`: `MustCompile`, `MatchString`, `FindStringSubmatch`, ... Unlike RE2, matching is not guaranteed to be linear-time.

### pjson
**Repo:** https://github.com/KarpelesLab/pjson · **Module:** `github.com/KarpelesLab/pjson` · **License:** BSD-3-Clause

A fork of `encoding/json` that adds two things:
- `MarshalContext(ctx, v)`, which passes a context to `MarshalContextJSON` methods.
- Group marshaling, which batches the data lookups of many objects (for example one SQL `IN` query) before rendering.

### gomailparse
**Repo:** https://github.com/KarpelesLab/gomailparse · **Module:** `github.com/KarpelesLab/gomailparse` · **License:** MIT

A single-pass streaming MIME parser. It records headers and byte offsets (`StartPos`, `BodyPos`, `EndPos`) and never buffers bodies, which makes it good for indexing mailboxes.

### ini / csscolor / tpl / graphql
- **ini** (MIT): INI read/write via `io.ReaderFrom`/`io.WriterTo`, case-insensitive, with a thread-safe `IniSafe` wrapper.
- **csscolor** (MIT): parses CSS color strings into `color.Color`.
- **tpl** (MIT): a legacy template engine compatible with an older PHP engine. Use it only for KLB legacy templates.
- **graphql** (MIT): a GraphQL query parser. Marked experimental.

## Internationalization

### strtotime
**Repo:** https://github.com/KarpelesLab/strtotime · **Module:** `github.com/KarpelesLab/strtotime` · **License:** MIT

A PHP-compatible `strtotime()`. It parses relative expressions ("tomorrow", "next friday", "+2 days"), ISO 8601, RFC 2822, US and European formats, and timezones. Call it as `t, err := strtotime.StrToTime("next friday")`.

### strftime
**Repo:** https://github.com/KarpelesLab/strftime · **Module:** `github.com/KarpelesLab/strftime` · **License:** MIT

A fast strftime localized by BCP 47 language tag: `strftime.Format(language.French, "%c", t)`, `strftime.EnFormat(...)`, or `strftime.New(lang)`.

### gotz
**Repo:** https://github.com/KarpelesLab/gotz · **Module:** `github.com/KarpelesLab/gotz` · **License:** MIT

Embeds IANA tzdata and exposes what `time.Location` keeps private: transitions, zone types, POSIX TZ rules and leap seconds. Call `gotz.Load("America/New_York")`.

### goicu
**Repo:** https://github.com/KarpelesLab/goicu · **License:** no license file

ICU-compatible features in pure Go. Subpackages: `transliterate`, which takes ICU transform IDs such as `"Hiragana-Katakana;Fullwidth-Halfwidth"`, and `breakiter`.

### Reference data
- **countrydb** (LGPL-2.1): ISO 3166-1 countries with translations. Examples: `countrydb.France`, `countrydb.ByAlpha2[code]`.
- **currencydb** (LGPL-2.1): currencies, with `currencydb.EUR`, `currencydb.All[code]` and `currencydb.Country["US"]`.
- **lngdb** (no license file): extra information per language, complementing `x/text/language`.
- **flagemoji** (Unlicense): converts an ISO alpha-2 code to a flag emoji. It is also published on npm as `flagemoji`.
- **ibanlib** (MIT): parses, validates and generates IBANs using per-country rules.

## Hardware

- **streamdeck** (MIT): Elgato/Corsair Stream Deck control without the vendor software. No cgo, Linux only.
- **hid** (MIT): a pure-Go HID driver with no dependencies. Linux 386/amd64 only.
- **pixoo64** (MIT): sends images and commands to a Divoom Pixoo64 over the LAN, by IP address.
- **intel-dcapd** (https://github.com/KarpelesLab/intel-dcapd): a single-binary replacement, written in Go, for Intel's SGX DCAP PCCS caching service. Unofficial. Install with `make install` and an Intel API key.
- `usbmagic` is Rust and `selecard` is Python; see their repos.

## Go utilities

### xuid
**Repo:** https://github.com/KarpelesLab/xuid · **Module:** `github.com/KarpelesLab/xuid` · **License:** BSD-3-Clause

UUID-sized IDs with a type prefix, encoded in base32 (for example `user-zha364-5mdr-dmzk-gsil-g73eiu7u`). UUIDv7 by default. Implements `sql.Scanner`, `driver.Valuer` and JSON. Create one with `xuid.New("user")` and parse it with `xuid.Parse(s)`. These are the ID format used by the KLB platform.

### unison
**Repo:** https://github.com/KarpelesLab/unison · **Module:** `github.com/KarpelesLab/unison` · **License:** MIT

A generic singleflight: `unison.Group[K, V]` runs one call per key while concurrent callers share its result. Adds time-based caching and batching helpers.

### typutil
**Repo:** https://github.com/KarpelesLab/typutil · **Module:** `github.com/KarpelesLab/typutil` · **License:** MIT

Flexible conversion between types (`As`, `Assign`), `map[string]any` to struct with JSON tags, struct validation, and dynamic function calls with argument conversion.

### goupd
**Repo:** https://github.com/KarpelesLab/goupd · **Module:** `github.com/KarpelesLab/goupd` · **License:** MIT

Auto-updater for binaries built with KLB's make-go pipeline. Calling `goupd.AutoUpdate(false)` checks for updates hourly or on SIGHUP, supports release channels, and replaces the binary atomically. Only useful with that release infrastructure. The Rust successor is `rsupd`.

### Small utilities
- **uhash** (MIT, CLI): runs many hash algorithms in parallel over one read of stdin, a file or a URL. `go install github.com/KarpelesLab/uhash@latest`.
- **bitmap** (no license file): bitmaps, with atomic operations.
- **ringbuf** (MIT): a thread-safe ring buffer with multiple, optionally blocking, readers. Useful as a log buffer.
- **weak** (MIT): a generic weak-reference `weak.Map` and `weak.Ref`. Since Go 1.24, prefer the stdlib `weak` package for new code.
- **textutil** (MIT): word wrapping, forked from mitchellh/go-wordwrap.
- **rndstr** (BSD-3-Clause, Go 1.24+): random strings. Use `SimpleReader` with `crypto/rand.Reader` for secrets.
