# Support `raw` vectors for `stream` (and `seed`)

Status: design draft, not started.
Scope: API ergonomics. No change to the underlying RNG state model
(which remains a 64-bit `uint64_t` for `seed` and `stream` everywhere
the generators are seeded today).

## 1. Current state

`stream` is a 64-bit integer that is split into one or two R `integer`s
and packed back into `uint64_t` on the C++ side:

- `R/dqset.seed.R:96` — `dqset.seed(seed, stream = NULL)`; passes
  `stream` to `dqset_seed()`.
- `src/dqrng.cpp:36-49` — `dqset_seed()` declares
  `Rcpp::Nullable<Rcpp::IntegerVector> stream`, converts via
  `dqrng::convert_seed<uint64_t>(stream.as())`, then calls
  `rng->seed(_seed, _stream)`.
- `inst/include/convert_seed.h:104-117` — `convert_seed<T>(...)` is
  overloaded for `const uint32_t*`, `const int*`, and
  `Rcpp::IntegerVector`. Bit-packing is MSB-first, 32 bits per element,
  with explicit overflow checks. No `RawVector` / `uint8_t` overload.
- `inst/include/dqrng_generator.h:62,65,121` — `seed(result_type seed,
  result_type stream)` and the PCG64 specialization both take two
  `uint64_t`. `clone(uint64_t stream)` likewise.
- `inst/include/dqrng_types.h:75-76,297,301` — the abstract base
  declares `seed(uint64_t, uint64_t)` and `clone(uint64_t)`. No raw
  surface.
- `src/generateSeedVectors.cpp` — returns a `list` of `IntegerVector`s
  of length `nwords` (default 2). Documents the bit-packing convention.
- `R/zzz.R:2,14` — only existing global option is
  `dqrng.register_methods`.

The user-facing argument therefore today is one of:

- `integer(1)` (32 bits),
- `integer(2)` (full 64 bits, MSB first).

## 2. Goal

1. **Ingestion (input).** Allow `stream` (and, symmetrically, `seed`)
   to be a `raw` vector. The canonical length is **8** (64 bits). We
   should also accept **4** (32 bits) for parity with the
   integer-scalar shortcut, so the two existing input shapes map
   cleanly to two raw shapes. Larger lengths are out of scope for now
   (see §7).
2. **Reading (output).** `generateSeedVectors()` should be able to
   return `raw` vectors instead of integer vectors. The format is
   controlled by:
   - a new global option `dqrng.seed_format` (`"integer"` | `"raw"`);
     default `"integer"` (fallback = current behaviour),
   - a new per-call argument `format = c("integer", "raw")` that, when
     supplied, overrides the option.
3. Round-trip: a value returned with `format = "raw"` must be accepted
   as-is by `dqset.seed()` / `dqset_seed()` / `clone()` without manual
   conversion.

## 2a. Byte / bit layout (MSB-first)

The current integer-vector convention is **MSB-first**: the first
element holds the most significant bits. The raw-vector convention
will be the same: byte 1 is the most significant byte, byte 8 the
least significant. This matches `writeBin(..., endian = "big")` and
matches `dqrng::convert_seed_internal` (`inst/include/convert_seed.h`),
which shifts left as it consumes each element.

Worked example — same 64-bit value `0x0011223344556677` in three
representations:

```
R integer(2)  (current API):

  stream <- c(0x00112233L, 0x44556677L)
                  │             │
                  │             └──── stream[2] = low  32 bits
                  └────────────────── stream[1] = high 32 bits

  bit positions:  63        32 31         0
                  ▼          ▼ ▼          ▼
                  ┌──────────┐ ┌──────────┐
                  │00112233  │ │44556677  │
                  └──────────┘ └──────────┘
                  ↑ MSB                  ↑ LSB

R raw(8)  (proposed API):

  stream <- as.raw(c(0x00, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77))
                     │                                          │
                     stream[1]                                  stream[8]
                     = high byte                                = low byte

  byte index:    [1]  [2]  [3]  [4]  [5]  [6]  [7]  [8]
                 ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
  value:         │00│ │11│ │22│ │33│ │44│ │55│ │66│ │77│
                 └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘
  bit range:     63.56 55.48 47.40 39.32 31.24 23.16 15.8 7..0
                 ↑ MSB                                     ↑ LSB

C++ uint64_t  (what the RNG actually sees):

  bit position:   63                                              0
                  ▼                                               ▼
                  0x  00   11   22   33   44   55   66   77
                       ↑                                       ↑
                       most significant                        least significant
```

