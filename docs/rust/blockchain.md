# Blockchain & Wallets (Rust)

Pure-Rust crates for building multi-chain cryptocurrency wallets, all built on
[`purecrypto`](https://github.com/KarpelesLab/purecrypto) for their cryptography (no C code, no OpenSSL). `tsslib`
provides threshold signing (MPC), `outscript` provides addresses, scripts and transaction building/signing for many
chains (including Zano, Monero and shielded Zcash since 0.3), `ethrpc-rs` talks JSON-RPC to EVM nodes, and `lsdk`
drives BOLOS hardware wallets (genuine check, app/firmware management, EVM/BTC/Solana signing). `zanolib` (Zano) is
archived in favor of outscript's `zano` module. `libwallet` is the application that
ties them together (a TSS wallet backend exposed over C FFI/WASM). The rest are narrower: `evmabiless` (recovers the
ABI from EVM bytecode, decodes calldata), `erigon-seg` (Erigon 3 snapshot files), `btcwallet` (a read-only
`wallet.dat` inspector) and `chiefsplitter` (an on-chain Solana program, now in TibaneLabs). `outscript` and `ethrpc-rs` are ports of Go libraries in KarpelesLab
([outscript](https://github.com/KarpelesLab/outscript), [ethrpc](https://github.com/KarpelesLab/ethrpc)). `tsslib`
started as a port of Go [tss-lib](https://github.com/KarpelesLab/tss-lib) (now archived) but no longer documents compatibility with it.

## Quick pick
| Need | Use |
|------|-----|
| Threshold (t-of-n) signatures: FROST Ed25519/ristretto255, DKLs23 ECDSA, threshold ML-DSA | [`tsslib`](#tsslib-rs) |
| Threshold BIP340 Schnorr / Bitcoin Taproot key-path signing (FROST secp256k1) | [`tsslib`](#tsslib-rs) (`frostsecp256k1tss`) |
| Migrate existing GG18/GG20 ECDSA or GG18-style EdDSA key shares | [`tsslib`](#tsslib-rs) (`ecdsatss`, `eddsatss`) |
| Derive addresses from a pubkey (BTC-family, EVM, Solana, Cardano, Massa, Tron, Zcash, Monero, Zano) | [`outscript`](#outscript-rs) |
| Parse/validate a crypto address for many chains | [`outscript`](#outscript-rs) |
| Build and sign BTC / PSBT / Taproot / EVM / Solana / Cardano / Tron / Zcash (transparent and shielded) / Monero / Zano transactions | [`outscript`](#outscript-rs) |
| Move PSBTs as QR codes (UR / BBQr) | [`outscript`](#outscript-rs) |
| no_std / no-alloc transaction signing (hardware wallet, embedded) | [`outscript`](#outscript-rs) |
| Async JSON-RPC to Ethereum/EVM nodes (native Tokio or browser WASM) | [`ethrpc-rs`](#ethrpc-rs) |
| EVM chain metadata (names, explorers, native currency) without networking | [`ethrpc-rs`](#ethrpc-rs) (`default-features = false`) |
| `eth_call` a contract with ABI encode/decode | [`ethrpc-rs`](#ethrpc-rs) |
| Guess the ABI of a deployed contract from its bytecode | [`evmabiless`](#evmabiless) |
| Decode EVM tx calldata for display (no_std, no alloc) | [`evmabiless`](#evmabiless) |
| Read/write/merge Erigon 3 `.kv` / `.bt` / `.kvei` snapshot files | [`erigon-seg`](#erigon-seg) |
| Zano: addresses, keys, scanning, offline signing, building | [`outscript`](#outscript-rs) (`zano` module; [`zanolib`](#zanolib) is archived) |
| Sign with or manage a BOLOS hardware wallet from Rust or the browser | [`lsdk`](#lsdk) |
| Inspect an old Bitcoin Core `wallet.dat` without Bitcoin Core or libdb | [`btcwallet`](#btcwallet) |
| Full TSS wallet backend for a mobile/desktop/web app (Dart/Flutter client) | [`libwallet`](#libwallet) |
| On-chain SOL/SPL fee splitter (Solana program) | [`chiefsplitter`](#chiefsplitter) |

## tsslib-rs

**Repo:** https://github.com/KarpelesLab/tsslib-rs · **Crate:** `tsslib` (crates.io `0.2.17`; **use 0.2.17 or later**, see the security gotchas) · **License:** MIT · **Status:** usable. FROST and DKLs23 are the main paths. `mldsatss`, `ecdsatss` and `eddsatss` are experimental and unaudited.

A pure-Rust threshold signature library. It started as a port of the Go [`tss-lib`](https://github.com/KarpelesLab/tss-lib),
but upstream dropped all references to Go tss-lib in October 2026 (commit b835506), so do not assume compatibility with
Go nodes. The JSON message and key-share formats are stable across releases. The crate is `#![no_std]` + `alloc`
(the default `std` feature adds OS randomness, blocking `wait()` and OS locks), has `#![forbid(unsafe_code)]`, and does
no field arithmetic of its own, since all group and lattice math comes from `purecrypto`. Edition 2024, MSRV 1.95.

| Module / feature | Scheme | Output |
|---|---|---|
| `frosttss` | FROST(Ed25519, SHA-512), RFC 9591 | Ed25519 signatures |
| `frostristretto255tss` | FROST(ristretto255, SHA-512) | ristretto255 Schnorr |
| `frostsecp256k1tss` | FROST(secp256k1, SHA-256), RFC 9591 | BIP340 Schnorr, optional BIP341 Taproot tweak |
| `dklstss` | DKLs23 threshold ECDSA, secp256k1 | ECDSA (r, s, v) |
| `mldsatss` | Threshold ML-DSA-44 (FIPS 204), `2 ≤ t ≤ n ≤ 6` | ML-DSA-44 signatures |
| `ecdsatss` | Legacy GG18/GG20 (Paillier + MtA) | ECDSA; loads legacy GG18/GG20 save files for migration |
| `eddsatss` | Legacy GG18-style threshold EdDSA | Ed25519; for migrating legacy GG18-style keys |

> **Security: upgrade to 0.2.17 or later.** Before 0.2.17, `dklstss` signing (`sign`, `sign_checked`, `SigningParty`,
> `CheckedSigningParty`, `presign`) leaked the weighted key share λᵢ·xᵢ through the OT-extension KOS consistency check:
> a co-signer, or anyone who could read the point-to-point messages, could recover it from about four signing sessions
> with the same signer set (fewer with `CheckedSigningParty`, one when t ≥ 4). Fixed in commit df91eae (2026-10-31). The
> fix bumps the signing wire version (`dkls:sign2:*` / `dkls:csign2:*`), so peers on 0.2.16 or earlier and peers on
> 0.2.17 or later cannot sign together: upgrade every party, and treat keys that signed with older versions over a
> channel an attacker could read, or with an untrusted co-signer, as exposed (reshare or rotate them).

**Use it when:** you need t-of-n keygen, signing, resharing or refresh where no single host holds the key; you need
HD derivation of threshold keys; you need threshold Taproot signing; or you must migrate existing GG18/GG20 key shares.
**Don't use it when / limits:** you need peer authentication, which you must supply through your `MessageBroker`
transport. Don't use `mldsatss` in production (the README calls it an academic-grade prototype). Use `ecdsatss` and
`eddsatss` only to migrate existing keys; new deployments should use `dklstss` or `frosttss`.

**Add it:**
```toml
[dependencies]
tsslib = "0.2"   # or pick protocols: { version = "0.2", default-features = false, features = ["std", "json", "dklstss"] }
purecrypto = { version = "0.10", default-features = false, features = ["std", "rng"] }  # for OsRng
```

**Key features / cargo features:**
- Each protocol has a like-named feature, and all are on by default, as are `std`, `json` and `error-messages` (descriptive serde error text; turn it off to save code size). Without `std`, register
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
- **KOS leak before 0.2.17:** see the security note above. 0.2.17 also documents that a KOS check failure blamed on a
  peer may leak one bit of this party's OT-sender secret per probe; stop signing with that peer on the existing pairwise
  OT state and rebuild it with `PairSetupParty` first.
- **`n ≤ 2t` keygen, fixed in 0.2.15** (commit 3c1c532): before it, colluding dealers that move last could end up knowing
  the whole key when `n ≤ 2t` (2-of-2, 3-of-3, ...). Each dealer now proves knowledge of its constant term. This changed
  the keygen round-1 messages, so all parties must run 0.2.15 or later. Rushing dealers can still bias the key's
  distribution, but not choose it.
- **Selective-failure aborts (still open):** the default `dklstss` signing path leaks about one bit of a secret share or
  nonce per aborted session to a malicious co-signer. `sign_checked` / `CheckedSigningParty` costs about 2x and catches
  inconsistent deviations, but per the source it does not fully close the leak; the real DKLs23 check needs a versioned
  protocol change. With untrusted co-signers, cap retries and reshare after repeated unexplained aborts.
- **`dklstss` and `ecdsatss` send VSS shares in the clear.** Their point-to-point messages must travel over a confidential
  channel (TLS, Noise, ...), or anyone reading the transport can rebuild the key. The FROST, `eddsatss` and `mldsatss`
  keygens and reshares seal shares end to end.
- The README's install snippets say `version = "0.3"`, but crates.io is at 0.2.17. Use `0.2`.
- `purecrypto` must resolve to 0.10 across your whole dependency graph, because its types cross crate boundaries.

## outscript-rs

**Repo:** https://github.com/KarpelesLab/outscript-rs · **Crate:** `outscript` (crates.io `0.3.0`) · **License:** MIT · **Status:** usable. It is actively developed and verified against BIP-174 test vectors.

A Rust port of Go [`outscript`](https://github.com/KarpelesLab/outscript). It generates output scripts and
addresses, parses addresses, and builds and signs transactions for Bitcoin-family chains (BTC, BCH, LTC, DOGE,
Namecoin, Monacoin, Dash, Electraproto), EVM, Solana, Cardano, Massa, Tron, Zcash (transparent, Sapling and
Orchard/Ironwood shielded), Monero and Zano. It is
`#![no_std]`, and a heap-free core covers keys, signing, addresses, PSBT, raw BTC/EVM/Zcash transactions, and
UR/BBQr parts. All crypto comes from `purecrypto`. Edition 2024, MSRV 1.95.

**Use it when:** you need wallet-side address derivation, validation, or offline transaction signing for several
chains in one dependency; PSBT workflows (create, update, sign, combine, finalize, extract) including Taproot key
and script paths; signing through external signers (the `Signer::sign_taproot`, `PsbtSigner` and `CardanoSigner`
traits work with TSS or HSMs); or embedded/hardware-wallet targets with no allocator.
**Don't use it when / limits:** you need node or network access (pair it with `ethrpc-rs` or another RPC client),
or Cardano Plutus, certificates or metadata. There is no coin selection. Monero multisig is experimental.

**Add it:**
```toml
[dependencies]
outscript = "0.3"
# only some chains, no_std with alloc:
# outscript = { version = "0.3", default-features = false, features = ["alloc", "bitcoin", "evm"] }
```

**Key features / cargo features:**
- Runtime tiers: `std` (the default, which adds `std::io` adapters), `alloc` (`Script`/`Out`, parsing, all tx
  types, RLP/CBOR/JSON, multi-part UR/BBQr), and none (a heap-free core that writes into caller buffers:
  `generate_script`, `address::encode_address_to_slice`, `psbt::Psbt`, `btcraw::RawTx`, `evmraw::RawEvmTx`,
  `zcashtx::ZcashTx`).
- Chain features: `bitcoin`, `evm`, `solana`, `cardano`, `massa`, `zcash`, `tron` (added in 0.2.6), `monero` and
  `zano` (0.3, via `cryptonote`). Zcash shielded: `zcash-sapling`, `zcash-orchard` (both default), `zcash-shielded`,
  `zcash-sapling-prover` (needs the Sapling parameters at run time) and `zcash-orchard-prover` (both non-default). Transport features: `bcur` (Uniform
  Resources) and `bbqr` (Coinkite BBQr). Curve features: `secp256k1`, `ed25519`. All chain and transport features
  are on by default.
- Transaction types: `BtcTx` + `BtcTxSign`, `psbt::Psbt`, `taproot` (script trees, control blocks),
  `EvmTx` / `RawEvmTx` (legacy, EIP-2930, EIP-1559), `solana::new_solana_tx` / `_v0` / `_v1`, `CardanoTx`
  (including CIP-1852 HD derivation via `cardano_icarus_master_key`), `zcashtx::ZcashTx` (v5, ZIP-244; v6, ZIP-229), `zcash_builder::Builder`
  (shielded), `monero::TransactionBuilder`, `zano::Wallet`, and
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
- Some `no_std` snippets in the README still say `version = "0.1"`. The current version is 0.3.
- `EvmTx` and `CardanoTx` are `#[non_exhaustive]` since 0.2.8: start from `EvmTx::default()` /
  `CardanoTx::default()` and assign fields; struct literals no longer compile outside the crate. 0.3 also dropped the `p2wsh:p2wpkh` format,
  `EvmTx::sign` refuses a transaction with no chain id, and unimplemented ECDSA sighash types return
  `UnsupportedSighash`.
- The default `bbqr` feature turns on `compcol` 0.8's `embed` feature graph-wide, so every compcol 0.8 user in the
  same build (for example purecrypto's `cert-compression`) gets the small heap-free deflate. Use
  `default-features = false` without `bbqr` if that matters.
- 0.3.0 requires `purecrypto` ^0.10. Keep `purecrypto` at 0.10 across the whole graph (tsslib 0.2.16+ needs it too), or you get two copies whose types don't mix.

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
- 0.3.2's optional `abi` feature depends on `purecrypto` 0.6 (for Keccak), so next to current `outscript`/`tsslib` you
  get a second, older `purecrypto` in the dependency tree and more code size.

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

## lsdk

**Repo:** https://github.com/KarpelesLab/lsdk · **Crate:** `lsdk` library + `lsdk` CLI (crates.io `0.1.1`; HW.1 / original Nano support (`btchip` feature, `lsdk hw1 …`), `app load --legacy-layout` and `ca install --regular` are on master only, so install from git for them) · **License:** MIT (the Bitcoin client-command interpreter `src/apps/btc/merkle.rs` is a port of the vendor's Apache-2.0 reference client) · **MSRV:** 1.95, edition 2024 · **Status:** usable, very new (first release 2026-11-10) and moving fast. Web console: https://karpeleslab.github.io/lsdk/

A Rust SDK, CLI and browser console for hardware wallets that run the BOLOS operating system (Nano S, Nano S Plus, Nano X, Stax, Flex, Nano Gen5), over native USB HID or WebHID from wasm. It covers the dashboard (device info, list/open/quit apps, rename, battery, language packs, screensaver, raw APDUs), genuine checks (online through the vendor HSM over websocket, or **offline** by verifying the device attestation against pinned issuer keys), app install/uninstall and firmware updates through the vendor HSM, the dashboard secure channel (custom CA, sideloading apps from their ELF, memory info, endorsement keys), self-hosted seed backup through the Recover protocol (your own CA as the backup service), and app clients for Ethereum/EVM, Bitcoin (wallet policies, PSBT) and Solana. Transactions are built with [`outscript`](#outscript-rs), crypto comes from [`purecrypto`](https://github.com/KarpelesLab/purecrypto), HTTP and the HSM websocket go through `rsurl`, and on Linux it falls back to raw usbfs through [`rawusb`](ui-hardware.md#rawusb) when the hidraw node is not accessible. Not affiliated with the device manufacturer.

**Use it when:** you need to drive a BOLOS hardware wallet from Rust or the browser: derive and display addresses, sign EVM / PSBT / Solana transactions on the device, run an offline genuine check, manage apps and firmware without the vendor's desktop app, or sideload your own app during development.
**Don't use it when / limits:** no BLE or emulator (Speculos) transport ships; `Transport` is a one-method trait, so you can add one. Only the Ethereum, Bitcoin and Solana app clients exist (plus legacy BTChip on master). The HSM operations (online genuine check, install, firmware) depend on the vendor's servers. Vendor endorsement (`endorse setup --vendor`) did not work as of 2026-11 (the HSM sends an empty certificate; lsdk refuses to forward it). A genuine check proves the secure element and its firmware, not that the rest of the device is free of added hardware.

**Add it:**
```toml
[dependencies]
lsdk = { version = "0.1", features = ["manager", "attestation", "eth"] }   # default = ["hid"]
tokio = { version = "1", features = ["rt", "macros"] }   # any executor works; lsdk does not need tokio
hex = "0.4"                                              # only for the example output
```
```sh
cargo install lsdk --features cli                                    # 0.1.1
cargo install --git https://github.com/KarpelesLab/lsdk --features cli   # master (HW.1 support)
lsdk info; lsdk genuine --offline; lsdk install Bitcoin; lsdk eth address --display
```
Native HID on Linux needs `libudev` and a udev rule for vendor id `2c97` (the README gives it); the browser console needs a Chromium-based browser.

**Key features / cargo features:**
- core (always): `Device<T: Transport>` dashboard commands (`get_version`, `list_apps`, `open_app`, `get_battery_status`, …), `Apdu`/`Response`/`StatusWord`, HID framing, `DeviceModel`, `path::DerivationPath` (`FromStr`).
- `hid` (default): `transport::hid::HidTransport` (`open_first`, `list`, `open`, `wait_and_open`, `open_usbfs`). `webhid`: `WebHidTransport` (wasm32).
- `attestation`: `attestation::offline_genuine_check(&device, &extra_issuers, &mut on_event) -> Attestation { genuine, issuer, proof_of_possession, .. }`, no network.
- `manager` (= `manager-api` + `hsm`): `manager::Manager` (`Default` = vendor endpoints) with `genuine_check`, `install_app`, `install_app_with_deps`, `uninstall_app_by_name`, `check_firmware_update`, `install_osu`, language packs.
- `scp`: secure channel (`setup_custom_ca`, `list_apps`, `delete_app`, `hash_firmware`, …); `loader`: `AppImage::from_elf` / `from_ihex` for sideloading; `recover`: Recover-protocol seed backup with your own CA.
- `eth`, `btc`, `sol`, `apps`: `apps::eth::EthApp`, `apps::btc::BtcApp` (`default_policy`, `register_wallet`, `get_wallet_address_checked`, `sign_psbt`; `apps::btc::psbt::build_psbt`), `apps::sol::SolApp`. On master `apps` also enables `btchip`.
- `full` (everything native), `wasm` (JS bindings for the web console), `cli`, `serde`.

**Example (verified to compile against crates.io 0.1.1; not run against a device):**
```rust
use lsdk::{Device, transport::hid::HidTransport, manager::Manager, attestation::offline_genuine_check};
use lsdk::apps::eth::{EthApp, EvmTx, EvmTxType, parse_units};

#[tokio::main(flavor = "current_thread")]
async fn main() -> lsdk::Result<()> {
    let device = Device::new(HidTransport::open_first()?);
    let os = device.get_version().await?;
    println!("{} SE {}", os.model().map(|m| m.name()).unwrap_or("?"), os.se_version);

    // Offline genuine check: attestation verified against pinned issuer keys.
    let att = offline_genuine_check(&device, &[], &mut |ev| println!("{ev:?}")).await?;
    println!("genuine: {} (issuer {:?})", att.genuine, att.issuer);

    // Vendor HSM over websocket; the user approves on the device.
    let manager = Manager::default();
    manager.install_app_with_deps(&device, "Ethereum", None, &mut |_, ev| println!("{ev:?}")).await?;

    device.open_app("Ethereum").await?;
    let eth = EthApp::new(&device);
    let path = "m/44'/60'/0'/0/0".parse()?;
    println!("address {}", eth.get_address(&path, false, false).await?.address);

    let mut tx = EvmTx::default(); // #[non_exhaustive]: assign fields, no struct literal
    tx.tx_type = EvmTxType::Eip1559;
    tx.chain_id = 1;
    tx.nonce = 0;
    tx.to = "0x000000000000000000000000000000000000dEaD".into();
    tx.value = parse_units("0.01", 18)?;
    tx.gas = 21_000;
    tx.gas_tip_cap = parse_units("1", 9)?;
    tx.gas_fee_cap = parse_units("30", 9)?;
    eth.sign_tx(&path, &mut tx).await?;                   // signature is written into tx
    let raw = tx.to_bytes().map_err(|e| lsdk::Error::App(format!("{e:?}")))?;
    println!("{}", hex::encode(raw));                     // ready to broadcast
    Ok(())
}
```

**Gotchas:**
- The README examples are out of date: `install_app_with_deps` takes an `installed: Option<&[String]>` argument before the callback, and `EvmTx` is `#[non_exhaustive]` (outscript ≥ 0.2.8), so `EvmTx { .., ..Default::default() }` does not compile outside outscript. Build it from `EvmTx::default()`. `EvmTx::to_bytes()` returns `Result<_, outscript::Error>`, which has no `From` conversion into `lsdk::Error`.
- Many dashboard commands prompt the user on the device (listing apps, reading the device name, opening the secure channel). The library never lists apps implicitly for that reason. Pass `installed` if you know it.
- Manager/HSM and secure-channel operations require the device to be on the dashboard (`Error::DashboardExpected`); app clients need the app open (`device.open_app("Ethereum")`).
- While a custom CA is installed, vendor app installs fail (`0x6804`) and on some devices vendor HSM sessions are refused; `lsdk ca remove` first. Setting the PIN during onboarding erases a custom CA.
- `HidTransport::exchange` is `async` in name only: it blocks the calling thread on hidapi until the device answers (which includes waiting for the user to confirm). `Transport` is an `async fn` trait with no `Send` bound, so code generic over `T: Transport` cannot `tokio::spawn` its futures; a current-thread runtime is the simplest fit.
- The raw-usbfs fallback detaches the kernel HID driver from the interface while open and re-attaches it on drop.

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
requires `purecrypto` 0.10 across the whole graph (the Rust workspace pins `0.10.1` and MSRV 1.95). Open TODOs include reshare improvements, Token-2022 transfer
fees, and Bitcoin transaction history.

## btcwallet

**Repo:** https://github.com/KarpelesLab/btcwallet · **Crate:** none; binary-only (no `lib.rs`), git only. The name `btcwallet` is not registered on crates.io · **License:** MIT · **MSRV:** 1.95, edition 2024 · **Status:** working CLI (2 commits, 2026-11-09). Its unit and CLI tests pass offline against a BDB 4.8 fixture.

A read-only command-line inspector for legacy Bitcoin Core / Bitcoin-Qt `wallet.dat` files (Berkeley DB 4.8, also the forks: Litecoin, Dogecoin, Namecoin, testnet, regtest). It parses the Berkeley DB btree itself (no libdb, no Bitcoin Core) and prints the wallet version, encryption parameters (master-key KDF, iterations, salt), HD chain, keys with their addresses, key pool, address book, transactions, scripts, watch-only entries, accounts and descriptors. It **never prints private keys** and does not need the passphrase. `--salvage` scans every leaf page of damaged or truncated files. `--balance` (off by default) sends every wallet address to the modchain `modchain_assets` JSON-RPC endpoint and totals the UTXOs. Its only dependencies are [`outscript`](#outscript-rs) (address encoding) and [`purecrypto`](https://github.com/KarpelesLab/purecrypto) (hashing, and TLS for its built-in HTTP client).

**Use it when:** you need to find out what an old `wallet.dat` contains (which addresses, whether it is encrypted or HD, key dates) or whether it still holds funds, without installing Bitcoin Core or libdb.
**Don't use it when / limits:** it does not decrypt or export keys and cannot spend or sweep. It reads only BDB wallets, not the SQLite descriptor wallets of current Bitcoin Core. It is a binary, so it cannot be used as a library. `--balance` reveals every wallet address to the balance service.

**Install / run:**
```sh
cargo install --git https://github.com/KarpelesLab/btcwallet
btcwallet wallet.dat            # summary
btcwallet -k -a wallet.dat      # keys, with P2SH-P2WPKH and P2WPKH forms of compressed keys
btcwallet -l -t wallet.dat      # address book, transaction ids
btcwallet -s -k damaged.dat     # salvage mode
btcwallet -b wallet.dat         # online balance lookup (privacy: sends all addresses)
```
Flags: `-n/--network` (bitcoin, testnet, regtest, litecoin, dogecoin, namecoin; guessed from the address book by default), `-d/--db` (sub-database), `-r/--records` (raw records as hex, private keys redacted), `--balance-url` (`{chain}` is replaced by the chain name), `--timeout`.

**Gotchas:**
- Work on a copy, and not on a wallet Bitcoin Core has open: recent changes may still be in the BDB log files and not yet in `wallet.dat`.
- `--salvage` may return stale records from freed pages.
- HTTPS uses the system CA bundle (`SSL_CERT_FILE` overrides it) and honours `https_proxy` / `no_proxy`.

## chiefsplitter

**Repo:** https://github.com/TibaneLabs/chiefsplitter (moved from KarpelesLab to the sister org Tibane Labs in November 2026) · **Crate:** `chiefsplitter` (Solana on-chain program, not on crates.io) · **License:** MIT · **Status:** usable. It has reproducible `solana-verify` builds and a `security_txt`.

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

**Repo:** https://github.com/KarpelesLab/zanolib · **Crate:** `zanolib` (crates.io `0.3.0`) · **License:** MIT · **Status:** **archived** (read-only since November 2026; its README has no successor note).

> **Use [`outscript`](#outscript-rs)'s `zano` module (0.3+) for new code.** It started as a port of zanolib and covers
> addresses, keys, scanning, offline signing, building, full verification and blocks (`outscript::zano::Wallet`).
> It does **not** port zanolib's `zanorpc` client or its `zanompc` threshold signer; `InputSigner` is the hook a
> threshold signer plugs into. The rest of this section describes the archived crate.

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
- zanolib 0.3.0 still depends on `purecrypto` 0.9, while current `tsslib` (0.2.16+) and `outscript` use 0.10. With
  the `mpc` feature, or next to those crates, the build pulls in two `purecrypto` versions; if types from them have
  to meet, compilation fails.

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
