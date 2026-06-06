# Philosophy (P.1–P.13)

**Source:** [C++ Core Guidelines — P: Philosophy](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

Philosophical rules are often **not** mechanically checkable by themselves, but they motivate hundreds of concrete rules that **are** checkable.

---

## P.1: Express ideas directly in code

**Idea:** Compilers and most tooling **do not** read comments or design PDFs. Semantics live in **types, function signatures, and control flow**.

**Example (from the guidelines):** A member `Month month() const` tells you the return type is a **`Month`** and the call does not mutate `Date`. An `int month()` leaves meaning and const-correctness ambiguous.

**Example:** Re-implementing `std::find` with a raw index loop hides the intent “find position of value.” Using **`find(begin(v), end(v), val)`** states the algorithm directly.

**Example:** `change_speed(double)` does not say whether the value is absolute or delta, or which units apply. A **`Speed`** type (possibly with unit literals) makes illegal states unrepresentable at compile time.

**Practice:** Prefer **strong types** (newtypes), **`enum class`**, **`std::optional`**, **`std::variant`**, and **named factories** over primitive soup.

```cpp
// P.1: prefer types that carry meaning (illustrative)
enum class Month : unsigned { Jan = 1, Feb, /* ... */ };

class Date {
public:
    Month month() const;   // clear: Month, read-only
    // int month();        // unclear; may or may not mutate
private:
    /* ... */
};

// Same idea: say "find" not manual index loop
void use(std::vector<std::string>& v, const std::string& val) {
    auto it = std::find(v.begin(), v.end(), val);
    (void)it;
}
```

---

## P.2: Write in ISO Standard C++

**Idea:** The guidelines target **portable, standard** C++. Vendor extensions should be **localized** (wrapped behind interfaces, `#ifdef` only where needed) so the rest of the code base stays standard.

**Why:** Extensions often lack **rigorous** cross-vendor semantics; portability and tooling suffer.

**Note:** “Valid ISO C++” still allows **implementation-defined** and **undefined** behavior — the rest of the document tries to steer you away from those.

```cpp
// P.2: wrap vendor / extension APIs behind a portable interface
#if defined(_WIN32)
void platform_sleep_ms(int ms);  // declared in your .cpp with Windows headers
#else
void platform_sleep_ms(int ms);  // POSIX implementation
#endif
```

---

## P.3: Express intent

**Idea:** Prefer constructs that say **what** you mean: range-`for`, algorithms, `final`/`override`, `= delete`, `[[nodiscard]]`, etc.

Intent overlaps with **P.1** but emphasizes **reader** and **maintainer** cognition, not only compiler checking.

```cpp
// P.3: override/final, = delete, [[nodiscard]] state intent
struct Base { virtual void f() = 0; virtual ~Base() = default; };
struct Derived final : Base {
    void f() override;
};

struct Handle {
    Handle() = default;
    Handle(const Handle&) = delete;
    Handle& operator=(const Handle&) = delete;
};

[[nodiscard]] int compute() { return 42; }  // caller cannot silently ignore return value
```

---

## P.4: Ideally, a program should be statically type safe

**Idea:** Catch as many errors as possible at **compile** time. Where that is impossible, narrow problems to **explicit** runtime checks (contracts, assertions, bounds-checked views).

This is the philosophical umbrella for **profiles (type, bounds, lifetime)** and for discouraging unchecked **`reinterpret_cast`**, C-style varargs, etc.

```cpp
// P.4: catch invalid states at compile time where possible
enum class Color { Red, Green };

void set_color(Color c);  // cannot pass arbitrary int without a cast
```

---

## P.5: Prefer compile-time checking to run-time checking

**Idea:** If a property can be encoded in the **type system** or **`constexpr`**, prefer that to “hope we test every path.”

Examples: `static_assert`, concepts (C++20), array sizes as template parameters, `enum class` instead of loose `int` codes.

```cpp
// P.5: constexpr + static_assert
constexpr int buffer_size = 1024;
static_assert(buffer_size % 16 == 0, "alignment requirement");

// C++20: concept documents requirements at compile time
#if __cplusplus >= 202002L
#include <concepts>
template<std::integral T>
constexpr T square(T x) { return x * x; }
#endif
```

---

## P.6: What cannot be checked at compile time should be checkable at run time

**Idea:** When you must use dynamic behavior, make violations **detectable**: invariants on classes, assertions, sanitizers, fuzzing—not silent corruption.

```cpp
// P.6: explicit runtime check (contracts / assertions are the scalable form)
void take_index(std::vector<int>& v, std::size_t i) {
    if (i >= v.size()) {
        throw std::out_of_range("take_index");
    }
    use(v[i]);
}
```

---

## P.7: Catch run-time errors early

**Idea:** Fail **fast** and **locally** (assertions near the bug) rather than far away with corrupted state. Combine with **P.8** and **R.1** (RAII) so failures do not skip cleanup.

```cpp
// P.7: invariant checked at entry of public member (debug build often uses assert)
class Buffer {
    std::vector<char> data_;
    bool invariant() const { return /* ... */ true; }
public:
    void push(char c) {
        assert(invariant());
        data_.push_back(c);
        assert(invariant());
    }
};
```

---

## P.8: Don’t leak any resources

**Idea:** Every resource (memory, file, lock, handle) should live under **RAII**: acquisition in a constructor (or factory), release in a destructor or **deleter**, with **move** semantics transferring ownership clearly.

This philosophy is spelled out concretely in **R.1** and the smart-pointer section.

```cpp
// P.8: RAII — destructor runs on every exit from scope (including exceptions)
void example(std::mutex& m) {
    std::lock_guard<std::mutex> lock(m);
    might_throw();
}  // unlock here always
```

---

## P.9: Don’t waste time or space

**Idea:** Correctness first, but **avoid gratuitous** copies, allocations, and indirection. When you optimize, **measure**; the guidelines warn against “clever” parameter-passing rumors (see **F.16–F.19**).

```cpp
// P.9: prefer returning by value + move over shared_ptr when sole ownership ends here
std::vector<int> make_data();  // often NRVO / move — measure if hot
```

---

## P.10: Prefer immutable data to mutable data

**Idea:** **`const`**, **`const`&** parameters, **value semantics** where copying is cheap, **concurrent read** without locks when data does not change.

Immutability reduces the reasoning needed for **data races** and unexpected aliasing.

```cpp
// P.10: const reference for read-only input
int sum(const std::vector<int>& v) {
    int s = 0;
    for (int x : v) { s += x; }
    return s;
}
```

---

## P.11: Encapsulate messy constructs

**Idea:** Isolate **undefined-behavior-prone** or platform code behind small, reviewed abstractions (your own port layer, GSL helpers, etc.) instead of scattering casts and intrinsics everywhere.

```cpp
// P.11: one reviewed place for “messy”
std::uint32_t load_le_u32(const unsigned char* p);  // implementation hides endian details
```

---

## P.12: Use supporting tools as appropriate

**Idea:** Static analyzers, sanitizers, code formatters, compilers’ warnings-as-errors — the guidelines are partly written as a **spec for tools**. Humans should not be the only line of defense.

```text
# P.12: typical CI flags (illustrative — adjust for your compiler)
# clang++ -Wall -Wextra -Wpedantic -Werror -fsanitize=address,undefined
```

---

## P.13: Use support libraries as appropriate

**Idea:** Know **your standard library**, your **project’s foundation libraries**, and the **GSL** (`span`, `not_null`, etc.). Reinventing poorly tends to lose safety and clarity.

```cpp
// P.13: standard library over ad-hoc
#include <algorithm>
#include <vector>

bool contains(const std::vector<int>& v, int x) {
    return std::binary_search(v.begin(), v.end(), x);  // requires sorted v — document precondition
}
```

---

## Synthesis

| Rule cluster | One-line habit |
|--------------|----------------|
| P.1, P.3 | **Types and names** carry meaning |
| P.4–P.7 | **Compile-time first**, then **explicit** runtime checks |
| P.8 | **RAII** everywhere |
| P.9 | **Correct**, then **measure** performance |
| P.10–P.11 | **const** / immutability / **wrappers** reduce bugs |
| P.12–P.13 | **Tools + libraries** scale better than policy memos |

Next: [02_interfaces_I.md](02_interfaces_I.md).
