---
layout: page
permalink: /projects/
title: Projects
---

## MatchEngine

-   Built a minimal matching engine for a single stock using
    **integer-based pricing**
-   Focused on **performance, determinism, and correctness**
-   Emphasized exchange-style constraints such as **price-time
    priority**
-   [GitHub](https://github.com/Felix772/MatchEngine)

---

## Issues Encountered & Optimizations

### 1. Floating-point precision in price comparison

Stock trading systems require **exact ordering, exact equality, and
deterministic behavior**.\
IEEE floating-point (`float` / `double`) cannot guarantee these
properties due to rounding errors, platform differences, and
non-associative arithmetic.

Using floats in a matching engine can lead to: - Incorrect order
matching - Violations of price-time priority - Audit and regulatory
risks - Incorrect P&L or tax calculations

**Solution**

Prices are represented as **integer ticks** instead of floating-point
values.

Example: - NVDA price: `$187.28` - Stored internally as: `18728` - Tick
size: `$0.01`

``` cpp
int64_t price_ticks = 18728; // $187.28
```

---

### 2. `std::getline` behavior difference between Windows and Linux

Windows and Linux use different line-ending conventions:

-   **Windows**: `\r\n`
-   **Linux / macOS**: `\n`

When parsing the same input file across platforms, Windows-formatted
files may leave a trailing `\r` character at the end of each line when
processed on Linux.

**Solution**

``` cpp
if (!line.empty() && line.back() == '\r') {
    line.pop_back();
}
```

---

### 3. Enabling move semantics (lvalues vs rvalues)
This is an essential part of my implementation in regards to logic clarity, performance optimization, memory allocations, etc. And before getting into move semantics, it's essential to first understand lvalues and rvalues in C++.
-   An **lvalue** identifies a named object with identity.
-   An **rvalue** represents a temporary or expiring object.

``` cpp
int x = 10; // x is an lvalue, 10 is an rvalue
```

#### Overload selection example

``` cpp
void g(const std::string& x) { }
void g(std::string&& x)      { }

std::string s = "hello";

g(s);
g("hello");
g(std::move(s));
```

#### Move Semantic application in Match Engine

``` cpp
\\adding a new order to buy/sell books
Order o = parse_order();
order_book.add(std::move(o));
```

### Why move semanic matters in the matching engine?

### 1) Matching engines are dominated by object movement

The price/quantity comparisons are cheap. The expensive parts are:

- Parsing getline inputs into internal objects  
- Inserting orders into book structures  
- printing trade events  

All of these involve **transferring ownership of data**. If you copy that data repeatedly, latency spikes and throughput collapses.

Move semantics exist to make the cost of ownership transfer cheaper.

---

### 2) The real performance cost: allocations, memory copies, and cache misses

#### What gets copied in a naive engine

An `Order` has:

- trader (`std::string`, optimized to `std::array(char, 16)` in later versions)
- qty (unsigned)
- order id (int)  
- price (int)
- etc.

Even price aand qty are integers, **trader can be variable-length and often heap-allocated**.

If you deep copy orders/events, you often copy:

- Heap buffers (strings / vectors)  
- Maps of tags  
- Shared structures holding optional fields  

This causes:

- Heap activity (malloc/free, fragmentation, allocator contention)  
- Memory bandwidth pressure (copying bytes)  
- Cache misses (moving large objects evicts hot order book data)  
- Tail latency explosions (rare but huge pauses)  

Move semantics reduces the first two directly, and indirectly improves the last two.

---

### 3) In matching engines, tail latency is as important as average latency

Even if copies only occasionally trigger allocations, those rare pauses show up as:

- p99 / p99.9 latency spikes  
- Missed market opportunities  
- Time-priority unfairness (orders processed later than they should be)  

Move semantics helps keep the “slow path” from happening frequently:

- Fewer allocations  
- Less copying  
- Less allocator contention in multi-threaded paths  

This is why move semantics is not a micro-optimization here—it is **architecture-relevant**.

---

### 4) Ownership boundaries are everywhere; move semantics makes them explicit and cheap

A typical match engine pipeline looks like:

**Network thread**  
→ decode message  
→ construct `Order`  
→ matching orders
→ push to containers (if qty > 0)
→ produce `Trade`
→ encode reports  
→ send  

At each boundary, you want **hand-off** semantics:

> “You own this now. I’m done with it.” or "I'm passing this ball to you"

In C++, that is exactly what `std::move` plus `T&&` APIs express.

Why that matters:

- With copies, it’s unclear who owns the “real” object.  
- With moves, the boundary is explicit: **after handoff, don’t touch**.  

This reduces bugs like:

- Double-frees (if you tried raw pointers)  
- Accidental shared mutation  
- Stale references after cancellations/updates  

---

### 5) Move semantics enables “zero-copy-ish” internal pipelines

You’ll never get literally zero copies end-to-end (network buffers → parsing → encoding), but move semantics helps you avoid **extra internal copies** that don’t add value.

#### A) Move into containers instead of copying

```cpp
std::unordered_map<OrderId, Order> live;
Order o = parse();
live.emplace(o.id, std::move(o)); // move into the book
```

Without move, the strings/vectors inside `Order` get deep-copied: expensive and noisy.

#### B) Move out of the book when an order is filled/canceled

When you remove an order from a container and create an event:

- Move fields into the event  
- Avoid re-copying IDs/tags  

---

### 6) Move semantics unlocks safer designs: move-only types and `std::unique_ptr`

Matching engines specifically benefit from designing key objects as **non-copyable**:

- Forces you to think about ownership and lifetime  
- Prevents accidental expensive deep-copies  
- Prevents subtle correctness bugs from “two copies of the same order”

I've not yet implemented this by the time of 1/6/2026, consideing the presence of copy contructing and copy assigning have been fully eleminated. However if I were to build a larger match engine supporting multiple threads this idea will definitely be considered.

Example code:

```cpp
struct Order {
  Order(const Order&) = delete; //disable copy constructor
  Order& operator=(const Order&) = delete; //disable copy assignment operator
  Order(Order&&) noexcept = default;
  Order& operator=(Order&&) noexcept = default;
};
```

Now the compiler enforces a property that is good for engines:

- Orders are **uniquely owned and transferred**, not duplicated  

This is a huge correctness win.

---

### 7) Determinism and fairness: move semantics helps keep processing predictable

Fairness in a match engine depends on strict ordering:

- Process messages in arrival order  
- Avoid stalls that reorder timing under load  

Copies + allocations introduce unpredictable stalls and contention. Under burst load:

- One message triggers allocation and stalls  
- Later messages slip ahead in scheduling or batching  
- Tail latency causes effective unfairness  

Moving tends to:

- Avoid heap churn  
- Keep per-message cost more uniform  
- Reduce long pauses  

Thus move semantics supports **operational fairness**, not just “correct logic.”

---

### 8) Move semantics reduces memory pressure and improves cache locality

When copying big objects:

- More bytes live simultaneously  
- Higher peak memory usage  
- More cache eviction of hot data (best bid/ask, maps, order nodes)  

Moving often allows:

- Reusing buffers  
- Transferring pointers  
- Keeping the working set smaller  

Smaller working set → better L1/L2 hit rates → lower latency.

---

### 9) Why `noexcept` moves matter in engine containers

Standard containers (like `std::vector`) prefer moving elements during reallocation **only if** the move constructor is `noexcept`. Otherwise they may copy to preserve strong exception safety.

If your event buffer is a vector and it grows:

- `noexcept` move keeps growth cheap  
- Non-`noexcept` move may fall back to copy (bad)  

In engines, you typically want:

- `noexcept` move constructors for core structs  
- Or preallocation to avoid reallocation  

Move semantics + `noexcept` is a real performance lever.

---

### 10) Concrete scenarios where move semantics pays off

- **High message rate + variable-length IDs**: copying repeats string copies; moving is pointer swaps  
- **Cancel/replace flows**: move unchanged fields (IDs, symbol) instead of copying  
- **Event journaling**: assemble event objects without deep-copying buffers  
- **Multi-thread queues** (as mentioned ###6): move-only ownership avoids ref-counting overhead and shared mutation  

---

### 11) Rule set for match engine code

- Never copy orders/events in the hot path unless proven cheap  
- Prefer APIs that express ownership:  
  - `foo(const T&)` = observe  
  - `foo(T&&)` or `foo(T)` = take  
- Prefer move-only for unique-ownership objects (`Order`, `Event`)  
- Mark move operations `noexcept` where possible  
- Use `emplace` and `std::move` intentionally at ownership boundaries  
- Don’t `std::move` something you still need (avoid use-after-move bugs)  


------------------------------------------------------------------------
### 4. Casting
