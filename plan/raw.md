# `dqrng2`: a clean wrapper around `dqrng`

Status: design draft, not started.
Replaces earlier vendoring and in-place sketches.
Outcome: a new R package `dqrng2` that **wraps** `dqrng` 1:1, plus
fixes its API. Every public `dqrng` function has a `dqrng2`
counterpart with the clean shapes, even when the wrapper just
forwards. Shared global RNG state with `dqrng` (no separate engine).
Backporting to `dqrng` proceeds at the maintainer's pace.

## 1. Approach: wrap, don't fork

`dqrng2` is a thin wrapper:

```
[user code]
   │
   ▼
dqrng2 R API  (clean shapes — see §4)
   │
   ▼
dqrng2 Rcpp glue  (byte ↔ uint64 conversion, state parser)
   │
   ▼
dqrng C++ API + dqrng global RNG  (LinkingTo: dqrng)
```

`dqrng2` does **not** vendor RNG headers, does **not** carry its own
global RNG, and does **not** duplicate distribution code. It is one
layer of API translation, plus a few R-level conveniences.

Consequences:

- **Shared state.** `dqrng2_set_seed()` seeds *dqrng's* global RNG.
  `library(dqrng); library(dqrng2)` users see one engine, two
  surfaces. No "which set_seed did I call" confusion.
- **`register_methods()` works from day one.** Because state is
  shared, registering dqrng as R's user-supplied RNG (`set.seed()`
  → dqrng) automatically extends to dqrng2. The dqrng2 wrapper for
  `register_methods()` exists and is a thin forwarder.
- **Backporting is harder.** Wrapping demonstrates the *shape* of
  the API but not the implementation. When the maintainer adopts an
  idea into dqrng, they re-implement it in `dqrng`'s C++ rather
  than copying from `dqrng2`. We accept this cost in exchange for
  the wrap-not-fork simplicity.
- **dqrng2 is bound by dqrng's choices** where it forwards. We
  cannot fix PCG64's "stream relative to current" semantics in
  dqrng2 without touching dqrng. Document the carry-over; flag it
  in the backport list.

`dqrng2` is **experimental** and starts at `0.0.1`. Minor versions
may break the surface freely until `0.1.0`.

## 2. Scope — every dqrng public function is wrapped

The `dqrng2` public surface mirrors `dqrng`'s, function for function,
in the clean shapes. No omissions. The full table:

| dqrng | dqrng2 | Implementation |
|---|---|---|
| `dqset.seed(seed, stream)` | `dq_set_seed(seed, stream)` | Convert `raw` → `integer` per dqrng's MSB packing, call `dqrng::dqset_seed()`. Returns seed used as `raw(8)`, invisibly. |
| `dqRNGkind(kind, normal_kind)` | `dq_RNGkind(kind)` | Validate `kind` against `dq_kinds()`; forward to `dqrng::dqRNGkind()`. Drop the unused `normal_kind` arg. |
| `dqrng_get_state()` | `dq_get_state()` | Call `dqrng::dqrng_get_state()` (character vec), parse the decimal words per RNG kind, return `dq_state` S3 list. |
| `dqrng_set_state(state)` | `dq_set_state(state)` | Reverse: unpack `dq_state` → character vec → `dqrng::dqrng_set_state()`. |
| `dqrunif(n, min, max)` | `dq_runif(n, min, max)` | Forward. |
| `dqrnorm(n, mean, sd)` | `dq_rnorm(n, mean, sd)` | Forward. |
| `dqrexp(n, rate)` | `dq_rexp(n, rate)` | Forward. |
| `dqrrademacher(n)` | `dq_rademacher(n)` | Forward. |
| `dqsample(x, size, replace, prob)` | `dq_sample(x, size, replace, prob)` | Forward. |
| `dqsample.int(n, size, replace, prob)` | `dq_sample_int(n, size, replace, prob)` | Forward. |
| `dqrmvnorm(n, ...)` | `dq_rmvnorm(n, ...)` | Forward. |
| `generateSeedVectors(nseeds, nwords)` | `dq_keys(n, bytes)` | Reimplemented in dqrng2's own C++ (one loop, `RNGScope`, raw output). Same entropy source. |
| `register_methods(kind)` | `dq_register_methods(kind)` | Forward to `dqrng::register_methods()`. |
| `restore_methods()` | `dq_restore_methods()` | Forward to `dqrng::restore_methods()`. |

