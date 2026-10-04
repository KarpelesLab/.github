# Internationalization, Time, Mathematics & Geometry

Pure-Rust, dependency-free building blocks for text and locale handling, dates and time zones, arbitrary-precision math, SMT solving and 2D geometry. `timezone-data` is the base layer: an embedded, pre-parsed IANA tz database used by both `strtotime` (PHP-style date parsing) and `intl` (an ICU analog covering Unicode algorithms and CLDR formatting). `puremp` is a GMP/MPFR-class bignum library, and `z3rs` is a pure-Rust port of the Z3 theorem prover built on top of `puremp`. `polyclip` does exact integer 2D polygon geometry (booleans, offsetting, triangulation) and backs the `cadlab` PCB tool. Everything here is `no_std` (most of it with `alloc`) and has no C or `-sys` dependencies.

## Quick pick

| Need | Use |
|------|-----|
| Parse `"next monday"`, `"+2 days"`, `"2023-05-15 EST"` into a Unix timestamp (PHP `strtotime()` semantics) | [`strtotime`](#strtotime-rs) |
| IANA time-zone offsets, DST, transitions, POSIX `TZ` rules without relying on OS tzdata | [`timezone-data`](#timezone-data-rs) |
| Unicode properties, normalization (NFC/NFD), collation, grapheme/word/line segmentation, bidi, IDNA, confusables | [`intl`](#intlrs) |
| Locale-aware number, currency, date, list, relative-time and unit formatting, plural rules, BCP-47 tags (CLDR) | [`intl`](#intlrs) |
| Transliteration to ASCII, spell-out of numbers, non-Gregorian calendars | [`intl`](#intlrs) |
| Big integers, exact rationals, arbitrary-precision floats with correct rounding, decimals | [`puremp`](#puremp) |
| Factorization, primality proofs, polynomials, matrices, finite fields, elliptic curves, number fields | [`puremp`](#puremp) |
| SMT solving (SMT-LIB 2), SAT (DIMACS), a Z3 replacement without native libs | [`z3rs`](#z3rs) |
| Exact integer 2D polygon booleans, offsetting, distance/DRC queries, triangulation | [`polyclip`](#polyclip) |

## strtotime-rs

**Repo:** https://github.com/KarpelesLab/strtotime-rs · **Crate:** `strtotime` (crates.io `0.1.2`) · **License:** MIT · **Status:** usable (matches PHP on its full 669-case test corpus)

A `#![no_std]`, allocation-free port of PHP's `strtotime()` (via the Go library `KarpelesLab/strtotime`). It parses absolute and relative date/time expressions into a Unix timestamp relative to a base instant, in a given time zone. `#![forbid(unsafe_code)]`. The only dependency is the optional `timezone-data` crate.

**Use it when:** you need to accept human- or PHP-style date strings (`"tomorrow"`, `"last Friday"`, `"first day of next month"`, `"@1234567890"`, ISO 8601, HTTP-log dates) and match PHP's behavior exactly, including DST handling.
**Don't use it when / limits:** you need date formatting or calendar arithmetic APIs (it only parses; it returns `i64` seconds or a `DateTime` struct of broken-down fields). English keywords only, as in PHP.

**Add it:**
```toml
[dependencies]
strtotime = "0.1"
# minimal build, no IANA database (UTC, numeric offsets, abbreviations only):
# strtotime = { version = "0.1", default-features = false }
```

**Key features / cargo features:**
- `iana` (default): DST-aware named zones through `timezone-data` 0.2 (`Tz::Iana(zone)`), and IANA names such as `Europe/Paris` inside the input.
- `std`: `now_unix()`, `strtotime_now(input, tz)`, `system_time_from_unix(unix)`, and `From<DateTime> for SystemTime`.
- API: `strtotime(input, base_unix, tz) -> Result<i64, Error>`, `strtotime_civil(...) -> Result<DateTime, Error>`, `strtotime_micros(...) -> Result<i64, Error>`.
- `Tz` enum: `Utc`, `Fixed(i32 seconds east)`, `Iana(timezone_data::Zone)`.
- `DateTime` has public `year, month, day, hour, minute, second, micros, offset` fields and `unix()`, `unix_micros()`, `weekday()`.

**Example:**
```rust
use strtotime::{strtotime, strtotime_civil, strtotime_micros, Tz};

let base = 946728000; // 2000-01-01 12:00:00 UTC
assert_eq!(strtotime("tomorrow", base, Tz::Utc).unwrap(), 946771200);
assert_eq!(strtotime("next year + 4 days", base, Tz::Utc).unwrap(), 978696000);
assert_eq!(strtotime("@1234567890", 0, Tz::Utc).unwrap(), 1234567890);

let dt = strtotime_civil("2008-07-01 22:35:17", 0, Tz::Fixed(2 * 3600)).unwrap();
assert_eq!((dt.year, dt.month, dt.day, dt.hour), (2008, 7, 1, 22));
assert_eq!(dt.unix(), 1214944517);

// Sub-second precision is kept on the civil result and on *_micros
assert_eq!(strtotime_micros("2008-07-01T22:35:17.02", 0, Tz::Utc).unwrap(), 1_214_951_717_020_000);

// Named zone (feature `iana`)
let ny = timezone_data::load("America/New_York").unwrap();
let ts = strtotime("2023-07-04 09:00", 0, Tz::Iana(ny)).unwrap();
```

**Gotchas:**
- `strtotime()` truncates to whole seconds, as PHP does. Use `strtotime_civil` or `strtotime_micros` for fractions. Nanosecond input is truncated to microseconds.
- `base_unix` is ignored for fully absolute inputs.
- In DST gaps and folds, wall-clock times resolve using PHP's fall-forward behavior.
- `Tz::Iana` requires a `timezone_data::Zone` from `timezone-data` 0.2. Add `timezone-data = "0.2"` to call `load()` yourself.

## timezone-data-rs

**Repo:** https://github.com/KarpelesLab/timezone-data-rs · **Crate:** `timezone-data` (crates.io `0.2.0`) · **License:** MIT (data: IANA, public domain) · **Status:** usable

A `#![no_std]`, no-`alloc`, zero-dependency crate that embeds the complete IANA tz database (598 zones plus `zone1970.tab`/`iso3166.tab` metadata), pre-parsed at build time into `&'static` Rust data. `load()` is a binary search: nothing is parsed at runtime, there is no build script, and it never reads the host's `/usr/share/zoneinfo`. `#![forbid(unsafe_code)]`. It is a port of the Go package `gotz`.

**Use it when:** you need UTC offsets, DST flags, abbreviations, historical transitions, leap seconds or POSIX `TZ` rules for named zones in embedded, WASM or static binaries, or you want tz behavior that does not depend on the host OS.
**Don't use it when / limits:** you want a full date/time library (it deals only in `i64` Unix seconds; convert to `chrono`/`time` yourself). The data changes only when a new crate version ships. It does not parse arbitrary TZif files; 0.2 removed runtime `parse()`.

**Add it:**
```toml
[dependencies]
timezone-data = "0.2"
```

**Key features / cargo features:** no cargo features.
- `load(name)` (`""`/`"UTC"` gives UTC), `load_insensitive(name)`, `names()`, `parse_posix_tz(s)`.
- `Zone`: `lookup(unix) -> ZoneType { abbrev, offset, is_dst }`, `types()`, `transitions()`, `leap_seconds()`, `transitions_for_range(start, end)` (stored plus POSIX-generated transitions), `extend()`/`extend_raw()` (POSIX footer), `meta()` (countries, lat/lon), `name()`.
- `Error` is `#[non_exhaustive]`: `NotFound`, `BadPosixTz(&'static str)`.

**Example:**
```rust
use timezone_data::load;

let z = load("America/New_York")?;
let zt = z.lookup(1_700_000_000);
println!("{} offset={} dst={}", zt.abbrev, zt.offset, zt.is_dst);

if let Some(rule) = z.extend() {
    let (start, end) = rule.transitions_for_year(2025).unwrap();
    println!("DST starts {start}, ends {end}");
}
if let Some(m) = z.meta() {
    if let Some(c) = m.countries().next() {
        println!("{} ({}) at {}, {}", c.name, c.code, m.lat, m.lon);
    }
}
# Ok::<(), timezone_data::Error>(())
```

**Gotchas:**
- 0.2 is a breaking change from 0.1: `Zone` and `ZoneType` lost their lifetime parameter, and `types()`, `transitions()` and `leap_seconds()` now return slices. `strtotime` 0.1.2 and `intl` 0.6.4 both use 0.2, so they share one copy and one `Zone` type. Older `intl` releases (0.6.3 and earlier) still pulled in 0.1.
- `Zone` is `Copy`, so it is cheap to pass by value.

## intlrs

**Repo:** https://github.com/KarpelesLab/intlrs · **Crate:** `intl` (crates.io `0.6.4`) · **License:** MIT · **Status:** usable (official Unicode conformance suites pass 100% for normalization, collation, grapheme/word/sentence; about 99.98% for line breaking and 99.996% for bidi)

A pure-Rust, always-`#![no_std]` analog of ICU, targeting Unicode 17.0.0 and CLDR. UCD property tables are compiled into `const fn` `match` lookups, and CLDR data is embedded with `include_bytes!`, so nothing is initialized at runtime. The formatters need `alloc`, but the property lookups and plural rules work without an allocator. MSRV is 1.88 and edition 2024. The only optional dependency is `timezone-data` 0.1.

**Use it when:** you need Unicode text algorithms (NFC/NFD/NFKC/NFKD, UTS #10 collation with locale tailoring and numeric ordering, UAX #29/#14 segmentation, UAX #9 bidi, case mapping and folding, UAX #31 identifiers, UTS #39 confusables, UTS #46 IDNA/Punycode, scripts, East Asian width, character names). It also covers ECMA-402-style formatting: numbers, currency, compact, percent, units, durations, dates and skeletons, lists, relative time, display names, plural rules, a MessageFormat subset, BCP-47 parsing with likely subtags and negotiation, RBNF spell-out, transliteration, non-Gregorian calendars, and localized time-zone names.
**Don't use it when / limits:** you need full ICU parity. IDNA does not yet enforce the contextual CheckBidi/CheckJoiners rules, MessageFormat is only a subset, and the Chinese lunisolar calendar covers 1900-2099. The default build is several MB of tables, so trim features for WASM or embedded targets.

**Add it:**
```toml
[dependencies]
intl = "0.6"
# size-trimmed example: collation only (pulls case + normalization + alloc)
# intl = { version = "0.6", default-features = false, features = ["collation"] }
```

**Key features / cargo features:** everything is on by default; opt out with `default-features = false`.
- Range tiers: `ascii` < `latin1` < `bmp` < `full` (default). Codepoints outside the compiled tier report `Unassigned`/`false`.
- Unicode components: `normalization`, `segmentation`, `bidi`, `case`, `identifiers`, `collation` (about 1.9 MB), `idna`, `confusables`, `names` (about 1.3 MB). Opt-in only: `segmentation-dict` (Thai), `-cjk`, `-lao`, `-km`, `-my`, and `collation-zh`.
- CLDR formatters: `number`, `number-numsys`, `number-range`, `currency`, `units`, `units-narrow`, `datetime`, `calendars-extra` (or per-calendar `cal-*`), `displaynames`, `list`, `relative`, `message`, `spellout`, `transliterate`, `locale`, `iana-tz`.
- `tz-names` (about 2.1 MB) and the per-area `tz-names-america`, `tz-names-europe` and so on are **not** in the default build. Without them, localized zone names fall back to GMT offsets.
- Always available with no feature: `plural`, `calendar` arithmetic, and the basic property lookups (`general_category`, predicates via `CharExt`, `script`, `east_asian_width`, `numeric_value`).

**Example:**
```rust
use intl::unicode::{general_category, GeneralCategory, CharExt, nfc, graphemes};
use intl::unicode::collate::compare;
use intl::number::{format_decimal, format_currency};
use intl::plural::{plural_category, PluralOperands, PluralCategory};
use intl::spellout::spell_cardinal;
use intl::translit::latin_ascii;
use core::cmp::Ordering;

assert_eq!(general_category('A'), GeneralCategory::UppercaseLetter);
assert!('٣'.is_numeric());
assert_eq!(nfc("e\u{0301}".chars()).collect::<String>(), "é");
assert_eq!(compare("café", "cafz"), Ordering::Less);
let clusters: Vec<&str> = graphemes("e\u{0301}x").collect(); // ["é", "x"]

assert_eq!(format_decimal("de", 1234.5), "1.234,5");
assert_eq!(format_currency("en", 1234.5, "USD"), "$1,234.50");
assert_eq!(plural_category("pl", &PluralOperands::from_int(5)), PluralCategory::Many);
assert_eq!(spell_cardinal("en", 21).as_deref(), Some("twenty-one"));
assert_eq!(latin_ascii("Straße"), "Strasse");

// IANA zones (feature `iana-tz`)
let ny = intl::timezone::load_zone("America/New_York").unwrap();
let offset = ny.offset_at(1_700_000_000); // seconds east of UTC
```

**Other entry points (verified names):** `unicode::collate::Collator::new(..).with_numeric(true)`, `Tailoring::for_locale("sv")`, `unicode::idna::to_ascii`, `unicode::spoof::skeleton`, `unicode::words`/`sentences`/`line_breaks`, `unicode::bidi::process`, `locale::Locale::parse(..).maximize()`, `locale::negotiate`, `number::format_compact`, `list::format_list(locale, &[..], &ListFormatOptions)`, `relative::format_relative(locale, f64, RelativeUnit::Day, &opts)`, `datetime::format_date(locale, &DateTime, DateStyle::Long)`, `datetime::DateTime::parse_iso8601`, `unit::format_unit`, `unit::format_duration`, `display::language_name`, and `display::region_name`.

**Gotchas:**
- Most formatter functions take a BCP-47 locale string as their first argument and return `String`. Unknown locales fall back through CLDR inheritance to root.
- Missing non-Gregorian calendar features produce a compile error (`format_<cal>_date` does not exist) or `None`, never a wrong Gregorian string.
- Number formatting uses the locale's default numbering system, as ECMA-402 does. For example, `ar-EG` yields Arabic-Indic digits.

## puremp

**Repo:** https://github.com/KarpelesLab/puremp · **Crate:** `puremp` (crates.io `0.2.5`) · **License:** MIT · **Status:** usable (broad, actively developed; some advanced algorithms are marked correctness-first rather than tuned for speed)

A clean-room, pure-Rust GMP+MPFR-class arbitrary-precision library covering `Nat`, `Int`, `Rational`, `InfRational`, correctly rounded binary `Float` and `FixedFloat`, `Decimal`, `Dyadic`, `Padic`, `Complex<T>`, `ModInt`, `Poly<T>`, `Matrix<T>`, `Interval`, `Ball`, `GaloisField`, `EllipticCurve`, `Algebraic`/`Quadratic` and `NumberField`. It is `no_std` + `alloc` (CI-verified on `thumbv7em-none-eabi`), its core has zero dependencies, and `unsafe` is denied everywhere except the opt-in C ABI. It also ships a CLI calculator and a C library (`include/puremp.h`). MSRV is 1.88 and edition 2024.

**Use it when:** you need big-integer or exact-rational arithmetic, multi-precision floats with directed rounding (`Float::pi`, `sqrt`, special functions), or number theory (factorization via Pollard rho, ECM and a quadratic sieve; BPSW; primality certificates including ECPP; CRT; `sqrt_mod`; discrete log). It also handles exact polynomial and matrix algebra over ℤ, ℚ, ℤ/nℤ and GF(pᵏ), LLL, PSLQ/`identify`, and elliptic-curve point counting (Schoof).
**Don't use it when / limits:** you need constant-time or side-channel-resistant arithmetic for cryptography; use the sibling `purecrypto` instead. It is not a drop-in replacement for GMP's C API. GNFS is correct only to about 19 digits and is not tuned.

**Add it:**
```toml
[dependencies]
puremp = "0.2"
# lean: integers + rationals only, no_std + alloc
# puremp = { version = "0.2", default-features = false, features = ["rational"] }
```
The CLI is `cargo install puremp`, which installs the `puremp` REPL. It supports `+ - * / % **`, parentheses and `name = expr`.

**Key features / cargo features:**
- The default build enables almost everything: `std rational float dyadic padic decimal complex poly interval ball matrix lattice identify algebraic primality galois elliptic numberfield cli`.
- Opt-in: `dlog`, `gnfs`, `ffi` (C ABI; build with `cargo rustc --lib --features ffi --crate-type staticlib|cdylib`), `serde`, `rand` (a `rand_core` bridge), `num-traits`.
- Type layers stack: `int` is the base, then `rational`, then `float`, then `interval` and `ball`. `--no-default-features --features int` gives bare integers.

**Example:**
```rust
use puremp::{Float, Int, Rational, RoundingMode};

let big = Int::from_i64(2).pow(100);
assert_eq!(big.to_string(), "1267650600228229401496703205376");
let n: Int = "1000000007".parse().unwrap();
assert!(n.is_prime_bpsw());
let r = n.modpow(&Int::from_i64(5), &Int::from_i64(97));

let sum = Rational::new(Int::from_i64(1), Int::from_i64(2))
    .add(&Rational::new(Int::from_i64(1), Int::from_i64(3)));
assert_eq!(sum.to_string(), "5/6");

let pi = Float::pi(200, RoundingMode::Nearest); // 200-bit precision
assert_eq!(pi.to_decimal_string(20), "3.14159265358979323846");
```

**Gotchas:**
- `Rational::new` returns `Rational` and **panics** on a zero denominator. Use `Rational::checked_new` to get an `Option` instead.
- Arithmetic is mostly method-based (`add`, `pow`, `modpow`) and takes references. `Float` operations take an explicit precision in bits and a `RoundingMode`.
- The `Int` API is not constant-time.

## z3rs

**Repo:** https://github.com/KarpelesLab/z3rs · **Crate:** `z3rs` (crates.io `0.0.9`) · **License:** MIT (derivative of Microsoft's MIT-licensed Z3) · **Status:** experimental to usable (all Z3 theories present and sound, 0 known wrong verdicts across about 90k differential-fuzzed scripts, completeness and performance still behind upstream)

A pure-Rust, `no_std` + `alloc` port of the Z3 theorem prover, pinned to Z3 v4.17.0 and ported file by file. There is no GMP, no C and no `-sys` crate, and the only dependency is `puremp`. It includes a full SMT-LIB 2 front end, a CDCL SAT core (DIMACS, DRAT checking), DPLL(T) with Nelson-Oppen, and theories for UF, LIA/LRA, NRA (CAD), NIA, bit-vectors, arrays, datatypes, floating point, strings/sequences/regex, quantifiers (E-matching), CHC/Horn, Datalog and optimization. It also has a Z3-compatible CLI and a C ABI subset of `z3_api.h`.

**Use it when:** you need an SMT or SAT solver in a pure-Rust, WASM or embedded build, or you want to avoid linking native Z3 through the `z3`/`z3-sys` crates.
**Don't use it when / limits:** you need upstream Z3's full completeness or speed on large problems. z3rs returns a sound `unknown` where it cannot decide; see `PARITY.md` and `ROADMAP.md` in the repo. The API is SMT-LIB-text-centric and differs from the `z3` crate on crates.io, which is unrelated bindings to native Z3.

**Add it:**
```toml
[dependencies]
z3rs = "0.0.9"
# z3rs = { version = "0.0.9", features = ["std"] }   # timers/threads/std::error::Error
```
The CLI is `cargo install z3rs`, then `z3rs problem.smt2` or `z3rs problem.cnf`. Flags include `-dimacs`, `-dl` (Datalog), `-drat <cnf> <proof>` and `-memory:<MB>`, and the environment variable `Z3RS_MEMORY_LIMIT_MB` also sets a memory limit.

**Key features / cargo features:**
- The default is `no_std` + `alloc`. `std` enables wall-clock timeouts, threads, filesystem helpers and `std::error::Error`. `ffi` enables the C ABI (`include/z3rs.h`).
- `z3rs::api::Solver` is an incremental solver driven by SMT-LIB 2 text. It provides `assert`, `check`, `check_assuming`, `get_value`, `get_model`, `get_unsat_core`, `push`, `pop`, `reset`, `simplify` and `eval` (raw script to output lines).
- `z3rs::api::build::{Context, Sort}` is a typed term-builder API over the same engine.
- There are also lower-level entry points: `z3rs::cmd_context::run_smt2(script)` and `z3rs::sat::parse_dimacs(text)?.solve()`. It re-exports `puremp`.

**Example:**
```rust
use z3rs::api::{SatResult, Solver};
use z3rs::api::build::{Context, Sort};

let mut s = Solver::new();
s.assert("(declare-const x Int)(assert (> x 5))(assert (< x 8))").unwrap();
assert_eq!(s.check().unwrap(), SatResult::Sat);
let v = s.get_value("x").unwrap(); // "6" or "7"

let mut ctx = Context::new();
let x = ctx.const_("x", Sort::Int);
let y = ctx.const_("y", Sort::Int);
ctx.assert(&x.gt(&y));
ctx.assert(&x.lt(&y.add(&Context::int(1))));
assert_eq!(ctx.check(), SatResult::Unsat);
```

**Gotchas:**
- `Solver` methods return `Result<_, String>` for parse and type errors. `SatResult::Unknown` means a budget ran out or the fragment is undecided, and it is never a guess.
- The crate name is `z3rs`, not `z3`. The `z3` crate on crates.io is a different project that binds native Z3.
- Version 0.0.x means the API may change between releases.

## polyclip

**Repo:** https://github.com/KarpelesLab/polyclip · **Crate:** `polyclip` (crates.io `0.0.4`) · **License:** MIT · **Status:** usable, pre-1.0. It is covered by property tests, weekly `cargo-fuzz` runs of every public operation, and differential tests against Clipper2 (the `oracle/` crate). It was built for and is used by [`cadlab`](apps-engines.md#cadlab).

2D polygon geometry on integer `i64` coordinates with exact predicates (`i128`/wide arithmetic). It does booleans, offsetting, arc approximation, distance queries, fracturing, triangulation and simplification. Output is **guaranteed valid after snap rounding**: simple rings, no crossings, correct nesting, vertices moved by at most √2/2. It is also **canonical and bit-identical across platforms**: outer rings are CCW and holes CW, each ring starts at its smallest vertex, and polygons are sorted. The crate does not panic, uses no `unsafe` and has no global state. Its only required dependency is `libm`, which keeps arc results deterministic.

**Use it when:**
- You need robust polygon clipping, offsetting or minimum-distance checks where floating-point drift is unacceptable, as in PCB/EDA (zone fills, DRC, Gerber regions), CNC/laser toolpaths, GIS on a fixed grid, or deterministic simulations.
- You want a pure-Rust Clipper2 alternative with stronger validity guarantees.

**Don't use it when / limits:**
- You work in floats. Pick a scale yourself (cadlab uses 1 unit = 1 nm). The supported range is ±2^40, and coordinates outside it return an error.
- `curved_boolean` keeps arcs as arcs, but it approximates and then reconstructs them, so short polyline pieces remain near tangencies.
- A union of 50,000 heavily overlapping circles takes about 3 s, single-threaded (Clipper2 takes minutes on the same input). Typical zone-fill, offset and distance workloads match or beat Clipper2.
- The API is still 0.0.x. Pin the patch version.

**Add it:**
```toml
[dependencies]
polyclip = "0.0.4"
# polyclip = { version = "0.0.4", features = ["serde", "rayon"] }
```

**Key features / cargo features:**
- **Booleans:** `boolean(op, &a, &b, rule)`, the `Boolean::new().subject(..).clip(..).op(..).execute()` builder, and `union_all` (an N-ary union that also normalizes self-intersecting input). Fill rules are `EvenOdd`, `NonZero`, `Positive` and `Negative`. Output is a `PolygonSet` (`Vec<Polygon { outer, holes }>`) or a `PolyTree`. `clip_paths` handles open paths.
- **Offsetting:** `offset`, `offset_tree`, `offset_paths` (`EndCap`), `offset_shape`, and `opening`/`closing` for minimum-width enforcement. Joins are `Round`, `Miter`, `Bevel` and `Square`.
- **Arcs:** `Circle`, `Curve` and `Shape` are approximated by `ArcTol::new(max_err, Side::{Outside, Inside, Nearest})`, so clearances are never under-estimated.
- **Incremental:** `ZoneFill` inserts, updates or removes obstacles by `u64` id, and refills in well under a millisecond with the same result as a full recompute.
- **Provenance tags:** a `u64` per edge survives booleans and offsets (the `*_tagged` functions, `arcs_from_tags`).
- **Queries:** `area2`, `centroid`, `locate`, `intersects`, `contains`, `distance` (with the closest points), `distance_sq`, and a fast `distance_less_than` for DRC.
- **Utilities:** `validate`, `fracture`/`fracture_set` (holes joined by zero-width cut-ins, for Gerber regions), `simplify_*`, `convex_hull`, `minkowski_sum`, `triangulate`, `triangulate_delaunay`, `trapezoids`.
- `serde` derives `Serialize`/`Deserialize` for all types. `rayon` parallelizes boolean phases, and the output is identical for any thread count.

**Example:**
```rust
use polyclip::*;

fn main() -> polyclip::Result<()> {
    // 1 unit = 1 nm here, so 10_000_000 = 10 mm.
    let zone = Ring::from([(0, 0), (10_000_000, 0), (10_000_000, 10_000_000), (0, 10_000_000)]);
    let tol = ArcTol::new(1_000, Side::Outside); // arcs within 1 um, never inside the true arc
    let pad = Circle::new(Point::new(3_000_000, 3_000_000), 500_000).to_ring(tol)?;
    let keepout = offset(&pad, 200_000, Join::Round, tol)?; // 0.2 mm clearance

    let fill = boolean(Op::Difference, &zone, &keepout, FillRule::NonZero)?;
    assert_eq!((fill.len(), fill[0].holes.len()), (1, 1));

    let regions: Vec<Ring> = fill.iter().map(fracture).collect::<Result<_>>()?; // Gerber regions
    assert_eq!(regions.len(), 1);

    let track = Path::from([(0, 2_500_000), (10_000_000, 2_500_000)]);
    assert!(distance_less_than(&track, &pad, 200_000)); // DRC-style check

    let mut zf = ZoneFill::new(&zone, FillRule::NonZero)?; // incremental refill
    zf.insert(1, &keepout)?;
    assert_eq!(zf.fill(), fill);
    Ok(())
}
```

**Gotchas:**
- `area2`/`ring_area2` return **twice** the signed area as `i128`.
- Inputs can be any `RingSource` (`Ring`, `Polygon`, `PolygonSet`, slices of rings), so an `offset` result can be passed straight back in as a clip operand.
- cadlab currently pins `polyclip 0.0.2`. Under Cargo's 0.0.x rules each patch release is semver-incompatible, so the two versions do not unify in a dependency tree.
