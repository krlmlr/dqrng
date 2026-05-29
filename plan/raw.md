# `dqrng2`: a clean, experimental sister package

Status: design draft, not started.
Replaces the in-place redesign sketches.
Outcome: a new R package `dqrng2`, parallel to `dqrng`, with a clean R
and C++ interface from day one. Marked experimental.
Changes are ported back to `dqrng` step by step, at the maintainer's
pace, one mergeable PR per idea.

## 1. Why a separate package

Three things make in-place evolution of `dqrng` painful:

1. **C++ ABI surface.** `src/dqrng.cpp` is `// [[Rcpp::interfaces(r,
   cpp)]]`, so every exported function's signature is baked into
   `inst/include/dqrng_RcppExports.h` and `validateSignature`-checked
   inside compiled downstream packages. Any rename or signature
   change breaks them at load time. The reverse-dependency surface
   on CRAN is non-trivial.
2. **Public C++ types.** `random_64bit_generator`,
   `dqrng::convert_seed<T>`, `dqrng::rng64_t` are baked into
   downstream code. Widening or replacing them is an ABI break for
   every subclass, every includer.
3. **Maintainer bandwidth.** The current maintainer is upstream of
   us. Forcing a big-bang rewrite onto their queue is rude and slow.

A separate package, `dqrng2`, sidesteps all three:

- No `validateSignature` constraints — nobody links against `dqrng2`
  yet.
- Free to pick whatever C++ types make sense.
- Each clean idea becomes a small, self-contained PR for `dqrng`
  that the maintainer can take or leave. `dqrng2` is the proving
  ground; `dqrng` adopts only what survives.

`dqrng2` is **not** a fork. It does not aim to replace `dqrng`. It is
a sandbox for the API we wish `dqrng` had, plus a porting source.

## 2. Scope

`dqrng2` ships with:

- The same set of RNGs as `dqrng`: xoroshiro128+/++/\*\*,
  xoshiro256+/++/\*\*, PCG64, Threefry-20-64.
- The same set of distributions: uniform, normal (Ziggurat),
  exponential (Ziggurat), Rademacher.
- Sampling: weighted/unweighted with/without replacement.
- A consistent R API built on two canonical shapes (§4).
- A clean C++ API in namespace `dqrng2` (§7), header-only enough
  for `LinkingTo: dqrng2` from day one.

`dqrng2` does **not**:

- Depend on `dqrng` (`LinkingTo` or `Imports`). It vendors the same
  upstream headers (`xoshiro.h`, `pcg_*`, the sitmo threefry impl,
  `boost.random` Ziggurat) so it can iterate freely.
- Replace R's RNG via `register_methods()` — out of scope for the
  experimental phase.
- Promise CRAN stability. Versioning starts at `0.0.1`; minor
  releases may break API freely.

## 3. Two canonical shapes for "key material"

The whole R surface uses exactly two shapes for anything that is
conceptually a fixed-width byte string:

1. **`raw(N)`** — a self-contained binary value (seed, stream
   selector, generated key).
2. **`list(kind = character(1), state = raw(N))`** with S3 class
   `"dqrng2_state"` — used wherever a binary value carries a tag
   identifying which RNG produced it.

Nothing else. No bit-packed integer vectors, no decimal-encoded
character vectors, no `nwords` arithmetic.

## 4. Byte / bit layout (MSB-first)

A `raw(N)` carries an `N*8`-bit unsigned value. Byte 1 is the most
significant byte; byte `N` is the least significant. Matches
`writeBin(..., endian = "big")`.

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

Packed uint64_t (what the RNG sees internally):

  bit position:   63                                              0
                  ▼                                               ▼
                  0x  00   11   22   33   44   55   66   77
                       ↑                                       ↑
                       most significant                        least significant
