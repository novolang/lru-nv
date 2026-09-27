# lru-nv

A cache holds a bounded number of recently computed answers, and
decides which one to drop when it is full. This package brings three
of those decisions to novo-lang: **least recently used** (LRU), the
policy the Linux page cache and Python's `functools.lru_cache` use;
**least frequently used** (LFU); and a **time to live** (TTL), where an
entry expires a fixed time after it was written. The reference
implementations are the Rust crate [lru](https://docs.rs/lru) and the
Python library [cachetools](https://cachetools.readthedocs.io/).

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

The program prints `cached: ada`, then `evicted 1 entry`, then
`next out: user:1`. The write of `user:3` evicted `user:2`, because the
read had made `user:1` the more recent of the two.

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

1. **Every operation answers a new cache, and the caller keeps it.**
   `put`, `remove`, `clear` and `shrink_to` answer a cache. `get`
   answers a value and a cache. A caller that drops the cache a `get`
   answered has dropped the read, and a later write evicts a different
   entry.
2. **A read changes the cache.** `lrucache.get` moves its entry to the
   most recently used end. `lrulfu.get` adds one to its use count.
3. **Keys are `Str`.** A caller with another key type formats it first,
   and formats it the same way every time.
4. **Two limits hold at once.** `lrucache.new` sets the most entries,
   and `with_max_weight` sets the most total weight. An entry is
   evicted when either limit would be passed. A cache with neither
   limit never evicts.
5. **A weight is whatever the caller says it is**, such as bytes, rows
   or pixels. `put` weighs an entry `1`, so a cache with only an entry
   limit behaves the same with either call.
6. **An entry heavier than the whole weight limit is refused.** It is
   not stored, and nothing else is evicted. It comes back as an
   `LruWeightExceeded` eviction that carries the value.
7. **Everything that leaves a cache is handed back** with the reason it
   left. The reasons are `capacity`, `weight`, `expired`, `replaced`
   and `removed`.
8. **`lruevict.was_forced` is true for the first three reasons.** Those
   are the evictions the cache chose. `replaced` and `removed` are the
   ones the caller asked for.
9. **The eviction order is total.** `eviction_order` answers every key,
   the next one out first. In the LFU cache, the fewest uses leave
   first, and entries with equal use counts leave least recently used
   first.
10. **A new LFU entry starts at one use.** A write replacing a key keeps
    the old entry's count.
11. **Use counts age only when the caller asks.** `lrulfu.halve_uses`
    halves every count, rounding down. Nothing else lowers a count.
12. **The current instant is an argument.** Every `lruttl` function that
    has to know whether an entry has expired takes it, and no function
    reads a clock.
13. **The epoch is the caller's.** `lruttl.instant` takes a count of
    milliseconds. Unix time and a monotonic counter both work, if one
    program uses one of them throughout.
14. **An entry is answered up to and including its expiry instant.** It
    has expired at every later instant.
15. **Expiry is lazy.** An expired entry stays stored until an operation
    looks at it. `lruttl.len` counts what is stored, `live_len` counts
    what would be answered at an instant, and `purge` drops the expired
    entries and reports them.
16. **A full expiring cache evicts an expired entry first.** It takes
    the least recently used of the expired entries, or of all entries
    when none has expired.
17. **`lruttl.time_to_live_millis` answers `0` for an expired key and
    for a key that is not stored.**

## What is not included

- **A clock.** See rule 12. No function in this package performs input
  or output, reads a clock or starts a task.
- **A microcontroller build.** A cache is a list of entries on the
  heap, and a build for a microcontroller with no heap allocator
  refuses one. heapless-nv holds fixed-capacity containers for that
  case.
- **A background sweep.** Nothing here runs on a timer. The caller's
  own periodic task calls `lruttl.purge`.
- **A loading cache.** A cache that computed a missing value would run
  the caller's function inside every miss. A caller writes the few
  lines around a miss instead.
- **Thread safety.** A cache is a value, not a shared mutable container.
  A cache shared between tasks goes behind whatever the program already
  uses to share state.
- **Hit and miss counters.** A caller counts at the call site, usually
  per route rather than per cache.
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
novo test tests/lrucache_tests.nv    # 5 tests: the eviction order and the two limits
novo test tests/lruttl_tests.nv      # 4 tests: expiry, and the instant that decides it
novo test tests/lrucover_tests.nv    # 4 tests: the LFU cache and the eviction reasons
novo test tests/model_tests.nv       # 2 tests: the LFU and expiring caches against a model
bash tests/coverage.sh               # line coverage over src/
```

The reference implementations are the Rust crate `lru`, whose
`get` and `peek` split this package keeps, and Python's `cachetools`,
whose `LRUCache`, `LFUCache` and `TTLCache` are the three policies
here.

The API suites assert that the eviction order is least recently used
first, that a read moves its entry and a peek does not, and that both
limits evict. They assert that an entry heavier than the weight limit
is refused, that every eviction comes back with its reason, and that
the caller's instant alone decides an expiry.

The model suite runs a few hundred operations from a fixed seed on the
LFU and the expiring cache, and on a plain list that keeps what each
cache should hold. After every operation it compares the answers, the
evictions and the eviction order.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
