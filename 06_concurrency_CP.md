# Concurrency and parallelism (CP.* — selected)

**Source:** [C++ Core Guidelines — CP: Concurrency and parallelism](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

This section complements the **lifetime** and **type** profiles: data races are **undefined behavior** even when single-threaded “logic” looks fine.

---

## CP.20: Use RAII for locks — never raw `lock()` / `unlock()`

**Idea:** Same as **R.1** applied to mutexes: **`lock_guard`**, **`unique_lock`**, **`scoped_lock`** guarantee unlock on **every** exit path.

```cpp
// Bad: easy to forget unlock on return/throw
// mtx.lock(); ... mtx.unlock();

// Good
std::lock_guard<std::mutex> lock(mtx);
// ...
```

---

## CP.21: Use `std::lock` or `std::scoped_lock` for multiple mutexes

**Idea:** Locking **`m1`** then **`m2`** in one thread and **`m2`** then **`m1`** in another invites **deadlock**. **`std::lock(m1, m2)`** acquires both with a deadlock-avoidance algorithm; then adopt ownership into **`lock_guard`** with **`std::adopt_lock`**.

**C++17:** **`std::scoped_lock lk{m1, m2};`** does the common pattern in one line.

```cpp
#include <mutex>

void both(std::mutex& m1, std::mutex& m2) {
    std::scoped_lock lk{m1, m2};  // CP.21: deadlock-safe lock of both
    // critical section
}
```

---

## CP.22: Do not call unknown code while holding a lock

**Idea:** A callback/user code might **block**, **re-enter** your module, or **take locks** in another order → deadlock or latency spikes. Shrink critical sections; copy small data, release lock, then invoke.

```cpp
#include <functional>
#include <mutex>
#include <vector>

std::mutex g_mtx;
std::vector<int> g_data;

void bad(std::function<void()> on_update) {
    std::lock_guard<std::mutex> lock(g_mtx);
    g_data.push_back(1);
    on_update();  // CP.22: user code runs under lock — may deadlock or block
}

void better(std::function<void()> on_update) {
    {
        std::lock_guard<std::mutex> lock(g_mtx);
        g_data.push_back(1);
    }
    on_update();  // lock released first
}
```

---

## CP.31: Prefer passing **small** data to threads **by value**

**Idea:** A **`const&`** to stack data in the parent can **dangle** after the parent thread’s stack frame moves on. **Values** (moves) into the thread lambda capture or thread callable avoid **lifetime races**.

**Cross-reference:** *Effective Modern C++* Item 37 (joinable threads) and Item 40 (`atomic` vs `volatile`).

```cpp
#include <thread>

// CP.31: dangerous — reference to local may dangle when parent returns
void spawn_bad() {
    int payload = 42;
    std::thread t([&payload] {
        (void)payload;  // UB if parent stack is gone
    });
    t.detach();
}

// Better: pass by value into thread
void spawn_ok() {
    int payload = 42;
    std::thread t([payload] { (void)payload; });
    t.join();
}
```

---

## CP.32: Share ownership between threads with `shared_ptr`

**Idea:** When lifetime must extend until **last** user thread finishes, **`shared_ptr`** (with care for **cycles** via **`weak_ptr`**) is the standard vocabulary.

```cpp
#include <memory>
#include <thread>

void worker(std::shared_ptr<int> p) {
    ++*p;
}

void demo() {
    auto data = std::make_shared<int>(0);
    std::thread t(worker, data);  // copy shared_ptr — shared lifetime
    t.join();
}
```

---

## CP.42 / CP.43: Waiting and critical sections

**Idea:** Do not **busy-wait** on atomics without backoff when waiting is long; prefer **condition variables** with predicates for complex waiting. **Minimize** work under mutex — contention kills scalability.

```cpp
#include <condition_variable>
#include <mutex>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> work;

void producer(int x) {
    {
        std::lock_guard<std::mutex> lock(mtx);
        work.push(x);
    }
    cv.notify_one();
}

void consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return !work.empty(); });  // CP.42: wait with predicate
    work.pop();
}
```

---

## CP.44: Name your `lock_guard` / `unique_lock`

**Idea:** Anonymous temporaries like **`lock_guard<mutex>(mtx)`** as a statement by itself can “lock then immediately unlock” at the end of the **full-expression** — a classic bug. Bind to a **named variable** whose scope is the critical section.

```cpp
std::lock_guard<std::mutex> guard(mtx);  // good: name extends to end of block
```

---

## CP.50: Colocate mutex with guarded data

**Idea:** A **`struct`** holding **`mutex`** + **`data`**, or **`synchronized_value<T>`** where available, makes the invariant “access **`data` only under lock**” easier to see and to annotate for tools.

```cpp
#include <mutex>
#include <vector>

// CP.50: mutex colocated with guarded data
struct WorkQueue {
    mutable std::mutex m;
    std::vector<int>   items;

    void push(int x) {
        std::lock_guard<std::mutex> lock(m);
        items.push_back(x);
    }
};
```

---

## Tools (from CP intro material)

- **Thread Sanitizer (TSan)** — high value for finding **data races** in test runs; significant runtime cost.  
- **Static annotation** (e.g. mutex attributes on members) — turns some races into **compile errors** in supported builds.

---

## Synthesis

| Rule | One line |
|------|----------|
| CP.20 | **RAII** for every mutex |
| CP.21 | **Multi-lock** with `scoped_lock` / `std::lock` |
| CP.22 | **No user callbacks** under lock |
| CP.31 | **By-value** for small cross-thread inputs when parent stack might end |
| CP.44 | **Name** your lock guards |

Next: [07_errors_templates_misc.md](07_errors_templates_misc.md).
