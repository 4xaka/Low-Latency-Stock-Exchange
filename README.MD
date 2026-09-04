# Stock Exchange

### Build Your Own Stock Exchange From Scratch in C++

A high-performance, low-latency **stock exchange matching engine** built from scratch in modern C++.

The goal of this project is to understand how the core of a real electronic exchange works — from accepting orders and maintaining an order book to matching buyers and sellers efficiently, while paying close attention to **memory allocation, cache behavior, data structures, concurrency, and predictable execution**.

> 🚧 **Project Status:** Work in Progress
> This project is being developed incrementally, with a focus on understanding and implementing the performance-critical components of a trading system.

---

## What Problem Does This Solve?

Traditional trading applications often hide the most important part of an exchange behind APIs and frameworks.

This project focuses on building that core system from the ground up.

The exchange needs to:

* Accept buy and sell orders.
* Maintain an efficient **Limit Order Book (LOB)**.
* Match compatible buy and sell orders.
* Preserve **price-time priority**.
* Process different types of orders correctly.
* Generate events for trades and order-state changes.
* Handle cancellations and modifications.
* Avoid unnecessary dynamic memory allocation on the trading hot path.
* Minimize synchronization and cache-related overhead.
* Provide deterministic and predictable behavior.

The project therefore serves as a practical exploration of **low-latency systems programming and exchange-engine architecture**.

---

## Core Architecture

At a high level, the system is organized around the following flow:

```text
                ┌─────────────────────┐
                │   Trader / Client   │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │    Order Gateway    │
                └──────────┬──────────┘
                           │
                           ▼
              ┌──────────────────────────┐
              │      Matching Engine     │
              │                          │
              │  • Validation            │
              │  • Price-Time Priority   │
              │  • Order Matching        │
              │  • Trade Generation      │
              └────────────┬─────────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │     Order Book      │
                │                     │
                │  Bids      Asks     │
                │  ─────     ─────    │
                │  Price     Price    │
                │  Levels    Levels   │
                └──────────┬──────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │       Events        │
                │                     │
                │  • Accepted         │
                │  • Rejected         │
                │  • Trade            │
                │  • Cancelled        │
                │  • Filled           │
                └─────────────────────┘
```

---

## Features Implemented

### Order Book

The exchange maintains separate bid and ask sides using price-indexed structures.

* Bids are ordered from **highest → lowest price**.
* Asks are ordered from **lowest → highest price**.
* Best bid and best ask are tracked.
* Orders are organized into price levels.
* FIFO ordering is maintained within each price level.

This enables the engine to implement the fundamental **price-time priority** rule used by electronic exchanges.

---

### Matching Engine

The matching engine determines whether an incoming order can trade against resting liquidity.

For example:

```text
BUY  @ 105
      │
      ▼
Best ASK = 103

105 >= 103
      │
      ▼
   TRADE
```

For a sell order:

```text
SELL @ 95
      │
      ▼
Best BID = 97

95 <= 97
      │
      ▼
   TRADE
```

Trades execute against the available liquidity at the resting price.

---

### Supported Order Concepts

The current matching implementation contains logic for multiple order types, including:

* Market
* Limit
* IOC
* FOK
* GTC
* Stop
* Stop-Limit
* Iceberg
* Post-Only

The engine also validates orders before they enter the book, rejecting invalid orders such as duplicate IDs, zero quantity, invalid prices, insufficient FOK liquidity, or post-only orders that would immediately cross the book.

---

### Order Cancellation

Active orders can be cancelled.

The engine:

1. Finds the active order.
2. Removes it from its price level.
3. Updates the order status.
4. Emits a cancellation event.
5. Reclaims the order's memory.

---

### Order Modification

Orders can also be modified.

The implementation distinguishes between:

* Quantity reduction at the same price.
* Price changes.
* Quantity increases.
* Replacement of the existing order.

This allows the exchange to preserve the appropriate matching semantics when an order changes.

---

### Iceberg Orders

The order book supports **iceberg orders**, where only a portion of the total quantity is publicly visible.

Example:

```text
Total Quantity = 10,000
Visible Quantity = 1,000

Order Book sees:

1,000

while internally:

10,000 total
```

Once the displayed quantity is consumed, another visible slice can be replenished.

---

### Stop Orders

The engine maintains separate stop-order structures and monitors the last traded price.

When the trigger condition is satisfied, a stop order can be activated and converted into the appropriate executable order type.

---

# Performance-Oriented Design

Performance is a major focus of this project.

Financial exchanges can process extremely large numbers of orders, so the design attempts to minimize expensive operations in latency-sensitive paths.

