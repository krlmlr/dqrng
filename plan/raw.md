# Consistent `raw`-based seed / stream / state API

Status: design draft, not started.
Replaces the earlier "add raw input" sketch.
Two parts: (A) R-level API cleanup, (B) C++ API/ABI redesign from
scratch. (A) is value on its own and can ship without (B). (B) makes
(A) cheap to implement and removes the conventions that make the
current code painful.

## 1. Problem

dqrng's R API moves "key material" — seeds, stream selectors, RNG
state, generated seed pools — across four channels, and each channel
uses a *different* R shape for what is conceptually a fixed-width
byte string:

| Conceptual value | R shape today | How bits map |
|---|---|---|
| 64-bit seed / stream | `integer(1)` *or* `integer(2)` | bit-packed, MSB-first, 32 bits/word |
| RNG state | `character` vector | first element = kind tag, rest = decimal-encoded 64-bit (or 128-bit, for PCG64) words, space-split by `dqrng_get_state` |
| Generated seed pool | `list<integer(nwords)>` | same bit-packing as seed, `nwords` chosen by caller |
| Auto-generated seed (when `dqset.seed(NULL)`) | none — pulled from R's RNG and silently discarded | n/a |

The result:

- **Same value, two shapes.** `dqset.seed(42L)` and
  `dqset.seed(c(0L, 42L))` are the same uint64; the caller has to
  remember the MSB-first 32-bit-per-word convention to round-trip a
  64-bit value through `integer(2)`.
- **State is a string.** `dqrng_get_state()` returns
  `c("xoroshiro128++", "12345", "67890")` — a tag plus
  decimal-encoded internal state, RNG-kind dependent length, parsing
  done with `std::stringstream` (`src/dqrng.cpp:84-100`). Anyone who
  wants to hash, store, or transport state has to defensively
  preserve a `character` vector.
- **Seed pool comes as ints.** `generateSeedVectors()` returns
  `list<integer(nwords)>` and the user has to keep the same
  packing convention straight on the way back into `dqset.seed()`.
- **Auto-seed disappears.** When `seed = NULL`, the seed pulled from
  R's RNG is consumed inside `dqset_seed` and never surfaced.
  Replay requires the user to draw their own seed.

Adding "accept `raw` too" everywhere makes the surface bigger, not
cleaner. The fix is to pick one canonical shape and use it
everywhere — and to give downstream a single byte-oriented C++ API
that the R glue is a thin wrapper over.

## 2. Two canonical shapes

The R API should carry key material in exactly two shapes:

1. **`raw(N)`** — a self-contained, fixed-width binary value. Used
   for seeds, stream selectors, and generated seed pools.
2. **`list(kind = character(1), state = raw(N))`** with S3 class
   `"dqrng_state"` — used wherever a binary value carries a tag
   identifying *which* RNG produced it.

Nothing else. No bit-packed integer vectors, no decimal-encoded
character vectors, no `nwords` arithmetic.

## 3. Byte / bit layout (MSB-first)

A `raw(N)` carries an `N*8`-bit unsigned value. Byte 1 is the most
significant byte; byte `N` is the least significant. This matches
`writeBin(..., endian = "big")` and matches the existing integer-side
convention in `dqrng::convert_seed_internal`
(`inst/include/convert_seed.h`).

Worked example — the value `0x0011223344556677`:

```
R raw(8):

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

Packed uint64_t:

  bit position:   63                                              0
                  ▼                                               ▼
                  0x  00   11   22   33   44   55   66   77
                       ↑                                       ↑
                       most significant                        least significant

Legacy integer view (kept only for the deprecation window):

  stream <- c(0x00112233L, 0x44556677L)
                  │             │
                  └─ stream[1]  └─ stream[2]
                     high 32       low 32
```

Cross-check:

```r
writeBin(0x0011223344556677, raw(), size = 8, endian = "big")
#> 00 11 22 33 44 55 66 77   ← matches raw(8) above
```

Bytes go in in array order; no host-endianness dependency. We commit
to big-endian; little-endian is explicitly not supported.

## 4. Target R API

### 4.1 `dqset.seed()`

```r
dqset.seed(seed, stream = NULL)
```

