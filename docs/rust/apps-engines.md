# Applications & Engines

End-user programs and large language/runtime engines built on top of the KarpelesLab pure-Rust foundation crates (`purecrypto`, `rsurl`, `compcol`, `puremp`, `z3rs`, `oxideav`, `puressh`). Several of them are layered: `argus` (web browser) runs page scripts in `kataan` (JavaScript engine) and fetches over `rsurl`; `puregit` and `cterm` reuse `rsurl`/`puressh`; `mathesis` is a WASM front end to `puremp` + `z3rs`; `cadlab` (headless electronics CAD for agents) builds on `polyclip` and, optionally, the OxideAV 3D crates. Next to it sit `guide` (a place-and-route engine shared by PCBs and FPGAs, on `polyclip`) and the `csx` + `ems` pair (CSXCAD-compatible models and an FDTD solver replacing openEMS, with HDF5 output through `hdf5rs`). `nanami` synthesizes speech by simulating a vocal tract, with OxideAV for audio I/O. Most are applications first, but `kataan`, `fstool`, `puregit`, `mathesis`, `cadlab`, `ems`, `csx`, `guide` and `cterm-core` are also usable as library crates. Maturity varies widely, from production-grade (`fstool`, `kataan`) to experimental (`argus`, `origami`, `guide`, `nanami`).

## Quick pick