```

Cross-check:

```r
writeBin(0x0011223344556677, raw(), size = 8, endian = "big")
#> 00 11 22 33 44 55 66 77   ← matches raw(8) above
```

Bytes go in in array order; no host-endianness dependency. We commit
to big-endian only.

## 5. Package layout

```
dqrng2/
├── DESCRIPTION              # Package: dqrng2; LinkingTo: Rcpp, BH;
│                            #   no dependence on dqrng
├── NAMESPACE                # exports dqrng2_*
├── R/
│   ├── dqrng2-package.R     # roxygen for the package
│   ├── set_seed.R           # dqrng2_set_seed()
│   ├── state.R              # dqrng2_get_state(), dqrng2_set_state()
│   ├── keys.R               # dqrng2_keys()
│   ├── kind.R               # dqrng2_kind(), dqrng2_kinds()
│   ├── distributions.R      # dqrng2_runif/rnorm/rexp/rademacher
│   ├── sample.R             # dqrng2_sample, dqrng2_sample_int
│   ├── state-class.R        # S3: dqrng2_state, print, format, ==
│   └── zzz.R                # .onLoad: seed from R's RNG
├── src/
│   ├── dqrng2.cpp           # Rcpp glue, [[Rcpp::interfaces(r, cpp)]]
│   └── Makevars[.win]       # if needed
├── inst/include/
│   ├── dqrng2.h             # umbrella include
│   ├── dqrng2_types.h       # byte_span, byte_buffer, rng_base
│   ├── dqrng2_rcpp.h        # Rcpp adapters (kept separate from types)
│   ├── dqrng2_serde.h       # per-RNG byte layout (header-only)
│   ├── dqrng2_distribution.h
│   ├── dqrng2_sample.h
│   ├── dqrng2_generator.h   # rng_impl<RNG> wrappers
│   ├── dqrng2_RcppExports.h # generated
│   └── upstream/            # vendored
│       ├── xoshiro.h
│       ├── pcg_random.hpp
│       ├── pcg_extras.hpp
│       ├── pcg_uint128.hpp
│       └── threefry.h       # extracted from sitmo, or LinkingTo: sitmo
├── tests/testthat/...
└── plan/                    # this design + porting notes,
                             # added to .Rbuildignore
```

`dqrng2_types.h` has **no `Rcpp` dependency**. That is the single
biggest leverage over the existing layout: downstream consumers can
use `dqrng2::rng_base` without dragging Rcpp into their public
headers if they don't want to.

## 6. R API

Public function naming: prefix everything with `dqrng2_`. See §13 for
the open question on whether to drop the prefix.

### 6.1 Seeding

```r
dqrng2_set_seed(seed = NULL, stream = NULL)
```

- `seed`: one of
  - `NULL` — draw a fresh seed from R's RNG (no state alteration via
    `RNGScope`),
  - `raw(8)` — 64-bit seed, MSB-first,
  - `raw(4)` — 32-bit seed, zero-padded into the low half of the
    64-bit value,
  - a `dqrng2_state` object — sets RNG kind + state in one call;
    `stream` must be `NULL` in this case.
- `stream`: `NULL`, `raw(8)`, or `raw(4)`.

Returns invisibly the seed actually used, as `raw(8)`. So
`saved <- dqrng2_set_seed(NULL)` lets you replay later via
`dqrng2_set_seed(saved)`.

Wrong-length or wrong-type inputs error with a clear message. No
silent coercion from integer.

### 6.2 State

```r
state <- dqrng2_get_state()
# <dqrng2_state>
#   kind:   "xoroshiro128++"
#   state:  raw[21]  7f a3 ... 00

