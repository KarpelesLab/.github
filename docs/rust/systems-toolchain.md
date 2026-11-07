# Systems & Toolchain

Low-level pure-Rust infrastructure: an OS kernel (kintane), three ways to run without libc (purestd, fullrust, rustlibc), a compiler back-end framework (latticefoundry), a standalone linker (qld) and assembler (rsasm), an HDL compiler (reticle), a byte-exact decompiler/recompiler (univdreams), a multi-machine emulator (rsemu), and a pseudocode-to-JS/Python transpiler (algoc). Most are independent projects; the exceptions are purestd and fullrust, which are built to work together, and the latticefoundry chain (lode → latticefoundry → rsasm/qld). The libc story in brief: **purestd** is a `std`-shaped library built on raw syscalls; **fullrust** is (a) a patched rustc whose *real* `std` uses raw syscalls, shipped as a Docker image or GitHub Action, and (b) the small runtime crate plus `cargo fullrust` that make purestd programs fully libc-free; **rustlibc** runs the other way and gives **C** programs a libc written in Rust. For loading shared libraries at run time, **dlsym** is a memory-safe `dlopen`/`LoadLibrary` wrapper with a C-header binding generator, and **dldota** lets static or libc-free (fullrust) binaries load system libraries by booting the system's own `ld.so`; dlsym uses dldota on the fullrust target. latticefoundry builds on z3rs/puremp and uses rsasm (behind `lf-as`) and qld (behind `lf-ld`); [lode](#lode) is a language whose compiler targets latticefoundry. qld and rsasm are also general-purpose drop-in replacements for GNU ld and GNU as.

## Quick pick

