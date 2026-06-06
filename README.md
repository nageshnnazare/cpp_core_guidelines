# C++ Core Guidelines — study notes

**Official document:** [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) (ISO C++ Foundation; editors Bjarne Stroustrup and Herb Sutter).

These files are **tutorial-style explanations** of important themes and selected rules. Each chapter file includes **C++ examples** (good vs bad patterns, `#if` guards for C++20 where needed). They are **not** a full reproduction of the guidelines (the live document is very large and updated over time). For rule IDs, exact wording, and **Enforcement** sections, always consult the official site.

## Contents (local)

| File | Topics |
|------|--------|
| [00_introduction_and_profiles.md](00_introduction_and_profiles.md) | Aims, gradual adoption, **profiles** (type / bounds / lifetime), enforcement philosophy |
| [01_philosophy_P.md](01_philosophy_P.md) | **P.1–P.13** — express intent, static safety, immutability, tools, libraries |
| [02_interfaces_I.md](02_interfaces_I.md) | **I.1–I.13** — contracts, `not_null`, no array-as-pointer, ownership |
| [03_functions_F.md](03_functions_F.md) | **F.15–F.21** — parameter passing, in / in-out / out / consume |
| [04_resource_management_R.md](04_resource_management_R.md) | **RAII**, raw pointer meaning, `span`, smart pointers, `make_*` |
| [05_classes_and_hierarchies_C.md](05_classes_and_hierarchies_C.md) | Destructors, **C.35** base class dtor, copy/move, **C.67** polymorphic types |
| [06_concurrency_CP.md](06_concurrency_CP.md) | Locks, RAII, `scoped_lock`, data between threads |
| [07_errors_templates_misc.md](07_errors_templates_misc.md) | Exceptions vs errors, constants, templates — headline rules |
| [08_gsl_guidelines_support_library.md](08_gsl_guidelines_support_library.md) | **`span`**, **`not_null`**, **`owner`**, contracts vocabulary |

## License / attribution

The [published guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) are offered under an MIT-style license. These derivative notes are original prose informed by that document; rule numbers (**P.1**, **R.1**, etc.) refer to the public guideline set.