---

## Custom Memory Pool

The project implements a custom fixed-capacity `MemoryPool<T, N>`.

Instead of repeatedly performing:

```text
malloc()
   ↓
construct object
   ↓
destroy object
   ↓
free()
```

the pool pre-allocates storage and maintains a **free list**.

```text
                Memory Pool
┌─────────────────────────────────────────┐
│ Slot │ Slot │ Slot │ Slot │ Slot │ ... │
└───┬─────┬─────┬────────────────────────┘
    │     │     │
    ▼     ▼     ▼
   Free  Free  Free
    │
    ▼
 Free List
```

After initialization, allocation can simply pop a slot from the free list.

This reduces allocator overhead and makes allocation/deallocation much more predictable.

The current implementation also uses cache-line-oriented slot alignment and an intrusive free-list pointer stored inside the available object storage.

### Why?

The goal is to avoid calls to the general-purpose allocator during latency-sensitive operations whenever the pre-allocated capacity is sufficient.

---

## SPSC Queue

The project also contains a **Single Producer Single Consumer (SPSC) lock-free queue**.

```text
Producer
   │
   ▼
┌──────────────────────────────┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ ... │
└──────────────────────────────┘
              ▲
              │
           Consumer
```

The queue uses:

* `std::atomic`
* Acquire/release memory ordering
* A fixed-size ring buffer
* Power-of-two capacity
* Bit-mask indexing
* Cache-line separation of producer/consumer cursors

Because there is exactly one producer and one consumer, the queue can avoid traditional mutex-based synchronization.

---

## Cache Awareness

Performance-sensitive structures are designed with CPU cache behavior in mind.

For example, the SPSC queue separates its head and tail cursors using alignment, reducing the possibility of **false sharing** between producer and consumer threads.

The memory pool also aligns slots to at least a cache-line-sized boundary.

The broader objective is:

```text
Better locality
      ↓
Fewer cache misses
      ↓
Less memory latency
      ↓
More predictable execution
```

---

# Memory Resource Management

The order book uses C++ polymorphic memory resources (`std::pmr`) for several internal data structures.

Pre-allocated monotonic buffers are used for indexes such as:

* Order index
* Bid levels
* Ask levels
* Stop-order indexes

This allows the exchange to control where memory comes from rather than relying entirely on general-purpose allocation.

The design therefore combines:

```text
Memory Pool
     +
std::pmr
     +
Pre-allocated storage
     +
Cache-aware layout
```

to make memory behavior more predictable.

---

# Data Structures

The current exchange uses several important data structures.

| Component   | Data Structure / Technique | Purpose                          |
| ----------- | -------------------------- | -------------------------------- |
| Bid Book    | `std::pmr::map`            | Price-ordered bid levels         |
| Ask Book    | `std::pmr::map`            | Price-ordered ask levels         |
| Stop Orders | `std::pmr::multimap`       | Trigger-price organization       |
| Order Index | `std::pmr::unordered_map`  | Fast order lookup                |
| Orders      | Custom `MemoryPool`        | Predictable allocation           |
| Queue       | Fixed-size ring buffer     | SPSC communication               |
| Price Level | FIFO linked structure      | Price-time priority              |
| Events      | Pre-allocated array        | Avoid repeated event allocations |

The order book implementation explicitly maintains separate bid/ask maps and an order lookup index.

---

# C++ Techniques Used

This project is also intended as a deep dive into modern C++ systems programming.

### Modern C++

* C++20
* Templates
* `constexpr`
* `static_assert`
* `std::atomic`
* `std::array`
* `std::span`
* `std::optional`
* `std::unique_ptr`
* `std::byte`
* `std::construct_at`
* `std::destroy_at`
* `std::pmr`
* Strongly typed enums
* RAII
* `noexcept`

### Low-Level / Performance Techniques

* Custom memory pools
* Intrusive free lists
* Pre-allocation
* Cache-line alignment
* False-sharing avoidance
* Lock-free SPSC communication
* Acquire/release memory ordering
* Ring buffers
* Object lifetime management
* Contiguous storage
* Avoiding unnecessary heap allocations
* Branch prediction hints such as `[[likely]]` / `[[unlikely]]`

---

# Project Structure

```text
stock_exchange/
│
├── CMakeLists.txt
│
├── src/
│   ├── exchange_main.cpp
│   │
│   ├── core/
│   │   ├── clock.h
│   │   ├── memory_pool.h
│   │   ├── spsc_queue.h
│   │   └── types.h
│   │
│   └── matching/
│       ├── events.h
│       ├── level.h
│       ├── matching_engine.h
│       ├── order.h
│       └── order_book.h
│
└── tools/
    ├── trader_terminal.cpp
    └── liquidity_bot.cpp
```