Plus the new constructive functions that dqrng lacks today:

| dqrng2 | Purpose |
|---|---|
| `dq_kinds()` | character vector of valid RNG kinds (fixed enumeration). |
| `dq_kind()` | return the currently active kind. |
| `dq_state()` | constructor: `dq_state(kind, state)` → `dq_state` S3. |
| `as.raw.dq_state(x)` | flatten to a single `raw` blob (kind length-prefixed). |
| `as_dq_state(raw)` | inverse. |

Naming convention is `dq_*` (short prefix, no base R clash). See
§13 for the open question on naming.

## 3. Byte / bit layout — **little-endian**

We drop the big-endian convention. Modern hardware that R is
expected to run on is uniformly little-endian:

| Platform | Endianness |
|---|---|
| x86, x86_64 (Intel, AMD) | LE |
| ARM64 / Apple Silicon (default mode) | LE |
| RISC-V (default) | LE |
| POWER on Linux (ppc64le) | LE |
| RISC-OS, embedded MIPSel | LE |
| ARM32 (default) | LE |
| AIX on POWER, z/Architecture | BE (rare; not on CRAN test farm) |

CRAN's automated test farm runs Linux x86_64, macOS arm64, and
Windows x86_64 — all little-endian. The last big-endian platform
CRAN officially supported (Solaris SPARC) was retired years ago.

So: `raw(N)` carries an `N*8`-bit unsigned value, **byte 1 is the
least significant byte**, byte `N` is the most significant. This
matches the in-memory representation of a `uint64_t` on every
common host, which means R↔C++ conversion is a `memcpy` on those
hosts and a byte-reverse on the rare BE host.

Worked example — the value `0x0011223344556677`:

```
R raw(8) (LE convention):

  stream <- as.raw(c(0x77, 0x66, 0x55, 0x44, 0x33, 0x22, 0x11, 0x00))
                     │                                          │
                     stream[1]                                  stream[8]
                     = low byte                                 = high byte

  byte index:    [1]  [2]  [3]  [4]  [5]  [6]  [7]  [8]
                 ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
  value:         │77│ │66│ │55│ │44│ │33│ │22│ │11│ │00│
                 └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘
  bit range:      7..0 15.8  23.16 31.24 39.32 47.40 55.48 63.56
                  ↑ LSB                                    ↑ MSB

Packed uint64_t:

  bit position:   63                                              0
                  ▼                                               ▼
                  0x  00   11   22   33   44   55   66   77
                       ↑                                       ↑
                       most significant                        least significant

Hex literal vs raw vector — note the visual reversal:

       hex literal:   0x  00 11 22 33 44 55 66 77    (reads MSB → LSB)
       raw(8):           77 66 55 44 33 22 11 00     (reads LSB → MSB)
       in-memory bytes
       on LE host:       77 66 55 44 33 22 11 00     ← same as raw(8)
```

The visual reversal between hex literals and the raw vector is the
price of admission for LE — flagged once, never papered over.

Cross-check against R built-ins:

```r
writeBin(0x0011223344556677, raw(), size = 8, endian = "little")
#> 77 66 55 44 33 22 11 00   ← matches dq_set_seed(stream = ...)

# Default for writeBin() on x86_64 hosts is "little" already.
identical(writeBin(x, raw(), size = 8),
          writeBin(x, raw(), size = 8, endian = "little"))
#> [1] TRUE
```

Sequences (key pools) that round-trip via `writeBin(..., endian =
"big")` will look byte-reversed in `raw(8)` form. Document this with
a single worked example in `?dq_set_seed`; users who don't reach
for raw representations don't notice.

## 4. Two canonical R shapes

Everywhere in the dqrng2 R surface, key material is one of:

1. **`raw(N)`** — a self-contained little-endian binary value. Used
   for seeds, stream selectors, and generated keys.
2. **`list(kind = character(1), state = raw(N))`** with S3 class
   `"dq_state"` — used wherever a binary value carries a tag
   identifying which RNG produced it.

Nothing else. No integer packing, no decimal strings, no `nwords`.

## 5. R API

Naming: `dq_*` for everything. Snake_case. No clash with base R.

### 5.1 Seeding

```r
dq_set_seed(seed = NULL, stream = NULL)
```