Cross-check against R built-ins:

```r
writeBin(0x0011223344556677, raw(), size = 8, endian = "big")
#> [1] 00 11 22 33 44 55 66 77   ← matches the raw(8) above

# Length-4 raw / length-1 integer carry 32 bits, padded with
# zero in the high half:
stream4 <- as.raw(c(0x44, 0x55, 0x66, 0x77))
# equivalent to integer scalar 0x44556677L
# packs to uint64_t 0x0000000044556677
```

The bit at index `k` of the raw vector lives at C++ bit position
`8*(N-i) + (7-j)` for byte `i ∈ {1..N}`, bit-within-byte `j ∈ {0..7}`.
No host-endianness dependency: bytes are written into the `uint64_t`
in the order they appear in the R vector, via repeated `(sum << 8) |
byte` in `convert_seed_internal`.

We commit to MSB-first / big-endian only. `endian = "little"` is
explicitly **not** supported (see §7).

## 3. Design

### 3.1 C++ layer — `convert_seed.h`

Add a `RawVector` (and matching pointer) overload that performs the
same MSB-first packing as the integer path, but 8 bits per element:

```cpp
// new in inst/include/convert_seed.h
template<typename T>
T convert_seed(const uint8_t* bytes, size_t N) {
    // SHIFT = 8 specialisation of convert_seed_internal
    return convert_seed_internal<T, uint8_t, 8>(bytes, N);
}

template<typename T>
T convert_seed(Rcpp::RawVector bytes) {
    return convert_seed<T>(
        reinterpret_cast<const uint8_t*>(RAW(bytes)), bytes.size());
}
```

Notes:

- `convert_seed_internal` already accepts a `SHIFT` template parameter
  and validates length-vs-output-width, so this is a thin overload.
- For `T = uint64_t` and `N = 8` this packs exactly 64 bits, MSB
  first, matching `writeBin(x, raw(), endian = "big")`-style
  conventions. For `N < 8` the value is left-padded with zero bits
  (i.e. interpreted as the low-order bytes of a 64-bit number) — same
  semantics the integer path uses when given length 1.
- Accept `N \in {1..8}` for 64-bit output; throw a clear error for
  longer raws here (until §6 lands).
- Add specialisations for the existing 16-bit / 32-bit `right_shift` /
  `left_shift` so `uint8_t` does not warn on `>> 8` on platforms with
  16-bit `int` promotion; trivial.

A new `dqset_raw_compat` test in `tests/testthat/cpp/convert.cpp` will
exercise `convert_seed<uint64_t>(RawVector)` against the integer path
to prove parity.

### 3.2 C++ layer — `dqset_seed`

**ABI constraint.** `src/dqrng.cpp` is under
`// [[Rcpp::interfaces(r, cpp)]]` (line 33), so every exported function
is published into `inst/include/dqrng_RcppExports.h` as a free function
with a baked-in `validateSignature()` string. For `dqset_seed`:

```cpp
// inst/include/dqrng_RcppExports.h:32
validateSignature("void(*dqset_seed)(Rcpp::Nullable<Rcpp::IntegerVector>,
                                     Rcpp::Nullable<Rcpp::IntegerVector>)");
```

Downstream packages that already link against `dqrng` have this string
compiled into their binary. If we change `dqset_seed`'s parameter
types in `src/dqrng.cpp`, the registered signature also changes, and
`validateSignature` throws at *their* load time — i.e. silent C++ ABI
break. The same applies to `dqRNGkind`, `dqrng_get_state`,
`dqrng_set_state`, `dqrunif`, `dqrnorm`, `dqrexp`, `dqrrademacher`.

So we do **not** change any existing exported signature. We add a
sibling export and dispatch on the R side:

```cpp
// src/dqrng.cpp — new export, leaves dqset_seed() untouched
// [[Rcpp::export(rng = false)]]
void dqset_seed_raw(Rcpp::Nullable<Rcpp::RawVector> seed,
                    Rcpp::Nullable<Rcpp::RawVector> stream = R_NilValue) {
  if (seed.isNull()) {
    rng = dqrng::generator();
    return;
  }
  uint64_t _seed = dqrng::convert_seed<uint64_t>(seed.as());
  if (stream.isNotNull()) {
    rng->seed(_seed, dqrng::convert_seed<uint64_t>(stream.as()));
  } else {
    rng->seed(_seed);
  }
}
```

