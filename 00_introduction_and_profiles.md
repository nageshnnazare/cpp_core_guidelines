# Introduction, aims, and profiles

**Source:** [C++ Core Guidelines — Introduction](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) (section **In:**).

---

## What the guidelines are for

The document targets **all C++ programmers** who want to write **modern ISO C++** (the live text emphasizes C++17/C++20; most rules still apply to C++11/14). Goals include:

- **More uniform style** across teams and code bases.  
- **Static type safety** and **resource safety** (fewer leaks, fewer dangling uses, fewer logic errors caught only in production).  
- **Simplicity** where possible, with necessary complexity hidden behind **clear interfaces**.  
- **Zero-overhead** abstractions used **well** — the intent is not slower “safe” code, but code that is at least as good as hand-rolled lower-level patterns when measured.

The guidelines explicitly reject being a **minimal subset** of C++ (like a “safe dialect” that throws away expressiveness). They recommend a **small set of library patterns** (including the **Guidelines Support Library**, GSL) so that the most error-prone raw patterns become unnecessary.

---

## In.0: Don’t panic

Rules have **trade-offs**. Before mass-refactoring legacy code, **understand** what a rule buys you for *your* system (real-time, embedded, ABI constraints, etc.). The document expects **gradual** adoption and **local** extensions (stricter rules for a flight-control project, for example).

---

## What the guidelines are *not*

Useful mental model from **In.not**:

- Not a **serial textbook** — browse by link or tool.  
- Not a **replacement** for the ISO standard on corner cases.  
- Not a **step-by-step migration guide** for old code bases (though appendices discuss modernization).  
- Not **orthogonal** — general principles and specific rules overlap on purpose.

---

## Enforcement and “profiles”

Many rules include an **Enforcement** subsection: ideas for **static analyzers**, **compilers**, **reviews**, or occasional **runtime** checks. The philosophy is: rules without enforcement do not scale to huge code bases, but **different communities need different subsets**.

Three foundational **profiles** are named repeatedly:

| Profile | Intent |
|---------|--------|
| **type** | No violations of the type system via unsafe casts, unions used as type-puns, varargs abuse, etc. |
| **bounds** | No out-of-bounds access (arrays, buffers). |
| **lifetime** | No leaks, no double-delete, no use of invalid objects (**`nullptr`**, **dangling** pointers/references). |

Profiles are meant for **tools** and for **communication** (“this module is checked for **bounds**”). They are not the only way to subset the rules.

### Examples: what each profile catches

**Type profile** — reinterpreting bits without a safe story:

```cpp
float f = 3.14f;
// Bad: type-punning through a pointer — UB in many uses
// int bits = *reinterpret_cast<int*>(&f);

std::byte buf[sizeof(float)];
std::memcpy(buf, &f, sizeof(f));  // OK: defined copy of object representation
```

**Bounds profile** — pointer + length not tied in the type system:

```cpp
void sum_bad(int* p, int n) {
    int s = 0;
    for (int i = 0; i <= n; ++i) {  // off-by-one: can read past end — bounds violation
        s += p[i];
    }
}
```

**Lifetime profile** — use after free / double delete:

```cpp
int* leak() {
    int* p = new int{42};
    return p;   // caller must delete — easy to leak (lifetime obligation unclear)
}

// Better: return std::unique_ptr<int> or value type
```

---

## Practical takeaway

1. Read **Philosophy (P.)** once — it explains *why* the rest exists.  
2. Adopt **R.** (resources) and **I.** (interfaces) early — they prevent the largest class of systemic bugs.  
3. Use **static analysis + sanitizer builds** where the guidelines expect tooling rather than human memory.

Next: [01_philosophy_P.md](01_philosophy_P.md).
