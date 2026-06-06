# Classes, special members, and hierarchies (selected C.*)

**Source:** [C++ Core Guidelines — C: Classes and class hierarchies](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

This file highlights rules that prevent **slicing**, **undefined deletion through base**, and **silent performance** degradation from special-member rules.

---

## C.30–C.33: Destructors and owning members

**C.30:** If a class needs **non-trivial** work at destruction (flush, unlock custom resource), declare a **user-defined destructor**.

**C.31:** Every resource acquired by the class must be **released** by the destructor (or member destructors) — same RAII story as **R.1**.

**C.32–C.33:** Raw **`T*`** members might be owning — if so, **rule of five** / **`unique_ptr`** applies; **`unique_ptr` with incomplete type** needs **out-of-line destructor** (same lesson as *Effective Modern C++* Pimpl + Item 22).

```cpp
#include <memory>

// C.33: owning raw pointer is ambiguous — prefer unique_ptr
struct Prefer {
    std::unique_ptr<int> data{std::make_unique<int>(0)};
};
```

---

## C.35: Base class destructor — public `virtual` or protected non-`virtual`

**Idea:** If code may **`delete` through `Base*`** pointing at a **`Derived`**, **`Base`’s destructor must be `virtual` and public`**. Otherwise **`Derived`’s destructor never runs** → undefined behavior and leaks of **`Derived`** members.

If you **forbid** deleting via **`Base*`** (object identity always concrete **`Derived`***), you can use **protected non-virtual destructor** — rare in application code; know what you are doing.

**Bad:**

```cpp
struct Base {
    // public non-virtual ~Base() — deleting Derived through Base* is UB
    virtual void foo() = 0;
};
```

**Good (polymorphic delete):**

```cpp
struct Base {
    virtual ~Base() = default;
    virtual void foo() = 0;
};
```

---

## C.36 / C.37: Destructors must not fail; prefer `noexcept`

**Idea:** Throwing from destructors during stack unwinding leads to **`std::terminate`**. Destructors should **`noexcept`** (implicitly for many destructors in modern code).

---

## C.60–C.66: Copy and move operations (headlines)

- **Copy assign:** non-`virtual`, take **`const T&`**, return **`T&`**, handle **self-assignment**.  
- **Move assign:** non-`virtual`, take **`T&&`**, leave source in **valid** state, **`noexcept`** when possible (important for **`vector` reallocation** and swap-based algorithms).

```cpp
#include <vector>

// C.62: self-assignment safe copy-assign pattern
class Buffer {
    std::vector<int> data_;
public:
    Buffer& operator=(const Buffer& rhs) {
        if (this == &rhs) {
            return *this;
        }
        Buffer tmp(rhs);
        data_.swap(tmp.data_);
        return *this;
    }
};
```

---

## C.67: A polymorphic class should suppress **public** copy/move

**Idea:** If **`virtual`** functions imply runtime type varies, **slicing** on copy is a footgun: **`Derived`** copied into **`Base`** loses derived state. Prefer **`= delete`** public copy/move, or **`protected`** copy/move if derived classes need them for cloning patterns.

**Connects to:** Meyers’ **slicing** discussion (Effective Modern C++ Item 41) and Core **I.4** (strong types / intent).

```cpp
#include <string>

// C.67: polymorphic base — public slicing is dangerous
struct Animal {
    virtual ~Animal() = default;
    virtual const char* name() const = 0;

    Animal(const Animal&) = delete;
    Animal& operator=(const Animal&) = delete;
};

struct Dog : Animal {
    std::string nickname{"pluto"};
    const char* name() const override { return nickname.c_str(); }
};

// void foo(Animal a);  // BAD: pass-by-value slices Dog to Animal subobject
void bar(const Animal& a);  // OK: reference — no slicing
```

---

## Abstract interfaces (C.hier — headline)

**Idea:** Interfaces are often **abstract classes** with **pure virtual** functions and a **public virtual destructor** (or protected non-virtual pattern if no delete-through-base). **Concrete data** in interface bases is brittle.

---

## Synthesis

| Situation | Rule of thumb |
|-----------|----------------|
| Polymorphic base | **Virtual public dtor** (**C.35**) |
| Polymorphic type | **Delete or protect** public copy (**C.67**) |
| Destructor body | **Never throw** (**C.36**) |
| Owning raw pointer member | **Stop** — use **`unique_ptr`** (**R.3**, **C.33**) |

Next: [06_concurrency_CP.md](06_concurrency_CP.md).
