# Changelog

All notable changes to lru-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-09-27

The first implementation of the interface published as 0.0.1: the LRU
cache with its two limits, the LFU cache, the expiring cache and the
eviction reasons.

### Added

- Every body in `lruevict`, `lrucache`, `lrulfu` and `lruttl`.  The
  signatures are the ones 0.0.1 published.
- `tests/model_tests.nv` runs a few hundred operations from a fixed
  seed on the LFU and the expiring cache, and on a plain list that
  keeps what each should hold, and compares them after every step.
- `tests/coverage.sh` reports the line coverage over `src/`, merged
  across the suites.

### Behaviour the interface left open

- An entry is answered up to and including its expiry instant, and has
  expired at every later instant.
- A full expiring cache evicts the least recently used expired entry
  first, and the least recently used entry when none has expired.  An
  expired entry evicted this way leaves as `expired`.
- A write that replaces an LFU key keeps the old entry's use count.
  `halve_uses` rounds each count down.
- An entry heavier than the whole weight limit evicts nothing else.

### Changed

- The README no longer claims a microcontroller build.  A cache is a
  list of entries on the heap, and a build for a microcontroller with
  no heap allocator refuses one.

### Toolchain

- The toolchain floor is 0.14.0.  The bodies target novo 0.14.0 and
  carry no workaround for a compiler defect.
- One line of `src/`, the `Some(...)` that ends `lrulfu.peek`, is
  reported uncovered although the API suite runs it: `novo test --cov`
  puts no hook on a function's closing `Some(...)` expression.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `lrucache` — the load-bearing interface. A cache is a VALUE, so a
  read is a state change the caller has to keep: `get` answers
  `LruLookup`, which carries the value and the cache that recorded the
  hit. Every library that hides the recency update behind a mutation
  makes it possible to read a cache and not update it; here the type
  says what happened, and `peek` is the read that changes nothing. Two
  capacities — entries and a caller-supplied weight — hold at once.
- `lruevict` — everything that leaves a cache comes back, with the
  reason it left. `was_forced` divides the evictions the cache chose
  from the ones the caller asked for, which is the division a hit-ratio
  dashboard needs and no library gives.
- `lrulfu` — the least-frequently-used cache, with the tie-break
  stated: equal use counts are ordered by the last read. Without a
  stated tie-break an LFU's eviction order is not a function of its
  contents and no test can assert on it. `halve_uses` is the ageing
  step, and the caller chooses when it happens because this package
  reads no clock.
- `lruttl` — the expiring cache, with the instant as a PARAMETER: a
  `core` package may not read a clock, and the by-product is that
  expiry is testable without a fake clock and a replay of a request log
  expires exactly what it expired the first time. Expiry is lazy, and
  `len` and `live_len` are the two counts that follow from that.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  lru-nv.<module>.<fn>`.
- **The tests need a toolchain newer than 0.9.0.** They read a field
  off an element of a list of a generic struct — `put.evicted[0].key`
  — which 0.9.0 answers with an internal compiler error; the fix is on
  `main` after 0.9.0. The annotated bindings that worked around it are
  gone, so the suite reads naturally and no longer compiles on 0.9.0.
  No signature is affected, and `src/` builds on 0.9.0 unchanged.
- Keys are `Str`. A generic key would need the eviction structures to
  hash and compare a bare type parameter, and the README states the
  restriction rather than publishing a signature that cannot be
  implemented.
