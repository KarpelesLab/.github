# Blockchain & Wallets (Rust)

Pure-Rust crates for building multi-chain cryptocurrency wallets, all built on
[`purecrypto`](https://github.com/KarpelesLab/purecrypto) for their cryptography (no C code, no OpenSSL). `tsslib`
provides threshold signing (MPC), `outscript` provides addresses, scripts and transaction building/signing for many
chains, `ethrpc-rs` talks JSON-RPC to EVM nodes, and `zanolib` covers Zano. `libwallet` is the application that
ties them together (a TSS wallet backend exposed over C FFI/WASM). The rest are narrower: `evmabiless` (recovers the
ABI from EVM bytecode, decodes calldata), `erigon-seg` (Erigon 3 snapshot files) and `chiefsplitter` (an on-chain
Solana program). `outscript` and `ethrpc-rs` are ports of Go libraries in KarpelesLab
([outscript](https://github.com/KarpelesLab/outscript), [ethrpc](https://github.com/KarpelesLab/ethrpc)). `tsslib`
started as a port of Go [tss-lib](https://github.com/KarpelesLab/tss-lib) but no longer documents compatibility with it.

## Quick pick
| Need | Use |
|------|-----|
| Threshold (t-of-n) signatures: FROST Ed25519/ristretto255, DKLs23 ECDSA, threshold ML-DSA | [`tsslib`](#tsslib-rs) |
| Threshold BIP340 Schnorr / Bitcoin Taproot key-path signing (FROST secp256k1) | [`tsslib`](#tsslib-rs) (`frostsecp256k1tss`) |
| Migrate existing GG18/GG20 ECDSA or GG18-style EdDSA key shares | [`tsslib`](#tsslib-rs) (`ecdsatss`, `eddsatss`) |
| Derive addresses from a pubkey (BTC-family, EVM, Solana, Cardano, Massa, Tron, Zcash) | [`outscript`](#outscript-rs) |
| Parse/validate a crypto address for many chains | [`outscript`](#outscript-rs) |
| Build and sign BTC / PSBT / Taproot / EVM / Solana / Cardano / Tron / Zcash-transparent transactions | [`outscript`](#outscript-rs) |
| Move PSBTs as QR codes (UR / BBQr) | [`outscript`](#outscript-rs) |
| no_std / no-alloc transaction signing (hardware wallet, embedded) | [`outscript`](#outscript-rs) |
| Async JSON-RPC to Ethereum/EVM nodes (native Tokio or browser WASM) | [`ethrpc-rs`](#ethrpc-rs) |
| EVM chain metadata (names, explorers, native currency) without networking | [`ethrpc-rs`](#ethrpc-rs) (`default-features = false`) |
| `eth_call` a contract with ABI encode/decode | [`ethrpc-rs`](#ethrpc-rs) |
| Guess the ABI of a deployed contract from its bytecode | [`evmabiless`](#evmabiless) |
| Decode EVM tx calldata for display (no_std, no alloc) | [`evmabiless`](#evmabiless) |
| Read/write/merge Erigon 3 `.kv` / `.bt` / `.kvei` snapshot files | [`erigon-seg`](#erigon-seg) |
| Zano: addresses, offline signing, deposit scanning, sending, MPC spend key | [`zanolib`](#zanolib) |
| Full TSS wallet backend for a mobile/desktop/web app (Dart/Flutter client) | [`libwallet`](#libwallet) |
| On-chain SOL/SPL fee splitter (Solana program) | [`chiefsplitter`](#chiefsplitter) |

## tsslib-rs

**Repo:** https://github.com/KarpelesLab/tsslib-rs · **Crate:** `tsslib` (crates.io `0.2.13`) · **License:** MIT · **Status:** usable. FROST and DKLs23 are the main paths. `mldsatss`, `ecdsatss` and `eddsatss` are experimental and unaudited.

A pure-Rust threshold signature library. It started as a port of the Go [`tss-lib`](https://github.com/KarpelesLab/tss-lib),
but upstream dropped all references to Go tss-lib in October 2026 (commit b835506), so do not assume compatibility with
Go nodes. The JSON message and key-share formats are stable across releases. The crate is `#![no_std]` + `alloc`
(the default `std` feature adds OS randomness, blocking `wait()` and OS locks), has `#![forbid(unsafe_code)]`, and does
no field arithmetic of its own, since all group and lattice math comes from `purecrypto`. Edition 2024, MSRV 1.89.

| Module / feature | Scheme | Output |
|---|---|---|
| `frosttss` | FROST(Ed25519, SHA-512), RFC 9591 | Ed25519 signatures |
| `frostristretto255tss` | FROST(ristretto255, SHA-512) | ristretto255 Schnorr |
| `frostsecp256k1tss` | FROST(secp256k1, SHA-256), RFC 9591 | BIP340 Schnorr, optional BIP341 Taproot tweak |
| `dklstss` | DKLs23 threshold ECDSA, secp256k1 | ECDSA (r, s, v) |
| `mldsatss` | Threshold ML-DSA-44 (FIPS 204), `2 ≤ t ≤ n ≤ 6` | ML-DSA-44 signatures |
| `ecdsatss` | Legacy GG18/GG20 (Paillier + MtA) | ECDSA; loads legacy GG18/GG20 save files for migration |
| `eddsatss` | Legacy GG18-style threshold EdDSA | Ed25519; for migrating legacy GG18-style keys |

**Use it when:** you need t-of-n keygen, signing, resharing or refresh where no single host holds the key; you need
HD derivation of threshold keys; you need threshold Taproot signing; or you must migrate existing GG18/GG20 key shares.
**Don't use it when / limits:** you need peer authentication, which you must supply through your `MessageBroker`
transport. Don't use `mldsatss` in production (the README calls it an academic-grade prototype). Use `ecdsatss` and
`eddsatss` only to migrate existing keys; new deployments should use `dklstss` or `frosttss`.

**Add it:**
```toml
[dependencies]
tsslib = "0.2"   # or pick protocols: { version = "0.2", default-features = false, features = ["std", "json", "dklstss"] }
purecrypto = { version = "0.9", default-features = false, features = ["std", "rng"] }  # for OsRng
```

**Key features / cargo features:**
- Each protocol has a like-named feature, and all are on by default, as are `std` and `json`. Without `std`, register
  entropy with `rng::set_entropy_source`.
- `tss` core: `PartyId`, `Parameters` / `ReSharingParameters`, the `MessageBroker` / `MessageReceiver` traits,
  `JsonMessage`, and `TssError`, which identifies the culprit through `culprits()`.
- Networked protocols run as broker-driven parties: `frosttss::Keygen`, `dklstss::KeygenParty`, `SigningParty`,
  `CheckedSigningParty`, `ResharingParty`, `RefreshParty`. Each has `new(params)`, then `wait()` / `try_result()`.
- `dklstss` also has a synchronous in-process API (`keygen`, `sign`, `sign_checked`, `reshare`, `refresh`), offline
  presigning (`presign` / `sign_with_presign`, which enforces single use), and HD signing
  (`derive_child`, `derive_and_sign`, `sign_with_tweak`).
- `KeyImageParty` (in `frosttss` and `dklstss`) is a threshold PRF that serves as the building block for
  hardened derivation.
- `dklstss::PairSetupParty` (or `setup_pair`) rebuilds a key's pairwise OT-extension state one pair at a time, so that
  state can be stored apart from the key core.
- `frostsecp256k1tss` outputs 64-byte BIP340 signatures, optionally under a BIP341 output key (key-path only or
  committing to a script tree) and/or a non-hardened BIP32 child. `import_key` brings in an existing secp256k1 key to reshare.
- Two encodings: JSON (`to_json` / `from_json`, `json` feature) and the compact binary format of `tsslib::wire`
  (`write_to` / `read_from`, `to_bytes` / `from_bytes`), which is always available and needs no serde_json. Binary is
  2-3.5x smaller (a DKLs23 key is about 90 KB as JSON, 26 KB binary).

**Example** (in-process DKLs23 2-of-3, from the crate's tests):
```rust
use purecrypto::rng::OsRng;
use tsslib::dklstss::{keygen, sign};
use tsslib::tss::PartyId;

let ids = PartyId::sort(
    (1..=3u8).map(|i| PartyId::new(i.to_string(), format!("P{i}"), vec![i])).collect(),
    0,
);
let keys = keygen(3, 1, &ids, &mut OsRng)?;          // n=3, t=1: any t+1 = 2 parties sign
let sig = sign(&keys, &[0, 2], &msg_hash, &mut OsRng)?; // sig.r, sig.s (32-byte BE), sig.v
let saved = keys[0].to_json()?;                        // stable JSON save format (or to_bytes() for binary)
```

**Gotchas:**
- Threshold convention: `t` is the polynomial degree, so signing needs `t + 1` parties, and `keygen` requires `1 ≤ t < n`.
- DKLs keygen has a documented weakness when `n ≤ 2t`: colluding last-moving dealers can bias the key. Either run
  keygen only among non-colluding parties or use `n > 2t`. Fixing it changes the wire format, so it needs a versioned
  protocol change, which has not happened yet.
- The default `dklstss` signing path has a selective-failure leak of about one bit per aborted session. Its wire format
  is kept fixed because existing peers depend on it. Cap retries, and reshare after repeated unexplained aborts. Alternatively,
  use `sign_checked` / `CheckedSigningParty`, which costs about 2x and catches inconsistent deviations
  but, per the source, does not fully close the one-bit leak.
- The README's install snippet says `version = "0.3"`, but crates.io is at 0.2.13. Use `0.2`.
- `purecrypto` must resolve to 0.9 across your whole dependency graph, because its types cross crate boundaries.

## outscript-rs

**Repo:** https://github.com/KarpelesLab/outscript-rs · **Crate:** `outscript` (crates.io `0.2.6`) · **License:** MIT · **Status:** usable. It is actively developed and verified against BIP-174 test vectors.

A Rust port of Go [`outscript`](https://github.com/KarpelesLab/outscript). It generates output scripts and
addresses, parses addresses, and builds and signs transactions for Bitcoin-family chains (BTC, BCH, LTC, DOGE,
Namecoin, Monacoin, Dash, Electraproto), EVM, Solana, Cardano, Massa, Tron and Zcash (transparent only). It is
`#![no_std]`, and a heap-free core covers keys, signing, addresses, PSBT, raw BTC/EVM/Zcash transactions, and
UR/BBQr parts. All crypto comes from `purecrypto`. Edition 2024, MSRV 1.89.

**Use it when:** you need wallet-side address derivation, validation, or offline transaction signing for several
chains in one dependency; PSBT workflows (create, update, sign, combine, finalize, extract) including Taproot key
and script paths; signing through external signers (the `Signer::sign_taproot`, `PsbtSigner` and `CardanoSigner`
traits work with TSS or HSMs); or embedded/hardware-wallet targets with no allocator.
**Don't use it when / limits:** you need node or network access (pair it with `ethrpc-rs` or another RPC client),
Zcash shielded (Sapling/Orchard) spends, or Cardano Plutus, certificates or metadata. There is no coin selection.

**Add it:**
```toml
[dependencies]
outscript = "0.2"
# only some chains, no_std with alloc:
# outscript = { version = "0.2", default-features = false, features = ["alloc", "bitcoin", "evm"] }
```

**Key features / cargo features:**
- Runtime tiers: `std` (the default, which adds `std::io` adapters), `alloc` (`Script`/`Out`, parsing, all tx
  types, RLP/CBOR/JSON, multi-part UR/BBQr), and none (a heap-free core that writes into caller buffers:
  `generate_script`, `address::encode_address_to_slice`, `psbt::Psbt`, `btcraw::RawTx`, `evmraw::RawEvmTx`,
  `zcashtx::ZcashTx`).
- Chain features: `bitcoin`, `evm`, `solana`, `cardano`, `massa`, `zcash`, `tron` (added in 0.2.6). Transport features: `bcur` (Uniform
  Resources) and `bbqr` (Coinkite BBQr). Curve features: `secp256k1`, `ed25519`. All chain and transport features
  are on by default.
- Transaction types: `BtcTx` + `BtcTxSign`, `psbt::Psbt`, `taproot` (script trees, control blocks),
  `EvmTx` / `RawEvmTx` (legacy, EIP-2930, EIP-1559), `solana::new_solana_tx` / `_v0` / `_v1`, `CardanoTx`
  (including CIP-1852 HD derivation via `cardano_icarus_master_key`), `zcashtx::ZcashTx` (v5, ZIP-244), and
  `trontx::TronTx` (protobuf transactions: TRX, TRC-10, TRC-20 / smart-contract calls; addresses via `parse_tron_address`).
- Also supports Bitcoin Knots unified sighash (`btcraw::SIGHASH_UNIFIED`). `SecpPrivateKey` and
  `CardanoExtendedKey` zeroize on drop.

**Example** (from the README):
```rust
use outscript::{Script, BtcTx, BtcTxSign, parse_evm_address};
use outscript::crypto::secp256k1::SecpPrivateKey;

let key = SecpPrivateKey::from_bytes(&secret)?;        // &[u8; 32]
let s = Script::new(key.public_key());
let btc = s.address("p2wpkh", &["bitcoin"])?;          // bc1q...
let eth = s.address("eth", &[])?;                      // 0x... (EIP-55)
let out = parse_evm_address("0x2AeB8ADD...")?;

let mut tx = BtcTx::from_bytes(&raw_unsigned)?;
tx.sign(&[BtcTxSign::new(&key, "p2wpkh").amount(600_000_000)])?;
let signed = tx.bytes();
```

**Gotchas:**
- With `default-features = false`, every chain is disabled too, so name the ones you need. Formats for disabled
  chains are simply "unknown".
- Taproot and unified-sighash inputs need `.amount(..)` and `.prev_script(..)` on `BtcTxSign`.
- Some `no_std` snippets in the README still say `version = "0.1"`. The current version is 0.2.

## ethrpc-rs

**Repo:** https://github.com/KarpelesLab/ethrpc-rs · **Crate:** `ethrpc-rs`, imported as `ethrpc_rs` (crates.io `0.3.2`) · **License:** MIT · **Status:** usable.

A lightweight async JSON-RPC client for Ethereum-compatible nodes, ported from Go
[`ethrpc`](https://github.com/KarpelesLab/ethrpc). It is built on the pure-Rust `rsurl` HTTP client. It runs on
native targets (Tokio, via rsurl's `tokio-rt` adapter; there is no direct tokio dependency) and on
`wasm32-unknown-unknown` (the browser Fetch API, where `Handler` drops its `Send`/`Sync` bounds). Edition 2021,
MSRV 1.89.

**Use it when:** you need raw EVM JSON-RPC calls with convenient result decoding, need to race several endpoints
and pick the fastest, want simple read-only `eth_call`s with ABI encoding, need to proxy JSON-RPC requests, or need
static chain metadata.
**Don't use it when / limits:** you need transaction signing (use `outscript`), subscriptions or WebSockets, or a
full typed provider like alloy/ethers. The ABI codec covers common types only (`address`, `uint`/`int`, `bool`,
`bytesN`, `bytes`, `string`, arrays).

**Add it:**
```toml
[dependencies]
ethrpc-rs = "0.3"
# chain metadata + ABI codec only, no HTTP stack:
# ethrpc-rs = { version = "0.3", default-features = false }
```

**Key features / cargo features:**
- `rpc` (default) enables `Rpc`, `Api`, `Handler`, `RpcList`, `evaluate`, and `abi::eth_call_abi`.
- `abi` (default) enables `abi::{encode, decode, encode_call, function_selector}`, with Keccak from `purecrypto`.
- `Rpc` offers `call`, `call_named`, `call_as::<T>`, `set_basic_auth`, `set_override` (local method
  interception), and `forward` (builds a proxied HTTP response).
- `ValueExt` on `serde_json::Value`: `to_u64()`, `to_big_int()`, `to_str()`.
- The `chains` module has `chains::get(chain_id)`, which returns the name, native currency, `has_feature("EIP1559")`
  and explorer URLs.

**Example:**
```rust
use ethrpc_rs::{Rpc, ValueExt};
use ethrpc_rs::abi::{eth_call_abi, ParamType, Token};

#[tokio::main]
async fn main() -> Result<(), ethrpc_rs::Error> {
    let rpc = Rpc::new("https://cloudflare-eth.com");
    let block = rpc.call("eth_blockNumber", vec![]).await?.to_u64()?;
    let out = eth_call_abi(&rpc, "0xdAC17F958D2ee523a2206206994597C13D831ec7",
        "balanceOf(address)", &[Token::address("0x28C6c06298d514Db089934071355E5743bf21d60")?],
        &[ParamType::Uint(256)]).await?;
    println!("{block} {:?}", out[0].as_uint());
    Ok(())
}
```

**Gotchas:**
- The crates.io crate named `ethrpc` (without `-rs`) is an unrelated third-party crate. Use `ethrpc-rs`.
- Cancel a call by dropping the future (for example with `tokio::time::timeout`). There is no context object like
  in Go.

## erigon-seg

**Repo:** https://github.com/KarpelesLab/erigon-seg · **Crate:** `erigon-seg`, imported as `erigon_seg` (crates.io `1.3.1`) · **License:** MIT · **Status:** production-quality. It is verified byte-exact against real Erigon v1.1 files.

Reads, queries, writes and merges the Erigon 3 **seg** state-snapshot file triple: `.kv` (seg-compressed
key/value words), `.bt` (Elias-Fano B-tree offset index) and `.kvei` (bloom existence filter). The only
dependencies are `memmap2` and `libc` (for mlock). The one `unsafe` is the mmap call. Edition 2024, MSRV 1.88.

**Use it when:** you need to read Ethereum state (accounts, storage, and so on) directly from an Erigon 3 datadir
without running Erigon, build Erigon-readable domain files, or merge domain files with Erigon's newest-wins and
deletion semantics.
**Don't use it when / limits:** you need Erigon 2 formats or the newer "fuse filter" `.kvei` layout. That layout is
detected and skipped, so lookups stay correct but are not bloom-accelerated.

**Add it:**
```toml
[dependencies]
erigon-seg = "1.3"
```

**Key features:**
- `KvReader::open` / `open_with`, then `get(key)` (an `O(log n)` search via `.bt`, or a linear scan when no index
  is present) and `iter()`.
- `enable_bloom(Salt::Find(n) | Salt::Known(salt))` for fast negative lookups. The salt can be brute-forced or read
  from `salt-state.txt`.
- Performance knobs: `preload_index` / `lock_index`, `preload_all` / `lock_all` (mlock), `advise_random`,
  `raise_memlock_limit`. Build with `-C target-cpu=native` for BMI2 `pdep`.
- Writing: `DomainWriter::create(path, DomainOptions { salt, compress, .. })`, with `add(k, v)` and `finish()`.
  Lower level: `SegWriter`, `build_bt`, `KveiBuilder`.
- Merging: `merge(&inputs, output, MergeOptions::default())`.
- Examples: `examples/inspect.rs`, `examples/recode.rs`.

**Example:**
```rust
use erigon_seg::{KvReader, Salt};

let mut r = KvReader::open("v1.1-accounts.0-1024.kv")?;
r.enable_bloom(Salt::Find(8));
if let Some(value) = r.get(b"\x00\x01\x02")? {
    println!("{} bytes", value.len());
}
for kv in r.iter() {
    let (key, value) = kv?;
}
# Ok::<(), erigon_seg::Error>(())
```

**Gotchas:**
- `DomainWriter::add` requires strictly increasing keys.
- `advise_random` helps when the files are much larger than RAM and makes lookups 2-3x slower when they fit in the
  page cache.
- Check `total_bytes()` against free memory before calling `preload_all`.

## libwallet

**Repo:** https://github.com/KarpelesLab/libwallet · **Crate:** `libwallet` (not on crates.io; built as `cdylib`/`staticlib`/`rlib` from `rust/`). The Dart client is on pub.dev as `libwallet`. · **License:** Karpeles Lab Non-Commercial (free for personal, educational, and under 10k MAU; commercial use requires a license) · **Status:** usable application backend, still evolving.

A complete TSS cryptocurrency wallet backend. Keys are held as threshold shares (via `tsslib`) across several
locations, so a wallet can be recovered without a single point of compromise. It supports EVM chains,
Bitcoin-family chains and Solana: balances, transfers, ERC-20/SPL tokens, NFTs, WalletConnect/Web3, price quotes,
and backup and restore. It composes `tsslib`, `outscript`, `ethrpc-rs`, `bottlers`, `graphitesql` (pure-Rust
SQLite), `rsurl`, `spotlib` and `klbfw`. It is a Rust port of an earlier Go implementation.

**Use it when:** you are building a wallet app (Flutter, native, or browser) and want the whole backend: storage,
chain logic, and TSS signing ceremonies.
**Don't use it when / limits:** you only need a building block, in which case depend on `tsslib` / `outscript` /
`ethrpc-rs` directly. It is not meant as a Rust library dependency: its API is a JSON request/response FFI. Check
the license before commercial use. The TSS, networking and database layers are native-only. The WASM build
currently covers the offline single-key core (`walletcore`: BIP-39, HD derivation, Solana tx building, signing).

**Build / use:**
```bash
git clone https://github.com/KarpelesLab/libwallet && cd libwallet
make              # cargo build --release in rust/ -> liblibwallet.{so,dylib,a}
make test
make dart-native  # stage the native lib for the local Dart client
```

- The C ABI is `LibwalletInit(data_dir) -> handle`, `LibwalletRequest(handle, request_json, cb, user_data)`,
  `LibwalletSetEventCallback`, `LibwalletDestroy`, and `LibwalletFree` (which frees returned strings).
- A request is JSON of the form `{"path": "...", "verb": "GET|POST|...", "params": {...}}`, and a response is a KLB
  envelope: `{"result":"success","data":...}` or `{"result":"error","error":...,"code":...}`. `"progress"` and
  `"event"` messages can also arrive. Paths are object/action routes such as
  `Wallet/Key`, `Web3/Connection`, and `Path:action`.
- All state lives in a single SQLite `sql.db` in the data directory.
- Dart: `LibwalletClient.initialize('/path/to/data')`. The package build hook downloads prebuilt binaries from
  GitHub Releases for Android, iOS, macOS and Linux.

**Gotchas:** Panics never cross the FFI boundary, because every export is wrapped in `catch_unwind`. The crate
requires `purecrypto` 0.9 across the whole graph. Open TODOs include reshare improvements, Token-2022 transfer
fees, and Bitcoin transaction history.

## chiefsplitter

**Repo:** https://github.com/KarpelesLab/chiefsplitter · **Crate:** `chiefsplitter` (Solana on-chain program, not on crates.io) · **License:** MIT · **Status:** usable. It has reproducible `solana-verify` builds and a `security_txt`.

A native Solana program (not Anchor; `solana-program` 2.0 with borsh instructions) that splits SOL and
SPL/Token-2022 tokens between up to 10 recipients by basis-point shares. Anyone can create any number of splitter
PDAs. Distribution is a permissionless crank. Non-whitelisted tokens can be swapped to SOL (or to a whitelisted
token) through CPI into admin-approved DEX programs such as Jupiter or Raydium.

**Program ID:** `ChiefYGYadRjMCgMNqbbFV8GUfiP2TqfRWzcWNynEoPh`

**Use it when:** you need an on-chain revenue or fee split that pays out automatically to fixed recipients,
optionally with guaranteed minimum shares (`LockRecipient`) or an immutable configuration (`RevokeSplitterAdmin`).
**Don't use it when / limits:** you need more than 10 recipients, more than 10 whitelisted mints, or more than 5
approved swap programs. Rounding dust stays on the PDA until the next distribution.

**Build / use:**
```bash
./scripts/build-sbf.sh                                # target/deploy/chiefsplitter.so
cargo test                                            # unit tests
cd tests/typescript && npm install && npm test        # E2E against solana-test-validator
solana-verify build --library-name chiefsplitter      # reproducible build
```
- To call the program from Rust, depend on the program crate through git with `features = ["no-entrypoint"]` and
  borsh-serialize `chiefsplitter::SplitterInstruction`. `idl.json` is in the repo root.
- PDAs: Splitter `["splitter", creator, nonce.to_le_bytes()]`; SellConfig `["sell_config", splitter]`.
- Shares are in basis points and must total 10000. Each recipient gets `floor(total * share / 10000)`.

**Instructions (borsh enum order = discriminant):** 0 `CreateSplitter { nonce, name }`,
1 `SetSplitterDistribution { recipients: Vec<(Pubkey, u16)> }`, 2 `SetSplitterAdmin`, 3 `RevokeSplitterAdmin`,
4 `DistributeSOL`, 5 `DistributeToken`, 6 `LockRecipient { min_share }`, 7 `SetSplitterName`,
8 `SetSellConfig`, 9 `CloseSellConfig`, 10 `SwapToken { swap_data }`, 11 `SnsProxy { sns_data }`. `SnsProxy` is an
admin-only CPI restricted to the Bonfida SNS program.

**Gotchas:**
- The README instruction table is stale. It omits `SetSplitterName` and `SnsProxy` and numbers `SetSellConfig` as
  7. Trust the `SplitterInstruction` enum in `programs/chiefsplitter/src/lib.rs`.
- The README gives the Splitter size as 442 bytes, but the current `Splitter::LEN` includes a 64-byte name plus a
  length byte, for 507 bytes.
- The recipient accounts passed to `Distribute*` must be in the configured recipient order.

## zanolib

**Repo:** https://github.com/KarpelesLab/zanolib · **Crate:** `zanolib` (crates.io `0.3.0`) · **License:** MIT · **Status:** usable. It is tested against about 1.9k KAT vectors and mainnet transactions from every era. Threshold broadcast is not complete.

A Rust library for [Zano](https://zano.org/) (a CryptoNote-family chain). It covers address handling (standard,
integrated, auditable), offline signing of view-only-wallet transactions, deposit scanning with only the view key,
building and broadcasting transactions from scanned deposits, and a threshold (MPC) spend key via FROST-Ed25519
(`tsslib`). It implements CLSAG-GGX, Bulletproof+ and BGE proofs, and its binary serialization matches Zano's C++
byte for byte. It has `#![forbid(unsafe_code)]`, all crypto comes from `purecrypto`, and the dependency tree
contains no foreign code. It replaces an earlier Go implementation, which remains in the repo history.
Edition 2024, MSRV 1.89.

**Use it when:** you are building a Zano custodial or exchange deposit flow (per-user integrated addresses, then
scan, then sweep), signing Zano transactions offline, or holding a Zano spend key under MPC.
**Don't use it when / limits:** it has no automatic coin selection or change outputs (inputs must sum exactly to
outputs plus fee). It cannot pay to gateway addresses, and payment IDs longer than 8 bytes are not supported. Decoy
selection is uniform rather than Zano's gamma distribution. Mempool scanning and spent tracking are not included.
Offline signing supports Zano 2.1.0.382 through 2.2.2.513, and only ZC-to-ZC transactions.

**Add it:**
```toml
[dependencies]
zanolib = "0.3"   # default-features = false -> offline core only
```

**Key features / cargo features:**
- `rpc` (default): the blocking JSON-RPC and `.bin` daemon client `rpc::Client` (with `sweep_to`) and
  `rpc::Scanner` (`scan_range`). It uses `rsurl`. Passing `""` as the endpoint selects the public modchain gateway.
- `mpc` (default): `mpc::address`, `mpc::partial_key_image`, `ThresholdInputSigner` (including
  `derive_view_secret()`), and `ClsagParty` / `ClsagCoordinator`. These run over tsslib's `MessageBroker`.
- Core: `Wallet::load_spend_secret`, `parse_ftp`, `sign`, `sign_with` (through the `InputSigner` trait),
  `encrypt`, `scan_tx`, and `build_transfer`. The modules are `crypto`, `base`, `epee` and `proof`.

**Example** (offline signing, from the README):
```rust
use zanolib::rng::OsRng;
use zanolib::Wallet;

let wallet = Wallet::load_spend_secret(&secret, 0)?;       // flags = 1 for auditable
let ftp = wallet.parse_ftp(&unsigned_tx_blob)?;            // from a view-only simplewallet
let finalized = wallet.sign(&mut OsRng, &ftp, None)?;
std::fs::write("zano_tx_signed", wallet.encrypt(&finalized)?)?;
```

**Gotchas:**
- The unsigned-transaction blob format is not versioned. The parser tries the Zano 2.2 layout first and falls back
  to the older one.
- `sweep_to` requires at least 10 confirmations per deposit and accepts at most 80 deposits per transaction.
- A threshold wallet's view key is derived from the shares, not from `keccak(spend)`. Seed-phrase recovery is
  therefore not possible for MPC wallets.

## evmabiless

**Repo:** https://github.com/KarpelesLab/evmabiless · **Crate:** `evmabiless` (crates.io `0.1.17`) · **License:** MIT · **Status:** usable.

Recovers the ABI of a deployed EVM contract from its bytecode. It scans the Solidity dispatch prologue
(`DUP1 PUSH4 sel EQ PUSH2 dest JUMPI`) for 4-byte selectors and matches them against a built-in table of several
hundred known function, event and error signatures. It also decodes calldata strictly. The crate is `#![no_std]`,
**never allocates**, has **zero dependencies**, and uses `#![forbid(unsafe_code)]`. The same repo ships Go and npm
versions. Edition 2024, MSRV 1.89.

**Use it when:** you need to call or inspect a contract that has no published ABI (tokens discovered on-chain), or
to show a human-readable decoding of transaction calldata, for example on a hardware wallet.
**Don't use it when / limits:** it cannot resolve selectors that are not in the table (`abi_list` drops them). A
4-byte selector match is only a guess, and the table keeps one entry per selector. Contracts from Vyper,
hand-written assembly or non-standard compilers may not match the dispatch pattern. Return types are known only for
table entries.

**Add it:**
```toml
[dependencies]
evmabiless = "0.1"
# calldata decoding only, small table:
# evmabiless = { version = "0.1", default-features = false, features = ["decode", "common-signatures"] }
```

**Key features / cargo features** (`decode`, `scan` and `signatures` are the defaults):
- `scan`: `scan_contract` / `scan_contract_hex`, which return lazy iterators of `MethodPrefix`. With a table
  feature, also `abi_list` / `abi_list_hex`.
- `signatures`: the full table (`lookup_abi`, `signatures()`). This is most of the crate's size.
  `common-signatures` is a small table covering ERC-20/721/1155 transfers and approvals, ERC-2612 permit, and WETH.
- `decode`: `Abi::decode_input`, plus `decode_calldata` when a table feature is enabled. Decoding is zero-copy.
  It is strictly canonical by default, and `DecodeMode::Lenient` relaxes that.
- You can declare your own `static` ABIs with `Abi::new(AbiType::Function, MethodPrefix([..]), name, &[AbiIO::new(..)])`.

**Example:**
```rust
use evmabiless::{abi_list_hex, lookup_abi, decode_calldata, MethodPrefix};

for abi in abi_list_hex(&bytecode_hex)? {            // hex from eth_getCode, 0x optional
    println!("{}", abi.abi);                          // function transfer(address to, uint256 value) returns (bool)
}
let transfer = lookup_abi(MethodPrefix::from_hex("a9059cbb")?).unwrap();
assert_eq!(transfer.compact, "transfer(address,uint256)");

let call = decode_calldata(&tx_input)?;
for param in call.params {
    println!("{}: {}", param.io.name, param.value);
}
```