The existing `dqset_seed(Nullable<IntegerVector>, Nullable<IntegerVector>)`
stays bit-for-bit identical so its `validateSignature` string is
unchanged.

R-side dispatch happens in `dqset.seed()`:

```r
dqset.seed <- function(seed, stream = NULL) {
  if (is.raw(seed) || is.raw(stream)) {
    dqset_seed_raw(if (is.null(seed)) NULL else as.raw(seed),
                   if (is.null(stream)) NULL else as.raw(stream))
  } else {
    dqset_seed(seed, stream)
  }
}
```

Mixed inputs (e.g. integer `seed` + raw `stream`) are normalised at the
R level — convert the integer side to its raw form before calling
`dqset_seed_raw`, so the C++ entry point only ever sees a single type
per argument. Keeps the C++ surface small and the dispatch obvious.

Note that `dqset_seed_raw` is intentionally **also** added to the
`r, cpp` interface — downstream C++ packages will then be able to
seed dqrng directly from a raw vector after a rebuild against the new
dqrng, without losing the ability to use the old integer entry point.

### 3.3 R layer

- `R/dqset.seed.R:96` — keep `dqset.seed(seed, stream = NULL)`
  unchanged at the R surface, but update the roxygen for both
  `@param seed` and `@param stream` to document the new accepted
  shapes (`integer(1|2)` or `raw(4|8)`). Add examples that use
  `raw(8)`.
- No change to defaults; old code keeps working.

### 3.4 Output side — `generateSeedVectors()`

`src/generateSeedVectors.cpp:43` becomes parameterised:

```cpp
Rcpp::List generateSeedVectors(int nseeds,
                               int nwords = 2,
                               Rcpp::Nullable<Rcpp::String> format = R_NilValue);
```

with an R-side wrapper that resolves `format` from
`getOption("dqrng.seed_format", "integer")` when `NULL`:

```r
# R/generateSeedVectors.R (new thin wrapper, replacing the direct
# RcppExports re-export)
generateSeedVectors <- function(nseeds, nwords = 2L,
                                format = getOption("dqrng.seed_format",
                                                   "integer")) {
  format <- match.arg(format, c("integer", "raw"))
  .Call(`_dqrng_generateSeedVectors`, nseeds, nwords, format)
}
```

Inside C++, when `format == "raw"`, each element of the returned list
is a `RawVector` of length `4 * nwords` (same total entropy, byte-wise
MSB-first). When `format == "integer"`, behaviour is unchanged.

Round-trip test: for every `(nseeds, nwords)`, calling
`convert_seed<uint64_t>` on the raw form yields the same value as the
integer form. (Only validated when `nwords <= 2`; for larger `nwords`
the value is conceptually >64 bit and not representable as a single
`uint64_t` today — out of scope, see §6.)

### 3.5 Global option

Document `dqrng.seed_format` in `?dqrng-functions` and `?generateSeedVectors`:

> `dqrng.seed_format`: one of `"integer"` (default) or `"raw"`.
> Controls the default storage format for seed and stream values
> produced by `generateSeedVectors()`. Individual functions accept a
> `format` argument that overrides the option. Ingestion functions
> (`dqset.seed`, `dqRNGkind`-related entry points) accept both formats
> regardless of the option.

Add to `R/zzz.R`: nothing required at load time — the option is read
lazily where used. (Setting it in `.onLoad` would clobber user
settings; do not do that.)

### 3.6 Symmetric extension to `seed`

Mirror the same dispatch for the `seed` argument of `dqset_seed` (it
already shares the same `convert_seed<uint64_t>` path, so the
overload in §3.1 covers it). No further code change beyond the
TYPEOF-switch in `dqset_seed`.

### 3.7 Header-level API (`clone`, `seed`)

