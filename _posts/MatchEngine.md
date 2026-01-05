---
layout: post
title: "Building and Profiling a Minimal Matching Engine in Modern C++"
date: 2026-01-04
author: Felix Hao
categories: [C++, Systems, Trading]
tags:
  - order-book
  - matching-engine
  - c++20
  - systems-programming
  - performance
  - benchmarking
---

Designing a matching engine is one of those problems that looks trivial at first glance but quickly exposes the tradeoffs between correctness, performance, and clarity. In this post, I walk through my implementation of a **price–time priority order book** in modern C++, explain my thought process and data-structure choices, and then dive into how I **measured and optimized performance** using **Google Benchmark** and **Linux `perf`**.

This implementation processes a CSV stream of orders, supports **add** and **cancel** operations, and performs **continuous matching** between bids and asks.

---

## Design Goals

Before writing any code, I set a few explicit goals:

1. Correct price–time priority  
2. Fast cancels (O(1) average)  
3. Simple, predictable data structures  
4. Minimal heap allocations  
5. Clear separation of parsing, matching, and bookkeeping  
6. Measurable performance characteristics  

This is not a production exchange. There’s no persistence, concurrency, or fault tolerance. The focus is the **core mechanics** and how they behave under load.

---

## Order Representation

```cpp
struct Order {
  OrderType type : 1;
  Side side : 1;
  unsigned qty : 30 {0};
  int ts{0};
  int order_id{0};
  int price{0};
  std::array<char, 16> trader{};
};
```

### Rationale

- Bitfields reduce memory footprint and improve cache density.  
- `std::array<char, 16>` avoids heap allocation for trader identifiers.  
- Orders are mutable: quantity is decremented during matching.  
- The structure is trivially movable and cheap to store inside containers.  

Memory layout matters in a matching engine — not because of micro-optimizations, but because poor locality compounds quickly at scale.

---

## Order Book Layout

```cpp
static std::map<int, std::list<Order>> asks;
static std::map<int, std::list<Order>, std::greater<int>> bids;
```

The book is modeled as:

- A price-indexed map  
- Each price level containing a FIFO list of orders  

### Why `std::map` + `std::list`?

- `std::map` maintains sorted prices with stable iterators.  
- `std::list` preserves time priority and supports O(1) erasure.  
- The structure mirrors how many real-world engines are implemented.  

Asks are sorted ascending, bids descending, allowing best-price access via `begin()`.

---

## Fast Cancel Support

Cancel performance is often overlooked and becomes a bottleneck under stress. To avoid scanning the book, I maintain a direct index:

```cpp
struct location {
  Side side;
  int price;
  std::list<Order>::iterator it;
};

static std::unordered_map<int, location> orderIndex;
```

This enables:

- O(1) average lookup by order ID  
- Direct erasure from the correct price level and FIFO list  

The index is updated on:

- Order insertion  
- Order cancellation  
- Full fills during matching  

This is the single most important optimization in the design.

---

## Parsing the Input Stream

I use C++20 ranges to parse CSV input:

```cpp
auto parts = line | std::views::split(',');
```

This avoids:

- Temporary vectors  
- Repeated substring allocations  
- Excessive copying  

Parsing is intentionally kept simple and deterministic. Any malformed line is ignored early to avoid contaminating the book state.

---

## Matching Logic

### Trade Condition

```cpp
static bool canTrade(Side s, int oppoPrice, int selfPrice) {
  return s == Side::Buy ? oppoPrice > selfPrice
                        : oppoPrice < selfPrice;
}
```

This enforces:

- Buy orders match asks priced ≤ bid  
- Sell orders match bids priced ≥ ask  

### Core Matching Loop

```cpp
static void process(Order incoming, auto &opposite) {
  while (incoming.qty > 0 && !opposite.empty()) {
    auto oppoIt = opposite.begin();
    if (canTrade(incoming.side, oppoIt->first, incoming.price))
      break;

    auto &level = oppoIt->second;
    auto &topOppo = level.front();

    unsigned traded = std::min(incoming.qty, topOppo.qty);
    incoming.qty -= traded;
    topOppo.qty -= traded;

    maybe_print_trade(...);

    if (topOppo.qty == 0) {
      orderIndex.erase(topOppo.order_id);
      level.pop_front();
    }

    if (level.empty())
      opposite.erase(oppoIt);
  }

  if (incoming.qty > 0)
    addOrder(std::move(incoming));
}
```

### Properties

- Always matches against the best available price  
- Preserves FIFO time priority  
- Handles partial and full fills  
- Cleans up empty price levels immediately  

The function is templated (`auto &opposite`) to remain zero-cost and avoid virtual dispatch.

---

## Adding and Canceling Orders

### Adding

```cpp
static void addOrder(Order &&o) {
  auto &book = (o.side == Side::Buy) ? bids : asks;
  auto &level = book[o.price];

  level.push_back(std::move(o));
  auto it = std::prev(level.end());

  orderIndex[it->order_id] = location{it->side, it->price, it};
}
```

### Canceling

```cpp
static bool cancelOrder(int order_id) {
  auto idx = orderIndex.find(order_id);
  if (idx == orderIndex.end())
    return false;

  const location &loc = idx->second;
  auto &book = (loc.side == Side::Buy) ? bids : asks;

  auto levelIt = book.find(loc.price);
  if (levelIt == book.end()) {
    orderIndex.erase(idx);
    return false;
  }

  levelIt->second.erase(loc.it);
  if (levelIt->second.empty())
    book.erase(levelIt);

  orderIndex.erase(idx);
  return true;
}
```

Both operations are O(1) average, excluding the map lookup.

---

## Controlling Output for Benchmarking

```cpp
static bool g_should_print = true;
```

Printing is gated so the same codebase can be used for:

- Functional testing  
- Benchmarking  
- Profiling  

This is critical: I/O will completely dominate runtime otherwise.

---

## Benchmarking with Google Benchmark

To measure performance objectively, I used Google Benchmark to isolate and test:

- Order insertion throughput  
- Matching throughput  
- Cancel latency  

### Example Benchmark Setup

```cpp
static void BM_AddOrders(benchmark::State& state) {
  for (auto _ : state) {
    resetBook();
    for (int i = 0; i < state.range(0); ++i) {
      Order o{OrderType::Add, Side::Buy, 100, 0, i, 100 + i, {}};
      addOrder(std::move(o));
    }
  }
}

BENCHMARK(BM_AddOrders)->Range(1 << 10, 1 << 20);
```

This allowed me to observe:

- Logarithmic behavior from `std::map`  
- Cache effects as book size grows  
- Sensitivity to order distribution  

---

## Profiling with Linux `perf`

Benchmarks tell you how fast something is. `perf` tells you why.

### Typical Workflow

```bash
perf record -g ./matching_engine benchmark_input.csv
perf report
```

### Key Findings

- Cancels were dominated by hash table access, not list erasure  
- Matching loop was branch-predictable under realistic order flow  
- I/O completely dwarfed computation when printing was enabled  
- `std::map` iterator chasing was the primary cache miss source  

These insights validated the design and pointed clearly to where future optimizations would matter.

---

## Final Thoughts

This project reinforced a few core lessons:

- Data structure choices dominate performance  
- Cancels must be first-class operations  
- Benchmarks without profiling are incomplete  
- Modern C++ can be both expressive and fast  

This order book is intentionally minimal, but it models real matching behavior accurately. With Google Benchmark and `perf`, it also provides a solid platform for evidence-driven optimization, not guesswork.

Thanks for reading.
