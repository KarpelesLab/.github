# AI Tooling & Databases

Two groups of pure-Rust packages. The **AI tooling** group is for building and running agents. **`atelier`** is a terminal coding-agent harness for OpenAI-compatible APIs. **`manu`** is an MCP server that gives an agent real-world actions (wallet and email are planned). **`nixvm`** is a syscall-emulating Linux sandbox, so an agent can run untrusted commands (`npm install`, build scripts) away from the host. The **database** group has two storage engines, each compatible with an existing on-disk format. **`graphitesql`** reimplements SQLite (the file format and the SQL dialect) with no dependencies and `no_std` support. **`pebbledb`** ports CockroachDB's Pebble LSM key-value store. These packages do not depend on each other. `atelier` uses `rsurl` for HTTP, and `nixvm` can optionally use `fstool` and `pktkit`.

## Quick pick
| Need | Use |
|------|-----|
| A terminal coding agent against a local or self-hosted OpenAI-compatible model | [`atelier`](#atelier) |
| An MCP server that lets Claude/agents take real-world actions (wallet, email) | [`manu`](#manu) (early scaffold) |
| Run an untrusted Linux command without touching the host root filesystem, no Docker or root needed | [`nixvm`](#nixvm) |
| An embedded SQL database that reads and writes real SQLite files, pure Rust, `no_std`/WASM | [`graphitesql`](#graphitesql) |
| A SQLite-compatible C ABI (`libsqlite3`-style) built from safe Rust | [`graphitesql`](#graphitesql) (`capi`) |
| An ordered LSM key-value store, or reading/writing Pebble (CockroachDB) data directories | [`pebbledb`](#pebbledb) |

## atelier

**Repo:** https://github.com/KarpelesLab/atelier · **Binary:** `atelier` (git only. The crates.io crate named `atelier` is an **unrelated project**, so do not `cargo install atelier`.) · **License:** MIT · **Status:** usable, actively developed, pre-release.

An AI coding harness that runs in a terminal with a deliberately minimal interface. It runs an agent loop against **OpenAI-compatible** chat-completion endpoints (bring your own: Ollama, vLLM, llama.cpp server, hosted APIs). It runs tools inside the project directory, acts as an MCP client over stdio and Streamable HTTP, and feeds the model fresh context on every turn: git status and diff, project layout, and `cargo check` diagnostics. It is a single binary with a small interface: one input line and a status strip, with everything else printed append-only to the terminal scrollback. MSRV is 1.95 (edition 2024).

**Use it when:**
- You want a Claude Code-style coding agent for local or self-hosted models.
- You need scriptable one-shot agent runs in CI (`--print`).
- You want file tools confined to the project root that run without prompts, and approval only for unconfined actions.

**Don't use it when / limits:**
- You want to use a Claude or ChatGPT *subscription*. By design it supports only OpenAI-compatible API endpoints.
- A turn in progress cannot be cancelled. Ctrl-C only clears or drops input that has not been sent.
- `atelier.toml` is read only from the project root. There is no global config.

**Install / run:**
```sh
cargo install --git https://github.com/KarpelesLab/atelier      # add --no-default-features for a headless build (no TUI)
cd my-project
ATELIER_BASE_URL=http://localhost:11434/v1 ATELIER_MODEL=qwen3:32b atelier
atelier --continue                          # -c: resume .atelier/session.json
atelier --print "summarize src/main.rs"     # -p: one-shot; the final answer goes to stdout, everything else to stderr
ATELIER_APPROVE=all atelier -p < task.txt   # headless run that auto-approves bash/MCP/network tools
```

**Key features:**
- Environment variables: `ATELIER_BASE_URL`, `ATELIER_MODEL`, `ATELIER_API_KEY` (sent as a Bearer token), `ATELIER_APPROVE`, `ATELIER_CONTEXT_LIMIT` (compaction threshold, default 8000 tokens), `ATELIER_REVIEW`, `ATELIER_HTTP_TIMEOUT_MS`, `ATELIER_DEBUG`, `ATELIER_TRACE` (a JSONL request log).
- Built-in tools:
  - Run automatically, confined to the project root: `read`, `write`, `edit`, `multiedit`, `apply_patch`, `grep`, `glob`, `ls`, `tree`, `todo`.
  - Need approval: `bash`, `web_fetch`, and every MCP tool.
  - `node` runs sandboxed JavaScript with `fs` access. It needs approval only when called with `network: true`.
- MCP servers are configured in `atelier.toml` with `[[mcp]]` (stdio: `name`, `command`, `args`) or `[[mcp_http]]` (`name`, `url`, `headers`), or added with `/mcp add`. Their tools are namespaced `mcp__<name>__<tool>`.
- Sessions persist to `.atelier/session.json` and are compacted automatically. `/review on` turns on a parallel read-only "subconscious" reviewer model.
- Slash commands: `/help`, `/models`, `/model`, `/tools`, `/mcp`, `/review`, `/config`, `/new`, `/image <path>` (vision), `/clear`, `/quit`.

**Gotchas:**
- The default `ATELIER_BASE_URL` points at a LAN address the author uses. Always set it.
- `--print` denies any tool that needs approval unless `ATELIER_APPROVE=all` is set.
- Add `.atelier/` to `.gitignore`.

## manu

**Repo:** https://github.com/KarpelesLab/manu · **Binary:** `manu` (git only) · **License:** MIT · **Status:** early scaffold. Only the `system` tools work. The `wallet` and `email` tools are typed and discoverable, but they return "not implemented".

A local **Model Context Protocol server over stdio** (JSON-RPC, built on `rmcp`) that is meant to give agents "hands": holding and moving value, and creating and using email addresses. It runs as a subprocess of the agent's MCP client and holds its own state (eventually including keys), so it is the trust boundary between the agent and the outside world. The protocol uses stdout. Logs go to stderr.

**Use it when:**
- You are wiring up, or contributing to, the KarpelesLab agent-actions server and want the tool surface in place now.

**Don't use it when / limits:**
- You need working wallet or email actions today. Every handler except `manu_status` and `manu_ping` returns "not implemented".

**Tools:**

| Tool | Arguments | Status |
|------|-----------|--------|
| `manu_status` | none. Returns name, version, `data_dir`, and the status of each feature area. **Call this first.** | available |
| `manu_ping` | none. Returns `"pong"` | available |
| `wallet_balance` | `asset` (e.g. `"BTC"`, `"ETH"`, `"USDC"`) | scaffold |
| `wallet_address` | `asset` | scaffold |
| `wallet_send` | `to`, `amount` (a decimal **string**), `asset` | scaffold |
| `email_create` | `name` (optional mailbox local part, generated if omitted) | scaffold |
| `email_list` | none | scaffold |
| `email_send` | `from` (a managed address), `to`, `subject`, `body` (plain text) | scaffold |

**Install / configure:**
```sh
git clone https://github.com/KarpelesLab/manu && cd manu
cargo build --release                        # -> target/release/manu (edition 2024, Rust 1.85+)

# Claude Code (user- or project-scoped stdio server):
claude mcp add manu -- "$PWD/target/release/manu"
claude mcp add --scope project manu -- "$PWD/target/release/manu"   # writes .mcp.json
```
Equivalent JSON for `.mcp.json`, `claude_desktop_config.json` or any MCP client:
```json
{
  "mcpServers": {
    "manu": {
      "command": "/absolute/path/to/manu/target/release/manu",
      "env": { "RUST_LOG": "info", "MANU_DATA_DIR": "/home/me/.manu" }
    }
  }
}
```
Configuration comes from environment variables: `MANU_DATA_DIR` (default `~/.manu`, where the keystore and caches live) and `RUST_LOG` (default `info`, written to stderr). In Claude Code, the tools appear as `mcp__manu__manu_status` and so on. In atelier, add `[[mcp]] name = "manu" command = "/abs/path/manu"`.

**Smoke test without a client:**
```sh
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"manu_status","arguments":{}}}' \
  | ./target/release/manu
```

**Gotchas:**
- Check `manu_status` → `features` before calling wallet or email tools. They are listed in `tools/list` but not functional yet.
- Anything that writes to stdout breaks the protocol. Logs belong on stderr only.
- The design notes for the planned work are `docs/wallet.md` (a spend-authorization policy) and `docs/email.md` (backed by the Karpelès Lab email APIs). `ARCHITECTURE.md` has a recipe for adding a feature.

## nixvm

**Repo:** https://github.com/KarpelesLab/nixvm · **Crate:** `nixvm` library + CLI (crates.io `0.0.1`. The project moves daily, so prefer git.) · **License:** MIT · **Status:** experimental but functional. Alpine busybox, `apk` and Node.js run.

A portable sandbox in the style of a VM that runs a **real Linux userland without a guest kernel**, like gVisor. Guest code runs on a software CPU interpreter (aarch64, and a growing x86-64) or on KVM (Linux/x86-64) / Hypervisor.framework (macOS/arm64). Every `syscall`/`svc` traps into nixvm's own Rust "kernel", which implements files, memory, processes, threads, signals and networking in userspace. It needs no root, no Docker and no namespaces. Supported loaders: static, static-PIE and dynamically linked ELF (real `ld-musl`). The core has zero dependencies and builds for wasm; there is a [live browser demo](https://karpeleslab.github.io/nixvm/). `unsafe` is limited to four documented FFI sites.

**Use it when:**
- An agent must run untrusted or messy commands (`npm install`, `pip install`, `./configure`, `apk add`) and should not modify the host root filesystem or reach the network by default.
- You need a sandbox that works identically on Linux, macOS and wasm, with no privileges.

**Don't use it when / limits:**
- You need a security boundary that has been audited. It is experimental, and syscalls it does not support return `ENOSYS`.
- You need custom signal handlers. Only the default dispositions are delivered.
- You need strong network isolation. Guest traffic is loopback-only by default, and `NIXVM_NET=host` bridges it to real host sockets.
- You need speed on the interpreter path, which is much slower than native. Hardware backends run static binaries; dynamic linking on them is still in progress.
- Image download and caching (`--image`) are stubs. You must supply an extracted rootfs directory.

**Install:**
```sh
cargo install --git https://github.com/KarpelesLab/nixvm     # or: cargo install nixvm
# a guest root: any extracted Linux rootfs, e.g. Alpine minirootfs
mkdir alpine && curl -fsSL https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.0-x86_64.tar.gz | tar xz -C alpine
```

**How an agent sandboxes a command** (tested on Linux/x86-64 against the repo at 2026-09-28):
```sh
nixvm run --root ./alpine --workdir "$PWD" -e CI=1 -- /bin/busybox sh -c 'cd /work && make test'
echo $?   # the guest's exit code is propagated
```
- `--root DIR` is the **read-only** lower layer of a copy-on-write overlay. Writes to `/`, `/etc` and so on go to an in-memory tmpfs and are discarded when the run ends. The host rootfs is never modified.
- `--workdir DIR` is mounted at **`/work` with read-write passthrough**, so the guest's changes there *do* land on the host. The default is the current directory. Point it at a scratch copy if the command is untrusted. The passthrough is symlink- and TOCTOU-safe and cannot escape its root.
- The initial cwd is `/root`, so `cd /work` first. `HOME=/root`, `HOSTNAME=nixvm`. Add variables with `--env/-e KEY=VAL`.
- `--mem 2G` caps guest RAM. Other environment variables: `NIXVM_CPUS=N` (SMP worker threads), `NIXVM_TRACE=1` (log every syscall), `NIXVM_INTERP=1` (force the interpreter), `NIXVM_NET=host` (allow egress. DNS worked in our test. Treat it as experimental.)
- `/tmp`, `/dev`, `/proc` and `/sys` are synthesized inside the sandbox. The host's home directory and other paths are not visible unless you bind them.

**Library example** (`Sandbox` builder; `bind`/`bind_ro` add extra host directories):
```rust
use nixvm::Sandbox;

fn main() -> Result<(), nixvm::Error> {
    let code = Sandbox::builder()
        .root_dir("./alpine")                 // read-only lower + tmpfs upper
        .work_dir("/tmp/scratch-project")     // mounted rw at /work
        .bind_ro("/opt/cache", "/cache")
        .env("CI=1")
        .mem_bytes(2 << 30)
        .command(["/bin/busybox", "sh", "-c", "cd /work && ls"])
        .run()?;                              // guest exit code (i32)
    std::process::exit(code);
}
```

**Gotchas:**
- **PID 1 (`argv[0]`) must be an absolute path to a real ELF file.** There is no `PATH` lookup and no symlink resolution at that step. For example, `-- /bin/sh` fails on Alpine with "ELF file is truncated", because `/bin/sh` is a symlink to busybox. Use `/bin/busybox sh -c '...'`. Inside the guest, `PATH` and symlinks work normally. For the same reason, `nixvm shell` currently fails on stock Alpine.
- Without `--root`, `/` is an empty tmpfs, so nothing can run unless you use `exec_elf`.
- `scripts/build-claude-root.sh` builds an Alpine root containing the musl build of Claude Code, to run `claude -p` inside nixvm with `NIXVM_NET=host`.
- Cargo features: `kvm`/`hvf`/`interp` (backends), `fstool` (ext/squashfs images), `targz`, `fetch` (image download over HTTP with `ureq`), `tunnel` (a pktkit-based network for the browser), `wasm`.

## graphitesql

**Repo:** https://github.com/KarpelesLab/graphitesql · **Crate:** `graphitesql` (crates.io `0.1.7`) · **License:** public domain (SQLite "blessing", SPDX `blessing`) · **Status:** usable for reads and writes, verified differentially against `sqlite3` (a corpus of 1,600+ queries and 360+ suites). Pre-1.0.

A pure, **safe** (`unsafe_code = "deny"`, with exceptions only in the opt-in FFI modules), **zero-dependency**, `no_std` + `alloc` reimplementation of SQLite in one crate. It opens files written by `sqlite3` and writes databases that pass `sqlite3`'s `PRAGMA integrity_check`, and its error messages match `sqlite3` byte for byte. `SELECT` runs on a register-machine VDBE and falls back to a tree-walker. It covers most of SQLite: joins (including RIGHT/FULL), CTEs, window functions, UPSERT/`RETURNING`, `STRICT` tables, generated columns, triggers, foreign keys, `ATTACH`, `ALTER TABLE`, JSON/JSONB, WAL read and write, the rollback journal, `VACUUM`, the session/changeset extension, and the virtual tables `fts5`, `rtree`, `series`, `dbstat` and `sqlite_dbpage`. MSRV is 1.89.

**Use it when:**
- You need SQLite files or SQL from pure Rust with no C toolchain, for example in WASM (with an OPFS VFS), on embedded targets, or in capability-sandboxed hosts (all I/O goes through a `Vfs` trait you control).
- You need an in-memory SQL engine in `no_std`.

**Don't use it when / limits:**
- You need maximum performance or battle-tested durability. Its goal is correctness first, and `rusqlite` with C SQLite is faster and more proven.
- You need `rusqlite`-style prepared-statement handles. The API is `execute`/`query` on SQL strings. Parameter binding exists (`query_params`/`execute_params`), but its `Params` type lives in the doc-hidden `graphitesql::exec::eval` module.
- `upper()`/`lower()` fold only ASCII unless you enable `unicode`.

**Add it:**
```toml
[dependencies]
graphitesql = "0.1"
# no_std (in-memory or your own Vfs):
# graphitesql = { version = "0.1", default-features = false }
```

**Cargo features:** `std` (default: file VFS and `std::error::Error`), `fts5` (default), `unicode` (full Unicode case folding), `capi` (exports a `sqlite3_*` C ABI. Build it with `cargo rustc --release --features capi --crate-type cdylib`; the header is `include/sqlite3.h`), `wasm` (wasm-bindgen + OPFS).

**Example:**
```rust
use graphitesql::{Connection, Value};

fn main() -> graphitesql::Result<()> {
    let mut db = Connection::open_memory()?;   // or Connection::create("app.db")? / open("app.db")? / open_readonly(..)
    db.execute("CREATE TABLE users(id INTEGER PRIMARY KEY, name TEXT)")?;
    db.execute("INSERT INTO users(name) VALUES ('ada'), ('grace')")?;
    let result = db.query("SELECT id, name FROM users ORDER BY name")?; // QueryResult { columns, rows }
    for row in &result.rows {
        if let (Value::Integer(id), Value::Text(name)) = (&row[0], &row[1]) {
            println!("{id}: {name}");
        }
    }
    Ok(())
}
```
Other `Connection` methods: `execute_batch`, `execute_returning`, `last_insert_rowid`, `changes`, `serialize`/`deserialize`, `register_function`/`register_aggregate_function`/`register_collation`/`register_module`, update, commit and rollback hooks, `set_authorizer`, and `create_session`/`changeset_apply`.

**Gotchas:**
- `execute` takes `&mut self`, while `query` takes `&self`.
- There is a `graphitesql` CLI modeled on `sqlite3`: `cargo run --bin graphitesql -- app.db "SELECT ...;"`, with `.tables`, `.schema`, `.headers` and `.quit`.

## pebbledb

**Repo:** https://github.com/KarpelesLab/pebbledb · **Crate:** `pebbledb` (crates.io `0.0.1`, git for the latest) · **License:** BSD-3-Clause (a derivative of Pebble, LevelDB-Go and RocksDB) · **Status:** broad parity implemented, pre-1.0. The API is not stable yet.

A Rust port of CockroachDB's **Pebble**, an LSM-tree key-value store in the LevelDB/RocksDB lineage, whose sstable, WAL and MANIFEST formats are **binary-compatible** with Pebble's. A Go interop suite checks this in CI. Compression comes from `compcol` (snappy and zstd, pure Rust) and `minlz`. MSRV is 1.88.

**Use it when:**
- You need an embedded, ordered, persistent key-value store with snapshots, batches, range deletes and merge operators, in pure Rust with no C++ dependency (unlike `rocksdb`).
- You need to read or write Pebble or CockroachDB data directories, sstables, WALs or MANIFESTs from Rust.

**Don't use it when / limits:**
- You need an API that will not change. It mirrors Pebble's semantics with Rust naming and is not frozen.
- Byte parity for a few newer formats (the objstorage catalog and the columnar key schema) is still being finished.

**Add it:**
```toml
[dependencies]
pebbledb = "0.0.1"   # or: pebbledb = { git = "https://github.com/KarpelesLab/pebbledb" }
```

**Key features:**
- Writes: `set`, `delete`, `single_delete`, `merge`, `delete_range`, range keys, and atomic `Batch` (`Batch::new()`, `batch.set(..)`, `db.apply(batch)`). `Db::indexed_batch` gives read-your-own-writes.
- Reads: `get`, `snapshot()`, and a bidirectional iterator (`first`/`last`/`next`/`prev`/`seek_ge`/`seek_lt`/`seek_prefix_ge`) with bounds and bloom and block-property filters.
- Engine: a WAL with group commit, failover and recycling; leveled, multilevel and read-triggered compactions with a concurrent scheduler; block and table caches; value blocks and blob files; `checkpoint`, `ingest`, `excise`; `Metrics`, `EventListener`, `check_consistency`.
- VFS: `DiskFs`, `MemFs` for tests, and pluggable `RemoteStorage` for shared sstables.
- CLI: `pebbledb sstable|wal|manifest dump`, `db get|scan|lsm`, `find`, `bench`.

**Example:**
```rust
use pebbledb::{Db, Options};

fn main() -> pebbledb::Result<()> {
    let db = Db::open("/tmp/mydb", Options::default())?;
    db.set(b"hello", b"world")?;
    assert_eq!(db.get(b"hello")?, Some(b"world".to_vec()));

    let snap = db.snapshot();                 // consistent read view
    db.set(b"hello", b"again")?;
    assert_eq!(snap.get(b"hello")?, Some(b"world".to_vec()));

    let mut it = db.iter()?;
    it.first()?;
    while it.valid() {
        println!("{:?} => {:?}", it.key(), it.value());
        it.next()?;
    }
    Ok(())
}
```

**Gotchas:**
- Methods take `&self` (the `Db` is internally synchronized). Values come back as owned `Vec<u8>`.
- `Db::open_read_only` exists for inspecting a database another process owns.
