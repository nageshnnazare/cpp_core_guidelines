# Error handling, constants, templates, C-style, standard library (headlines)

**Source:** [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) — sections **E**, **Con**, **T**, **CPL**, **SL**, plus cross-links.

This file is a **short map**; each section in the official document is large. Use this as a **reading checklist**, then jump to the linked anchors on the official site.

---

## E: Error handling (themes)

- **Exceptions** for failures that prevent meeting **postconditions** / invariants (**I.10**).  
- Prefer **RAII** so stack unwinding still releases resources (**R.1**, **E.6** “minimize `try`” by using constructors / scope guards).  
- Use **`noexcept`** where appropriate (move operators on types stored in **`vector`**, destructors).  
- Do not use **error codes alone** for things callers **ignore** silently — combine with **`[[nodiscard]]`** or types that force handling.

**Why:** The guidelines want **impossible-to-ignore** error paths or **exceptional** propagation, not stale return-code conventions that every second caller skips.

```cpp
#include <fstream>
#include <stdexcept>
#include <vector>

// E: exceptions + RAII — vector cleans up on stack unwind if open throws
void load(const char* path) {
    std::ifstream in(path);
    if (!in) {
        throw std::runtime_error(path);
    }
    std::vector<char> buf(std::istreambuf_iterator<char>(in), {});
    (void)buf;
}

// [[nodiscard]] forces caller to acknowledge error channel (when not using exceptions)
enum class ParseError { Ok, BadFormat };
[[nodiscard]] ParseError try_parse(const char* s, int& out);
```

---

## Con: Constants and immutability (themes)

- Prefer **`const`** on data that does not change after initialization.  
- **`constexpr`** for values computable at **compile** time — enables use in **array bounds**, templates, and clearer intent (**P.5**).  
- **Thread-safe `const` member functions** may still need **synchronization** or **`mutable mutex`** when internal caches exist (see also your notes on **EMC++** Item 16).

```cpp
#include <mutex>
#include <string>

// Con: logical const with internal synchronization
class Memo {
    mutable std::mutex mtx_;
    mutable std::string cache_;
public:
    std::string expensive() const {
        std::lock_guard<std::mutex> lock(mtx_);
        if (cache_.empty()) {
            cache_ = "computed once per lock";
        }
        return cache_;
    }
};
```

---

## T: Templates and generic programming (themes)

- **Concepts** (C++20) or **`static_assert`** with **`type_traits`** to constrain template parameters (**I.9**).  
- Avoid **over-constraining** (requiring `>` when `!=` would suffice) to keep types usable.  
- Prefer **algorithms** and **ranges** over raw loops when they express intent (**P.1**, **P.3**).

```cpp
#if __cplusplus >= 202002L
#include <concepts>

template<std::integral T>
constexpr T twice(T x) { return 2 * x; }
#endif

#include <algorithm>
#include <vector>

void sort_ints(std::vector<int>& v) {
    std::sort(v.begin(), v.end());  // T / SL: algorithm over raw loop
}
```

---

## CPL: C-style programming (themes)

- Prefer **`std::array` / `vector` / `span`** over pointer + length C arrays.  
- Minimize **`malloc`/`free`**, **`printf` variadic** style, and unchecked casts — they fight the **type** and **bounds** profiles.

```cpp
#include <array>
#include <vector>

#if __cplusplus >= 202002L
#include <span>

// CPL: prefer typed, sized buffers
void fill(std::span<int> s) {
    for (int& x : s) {
        x = 0;
    }
}

void demo() {
    std::array<int, 4> a{};
    fill(a);
    std::vector<int> v(10);
    fill(v);
    // int* p = (int*)malloc(40); fill({p, 10});  // avoid as primary style
}
#endif
```

---

## SL: Standard library (themes)

- Prefer **`string_view`** (non-owning) for read-only string parameters when lifetime is clear; watch **dangling** if the view outlives the string.  
- Prefer **`optional`**, **`variant`**, **`any`** (when justified) over sentinel values and `void*`.  
- Know **iterator invalidation** rules for containers you use heavily.

```cpp
#include <optional>
#include <string>
#include <string_view>

// SL: string_view is non-owning — ensure string outlives the view
std::optional<int> parse_int(std::string_view s) {
    // simplified illustration only
    (void)s;
    return {};
}

void danger() {
    std::string_view v;
    {
        std::string tmp = "hello";
        v = tmp;  // BAD: v dangles after block — tmp destroyed
    }
    (void)v;
}
```

---

## Architectural sections (A, NR, FAQ)

- **A:** Large-scale structure (modules, layers) — read when designing subsystems.  
- **NR:** “Non-rules” — myths the guidelines **explicitly** do not endorse (useful to debunk team folklore).  
- **FAQ:** Answers to common objections (“exceptions are slow”, etc.) with nuance.

---

## Synthesis

| Area | Default habit |
|------|----------------|
| Errors | **Exceptions + RAII** or enforced **nodiscard** types |
| Constants | **`const` by default**, **`constexpr` when possible** |
| Templates | **Constraints** at the API |
| C-style | **Replace** with typed, bounds-safe abstractions |
| STL | **Prefer** standard vocabulary over custom wheels |

You have completed the local overview set. Return to [README.md](README.md) for the file index.
