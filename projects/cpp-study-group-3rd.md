---
layout: page
permalink: /projects/cpp-study-group-3rd/
title: Qishi C++ Low-Latency Systems Study Group
---

## Qishi C++ Low-Latency Systems Study Group

- Completed an 8-week C++ systems curriculum through the Qishi C++ Study Group
- Curriculum adopted from *Building Low-Latency Applications with C++*
- Week 8 sharing topic: **Instrumentation & Optimization in Low-Latency C++ Applications**
- Prepared and presented materials on RDTSC, hop-by-hop timestamping, latency distributions, and hot-path tuning
- Wrote a standalone `rdtsc_latency_demo.cpp` to demonstrate measurement, p95/p99 analysis, and spiky vs stable latency paths
- [Curriculum GitHub](https://github.com/qishiclub/cpp-study-group-3rd.git)

## Table of contents
- [Curriculum Outline](#curriculum-outline)
- [Week 8 Sharing Materials](#week-8-sharing-materials)
- [Measurement Philosophy](#measurement-philosophy)
- [RDTSC Microbenchmarking](#rdtsc-microbenchmarking)
- [TTT Hop Timestamping](#ttt-hop-timestamping)
- [Timing Utilities](#timing-utilities)
- [Measurement Macros](#measurement-macros)
- [Where to Instrument](#where-to-instrument)
- [Performance Analysis Workflow](#performance-analysis-workflow)
- [Latency Distribution Reading](#latency-distribution-reading)
- [Optimization Case Studies](#optimization-case-studies)
- [Data Structure Tuning](#data-structure-tuning)
- [RDTSC Demo Code Notes](#rdtsc-demo-code-notes)
- [Connection to MatchEngine](#connection-to-matchengine)

---

## Curriculum Outline

The study group was structured as an 8-week path from C++ performance foundations to a full electronic-trading-style system.

| Week | Topic | Focus |
|---|---|---|
| 1 | C++ low-latency systems overview | Performance goals, why C++ matters for latency-sensitive systems, compiler flags, and common designs across streaming, gaming, IoT, electronic trading, AI infrastructure, and crypto |
| 2 | Core building blocks | Threads, concurrency models, memory pools, lock-free queues, low-latency logging, TCP/UDP utilities |
| 3 | Trading ecosystem design | Trading system requirements, exchange/client/strategy architecture, and system constraints |
| 4 | Matching engine | Limit order book design and order matching implementation |
| 5 | Exchange connectivity | Market data publisher, order server, and assembling the exchange binary |
| 6 | Client connectivity | Market data consumer, client-side order book, and order gateway |
| 7 | Strategy framework and trading strategies | Position/PnL tracking, signal generation, order management, risk controls, market making, liquidity taking, and end-to-end client integration |
| 8 | Instrumentation and optimization | High-granularity timestamping, performance analysis, visualization, tuning, benchmark gains, and extension ideas |

The weekly format mixed theory, code demos, GitHub code analysis, STL internals, and participant project discussion. Code was the real homework: every concept had to connect back to something measurable or implementable.

---

## Week 8 Sharing Materials

My Week 8 sharing focused on turning "low latency" from a vague goal into a measurable engineering loop:

```text
instrument -> collect logs -> compute metrics -> visualize -> diagnose -> optimize -> re-measure
```

The materials had two parts:

- slides: instrumentation and optimization concepts from Chapters 11 and 12 of *Building Low-Latency Applications with C++*
- demo code: a standalone RDTSC latency measurement program simulating matching-engine and sequencing paths

The most important point was that optimization without measurement is just guessing. In a trading-style system, the question is rarely "is it fast once?" It is usually "is it predictably fast at p99 under burst load?"

---

## Measurement Philosophy

### 1. "You cannot optimize what you cannot measure"

Low-latency work begins with a baseline. Without instrumentation, it is easy to optimize the wrong function, improve average latency while hurting tails, or mistake I/O noise for CPU cost.

**Practical rule:** before changing a hot path, record at least one baseline distribution.

---

### 2. Observer effect

Measurement itself has cost. If the instrumentation is too heavy, the measurement changes the system being measured.

This is why the slides separate:

- low-overhead cycle measurements for individual functions
- absolute timestamps for end-to-end flow analysis

**Practical rule:** use the lightest measurement that answers the question.

---

### 3. Average latency is not enough

Average latency can hide bursts. A function can have a reasonable mean but still produce dangerous p99 spikes.

For latency-sensitive systems, the page-worthy metrics are:

- min
- max
- mean
- median
- p50
- p95
- p99
- distribution shape

**Practical rule:** if p99 is bad, the system is not predictably low-latency even if the mean looks good.

---

## RDTSC Microbenchmarking

### 1. What RDTSC measures

`rdtsc` reads the CPU Time Stamp Counter, a 64-bit register that counts CPU cycles.

It is useful for:

- hot function profiling
- microbenchmarking small code regions
- comparing before/after changes
- detecting tight vs spiky latency distributions

It is not the same as wall-clock time. To convert cycles into time, you need an estimate of CPU frequency.

---

### 2. Why use RDTSC instead of `std::chrono` everywhere

The slides compare RDTSC with higher-level clocks:

- RDTSC overhead is small enough for hot-path measurement
- `std::chrono::high_resolution_clock` is easier to use but usually heavier
- `std::chrono` is still useful for wall-clock timestamps and coarse timing

**Practical rule:** RDTSC is for very fine-grained local timing; `chrono` is better for readable timestamps and broader elapsed-time logic.

---

### 3. x86-specific limitation

The demo code uses inline assembly:

```cpp
__asm__ __volatile__("lfence\n\t"
                     "rdtsc\n\t"
                     : "=a"(lo), "=d"(hi)
                     :
                     : "memory");
```

This is x86/x86_64 specific. It is appropriate for explaining low-latency measurement on common trading-system hardware, but it is not portable C++.

**Practical rule:** write architecture-specific timing utilities behind a small abstraction.

---

### 4. `lfence` before `rdtsc`

The demo uses `lfence` before `rdtsc`.

Why it matters:

- modern CPUs execute instructions out of order
- without serialization, measured regions can be polluted by reordering
- `lfence` helps ensure earlier instructions retire before the timestamp is read

**Practical rule:** measuring a small code block needs ordering control, not just a clock call.

---

### 5. `__volatile__` in inline assembly

The inline assembly is marked `__volatile__`.

Why it matters:

- tells the compiler the assembly has observable behavior
- reduces the chance the compiler removes or reorders the timing instruction
- helps preserve measurement intent

**Practical rule:** compiler optimization is your friend in release builds, but it must not optimize away the measurement itself.

---

### 6. Combining high and low registers

`rdtsc` returns the counter through two 32-bit pieces:

```cpp
return (static_cast<uint64_t>(hi) << 32) | lo;
```

This combines the high and low halves into one 64-bit cycle count.

**Practical rule:** tiny bit-level details matter in performance tooling because the measurement path must be correct before any result is meaningful.

---

### 7. `noexcept` on measurement utilities

The demo marks timing helpers as `noexcept`.

Why it matters:

- timing helpers should not throw
- exception metadata and pessimistic assumptions are undesirable in hot utilities
- it documents that the function is safe to call in latency-sensitive code

**Practical rule:** hot-path helper functions should express their guarantees in the type system where possible.

---

## TTT Hop Timestamping

### 1. What TTT means in the slides

TTT-style measurement records absolute nanosecond timestamps at system boundaries.

RDTSC answers:

```text
How many cycles did this function take?
```

TTT answers:

```text
How long did the message take to move from this component to the next?
```

Both are needed because a fast function can still sit behind a slow queue, network hop, or scheduler delay.

---

### 2. Exchange-side hops

The exchange topology tracks message flow from ingress to egress:

| Hop | Meaning |
|---|---|
| `T1` | TCP read at order server ingress |
| `T2` | lock-free queue write toward matching engine |
| `T3` | lock-free queue read by matching engine |
| `T4` | market data queue write |
| `T5` | market data queue read by publisher |
| `T6` | UDP/multicast market data send |
| `T4t` | response queue write |
| `T5t` | response queue read |
| `T6t` | TCP response write |

**Practical rule:** end-to-end latency is a chain. Measuring only the matching function misses the queue and network boundaries.

---

### 3. Client-side hops

The client topology tracks market data, strategy, order gateway, and response paths:

| Hop | Meaning |
|---|---|
| `T7` | UDP market data read |
| `T8` | queue write toward trade engine |
| `T9` | trade engine queue read |
| `T10` | order request queue write |
| `T11` | order gateway queue read |
| `T12` | TCP order send |
| `T7t` | TCP response read |
| `T8t` | response queue write |
| `T9t` | response queue read |

**Practical rule:** client latency is not just strategy logic; market data ingestion, queueing, and order gateway egress all count.

---

### 4. Lock-free queues still need measurement

The architecture uses LFQueues between components. "Lock-free" does not mean "free."

Queue cost can come from:

- cache-line transfer between cores
- producer/consumer imbalance
- bursts that create backlog
- false sharing
- memory-ordering costs

**Practical rule:** measure queue write/read hops directly instead of assuming the queue is negligible.

---

## Timing Utilities

### 1. `getCurrentNanos()`

The demo uses:

```cpp
std::chrono::steady_clock::now()
```

This is a good choice for monotonic elapsed-time measurement because `steady_clock` does not move backward due to wall-clock adjustments.

**Practical rule:** use `steady_clock` for elapsed time; use wall-clock time only when human-readable timestamps matter.

---

### 2. `getCurrentTimeStr()`

The demo formats wall-clock time as:

```text
HH:MM:SS.nnnnnnnnn
```

This is useful for logs because humans need readable timestamps when correlating events.

**Practical rule:** keep two time concepts separate:

- monotonic time for measurement
- wall-clock time for log readability

---

### 3. `localtime_r` portability

The demo uses `localtime_r`, which is POSIX-style and works on Linux/macOS. On Windows, equivalent code usually uses `localtime_s`.

**Practical rule:** demo code can target Linux-style low-latency environments, but production utility code should isolate platform-specific time formatting.

---

## Measurement Macros

### 1. `START_MEASURE`

The demo macro is:

```cpp
#define START_MEASURE(TAG) const uint64_t TAG = Common::rdtsc()
```

It captures a start cycle count into a variable named by the tag.

**Why this is useful:** the tag becomes both the measurement variable and the identity of the measured region.

---

### 2. `END_MEASURE`

The demo macro is:

```cpp
#define END_MEASURE(TAG, LOGGER)                                               \
  do {                                                                         \
    const uint64_t end_##TAG = Common::rdtsc();                                \
    (LOGGER).logRDTSC(#TAG, end_##TAG - TAG);                                  \
  } while (false)
```

Key C/C++ macro tricks:

- `#TAG` stringifies the tag, so it can be logged as text
- `end_##TAG` uses token pasting to create a unique variable name
- `do { ... } while (false)` makes the macro behave like one statement

**Practical rule:** if you use macros for instrumentation, make them syntactically safe and easy to remove or disable.

---

### 3. `TTT_MEASURE`

The demo also defines:

```cpp
#define TTT_MEASURE(TAG, LOGGER)                                               \
  do {                                                                         \
    const uint64_t ts_##TAG = Common::getCurrentNanos();                       \
    (LOGGER).logTTT(#TAG, ts_##TAG);                                           \
  } while (false)
```

It records an absolute timestamp rather than a cycle delta.

**Practical rule:** use delta measurements for local function cost; use absolute timestamps to reconstruct a message path.

---

### 4. Macro scope and early returns

In `MEOrderBook::removeOrder`, the code calls `END_MEASURE` before each early return:

```cpp
if (it == id_map_.end()) {
  END_MEASURE(Exchange_MEOrderBook_removeOrder, g_logger);
  return false;
}
```

This matters because failure paths are still paths. In a real exchange, cancels for missing or invalid orders may be frequent enough to measure.

**Practical rule:** every return path inside an instrumented function should close the measurement.

---

## Where to Instrument

### 1. Wrap the critical section

The slides show `START_MEASURE` at the beginning of the core operation and `END_MEASURE` after the core work.

Do not wrap too much:

- includes unrelated logging
- includes setup/teardown noise
- hides the true function cost

Do not wrap too little:

- misses important data structure operations
- undercounts validation or early-return behavior
- produces misleading numbers

**Practical rule:** the measurement boundary should match the question you are asking.

---

### 2. Tag naming convention

The slides use names like:

```text
Exchange_MEOrderBook_removeOrder
Exchange_FIFOSequencer_seqPub
```

The pattern is:

```text
Component_Class_Method
```

**Why it matters:** logs become much easier to parse and group when tag names are consistent.

---

### 3. Exchange-side functions worth measuring

The slides list these hot functions:

- `processClientRequest`
- `addOrder`
- `removeOrder`
- `match`
- `checkForMatch`
- `sequenceAndPublish`
- `send`

These map naturally to matching engine latency, order server latency, and market data publication latency.

---

### 4. Client-side functions worth measuring

The slides list these client-side functions:

- `PositionKeeper::addFill`
- `PositionKeeper::updateBBO`
- `FeatureEngine::onOrderBookUpdate`
- `RiskManager::checkPreTradeRisk`
- `OrderManager::moveOrder`

These represent a typical trading-client path: update state, compute signal, check risk, and send orders.

---

## Performance Analysis Workflow

### 1. Parse logs

The raw logs include:

```text
timestamp RDTSC tag cycles
timestamp TTT tag timestamp_ns
```

The first analysis step is extracting:

- timestamp
- measurement type
- tag
- value

**Practical rule:** log format should be boring and parseable.

---

### 2. Convert cycles to time

RDTSC gives cycles, not seconds.

The demo estimates CPU frequency:

```cpp
const uint64_t elapsed_cycles = end_tsc - start_tsc;
const double cpu_freq_ghz =
    static_cast<double>(elapsed_cycles) / static_cast<double>(elapsed_ns);
```

Then it converts:

```cpp
cycles_to_ns = 1.0 / cpu_freq_ghz;
cycles_to_us = cycles_to_ns / 1000.0;
```

**Practical rule:** always state whether a number is cycles, nanoseconds, or microseconds.

---

### 3. Compute distribution metrics

The demo sorts a copy of the latency vector and computes:

- min
- max
- mean
- median
- p50
- p95
- p99
- sample count

Sorting a copy preserves the original raw sequence while making percentile lookup simple.

**Practical rule:** keep raw samples; derived stats are summaries, not replacements.

---

### 4. Visualize

The slides recommend:

- scatter plots for time-vs-latency
- histograms for distribution shape
- percentile bands for tail behavior

The demo prints a console histogram with 20 bins. It is not as polished as a Jupyter plot, but it demonstrates the same idea: shape matters.

**Practical rule:** a latency distribution is often more informative than a single "average latency" number.

---

### 5. Diagnose

The slides classify patterns:

- tight distribution: stable code path
- spiky distribution: contention, batching, burst behavior, or slow-path activity

**Practical rule:** a spike is a clue. Ask what event, branch, allocation, flush, queue backlog, or OS action caused it.

---

## Latency Distribution Reading

### 1. Stable path: `MEOrderBook::removeOrder`

The slide example treats order cancellation as a relatively tight path:

```text
min:  about 0.4 us
mean: about 1.2 us
max:  about 3.5 us
```

Interpretation:

- the operation is narrow and predictable
- it mostly touches known data structures
- tail behavior still depends on cache and tree state

**Practical rule:** tight distributions are good, but still validate the data structure and cache behavior under larger books.

---

### 2. Spiky path: `FIFOSequencer::sequenceAndPublish`

The slide example shows a much wider distribution:

```text
average: about 90 us
spikes:  500-1200 us
pattern: burst-driven
```

Interpretation:

- sequencing cost depends on how many pending requests arrive
- publishing/batching creates periodic heavy work
- queue flush behavior can dominate tail latency

**Practical rule:** if the path processes variable-size batches, measure batch size alongside latency.

---

### 3. Network and queue latency

The slides call out:

- one network hop can add roughly 15-20 us
- consumer queue latency can average around 100 us and spike much higher

These numbers are context-dependent, but the lesson is stable:

```text
function latency != system latency
```

**Practical rule:** combine function-level RDTSC with hop-level TTT to see both local and system-level cost.

---

## Optimization Case Studies

### 1. Logger: avoid character-by-character hot-path work

The original logger path pushed one log element per character:

```cpp
for (auto c : str) {
  pushValue(c);
}
```

The optimized path copies the string into a fixed-size buffer and pushes once.

Slide result:

```text
before: 25,757 cycles/op
after:     466 cycles/op
speedup: about 55x
```

Why it matters:

- logging is often thought of as "outside the core logic"
- but if log construction happens on the hot path, it can dominate latency
- one push per character multiplies queue traffic and memory movement

**Practical rule:** hot-path logging should batch, defer, or use fixed-size preallocated buffers.

---

### 2. Fixed-size string buffer in a union

The slide optimization stores a string payload inside a fixed-size union member:

```cpp
struct LogElement {
  LogType type_;
  union { char s[64]; /* ... */ } u_;
};
```

Why this helps:

- avoids dynamic allocation
- keeps log element size predictable
- turns variable-length string handling into bounded copy work

Trade-off:

- long strings must be truncated or handled separately

**Practical rule:** bounded buffers are common in low-latency systems because predictability often beats flexibility.

---

### 3. Memory pool: remove debug checks from release hot path

The slide memory-pool example shows `ASSERT` checks inside `allocate` and `deallocate`.

Slide result:

```text
before: 343 cycles/op
after:   44 cycles/op
speedup: about 8x
```

The trick is not to remove safety forever. The trick is:

```cpp
#ifndef NDEBUG
#define ASSERT(cond, msg) assert((cond) && (msg))
#else
#define ASSERT(cond, msg) ((void)0)
#endif
```

Debug builds keep checks. Release builds remove them.

**Practical rule:** debug safety and release latency can coexist if checks compile out cleanly.

---

### 4. Thread affinity and CPU isolation

Thread affinity pins critical threads to dedicated cores.

The slide maps components like:

- market data publisher
- matching engine
- order server
- market data consumer
- trade engine
- order gateway

The hottest components are the matching engine and trade engine.

Linux kernel parameters mentioned:

```text
isolcpus=0-5
nohz_full=0-5
rcu_nocbs=0-5
```

Why this helps:

- fewer context switches
- less scheduler interference
- less cache pollution
- tighter p99 latency

**Practical rule:** OS-level tuning often improves tail stability more than mean latency.

---

### 5. Eliminating `std::function`

The slide replaces runtime callback dispatch with compile-time CRTP.

Problem with `std::function` in hot paths:

- type erasure
- possible heap allocation for captures
- indirect call
- harder branch prediction
- less inlining

CRTP shape:

```cpp
template <typename Handler>
class OrderServer {
  void onRecv(ClientRequest& req) {
    static_cast<Handler*>(this)->onClientRequest(req);
  }
};
```

Why this helps:

- callback target resolved at compile time
- direct call can be inlined
- no type-erasure object in the hot path

**Practical rule:** `std::function` is elegant, but in a hot callback loop it must justify its cost.

---

### 6. Continuous profiling

The final slide emphasizes that performance is iterative:

```text
measure -> optimize -> re-measure
```

A one-time benchmark can go stale as code changes, data volume changes, or workloads become burstier.

**Practical rule:** performance work is a feedback loop, not a one-off cleanup pass.

---

## Data Structure Tuning

### 1. Big-O is not the whole story

`std::unordered_map` is average O(1), but that does not guarantee it is fastest in practice.

CPU cost also depends on:

- cache locality
- pointer chasing
- allocator behavior
- load factor
- rehashing
- key type
- collision behavior

**Practical rule:** for hot paths, "O(1)" is the beginning of the discussion, not the end.

---

### 2. Fixed array as a fast map

If the key range is known and small, `std::array` can act like a simple map:

```text
key -> array index
```

Benefits:

- contiguous memory
- no dynamic allocation
- predictable access
- cache-friendly layout

Cost:

- wastes memory for sparse key ranges
- fixed maximum size
- not suitable for arbitrary keys

**Practical rule:** fixed-range ids are an opportunity for array-backed lookup.

---

### 3. `std::unordered_map` trade-off

The slide benchmark:

```text
array hash map:     142,650 cycles/op
unordered_map:      152,457 cycles/op
overhead:           about 6-7%
```

Why `unordered_map` can be slower:

- buckets may point to nodes elsewhere in memory
- node allocation hurts locality
- collisions add traversal
- rehash can be expensive if not controlled

Why it is still useful:

- dynamic growth
- standard library availability
- supports large or unknown key ranges
- works for strings and custom key types

**Practical rule:** choose the data structure for the workload, not just for the API convenience.

---

### 4. Alternatives to explore

The slides mention several alternatives:

- `absl::flat_hash_map`
- Robin Hood hashing
- hopscotch hashing
- `folly::AtomicHashMap`
- `emhash7::HashMap`

The common theme is improved memory layout or concurrency behavior compared with a node-heavy general-purpose map.

**Practical rule:** if a hash map sits in the hottest path, benchmark more than one implementation.

---

## RDTSC Demo Code Notes

### 1. Compile with optimization enabled

The demo compile command is:

```bash
g++ -O2 -std=c++17 -o rdtsc_demo rdtsc_latency_demo.cpp
```

`-O2` matters because measuring unoptimized code answers a different question. Debug builds are useful for correctness, but performance conclusions should come from optimized builds.

**Practical rule:** benchmark the kind of binary you actually care about.

---

### 2. Header choices reveal the demo scope

The demo includes:

- `<chrono>` for timestamps and frequency estimation
- `<cstdint>` for fixed-width integer types
- `<map>`, `<queue>`, `<tuple>`, and `<vector>` for simulated exchange components
- `<numeric>` and `<algorithm>` for stats
- `<iomanip>` for readable output

This makes the demo self-contained and easy to compile without external dependencies.

**Practical rule:** teaching demos should minimize setup friction so the concept is the focus.

---

### 3. `enum class Side`

The demo uses:

```cpp
enum class Side { BUY, SELL };
```

Why it is good:

- scoped enum avoids leaking `BUY` and `SELL` into the global namespace
- stronger type safety than a raw `char` or unscoped enum

Trade-off:

- real protocols may encode side as a byte for compactness

**Practical rule:** use type-safe representation in internal code, then encode/decode at boundaries.

---

### 4. Demo `Order` uses `double` price

The demo `Order` stores price as `double`.

For production matching engines, I would still use integer ticks. Here the demo is about instrumentation, not price correctness.

**Practical rule:** a demo can simplify unrelated concerns, but the simplification should be explicit.

---

### 5. `MEOrderBook` duplicates lookup state

The demo stores orders in:

- `bids_`
- `asks_`
- `id_map_`

This makes cancellation lookup simple:

```cpp
auto it = id_map_.find(order_id);
```

Then it erases from the side-specific map and from the id map.

**Practical rule:** matching engines often trade extra memory for faster cancel/modify paths.

---

### 6. Measure both success and failure paths

`removeOrder` logs measurement before returning false for:

- unknown order id
- wrong client id
- wrong side

This is important because rejected cancels still consume engine time.

**Practical rule:** failure paths can be part of the hot path.

---

### 7. `insert_or_assign`

The demo uses:

```cpp
bids_.insert_or_assign(order.order_id, order);
id_map_.insert_or_assign(order.order_id, order);
```

This keeps the code compact and makes repeated ids overwrite prior state in the demo.

In production, duplicate order ids would likely be rejected or handled explicitly.

**Practical rule:** demo convenience should not be confused with exchange semantics.

---

### 8. `FIFOSequencer` creates a bursty path

The demo's sequencer does two stages:

```text
pending queue -> sequenced queue -> published vector
```

Every tenth iteration adds 50 requests instead of 5. This intentionally creates latency spikes.

**Practical rule:** burst simulation is useful because real systems often fail at the tail, not at the average.

---

### 9. Periodic flush simulation

The sequencer clears `published_` when it grows past 100:

```cpp
if (published_.size() > 100) {
  published_.clear();
}
```

This simulates periodic batching or network flush behavior.

**Practical rule:** periodic cleanup work often explains latency spikes.

---

### 10. Turn off verbose logging during measurement

The demo prints the first few samples, then disables verbose logging:

```cpp
if (i == 5) {
  g_logger.setVerbose(false);
}
```

Why it matters:

- console I/O is extremely slow compared with the code being measured
- printing every iteration would dominate the measurement
- first few samples are useful for proving instrumentation works

**Practical rule:** never benchmark a hot path with uncontrolled printing inside the loop.

---

### 11. Pre-populate the order book

The demo inserts 1000 orders before measuring cancels.

Why it matters:

- measuring an empty book is not representative
- data structure state affects lookup and erase cost
- prepopulation creates a more realistic baseline

**Practical rule:** benchmark stateful systems in a realistic state.

---

### 12. CPU frequency estimation

The demo estimates CPU frequency by reading RDTSC, sleeping for 100ms, reading RDTSC again, and dividing cycles by elapsed nanoseconds.

This is useful for a demo, but production measurement needs care because:

- CPU frequency may vary
- turbo boost can affect assumptions
- TSC behavior depends on hardware
- cross-core TSC consistency matters

**Practical rule:** cycles are great for comparing code paths; converting to time requires hardware awareness.

---

### 13. Percentile index calculation

The demo computes percentile indexes with:

```cpp
size_t idx = static_cast<size_t>(p * static_cast<double>(n - 1));
```

Then clamps to `n - 1`.

This is simple and fine for a demo. More rigorous percentile definitions exist, but the lesson is to look beyond average latency.

**Practical rule:** do not let perfect statistics block useful tail-latency visibility.

---

### 14. Histogram binning

The demo prints 20 bins and scales each bar to a width of 50 characters.

Why this helps:

- you can see whether samples cluster tightly
- you can spot long tails
- you can compare distributions without a plotting library

**Practical rule:** a simple text histogram is better than no distribution view.

---

### 15. Compare components side by side

The demo ends with a summary table:

```text
Component                    Mean cycles    P99 cycles    Pattern
MEOrderBook::removeOrder     ...            ...           Tight
FIFOSequencer::seqPub        ...            ...           Spiky
```

This is a good presentation pattern because the audience can connect implementation differences to measured behavior.

**Practical rule:** performance reporting should name both the metric and the interpretation.

---

### 16. What the demo intentionally does not solve

The demo is not a production exchange engine. It intentionally skips:

- price-time priority matching
- integer tick correctness
- protocol parsing
- memory pool allocation
- real lock-free queues
- network I/O
- thread pinning

That is fine because the purpose is to teach instrumentation mechanics.

**Practical rule:** a focused demo should isolate one concept instead of pretending to be the whole system.

---

## Connection to MatchEngine

This Week 8 material directly shaped how I think about my MatchEngine project:

- add instrumentation before optimizing data structures
- keep printing outside benchmark loops
- report p95/p99, not only average runtime
- separate correctness decisions from performance demos
- keep order cancel paths measurable
- treat allocations, logging, and callbacks as possible hot-path costs
- consider memory pools for orders/events
- consider lock-free queue handoff if the engine becomes multi-threaded
- use integer ticks for price correctness even when demos simplify price representation
- compare alternative containers empirically instead of assuming `unordered_map` is always fastest

The next natural step for my own engine is to add a small instrumentation layer, run repeatable benchmarks for add/cancel/match paths, and report latency distributions the same way this Week 8 demo does.
