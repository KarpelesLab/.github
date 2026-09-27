# Systems & Toolchain

Low-level pure-Rust infrastructure: an OS kernel (kintane), three ways to run without libc (purestd, fullrust, rustlibc), a compiler back-end framework (latticefoundry), a standalone linker (qld) and assembler (rsasm), an HDL compiler (reticle), a byte-exact decompiler/recompiler (univdreams), a multi-machine emulator (rsemu), and a pseudocode-to-JS/Python transpiler (algoc). They are independent projects. None depends on another, apart from purestd and fullrust, which are built to work together. The libc story in brief: **purestd** is a `std`-shaped library built on raw syscalls; **fullrust** is (a) a patched rustc whose *real* `std` uses raw syscalls, shipped as a Docker image or GitHub Action, and (b) the small runtime crate plus `cargo fullrust` that make purestd programs fully libc-free; **rustlibc** runs the other way and gives **C** programs a libc written in Rust. latticefoundry has its own assembler/linker (`lf-as`, `lf-ld`) and does not use qld or rsasm. qld and rsasm are general-purpose drop-in replacements for GNU ld and GNU as.

## Quick pick

| Need | Use |
|------|-----|
| Build an unmodified Rust crate into a fully static, libc-free Linux x86-64 binary | [`fullrust`](#fullrust) (Docker image / GitHub Action) |
| A `std`-like API over raw syscalls for a `#![no_std]` program, zero deps | [`purestd`](#purestd) |
| A libc (`libc.a`/`.so`) written in Rust for C programs | [`rustlibc`](#rustlibc) |
| A fast, GNU-ld-compatible linker, or linking in-process from Rust | [`qld`](#qld) |
| Assemble GNU as / NASM / vendor syntax for about 20 CPU families, from a CLI or a library | [`rsasm`](#rsasm) |
| Compile, simulate, synthesize or formally verify Verilog/VHDL, or program an FPGA | [`reticle`](#reticle) |
| Decompile an ELF/PE/Mach-O/NE binary to editable source that recompiles byte for byte | [`univdreams`](#univdreams) |
| Call exports of a 32-bit Windows DLL from Rust in a sandbox | [`univdreams`](#univdreams) (`ud-emulator`) |
| Emulate NES/Game Boy/SMS/Amiga/Mac/PC/RISC-V/AArch64 machines, or embed an emulator | [`rsemu`](#rsemu) |
| An LLVM-like compiler back end (SSA IR, optimizer, x86-64 codegen, linker) | [`latticefoundry`](#latticefoundry) |
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

## latticefoundry

**Repo:** https://github.com/KarpelesLab/latticefoundry · **Crate:** `latticefoundry` (git only) · **License:** Apache-2.0 · **Status:** experimental. The README says "early scaffold", but `ROADMAP.md` says phases 0–9 are done and `lf build` produces running static x86-64 executables.

A clean-room compiler back-end framework in the role of LLVM. It provides a typed SSA IR (block arguments, poison/freeze, opaque pointers), a verifier backed by the [z3rs](https://github.com/KarpelesLab/z3rs) SMT solver, an optimizer (mem2reg, SCCP, simplify_cfg, DCE, e-graph equality saturation, LICM, inlining, `-O0..-O3`, LTO), x86-64 codegen (AArch64 also implemented), DWARF output, its own linker producing static ELF64, and a JIT. The only dependencies are the sibling crates `z3rs` and `puremp`. `unsafe` is limited to the JIT's executable memory.

**Use it when:** you are building a language front end and want a pure-Rust back end, or you are researching verified optimization.
**Don't use it when / limits:** you need production codegen or many targets. The output is x86-64 Linux static ELF only (no libc linking), and the AArch64 backend is validated only via an interpreter. `lf-as` and `lf-dis` are stubs. The public API is unstable. Not on crates.io.

**Build / run:**
```sh
cargo build --release          # binaries: lf, lf-opt, lf-ld, lf-as, lf-dis
lf build foo.lf -o foo -O2 -g  # .lf (text) / .lfb (binary) IR -> static executable; --lto, --entry, --no-verify
lf-opt -O2 foo.lf -o foo.lfb   # verify/optimize/re-encode IR; -p mem2reg,sccp,... ; --emit lf|lfb
lf-ld -o out -e main a.lfo b.lfo
```

**C front end:** `lf-cc/` is a separate crate in the same repo (not built from the root). Build it with cargo run inside `lf-cc/`:
```sh
lf-cc -O2 -g -o hello hello.c     # -S / --emit-lf dumps the lowered .lf IR
```

**Gotchas:** it is a single package, not a workspace. Rust 1.88+ (edition 2024). Design is documented in `docs/design-tenets.md` and `docs/ir-design.md`.

## qld

**Repo:** https://github.com/KarpelesLab/qld · **Crate:** `qld` (crates.io `0.1.0`) · **License:** MIT · **Status:** pre-alpha, but it already links real software. Used as the system linker, it builds and passes the test suites of coreutils, curl, OpenSSL, Python, LLVM and rustc, and it links a Linux kernel that boots.

A fast, parallel, deterministic linker in pure Rust and a **drop-in replacement for GNU ld/gold/lld/mold**: it accepts their argv, linker scripts and the LTO plugin API. x86-64 Linux ELF is the solid path: static/dynamic/PIE/static-PIE, `-shared`, `-r`, symbol versioning, RELRO, `DT_RELR`, `--gc-sections`, `--icf`, compressed debug, `--build-id`, `MEMORY`/`PHDRS` scripts, and `binary`/`ihex`/`srec` output. AArch64 ELF and PE32+ (MinGW) also link. It is roughly 5–8x faster than GNU ld on large links. It is also a library with no global state, no `process::exit`, and in-memory inputs and outputs.

**Use it when:** you want a faster system linker for C/C++/Rust on x86-64 Linux, or you need to link objects from inside a Rust tool.
**Don't use it when / limits:** you need other architectures or Mach-O (planned, not ready). The library API is unstable before 1.0; only `args`, `diag`, `error` and `target` plus the root re-exports are documented. MSRV 1.89. Dependencies: `rayon`, `memmap2`, `hashbrown`, `foldhash`.

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

**Repo:** https://github.com/KarpelesLab/reticle · **Crate:** `reticle` (crates.io `0.0.3`) · **License:** MIT · **Status:** early. The full pipeline runs end to end, and designs have been run on real FPGAs (Basys 3/Artix-7, Cynthion/ECP5, Tang Primer 20K/Gowin).

A from-scratch **Verilog-2005/SystemVerilog and VHDL-2008 compiler**. The default build has **zero dependencies** and a sans-I/O library core; `unsafe` is denied except in the C ABI and wasm modules. Both languages lower to one IR, which feeds a 4-state event-driven simulator (VCD/FST, SVA/PSL subset, coverage), synthesis (FF/memory/FSM inference, AIG optimizer, LUT and standard-cell mapping), formal verification (in-crate CDCL SAT: BMC, k-induction, equivalence), static timing and CDC analysis, and FPGA/ASIC back ends. It also has an IP manifest/registry system, formatters, an LSP server, an HTML schematic viewer, a C ABI and a WASM build. It can write Xilinx 7-series (Project X-Ray), ECP5 (Trellis) and Gowin (Apicula) bitstreams and program boards over JTAG with no vendor tools.

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
reticle program out.bit                    # needs --features program
reticle fmt --write counter.v ; reticle lsp
```

**Add it (library):**
```toml
[dependencies]
reticle = { version = "0.0.3", default-features = false, features = ["verilog", "sim"] }
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

**Repo:** https://github.com/KarpelesLab/univdreams · **Crates:** workspace of `ud-*` crates; CLI is `ud-cli` (binary `ud`) (crates.io `0.2.0`; repo is at `0.3.0`) · **License:** MIT · **Status:** usable for its core promise (byte-identical round-trip). High-level decompilation is partial.

A compiler **and** decompiler suite. It decompiles a binary to `.ud` source whose directives pin the compiler's choices (encodings, layout, padding), so that recompiling reproduces the input **byte for byte**. Formats: ELF64 (x86-64, x86, aarch64), PE/COFF (MSVC/MinGW), thin Mach-O (x86-64, arm64), 16-bit Windows NE, and raw 6502. It lifts to structured statements (if/switch/goto, calls with argument folding), reads DWARF signatures, and discovers functions via symtab, eh_frame, exports and signatures. It also includes a sandboxed **i386 Win32 emulator** (`ud-emulator`). Runs in the browser: https://karpeleslab.github.io/univdreams/. Dependencies include `iced-x86` and `gimli`, so it is not dependency-free.

**Use it when:** you are reverse engineering or binary patching (edit a function, rebuild an otherwise identical binary), inspecting what a compiler emitted, or safely calling into a 32-bit Windows DLL (codecs, malware triage) from Rust.
**Don't use it when / limits:** you need readable high-level C. Loops come out as `goto`, and there is no type or struct recovery without DWARF. It does not handle fat Mach-O, 32-bit Mach-O, 32-bit ARM, or packed/vectorized/LTO binaries. Editing a Mach-O invalidates its code signature.

**Install / run:**
```sh
cargo install ud-cli                         # installs `ud`
ud decompile path/to/binary > prog.ud        # auto-detects ELF / PE / Mach-O / NE / 6502
ud compile prog.ud -o rebuilt                # dispatches on @module.format
ud roundtrip --through-source path/to/binary --out /tmp/rebuilt
ud verify prog.ud                            # check @asm lines against pinned bytes
ud analyze codec.dll                         # run DllMain in the i386 sandbox, report Win32 calls/coverage
```

**Crates:** `ud-core`, `ud-ir`, `ud-ast`, `ud-format` (byte-exact ELF/PE/Mach-O/NE/raw read+write), `ud-arch-x86`, `ud-arch-aarch64`, `ud-arch-6502`, `ud-arch-bpf`, `ud-arch-codec`, `ud-analysis`, `ud-signatures`, `ud-debug` (DWARF), `ud-translate` (`.ud` <-> binary), `ud-emulator`, `ud-cli`, `ud-wasm`.

**Example (`ud-emulator`, call a DLL like `dlopen`):**
```rust
use ud_emulator::Guest;

let bytes = std::fs::read("codec.dll")?;
let mut guest = Guest::load("codec.dll", &bytes)?;          // also runs DllMain
let version: u32 = guest.call("GetCodecVersion", ())?;      // stdcall default
let ptr = guest.alloc(&vec![0u8; 4096])?;
let rc: i32 = guest.call("Decompress", (ptr, 4096u32))?;
let out = guest.read(ptr, 4096)?;
```
Variants: `Guest::load_raw` (skip DllMain), `load_into`/`load_raw_into` (bring your own `Sandbox` with an instruction budget or VFS `Context`), `alloc_cstr`, `write`, `sandbox_mut()`. The guest has no host filesystem, network or registry access unless you attach it. Use the git dependency if a feature is missing from the crates.io 0.2.0 release.

## rsemu

**Repo:** https://github.com/KarpelesLab/rsemu · **Crate:** `rsemu` (crates.io `0.0.5`) · **License:** MIT · **Status:** early but it boots software it did not write: Linux on RISC-V/x86/AArch64 (including SMP), AmigaOS/Workbench on 8 Amiga models, FreeDOS on `pc-at`, and Macintosh Plus/Classic. NES passes AccuracyCoin 141/141. These OS boots are not run in CI because guest images are not shipped.

A multi-platform emulator built on a generic framework: address spaces, clock-domain trees, buses, devices, a translation IR and JITs. Machines are described in **`.machine` files** rather than compiled in; 45 ship in `machines/`. CPU cores: 6502/65C02, Z80, SM83, 8086/80386/x86-64, 68000–68040 + 68881, MIPS R3000, RISC-V RV64GC, ARMv5TE/ARMv7-M, AArch64. It is conformance-measured against SingleStepTests, riscv-tests and others. The core is `no_std + alloc`, **zero dependencies by default**, deterministic (bit-reproducible, snapshot/rewind), with `unsafe` confined to a handful of modules. Execution engines are `interp`, `jit` (portable IR), `jit-host` (x86-64 native code) and `jit-wasm`. It also supports KVM acceleration via raw ioctls, a GDB stub, a C ABI, and `wasm32` (browser) builds.

**Use it when:** you want to run or debug retro or modern guests, build a board from a description file, or embed a deterministic emulator in a test harness (Rust, C ABI or wasm).
**Don't use it when / limits:** you need QEMU-level device breadth or speed today. The default build includes **no machines**: each machine and device is a cargo feature. No guest images (ROMs, BIOS, kernels) are shipped. MSRV 1.88, stable only.

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

**Gotchas:** the library API (`Machine::run_until`, snapshots and so on) is not stable. Read `ROADMAP.md` and docs.rs before embedding. `--no-default-features` gives the `no_std` core.

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