- `seed`: `raw(8)`, `raw(4)`, `NULL` (auto-seed from R's RNG), or a
  `"dqrng_state"` object (in which case `stream` must be `NULL`;
  `kind` switches the active RNG and `state` is loaded verbatim).
- `stream`: `raw(8)`, `raw(4)`, or `NULL`.

Length 4 and length 8 are accepted; both pack MSB-first into a
`uint64_t` (length 4 leaves the high 32 bits zero).

Integer input remains accepted for one release with a `lifecycle`
deprecation warning that points at `as.raw()` / `writeBin()`.

Return value: invisibly, the seed actually used as `raw(8)`. Fixes
the "auto-seed disappears" problem — `s <- dqset.seed(NULL)` lets
you record the run and replay it.

### 4.2 `dqrng_get_state()` / `dqrng_set_state()`

```r
dqrng_get_state()
#> <dqrng_state: xoroshiro128++>
#>   kind:  "xoroshiro128++"
#>   state: raw[16] 7f a3 ...

dqrng_set_state(state)
```

Returns / accepts an S3 object:

```r
structure(
  list(kind = "xoroshiro128++", state = raw(16)),
  class = "dqrng_state"
)
```

`state` is a `raw` vector of RNG-kind-specific length:

| Kind | `state` length |
|---|---|
| `xoroshiro128+`, `xoroshiro128++`, `xoroshiro128**` | 16 (2 × 64 bits) |
| `xoshiro256+`, `xoshiro256++`, `xoshiro256**` | 32 (4 × 64 bits) |
| `pcg64` | 32 (state + stream, both 128-bit) |
| `threefry` (sitmo 20-round 64-bit) | 64 (key 256 + ctr 256) |

Plus a one-byte cache flag + four-byte cached value for the 32-bit
half of the bit-cache in `random_64bit_generator`
(`inst/include/dqrng_types.h:48-55`) — `state_bytes` advertises this
per RNG (see §5.3).

`print.dqrng_state`, `format.dqrng_state`, and a stable
`identical()`-friendly representation come for free with the S3
class.

### 4.3 Seed pool generation

```r
dqrng_keys(n = 1L, bytes = 8L)
```

Returns a `list` of `raw(bytes)` drawn from R's RNG (same source as
`generateSeedVectors()`, no state alteration via `RNGScope`).

A single key (`n = 1`) returns `list(raw(bytes))` for consistent
unboxing; or accept a `simplify = FALSE` default with a documented
`simplify = TRUE` shortcut that returns the bare `raw(bytes)`.

`generateSeedVectors(nseeds, nwords)` becomes a thin deprecated
wrapper:

```r
generateSeedVectors <- function(nseeds, nwords = 2L) {
  lifecycle::deprecate_warn("0.6.0", "generateSeedVectors()",
                            "dqrng_keys()")
  lapply(dqrng_keys(nseeds, bytes = 4L * nwords),
         raw_to_int_vector)  # exact round-trip of the old output
}
```

### 4.4 What disappears (deprecated, then removed)

- Integer input to `dqset.seed(seed = ..., stream = ...)`.
- `character`-vector form of `dqrng_get_state()`/`dqrng_set_state()`.
- `generateSeedVectors()` (replaced by `dqrng_keys()`).
- The `nwords` parameter (replaced by `bytes`).

All of the above remain accepted for one minor release (e.g. `0.5.x`)
behind `lifecycle::deprecate_warn`, then are removed at `0.6.0`.

### 4.5 Convenience helpers

```r
as_dqrng_key(x)        # raw | integer | hex-character → raw(N)
format(<dqrng_state>)  # hex-encoded one-liner suitable for logs
```

Both are pure R, no C++ work.

## 5. Target C++ API/ABI — greenfield

The current C++ surface bakes the integer-packing convention into
the public ABI (`convert_seed.h`), the state serde into text
(`random_64bit_generator::output/input`), and the stream/seed
arguments into `uint64_t` (`dqrng_types.h`). All three pin us to
the existing R shapes. A clean redesign:

### 5.1 `dqrng::byte_span`

```cpp
// inst/include/dqrng_types.h (new)
namespace dqrng {
struct byte_span {
  const std::uint8_t* data;
  std::size_t size;
};
struct byte_buffer {              // owning, for return values
  std::vector<std::uint8_t> bytes;
  operator byte_span() const { return {bytes.data(), bytes.size()}; }
};
}
```