The current repository follows this separation between core utilities, matching-engine components, and executable tools.

---

# Building the Project

## Requirements

* C++20 compatible compiler
* CMake 3.24+
* Ninja or another supported build system

The project explicitly requires C++20 and CMake 3.24 or newer.

---

## Clone

```bash
git clone https://github.com/RivaanRanawat/stock_exchange.git

cd stock_exchange
```

---

## Configure

Using CMake:

```bash
cmake -S . -B build
```

Or with Ninja:

```bash
cmake -S . -B build -G Ninja
```

---

## Build

```bash
cmake --build build
```

---

# Executables

The project currently defines three executable targets:

### Exchange

```bash
./build/src/exchange
```

Main exchange executable.

### Trader Terminal

```bash
./build/tools/trader_terminal
```

Client-side trading tool.

### Liquidity Bot

```bash
./build/tools/liquidity_bot
```

Liquidity-provider / market-making tool.

These executable targets are defined directly in the project's CMake configuration.

> **Note:** Some executable components are currently scaffolding and will evolve as the exchange implementation progresses.

---

# Design Goals

The project is being developed around several core goals:

### 1. Correctness

Orders must be matched according to well-defined exchange rules.

### 2. Predictable Latency

Avoid unnecessary operations that introduce unpredictable latency.

### 3. Efficient Memory Usage

Pre-allocate important resources and minimize runtime allocation.

### 4. Cache Efficiency

Design data structures with CPU cache behavior in mind.

### 5. Low Synchronization Overhead

Use specialized concurrency primitives such as SPSC queues where the communication model allows it.

### 6. Maintainable Architecture

Keep exchange logic separated into:

```text
Core utilities
      ↓
Order book
      ↓
Matching engine
      ↓
Trading tools
```

---

# Current Implementation

### Implemented / In Progress

* [x] C++20 project setup
* [x] CMake build system
* [x] Core type definitions
* [x] Custom memory pool
* [x] SPSC queue
* [x] Order representation
* [x] Price levels
* [x] Order book
* [x] Order indexing
* [x] Bid/ask management
* [x] FIFO matching at price levels
* [x] Trade generation
* [x] Order cancellation
* [x] Order modification
* [x] Multiple order types
* [x] Stop-order handling
* [x] Iceberg order handling
* [x] Post-only validation
* [x] FOK liquidity validation
* [ ] Production-grade networking
* [ ] Full external client interface
* [ ] Comprehensive benchmarking
* [ ] Production-grade persistence
* [ ] Distributed deployment

---

# Why Build an Exchange From Scratch?

A stock exchange is a great systems-programming problem because it combines multiple difficult areas in one system:

```text
              Stock Exchange
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Algorithms    Memory       Concurrency
       │            │            │
       ▼            ▼            ▼
 Order Matching   Pools       Atomics
 Order Book      PMR         SPSC Queue
 FIFO            Cache       Memory Ordering
       │            │            │
       └────────────┼────────────┘
                    ▼
             Low-Latency System
```

Instead of treating C++ as simply a language for writing business logic, this project explores how **data structures, memory layout, CPU caches, allocation strategies, and concurrency primitives directly affect system performance**.

---

# Learning Objectives

This project is being used to develop practical understanding of:

* Electronic trading systems
* Limit order books
* Matching algorithms
* Price-time priority
* Market microstructure
* Low-latency C++
* Memory management
* Custom allocators
* Lock-free programming
* Atomic memory ordering
* CPU cache behavior
* False sharing
* Data-oriented design
* CMake
* Systems architecture

---

# Performance Philosophy

The central philosophy of the project is:

> **Don't optimize blindly. Understand where latency comes from first.**

For a latency-sensitive system, seemingly small decisions matter:

```text
malloc/free
    ↓
allocator overhead
    ↓
unpredictable latency

vs.

Pre-allocated memory
    ↓
free-list allocation
    ↓
predictable hot-path behavior
```

Similarly:

```text
Mutex
  ↓
Lock contention
  ↓
Context / synchronization overhead

vs.

SPSC Queue
  ↓
Atomic synchronization
  ↓
Specialized producer-consumer communication
```

The project therefore focuses on understanding these trade-offs rather than simply adding optimizations without measuring them.

---



# Disclaimer

This is an educational systems-programming project.

It is **not connected to a real stock exchange or brokerage**, and it does not execute real financial transactions.

The primary objective is to explore the engineering principles behind high-performance exchange infrastructure.

---