| Need | Use |
|------|-----|
| Build an unmodified Rust crate into a fully static, libc-free Linux x86-64 binary | [`fullrust`](#fullrust) (Docker image / GitHub Action) |
| A `std`-like API over raw syscalls for a `#![no_std]` program, zero deps | [`purestd`](#purestd) |
| A libc (`libc.a`/`.so`) written in Rust for C programs | [`rustlibc`](#rustlibc) |
| Load a shared library at run time with borrow-checked symbols, or generate run-time-loaded bindings from a C header | [`dlsym`](#dlsym) |
| Load system `.so` libraries from a static or libc-free Linux x86_64 binary | [`dldota`](#dldota) |
| A fast, GNU-ld-compatible linker, or linking in-process from Rust | [`qld`](#qld) |
| Assemble GNU as / NASM / vendor syntax for about 20 CPU families, from a CLI or a library | [`rsasm`](#rsasm) |
| Compile, simulate, synthesize or formally verify Verilog/VHDL, or program an FPGA | [`reticle`](#reticle) |
| Decompile an ELF/PE/Mach-O/NE binary to editable source that recompiles byte for byte | [`univdreams`](#univdreams) |
| Call exports of a 32-bit Windows DLL from Rust in a sandbox | [`univdreams`](#univdreams) (`univdreams::emulator`) |
| Decompile, edit and recompile a deployed EVM contract | [`univdreams`](#univdreams) (`ud evm`) |
| Emulate NES/Game Boy/SMS/Amiga/Mac/PC/RISC-V/AArch64 machines, or embed an emulator | [`rsemu`](#rsemu) |
| An LLVM-like compiler back end (SSA IR, optimizer, x86-64 codegen, linker) | [`latticefoundry`](#latticefoundry) |
| Experimental systems language compiler (proven-safe integer arithmetic, no runtime) on LatticeFoundry | [`lode`](#lode) |
| Write crypto/codec code once in a DSL and generate JavaScript and Python | [`algoc`](#algoc) |
| A Rust OS kernel that scales from MMU-less MCUs to SMP servers | [`kintane`](#kintane) |

## kintane

**Repo:** https://github.com/KarpelesLab/kintane · **Crate:** none (never published; not a cargo project) · **License:** MIT · **Status:** experimental. It boots in QEMU on several architectures, but it is not for production use.

A modular Rust OS kernel. One codebase covers x86_64, i686, aarch64, riscv32 (with and without atomics) and ARMv7-M (runs in 56 KiB of RAM). Architecture differences are expressed as traits and associated constants, never as `#[cfg]` in function bodies. The native userspace ABI is capability-based. A per-process **Linux personality** runs unmodified static Linux binaries: fork/execve, signals, futexes, pipes, TCP/UDP sockets and epoll all work. Drivers can run in the kernel or in their own isolated domain behind an IOMMU (VT-d); that choice is a configuration setting.

Note: the README still says "pre-implementation", but `docs/roadmap.md` is current. Phases 0–1 are done, SMP (8 CPUs), driver isolation and most of the Linux personality are built, and real-hardware work has begun.

**Use it when:** you are studying or contributing to kernel design, need an MIT-licensed kernel base for unusual or tiny hardware, or want a test bed for Linux-ABI binaries under a non-Linux kernel.
**Don't use it when / limits:** you need a production OS or a library. You cannot depend on it from Cargo. It needs a hash-pinned **nightly** toolchain (`toolchain.toml`) because it uses custom target JSONs (`targets/*.json`). Testing is QEMU-first and real-hardware coverage is thin.

**Build / run:** everything goes through `kbuild`, a purpose-built Rust tool in `kbuild/` (Kconfig-like config language in `config/*.kcfg`, presets in `config/presets/`):

```sh
kbuild toolchain --fetch                 # install the pinned toolchain
kbuild config --preset x86_64-qemu       # resolve a configuration
kbuild menuconfig                        # interactive TUI
kbuild build --preset x86_64-qemu        # build the kernel image
kbuild run                               # boot it under QEMU
kbuild test --host                       # host-side test suites
kbuild image --format elf|bin|uki|uimage # write the image in another format
kbuild randconfig-build --count 20 --seed 1
```

**Key features:**
- Presets include `x86_64-qemu[-smp]`, `x86_64-efi`, `x86_64-isolated[-smp]`, `x86_64-iommu`, `i686-bios`, `aarch64-virt[-smp]`, `aarch64-smmu`, `riscv32-virt`, `riscv32i-virt`, `armv7m-tiny`, `armv7m-mps2`.
- Own bootloaders (`kinboot-efi`, `kinboot-bios`), loadable modules, separate debug-symbol bundles (`kbuild symbols`, `kbuild symbolize`).
- Size budgets, randomized-config builds, fuzzing (`kbuild fuzz`) and soak testing (`kbuild soak`/`stress`).

**Gotchas:**
- Start with `docs/roadmap.md`, `docs/portability.md` and `docs/build-system.md`. The README's status line is out of date.
- Do not try `cargo build` at the repo root. Only `kbuild` itself is a cargo project.

## purestd

**Repo:** https://github.com/KarpelesLab/purestd · **Crate:** `purestd` (crates.io `0.0.3`) · **License:** MIT OR Apache-2.0 · **Status:** usable/experimental. The std surface it covers is complete and exercised in CI on 8 targets.

A `std` replacement written in pure Rust that **never calls libc**. Every OS operation is a direct syscall (or a WASI import on wasm). It is `#![no_std]`, built on `core` + `alloc` only, has **zero dependencies**, and brings its own allocator (mmap free-list), SipHash-1-3 `HashMap`/`HashSet`, threads (clone+futex; Mach threads on macOS), `net` with its own DNS resolver (no NSS), `process::Command`, `sync`, `fs`, `time`, `path`, and `os::fd`/`os::unix`. Module paths mirror `std`, so aliasing it as `std` lets ordinary `use std::...` code resolve.

**Use it when:** you want a `std`-shaped API in a `#![no_std]` binary without libc, or a libc-free static binary together with [fullrust](#fullrust).
**Don't use it when / limits:** you just want a static binary from an ordinary crate. Use the fullrust toolchain for that; it needs no code changes. purestd supplies only `std`. The process entry point (`_start`) and the `mem*`/unwind symbols come from libc/crt0 in a normal build, or from the `fullrust` runtime crate in a libc-free build. The WASI build has a reduced surface (no threads, fs, sockets or processes). x86_64-macOS has no threads.

**Add it:**
```toml
[dependencies]
purestd = "0.0.3"

[profile.dev]
panic = "abort"
[profile.release]
panic = "abort"
```

**Key features / cargo features:**
- `rt` (default): provides `#[panic_handler]`, `#[global_allocator]` and `rust_eh_personality`. Disable it (`default-features = false`) when a host runtime already provides them.
- CI targets: `x86_64`/`aarch64`/`riscv64gc`/`i686`/`arm` Linux, `aarch64`/`x86_64` Darwin, `wasm32-wasip1`.
- All arch/OS-specific code is in `src/arch/` (one file per target).

**Example:**
```rust
#![no_std]
#![no_main]

use purestd::prelude::*;

fn main() {
    println!("hello from purestd");
}

purestd::entry!(main);   // emits C `main`; crt0 or fullrust's _start calls it
```

**Gotchas:**
- `panic = "abort"` is expected.
- For a fully libc-free link, add `extern crate fullrust;` and build with `cargo fullrust --runtime build` (see [fullrust](#fullrust)).

## fullrust

**Repo:** https://github.com/KarpelesLab/fullrust · **Crates:** `fullrust` (crates.io `0.2.0`, runtime), `cargo-fullrust` (crates.io `0.1.2`, cargo subcommand); the main deliverable is the Docker image `ghcr.io/karpeleslab/fullrust` · **License:** MIT OR Apache-2.0 · **Status:** usable. Linux x86-64 only.

It compiles **unmodified Rust crates** into fully static, libc-free ELF binaries: no `PT_INTERP`, no `.dynamic`, no `NEEDED`. It is a patched rustc with a built-in `x86_64-unknown-linux-fullrust` target, whose real `std` has a raw-syscall platform backend (the Go "no-cgo" model). Threads, TLS, fs, `process::Command`, TCP/UDP with its own DNS resolver, unwinding panics and symbolized backtraces (pure-Rust unwinder) all work. The image auto-patches `getrandom` and `socket2`, so `rand`, `uuid`, `mio` and tokio-style stacks build.

**Use it when:** you want a single static binary that depends only on the kernel, for release artifacts, scratch containers or minimal systems, with no source changes.
**Don't use it when / limits:** you need FFI into C libraries (it fails at link time, by design), dynamic linking, or a target other than Linux x86-64. `cfg(unix)` is **false** on this target: `std::os::unix` is absent, and `std::os::fd` plus `std::os::fullrust` replace it. The `libc`/`nix` crates are unavailable.

**Run it (GitHub Action):**
```yaml
- uses: actions/checkout@v6
- uses: KarpelesLab/fullrust@master
  with:
    args: --release --bin myapp          # inputs: command, args, working-directory, image, no-ecosystem
# output: target/x86_64-unknown-linux-fullrust/release/myapp
```

**Run it (Docker, locally or as a `container:` job):**
```sh
docker run --rm -v "$PWD":/src ghcr.io/karpeleslab/fullrust:latest build --release
# test / run / clippy work the same; tags :1.88 … :1.98, :latest, immutable :<minor>-<commit>
```
The action defaults to `:1.88`. Set `image: ghcr.io/karpeleslab/fullrust:1.98` (or pin `:<minor>-<commit>` / a digest) to get a newer Rust or a reproducible build.

**Raw syscalls / exec on this target** (replaces `libc`/`nix`, guard with `#[cfg(target_os = "fullrust")]`):
```rust
use std::os::fullrust::process::CommandExt;
use std::os::fullrust::syscall::{self, nr};
use std::process::Command;

let tid = unsafe { syscall::syscall0(nr::GETTID) }?;
let err = Command::new("/bin/sh").arg0("sh").args(["-c", "echo hi"]).exec();
```

**The crates (purestd path, no Docker):**
- `fullrust` provides `_start`, the `mem*`/`strlen` intrinsics, `getauxval` (aarch64) and an `_Unwind_Resume` stub for [purestd](#purestd) programs (`extern crate fullrust;` + `purestd::entry!(main)`).
- `cargo install cargo-fullrust`, then `cargo fullrust build --release`. Zero-touch mode builds a cached sysroot whose `std` is purestd; needs nightly + `rust-src`. `cargo fullrust --runtime build` is for explicit `#![no_std]` + purestd + fullrust crates (`--stable` uses precompiled core/alloc).

## rustlibc

**Repo:** https://github.com/KarpelesLab/rustlibc · **Crate:** `rustlibc` (git only, `publish = false`) · **License:** MIT · **Status:** early scaffold. The pure-computation parts are real; I/O, math and threads are partly stubbed.

A C standard library written as one `#![no_std]` Rust crate. It exports the canonical C symbols and provides its own `_start`, producing `librustlibc.a` / `librustlibc.so` that C/C++ programs link **instead of glibc/musl**. It talks to Linux via raw syscalls and ships its own headers in `include/`.

**Use it when:** you are experimenting with building C programs against a Rust libc on x86_64/aarch64 Linux.
**Don't use it when / limits:** you are a Rust program (use [purestd](#purestd)/[fullrust](#fullrust)), or you need a complete libc. `scanf`, `fopen`, `getenv`, signal handlers, `pthread_create`, locale and math transcendentals are **stubs**. stdio is unbuffered and `malloc` does one mmap per allocation. It needs **nightly** (`c_variadic` for `printf`).

**Build / use:**
```sh
make lib                                  # release librustlibc.a + librustlibc.so
make run                                  # build and run examples/hello.c, sysprobe.c
make test                                 # cargo test --lib
make hello ARCH=aarch64-unknown-linux-gnu # cross-build
# C programs link with: -nostdinc -Iinclude -nostdlib -nostartfiles -static ... librustlibc.a
```

**Key features / cargo features:**
- `crt` (default): provides `_start`/`__libc_start_main`. Disable it when another runtime owns the entry point.
- crate-type `staticlib` + `cdylib` + `rlib`. C symbols are exported with `#[cfg_attr(not(test), unsafe(no_mangle))]`.
- Real today: `string.h`, `ctype.h`, the `printf` family (floats best-effort), `strtol`/`qsort`/`bsearch`, `malloc` family, `unistd`/`fcntl`/`stat`/`mman`/`dirent`/`poll`/`wait`, `setjmp`/`longjmp`.

## dlsym

**Repo:** https://github.com/KarpelesLab/dlsym · **Crate:** `dlsym` (crates.io `0.1.0`; `dlsym-gen` binary behind the `codegen` feature) · **License:** MIT · **Status:** usable; already used by oxinfer, puressh (security keys, PKCS#11) and OxideAV's hardware bridges (`oxideav-mjpeg`, `oxideav-vaapi`). MSRV 1.95

Run-time dynamic library loading (`dlopen`/`dlsym`, `LoadLibrary`/`GetProcAddress`) with the safety guarantee that matters most: a symbol borrows its library, so the borrow checker rejects unloading a library while a symbol from it is alive. Symbols don't deref to raw pointers (functions are called with `call((args,))`, variadics with `with(|f| f(..))`), only `unsafe extern` function pointers and raw data pointers can be requested, and a null symbol (`NullSymbol`) is told apart from a missing one (`SymbolNotFound`). `ArcSymbol` (from `Library::get_arc`) and `Library::leak` give owned and `'static` symbols without `transmute`. The `library!` macro binds a whole API in `unsafe extern` block syntax, resolving every non-`#[optional]` symbol at construction. On the libc-free [`fullrust`](#fullrust) target it loads through [`dldota`](#dldota) with the same code.

**Use it when:** a crate must use a system library only when it is present (GPU drivers, codecs, PKCS#11 modules, libclang) without link-time dependencies or `-sys` crates, or you want generated bindings for a C header that load at run time.
**Don't use it when / limits:** you can link the library normally. Loading foreign code is inherently `unsafe` (opening runs its initialisers, and signatures are your assertion). On fullrust, `AddressInfo` and `Library::path` are unavailable, and callbacks passed to foreign code must be wrapped with `dlsym::os::fullrust::context()?.callback(..)`.

**Add it:**
```toml
[dependencies]
dlsym = "0.1"
```
```sh
cargo install dlsym --features codegen      # dlsym-gen; prebuilt binaries on the GitHub releases page
dlsym-gen /usr/include/zlib.h --crate zlib-dl -l libz.so.1 -l libz.1.dylib -l zlib1.dll
```

**Example (README; verified to compile and run on Linux):**
```rust
use dlsym::Library;

// SAFETY: libm's initialisers are trusted.
let libm = unsafe { Library::open("libm.so.6")? };
// SAFETY: `cos` has this signature.
let cos = unsafe { libm.get::<unsafe extern "C" fn(f64) -> f64>(c"cos")? };
assert_eq!(unsafe { cos.call((0.0,)) }, 1.0);
// drop(libm) here would not compile: `cos` borrows it.
```
Also `Library::open_first([..])`, `Library::this()`, `dlsym::library_filename("z")`, `AddressInfo::of(ptr)`, and `os::unix::OpenOptions` / `os::windows::OpenOptions` for loader flags.

**Gotchas:**
- `dlsym-gen` itself needs a libclang at run time (`--libclang`, `$LIBCLANG_PATH`, `llvm-config`, or the usual install paths). The generated bindings do not.
- `dlsym-gen` output checks struct layouts at compile time against the C compiler's; bindings generated for one platform fail to compile on another (`--target` generates for a different one).
- C++ headers (`--cpp`) are bound only where the ABI is C-like; virtual functions are not bound.

## dldota

**Repo:** https://github.com/KarpelesLab/dldota · **Crate:** `dldota` (crates.io `0.1.1`; also a static C library `libdldota.a` with `include/dldota.h`) · **License:** MIT · **Status:** working on Linux x86_64 (glibc and musl systems). aarch64 is not implemented: the README's "Supported" line lists it, but the aarch64 boot and trampoline code is `unreachable!()`. MSRV 1.95

Lets a **statically linked** program (including a libc-free [`fullrust`](#fullrust) binary) load the host system's shared libraries. It maps the system's own dynamic loader (`ld-linux-*.so`, `ld-musl-*.so`) into the process, boots it as the kernel would, and exposes its `dlopen`/`dlsym`. Calls into loaded code go through trampolines that switch the thread pointer, so the host runtime and the system libc coexist; a per-context heap serves every foreign allocation, and a context can be unloaded. It has no dependencies. [`dlsym`](#dlsym) uses it automatically on the fullrust target, so most code should go through dlsym rather than call dldota directly.

**Use it when:** a fully static or libc-free Linux binary still needs an optional system library (GPU driver, audio, codecs) at run time.
**Don't use it when / limits:** you are on aarch64 or any non-Linux OS (it refuses to compile off Linux), or your program is dynamically linked anyway (use `dlsym`). `docs/design.md` lists the known hazards of running two libcs in one process.

**Add it:**
```toml
[dependencies]
dldota = "0.1"
```
C consumers: `cargo build --release --target x86_64-unknown-linux-musl` (or the `-gnu` target for a static glibc program) and link `libdldota.a`.

**Example (verified to compile and run on x86_64 Linux):**
```rust
use dldota::{Context, OpenFlags};

let ctx = Context::new()?;                              // boots the system's ld.so
let libz = ctx.open(c"libz.so.1", OpenFlags::NOW)?;
let version = unsafe {
    libz.symbol::<unsafe extern "C" fn() -> *const core::ffi::c_char>(c"zlibVersion")?
};
let v = unsafe { std::ffi::CStr::from_ptr((*version)()) }; // Symbol derefs to the trampoline
```
`Context::builder().interpreter(path).env(vars).build()` picks the loader and environment, `ctx.callback(f)` wraps a host function for foreign code, `ctx.heap_usage()` reports the foreign heap, and `ctx.unload()` tears the context down. Set `DLDOTA_DEBUG=1` to trace to stderr.

## latticefoundry

**Repo:** https://github.com/KarpelesLab/latticefoundry · **Crate:** `latticefoundry` (crates.io `0.0.2`) · **License:** MIT · **Status:** experimental but substantial. Roadmap phases 0–9 are complete and most of phase 10; `lf build` produces running executables, and `lf-cc` builds gzip, bzip2, Lua and SQLite byte-identical to gcc.

A clean-room compiler back-end framework in the role of LLVM. It provides a typed SSA IR (block arguments, poison/freeze, opaque pointers), a verifier and refinement checker backed by the [z3rs](https://github.com/KarpelesLab/z3rs) SMT solver, an optimizer (mem2reg, SCCP, simplify_cfg, DCE, e-graph equality saturation, a z3rs superoptimizer, LICM, inlining, `-O0..-O3`, LTO), x86-64 and AArch64 codegen, DWARF output, and an in-process JIT. Direct dependencies are only sibling crates: `z3rs`, `puremp`, [rsasm](#rsasm) (behind `lf-as`) and [qld](#qld) (behind `lf-ld` for non-`.lfo` inputs). `unsafe` is confined to the JIT.

**Use it when:** you are building a language front end and want a pure-Rust back end (this is what [lode](#lode) does), you need to compile C without gcc/clang (`lf-cc`), or you are researching verified optimization.
**Don't use it when / limits:** you need production codegen or broad targets. x86-64 is the solid target (runs natively, System V and Microsoft x64 ABIs); AArch64 is validated against `llvm-mc` and an interpreter. `lf-cc` has no `long double` (it is `double`), `_Float128` or `_Complex` arithmetic. The public API is unstable (0.0.x).

**Add it / install:**
```toml
[dependencies]
latticefoundry = "0.0.2"
```
```sh
cargo install latticefoundry   # binaries: lf, lf-opt, lf-ld, lf-as, lf-dis
```

**CLI:**
```sh
lf build foo.lf -o foo -O2 -g        # .lf (text) / .lfb (binary) IR -> static ELF64, no libc, no system linker; --lto
lf build --target <triple> -c foo.lf # relocatable ELF, PE/COFF or Mach-O object; --oformat binary|ihex for firmware
lf build --shared -o libfoo.so foo.lf  # x86-64 PIC shared library linked by qld; --pie, -c --pic
lf-opt -O2 foo.lf -o foo.lfb         # verify/optimize/re-encode IR; -p mem2reg,sccp,... ; --emit lf|lfb
lf-as / lf-dis                       # GNU-syntax assembler (x86-64, AArch64, RISC-V via rsasm) / disassembler
lf-ld -o out -e main a.lfo b.lfo     # own static linker for .lfo; any other input takes a full GNU ld command line (qld)
```

`lf-dis` disassembles ELF, `.lfo`, COFF/PE, Mach-O, wasm modules and flat binaries (`--raw --arch`) for x86-64 (AT&T or `--syntax intel`), AArch64, RISC-V RV64GC (with arch detection), Thumb-2, AVR and wasm32; output matches `llvm-objdump` on LF's own objects.

**C front end (`lf-cc`):** a separate crate in the same repo, not published and not built from the root (`cd lf-cc && cargo build --release`). It compiles against the real glibc `/usr/include` and links through qld. Coverage includes GNU attributes, statement expressions, computed goto, `__builtin_*`, the System V struct ABI (objects mix with gcc-compiled code), C11 atomics (`_Atomic`, `<stdatomic.h>`, `__atomic_*`/`__sync_*`), GCC vector types, thread-local storage, `__int128` with gcc's ABI, visibility/weak symbols, and `-fPIC`/`-shared`/`-pie`.
```sh
lf-cc -O2 -g -o hello hello.c     # -S / --emit-lf dumps the lowered .lf IR
```

**Gotchas:** it is a single package, not a workspace. Rust 1.89+ (edition 2024). Design is documented in `docs/design-tenets.md` and `docs/ir-design.md`.

## lode

**Repo:** https://github.com/KarpelesLab/lode · **Crate:** `lode` (git only. The crates.io crate named `lode` is an unrelated Ruby package manager) · **License:** MIT · **Status:** early development. The design docs are far ahead of the compiler, which handles a small subset of the language and has one target.

A new systems programming language, pitched as "the best of Rust, Zig and Go, each pushed to its limit". This repo is its compiler, written in Rust as one package that provides both the `lode` library and the `lode` binary. The aims are memory safety and data-race freedom without lifetime annotations, no undefined behaviour and no hidden crash paths in safe code, no runtime, Zig-style `comptime`, and opt-in green threads and structured concurrency. The back end is [`latticefoundry`](#latticefoundry), taken from crates.io (`0.0.2`). The compiler itself is `#![forbid(unsafe_code)]`. A hello world compiles to a 1,049-byte static ELF that makes exactly two syscalls (`write`, `exit`).

**Use it when:** you are experimenting with or contributing to the language, or you want a working example of a front end that emits LatticeFoundry IR and links with LatticeFoundry's linker.
**Don't use it when / limits:**
- Do not use it for real programs. The only output is static x86-64 Linux executables.
- The working subset is functions, `let`/`var`, `if`/`while`/`loop`, `break`/`continue`, `const`, integer types, `bool`, `str`, packages (`import "std/..."`, `pub`), `unsafe` blocks, raw pointers and the `syscall` intrinsic.
- None of these are implemented yet: structs, sum types, generics/`comptime`, `throws`, allocators, concurrency, `lode fmt`.
- The standard library is only `std/io` (`print`, `eprint`) and `std/os`.

**Install / run:**
```sh
git clone https://github.com/KarpelesLab/lode && cd lode
cargo build --release                      # MSRV 1.89, edition 2024
target/release/lode run hello.lode           # build to a temp file, run it, exit with its status
target/release/lode build hello.lode -O2 -o hello   # -O0 (default) .. -O3
target/release/lode build hello.lode --emit=ir      # print LatticeFoundry IR instead
target/release/lode check hello.lode         # type/proof check only
```

```
package main

import "std/io"

fn main() {
	io.print("hello world\n")
}
```
`fn main() -> u8` makes the return value the exit status.

**Key features:**
- **Proven arithmetic.** Plain `+ - * / % <<`, negation and narrowing conversions such as `u8(x)` compile only when the checker can prove they cannot overflow, divide by zero or truncate. The checker tracks value ranges and relations between variables, and narrows them on conditions and early returns. When it cannot prove safety, the build fails with a diagnostic: `cannot prove that this \`u32\` value fits in \`u8\``. Use `+% -% *% <<%` to wrap and `+| -|` to saturate.
  ```
  fn clamp_to_u8(x: i32) -> u8 {
  	if x < 0 { return 0 }
  	if x > 255 { return 255 }
  	return u8(x)          // accepted: 0 <= x <= 255 is proven here
  }
  ```
- Library API (`lode::check`, `lode::compile_ir`, `lode::ir_text`, `lode::build_executable`) operates on a `lode::source::SourceMap` and a root `FileId`. `build_executable` returns the ELF image as `Vec<u8>`.
- The design specs live in `docs/`: `concept.md`, `safety.md`, `memory.md`, `comptime.md`, `concurrency.md`, `errors.md`, `packages.md`, `syntax.md`, `types.md` and `backend.md`. The planned targets run from 8-bit MCUs to wasm.

**Gotchas:**
- The compiler finds the standard library through `$LODE_STD`, or falls back to the `std/` directory of the source tree it was built from (`CARGO_MANIFEST_DIR`). If you move the binary, or `cargo install` it and later delete the source checkout, set `LODE_STD`.
- Test programs in `tests/programs/*.lode` state their expected exit status or errors in their first comment lines. They are the best reference for what currently compiles.
- Source files use the `.lode` extension and tab indentation (Go style).

## qld

**Repo:** https://github.com/KarpelesLab/qld · **Crate:** `qld` (crates.io `0.1.1`) · **License:** MIT · **Status:** alpha, but it already links real software. Used as the system linker, it builds and passes the test suites of coreutils, curl, OpenSSL, Python, LLVM/clang/lld and rustc (on macOS: zlib, Lua, SQLite, {fmt}), and it links a Linux kernel that boots. Output is byte-identical whatever the thread count.

A fast, parallel, deterministic linker in pure Rust and a **drop-in replacement for GNU ld/gold/lld/mold**: it accepts their argv, linker scripts and the LTO plugin API. Formats: ELF, PE/COFF (MinGW), Mach-O (`ld64` flavor, including fat), AmigaOS Hunk, and raw `binary`/`ihex`/`srec`. ELF architectures: x86-64, i386, x32, AArch64, 32-bit Arm, RISC-V 64/32, PowerPC64 LE/BE (BE static only), LoongArch64, s390x and m68k, each compared in CI with GNU ld, lld or vlink and run natively or under qemu. Features include static/dynamic/PIE/static-PIE, `-shared`, `-r`/`--emit-relocs`, symbol versioning, RELRO, `DT_RELR`, `--gc-sections`, `--icf`, `--gdb-index`, compressed debug, `--build-id`, `MEMORY`/`PHDRS` scripts, and LTO through the compiler's plugin or `libLTO` on macOS. It is roughly 5–8x faster than GNU ld on large links. It is also a library with no global state, no `process::exit`, and in-memory inputs and outputs.

**Use it when:** you want a faster system linker for C/C++/Rust on x86-64 Linux, or you need to link objects from inside a Rust tool.
**Don't use it when / limits:** you need a linker with years of field use (it has had little use outside its own CI). The library API is unstable before 1.0; only `args`, `diag`, `error` and `target` plus the root re-exports are documented. MSRV 1.89. Dependencies: `rayon`, `memmap2`, `hashbrown`, `foldhash`.

**Install / run:**
```sh
cargo install qld
gcc -B/usr/libexec/qld hello.c -o hello        # dir containing an `ld` that is qld
clang -fuse-ld=qld hello.c -o hello            # finds ld.qld in PATH
clang --ld-path=/opt/qld/bin/ld hello.c -o hello
RUSTFLAGS="-C linker=gcc -C linker-features=-lld -C link-arg=-B/opt/qld/bin" cargo build --release
```
`argv[0]` selects the flavor (`ld`/`ld.qld` = GNU, `ld64.qld` = Apple ld64). Links run in a forked child by default; pass `--no-fork` to disable. The `plugin` feature (default) enables LTO via `dlopen` of the compiler's plugin, which is the only FFI in qld.

**Example (library):**
```rust
use qld::{InputAttrs, InputKind, LinkOptions, OutputKind};
use qld::diag::Collect;

let mut options = LinkOptions::new();              // hermetic: no env, no stdout, no exit
options.kind = OutputKind::StaticExecutable;
options.push_input(InputKind::bytes("main.o", object), InputAttrs::default());
let diagnostics = Collect::new();
let image: Vec<u8> = qld::link_to_memory(&options, &diagnostics)?;
```
To parse a GNU command line, use `qld::parse_gnu(&args)?` (returns `ParseOutcome::Link(options)`) followed by `qld::link(&options, &sink)`. The `examples/` directory has `link_argv`, `in_memory`, `custom_sink`, `rayon_pool`, `cancel` and `link_map`.

## rsasm

**Repo:** https://github.com/KarpelesLab/rsasm · **Crate:** `rsasm` (crates.io `0.1.3`) · **License:** MIT · **Status:** early but broad and heavily verified. Every encoding is checked byte for byte against GNU as, llvm-mc, vasm, ca65 or AS.

A multi-syntax, multi-architecture assembler with **zero dependencies**, usable as a CLI and as a library. Targets: x86-64/i386/i8086 (through AVX-512, AVX10.2, AMX), AArch64 (NEON, SVE/SVE2), ARM/Thumb, RISC-V RV32/64 IMAFDC, PowerPC 32/64 (AltiVec/VSX), MIPS, SPARC, m68k/ColdFire, SuperH, Renesas RX/RL78/V850/RH850, NEC 78K0, AVR, MSP430, Z80, 6502, 8080 and 8051. Dialects: GNU as (AT&T and Intel), NASM (including its preprocessor), Motorola, Renesas CC-RL/CC-RH/CC-RX, and 8-bit vendor syntaxes. Outputs: ELF32/64 relocatables, PE/COFF (x86-64/i386/ARM64), Mach-O (x86-64/arm64), flat binary and Intel HEX. It supports DWARF 2–5 (`.loc`, `.cfi_*`, `-g`), TLS relocations, macros and relaxation. `.arch` switches the target mid-file, so one file can mix CPUs.

**Use it when:** you need `as`/`nasm` replacement or cross-assembly without binutils, or you want to assemble to bytes in-process (JITs, test harnesses, retro toolchains).
**Don't use it when / limits:** you need a disassembler (not included) or a linker (use [qld](#qld) or the system `ld`). Library API surface is deliberately small.

**Install / run:**
```sh
cargo install rsasm
rsasm -o hello.o hello.s && ld -o hello hello.o
rsasm -a aarch64 -o x.o x.s                  # -a also takes triples: x86_64-apple-macos (Mach-O), x86_64-pc-windows-msvc (COFF)
rsasm -d nasm -f win64 -o x.obj x.asm        # -f elf|elf32|elf64|coff|win64|win32|macho|bin|ihex
rsasm -a z80 -f bin --base 0x100 --hex x.s   # print bytes as hex
rsasm --list-arch
```
Architectures are cargo features, all on by default: `x86`, `aarch64`, `arm`, `riscv`, `powerpc`, `mips`, `sparc`, `retro` (Z80/6502/8080/8051), `m68k`, `superh`, `rx`, `rl78`, `v850`, `k78`, `avr`, `msp430`. As a library, `rsasm = { version = "0.1", default-features = false, features = ["x86"] }` keeps the build slim.

**Example (library):**
```rust
use rsasm::assembler::{Assembler, Options};
use rsasm::lexer::Dialect;
use rsasm::section::SectionId;
use rsasm::{arch, output};

let options = Options::new().with_dialect(Dialect::Gas);
let mut asm = Assembler::new(arch::lookup("x86-64").unwrap(), options);
asm.assemble_str("example.s", "movq %rbx, %rax\nret\n");
if !asm.finish() || asm.diags().has_errors() {
    eprint!("{}", asm.diags().render(asm.source_map(), false));
} else {
    let text: &[u8] = asm.section_bytes(SectionId(0));
    let object = output::elf::build(&asm).unwrap();   // ELF relocatable bytes
}
```

**Gotchas:** `Options` is `#[non_exhaustive]`, so build it with `Options::new().with_*()`. `with_relocatable(false)` gives a flat image at address 0.

## reticle

**Repo:** https://github.com/KarpelesLab/reticle · **Crate:** `reticle` (crates.io `0.0.7`) · **License:** MIT · **Status:** early. The full pipeline runs end to end, and designs have been run on real FPGAs (Basys 3/Artix-7, Cynthion/ECP5, Tang Primer 20K/Gowin, and the programmable logic of a Zynq UltraScale+ ZCU104).

A from-scratch **Verilog-2005/SystemVerilog and VHDL-2008 compiler**. The default build has **zero dependencies** and a sans-I/O library core; `unsafe` is denied except in the C ABI and wasm modules. Both languages lower to one IR, which feeds a 4-state event-driven simulator (VCD/FST, SVA/PSL subset, coverage), synthesis (FF/memory/FSM inference, AIG optimizer, LUT and standard-cell mapping), formal verification (in-crate CDCL SAT: BMC, k-induction, equivalence), static timing and CDC analysis, and FPGA/ASIC back ends. It also has an IP manifest/registry system, formatters, an LSP server, an HTML schematic viewer, a C ABI and a WASM build. It can write Xilinx 7-series (Project X-Ray), Zynq UltraScale+ (Project U-Ray; a `.bin` for Linux's FPGA manager, with the clock network still borrowed from the board's PYNQ image via `--borrow-clock`), ECP5 (Trellis) and Gowin (Apicula) bitstreams, and program boards over JTAG with no vendor tools. 7-series mapping covers DSP48E1, PLLE2/MMCME2, block RAM, XADC and LVDS. IP dependencies resolve into a lock file (`reticle build --locked` fails instead of rewriting it).

**Use it when:** you want to lint, simulate, synthesize or formally check HDL from Rust or the command line, write HDL testbenches as Rust `#[test]`s, or run an open FPGA flow without vendor tools.
**Don't use it when / limits:** you need full-coverage vendor-grade P&R or timing closure. Parts of VHDL (VUnit/UVVM/OSVVM testbenches) are unsupported. Bitstream generation is proven only on small designs (for example, a carry chain does not yet route on 7-series). MSRV 1.89.

**Install / run:**
```sh
cargo install reticle                     # features cli (default) = every stage
reticle check counter.v counter.vhd
reticle sim --vcd waves.vcd testbench.v counter.v
reticle synth --report --lut 4 --output netlist.rtl counter.vhd
reticle verify --depth 20 --trace cex.vcd design.rtl
reticle emit --format verilog netlist.rtl       # also VHDL, Yosys JSON, BLIF, EDIF
reticle fpga --device ice40-hx1k-tq144 --constraints pins.rcf blinky.v   # netlist for nextpnr
reticle fpga --device xc7a35t-cpg236 --constraints b.rcf --bitstream out.bit top.v
reticle fetch prjuray-db prjuray-db-2020   # once, for Zynq UltraScale+
reticle fpga --device xczu7ev-ffvc1156 --constraints c.rcf --bitstream counter.bit --borrow-clock base.bit top.v
reticle program out.bit                    # needs --features program
reticle fmt --write counter.v ; reticle lsp
```

**Add it (library):**
```toml
[dependencies]
reticle = { version = "0.0.7", default-features = false, features = ["verilog", "sim"] }
```

**Key features / cargo features:** `verilog`, `vhdl`, `sim`, `synth` (default together with `cli`); `formal`, `fpga`, `asic`, `timing`, `ip`, `lsp`, `cache`, `viewer`, `ffi`, `wasm`. Only `program` (adds `rawusb`) and `apicula` (adds `compcol`) pull in a dependency, and both are off by default.

**Example:**
```rust
use reticle::diag::Diagnostics;
use reticle::source::SourceMap;
use reticle::verilog::ast::ItemKind;
use reticle::verilog::{Dialect, NoIncludes, parse_source};

let mut map = SourceMap::new();
let id = map.add("t.sv", "module m(input logic a, output logic y);\n  assign y = ~a;\nendmodule").unwrap();
let mut diags = Diagnostics::new();
let file = parse_source(&mut map, id, Dialect::SystemVerilog, &mut NoIncludes, &mut diags);
assert!(diags.is_empty());
let ItemKind::Module(m) = &file.items[0].kind else { panic!() };
assert_eq!(m.name.name, "m");
```

**Gotchas:** the library never touches the filesystem, so feed sources through `SourceMap`. The `ip/` HDL library and the `examples/` directory (RISC-V SoC, 6502 computer, Apple II, NES) are in the repo but not in the published crate.

## univdreams

**Repo:** https://github.com/KarpelesLab/univdreams · **Crate:** `univdreams` (crates.io `0.5.0`; binary `ud` behind the default `cli` feature) · **License:** MIT · **Status:** usable for its core promise (byte-identical round-trip). High-level decompilation is partial. MSRV 1.89 (edition 2024)

A compiler **and** decompiler suite. It decompiles a binary to `.ud` source whose directives pin the compiler's choices (encodings, layout, padding), so that recompiling reproduces the input **byte for byte**. Formats: ELF64 (x86-64, x86, aarch64; any other ELF machine round-trips with its code kept opaque), PE/COFF (MSVC/MinGW), thin Mach-O (x86-64, arm64), 16-bit Windows NE, raw 6502, and EVM bytecode (`ud evm <address|deploy tx hash>` fetches, decompiles, and recompiles a contract). It lifts to structured statements (if/switch/goto, calls with argument folding), reads DWARF signatures, and discovers functions via symtab, eh_frame, exports and signatures. It also includes a sandboxed **i386 Win32 emulator** and a Linux user-mode personality for static ELF guests (i386 wired today), with an optional KVM backend. Runs in the browser: https://karpeleslab.github.io/univdreams/. Dependencies include `iced-x86` and `gimli`, so it is not dependency-free.

**Use it when:** you are reverse engineering or binary patching (edit a function, rebuild an otherwise identical binary), inspecting what a compiler emitted, editing a deployed EVM contract, or safely calling into a 32-bit Windows DLL (codecs, malware triage) from Rust.
**Don't use it when / limits:** you need readable high-level C. Loops come out as `goto`, and there is no type or struct recovery without DWARF. It does not handle fat Mach-O, 32-bit Mach-O, 32-bit ARM, or packed/vectorized/LTO binaries. Editing a Mach-O invalidates its code signature.

**Install / run:**
```sh
cargo install univdreams                     # installs `ud`
ud decompile path/to/binary > prog.ud        # auto-detects ELF / PE / Mach-O / NE / 6502
ud compile prog.ud -o rebuilt                # dispatches on @module.format
ud roundtrip --through-source path/to/binary --out /tmp/rebuilt
ud verify prog.ud                            # check @asm lines against pinned bytes
ud analyze codec.dll                         # run DllMain in the i386 sandbox, report Win32 calls/coverage
ud evm 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2   # EVM contract by address
```

**Cargo features:** `cli` (default; the `ud` binary), `script` (`ud script`, drive the sandbox from JavaScript via kataan), `wasm` (browser bindings), `trace`/`trace-exec` (emulator instrumentation), `kvm` (Linux guests under KVM, Linux hosts only), `fstool` (mount ext4/NTFS/FAT images into the emulator via fstool). Library users who don't need the CLI should set `default-features = false`.

**Modules:** the former `ud-*` crates are now modules of the one crate: `format` (byte-exact ELF/PE/Mach-O/NE/raw read+write), `arch::{x86, aarch64, mos6502, bpf, evm, codec}`, `ir`, `ast`, `analysis`, `signatures`, `debug` (DWARF), `translate` (`.ud` <-> binary), `emulator`, `cli`, `wasm`.

**Example (`univdreams::emulator`, call a DLL like `dlopen`):**
```rust
use univdreams::emulator::Guest;

let bytes = std::fs::read("codec.dll")?;
let mut guest = Guest::load("codec.dll", &bytes)?;          // also runs DllMain
let version: u32 = guest.call("GetCodecVersion", ())?;      // stdcall default
let ptr = guest.alloc(&vec![0u8; 4096])?;
let rc: i32 = guest.call("Decompress", (ptr, 4096u32))?;
let out = guest.read(ptr, 4096)?;
```
Variants: `Guest::load_raw` (skip DllMain), `load_into`/`load_raw_into` (bring your own `Sandbox` with an instruction budget or VFS `Context`), `alloc_cstr`, `write`, `sandbox_mut()`. The guest has no host filesystem, network or registry access unless you attach it.

**Gotchas:**
- The 16 `ud-*` crates were folded into `univdreams` on 2026-10-05 and are frozen at 0.3.0 on crates.io. Code using `ud_emulator::Guest` or `cargo install ud-cli` should move to `univdreams::emulator::Guest` and `cargo install univdreams`.

## rsemu

**Repo:** https://github.com/KarpelesLab/rsemu · **Crate:** `rsemu` (crates.io `0.0.8`) · **License:** MIT · **Status:** early but it boots software it did not write: Linux on RISC-V/x86/AArch64 (including SMP), AmigaOS/Workbench on 8 Amiga models, FreeDOS on `pc-at`, and Macintosh Plus/Classic. NES passes AccuracyCoin 141/141. These OS boots are not run in CI because guest images are not shipped.

A multi-platform emulator built on a generic framework: address spaces, clock-domain trees, buses, devices, a translation IR and JITs. Machines are described in **`.machine` files** rather than compiled in; 49 ship in `machines/`, including `q35-uefi`, the Armv8-M `stm32u575`/`stm32u585` (Cortex-M33, tested against ST's STM32U5 HAL) and `alphard-navi` (the Renesas R-Car Gen1 navigation computer of a Toyota Alphard). CPU cores: 6502/65C02, Z80, SM83, 8086/80386/x86-64, 68000–68040 + 68881, MIPS R3000, SH-4A, RISC-V RV64GC, ARMv5TE, ARMv7-A (Cortex-A9 with VMSAv7 MMU), ARMv7-M, Armv8-M (Cortex-M33), AArch64. It is conformance-measured against SingleStepTests, riscv-tests and others. The core is `no_std + alloc`, **zero dependencies by default**, deterministic (bit-reproducible, snapshot/rewind), with `unsafe` confined to a handful of modules. Execution engines are `interp`, `jit` (portable IR), `jit-host` (x86-64 native code) and `jit-wasm`. It also supports KVM acceleration via raw ioctls, a GDB stub, a C ABI, and `wasm32` (browser) builds.

**Use it when:** you want to run or debug retro or modern guests, build a board from a description file, or embed a deterministic emulator in a test harness (Rust, C ABI or wasm).
**Don't use it when / limits:** you need QEMU-level device breadth or speed today. The default build includes **no machines**: each machine and device is a cargo feature. No guest images (ROMs, BIOS, kernels) are shipped. MSRV 1.91, stable only.

**Install / run:**
```sh
cargo install rsemu --features machine-nes,machine-apple1,machine-gameboy
rsemu machines                                  # what this build can emulate
rsemu run nes-ntsc --cart game.nes
rsemu run apple1                                # interactive over the terminal (own `rsmon` monitor; --monitor wozmon)
rsemu run pc-at --floppy dos.img                # own legacy BIOS unless --bios given
rsemu run riscv-virt --drive hd0=disk.qcow2 -p <name>=<value>   # --drive needs dev-blk
rsemu debug <machine>                           # stopped, GDB on :1234 (feature gdb)
rsemu devices ; rsemu describe <class> ; rsemu convert <machine>
```
Other run options: `--threading deterministic|parallel[:N]` and `--accel kvm` (feature `accel-kvm`, Linux x86-64).

**Key features / cargo features:** `std`, `cli` (default); `machine-*` (e.g. `machine-nes`, `machine-sms`, `machine-gameboy`, `machine-pc-at`, `machine-q35-linux`, `machine-riscv-virt`, `machine-arm64-virt`, `machine-amiga-a500`, `machine-mac-plus`); `cpu-*`/`dev-*` building blocks; `jit`, `jit-x86`, `jit-arm64`, `jit-wasm`; `gdb`; `ffi` (build with `cargo rustc --lib --features ffi --crate-type staticlib`, header `include/rsemu.h`); `wasm`, `demo`; `usermode` (hooks for Linux user-mode emulation, used by the separate `nixvm` project); `accel-kvm`.

**Gotchas:** the library API (`Machine::run_until`, snapshots and so on) is not stable, and the library `Config` types are `#[non_exhaustive]` since 0.0.8 (code outside the crate cannot build them with struct literals). Read `ROADMAP.md` and docs.rs before embedding. `--no-default-features` gives the `no_std` core.

## algoc

**Repo:** https://github.com/KarpelesLab/algoc · **Crate:** `algoc` (git only) · **License:** MIT · **Status:** experimental/usable. The JavaScript and Python back ends work.

A transpiler from a Rust/C-like algorithm DSL (`.algoc`) to **JavaScript and Python**. It is meant for writing crypto, compression and codec code once and generating it for several languages. The DSL has fixed-width and **endian-typed integers** (`u32be`, `u64le`, with slice casts like `buf[0..4] as u32be`), structs/enums/`match`, interfaces with monomorphized generics, built-ins (`rotr`, `bswap`, `constant_time_eq`, `secure_zero`) and embedded test vectors. The stdlib includes SHA-256, SHA-1, MD5, HMAC, CRC32, AES, DEFLATE/gzip decode, PNG and BMP.

**Use it when:** you need identical algorithm implementations in JS and Python generated from one source.
**Don't use it when / limits:** you need Rust or C output (not a target), or a general-purpose language. Dependencies: `thiserror`, `ariadne`.

**Install / run:**
```sh
cargo install --git https://github.com/KarpelesLab/algoc
algoc check   stdlib/hash/sha256.algoc
algoc compile stdlib/hash/sha256.algoc -t js -o sha256.js     # -t js | py ; --test includes tests
algoc test    stdlib/hash/sha256.algoc -t py                  # run embedded vectors via interpreter
algoc parse   stdlib/hash/sha256.algoc                        # dump AST
```

**Example (DSL):**
```
use "stdlib/runtime.algoc"

fn write_hash(output: &mut [u8], h: &[u32; 8]) {
    for i in 0..8 {
        output[i * 4..i * 4 + 4] as u32be = h[i];
    }
}

test sha256_abc {
    input: bytes("abc"),
    expect: hex("ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad")
}
```

**Gotchas:** JS output uses `BigInt` for 64/128-bit values, and `use` paths are relative to the working directory (e.g. `stdlib/...`).
