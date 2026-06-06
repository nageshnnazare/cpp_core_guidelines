# Functions — parameter passing (F.15–F.21)

**Source:** [C++ Core Guidelines — F.call: Parameter passing](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

The guidelines split parameters into roles: **in**, **in-out**, **out**, **move-from (“consume”)**, and **forward**. The official document includes summary tables; this file explains the **ideas** and the most common rules.

---

## F.15: Prefer simple, conventional techniques

**Idea:** “Clever” micro-optimizations (exotic **`T&&`** on every parameter) confuse readers and often **do not** help performance. **Measure** if you depart from conventions, and **document** why.

**Exception:** When expressing **`shared_ptr`** lifetime semantics, follow **R.34–R.36** instead of the generic F.16 table.

```cpp
// F.15: conventional “in” parameters — easy to read, easy to check
#include <string>

void consume(const std::string& name, int id, double weight_kg);
```

---

## F.16: “In” parameters — by value vs `const T&`

**Rule of thumb:**

- **Cheap to copy** (roughly ≤ 2–3 pointer-sized words, architecture-dependent): pass **by value** (`int`, `double`, small trivial types).  
- **Potentially expensive** (e.g. `std::string`, large `struct`): pass **`const T&`**.

**Why `const T&` for large `T`:** avoids copying the object at the call site for lvalues; still binds to temporaries.

**Why by-value for small `T`:** no extra indirection; **clear** “function will not mutate caller’s object” for scalars.

**Anti-pattern from the guidelines:** `void f2(string s)` as a blanket style — copying a string every call can be costly compared to `const string&`.

**Advanced cases** (only when needed):

- Unconditional **sink** from rvalue: overload or **`T&&`** + **`std::move`** internally (**F.18**).  
- **Perfect-forwarding** template (**F.19**) for generic wrappers.

The document warns: most “pass by `&&` for speed” rumors are **false or brittle**.

**References:** A **`T&`** is never “null” in valid C++; if you need optional, use **`T*`**, **`std::optional`**, or a sentinel.

```cpp
#include <string>

void f1(const std::string& s);  // OK: large type, no copy on lvalue arg
// void f2(std::string s);       // often worse for large s (extra copy)

void f3(int x);                 // OK: cheap — pass by value
// void f4(const int& x);       // unnecessary indirection for int

void sink(std::string&& s);     // F.18 style: consumer of rvalue string
```

---

## F.17: “In-out” parameters — `T&` (non-const)

**Idea:** If the function **mutates** an object provided by the caller and the caller supplies the storage, use **non-const reference** (or pointer if null is allowed). The signature advertises mutation.

```cpp
#include <cctype>
#include <string>

void normalize(std::string& s) {  // in-out: caller sees updated s
    for (char& c : s) {
        c = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
    }
}
```

---

## F.18: “Consume” parameters — `std::move` into your storage

**Idea:** If the function **always** takes ownership or **always** moves from an rvalue into members, **`T&&`** overloads (or pass-by-value + move, per **Effective Modern C++** Item 41 trade space) express that contract.

```cpp
#include <memory>
#include <utility>

struct Widget {};

class Store {
    std::unique_ptr<Widget> w_;
public:
    void reset_to(std::unique_ptr<Widget>&& nw) {
        w_ = std::move(nw);  // sink: takes ownership from rvalue unique_ptr
    }
};
```

---

## F.20: Prefer return values over output parameters

**Idea:** Multiple **`out` parameters** via non-const references are easy to **swap** mentally and harder to compose. Prefer **`struct` return types**, **`std::optional`**, **`std::tuple`**, or **move** from local values **by return** (NRVO / move).

**Caveat:** Legacy APIs and huge structs may still use out-params; then be **explicit** (return `bool` success + write to out-param, etc.).

```cpp
#include <optional>
#include <string>

// Prefer: result type carries success + value
std::optional<std::string> read_line();

// vs bool read_line(std::string& out);  // easy to ignore bool return
```

---

## F.21: To return multiple “out” values, prefer returning a `struct` or `tuple`

**Idea:** Named members in a **`struct`** are self-documenting; **`tuple`** is acceptable when field names are obvious at the call site (`tie` / structured binding).

```cpp
#include <iosfwd>

struct ParseResult {
    int lines{};
    int errors{};
};

ParseResult parse_stream(std::istream& in);  // F.21 — define in .cpp with <istream>

#if __cplusplus >= 201703L
#include <tuple>
std::tuple<int, int> parse_compact(std::istream& in);
void demo() {
    auto [lines, errors] = parse_compact(std::cin);
    (void)lines;
    (void)errors;
}
#endif
```

---

## Smart-pointer parameters (cross-reference R.30–R.37)

Summary intent:

- **`unique_ptr<T>`** by value → callee **takes ownership**.  
- **`unique_ptr<T>&`** → callee may **reseat** (replace) the pointer.  
- **`shared_ptr<T>`** by value → **shared** ownership for the callee’s lifetime of the parameter object graph.  
- **`const shared_ptr<T>&`** → rare; read the guideline text carefully (possible retained refcount — marked with discussion in the source).

Do not pass **`T*`** derived from a **`shared_ptr`** and then let the `shared_ptr` die while the callee still holds the raw pointer (**R.37** — aliasing).

```cpp
#include <memory>

struct Widget {};

void take_owner(std::unique_ptr<Widget> w);  // R.32: callee assumes ownership

void share_user(std::shared_ptr<Widget> w) {  // R.34: share for duration of call
    (void)w;
}
```

---

## Synthesis

| Role | Preferred signature pattern |
|------|------------------------------|
| Read-only large object | `const T&` |
| Read-only small / trivial | `T` by value |
| Mutate caller’s object | `T&` |
| Sink / move from rvalue | `T&&` or value + move (design choice) |
| Ownership transfer | `unique_ptr<T>`, `shared_ptr<T>` (see **R.**) |
| Multiple outputs | `struct` / `tuple` / return value |

Next: [04_resource_management_R.md](04_resource_management_R.md).