Inputs:
- `seed`: `NULL` (auto from R's RNG, no `RNGScope` alteration),
  `raw(8)`, `raw(4)`, or a `dq_state` object.
- `stream`: `NULL`, `raw(8)`, or `raw(4)`.

Returns invisibly the `raw(8)` seed actually used. Replays:

```r
saved <- dq_set_seed(NULL)       # records the auto-seed
x <- dq_runif(100)
dq_set_seed(saved)
identical(x, dq_runif(100))      # TRUE
```

Under the hood: convert `raw` → `uint64_t` (LE), split into two
`integer` words for `dqrng::dqset_seed()`. Wrong length / wrong type
errors with a clear message — no silent integer coercion.

### 5.2 RNG kind

```r
dq_RNGkind(kind)              # set kind
dq_RNGkind()                  # get kind  (= dq_kind())
dq_kinds()                    # list valid kinds
```

`dq_kinds()` returns a fixed character vector — arbitrary lowercased
strings (which `dqrng::dqRNGkind()` accepts) are no longer the
contract.

### 5.3 State

```r
state <- dq_get_state()
# <dq_state: xoroshiro128++>
#   state: raw[16] 4f 9c ... a3
dq_set_state(state)
```

S3 list with class `"dq_state"`:

```r
structure(
  list(kind = "xoroshiro128++", state = raw(N)),
  class = "dq_state"
)
```

Implementation: dqrng exposes the state as a `character` vector of
decimal-encoded words; dqrng2's R wrapper parses it per kind, packs
each word as LE bytes via `writeBin(word, raw(), size = 8, endian =
"little")`, and concatenates. The inverse runs in `dq_set_state()`.
All in R, no new C++ needed for the state channel.

State sizes (the engine state words only; see §13 on the bit cache):

| Kind | state |
|---|---|
| `xoroshiro128+/++/**`      | `raw(16)` (2 × 8 bytes) |
| `xoshiro256+/++/**`        | `raw(32)` (4 × 8 bytes) |
| `pcg64`                    | `raw(32)` (state + stream, both 128-bit) |
| `threefry` (20-round 64)   | `raw(64)` (key 256 + counter 256) |

S3 methods: `print.dq_state`, `format.dq_state`, `==.dq_state`,
`as.raw.dq_state` (flatten: 1-byte kind length, kind bytes,
state bytes), `as_dq_state.raw` (inverse).

### 5.4 Key generation

```r
dq_keys(n = 1L, bytes = 8L)
```

Returns a `list` of length `n`, each element a `raw(bytes)`. Drawn
from R's RNG via `RNGScope` — no state alteration, exactly the same
entropy source `generateSeedVectors()` uses. Implemented as one
small Rcpp function in dqrng2; does not go through `dqrng`'s
`generateSeedVectors()` (which would force integer round-tripping).

### 5.5 Distributions and sampling

Pure forwarders — same R signature shape as dqrng's:

```r
dq_runif(n, min = 0, max = 1)
dq_rnorm(n, mean = 0, sd = 1)
dq_rexp(n, rate = 1)
dq_rademacher(n)
dq_sample(x, size, replace = FALSE, prob = NULL)
dq_sample_int(n, size = n, replace = FALSE, prob = NULL)
dq_rmvnorm(n, ...)   # forwards to mvtnorm via dq_rnorm
```

Output bit-for-bit identical to dqrng's counterparts.

### 5.6 `register_methods` — important from day one

```r
dq_register_methods(kind = c("both", "rng"))
dq_restore_methods()
```

Pure forwarders to `dqrng::register_methods()` /
`dqrng::restore_methods()`. Because dqrng2 shares dqrng's global RNG
(by virtue of forwarding), registering dqrng as R's user-supplied
RNG automatically routes `set.seed`, `runif`, etc. through dqrng —
and therefore through dqrng2 too. No additional plumbing required.

A new option `dqrng2.register_methods` (default `FALSE`) controls
auto-registration on load:

```r
# in dqrng2/R/zzz.R
.onLoad <- function(libname, pkgname) {
  if (isTRUE(getOption("dqrng2.register_methods", FALSE)))
    dq_register_methods()
}
```

Documented interaction: setting **either** `dqrng.register_methods`
**or** `dqrng2.register_methods` to `TRUE` produces the same observed
behaviour. Calling out the equivalence in the man page avoids a
double-registration trap.

Caveats from `?register_methods` carry over verbatim into
`?dq_register_methods` (notably: `dq_set_seed(NULL)` re-seeds from
R's RNG; with the methods registered, `set.seed(NULL)` is the
appropriate entry point).

## 6. C++ API

`dqrng2` exposes its own C++ surface in namespace `dqrng2`, even
though it delegates to `dqrng`'s implementation. The headers live in
`inst/include/` and are usable via `LinkingTo: dqrng2`.

### 6.1 `dqrng2::byte_span` / `byte_buffer`

`inst/include/dqrng2_types.h` (no Rcpp dependency):

```cpp
namespace dqrng2 {
struct byte_span { const std::uint8_t* data; std::size_t size; };
struct byte_buffer {
  std::vector<std::uint8_t> bytes;
  operator byte_span() const { return {bytes.data(), bytes.size()}; }
};
}
```

### 6.2 Little-endian byte / uint64 conversions

`inst/include/dqrng2_le.h`:

```cpp
namespace dqrng2 {

inline std::uint64_t le_to_u64(byte_span s) {
  // s.size in [1, 8]; high-order bytes assumed zero if size < 8
  std::uint64_t v = 0;
  for (std::size_t i = 0; i < s.size; ++i) {
    v |= std::uint64_t(s.data[i]) << (8 * i);
  }
  return v;
}

inline void u64_to_le(std::uint64_t v, std::uint8_t* out) {
  for (std::size_t i = 0; i < 8; ++i) {
    out[i] = std::uint8_t(v >> (8 * i));
  }
}

}  // namespace dqrng2
```

Loops, not `memcpy` + host-endian guess. Correct on every host;
optimisers fold the loop to a `mov` on LE.

### 6.3 `dqrng2::engine` — wrapper around dqrng's accessor

`inst/include/dqrng2_engine.h`:

```cpp
namespace dqrng2 {

class engine {
  dqrng::random_64bit_accessor acc;  // dqrng's accessor of the global RNG

public:
  engine() = default;

  std::uint64_t next();
  double        uniform01();

  void seed(byte_span s);
  void seed(byte_span s, byte_span stream);

  // RNG-kind metadata
  std::string_view kind() const;
  std::size_t      state_bytes() const;

  // byte-oriented state I/O — under the hood, calls dqrng's text
  // serde and converts.
  byte_buffer save() const;
  void        load(byte_span);

  // template variate/generate helpers
  template <typename Dist, typename... A>
  typename Dist::result_type variate(A&&...);
  template <typename Dist, typename Range, typename... A>
  void generate(Range&&, A&&...);
};

}  // namespace dqrng2
```

`engine` is the dqrng2-shaped C++ surface that downstream package
authors program against. They never see `dqrng::random_64bit_*` types
directly.

### 6.4 `dqrng2::rcpp` — adapters

`inst/include/dqrng2_rcpp.h`: `as_byte_span(SEXP)`,
`to_raw(byte_span)`, `as_state(Rcpp::List)`,
`to_state(string_view, byte_span)`. The only header that depends on
Rcpp.

### 6.5 `src/dqrng2.cpp` — Rcpp exports

Tiny:

```cpp
// [[Rcpp::interfaces(r, cpp)]]
// [[Rcpp::depends(dqrng)]]

#include <dqrng.h>
#include <dqrng2_engine.h>
#include <dqrng2_rcpp.h>

// [[Rcpp::export(rng = false)]]
Rcpp::RawVector dq_set_seed_impl(Rcpp::Nullable<Rcpp::RawVector> seed,
                                 Rcpp::Nullable<Rcpp::RawVector> stream);

// [[Rcpp::export(rng = true)]]
Rcpp::List dq_keys_impl(int n, int bytes);

// (state get/set live in R — see §5.3)
// distributions / sampling have no dqrng2 C++ glue; the R wrappers
// call dqrng::dqr* directly.
```

Distributions in §5.5 forward at the R level (`dq_runif <- function(n,
min = 0, max = 1) dqrng::dqrunif(n, min, max)`). The C++ surface in
`dqrng2::engine` exists for *downstream package authors* who want
clean dqrng2 types in their own C++ code; the R surface doesn't need
it.

## 7. Backporting to `dqrng` — issue-sized chunks

Each item is a self-contained PR for `dqrng`, mergeable in any
order. `dqrng2` is the reference implementation.

| # | Idea | Difficulty | ABI impact |
|---|---|---|---|
| 1 | Accept `raw(4\|8)` for `seed` / `stream` (LE) | small | additive (new Rcpp export + R dispatch) |
| 2 | `dqset.seed()` returns seed used invisibly | trivial | none |
| 3 | New `dqrng_keys(n, bytes)`; deprecate `generateSeedVectors()` | small | additive |
| 4 | `dqrng_state` S3 class for `dqrng_get_state` / `set_state` | small-medium | additive (new exports via kind-switched downcast) |
| 5 | `dqrng_kinds()` enumerator + validation in `dqRNGkind()` | trivial | none |
| 6 | PCG64 stream semantics: absolute rather than relative | medium | behavioural |
| 7 | Byte-oriented C++ virtuals on the base | large | C++ ABI |
| 8 | Drop `convert_seed<T>` integer packing | large | C++ ABI |

Items 1–5 are uncontroversial and mergeable today. 6 is a
behavioural call. 7–8 are the heavy lift; the maintainer takes them
when `dqrng2` has been running on the design long enough to be
confident.

Each item has a tracking issue in `daqana/dqrng` linking to the
corresponding `dqrng2` PR/commit as a reference implementation —
plus, for items 7–8, a porting guide in `dqrng2`'s repo showing the
before/after for every downstream call site type.

## 8. Tests

In `tests/testthat/`:

- `test-set-seed.R`: every accepted shape of `seed`/`stream`
  produces the same `dq_runif(100)` output as the canonical `raw(8)`
  form. Length 4 and 8 covered; bad lengths error; bad types error.
- `test-replay.R`: `s <- dq_set_seed(NULL); x <- dq_runif(100);
  dq_set_seed(s); y <- dq_runif(100); expect_identical(x, y)` —
  for every RNG kind.
- `test-state.R`: for every supported kind, draw →
  `dq_get_state()` → consume → `dq_set_state(saved)` → draw
  matches.
- `test-keys.R`: `dq_keys(n, bytes)` shape, contents drawn from
  R's RNG, `n = 0` returns `list()`.
- `test-register.R`: `dq_register_methods()` makes `set.seed(42L);
  runif(5)` match `dq_set_seed(42L); dq_runif(5)` (modulo the
  caveats from `?register_methods`).
- `test-le-layout.R`: hand-encoded byte patterns vs known uint64
  values; `writeBin(..., endian = "little")` cross-check; `0x12 0x34`
  in `raw(2)` reads as `0x3412`.
- `test-equivalence.R`: for every `dq_*` distribution / sampling
  function, output bit-for-bit equals dqrng's counterpart after
  equal seeding.
- `test-state-shape.R`: state lengths per kind match the table in
  §5.3.
- `tests/testthat/cpp/le.cpp`: round-trip
  `le_to_u64`/`u64_to_le` against hand-encoded patterns.
- `tests/testthat/cpp/engine.cpp`: dqrng2 `engine` smoke tests
  (next, seed, save/load round trip).

## 9. Package layout

```
dqrng2/
├── DESCRIPTION              # LinkingTo: Rcpp, dqrng; Imports: dqrng
├── NAMESPACE
├── R/
│   ├── dqrng2-package.R
│   ├── set_seed.R           # dq_set_seed()
│   ├── state.R              # dq_get_state(), dq_set_state(), parser
│   ├── state-class.R        # S3 methods for dq_state
│   ├── keys.R               # dq_keys()
│   ├── kind.R               # dq_RNGkind(), dq_kind(), dq_kinds()
│   ├── distributions.R      # dq_runif/rnorm/rexp/rademacher (forwarders)
│   ├── sample.R             # dq_sample, dq_sample_int (forwarders)
│   ├── mvnorm.R             # dq_rmvnorm (forwarder)
│   ├── register.R           # dq_register_methods, dq_restore_methods
│   └── zzz.R                # .onLoad with dqrng2.register_methods option
├── src/
│   ├── dqrng2.cpp           # Rcpp glue
│   └── Makevars[.win]
├── inst/include/
│   ├── dqrng2.h             # umbrella
│   ├── dqrng2_types.h       # byte_span (no Rcpp)
│   ├── dqrng2_le.h          # le_to_u64 / u64_to_le
│   ├── dqrng2_engine.h      # engine wrapping dqrng's accessor
│   ├── dqrng2_rcpp.h        # Rcpp adapters
│   └── dqrng2_RcppExports.h # generated
├── tests/testthat/...
├── plan/                    # this design, build-ignored
└── README.md                # "experimental wrapper" prominent
```

## 10. Repository plan

`dqrng2` lives in its own repo (`krlmlr/dqrng2` to start). Initial
commit is the package skeleton plus this plan (`plan/dqrng2.md`).
First milestone (`0.0.1`) is tagged from a single PR.

In this repo (`krlmlr/dqrng`), `plan/raw.md` becomes a one-page
pointer: short paragraph + link to `dqrng2` + the backport
checklist from §7 reproduced as a GitHub project board.

## 11. Roadmap

| Milestone | Contents |
|---|---|
| `dqrng2 0.0.1` | wrappers for every dqrng function in §2; `dq_state`; `dq_keys`; `dq_register_methods`; LE byte layout; full test suite |
| `dqrng2 0.0.2` | `dqrng2::engine` C++ surface for downstream consumers; LinkingTo smoke-test package |
| `dqrng2 0.0.3` | Performance pass — verify the wrapper layer adds negligible overhead vs direct `dqrng::dqr*` calls (within a few percent on `bench::mark`); document |
| `dqrng → ...` | Backport PRs 1–5 from §7, in any order |
| `dqrng2 0.1.0` | API freeze candidate after the wrapper has been load-tested by at least one downstream package |
| `dqrng` major | Items 6–8 from §7 (with reverse-dep check), coordinated with `dqrng2` |

`dqrng2` does not need to retire once `dqrng` adopts everything —
it stays as a canary track for the next round of experiments.

## 12. Out of scope

- BE / `endian = "big"` flag. We commit to LE.
- Multi-byte stream IDs > 8 bytes (PCG64's 128-bit selector,
  xoshiro256's 256-bit jump). `byte_span` makes this cheap to add
  later; not for `0.0.1`.
- Fixing PCG64's "stream relative to current" inside dqrng2 — we
  forward to dqrng's behaviour. Backport item §7 #6.
- Alternative RNGs beyond the dqrng set. dqrng2 is a wrapper; new
  RNGs belong upstream.
- A wire format / on-disk file format. `as.raw(<dq_state>)` is
  already a byte string; `saveRDS()` / `writeBin()` cover storage.

## 13. Open questions

- **Function naming.** Three candidates:
  - `dq_*`: short, distinctive, no base R clash. **Lean.**
  - `dqrng2_*`: full prefix, ugly in code.
  - unprefixed (`set_seed`, `runif`): clashes with base R if
    `library(dqrng2)` is used.
- **`dq_set_seed(NULL)` return value visibility.** Lean invisible
  (matches `base::set.seed`); document the recipe `s <-
  dq_set_seed(NULL)` for replay.
- **Bit cache in `dq_state`.** `dqrng`'s
  `random_64bit_generator::cache` / `has_cache` (the 32-bit half of
  the bit cache, `inst/include/dqrng_types.h:48-55`) is part of the
  observable RNG output. dqrng's character-vector state serialisation
  may or may not include it — needs verification before we trust
  `dq_get_state` / `dq_set_state` to round-trip perfectly. If dqrng
  drops it, add an explicit `cache` field to the `dq_state` list
  (separate `raw(5)`) and synthesise the missing half via a
  scratch draw.
- **Performance overhead.** Forwarding `dq_runif` through R adds an
  R-level function call per draw — negligible for vectorised calls,
  potentially noticeable for `n = 1` in a tight loop. Measure in
  `0.0.3`; if material, expose `dq_runif` etc. as Rcpp re-exports
  rather than R forwarders.
- **State parser fragility.** Parsing dqrng's text state encoding
  assumes a stable format (RNG kind tag + decimal words separated
  by whitespace). Pin a dqrng version range in `Imports`; flag any
  upstream change in CI.
- **Two `register_methods` options.** Both
  `dqrng.register_methods` and `dqrng2.register_methods` exist after
  this lands. If both are `TRUE` we call `register_methods()` twice
  on load (once per package's `.onLoad`); the second is a no-op but
  worth a comment in the docs.
