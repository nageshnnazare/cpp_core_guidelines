# Guidelines Support Library (GSL) — why it exists

**Source:** [C++ Core Guidelines — GSL: Guidelines support library](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

The GSL is a **small set of types and functions** designed to make Core Guidelines rules **expressible** and **checkable**. Implementations include Microsoft’s **`Microsoft/GSL`**, which tracks the specification closely enough for everyday use.

**Important:** GSL types are often **zero-overhead wrappers** at runtime but **rich** for static analysis (`not_null`, `span` bounds, etc.).

---

## `gsl::span<T>` / `std::span<T>` (C++20)

**Problem:** `T* p, std::size_t n` carries no **typed** link between pointer and length.

**Solution:** **`span<T>`** is a **non-owning** view of contiguous elements with **size**. Algorithms can bounds-check in debug, and APIs document “**n elements here**” without copying.

**Contrast:** **`vector`** **owns**; **`span`** **observes** (caller keeps storage alive).

```cpp
#include <vector>

#if __cplusplus >= 202002L
#include <span>

// Subspan without losing bounds information
void head(std::span<const int> s, std::size_t n) {
    for (int x : s.first(n)) {
        (void)x;
    }
}

void demo() {
    std::vector<int> v = {10, 20, 30, 40};
    head(v, 2);
}
#endif
```

---

## `gsl::not_null<T*>`

**Problem:** Comments saying “must not be null” are not enforced.

**Solution:** **`not_null<T*>`** cannot be constructed from **`nullptr`** without a forced cast lie. Signals **precondition** to humans and tools.

**Do not use** when **`nullptr`** is meaningful — use **`T*`**, **`optional`**, or **`T&`** when reference always valid.

```cpp
// With Microsoft GSL installed:
// #include <gsl/pointers>
// void bump(gsl::not_null<int*> p) { ++*p; }
//
// int x = 1;
// bump(&x);
// bump(nullptr);  // compile error

// Portable pattern without GSL: reference parameter
void bump_ref(int& r) { ++r; }
```

---

## `gsl::owner<T*>`

**Problem:** Raw **`T*`** members do not say **owning vs observing**.

**Solution:** **`owner<T*>`** is an **annotation** (often typedef-thin) marking **delete responsibility** for tools and readers. Prefer migrating to **`unique_ptr`** when you control the code.

```cpp
// Prefer unique_ptr (self-documenting ownership)
#include <memory>
struct Widget {
    std::unique_ptr<int> owned{std::make_unique<int>(0)};
    const int* observes{};  // non-owning
};
```

---

## `gsl::zstring` / `czstring`

**Problem:** `char*` might mean **one char**, **NTBS**, or **buffer**.

**Solution:** Named alias **`zstring`** documents **null-terminated C string** expectation (**I.13** exception path).

```cpp
// GSL: gsl::czstring name = "hello";  // documents NTBS from C API
// void greet(gsl::czstring msg);
```

---

## `Expects` / `Ensures` (contract vocabulary)

Depending on your **contract-checking** setup, these macros or functions express **preconditions** and **postconditions** at function entry/exit. C++ standard **contracts** are evolving; teams may use **custom** assertions until then.

```cpp
#include <cassert>
#include <vector>

// Minimal portable pattern (not the same as ISO contracts, but same idea)
void at(std::vector<int>& v, std::size_t i) {
    assert(i < v.size());  // precondition
    int& x = v[i];
    x += 1;
    assert(v[i] == x);     // trivial postcondition illustration only
}
```

---

## `finally` / scope guards

When a resource is not naturally an object with a destructor, a **scope guard** pattern (GSL **`finally`**) runs cleanup at scope exit — second-best to **RAII** but better than scattered manual cleanup.

```cpp
#include <utility>

// C++11: scope guard by hand (simplified)
template<typename F>
struct ScopeExit {
    F f;
    explicit ScopeExit(F fn) : f(std::move(fn)) {}
    ~ScopeExit() { f(); }
};

// Usage: ScopeExit guard([&]{ /* cleanup even on throw */ });
```

---

## Practical adoption

1. Introduce **`span`** on **new** APIs first (file I/O buffers, parsers).  
2. Use **`not_null`** at **trust boundaries** (after null checks, before deep call trees).  
3. Turn on **analyzers** that understand GSL annotations if your organization uses them.

Return to [README.md](README.md).