| Need | Use |
|------|-----|
| Run JavaScript/WASM from Rust (sandboxed, host functions, no V8/C deps) | [`kataan`](#kataan) |
| Evaluate a JS snippet from the shell, or embed JS through a C ABI | [`kataan`](#kataan) |
| Fetch a web page headlessly and get its text, links, forms, tables, a11y tree, JSON-LD, or a PNG screenshot | [`argus`](#argus) |
| Build/inspect/modify disk images (ext4, FAT, NTFS, XFS, APFS, ISO, SquashFS, qcow2, LUKS…) without root or loop mounts | [`fstool`](#fstool) |
| Transcode an archive or image in memory (tar.gz → ext4, ISO → tar, …) from Rust | [`fstool`](#fstool) |
| Read/write git repositories, clone over HTTPS/SSH, serve git, with no libgit2 or C | [`puregit`](#puregit) |
| Run PHP 8.5 scripts without a PHP install (94% of php-src tests pass) | [`goro-rs`](#goro-rs) |
| Exact arithmetic, number theory, polynomial derivatives, small SMT queries with Wolfram-like syntax | [`mathesis`](#mathesis) |
| Drive terminal sessions programmatically (spawn a PTY, send keys, read the screen as text) over gRPC | [`cterm`](#cterm) |
| Parse VT100/ANSI output into a screen grid in Rust | [`cterm`](#cterm) (`cterm-core`) |
| Headless PCB/EDA driven by CLI, Rust or MCP: circuit, ERC, autoroute, DRC, Gerber/IPC-2581 | [`cadlab`](#cadlab) |
| Electromagnetic FDTD simulation (antennas, microstrip, waveguides), or run openEMS XML models without openEMS | [`ems`](#ems) |
| Read/write/mesh CSXCAD (openEMS) geometry and material models | [`csx`](#csx) |
| Deterministic multi-net router over a grid/graph, VTR FPGA routing, KiCad-compatible clearance checks | [`guide`](#guide) |
| Offline speech/singing synthesis from LLM-friendly markup (physical vocal-tract model, research prototype) | [`nanami`](#nanami) |
| Physics-based protein structure building, minimization, Langevin/REMD dynamics, PDB output | [`origami`](#origami) |
| X11 server that forwards to macOS/Windows/X11 displays | [`x11anywhere`](#x11anywhere) |

## argus

**Repo:** https://github.com/KarpelesLab/argus · **Crate:** `argus` binary + 18 internal `argus-*` crates (git only, version `0.0.0`, never published) · **License:** MIT · **Status:** experimental (loads, lays out, scripts, and renders real pages; Phase 2-3 of roadmap)

A web browser written in Rust: HTML parser + DOM, CSS cascade, block/inline/flex/grid/table layout, CPU rasterizer (deterministic pixels), multi-process sandbox (content process never touches sockets). JS runs in `kataan`, networking uses `rsurl` + `purecrypto` TLS, images/text go through `oxideav`. Its main value to an agent today is the **headless dump CLI**: one command fetches a URL and prints structured page content.

**Use it when:** you want a no-C, no-Chromium way to fetch a page and extract rendered text, links, headings, forms, tables, metadata, JSON-LD/microdata, an accessibility tree, or a PNG screenshot.
**Don't use it when / limits:** you need full browser fidelity or interactive automation. The CDP-like `argus-headless` API from the design docs is not built yet; there is no stable library API. JS/DOM is partial: DOM mutations and click handlers work via a JS-side `document` shim, but timers/async and reading back layout do not. No floats/positioning yet. The windowed GUI is macOS-only; on other OSes the default mode is a headless verifier. CI runs on macOS only.

**Install / run:**
```sh
git clone https://github.com/KarpelesLab/argus && cd argus
cargo build --release          # binary: target/release/argus (MSRV 1.85)
```

**Headless CLI (all exit after printing; `--url=` omitted = built-in sample page):**
```sh
argus --url=https://example.com --dump-text        # innerText-style rendered text
argus --url=https://example.com --dump-links       # link text + resolved href
argus --url=https://example.com --dump-headings    # heading outline
argus --url=https://example.com --dump-forms       # forms and controls
argus --url=https://example.com --dump-tables      # tables as TSV
argus --url=https://example.com --dump-json        # JSON summary: title/headings/links
argus --url=https://example.com --dump-meta        # title/lang/OpenGraph etc.
argus --url=https://example.com --dump-jsonld      # JSON-LD blocks
argus --url=https://example.com --dump-microdata   # microdata items as JSON
argus --url=https://example.com --dump-images      # src/alt/dimensions
argus --url=https://example.com --dump-a11y        # accessibility tree
argus --url=https://example.com --dump-dom         # parsed DOM
argus --url=https://example.com --dump-domtree     # DOM as nested JSON (CDP-style)
argus --url=https://example.com --dump-page=/tmp/p.png --width=1280 --height=2000 --scroll=0
argus --download=https://example.com/file.zip --out=/tmp/dl   # else $ARGUS_DOWNLOADS or ~/Downloads
argus --eval='[1,2,3].map(x => x*2).join()'        # run JS in kataan, no DOM; prints console + "=> value"
argus                                              # windowed browser (macOS); headless verifier elsewhere
```

**Key features:** screenshot viewport via `--width` (default 800, max 16384) / `--height` (default 1600); `--width` also drives `@media` queries. Errors go to stderr with exit code 1.

**Gotchas:**
- The dump functions live in the internal `argus-browser` crate (`argus_browser::dump_text(url: Option<&str>) -> io::Result<String>` etc.) but are not a supported public API; prefer the CLI.
- The single binary re-executes itself with `--role=content|net|storage`; do not pass `--role` yourself.
- Pages that build their content with async JS/timers will be incomplete.

## kataan

**Repo:** https://github.com/KarpelesLab/kataan · **Crate:** `kataan` (crates.io `0.0.9`) · **License:** MIT · **Status:** usable (about 99.99% of tc39/Test262 passes; JIT and WASM tiers still being built)

A JavaScript (ECMAScript) engine in pure Rust with a tree-walking interpreter plus a register bytecode VM, ES modules, async/await, generators, Proxy, BigInt, Intl, Temporal, RegExp (in-house), and a `no_std` WebAssembly engine. The language core is sans-I/O and `no_std + alloc`; the host runtime (event loop, timers, console, URL, streams, `node:` compat, `fetch`, WebCrypto) is a separate feature-gated layer. Available as a Rust library, a C library (`kt_eval`), and the `kataan` CLI/REPL. `unsafe_code = "deny"` outside the FFI and a few audited VM primitives. There is a browser playground at https://karpeleslab.github.io/kataan/.

**Use it when:** you need to run untrusted or user-supplied JS inside a Rust program, expose Rust functions to scripts, or evaluate JS in a sandbox with no network or filesystem by default.
**Don't use it when / limits:** you need V8-level performance on hot object code (JIT is x86-64/Linux only and covers numeric paths). No DOM (see `argus`). `Interp` is `!Send` (it holds `Rc`s).

**Add it:**
```toml
[dependencies]
kataan = "0.0.9"
# no_std language core only:
# kataan = { version = "0.0.9", default-features = false, features = ["alloc"] }
```

**Cargo features:** default (crates.io 0.0.9) = `std, regex, intl, intl-tz-names, module, host, cli, crypto`; on master the default also includes `jit` (x86-64 Linux only, inert elsewhere) and `pool-alloc` (the pure-Rust pooling allocator the CLI installs). Optional: `fetch` (fetch/Node http over `rsurl`), `jit` (in 0.0.9), `ffi` (C ABI; build with `cargo rustc --lib --release --features ffi --crate-type cdylib`, header `include/kataan.h`). Library users can drop `cli`.

**CLI:**
```sh
cargo install kataan
kataan run -e 'console.log([1,2,3].map(x => x*x))'   # or: kataan run file.js
kataan hostrun file.js      # with timers + event loop drained to quiescence
kataan compile file.js -o file.ktbc && kataan run file.ktbc   # serialized bytecode
kataan lex|parse|disasm -e '...'   ;   kataan repl
```

**Example (embedding with a host function; from `examples/embed_host_fn.rs`):**
```rust
use kataan::parser::Parser;
use kataan::{Ctx, Interp};

// The program must outlive the interpreter (Interp<'a> borrows the AST).
let program = Parser::parse_program("hypot(3, 4) + 1").unwrap();
let mut interp = Interp::new();
interp.register_global_fn("hypot", 2, |cx: &mut Ctx, _this, args| {
    let a = cx.to_number(args.first().copied().unwrap_or(cx.undefined()))?;
    let b = cx.to_number(args.get(1).copied().unwrap_or(cx.undefined()))?;
    Ok(cx.number((a * a + b * b).sqrt()))
});
let v = interp.run(&program).unwrap();
assert_eq!(interp.display(v), "6");
print!("{}", interp.output()); // captured console.log output
```

One-shot evaluation, returning `(console_output, completion_value)` as strings:
```rust
let (console, value) = kataan::nbvm::execute("console.log('hi'); 6 * 7").unwrap();
```

**Gotchas:**
- `interp.display(value)` and `interp.realm().to_display_string(value)` both stringify a value (the README uses the latter).
- crates.io `0.0.9` (2026-09-05) lags master by roughly 470 commits (as of 2026-11-07). On master `Interp::run` executes on the register bytecode VM (`nbvm`), the JIT is a default feature and has deopt exits. The example above compiles against both. Use a git dependency for current behaviour.
- `Ctx` also has `new_object`, `set`, `get`, `call`, `construct`, `new_array`, `type_error`, `set_native_state`/`native_state`, `deferred` (promises). Install host globals with `kataan::host::install_all(&mut interp)`, then call `kataan::host::timers::run_event_loop`.
- Resource caps are set in `kataan::Limits` (`Interp::new_with_limits`, `nbvm::execute_with_limits`). There is no JS step budget. Stop runaway scripts with the `kataan::interrupt` watchdog (`nbvm::execute_typed_interruptible`).

## goro-rs

**Repo:** https://github.com/KarpelesLab/goro-rs · **Crate:** `goro` binary + `goro-*` workspace crates (git only; the crates.io `goro` is an unrelated project) · **License:** none declared (no LICENSE file, and the workspace Cargo.toml has no `license` field) · **Status:** active, clean-room rewrite (94.0% of the php-src 8.5.4 tests that run pass: 18,149 pass, 1,160 fail, 2,263 skipped, as of 2026-11-02)

A clean-room implementation of PHP 8.5 in Rust, written from scratch on 2026-10-28 from PHP's documentation, specifications and black-box behaviour. The earlier code base, written with php-src as a reference, is kept on the `old` branch and is out of bounds for the new one. It covers classes, enums, closures, generators, fibers, attributes, references, exceptions, output buffering, streams, SPL, Reflection and serialization, with a CGI SAPI (`goro --cgi`, or run as `php-cgi`). The rule is pure Rust with no C or FFI to C libraries (the `libc` crate for syscalls is allowed); dependencies are mostly KarpelesLab crates (`purecrypto` 0.10, `graphitesql` for PDO/sqlite3, `puremp` for gmp/bcmath).

**Use it when:** you need to run PHP scripts without a system PHP, run the php-src `.phpt` suite against a PHP binary, or embed a PHP engine in Rust.
**Don't use it when / limits:** you need full php-src parity today, or PHP's speed on call-heavy code: on the repo's `bench/` scripts goro takes 1.2-5.8x PHP 8.5's time (function calls are the slowest). There is no opcache optimizer or JIT, and phpdbg tests are skipped. The API is internal and unstable.

**Install / run:**
```sh
git clone https://github.com/KarpelesLab/goro-rs && cd goro-rs
cargo build --release                       # binary: target/release/goro
./target/release/goro script.php
./target/release/goro -r 'echo PHP_VERSION, "\n";'   # "<?php " is prepended
./target/release/goro --test -q -j 8 --results new.txt /path/to/php-src/tests   # .phpt runner
./target/release/goro --test-compare base.txt new.txt                          # regressions
```

**Workspace crates:** `goro-engine` (compiler, VM, the `goro` binary's entry points), `goro-syntax` (lexer, parser, AST; tokens match `token_get_all()`), `goro-runtime` (values, arrays, objects, conversions), `goro-phpt`, `goro-regex` (PCRE2 10.46-compatible), `goro-charset` (mbstring/iconv), `goro-xml`, and `goro-ext-{standard,date,json,hash,openssl,zlib,gmp,bcmath,curl,intl,sqlite,zip,phar,gd,exif,sodium,random,filter,fileinfo,uri,tidy}`.

**Example (embedding; verified to compile and print `HI`):**
```rust
use goro_engine::{RunOptions, Script};

// Like `goro -r`: "<?php " is prepended and output goes to stdout. Returns the exit status.
let status: i32 = goro_engine::run(RunOptions::new(Script::Code(b"echo strtoupper('hi'), PHP_EOL;".to_vec())));
```
`Script` also has `File(Vec<u8>)` and `Stdin`. `RunOptions` has `ini` (`-d name=value` pairs), `args`, `modules: Vec<&'static Module>` and `lines`. For a lower-level start, `goro_engine::startup(ini, modules) -> Vm` returns a VM with the extensions registered.

**Gotchas:**
- Depend on the crates by git (`goro-engine = { git = "https://github.com/KarpelesLab/goro-rs" }`). Code written against the old `goro-core`/`goro-parser` crates no longer builds; those crates exist only on the `old` branch.
- No license is declared anywhere in the repo; ask before redistributing.

## mathesis

**Repo:** https://github.com/KarpelesLab/mathesis · **Crate:** `mathesis` (git only) · **License:** MIT · **Status:** usable (live app; language still growing)

A Mathematica-style notebook running entirely in the browser (**https://karpeleslab.github.io/mathesis/**). The Rust crate owns a Wolfram-style lexer/parser/evaluator and hands all math to `puremp` (exact integers, rationals, reals, complex, matrices, number theory, elliptic curves, LLL) and `z3rs` (SMT). The UI is Vue 3 + KaTeX, compiled to WASM, with the engine running in a Web Worker. Notebooks are shareable via URL hash (`#c=` / `#n=`), with no server involved.

**Use it when:** you (or a user) need exact answers such as `2^128`, `Factor[360]`, `PrimeQ[2^61-1]`, `N[Pi, 40]`, `D[x^3 + x, x]`, `Det`/`Inverse`/`Eigenvalues`, `Solve[x^2 == 2, x]`, `SatisfiableQ[...]`, `Maximize[...]`, raw `SMT["..."]`, or plotting (`Plot`, `Plot3D`). Point a human at the live URL, or call the crate from Rust.
**Don't use it when / limits:** you need a general computer algebra system. There is no symbolic simplifier (`2·Pi` collapses to a decimal), `D` handles polynomials only, and z3rs decides only linear arithmetic (nonlinear constraints come back "unknown").

**Add it (as a Rust library; crate type includes `rlib`):**
```toml
[dependencies]
mathesis = { git = "https://github.com/KarpelesLab/mathesis" }
```

**API:** `mathesis::evaluate(input: &str) -> String` returns a JSON string: `{"ok":true,"text":"…","tex":"…"[,"approx":"…"]}`, `{"ok":false,"error":"…"}`, or variants carrying `graphics` (plots), `solutions` (Solve tables), or `plain` (SMT output). `mathesis::version()`.

**Example:**
```rust
let r = mathesis::evaluate("Fibonacci[100]");
assert!(r.contains("354224848179261915075"));
let r = mathesis::evaluate("1/2 + 1/3");   // {"ok":true,"text":"5/6","tex":"\\frac{5}{6}",...}
```

**Gotchas:**
- Session variables (`x = 5`) and `%` (the last result) are global engine state, shared by every call in the process.
- Build the web app with `wasm-pack build --target web --out-dir frontend/src/pkg --release`, then `cd frontend && npm install && npm run dev`.

## puregit

**Repo:** https://github.com/KarpelesLab/puregit · **Crate:** `puregit` (crates.io `0.0.2`) · **License:** MIT · **Status:** usable (clones GitHub byte-identically; puregit-made history passes `git fsck`; roadmap milestones complete)

A from-scratch git in Rust, in the spirit of libgit2: objects (SHA-1/SHA-256), loose + packed ODB with deltas, pack read/write, refs/packed-refs/reflog, index v2/v3, config, porcelain (`add`/`rm`/`mv`/`commit`/`status`/`branch`/`checkout`/`merge`/`diff`/`reset`/`gc`/`fsck`/worktrees), Git LFS, index v4, SHA-256 repositories, and the smart protocol (including protocol v2 push) as **client and server** over HTTP(S) (`rsurl`), SSH (`puressh`), and `git://`. The core is `no_std + alloc`, filesystem access goes through a `Vfs` trait, and the protocols are sans-IO. There is no C and no `*-sys` crate.

**Use it when:** an agent or tool needs to create commits, inspect history, or clone/fetch/push without shelling out to `git` or linking libgit2. It also fits cases where you need to serve git (upload-pack/receive-pack handlers) or run git logic over an in-memory VFS.
**Don't use it when / limits:** you need the full breadth of git porcelain (rebase, interactive tools, etc.). OpenPGP keyring management is a stated non-goal. The API is pre-1.0.

**Add it:**
```toml
[dependencies]
puregit = { version = "0.0.2", features = ["http"] }   # MSRV 1.95
```

**Cargo features:** default `std, client`. Optional: `http` (smart-HTTP over rsurl), `ssh` (over puressh), `server` (upload-pack/receive-pack), `signing` (SSHSIG verify), `ffi` (libgit2-shaped C ABI), `full`. `--no-default-features` gives the `no_std` core.

**Example:**
```rust
use puregit::{Repository, ObjectType};

let repo = Repository::init("/tmp/demo")?;
let id = repo.write_object(ObjectType::Blob, b"hello\n")?; // ce0136...
std::fs::write("/tmp/demo/a.txt", "hello\n")?;
repo.add_path("a.txt")?;
let sig = repo.signature_now(); // from user.name/user.email config
let commit = repo.commit(b"first commit\n", sig.clone(), sig)?;

// Clone over HTTPS (feature "http"):
use puregit::{oid::HashAlgo, transport::http::HttpTransport};
let mut t = HttpTransport::new("https://github.com/octocat/Hello-World.git", HashAlgo::Sha1);
let cloned = puregit::client::clone("/tmp/hello", &mut t)?;
println!("{}", cloned.head_id()?);
# Ok::<(), Box<dyn std::error::Error>>(())
```
Also: `Repository::open`, `read_object`, `head_id`, `refs()`, `index()`, `create_branch`, `checkout`, `reset`, `repack`, `prune`, and `client::{fetch, push(repo, transport, local_ref, remote_ref)}`. For SSH, use `transport::ssh::SshTransport::from_url(url, HashAlgo::Sha1)` with `.with_password()` or `.with_key(path, passphrase)`.

**CLI:** `cargo install puregit --features http,ssh` installs a binary named **`puregit`** (commands include `init`, `add`, `rm`, `mv`, `commit`, `log`, `show`, `status`, `branch`, `checkout`, `switch`, `reset`, `merge`, `diff`, `tag`, `worktree`, `clone`, `gc`, `fsck`, `lfs`, `cat-file`, `rev-parse`). The CLI has no `fetch`, `push` or `pull` commands (only `lfs pull`); use the library's `client::{fetch, push}` for those. Interop is tested against git and libgit2.

## fstool

**Repo:** https://github.com/KarpelesLab/fstool · **Crate:** `fstool` (crates.io `0.5.2`) · **License:** MIT · **Status:** production-ready for most formats (outputs are cross-validated against `e2fsck`, `fsck.vfat`, `xfs_repair`, `qemu-img`, `cryptsetup`, `sgdisk`). MSRV 1.95

Build, inspect, modify, and convert disk images and filesystem images entirely in userspace, **without root, loop devices, or mounts**. Read/write support: ext2/3/4, FAT12/16/32, exFAT, XFS, NTFS, HFS+, HFS, APFS (partial), F2FS, AFFS, littlefs, SquashFS, cramfs, ISO 9660, UDF, GRF, tar, zip (including encryption and split volumes), cpio, ar. Read-only: cab, 7z, rar5, lha, arc, sit, lzx. Partition tables: MBR, GPT, Apple Partition Map (read). Containers: qcow2 (read/write, compression, backing files, encryption), LUKS1/2 (read/write/create), DMG (read; writes UDRO/UDCO/UDZO/UDBZ/ULMO/ULFO and encrypted `encrcdsa` v2), `.sparseimage`/`.sparsebundle` (read/write/create). All dependencies are pure Rust (`compcol`, `purecrypto`). There is also a browser app at https://karpeleslab.github.io/fstool/.

**Use it when:** an agent needs to produce a bootable/partitioned disk image from a directory or TOML spec, look inside an image or archive, copy files in or out, convert between formats (e.g. ext4 ↔ tar, OCI-style layer merge with whiteouts), or do all of this in memory from Rust.
**Don't use it when / limits:** see the README "Limitations" section. Highlights: ext4 writer has no `flex_bg`; APFS in-place edits are whole-file and not `fsck_apfs`-clean; the NTFS reader lacks compressed/encrypted `$DATA`; DMG encrypted v1 (`cdsaencr`) is unsupported and the UDIF writer emits single-segment, unencrypted images (segmenting and encryption are separate operations); the F2FS/SquashFS/ISO writers are build-once.

**Install (CLI):**
```sh
cargo install fstool
fstool create -t ext4 ./rootfs -o out.img -O block_size=4096,volume_label=ROOT
fstool build disk.toml -o disk.img          # [image] + [[partitions]] spec (GPT/MBR)
fstool info out.img        ;  fstool info disk.img:2      # :N = 1-indexed partition
fstool ls -R out.img /     ;  fstool cat out.img /etc/hostname
fstool add out.img ./file /file   ;  fstool rm out.img /file
fstool repack in.img out.tar      ;  fstool repack base.tar patch.tar flat.tar
fstool convert disk.img disk.qcow2 ;  fstool analyze ./dir --json
fstool create -t ext4 --size 1G -o s.img tree/ --encrypt --password-file pw   # LUKS2
```
Maintenance commands (0.5): `check`, `verify`, `checksum`, `resize-fs`, `resize`, `compact`, `reclaim`, `segment`/`join`, `udif-derez`/`udif-rez`, `unsign`, and `makehybrid` (hybrid ISO/HFS images). Other commands: `shell` (an SFTP-like REPL with `find`/`grep`; it needs a TTY, so avoid it in agent automation), `dd` (ddrescue-style copy), `resources` (HFS resource forks), `mount` (ext via FUSE; needs the `fuse` feature). Block devices (`/dev/sdX`) require `--force`.

**Add it (library, no CLI deps):**
```toml
[dependencies]
fstool = { version = "0.5", default-features = false, features = ["filesystems", "containers", "codecs"] }
```

**Cargo features:** one per format (`ext`, `fat`, `exfat`, `ntfs`, `xfs`, `apfs`, `hfs-plus`, `iso9660`, `squashfs`, `tar`, `archive`, `qcow2`, `luks`, `dmg`, …), bundles `filesystems`/`containers`/`codecs`, plus `spec` (TOML), `json`, `cli`, `wasm`, `fuse`. `fat`, `exfat`, and `littlefs` work in `no_std` without an allocator (embedded SD/flash drivers).

**Example (in-memory, no filesystem access):**
```rust
use fstool::memconv::MemImage;
use fstool::memedit::Workspace;

// Inspect / transcode any supported image or archive.
let mut img = MemImage::open(std::fs::read("in.tar.gz")?)?;
for e in img.list("/")? { println!("{} {} {}", e.kind, e.size, e.name); }
let ext4_bytes: Vec<u8> = img.convert("ext4")?;

// Build a fresh FAT image from scratch.
let mut ws = Workspace::new_filesystem("fat32", 64 << 20, "")?;
ws.mkdir("/boot")?;
ws.add_file("/boot/config.txt", b"arm_64bit=1\n".to_vec())?;
std::fs::write("sd.img", ws.export()?)?;
# Ok::<(), Box<dyn std::error::Error>>(())
```
Option structs and enums are `#[non_exhaustive]` since 0.5.1, so construct them through their builders or `Default` rather than struct literals. Also `fstool::memconv::probe(&[u8])` (reports compression, partition table, and FS kind) and the low-level `fstool::block::FileBackend` + `fstool::fs::ext::Ext::format_with` (see `examples/format_empty_ext2.rs`).

## cadlab

**Repo:** https://github.com/KarpelesLab/cadlab · **Crate:** `cadlab` (crates.io `0.0.2`; library plus `cadlab` binary) · **License:** MIT · **Status:** pre-alpha by version number, but broad in practice. Roadmap milestones M0–M5, M8 and M9 are done, and most of M6/M7. The README "Status" paragraph is stale: it says only `project.*` exists, but there are about 169 commands. MSRV 1.95.

Headless electronics CAD (EDA), built to be **driven by programs and AI agents**. It covers parts and BOM, circuit (netlist-first), ERC, auto-generated schematic, board, placement, autorouting, DRC, and fab outputs, with no GUI. The only visual output is rendering (PNG/SVG, isometric 3D). Every operation is a typed, JSON-Schema-described command in one **command registry**. The Rust library, the CLI and the MCP server are thin layers over that registry, so names, arguments, validation and errors are identical on all three. Coordinates are integer nanometres, and output is deterministic byte for byte (the router is seeded). Polygon geometry comes from [`polyclip`](i18n-math.md#polyclip). The code is clean-room: KiCad and freerouting are used only as external test oracles, never as code sources.

**Use it when:** an agent or script has to produce a manufacturable PCB end to end, or one stage of it (ERC, autoroute, DRC, Gerber/IPC-2581 export, fab-rule checks, BOM costing), without driving a GUI. It also fits when you need to import KiCad boards or netlists into a scriptable pipeline.
**Don't use it when / limits:**
- You need interactive editing, or KiCad-grade completeness for schematic capture. There is no `.kicad_sch` import: circuits come in as netlists, and schematics are always derived.
- The router combines grid maze routing with a gridless shape-based search under negotiated congestion (`router: grid|gridless|auto`, default `auto`: grid first, then gridless for what is left), plus shove and fanout. Its stated goal is parity with freerouting, which is still in progress.
- ODB++ is intentionally not implemented (licence terms). Use IPC-2581.

**Install / run:**
```sh
cargo install cadlab            # or prebuilt binaries: GitHub release v0.0.2 (linux x86_64/aarch64, macOS universal, Windows)
claude mcp add cadlab -- cadlab mcp      # register as an MCP server (stdio)
```

**CLI shape:** `cadlab [-p DIR] [--json] [--dry-run] [-q] <group> <action> [ARGS]`. The project is the `-p` directory, or the first `cadlab.toml` found walking up from the current directory. Positional arguments come from the schema (`cadlab describe <cmd>` shows the `cli:` line), and every other argument is a `--flag`. Lengths **must carry units** (`"0.2mm"`, `"8mil"`). Verified session:
```sh
cadlab project new led --targets jlcpcb && cd led
cadlab circuit add "R 1k 1% 0603"                       # generic part -> R1 (symbol + IPC-7351B footprint generated)
cadlab circuit add "LED red 0603"                       # -> D1 (pins K, A)
cadlab part create --category connector --id HDR2 --package "PinHeader 1x02" \
  --pins '[{"number":"1","name":"VCC","kind":"power_out"},{"number":"2","name":"GND","kind":"power_out"}]'
cadlab circuit add HDR2                                 # -> J1
cadlab net connect VCC J1.VCC R1.1                      # pins by name or number; ranges U1.PA0..PA7, buses DATA[0..7]
cadlab net connect LED_A R1.2 D1.A
cadlab net connect GND D1.K J1.GND
cadlab circuit erc                                      # "ERC clean"
cadlab circuit summary                                  # compact text netlist, made for LLM context
cadlab board outline --width 20mm --height 15mm
cadlab place auto
cadlab route all                                        # "routed 3/3 connections (100%)"
cadlab drc run                                          # exit 3 if errors
cadlab render board top.png --realistic top             # PNG or SVG
cadlab fab check jlcpcb
cadlab fab export jlcpcb                                # out/fab/jlcpcb/: Gerbers, drill, BOM, CPL, zip, fab-lock.json
cadlab export all                                       # generic Gerber X2/X3, drill, PnP, IPC-D-356A, IPC-2581
cadlab undo                                             # history persists in .cadlab/ across processes
```
These generic entry points work on every command:
- `cadlab describe [cmd|group]` prints the argument and output schema.
- `cadlab call net.connect '{"net":"VCC","pins":["J1.1"]}'` runs any command with JSON arguments (`-` reads them from stdin).
- `cadlab batch steps.jsonl` runs one `{"cmd":..,"args":{..}}` per line as a single all-or-nothing transaction and a single undo step.
- `cadlab history`, `undo` and `redo` manage the undo history.

`--json` prints exactly one object: `{"ok":true,"command","output","diagnostics"}` or `{"ok":false,"error":{"kind","code","message","subjects","hint"}}`. The `hint` field carries "did you mean" suggestions. Exit codes: `0` ok, `1` command error, `2` usage, `3` completed with ERC/DRC/check errors.

**Command groups** (about 169 commands; run `cadlab describe` for the full list):
- Project and library: `project`, `part` (`generic`, `create`, `search`, `import_kicad_sym`), `footprint` (IPC-7351B `generate`, `import_kicad`, 3D `model_set`), `lib` (shared user libraries, `import_kicad`), `catalog`.
- Circuit: `circuit` (`add`, `erc`, `lint`, `summary`, `export`, `import` from a KiCad netlist), `net`, `netclass`, `diffpair`, `lengthgroup`, `block` (reusable sub-circuits).
- Sourcing: `bom` (`resolve`, `check`, `cost`, `substitutes`, `export`).
- Board: `board` (`setup`, `outline`, `rules`, `stackup`, `hole`, `cutout`, `import_kicad`, `export_kicad`), `place` (`auto`, `near`, `align`, ...), `track`, `via`, `zone`, `keepout`, `impedance` (`calc`, and `solve` to fit a net class's width to a target impedance), `current`.
- Routing and checks: `route` (`all`, `nets`, `fanout`, `diffpair`, `tune`, `import_ses`), `drc`.
- Output: `render` (`schematic`, `board`, `board3d`, `footprint`, `symbol`), `schematic` (`export` to `.kicad_sch`).
- Export: `export` (`gerber`, `drill`, `pnp`, `ipc356`, `ipc2581`, `step`, `idf`, `idx`, `dsn`, `spice`, `all`; master adds `openems`).
- Simulation (master only, after the 0.0.2 release): `sim sparams` runs cadlab's built-in FDTD full-wave solver over a board region (stack, copper, vias, lumped ports, CPML) and writes a Touchstone file. `export openems` writes the same region as an openEMS model, which [`ems`](#ems) can run.
- Fab: `fab` (`list`, `check`, `compare`, `export`, `substitute`).
- Built-in fab profiles: jlcpcb, pcbway, oshpark, aisler, eurocircuits, seeed, nextpcb, pcbgogo, allpcb, elecrow, generic. You can add your own in `~/.config/cadlab/fab-profiles`.

**MCP server** (`cadlab mcp`, stdio, newline-delimited JSON-RPC, no async runtime):
- **Tools:** one tool per group (`project`, `circuit`, `route`, ...) with `{action, args, project?, dry_run?}`, plus `describe`, `call` (`{command, args}`) and `batch` (`{steps:[{cmd,args}]}`). Each group tool's description lists its actions and argument signatures.
- **Session:** start with `project` `new` or `open`. Later calls act on the current project unless `project` names another directory. Several projects can be open at once. Changes autosave; with `--no-autosave` they wait for `project.save`.
- **Results:** a text summary plus `structuredContent`. `render.*` results also include the PNG as MCP **image content**, so multimodal agents can look at the board. Long commands (route, part search) send progress notifications and honour `notifications/cancelled`.
- The server implements tools only. `docs/INTERFACES.md` describes MCP resources and prompts, but they are not implemented.

**Library:**
```toml
[dependencies]
cadlab = { version = "0.0.2", default-features = false }   # drop CLI/network/PNG/rayon/3D-model deps
serde_json = "1"
```
```rust
use cadlab::prelude::*;
use cadlab::commands::project;
use serde_json::json;

let dir = std::env::temp_dir().join("cadlab-demo");
let mut s = Session::new();
run(&mut s, project::New { path: dir, name: Some("demo".into()), description: None, targets: vec![] },
    RunOptions::default())?;                                   // typed command
for (cmd, args) in [
    ("circuit.add", json!({"part": "R 1k 1% 0603"})),
    ("circuit.add", json!({"part": "LED red 0603"})),
    ("net.connect", json!({"net": "LED_A", "pins": ["R1.2", "D1.A"]})),
    ("board.outline", json!({"width": "20mm", "height": "15mm"})),
    ("place.auto", json!({})),
    ("route.all", json!({"seed": 0})),
    ("drc.run", json!({})),
] {
    let out = registry().execute(&mut s, cmd, args, RunOptions::default())  // by name, same as CLI/MCP
        .map_err(|f| f.error)?;
    println!("{}: {}", out.command, out.summary);              // out.output = structured JSON
}
s.save()?;
```

**Cargo features:** the defaults are `cli` (clap and rpassword, needed for the binary), `net` (ureq for the DigiKey, Mouser and Nexar supplier APIs), `png` (tiny-skia; SVG needs no feature), `parallel` (rayon plus `polyclip/rayon`) and `models3d` (STL/OBJ/glTF/USDZ import through `oxideav-mesh3d`; master adds STEP and VRML).

**Gotchas:**
- `docs/INTERFACES.md` shows the *target* CLI, and some of its examples do not run. `cadlab erc` is really `cadlab circuit erc`, `board outline rect 50mm 30mm` is `board outline --width 50mm --height 30mm`, and `route --all` is `route all`. The `p.bom().add(..)` wrapper API in that file does not exist yet; use `run` or `Registry::execute`. Trust `cadlab describe`.
- Part IDs in the library use underscores (`LED_red_0603`). Mistyped names fail with a hint such as ``did you mean `LED_red_0603`?``.
- Supplier research (`part.search`, `bom.resolve/check/cost`, `fab check --parts`) needs providers. Configure them with `cadlab config digikey|mouser|nexar`, with environment variables (`DIGIKEY_CLIENT_ID`/`_SECRET`, `MOUSER_API_KEY`, `NEXAR_CLIENT_ID`/`_SECRET`), or with offline JSON catalogs (`CADLAB_CATALOGS`, `~/.config/cadlab/catalogs/*.json`, `catalog.import` for a JLCPCB/LCSC CSV). `CADLAB_OFFLINE=1` serves only cached answers. Credentials are never stored in the project.
- A project is a git-friendly directory: `cadlab.toml`, `circuit.json`, `board.json`, `bom.json` and `library/`. `.cadlab/` holds caches and undo history, and `out/` holds generated files. Fab-specific choices go only in `out/fab/<fab>/fab-lock.json`.
- crates.io cadlab 0.0.2 requires `polyclip ^0.0.4` (an exact pin under 0.0.x rules) while master uses 0.0.6, so a project that also depends on polyclip directly gets two copies unless the versions match.
- `sim.sparams` uses cadlab's own FDTD engine, not `ems`; the two share the openEMS model description, not code.

## ems

**Repo:** https://github.com/KarpelesLab/ems · **Crate:** `ems` (crates.io `0.0.5`; library plus the `ems`, `ems-nf2ff` and `ems-sar-calc` binaries) · **License:** MIT · **Status:** early development (milestones 1–4 done; parity with openEMS checked against a black-box openEMS; API still changing). MSRV 1.95

A clean-room FDTD (finite-difference time-domain) electromagnetic field solver, written as a replacement for [openEMS](https://openems.de). Models come from its companion crate [`csx`](#csx); `ems` adds mesh compilation, time stepping, boundaries (PEC/PMC, Mur, CPML, periodic/Bloch), sources and plane waves, lumped elements, probes, ports (lumped, microstrip, CPW, stripline, coax, rectangular/circular waveguide) with S-parameters and Z_in, field dumps, NF2FF far fields and SAR. Cartesian and cylindrical grids are supported. The `ems` binary takes openEMS's command line, so scripts that call `openEMS` (including its Python and Octave wrappers) can call `ems` instead. `unsafe` is forbidden. The README reports 1.5–10x openEMS's speed on vacuum/PML boxes and results within 1e-7…1e-3 of openEMS on its end-to-end fixtures.

**Use it when:** you need antenna, microstrip, filter or waveguide simulation from Rust or a script with no C++ toolchain, or you want to run existing openEMS XML models unchanged.
**Don't use it when / limits:** you need a stable API (0.0.x, early development), or features the status table marks missing: rectangular-waveguide ports on cylindrical grids, lossy backgrounds around an NF2FF box, and openEMS's tolerance for polyhedron surfaces within 1e-3 of a cell. It has one engine; `--engine=sse` and friends are accepted but run the same code.

**Add it:**
```toml
[dependencies]
ems = "0.0.5"     # default feature `xml` runs openEMS XML files
csx = "0.0.2"     # must be the same csx version ems uses, or the model types won't match
```

**Cargo features:** `xml` (default; openEMS XML via `csx/xml`), `hdf5` (HDF5 field dumps and probes through the pure-Rust [`hdf5rs`](data-formats.md#hdf5rs); also required by the `ems-nf2ff` and `ems-sar-calc` binaries), `hdf5-deflate` (gzip-compressed datasets).

**CLI (drop-in `openEMS`):**
```sh
cargo install ems --features hdf5      # installs ems, ems-nf2ff, ems-sar-calc
ems model.xml --numThreads=8           # writes openEMS's probe files and VTK/HDF5 dumps to the current directory
ems model.xml --no-simulation --debug-PEC   # same debug and statistics options as openEMS
```

**Example (README quick start; verified to compile and run):**
```rust
use csx::primitives::BoxShape;
use csx::{ContinuousStructure, Primitive, Property};
use ems::prelude::*;

let mut model = ContinuousStructure::new();
model.grid.delta_unit = 1e-3; // drawing unit: mm
model.grid.set_lines(0, (0..=40).map(f64::from))?;
model.grid.set_lines(1, (0..=40).map(f64::from))?;
model.grid.set_lines(2, (0..=20).map(f64::from))?;
let sub = model.add_property(Property::material("sub", csx::Material::new(4.3, 0.0)));
model.add_primitive(sub, Primitive::new(BoxShape::new([10., 10., 0.], [30., 30., 2.])))?;
let patch = model.add_property(Property::metal("patch"));
model.add_primitive(patch, Primitive::new(BoxShape::new([14., 14., 2.], [26., 26., 2.])))?;
let port = LumpedPort::new(1, 50.0, [17., 20., 0.], [17., 20., 2.], Axis::Z)
    .excite(1.0)
    .add_to(&mut model)?;

let fdtd = Fdtd::new()
    .waveform(Waveform::gaussian(2e9, 1e9))
    .boundaries(Boundaries::all(Boundary::pml(8)).with(Face::ZMin, Boundary::Pec))
    .end_criteria(EndCriteria::energy_decay_db(-40.0));
let mut sim = Simulation::new(&model, &fdtd)?;
for w in sim.warnings() { eprintln!("warning: {w}"); } // solver-aware mesh checks
let res = sim.run()?;
let z_in = port.analyze(&res, &[1e9, 2e9, 3e9])?.z_in(); // Vec<Complex64>
```
Run an openEMS XML file from Rust with `Simulation::from_xml`, or `cargo run --release --example run_xml -- model.xml --out DIR` in the repo (probes as CSV, dumps as `.vtr`).

**Gotchas:**
- Build and run with `--release`, as the README's commands do.
- Pin `csx` to the version `ems` depends on (`0.0.2` for ems 0.0.5). A git `csx` next to a crates.io `ems` gives two incompatible `ContinuousStructure` types.
- [`cadlab`](#cadlab)'s `sim sparams` has its own FDTD engine; `export openems` is how a cadlab board region reaches `ems`.

## csx

**Repo:** https://github.com/KarpelesLab/csx · **Crate:** `csx` (crates.io `0.0.2`) · **License:** MIT · **Status:** usable, pre-1.0. MSRV 1.95

The geometry and material model under [`ems`](#ems): a clean-room, zero-dependency library compatible with CSXCAD (openEMS's model format). It has primitives (box, sphere, cylinder and shells, polygon, extruded and revolved polygon, curve, wire, point, polyhedron from STL/PLY, expression-defined regions) with transformations and Cartesian/cylindrical coordinates; materials (anisotropic, position-weighted, Drude/Lorentz/Debye dispersive), metals, conducting sheets, excitations, probes, dumps and lumped elements; parametric values; priority-based point queries; rectilinear grids with automatic meshing; CSXCAD/openEMS XML read and write; tessellation and STL/PLY/VTK export; and discretisation onto the FDTD grid.

**Use it when:** you need to read, write, build or query CSXCAD/openEMS models, mesh a structure, or export it to STL/VTK, with or without running a simulation.
**Don't use it when / limits:** you need CSXCAD's exact bounding-box clipping of cylindrical wires (csx keeps the true geometry, a documented divergence).

**Add it:**
```toml
[dependencies]
csx = "0.0.2"                                          # default feature `xml`
# csx = { version = "0.0.2", default-features = false } # no XML reader/writer
```

**Example (README; verified to compile and run):**
```rust
use csx::primitives::BoxShape;
use csx::properties::Material;
use csx::{ContinuousStructure, CoordSystem, Property, TypeMask, Vec3};

let mut model = ContinuousStructure::new();
let fr4 = model.add_property(Property::material("FR4", Material::new(4.3, 0.0)));
model.add_primitive(fr4, BoxShape::new([-10.0, -10.0, 0.0], [10.0, 10.0, 1.6])).unwrap();

let xml = model.to_xml_string();                        // feature `xml`
let model = ContinuousStructure::from_xml_str(&xml).unwrap();
let hit = model.property_at(Vec3::new(0.0, 0.0, 0.5), CoordSystem::Cartesian, TypeMask::ANY);
assert_eq!(hit.unwrap().property.name, "FR4");
```
Repo examples: `info` (summarise a file), `patch_antenna` (build and mesh a parametric antenna), `export_vtk`, `discretize`.

## guide

**Repo:** https://github.com/KarpelesLab/guide · **Crate:** `guide` (git only; crates.io `0.0.0` is the empty scaffold) · **License:** MIT · **Status:** pre-alpha. Roadmap milestones M0–M2 and M4 are done, M3 (FPGA routing) is largely in; PCB routing (M5) and placement (M7+) are not started. MSRV 1.95

A placement and routing engine meant to be shared across every problem that reduces to connecting terminals through contended resources: PCBs, FPGAs, ASIC global routing, multi-agent path finding. Today it has the core model (nets, terminals, a `RoutingSpace` abstraction with grid and explicit-graph spaces), Dijkstra/A* with reusable search state, Steiner trees, a negotiated-congestion (PathFinder) multi-net engine with rip-up and reroute, deterministic parallel routing, budgets and cancellation, partial results with failure reports, and an independent validator that shares no code with the router. The `fpga` feature imports VTR routing-resource graphs, architectures and placed designs, with static timing analysis and timing-driven routing. The `pcb` feature adds a geometry layer over [`polyclip`](i18n-math.md#polyclip) (clearances, spatial index, design rules, DRC), and `kicad` reads KiCad boards for clearance checks, agreeing with KiCad 10's DRC on 3448 of its 3450 findings on the curated boards. `unsafe` is forbidden; results are identical across thread counts and platforms.

**Use it when:** you need a deterministic multi-net router over a grid or graph you define, VTR-style FPGA routing, or an exact PCB clearance check against KiCad boards.
**Don't use it when / limits:** you need PCB routing or placement (not implemented; [`cadlab`](#cadlab) has a working PCB autorouter today), a CLI (planned for 1.0), or a stable API.

**Add it:**
```toml
[dependencies]
guide = { git = "https://github.com/KarpelesLab/guide" }                          # core engine
# guide = { git = "https://github.com/KarpelesLab/guide", features = ["fpga"] }   # VTR import + timing
# guide = { git = "https://github.com/KarpelesLab/guide", features = ["kicad"] }  # geometry + KiCad boards (implies pcb)
```

**Example (two nets negotiating around a wall on a 2-layer grid; verified to compile and run):**
```rust
use guide::engine::{route, EngineOptions};
use guide::id::NetId;
use guide::net::{Net, Terminal};
use guide::space::grid::Grid;

let mut grid = Grid::new(16, 16, 2)?;                       // width, height, layers
for y in 2..14 { grid.set_blocked(8, y, 0, true)?; }
let t = |x, y| Terminal::at(grid.node_at(x, y, 0).unwrap()); // node_at returns Option
let nets = vec![
    Net { id: NetId(0), terminals: vec![t(1, 8), t(14, 8)] },
    Net { id: NetId(1), terminals: vec![t(8, 0), t(8, 15)] },
];
let outcome = route(&grid, &nets, EngineOptions::default())?;
assert!(outcome.is_success());                              // also: outcome.routes, total_cost, overused
```
`route_parallel`, `route_timed` and `route_timed_parallel` are the threaded and timing-driven variants; `guide::validate::check(space, nets, &outcome.routes)` re-checks a result independently. `Grid::builder(w, h, layers)` sets connectivity and orthogonal/diagonal/via costs.

**Gotchas:**
- The crates.io release is the M0 scaffold and its README still says "empty scaffold"; depend on git.

## nanami

**Repo:** https://github.com/KarpelesLab/nanami · **Crate:** `nanami` (git only; Cargo.toml has no description, license or repository fields) · **License:** MIT (LICENSE file) · **Status:** early research prototype (v0.1)

A physically based speech and singing synthesizer: instead of a neural network it simulates the lungs, vocal folds and a moving vocal tract (Liljencrants–Fant glottal source, Kelly–Lochbaum waveguide at 96 kHz with a nasal branch, Bernoulli airflow and turbulence noise). It is driven by NML, a small markup language designed for language models to write directly: text or IPA, emotion, prosody, voice quality and melody (`docs/NML.md` is written to be pasted into an LLM's system prompt). English and Japanese, singing, and MIDI input are supported. Audio I/O uses OxideAV crates (WAV, FLAC, playback, MIDI, spectrogram PNG). Voices: `nanami` (default), `female`, `male`, `child`.

**Use it when:** an agent needs offline, dependency-light speech or singing output where every parameter is interpretable, or you are experimenting with articulatory synthesis.
**Don't use it when / limits:** you need natural-sounding or highly intelligible speech. The README's Whisper-measured word error rate is about 31% (UTMOS 2.53, against about 4 for natural speech). Onset clusters and some stops are often misheard, kanji are not converted (give kana or IPA), and changing voice mid-document starts a new run with no cross-fade.

**Install / run:**
```sh
git clone https://github.com/KarpelesLab/nanami && cd nanami
cargo run --release -- say "Hello! <happy>I'm Nanami.</happy> My name is /ˈnɑnəmi/." -o hello.wav
cargo run --release -- say "<ja>こんにちは、ななみです。</ja>" --play
cargo run --release -- say '<sing tempo="90">twinkle@C4,C4 twinkle@G4,G4 little@A4,A4 star@G4:2</sing>' -o star.flac
cargo run --release -- midi song.mid --lyrics "la la la" -o song.wav
cargo run --release -- phonemes "I can't believe it's 2026!"
```

**Library:** `nanami::render_markup(src: &str, voice: &Voice, lang: Lang) -> Rendered` (`Rendered { audio: Vec<f32>, sample_rate, warnings }`), with `nanami::voice::Voice::nanami()` / `Voice::by_name("male")`, `nanami::ipa::Lang::{English, Japanese, Generic}`, and `nanami::audio_io::{write_audio, play}` for output.

**Gotchas:**
- The repo commits a 10 MB espeak-ng Python wheel (used by the evaluation tools), so clones are heavier than the code suggests.
- Run it with `cargo run --release`, as the README does.

## origami

**Repo:** https://github.com/KarpelesLab/origami · **Crate:** `origami` binary from workspace crates `chem`, `translate`, `geom`, `energy`, `dynamics`, `io`, `gpu`, `cli` (git only; the crates.io `origami` is an unrelated project) · **License:** MIT (CHARMM36 data files under their own academic terms) · **Status:** experimental research code

Deterministic, physics-only protein folding (no ML priors): mRNA → amino-acid sequence → all-atom chain (NeRF) → CHARMM36-derived force field with GB-OBC II implicit solvent and analytical SASA → L-BFGS minimization, BAOAB Langevin dynamics, replica-exchange MD, and co-translational growth. It folds chignolin to 1.43 Å Cα RMSD; larger proteins (Trp-cage, villin) only compact. Unlike most of the ecosystem it is **not** first-party pure Rust: it uses `nalgebra`, `rayon`, `image`, and `wgpu` (Metal GPU kernels), and was developed on Apple Silicon.

**Use it when:** you need a scriptable CLI to translate mRNA, build PDB structures from sequences, score/minimize structures, run small-protein MD/REMD, render PNG frames, or compute RMSD/Rg/DSSP/contact maps against a reference PDB.
**Don't use it when / limits:** you need predictive structure accuracy (use AlphaFold-class tools) or large proteins. It runs at roughly 60-500 ns/day on 134-520 atom systems. The crate names (`io`, `geom`, …) are generic and not designed for external embedding.

**Install / run:**
```sh
git clone https://github.com/KarpelesLab/origami && cd origami
cargo build --release --workspace          # binary: target/release/origami
origami translate examples/insulin.fasta
origami build --seq NLYIQWLKDGGPSSGRPPPS --output trp_cage.pdb
origami energy trp_cage.pdb
origami minimize trp_cage.pdb --output trp_min.pdb --algorithm lbfgs
origami dynamics trp_min.pdb --output-trajectory traj.pdb --steps 1500 --save-every 50 --dt 2.0 --shake-h
origami remd trp_min.pdb --output-trajectory remd.pdb --temperatures 300,310,321,333 --time-ps 50 --swap-interval-ps 0.5 --dt 2.0 --shake-h
origami cotranslate ...    # ribosome-style residue-by-residue growth (see README for flags)
origami render traj.pdb --output-dir frames/ --width 800 --height 600 --frame-dt-fs 100
origami analyze traj.pdb --reference native.pdb --output metrics.tsv --cluster-cutoff 1.5
```

**Gotchas:** Requires Rust 1.89+ (edition 2024). The SASA hydrophobic term is roughly 30× slower. Its strength is tuned with the `ORIGAMI_SASA_GAMMA_SCALE` env var. Outputs are multi-MODEL PDB trajectories and TSV metrics.

## x11anywhere

**Repo:** https://github.com/KarpelesLab/x11anywhere · **Crate:** `x11anywhere` (git only) · **License:** MIT OR Apache-2.0 (per README/Cargo.toml; no license files in repo) · **Status:** experimental (core protocol implemented; testing/hardening phase; last change Feb 2026)

A portable X11 display server in Rust. It accepts X11 clients over Unix/TCP sockets and translates to a native backend: Win32/GDI on Windows, Cocoa/CoreGraphics (via Swift FFI) on macOS, or passthrough to an existing X server on Linux/BSD (nested, like Xephyr). It implements core-protocol windows, GCs, drawing, images, pixmaps, events, atoms/properties, selections, fonts, colors, and cursors. A security layer offers window isolation, property/selection mediation, and grab limits.

**Use it when:** you need to show Linux X11 apps on macOS/Windows without XQuartz/VcXsrv, or you want a nested/isolated X display for untrusted clients.
**Don't use it when / limits:** you need a headless X server for automation (use Xvfb). There is no framebuffer/headless backend exposed; the internal `NullBackend` is used only when no platform backend is compiled. No RENDER/XFIXES/DAMAGE/COMPOSITE extensions, so modern toolkits (GTK/Qt) are unlikely to work. The Wayland backend is not implemented, although the CLI help lists it. The backend is chosen by platform at compile time; `-backend` does not switch it at runtime. Uses `x11rb`/`windows-sys`/`nix`, so it is not dependency-free.

**Install / run:**
```sh
cargo install --git https://github.com/KarpelesLab/x11anywhere
x11anywhere -display 1 -security default     # permissive | default | strict
x11anywhere -display 1 -tcp                  # also listen on TCP 6000+N
DISPLAY=:1 xclock
```
On Linux the X11 backend connects to the host display from `$DISPLAY` (default `:0`). On macOS the Swift package under `swift/` must be built (see `GETTING_STARTED.md`).

## cterm

**Repo:** https://github.com/KarpelesLab/cterm · **Crates:** `cterm` (GUI), `ctermd` (from `cterm-headless`), libraries `cterm-core`, `cterm-proto`, `cterm-client` (git only; version `0.0.20`) · **License:** MIT · **Status:** usable (released binaries for macOS/Windows/Linux, auto-update)

A terminal emulator tuned for running AI coding assistants such as Claude Code. It has native UIs (AppKit on macOS, GTK4 on Linux, Win32/Direct2D on Windows), tabs, tab templates with Docker/devcontainer/SSH launch (`sticky_tabs.toml`), and Sixel/iTerm2 images. All PTYs live in the **`ctermd` daemon**, so sessions survive UI restarts and upgrades. For agents, `ctermd` is a **headless terminal server with a gRPC API**: create a PTY session, write input or keys, and read the rendered screen as text.

**Use it when:** an agent needs to drive interactive TUI programs (REPLs, `vim`, installers, another AI CLI) and read what is on screen rather than raw bytes. Also use it to host long-lived shells that outlive the controller, or to parse ANSI/VT output into a grid in Rust (`cterm-core`).
**Don't use it when / limits:** you need a minimal dependency tree. The GTK UI needs system GTK4/libadwaita (C). `ctermd` uses tokio/tonic, and `cterm-core` uses `vte`, `image`, `tokio`, and `puressh`. There are no split panes yet.

**Install / run:**
```sh
# Prebuilt: https://github.com/KarpelesLab/cterm/releases (DMG, Windows installer/zip, Linux x86_64/arm64 tar.gz)
git clone https://github.com/KarpelesLab/cterm && cd cterm
cargo build --release -p cterm-headless      # target/release/ctermd (no GTK needed)
ctermd --foreground                          # Unix socket: $XDG_RUNTIME_DIR/cterm/ctermd.sock on Linux
ctermd --foreground --tcp --bind 127.0.0.1 --port 50051 --scrollback 10000
ctermd --print-socket-path
```

**gRPC API** (`crates/cterm-proto/proto/terminal.proto`, service `cterm.terminal.TerminalService`): `Handshake`, `CreateSession{cols, rows, shell, args, cwd, env}`, `ListSessions`, `WriteInput{session_id, data}`, `SendKey`, `GetScreenText{session_id, include_scrollback, start_row, end_row}`, `GetScreen`, `GetCursor`, `StreamOutput`, `StreamScreenUpdates`, `StreamEvents`, `Resize`, `SendSignal`, `DestroySession`, `Shutdown`. Any gRPC client works (e.g. `grpcurl` with the proto). `cterm-client` is the Rust client used by the GUI.

**Example (`cterm-core` as a VT parser, no PTY):**
```toml
[dependencies]
cterm-core = { git = "https://github.com/KarpelesLab/cterm" }
```
```rust
use cterm_core::screen::ScreenConfig;
use cterm_core::Terminal;

let mut term = Terminal::new(80, 24, ScreenConfig::default());
term.process(b"Hello, \x1b[1mWorld\x1b[0m!");
assert_eq!(term.screen().get_cell(0, 0).unwrap().c, 'H');
println!("{}", term.screen().grid().text()); // visible text, rows joined by '\n'
```
`Terminal::with_shell(...)` attaches a real PTY. `process_collecting` returns terminal replies (e.g. DSR) for you to route back.

**Gotchas:** Sessions persist in the daemon until they are destroyed, so clean up with `DestroySession`. The repo's CLAUDE.md tells agents working *in* the repo to commit/push directly to master. That applies only to contributors, not consumers.
