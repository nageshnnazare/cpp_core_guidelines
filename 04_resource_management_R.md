# Resource management (R.1–R.5, R.10–R.24, R.30–R.37)

**Source:** [C++ Core Guidelines — R: Resource management](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

---

## R.1: RAII — manage resources with handles

**Idea:** Pair **acquire** / **release** APIs (`fopen`/`fclose`, `lock`/`unlock`, `new`/`delete`) belong inside **constructors** / **destructors** (or deleters) of small classes so **all paths** (returns, exceptions) clean up automatically.

**Bad pattern:** manual `lock()` … `unlock()` with early `return` or throw in the middle → **deadlock** or **skipped unlock**.

**Good pattern:** `std::lock_guard` / `std::unique_lock`, **`unique_ptr`** with custom deleter, **`Port`** wrapper in the guidelines’ `send()` example.

**See also:** **CP.20** — never raw `lock`/`unlock` without RAII.

```cpp
#include <mutex>

// R.1: RAII for lock + unique_ptr (pattern from guidelines)
void send(std::unique_ptr<int> x, std::mutex& m) {
    std::lock_guard<std::mutex> guard(m);
    (void)x;
}  // unlock + delete (via ~unique_ptr) on all paths
```

---

## R.2 / R.14: Raw pointers vs arrays; prefer `span`

**Idea:** `void f(int* p, int n)` does not couple length to pointer in the type system. Prefer **`gsl::span<int>`** or **`std::span`** (C++20) for non-owning **contiguous views** with size, or owning **`std::vector`**.

**R.2 nuance:** A raw **`T*`** can still mean **a single object** (not an array) — but then document it; **`not_null`** if non-null.

```cpp
#include <vector>

#if __cplusplus >= 202002L
#include <span>

void print_all(std::span<const int> s) {  // R.14: length bundled with pointer
    for (int x : s) { (void)x; }
}

void caller_span() {
    int a[] = {1, 2, 3};
    print_all(a);
    std::vector<int> v = {4, 5};
    print_all(v);
}
#endif
```

---

## R.3 / R.4: Raw pointers and references are **non-owning** by default

**Idea:** Unless marked otherwise (smart pointer, **`owner<T*>`** in GSL vocabulary), **`T*`** and **`T&`** do **not** convey **delete responsibility**. Owning raw **`new`** results are especially discouraged.

**Class design:** Two raw **`T*`** members without documentation are ambiguous; make ownership **`unique_ptr`**, **`shared_ptr`**, or annotate **`owner<T*>`** for human/tools.

```cpp
#include <memory>

// R.3: owning vs non-owning made explicit
struct Good {
    std::unique_ptr<int> owned;  // deletes in destructor
    const int* observes{};       // does not delete
};
```

---

## R.5: Prefer scoped objects; avoid unnecessary heap allocation

**Idea:** Stack objects are automatic and cache-friendly; heap is for **dynamic size** or **shared ownership** lifetimes.

```cpp
// R.5: small buffer on stack
void work(int n) {
    std::vector<int> buf(static_cast<std::size_t>(n));  // heap only when n large
    (void)buf;
}
```

---

## R.10 / R.11: Avoid `malloc`/`free`; avoid naked `new`/`delete`

**Idea:** Prefer **`vector`**, **`make_unique`**, **`make_shared`**, containers — so type and **count** stay together and exceptions cannot skip **`delete`**.

```cpp
#include <memory>

// R.11: prefer factory
auto p = std::make_unique<int>(42);
```

---

## R.12 / R.13: One allocation per statement; chain immediately to owner

**Idea:** **`f(shared_ptr<T>(new T), g())`** can interleave evaluation so **`g()`** throws **after** `new` but **before** the `shared_ptr` ctor → **leak**. Fix: **`make_shared<T>()`** or put **`new`** in a separate statement.

```cpp
#include <memory>

struct Widget {};

int maybe_throw();

// Risky (R.12): subexpression order can leak if maybe_throw throws after new
// void bad(std::shared_ptr<Widget> a, int b);

void caller() {
    // bad(std::shared_ptr<Widget>(new Widget{}), maybe_throw());

    auto w = std::make_shared<Widget>();  // safe: single allocation + no interleaving
    (void)w;
}
```

---

## R.20–R.24: Smart pointers

| Rule | Summary |
|------|---------|
| **R.20** | Use **`unique_ptr`** or **`shared_ptr`** to express **ownership**. |
| **R.21** | Prefer **`unique_ptr`** unless you truly need **shared** ownership. |
| **R.22** | Prefer **`make_shared`** (exception safety + often one allocation). |
| **R.23** | Prefer **`make_unique`**. |
| **R.24** | Use **`weak_ptr`** to break **`shared_ptr`** cycles. |

```cpp
#include <memory>

struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node>   prev;  // R.24: break cycle if both directions used shared_ptr
};
```

---

## R.30–R.37: Smart pointers **as parameters** (lifetime semantics)

**Idea:** Use the **smart pointer type** in the signature **only** when the function participates in **ownership** or **reseat** semantics:

- **`unique_ptr<Widget>`** by value → function **consumes** exclusive ownership.  
- **`unique_ptr<Widget>&`** → may **replace** what the caller holds.  
- **`shared_ptr<Widget>`** by value → **extends** shared lifetime for the duration of the call / stored copy.  
- **`shared_ptr<Widget>&`** → may reseat the caller’s **`shared_ptr`**.

**R.37:** Do not extract a raw **`T*`** from a **`shared_ptr`**, pass it across code that might **reset** the `shared_ptr`, then use the raw pointer — classic **dangling**.

```cpp
#include <memory>

struct Widget { int value{}; };

void use_after_reset() {
    auto sp = std::make_shared<Widget>();
    Widget* raw = sp.get();
    sp.reset();          // object destroyed
    (void)raw->value;    // R.37: undefined behavior — dangling
}
```

---

## Synthesis

1. **Own** → smart pointer or RAII handle.  
2. **Observe** → **`T&`**, **`T*`**, **`span`**, **`not_null`**, **`weak_ptr`**.  
3. **Never** `new` in a big expression with **other** throws unless **`make_*`** wraps it.

Next: [05_classes_and_hierarchies_C.md](05_classes_and_hierarchies_C.md).
