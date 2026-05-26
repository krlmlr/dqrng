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
   (see §6).
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

`src/dqrng.cpp:36` currently uses `Rcpp::Nullable<Rcpp::IntegerVector>`
for both `seed` and `stream`. Loosen to `SEXP` (or `Rcpp::RObject`) and
dispatch on `TYPEOF`:

```cpp
// pseudocode
static uint64_t coerce_seed_like(SEXP x, const char* what) {
    if (Rf_isNull(x)) Rcpp::stop("internal: null where value required");
    switch (TYPEOF(x)) {
    case INTSXP:
        return dqrng::convert_seed<uint64_t>(Rcpp::IntegerVector(x));
    case RAWSXP:
        return dqrng::convert_seed<uint64_t>(Rcpp::RawVector(x));
    default:
        Rcpp::stop("`%s` must be integer or raw", what);
    }
}
```

Then in `dqset_seed`:

```cpp
void dqset_seed(SEXP seed, SEXP stream = R_NilValue) {
  if (Rf_isNull(seed)) {
    rng = dqrng::generator();
    return;
  }
  uint64_t _seed = coerce_seed_like(seed, "seed");
  if (Rf_isNull(stream)) {
    rng->seed(_seed);
  } else {
    rng->seed(_seed, coerce_seed_like(stream, "stream"));
  }
}
```

This keeps the existing wire signature backwards compatible (Rcpp
attribute regenerates `RcppExports.R` — verify the resulting R-side
function still accepts `NULL`, `integer`, and now `raw`).

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
`seed(uint64_t, uint64_t)` stay numeric — exporting raw vectors
through the C++ ABI would force every consumer to depend on Rcpp.
Instead, leave the C++ ABI numeric and provide a free helper for
package authors who want to accept raw vectors at their own R surface:

```cpp
namespace dqrng {
inline uint64_t as_uint64_seed(SEXP x);   // dispatch on TYPEOF
}
```

This goes in `inst/include/convert_seed.h` and is the recommended
ingestion path for downstream packages.

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

## 6. Out of scope (follow-ups)

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
- Raw output from `dqrng_get_state()`. The state is a free-form
  text serialisation today; converting it to raw is a bigger
  redesign, unrelated to stream IDs.

## 7. Work breakdown

1. C++: add `RawVector` overloads in `convert_seed.h` + `as_uint64_seed`
   helper. Tests in `tests/testthat/cpp/convert.cpp`.
2. C++: switch `dqset_seed` signature to `SEXP` with TYPEOF dispatch.
   Regenerate `RcppExports.{R,cpp}`. Existing tests must still pass.
3. R: update `dqset.seed()` roxygen; add raw-input tests.
4. C++/R: parameterise `generateSeedVectors()` with `format`; add R
   wrapper that reads `dqrng.seed_format`. Tests for both formats and
   round-trip.
5. Docs: NEWS.md entry; man pages regenerated via `devtools::document()`;
   `air` format pass.
6. Optional: tiny vignette tweak demonstrating `raw(8)` streams in the
   `clusterApply` example.