C++17, no `std::span` dependency. (Switch to `std::span` when the
package's minimum C++ standard moves.)

### 5.2 `dqrng::rng_base` — abstract base

```cpp
namespace dqrng {
class rng_base {
public:
  using result_type = std::uint64_t;

  virtual ~rng_base() = default;

  // metadata
  virtual std::string_view kind() const = 0;
  virtual std::size_t state_bytes() const = 0;
  virtual std::size_t seed_bytes() const = 0;     // minimum useful seed width
  virtual std::size_t stream_bytes() const = 0;   // 0 if no streams

  // generation
  virtual result_type next() = 0;
  virtual std::uint32_t next_bounded(std::uint32_t) = 0;
  virtual std::uint64_t next_bounded(std::uint64_t) = 0;

  // seeding (all byte-oriented; shorter spans are zero-padded on the
  // high side, longer spans are rejected)
  virtual void seed(byte_span seed) = 0;
  virtual void seed(byte_span seed, byte_span stream) = 0;
  virtual std::unique_ptr<rng_base> clone(byte_span stream) const = 0;

  // state serde: caller-allocated, length state_bytes()
  virtual void save(std::uint8_t* out) const = 0;
  virtual void load(const std::uint8_t* in) = 0;

  // convenience non-virtuals
  byte_buffer save() const;                       // allocates state_bytes()
  void seed(std::uint64_t s);                     // adapter to byte_span
  void seed(std::uint64_t s, std::uint64_t st);
};
}
```

Drop the `uniform01`, `variate<>`, `generate<>` helpers from the
*virtual* surface — keep them as non-virtual templates that call
`next()`. The current `random_64bit_generator` mixes the
distribution-helper API into the abstract base
(`inst/include/dqrng_types.h:80-249`); splitting them keeps the
virtual contract small and the vtable shallow.

Drop the `cache`/`has_cache` members from the base — those are
implementation details of `random_64bit_wrapper`. Move them down.

Drop `operator<<` / `operator>>`. State I/O is `save`/`load` over
bytes.

### 5.3 Per-RNG concrete wrappers

```cpp
template<typename RNG>
class rng_impl : public rng_base {
  RNG gen;
  std::uint8_t cache_valid{0};
  std::uint32_t cache{0};

public:
  static constexpr std::string_view kind_v = ...;     // per specialisation
  std::string_view kind() const override { return kind_v; }
  std::size_t state_bytes() const override {
    return sizeof(RNG::state) + 1 /*cache_valid*/ + 4 /*cache*/;
  }
  void save(std::uint8_t* out) const override { ... };   // RNG-specific
  void load(const std::uint8_t* in) override { ... };
  ...
};
```

Each RNG gets a *byte-exact* serialisation; the layout is documented
per RNG (table in §4.2) and is the same on every platform (the words
are written big-endian regardless of host endianness — single
serialiser does the byte swap).

PCG64 stream semantics: drop the "relative to current stream"
behaviour (`inst/include/dqrng_generator.h:104-117`) — make
`seed(seed, stream)` set the absolute stream like every other RNG.
Same arg name should mean the same thing. The current behaviour
becomes a separate `nudge_stream(byte_span)` method on `rng_impl<pcg64>`
for callers who depended on it.

### 5.4 Rcpp adapter layer — `dqrng_rcpp.h` (new)

`dqrng_types.h` and `convert_seed.h` lose all their `Rcpp::...`
dependencies. A new header `inst/include/dqrng_rcpp.h` holds the
glue:

```cpp
namespace dqrng { namespace rcpp {

inline byte_span as_byte_span(SEXP x);     // RAWSXP only; throws otherwise
inline byte_buffer as_byte_buffer(SEXP x); // same, owning copy
inline Rcpp::RawVector to_raw(byte_span);
inline Rcpp::RawVector to_raw(std::uint64_t);

// dqrng_state list <-> { kind, byte_span }
struct state_view { std::string_view kind; byte_span state; };
inline state_view as_state(Rcpp::List);
inline Rcpp::List  to_state(std::string_view kind, byte_span state);

}}
```

`src/dqrng.cpp` becomes a thin Rcpp shell:

```cpp
// [[Rcpp::export(rng = false)]]
Rcpp::RawVector dqset_seed(Rcpp::Nullable<Rcpp::RawVector> seed,
                           Rcpp::Nullable<Rcpp::RawVector> stream) {
  // returns the seed actually used
  ...
}

// [[Rcpp::export(rng = false)]]
Rcpp::List dqrng_get_state() {
  return dqrng::rcpp::to_state(rng->kind(), rng->save());
}

// [[Rcpp::export(rng = false)]]
void dqrng_set_state(Rcpp::List state) {
  auto v = dqrng::rcpp::as_state(state);
  dqRNGkind_impl(std::string(v.kind));
  rng->load(v.state.data);
}
```

### 5.5 What disappears from the C++ API

- `dqrng::convert_seed<T>(...)` — replaced by `byte_span` everywhere.
  Header-only `convert_seed.h` is kept as a deprecation shim for one
  release; it `#warning`s on include and forwards to the new path.
- `random_64bit_generator::output(std::ostream&)` and `input`.
- `operator<<` / `operator>>` on the base.
- The `cache`/`has_cache` members on the base (moved to `rng_impl`).
- `dqrng::rng64_t` typedef (kept as alias to the new `rng_ptr` for
  one release).
- `random_64bit_accessor`'s text I/O (moved to byte I/O).

### 5.6 What stays

- The set of supported RNGs and their statistical properties.
- The `next()` contract (`uint64_t`, full range, period as before).
- `dqrng.h` as the public top-level include (its contents are
  rewritten to forward to the new headers).

## 6. Migration plan

**Phase 1 — Additive (no C++ ABI break).** Ship in `0.5.0`:

- `RawVector` overload of `dqrng::convert_seed<T>` (free function;
  pure addition to `convert_seed.h`).
- New `dqset_seed_raw` Rcpp export alongside the existing
  `dqset_seed`, so the `validateSignature` string in
  `inst/include/dqrng_RcppExports.h` for `dqset_seed` is unchanged.
  R-side `dqset.seed()` dispatches on `is.raw(...)`.
- `dqrng_keys(n, bytes)` new export; `generateSeedVectors()`
  unchanged.
- `dqrng_get_state_raw()` / `dqrng_set_state_raw()` new exports.
  Their internal implementation switches on `rng_kind` and downcasts
  the `Rcpp::XPtr<random_64bit_generator>` to the concrete
  `random_64bit_wrapper<RNG>*` to read/write the bytes — no new
  virtuals on the base.
- `dqrng_state` S3 class lives entirely in R.
- All new functions documented; old functions get a "see also"
  pointer but no deprecation warning yet.

After Phase 1 the user-visible API has the consistent shapes from
§4; the old shapes are still accepted; no downstream package needs
a rebuild.

**Phase 2 — ABI break (`0.6.0`, major bump).** Implements §5 in
full:

- New `dqrng::rng_base` replaces `random_64bit_generator`.
- New `dqrng_rcpp.h`, byte-oriented `save`/`load`.
- Old `random_64bit_generator`, `convert_seed.h`,
  `generateSeedVectors`, integer input to `dqset.seed`, and
  character form of `dqrng_get_state`/`set_state` are
  removed (or kept as `#warning`-emitting shims that forward).
- Downstream packages must rebuild and update against the new
  abstract base. The migration is mechanical:
  - `convert_seed<uint64_t>(ivec)` → seed directly from a
    `byte_span` (or use `dqrng::rcpp::as_byte_span`).
  - `gen->seed(s, st)` (numeric) → either still works via the
    non-virtual `seed(uint64_t,uint64_t)` adapter, or switch to
    the byte-span virtual.
  - `gen->output(os)` → `gen->save(buf)` then handle bytes.
- `NEWS.md` includes a porting checklist with `before` / `after`
  snippets for each removed symbol.

**Phase 2 prerequisites.** Reverse-dependency check (`revdepcheck::`)
across CRAN reverse imports of `dqrng` to size the migration impact;
publish a porting guide before tagging `0.6.0`.

## 7. Work breakdown

### Phase 1 (single PR, no ABI break)

1. `RawVector` overload + `as_uint64_seed(SEXP)` helper in
   `inst/include/convert_seed.h`. Tests in
   `tests/testthat/cpp/convert.cpp`.
2. `dqset_seed_raw` Rcpp export in `src/dqrng.cpp`. Verify the
   pre-existing `dqset_seed` signature string in
   `inst/include/dqrng_RcppExports.h` is byte-identical after
   `Rcpp::compileAttributes()`.
3. R `dqset.seed()` dispatches on input type; returns the seed used
   invisibly as `raw(8)`.
4. `dqrng_keys(n, bytes)` Rcpp export; `dqrng_state` S3 class +
   `print.dqrng_state` in R.
5. `dqrng_get_state_raw()` / `dqrng_set_state_raw()` Rcpp exports
   with kind-switched downcast. Per-RNG byte serialisation
   documented in `inst/include/dqrng_serde.h` (new header,
   header-only).
6. R wrappers: `dqrng_get_state()` and `dqrng_set_state()` are
   re-exposed as raw-state-returning functions; the old
   character-vector entry points are renamed
   `dqrng_get_state_text()` / `dqrng_set_state_text()` and kept
   exported for one release.
7. NEWS.md, man pages, `air` format pass.

### Phase 2 (major version)

8. Introduce `dqrng::byte_span` / `byte_buffer` in
   `inst/include/dqrng_types.h`.
9. Add `dqrng::rng_base` alongside the old
   `random_64bit_generator`. Implement `rng_impl<RNG>` for all
   supported RNGs with byte-exact `save`/`load` and dispatch through
   `byte_span` everywhere.
10. New `inst/include/dqrng_rcpp.h`; `src/dqrng.cpp` rewritten to
    use it.
11. Deprecate `convert_seed.h`, `random_64bit_generator`,
    `dqrng::rng64_t`, `generateSeedVectors`, integer input to
    `dqset.seed`, character-form state. Each emits a
    `lifecycle::deprecate_warn` / `#warning` on use.
12. Reverse-dependency check; porting guide; `0.6.0` release.

## 8. Tests

In `tests/testthat/`:

- `test-raw-roundtrip.R`: every accepted shape of `seed` / `stream`
  produces the same `dqrunif(100)` output as the canonical `raw(8)`
  form. Lengths 4 and 8 covered; bad lengths error.
- `test-state-roundtrip.R`: for every supported RNG kind, draw
  → `dqrng_get_state()` → consume → `dqrng_set_state(saved)` →
  draw matches. Compare against the text-form round-trip for
  byte parity during the deprecation window.
- `test-dqrng-keys.R`: `dqrng_keys(n, bytes)` returns a list of
  `raw(bytes)` of length `n`, `bytes ∈ {4, 8, 16, 32}` smoke-tested;
  using a key from `dqrng_keys()` as `stream` produces the
  documented behaviour.
- `test-deprecation.R`: each deprecated entry point emits the
  expected `lifecycle` warning under
  `withr::local_options(lifecycle_verbosity = "warning")`.
- C++ tests in `tests/testthat/cpp/convert.cpp`: parity between
  the integer and raw paths of `convert_seed` (Phase 1) and the
  `byte_span` API (Phase 2).
- Generator-suite tests (`tests/testthat/test-generators.R`)
  parameterised over the canonical raw form.

## 9. Out of scope

- LSB-first / `endian = "little"` flag. We commit to MSB-first.
- Multi-byte stream IDs > 8 bytes (PCG64's 128-bit stream selector,
  xoshiro256's 256-bit jump). Phase 2's `byte_span` API makes this
  cheap to add later — track as a Phase 3 follow-up.
- Replacing R's RNG entirely (`register_methods()` already exists
  for that and is orthogonal).
- A binary on-disk state file format. The `raw(N)` state from
  `dqrng_get_state()` is already a byte string; users can write it
  with `saveRDS()` or `writeBin()`.

## 10. Open questions

- `dqrng_keys()` vs `dqrng_seeds()` vs `dqrng_generate()` — pick one
  name. `dqrng_keys()` is short and reads well in
  `lapply(dqrng_keys(8), ...)`. Decide before Phase 1 ships.
- Should `dqset.seed(NULL)` return the seed used visibly or
  invisibly? Visibly is more discoverable; invisibly is closer to
  `base::set.seed()`. Lean invisible + document.
- The "cache" half of `random_64bit_generator` (one valid-bit + 32
  cached bits, `dqrng_types.h:48-55`) is part of the user-visible
  RNG output stream and must be in `state`. Confirm we capture this
  correctly in Phase 1's downcast-based serde before Phase 2 inherits
  the implementation.
