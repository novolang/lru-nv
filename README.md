# lru-nv

A cache holds a bounded number of recently computed answers, and
decides which one to drop when it is full. This package brings three
of those decisions to novo-lang: **least recently used** (LRU), the
policy the Linux page cache and Python's `functools.lru_cache` use;
**least frequently used** (LFU); and a **time to live** (TTL), where an
entry expires a fixed time after it was written. The reference
implementations are the Rust crate [lru](https://docs.rs/lru) and the
Python library [cachetools](https://cachetools.readthedocs.io/).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a cache is

A cache maps keys to values and holds at most a stated number of them.
The number is its **capacity**. When a write would take it past the
capacity, the cache **evicts** an entry — removes it to make room — and
which entry it removes is its **eviction policy**.

| Policy | What it evicts | What it is good at |
| --- | --- | --- |
| LRU | The entry read longest ago | A working set that moves over time |
| LFU | The entry read fewest times | A working set that is stable, with bursts of one-off keys |
| TTL | Any entry older than its lifetime, then as LRU | Answers that go stale whether or not anyone reads them |

An LRU cache orders its entries by when each was last read. A read
therefore **changes the cache**: the entry that was just read is now
the most recent one, and some other entry is now the one that will be
evicted next.

An LFU cache counts reads instead. A key read once an hour for a year
survives a burst of ten thousand keys that are each read once, which in
an LRU it would not. What it pays is that a key which was popular last
week outranks the key that is popular today, for as long as the cache
lives.

A TTL cache gives each entry an **expiry**: an instant after which it is
no longer answered. Knowing whether an entry has expired means knowing
what the time is, and no function in this package reads a clock. The
caller reads the clock and passes the instant in.

Every cache here is a **value**. An operation does not change the cache
it was given; it answers a new one. Keys are `Str`. Values are the
caller's own type.

## Install

```
novo pkg add lru-nv
```

## Example

```novo
use lrucache

fn main() [io]
    // A cache of two entries. The type argument is the value type.
    let c: LruCache<Str> = lrucache.new(2)

    // Each write answers the cache it produced, and whatever left it.
    let a = lrucache.put(c, "user:1", "ada")
    let b = lrucache.put(a.cache, "user:2", "grace")

    // A read answers the value and the cache that recorded the read.
    // Keeping that cache is what makes "user:1" the most recent entry.
    let r = lrucache.get(b.cache, "user:1")
    match r.value
        Some(name) => println("cached: ${name}")
        None       => println("not cached")

    // A third write is one too many, so one entry is evicted. It comes
    // back with the reason it left.
    let d = lrucache.put(r.cache, "user:3", "alan")
    let dropped = list.len(d.evicted)
    println("evicted ${dropped} entry")
    println("next out: ${lrucache.eviction_order(d.cache)[0]}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: lru-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `lruevict` | The reasons an entry leaves a cache, and the entry itself as it is handed back. |
| `lrucache` | The least-recently-used cache: reads, writes, removals, the two capacities and the eviction order. |
| `lrulfu` | The least-frequently-used cache, its use counts, and the halving that ages them. |
| `lruttl` | The expiring cache, the instant type it compares against, and the purge. |

## How to choose an entry point

**`lrucache` is the one to reach for first.** Most caches want the
entry read longest ago dropped, and this is that.

**`lrulfu` is for a workload with a stable working set**, where a scan
over many one-off keys would otherwise flush the entries that matter.
Its cost is a count per entry and a popularity that can go stale.

**`lruttl` is for answers that go stale on their own** — a DNS record,
an access token, a permissions lookup. It evicts by recency too, so it
is an LRU cache with an expiry rather than a different policy.

**`lrucache.get` records a read; `lrucache.peek` does not.** Use
`peek` in a metrics endpoint, a debugger and a test, none of which
should change what the next eviction takes.

## The rules a user needs

1. **Every operation answers a new cache, and the caller must keep
   it.** `put`, `remove`, `clear` and `shrink_to` answer a cache;
   `get` answers a value *and* a cache. A caller that drops the cache a
   `get` returned has dropped the recency update, and the cache will
   evict a different entry than it should.
2. **A read is a write.** `lrucache.get` moves its entry to the most
   recently used end, and `lrulfu.get` increments its use count. That
   is what these policies are.
3. **Keys are `Str`.** A caller keyed by something else formats it
   first, and must format it the same way every time.
4. **Two capacities hold at once.** The entry count is set by
   `lrucache.new`, the total weight by `with_max_weight`, and an entry
   is evicted when either limit would be exceeded. A cache with neither
   limit never evicts.
5. **A weight is whatever the caller says it is** — bytes, rows,
   pixels. `put` weighs an entry `1`, so a cache with only an entry
   limit behaves the same either way.
6. **An entry heavier than the whole weight limit is refused, not
   stored.** Storing it would evict everything else and then not fit.
   It comes back as a `LruWeightExceeded` eviction, carrying the value
   that was not stored.
7. **Everything that leaves a cache is handed back**, with the reason
   it left. A write-back cache flushes it, a connection pool closes it,
   a counter counts it. The reasons are `capacity`, `weight`,
   `expired`, `replaced` and `removed`.
8. **`lruevict.was_forced` separates the evictions the cache chose from
   the ones the caller asked for.** A hit-ratio dashboard counts the
   first three reasons and ignores the last two.
9. **The eviction order is total and stated.** `eviction_order` answers
   the keys with the next one out first. In the LFU cache, entries with
   equal use counts are ordered by their last read, least recent first.
10. **A new LFU entry starts at one use, not zero**, so the entry just
    written is not the first one dropped by a cache already full of
    entries read once.
11. **`lrulfu.halve_uses` is the only defence against stale
    popularity**, and the caller decides when to call it. This package
    reads no clock, so it cannot decide for itself.
12. **The current instant is an argument.** Every `lruttl` function
    that has to know whether an entry has expired takes it. Expiry is
    testable without a fake clock, and a replay of a request log
    expires exactly what it expired the first time.
13. **The epoch is the caller's.** `lruttl.instant` takes a
    millisecond count. Unix milliseconds and a monotonic counter both
    work, as long as one program uses one of them throughout.
14. **Expiry is lazy.** An expired entry stays stored until something
    looks at it. `lruttl.len` counts what is stored, `live_len` counts
    what would still be answered at an instant, and `purge` drops the
    expired entries and reports them.
15. **`lruttl.time_to_live_millis` answers `0` both for an expired key
    and for a key that was never written**, so a caller deciding
    whether to refresh asks one question.

## Running on a microcontroller

Every module in this package builds for a microcontroller: no function
performs any input or output, reads a clock, or starts a task. What a
cache needs from its environment — the current time, a place to write
an evicted value back to — arrives as an argument or is handed back to
the caller.

The memory a cache uses is the entries it holds, so a caller on a small
device sets a capacity and knows the bound.

## What is not included

- **A clock.** See rule 12. Reading one is an effect, and this package
  declares none.
- **A background sweep.** Nothing here runs on a timer. `lruttl.purge`
  is the call, and the caller's own periodic task makes it.
- **A loading cache.** A cache that computes a missing value has to
  call the function that computes it, and whatever that function costs
  is what the cache costs. A caller writes the three lines around a
  miss instead.
- **Thread safety.** These are values, not shared mutable containers. A
  cache shared between tasks belongs behind whatever the program
  already uses to share state.
- **Hit and miss counters.** Count at the call site. A cache that
  counted would answer a different cache from `peek` too, and the
  number a dashboard wants is usually per route rather than per cache.
- **Non-`Str` keys.** See rule 3.

## Related packages

- [heapless-nv](https://novo-lang.org/packages/heapless-nv) holds
  fixed-capacity containers for a device with no heap allocator. Take
  it when the bound matters more than the eviction policy.
- [ratelimit-nv](https://novo-lang.org/packages/ratelimit-nv) decides
  whether an action may happen now, from an instant the caller
  supplies. It takes the clock the same way this package does.
- [prometheus-nv](https://novo-lang.org/packages/prometheus-nv) is
  where a hit ratio goes once a caller counts one.

## Tests

```bash
novo test tests/lrucache_tests.nv    # the eviction order and the two capacities
novo test tests/lruttl_tests.nv      # expiry, and the instant that decides it
novo test tests/lrucover_tests.nv    # the LFU cache and the eviction reasons
```

The reference implementations are the Rust crate `lru`, whose
`get`/`peek` split this package keeps, and Python's `cachetools`, whose
`LRUCache`, `LFUCache` and `TTLCache` are the three policies here. The
suite asserts that the eviction order is least-recently-used first,
that a read moves its entry and a peek does not, that both capacities
evict, that an over-heavy entry is refused rather than stored, that
every eviction is reported with its reason, that an expiry is decided
by the instant the caller passes and by nothing else, and that an
expired entry is still stored until something looks at it.

The tests compile today and fail at run, each on the
`not implemented: lru-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `lruevict.LruEvictReason`, `.LruEvicted` and the other types | the types are declared |
| `lruevict.reason_name`, `.was_forced` | no |
| `lrucache.new`, `.with_max_weight`, `.put`, `.put_weighed` | no |
| `lrucache.get`, `.peek`, `.remove`, `.contains` | no |
| `lrucache.len`, `.capacity`, `.weight`, `.max_weight`, `.is_empty`, `.clear` | no |
| `lrucache.eviction_order`, `.entries_in_eviction_order`, `.next_eviction`, `.shrink_to` | no |
| `lrulfu.new`, `.put`, `.get`, `.peek`, `.remove` | no |
| `lrulfu.uses_of`, `.halve_uses`, `.len`, `.capacity`, `.eviction_order` | no |
| `lruttl.instant`, `.millis_of`, `.plus_millis` | no |
| `lruttl.new`, `.put`, `.put_for`, `.get`, `.peek`, `.remove`, `.purge` | no |
| `lruttl.expires_at`, `.time_to_live_millis`, `.len`, `.live_len`, `.ttl_millis`, `.eviction_order` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