`random_64bit_generator::clone(uint64_t stream)` and
`seed(uint64_t, uint64_t)` stay numeric. They are virtual methods on
the abstract base in `inst/include/dqrng_types.h`; changing their
signatures (even by overloading) would shift the vtable and break
binary compatibility with any downstream package that has a compiled
subclass. The path for raw input is purely additive at the
`convert_seed` overload level (§3.1): downstream code calls
`dqrng::convert_seed<uint64_t>(raw_vec)` and then passes the
`uint64_t` into the existing virtual.

For convenience, expose one free function in
`inst/include/convert_seed.h`:

```cpp
namespace dqrng {
// dispatch helper for downstream R-facing wrappers; pure addition,
// no ABI impact.
inline uint64_t as_uint64_seed(SEXP x);   // INTSXP -> existing path,
                                          // RAWSXP -> new overload,
                                          // NILSXP -> caller's problem
}
```

Inline + new symbol ⇒ ABI-safe.

## 4. Tests

In `tests/testthat/`:

- `test-raw-stream.R` (new):
  - `dqset.seed(seed = 42L, stream = as.raw(c(0,0,0,0,0,0,0,1)))`
    matches `dqset.seed(42L, c(0L, 1L))` bit-for-bit (compare via
    `dqrunif(10)`).
  - Same for `seed`: `dqset.seed(as.raw(c(0,0,0,0,0,0,0,42)))` matches
    `dqset.seed(42L)`.
  - Length-4 raw equals 32-bit integer scalar.
  - Wrong-length raw (e.g. `raw(3)`, `raw(9)`) errors with a clear
    message; type other than int/raw errors.
- Extend `test-generators.R`: parameterise the existing stream tests
  over `stream_repr \in {integer, raw}` (helper that picks the
  representation).
- Extend `tests/testthat/cpp/convert.cpp`: add `convert_64_raw`
  exports and an `is_raw_consistent` predicate analogous to
  `is_signed_consistent`.
- `test-seed-format-option.R` (new):
  - With `withr::local_options(dqrng.seed_format = "raw")`,
    `generateSeedVectors(5, 2)` returns a list of `raw(8)`.
  - Per-call `format = "integer"` overrides the option.
  - Round-trip: each raw element, fed back into `dqset.seed(stream =
    ...)`, reproduces the run obtained from the integer form.

## 5. Docs / NEWS

- `man/dqrng-functions.Rd` regenerated from updated roxygen.
- `man/generateSeedVectors.Rd` regenerated; add `@param format`.
- `NEWS.md`: under the next version, a bullet:
  - "`dqset.seed()` now accepts `raw` vectors for `seed` and `stream`
    (length 4 or 8). `generateSeedVectors()` gains a `format`
    argument; the default representation can also be set globally
    via the `dqrng.seed_format` option."
- Vignette: nothing structural, but the parallel example in the
  intro vignette would read more naturally with `raw(8)` stream IDs;
  consider a one-line aside.

## 6. Every R↔C++ multi-byte channel — touch in one swoop?

Below: every place the dqrng R API moves a value that is conceptually
more than one byte across the R/C++ boundary, with the verdict on
whether it can be `raw`-ified without breaking the C++ ABI (defined
here as: existing downstream packages keep working without
recompilation, i.e. no `validateSignature` mismatch in
`dqrng_RcppExports.h`, no virtual-method or class-layout change in
`dqrng_types.h`, no change to existing `dqrng::convert_seed`
overloads, no change to `random_64bit_generator::seed/clone/output/
input`).

| # | Channel | Direction | Type today | ABI-safe `raw` path? | In this swoop? |
|---|---|---|---|---|---|
| 1 | `dqset.seed(seed = ...)` | in | `IntegerVector` (length 1 or 2) | yes — add `dqset_seed_raw` (§3.2), R-side dispatch | **yes** |
| 2 | `dqset.seed(stream = ...)` | in | `IntegerVector` (length 1 or 2) | yes — same sibling export | **yes** |
| 3 | `generateSeedVectors()` | out | `list<IntegerVector>` of length-`nwords` | yes — not in `r,cpp` interface (R-only export), free to add `format = "raw"` | **yes** |
| 4 | `dqrng_get_state()` | out | `character` (RNG-kind string + decimal words) | no — see §6.1 | no, follow-up |
| 5 | `dqrng_set_state(state)` | in | `character` (same encoding) | no — mirror of (4) | no, follow-up |
| 6 | C++ API: `clone(uint64_t stream)` | in (C++ only) | numeric | n/a — not an R-side channel; raw input is converted in R/RcppExports glue before reaching here | stays numeric |
| 7 | C++ API: `seed(uint64_t, uint64_t)` virtuals | in (C++ only) | numeric | n/a — same | stays numeric |
| 8 | RNG output (`(*rng)()`, `dqrunif`, etc.) | out | `double` / `integer` | n/a — distribution-shaped output, not a byte channel | no |

