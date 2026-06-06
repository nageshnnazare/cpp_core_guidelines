# Interfaces (I.1–I.13 and related)

**Source:** [C++ Core Guidelines — I: Interfaces](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

Interfaces are **contracts** between callers and implementations: parameter types, return types, preconditions, postconditions, and **lifetime** obligations.

---

## I.1: Make interfaces explicit

**Idea:** Ambiguous signatures force readers to read **bodies**. Prefer types that encode **legal states** (see **P.1**).

```cpp
// Bad: what are the bools?
void draw_rectangle(int x, int y, int w, int h, bool fill, bool outline);

// Better: named aggregate (I.23 / I.24 territory too)
struct DrawStyle { bool fill{}; bool outline{true}; };
void draw_rectangle(int x, int y, int w, int h, DrawStyle style);
```

---

## I.2 / I.3: Avoid non-`const` global variables and singletons

**Idea:** Mutable globals hide **data flow**, complicate **testing**, and interact badly with **threads** (ordering, initialization). Singletons are often globals in disguise and fight **testability** and **lifetime clarity**.

**Practical stance:** If you need process-wide state, narrow it, make initialization explicit, and document **thread** rules (often: construct once, then read-only, or guard with synchronization).

```cpp
// I.2: hidden coupling — every translation unit can mutate
// int g_counter = 0;  // prefer context object or dependency injection

// I.3: singletons complicate testing (hard to swap implementation)
// Widget& instance();  // often better: pass Widget& into functions that need it
```

---

## I.4: Make interfaces precisely and strongly typed

**Idea:** **`int`** for everything invites mix-ups (index vs count, meters vs feet). Use **distinct types**, **`enum class`**, **`std::chrono`**, units libraries where appropriate.

```cpp
#include <chrono>

using Seconds = std::chrono::duration<double>;

void wait(Seconds d);  // clear unit; not wait(double) meaning unknown
```

---

## I.5 / I.7: State preconditions and postconditions

**Idea:** A **precondition** is what must be true **before** the call (e.g. `index < size()`). A **postcondition** is what the function guarantees **after** it returns successfully.

The guidelines discuss **`Expects`** / **`Ensures`** as vocabulary types / macros for expressing these in code (availability depends on your contract-checking setup; C++26 contracts are evolving).

**Why it matters:** Without stated assumptions, every caller must **reverse-engineer** invariants from implementation bugs.

```cpp
// Preconditions / postconditions as comments (minimum); tools may use Expects/Ensures
// Precondition: index < v.size()
// Postcondition: v unchanged
int at_read(const std::vector<int>& v, std::size_t index);
```

---

## I.6 / I.8: Prefer `Expects` / `Ensures` when you use that vocabulary

**Idea:** Centralized contract checking is easier to **toggle** (off in release, on in QA) and to **grep** than ad-hoc `if (!p) return;` scattered differently in each function.

```cpp
// Illustrative (macro names depend on your contract library)
// void f(int* p) {
//     Expects(p != nullptr);
//     *p = 1;
//     Ensures(*p == 1);
// }
```

---

## I.9: Templates — document parameters (concepts in C++20)

**Idea:** Template APIs without constraints accept **anything** and fail with **inside** the body with long errors. **Concepts** (or static asserts) document and enforce **semantic** requirements (`RandomAccessIterator`, `Sortable`, etc.).

```cpp
#if __cplusplus >= 202002L
#include <concepts>
#include <iterator>

template<std::random_access_iterator It>
void shuffle_slice(It first, It last) {
    (void)first;
    (void)last;
}
#endif
```

---

## I.10: Use exceptions to signal failure to perform a required task

**Idea:** Reserve exceptions for **exceptional** control flow: cannot meet postconditions / cannot establish invariants. Do not use them for ordinary control flow; combine with **RAII** so stack unwinding still releases resources.

(Your project may ban exceptions in embedded code — that becomes a **documented profile** exception, not a dismissal of the general rule.)

```cpp
#include <fstream>
#include <stdexcept>
#include <string>

std::string read_file(const std::string& path) {
    std::ifstream in(path);
    if (!in) {
        throw std::runtime_error("cannot open " + path);
    }
    return std::string(std::istreambuf_iterator<char>(in), {});
}
```

---

## I.11: Never transfer ownership by raw `T*` or `T&`

**Idea:** A raw pointer does not say **who deletes**. Ownership transfer should use **`std::unique_ptr`** (exclusive), **`std::shared_ptr`** (shared), or a **domain-specific handle** with clear move/copy semantics.

Returning **`new`’d** raw pointers forces every caller to remember **`delete`** and composes badly with exceptions.

```cpp
#include <memory>

// Bad: raw owning return — who deletes? exception path?
// Gadget* make_gadget();

// Good: unique_ptr transfers ownership clearly
struct Gadget {};
std::unique_ptr<Gadget> make_gadget() {
    return std::make_unique<Gadget>();
}
```

---

## I.12: A pointer that must not be null — `not_null`

**Idea:** If **`nullptr`** is invalid, encode it: **`gsl::not_null<T*>`** (or references where **`nullptr`** is impossible by type). Callers then **cannot** pass null without a cast lie; implementations can **assert less** redundantly.

**Contrast:** Optional absence should be **`std::optional<T>`** (by value), **`T*`** with documented null meaning, or overload sets — not `not_null`.

```cpp
// With Microsoft GSL: #include <gsl/gsl>
// void use(gsl::not_null<int*> p) { *p += 1; }
// int x = 1; use(&x);   // OK
// use(nullptr);         // ill-formed / caught

// Standard-only equivalent at call boundary:
void use(int& r) { r += 1; }  // reference: no null
```

---

## I.13: Do not pass an array as a single pointer

**Idea:** `void f(int* p, int n)` does not let the type system tie **`p`** to **`n`**. Prefer **`std::span`**, **`std::array`**, **`std::vector`**, or pairs that tools can check (**bounds profile**).

**Exception:** C-style **zstrings** (`char` null-terminated) remain a reality; the GSL introduces **`zstring`** as a **named convention** for “points to NTBS.”

```cpp
#include <vector>

// Bad: p and n not coupled in the type system
// void process(int* p, std::size_t n);

#if __cplusplus >= 202002L
#include <span>
// Good (C++20): span carries count
void process(std::span<const int> s) {
    for (int x : s) { (void)x; }
}

void caller_span() {
    std::vector<int> v = {1, 2, 3};
    process(v);  // implicit span from contiguous container
}
#endif
```

---

## I.23 / I.24: Few parameters; avoid ambiguous adjacent parameters

**Idea:** Many positional **`bool`**s or two **`int`s** that can be **swapped** by mistake (`f(height, width)` vs `f(width, height)`) are bug farms. Use **structs**, **`enum class`**, or **strong typedefs**.

```cpp
// Bad: easy to swap width/height at call site
// void resize(int width, int height);

struct Size { int width{}; int height{}; };
void resize(Size s);  // named members — harder to swap by accident
```

---

## Synthesis

| Guideline | Habit |
|-----------|--------|
| I.4 | Strong types at **boundaries** |
| I.5–I.8 | Write **contracts** (even if only comments at first) |
| I.11 | **Smart pointers** for ownership transfer |
| I.12 | **`not_null`** when null is invalid |
| I.13 | **`span`** / containers for **length + pointer** |

Next: [03_functions_F.md](03_functions_F.md).
