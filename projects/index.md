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
-   [GitHub](https://github.com/Felix772/Match-Engine.git)

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
g(std::move(s));//this passes s to g() as an rvalue
```

#### Move Semantic application in Match Engine

``` cpp
\\adding a new order to buy/sell books
Order o = parse_order();
order_book.add(std::move(o));
```
### Important Note:
#### Move semantics apply to objects regardless of whether they are stored on the stack or heap. They are useful for types that manage resources (often heap memory). Trivially copyable types like int and char gain no benefit from move semantics because copying them is already optimal.

### **Why move semanic matters in the matching engine?**

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

The inputs for my most recent version of match engine are expected to be in the format "A,ts,order_id,side,price,qty,trader" or "C,ts,order_id". And since these data will be stored into Order containers right away and will not be modified. They are iterated using lazy views for a faster and more readable code:

```cpp

auto p = line | std::views::split(',');
auto it = p.begin();

```
Here p is a range view, where each element in p is a subrange of characters and behaves similarly to std::string_view. To convert them into the desired types for connstructing Orders, it's necessary to first create a string from each subrange of characters.

```cpp
\\helper for creating strings
static std::string to_string_field(auto &&field) {
  return {field.begin(), field.end()};
}

```
Now we can convert these strings to parameter types using `std::stoi()` (for integers), or `*(*it++).begin()` (for chars). However, since the quantity parameter is of type unsigned. It's necessary to cast the input string to unsigned after using the `std::stoul` method to avoid narrowing conversions.

```cpp
o.ts = std::stoi(to_string_field(*it++));
  if (it == p.end())
    return false;
  o.order_id = std::stoi(to_string_field(*it++));
  if (it == p.end())
    return false;
  o.side = *(*it++).begin();
  if (it == p.end())
    return false;
  o.price = std::stoi(to_string_field(*it++));
  if (it == p.end())
    return false;
  o.qty = static_cast<unsigned>(std::stoul(to_string_field(*it++)));
  if (it == p.end())
    return false;
  o.trader = to_string_field(*it);
```

---
### 5. Padding Alignment

If you've ever been surprised by `sizeof(Order)`, you're in good company. The compiler isn't being random—it’s enforcing **alignment**, and it often inserts **padding** to keep memory accesses efficient (and sometimes even valid) on your target CPU.

We'll use this struct as the running example:

```cpp
struct Order {
  char type; // 'A' or 'C'
  int ts;    // timestamp
  int order_id;
  char side; // 'B' or 'S'
  int price;
  unsigned qty;
  std::string trader;
};
```

## Alignment vs padding (quick mental model)

- **Alignment**: a type wants to live at an address that’s a multiple of some number (`alignof(T)`).
- **Padding**: extra bytes the compiler inserts to satisfy alignment of subsequent members (and sometimes at the end of the struct).

You can inspect the requirements directly:

```cpp
#include <iostream>
#include <string>

int main() {
  std::cout << "alignof(char) = " << alignof(char) << "\n";
  std::cout << "alignof(int) = " << alignof(int) << "\n";
  std::cout << "alignof(unsigned) = " << alignof(unsigned) << "\n";
  std::cout << "alignof(std::string) = " << alignof(std::string) << "\n";
}
```

> Note: exact numbers vary by platform/ABI, but the *mechanics* do not.

## What the compiler likely does to `Order`

On many 64-bit ABIs:
- `char` has alignment **1**
- `int` and `unsigned` have alignment **4**
- `std::string` often has alignment **8** and size commonly **24** or **32** (implementation-defined)

That matters because your struct starts with a `char`, then immediately has an `int`. Since `int` typically needs to start at a multiple of 4, the compiler will usually add **3 bytes** of padding right after `type`.

### A typical layout (conceptually)

Below is a *typical* offset story you might see on a 64-bit system:

- `type` at offset 0 (1 byte)
- **padding** at offsets 1–3 (3 bytes) so `ts` can start at offset 4
- `ts` at offset 4 (4 bytes)
- `order_id` at offset 8 (4 bytes)
- `side` at offset 12 (1 byte)
- **padding** at offsets 13–15 (3 bytes) so `price` can start at offset 16
- `price` at offset 16 (4 bytes)
- `qty` at offset 20 (4 bytes)
- then `trader` needs (commonly) 8-byte alignment; offset 24 is already a multiple of 8, so often no padding here
- `trader` starts at offset 24

**Total size** then becomes: `24 + sizeof(std::string)`, potentially rounded up to a multiple of `alignof(Order)` (often 8).

Again: the *exact* size of `std::string` and its alignment are implementation-defined, so always measure on your target.

## Measure instead of guessing: `offsetof`, `sizeof`, `alignof`

The fastest way to make this concrete is to print offsets:

```cpp
#include <cstddef>
#include <iostream>
#include <string>

struct Order {
  char type;
  int ts;
  int order_id;
  char side;
  int price;
  unsigned qty;
  std::string trader;
};

int main() {
  std::cout << "sizeof(Order)  = " << sizeof(Order) << "\n";
  std::cout << "alignof(Order) = " << alignof(Order) << "\n\n";

  std::cout << "offsetof(type)     = " << offsetof(Order, type) << "\n";
  std::cout << "offsetof(ts)       = " << offsetof(Order, ts) << "\n";
  std::cout << "offsetof(order_id) = " << offsetof(Order, order_id) << "\n";
  std::cout << "offsetof(side)     = " << offsetof(Order, side) << "\n";
  std::cout << "offsetof(price)    = " << offsetof(Order, price) << "\n";
  std::cout << "offsetof(qty)      = " << offsetof(Order, qty) << "\n";
  std::cout << "offsetof(trader)   = " << offsetof(Order, trader) << "\n";
}
```

If you see offsets jumping in ways that don't match member sizes, those gaps are padding.

## The big gotcha: `std::string` is not “just bytes”

Two important implications:

1. **`sizeof(Order)` does not include the string’s characters**  
   `std::string` typically stores a pointer/size/capacity (and often small-string optimization state) *inside* the object, but the text may live on the heap.

2. **Copying `Order` is not a cheap memcpy**  
   Copying/moving `std::string` can allocate, reference-count (depending on implementation), or at least copy metadata.

So even if you perfectly minimize padding, the performance profile of `Order` may be dominated by the string.

## Reducing padding in this struct

A common rule of thumb: **order members from largest alignment to smallest**.

The highest-alignment member here is likely `std::string`. After that, the `int`/`unsigned` fields (alignment 4), then the `char` fields.

An often-better ordering looks like:

```cpp
struct OrderPackedBetter {
  std::string trader;

  int ts;
  int order_id;
  int price;
  unsigned qty;

  char type;
  char side;
};
```

This typically:
- eliminates the padding between `char` and `int` members (because the `char`s are last)
- may reduce (or at least not increase) tail padding because the struct alignment is driven by `std::string` anyway

### But should you do this?

Reordering is great **inside your own codebase**, but it can be risky if:
- the struct crosses a shared library boundary (ABI concerns)
- you serialize it by `memcpy` (don’t do that with `std::string` anyway)
- other code assumes a specific layout

## An even bigger win: split “hot” fields from “cold” fields

In many systems (especially trading/order book code), you frequently touch the numeric fields but rarely need the trader name. That suggests **separating the string**:

```cpp
struct OrderCore {
  int ts;
  int order_id;
  int price;
  unsigned qty;
  char type;
  char side;
};

struct Order {
  OrderCore core;
  std::string trader;
};
```

Benefits:
- `OrderCore` is compact and trivially copyable
- arrays/vectors of `OrderCore` pack tightly and are cache-friendly
- you only pay `std::string` costs when you actually need it

This kind of “hot/cold split” often beats micro-optimizing padding.

## About `#pragma pack`: the temptation and the trap

You might think: “What if I just pack the struct to remove padding?”

For example:

```cpp
#pragma pack(push, 1)
struct OrderPacked {
  char type;
  int ts;
  int order_id;
  char side;
  int price;
  unsigned qty;
  std::string trader;
};
#pragma pack(pop)
```

This is usually a bad idea for general C++ structs because:
- it can create **misaligned accesses** (slower, and on some architectures possibly illegal)
- it can break assumptions in library code
- it’s non-standard and compiler-specific

Packing is mainly for **binary protocol/file layout structs**, and even then you should usually avoid placing complex types like `std::string` inside packed structs.

## Takeaways

- **Padding is normal**: it’s how the compiler satisfies alignment.
- In your original `Order`, expect padding after `type` and after `side` on common platforms.
- `std::string` dominates both layout *and* runtime costs; optimizing around it matters more than shaving a few padding bytes.
- Best options:
  - reorder members to reduce padding (safe when you control the ABI)
  - or split hot numeric fields from cold/heavy fields (often the biggest real-world win)

Happy measuring—`sizeof` rarely lies, it just tells the truth you didn’t want to hear.