### 6.1 Why state get/set is not ABI-safe today

The current state serialisation runs through
`random_64bit_generator::output(std::ostream&)` /
`input(std::istream&)` (`inst/include/dqrng_types.h:66-67`), and the
result is reassembled into a `character` vector by `dqrng_get_state`.
The format is RNG-specific (decimal-formatted words: 2 for
xoroshiro128, 4 for xoshiro256, mixed 128-bit values for PCG64,
arrays for threefry).

Two non-options:

1. **Convert the existing text to bytes on the R side** (e.g.
   `charToRaw(paste(state, collapse = " "))`). That gives ASCII bytes,
   not a binary representation of the state — useless for compactness
   or for tools that want fixed-width state.
2. **Add new virtuals** `to_raw()` / `from_raw()` to
   `random_64bit_generator`. Adding a virtual method shifts the vtable
   layout for any downstream subclass compiled against the old header
   — classic C++ ABI break, even if no user code calls the new
   method.

A genuinely ABI-safe path would be to *additionally* register a new
free-function export (e.g. `dqrng_get_state_raw()`) that **internally**
performs an `rng_kind`-switched downcast — `Rcpp::XPtr` ->
`random_64bit_wrapper<RNG>*` for each known RNG — and reads its
internal state via the concrete type. This works without touching
`dqrng_types.h`, but it bakes the set of recognised RNGs into the
C++ dispatch, and the round-trip semantics need careful design
(versioned format header, length-per-RNG table). It is a meaningful
amount of work; not in scope here.

### 6.2 Things explicitly out of scope

- Lengths > 8 / multi-word stream IDs. Today the abstract API
  flattens both `seed` and `stream` to `uint64_t`. Supporting
  `raw(16)` for PCG64's full 128-bit stream selector, or `raw(32)`
  for xoshiro256's 256-bit jump state, would require widening the
  virtual `seed`/`clone` signatures (likely to take a byte-span or a
  `std::array<uint64_t, K>`), which is an ABI break for downstream
  packages that depend on `dqrng_types.h`. Track separately.
- Endianness flag. We commit to MSB-first (matches the current
  integer convention and `convert_seed_internal`). If LSB-first is
  ever wanted, add `endian = c("big", "little")` then.
- Raw form of `dqrng_get_state()` / `dqrng_set_state()`. See §6.1.
- Returning the auto-generated seed from `dqset.seed(NULL)` so it
  can be recorded for replay. Tempting (and the `raw(8)` format
  would be the natural carrier), but it's an independent feature —
  log it as a separate issue.

## 7. Work breakdown

1. C++: add `RawVector` overloads in `convert_seed.h` + inline
   `as_uint64_seed(SEXP)` helper. Tests in
   `tests/testthat/cpp/convert.cpp`. (Pure addition — `dqrng_RcppExports.h`
   unaffected.)
2. C++: add `dqset_seed_raw(Nullable<RawVector>, Nullable<RawVector>)`
   alongside (not replacing) `dqset_seed`. Regenerate
   `RcppExports.{R,cpp}` and verify the existing `dqset_seed`
   signature string in `dqrng_RcppExports.h` is byte-for-byte
   unchanged.
3. R: `dqset.seed()` gains type-dispatch (integer path → `dqset_seed`,
   raw path → `dqset_seed_raw`, mixed → normalise to raw). Update
   roxygen and examples. Add raw-input tests.
4. C++/R: parameterise `generateSeedVectors()` with `format`; add R
   wrapper that reads `dqrng.seed_format`. (Safe to change signature
   — not in `r,cpp` interface.) Tests for both formats and round-trip.
5. Docs: NEWS.md entry calling out the additive nature (no rebuild
   required for downstream packages); man pages regenerated via
   `devtools::document()`; `air` format pass.
6. Optional: tiny vignette tweak demonstrating `raw(8)` streams in
   the `clusterApply` example.
