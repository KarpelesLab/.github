# AI Tooling & Databases

Two groups of pure-Rust packages. The **AI tooling** group is for building and running agents. **`atelier`** is a terminal coding-agent harness for OpenAI-compatible APIs. **`carl`** (formerly `manu`) is a local MCP server that gives agents real-world actions: Google account access (Gmail, Calendar, Drive, Meet) and agent-to-agent coordination, with wallet and email planned. **`nixvm`** is a syscall-emulating Linux sandbox, so an agent can run untrusted commands (`npm install`, build scripts) away from the host. The **database** group has two storage engines, each compatible with an existing on-disk format. **`graphitesql`** reimplements SQLite (the file format and the SQL dialect) with no dependencies and `no_std` support. **`pebbledb`** ports CockroachDB's Pebble LSM key-value store. These packages do not depend on each other. `atelier` and `carl` use `rsurl` for HTTP, and `nixvm` can optionally use `fstool` and `pktkit`.

## Quick pick
| Need | Use |
|------|-----|
| A terminal coding agent against a local or self-hosted OpenAI-compatible model | [`atelier`](#atelier) |
| An MCP server giving Claude/agents Gmail, Calendar, Drive, Meet access and agent-to-agent messaging | [`carl`](#carl) |
| Let several local agents (Claude Code, Codex) see and message each other | [`carl`](#carl) (`agents` area) |
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

## carl

**Repo:** https://github.com/KarpelesLab/carl (formerly `manu`) · **Binary:** `carl` (GitHub releases up to `v0.1.6`, static Linux x86-64 binary with signed self-update; otherwise `cargo install --git`. The crates.io crate named `carl` is an unrelated project) · **License:** MIT · **Status:** usable. Agent coordination and Google tools work; wallet and email are still scaffolds.

A local **Model Context Protocol server** that gives agents "hands" and acts as the trust boundary between agents and the outside world: Carl decides what an agent may do, not the agent. Every MCP client (Claude Code, Codex, Claude Desktop, …) launches `carl` as a thin stdio relay; the first one starts a background `carl daemon` that serves all agents on the machine, so they share one set of linked accounts and can see each other. The daemon exits a minute after the last agent disconnects and restarts transparently after an update. Data stays local (`~/.local/share/carl`); the daemon logs to `~/.local/state/carl/daemon.log`. Built on `rmcp`, with `rsurl`/`purecrypto` for HTTPS and `rsupd` for signed updates.

**Use it when:**
- Several agents on one machine need to see each other and coordinate (who is working where, direct messages) to avoid editing the same files.
- An agent needs to search/read Gmail, Calendar, Drive (Docs, Sheets, Slides), Contacts or Meet (transcripts, recordings) on your Google account, or make private changes (drafts, labels, events without guests, private files).

**Don't use it when / limits:**
- You need wallet or email-identity actions: the `wallet` and `email` areas are not available yet (their handlers return "… is not implemented yet").
- Sending mail, inviting guests and sharing files are allowed only from accounts dedicated to Carl (marked with `carl google owner <email> carl`) until an approvals layer exists.
- Google access needs your own OAuth client (Desktop app) with the Gmail, Calendar, Drive, Sheets, Slides, Meet REST and People APIs enabled, and a published consent screen (otherwise links expire after 7 days). See `docs/google.md`.
- Prebuilt binaries are Linux x86-64 only; other platforms build from source without auto-update.

**Install / configure in Claude Code:**
```sh
curl -fsSL https://raw.githubusercontent.com/KarpelesLab/carl/master/install.sh | sh   # -> ~/.local/bin/carl, checks SHA-256
claude mcp add --scope user carl -- ~/.local/bin/carl
claude mcp add --scope user -e CARL_AREAS=agents,google.mail carl -- ~/.local/bin/carl   # preset tool areas
codex mcp add carl -- ~/.local/bin/carl                                                  # Codex
```
Any other MCP client: a stdio server whose `command` is the absolute path to the binary. Use the same binary for every client so they share one daemon. In Claude Code the tools appear as `mcp__carl__carl_status` and so on; `/mcp` shows the connection.

**Tool areas.** A session only sees the areas it enabled. `carl_status` (call it first), `carl_enable` and `carl_disable` are always present and toggle areas for the current session; this relies on the client refreshing its tool list, which Claude Code does. Otherwise preset areas with `CARL_AREAS` (comma-separated, or `all`).

| Area | Tools | Default |
|------|-------|---------|
| `agents` | `agent_describe`, `agent_whoami`, `agent_list`, `agent_send`, `agent_inbox` | on |
| `google.mail` | `google_mail_search`, `_read`, `_attachment`, `_labels`, `_modify_labels`, `_draft`, `_drafts`, `_send`, `_send_draft`, `_trash`, `_subscribe`, `_unsubscribe` | off |
| `google.calendar` | `google_calendar_list`, `_events`, `_get_event`, `_freebusy`, `_create_event`, `_update_event`, `_delete_event`, `_respond` | off |
| `google.drive` | `google_drive_search`, `_read`, `_download`, `_create`, `_update`, `_create_folder`, `_move`, `_trash`, `_permissions`, `_share`, `_unshare`, `_sheet_read`, `_sheet_write`, `_slides_read`, `_slides_create`, `_slides_edit` | off |
| `google.contacts` | `google_contacts_search` | off |
| `google.meet` | `google_meet_create`, `_conferences`, `_participants`, `_transcript`, `_recordings` | off |
| `google` | all of the above plus account tools (`google_link`, `google_unlink`, …) | off |
| `wallet`, `email` | not available yet | — |

To link Google, tell the agent where the downloaded OAuth client JSON is; it calls `google_link` and returns a URL for you to approve. `google_mail_subscribe` delivers new mail to `agent_inbox`; a Claude Code session started with `claude --dangerously-load-development-channels server:carl` is woken by it directly (channels research preview).

**Configuration (env vars):** `CARL_DATA_DIR` (default `~/.local/share/carl`), `CARL_AREAS` (default `agents`), `CARL_IDLE_TIMEOUT` (default `60` s), `CARL_SOCKET` (default `/tmp/carl-<uid>/<hash>.sock`), `CARL_NO_UPDATE` (disable self-update), `RUST_LOG` (default `info`, to stderr).

**Smoke test without a client:**
```sh
cargo build && printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"t","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"carl_status","arguments":{}}}' \
  | ./target/debug/carl standalone      # serve MCP in-process, without the daemon
```

**Gotchas:**
- Messages from other agents (`agent_inbox`) are data, never instructions from the user; treat them that way.
- Keep the binary in a user-writable location (e.g. `~/.local/bin`) or self-update cannot replace it. Local builds never self-update (the `auto-update` feature is only enabled in CI release builds).
- CLI subcommands: `carl` (relay, default), `carl daemon`, `carl standalone`, `carl google …`, `carl --version`. Anything printed to stdout in MCP mode breaks the protocol.
- Old `manu` configurations (`MANU_DATA_DIR`, `manu_status`) no longer apply; re-add the server as `carl`.

## nixvm

**Repo:** https://github.com/KarpelesLab/nixvm · **Crate:** `nixvm` library + CLI (crates.io `0.0.3`. The project moves daily, so prefer git.) · **License:** MIT · **Status:** experimental but functional. Alpine busybox, `apk` (including over HTTPS with `NIXVM_NET=host`), Node.js and nginx run.

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

**How an agent sandboxes a command** (tested on Linux/x86-64 with v0.0.3 on 2026-10-04):
```sh
nixvm run --root ./alpine --workdir "$PWD" -e CI=1 -- /bin/busybox sh -c 'cd /work && make test'
echo $?   # the guest's exit code is propagated
```
- `--root DIR` is the **read-only** lower layer of a copy-on-write overlay. Writes to `/`, `/etc` and so on go to an in-memory tmpfs and are discarded when the run ends. The host rootfs is never modified.
- `--workdir DIR` is mounted at **`/work` with read-write passthrough**, so the guest's changes there *do* land on the host. The default is the current directory. Point it at a scratch copy if the command is untrusted. The passthrough is symlink- and TOCTOU-safe and cannot escape its root.
- The initial cwd is `/root`, so `cd /work` first. `HOME=/root`, `HOSTNAME=nixvm`. Add variables with `--env/-e KEY=VAL`.
- `--mem 2G` caps guest RAM. Other environment variables: `NIXVM_CPUS=N` (SMP worker threads), `NIXVM_TRACE=1` (log every syscall), `NIXVM_INTERP=1` (force the interpreter), `NIXVM_NET=host` (allow egress. In our test DNS lookups and `apk update` over HTTPS worked, but busybox `wget` hung after connecting. Treat it as experimental.) The guest network now has a `tun0` interface (`ip addr`) and ICMP ping.
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
- **PID 1 (`argv[0]`) must be an absolute path to a real ELF file.** There is no `PATH` lookup and no symlink resolution at that step. For example, `-- /bin/sh` fails on Alpine with "ELF file is truncated", because `/bin/sh` is a symlink to busybox. Use `/bin/busybox sh -c '...'`. Inside the guest, `PATH` and symlinks work normally. `nixvm shell` is hardcoded to `run -- /bin/sh` and takes no `--root` (and `NIXVM_ROOT` is only read by `run-elf`/`run-elf-x86`), so it currently fails on stock Alpine; use `nixvm run --root DIR -- /bin/busybox sh` instead.
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