dqrng2_set_state(state)
```

`state` is an S3 list:

```r
structure(
  list(kind = "xoroshiro128++", state = raw(21)),
  class = "dqrng2_state"
)
```

`state` is a `raw` vector of RNG-kind-specific length. The
`dqrng2_serde.h` header (§7.4) documents the byte layout per RNG:
the engine's words, big-endian, followed by a 1-byte cache-valid
flag and a 4-byte cached 32-bit value (`random_64bit_generator`'s
bit cache must round-trip — see §13).

Approximate sizes:

| Kind                       | `state` length |
|----------------------------|----------------|
| `xoroshiro128+/++/**`      | 16 + 5 = 21    |
| `xoshiro256+/++/**`        | 32 + 5 = 37    |
| `pcg64`                    | 32 + 5 = 37    |
| `threefry` (20-round 64)   | 64 + 5 = 69    |

S3 methods: `print`, `format`, `==`, `as.raw` (returns
`c(charToRaw(kind), as.raw(0), state)` for serialisation as a
single blob).

### 6.3 Key generation

```r
keys <- dqrng2_keys(n = 1L, bytes = 8L)
# list of length n, each element raw(bytes)
```

`bytes` accepts any positive integer; common values 4, 8, 16, 32.
Replaces `dqrng::generateSeedVectors()`.

Draws from R's RNG via `RNGScope`, no state alteration.

For ergonomics, `n = 1L` still returns a length-1 list (so callers
can rely on `lapply(keys, ...)` regardless of `n`); add a
`simplify = FALSE` default with a documented shortcut to unbox.

### 6.4 RNG kind

```r
dqrng2_kind()                       # returns current "kind" string
dqrng2_kind(kind = "pcg64")         # sets it
dqrng2_kinds()                      # returns character vector of valid kinds
```

Replaces `dqRNGkind`. Kind names are a fixed enumeration (`dqrng2_kinds()`);
arbitrary lowercased strings are no longer accepted.

### 6.5 Distributions and sampling

Same shape as `dqrng`, renamed for prefix consistency:

```r
dqrng2_runif(n, min = 0, max = 1)
dqrng2_rnorm(n, mean = 0, sd = 1)
dqrng2_rexp(n, rate = 1)
dqrng2_rademacher(n)
dqrng2_sample(x, size, replace = FALSE, prob = NULL)
dqrng2_sample_int(n, size, replace = FALSE, prob = NULL, offset = 0L)
```

No behavioural change vs `dqrng` — same statistical output bit-for-bit
given the same seed.

### 6.6 What dqrng2 does *not* export

- `register_methods` / `restore_methods` (out of scope).
- An integer-input path for seeds. From day one, key material is
  bytes.
- A character-vector form of state.

## 7. C++ API

All in namespace `dqrng2`. `// [[Rcpp::interfaces(r, cpp)]]` from day
one so downstream can use it after a single `LinkingTo: dqrng2`.

### 7.1 `dqrng2::byte_span` / `byte_buffer`

`inst/include/dqrng2_types.h` (no Rcpp):

```cpp
namespace dqrng2 {

struct byte_span {
  const std::uint8_t* data;
  std::size_t         size;

  byte_span() : data(nullptr), size(0) {}
  byte_span(const std::uint8_t* d, std::size_t n) : data(d), size(n) {}
  template<std::size_t N>
  byte_span(const std::uint8_t (&arr)[N]) : data(arr), size(N) {}
};

struct byte_buffer {
  std::vector<std::uint8_t> bytes;
  operator byte_span() const { return {bytes.data(), bytes.size()}; }
};

}  // namespace dqrng2
```

C++17 baseline; switch to `std::span` when the package's minimum
standard moves.

### 7.2 `dqrng2::rng_base`

```cpp
namespace dqrng2 {

class rng_base {
public:
  using result_type = std::uint64_t;

  virtual ~rng_base() = default;

  // metadata
  virtual std::string_view kind() const = 0;
  virtual std::size_t state_bytes()  const = 0;
  virtual std::size_t seed_bytes()   const = 0;  // min useful seed width
  virtual std::size_t stream_bytes() const = 0;  // 0 if no streams

  // generation
  virtual result_type   next() = 0;
  virtual std::uint32_t next_bounded(std::uint32_t range) = 0;
  virtual std::uint64_t next_bounded(std::uint64_t range) = 0;

  // seeding — byte-oriented; short spans zero-pad on the high side,
  //                          long spans throw.
  virtual void seed(byte_span seed) = 0;
  virtual void seed(byte_span seed, byte_span stream) = 0;
  virtual std::unique_ptr<rng_base> clone(byte_span stream) const = 0;

  // state serde; caller-allocated, size == state_bytes()
  virtual void save(std::uint8_t* out) const = 0;
  virtual void load(const std::uint8_t* in)  = 0;

  // ── non-virtual conveniences (no ABI risk) ──────────────────────
  byte_buffer save() const;                                  // allocates
  void seed(std::uint64_t s);                                // adapts
  void seed(std::uint64_t s, std::uint64_t st);

  double uniform01();
  // distribution helpers as non-virtual templates below

  template <typename Dist, typename... A>
  typename Dist::result_type variate(A&&...);
  template <typename Dist, typename Range, typename... A>
  void generate(Range&&, A&&...);
};

}  // namespace dqrng2
```

Notes:

- `cache` / `has_cache` (the bit-cache currently on
  `random_64bit_generator` in `inst/include/dqrng_types.h:48-55`)
  move **down** into `rng_impl`. The abstract base is byte-pure.
- `operator<<` / `operator>>` are gone. State is bytes.
- The variate/generate helpers are non-virtual templates — no vtable
  slot, no ABI cost.

### 7.3 `rng_impl<RNG>` — concrete wrappers

`inst/include/dqrng2_generator.h`:

```cpp
template<typename RNG>
class rng_impl : public rng_base {
  RNG           gen;
  std::uint8_t  cache_valid{0};
  std::uint32_t cache{0};

public:
  static constexpr std::string_view kind_v = ...;  // per specialisation
  std::string_view kind() const override { return kind_v; }
  std::size_t state_bytes()  const override { return state_layout<RNG>::bytes; }
  std::size_t seed_bytes()   const override { return 8; }
  std::size_t stream_bytes() const override { return stream_layout<RNG>::bytes; }

  result_type next() override { return gen(); }

  void seed(byte_span s) override;
  void seed(byte_span s, byte_span st) override;
  std::unique_ptr<rng_base> clone(byte_span st) const override;

  void save(std::uint8_t* out) const override;   // dqrng2_serde.h does the work
  void load(const std::uint8_t* in)  override;
};
```

Per-RNG specialisations live in `dqrng2_serde.h` and are the only
RNG-specific code outside the upstream headers themselves.

PCG64 stream semantics: drop the "relative to current stream"
behaviour (which `dqrng` does at
`inst/include/dqrng_generator.h:104-117`). In `dqrng2`,
`seed(seed, stream)` sets the absolute stream like every other RNG.
The old behaviour, if anyone needs it, becomes a separate
`nudge_stream(byte_span)` method on `rng_impl<pcg64>`. Same arg name
should mean the same thing.

### 7.4 `dqrng2_serde.h` — byte layouts

One header documents and implements, per RNG, the on-disk-equivalent
byte layout:

```cpp
namespace dqrng2 { namespace serde {

template<typename RNG> struct layout;
// e.g.
template<> struct layout<::dqrng2::xoroshiro128plusplus> {
  static constexpr std::size_t bytes = 16;
  // big-endian write of two uint64_t words s[0], s[1]
  static void save(const ::dqrng2::xoroshiro128plusplus&, std::uint8_t*);
  static void load(::dqrng2::xoroshiro128plusplus&, const std::uint8_t*);
};
// ... same for the other RNGs

// Common 5-byte suffix for the bit cache:
inline void save_cache(std::uint8_t valid, std::uint32_t cache,
                       std::uint8_t* out);
inline void load_cache(std::uint8_t& valid, std::uint32_t& cache,
                       const std::uint8_t* in);

}}  // namespace dqrng2::serde
```

This is the entire RNG-specific surface for state serialisation, in
one file. Reviewers can audit the byte layout of every RNG by
reading a single screen of code.

### 7.5 `dqrng2_rcpp.h` — Rcpp adapters

This header is the **only** place `Rcpp::` appears in the C++ public
surface. `dqrng2_types.h` stays Rcpp-free.

```cpp
namespace dqrng2 { namespace rcpp {

byte_span    as_byte_span(SEXP x);              // RAWSXP only
byte_buffer  as_byte_buffer(SEXP x);            // RAWSXP only, owning copy
Rcpp::RawVector to_raw(byte_span);
Rcpp::RawVector to_raw(std::uint64_t);

// dqrng2_state list <-> { kind, byte_span }
struct state_view { std::string_view kind; byte_span state; };
state_view   as_state(Rcpp::List);
Rcpp::List   to_state(std::string_view kind, byte_span state);

}}  // namespace dqrng2::rcpp
```

### 7.6 `src/dqrng2.cpp` — exports

A thin Rcpp shell. Every C++ export is byte-oriented:

```cpp
// [[Rcpp::export(rng = false)]]
Rcpp::RawVector dqrng2_set_seed_impl(Rcpp::Nullable<Rcpp::RawVector> seed,
                                     Rcpp::Nullable<Rcpp::RawVector> stream);

// [[Rcpp::export(rng = false)]]
Rcpp::List dqrng2_get_state_impl();

// [[Rcpp::export(rng = false)]]
void dqrng2_set_state_impl(Rcpp::List state);

// [[Rcpp::export(rng = true)]]
Rcpp::List dqrng2_keys_impl(int n, int bytes);

// [[Rcpp::export(rng = false)]]
Rcpp::NumericVector dqrng2_runif_impl(R_xlen_t n, double min, double max);
// ... etc.
```

R wrappers in `R/` provide the public surface from §6 and do the
type checking before calling into C++.

## 8. Vendored upstream code

`dqrng` vendors:
- `external/pcg-cpp` (PCG headers),
- `external/xoroshiro128plus-cpp`, `external/xorshift-cpp` (Vigna originals),
- `inst/include/xoshiro.h` (dqrng's own wrapper around Vigna's code),
- Threefry via `LinkingTo: sitmo`.

`dqrng2` makes the same choice as `dqrng` did: vendor or `LinkingTo`
the same upstream sources, isolated under `inst/include/upstream/`.
No new statistical code; only the wrapper layer is new.

License/copyright lines preserved per upstream; `LICENSE.note`
mirrors `dqrng`'s.

## 9. Interop with dqrng (optional, useful)

While both packages can coexist (separate namespaces, separate
global RNG states), interop is a small, useful surface to ship:

```r
dqrng2_from_dqrng()   # copy dqrng's current state into dqrng2's
dqrng2_to_dqrng()     # the reverse
```

Implementation: get/set state on each side via the public R API of
each package. `dqrng`'s state is a `character` vector; `dqrng2`'s is
`dqrng2_state`. The translation table (one entry per RNG kind) is a
~50-line R function that parses the decimal words from `dqrng` and
packs them MSB-first into the `state` raw. No C++ work.

This makes the porting story concrete: a user who is mid-pipeline on
`dqrng` can hop to `dqrng2` (or vice versa) without losing
reproducibility.

## 10. Tests

In `tests/testthat/`:

- `test-set-seed.R`: every accepted shape of `seed` / `stream`
  produces the same `dqrng2_runif(100)` output as the canonical
  `raw(8)` form. Length 4 and 8; bad lengths error; bad types
  error.
- `test-state-roundtrip.R`: for every supported RNG kind:
  - draw → `dqrng2_get_state()` → consume → `dqrng2_set_state(saved)` →
    draw matches;
  - identity holds across serialisation to / from bytes (`as.raw`
    on the state, then back).
- `test-keys.R`: `dqrng2_keys(n, bytes)` returns `n` elements of
  exactly `bytes` bytes each; `n = 0` returns `list()`.
- `test-replay.R`: `s <- dqrng2_set_seed(NULL); x <- dqrng2_runif(100);
  dqrng2_set_seed(s); y <- dqrng2_runif(100); expect_identical(x, y)`.
- `test-interop.R` (if §9 lands): for every RNG kind, draw from
  `dqrng`, `dqrng2_from_dqrng()`, continue drawing, and compare
  against `dqrng`'s continuation.
- `tests/testthat/cpp/serde.cpp`: round-trip every RNG's
  `save`/`load` against a hand-encoded byte pattern. Lock the byte
  layout in.
- `tests/testthat/cpp/byte_span.cpp`: `dqrng2::byte_span` and
  `dqrng2::rcpp::as_byte_span` smoke tests.
- Generator-suite tests parameterised over all RNG kinds (mirror
  `dqrng/tests/testthat/test-generators.R`).

## 11. Porting back to `dqrng` — issue-sized chunks

`dqrng2` is the proving ground; `dqrng` adopts at its own pace. Each
of the following is a self-contained PR that does **not** require
the others to land. Each is small enough to review on a coffee
break.

| # | Idea | `dqrng` change | ABI impact |
|---|---|---|---|
| 1 | Accept `raw(4|8)` for `seed` / `stream` | New `dqset_seed_raw` Rcpp export; R-side dispatch in `dqset.seed()` | Additive |
| 2 | Return seed used from `dqset.seed(NULL)` | R-only change; returns `raw(8)` invisibly. Old code that ignores the return value is unaffected | None |
| 3 | `dqrng_keys(n, bytes)` | New Rcpp export returning `list<raw>`; old `generateSeedVectors` kept | Additive |
| 4 | `dqrng_state` S3 class for `dqrng_get_state` / `set_state` | New `dqrng_get_state_raw` / `dqrng_set_state_raw` exports via kind-switched downcast; R-level S3 class; old character form kept | Additive |
| 5 | `dqrng_kinds()` enumerator | R-only; reads from a small table | None |
| 6 | PCG64 stream semantics change | Behavioural break; needs a behind-an-option toggle or major bump | Behaviour |
| 7 | Byte-oriented `seed` / `clone` virtuals on the C++ base | Breaks `dqrng_types.h` ABI; major bump | C++ ABI |
| 8 | Drop `dqrng::convert_seed<T>` packing | After (1) and (7) land; major bump | C++ ABI |

Items 1–5 are mergeable into `dqrng` today (additive). Items 6–8 are
the bigger steps; the maintainer takes them when ready, and `dqrng2`
has been running on that design for long enough to know it works.

Each item gets a tracking issue in `daqana/dqrng` with the dqrng2
PR / commit as a reference implementation.

## 12. Roadmap

| Milestone | Contents | Rough sequence |
|---|---|---|
| `dqrng2 0.0.1` | RNGs + distributions + raw seed/stream + `dqrng2_get_state`/`set_state` + S3 class + tests; no interop yet | 1 |
| `dqrng2 0.0.2` | `dqrng2_keys`, replay (`dqrng2_set_seed(NULL)` returns seed), bounded sampling | 2 |
| `dqrng2 0.0.3` | Interop with `dqrng` (§9) | 3 |
| `dqrng → ...` | Porting PRs 1–5 from §11, in any order the maintainer accepts | 4+ |
| `dqrng2 0.1.0` | API freeze candidate after a CRAN reverse-dep equivalent (run dqrng's revdeps against `dqrng2_*` translation shims) | when stable |
| `dqrng` major | Items 6–8 from §11; coordinated with `dqrng2 0.1.0` | when maintainer ready |

`dqrng2` does not have to be retired even after `dqrng` adopts
everything — it can stay as a canary track for future experiments.

## 13. Open questions

- **Function naming.** `dqrng2_set_seed` (prefixed, discoverable
  globally) vs `set_seed` (clean inside `library(dqrng2)`, but
  collides with `base::set.seed` only in spirit, not by name). Lean
  prefixed for safety. Decide before `0.0.1`.
- **Visibility of the seed return value.** Visible would be more
  discoverable (`dqrng2_set_seed(NULL)` prints a `raw(8)`); invisible
  matches `base::set.seed`'s vibe. Lean invisible.
- **Bit-cache.** `dqrng::random_64bit_generator::cache` /
  `has_cache` is part of the user-visible output stream. Confirm
  the 5-byte suffix (1 valid flag + 4 cached bits) round-trips
  faithfully through `dqrng2_get_state` / `set_state` for every RNG.
  Hand-write a regression in `tests/testthat/cpp/serde.cpp` for the
  case "consume 32 bits, get_state, set_state, continue, compare".
- **Threefry source.** Inline the relevant headers from `sitmo` or
  keep `LinkingTo: sitmo`? `LinkingTo` is smaller surface but pins us
  to sitmo's release cadence. Lean `LinkingTo`, match `dqrng`'s
  current choice.
- **`dqrng2_kinds()` vocabulary.** Include the legacy "+" variants
  (`xoroshiro128+`, `xoshiro256+`) for parity with `dqrng`? Yes:
  parity matters more than aesthetic cleanup at this stage. Document
  the recommendation to prefer "++"/"**".
- **License.** AGPL-3 to match `dqrng`. Same copyright owners for
  the vendored RNG code.

## 14. Out of scope

- LSB-first / `endian = "little"` flag. We commit to MSB-first.
- Multi-byte stream IDs > 8 bytes (PCG64's 128-bit selector,
  xoshiro256's 256-bit jump state). `byte_span` makes this trivial
  to add later; not for `0.0.1`.
- Replacing R's RNG via `register_methods()`. Orthogonal feature.
- A wire format / on-disk file format. `as.raw(<dqrng2_state>)` is
  already a byte string; `saveRDS()` / `writeBin()` cover storage.

## 15. Repository plan

`dqrng2` lives in its own repo (`krlmlr/dqrng2` or under a neutral
org). Initial commit is the package skeleton plus this plan
(`plan/dqrng2.md`, build-ignored). First milestone (`0.0.1`) is
tagged from a single PR.

In the `dqrng` repo (this one), keep the plan at `plan/raw.md` as a
pointer: short paragraph explaining the dqrng2 approach + link to
the dqrng2 repo + the porting checklist from §11 reproduced as a
GitHub project board.
