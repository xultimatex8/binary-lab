# Tech Stack — BinaryLab

## Overview

| Layer | Technology |
|---|---|
| Language | C++17 |
| Desktop interface | Qt 6 Widgets |
| Build system | CMake |
| Build tool | Ninja |
| Compiler | MinGW-w64 GCC |
| Executable format | Windows Portable Executable (PE) |
| Testing | Automated tests for parsing and analysis logic (planned) |
| Version control | Git |

## Decisions

### Language: C++17

- **Why:** parsing a binary format means working directly with bytes, integer widths, endianness, alignment and memory layout. C++ exposes all of that explicitly, which is the point of a project meant to teach how executables are structured.
- C++17 gives `std::optional`, `std::string_view`, `std::variant` and structured bindings, which make bounds-checked, explicit-failure parsing code much cleaner than older standards.
- **Trade-off:** manual memory and bounds handling is a real risk when reading untrusted input. It is mitigated by a single bounds-checked reader through which all file reads go (see [the PE analysis flow](pe-format-analysis.md#defensive-parsing)) and by malformed-input tests.

### Desktop interface: Qt 6 Widgets

- Qt Widgets is mature, well documented and well suited to data-dense desktop tools: tables, trees, split views, filters and text panes.
- It is written in C++, so the interface and the engine share one language and one build.
- Widgets was chosen over QML because the interface is mostly tables and trees rather than animated, touch-oriented views.
- **Trade-off:** Qt is a large dependency and its licensing (LGPL) has to be respected when distributing binaries.

### Analysis engine: standard C++ only

- The engine (file identification, PE parser, rules, explanations, report model) is a separate static library that depends only on the C++ standard library. It has no `#include <Q...>` anywhere.
- This keeps it testable without a GUI and lets the same engine be reused later from a command-line tool.
- The GUI is a thin layer that calls the engine and renders its result structures.

### PE parsing: own parser, no third-party library

- Libraries such as LIEF or pefile are excellent, but using one would hide exactly what this project exists to teach.
- The parser reads the DOS header, NT headers, optional header, section table and import directory itself, following the documented PE/COFF specification.
- **Trade-off:** more code to write and test, and real-world files that bend the specification will need explicit handling. Both are accepted for the educational value and for full control over validation.

### Hashing: SHA-256

- SHA-256 is the de facto identifier for a file in security tooling, so a hash computed here can be looked up or compared elsewhere.
- *Proposed:* implement it in the engine, so it stays free of Qt, and validate it against the published NIST test vectors. `QCryptographicHash` is a reasonable alternative if keeping the engine Qt-free matters less than writing less code.

### Build: CMake + Ninja

- CMake is the standard for C++ projects and is what Qt Creator, VS Code (CMake Tools) and most CI systems understand natively.
- Ninja gives fast incremental builds.
- The project is organized as targets: engine library, GUI executable and test executables.

### Compiler: MinGW-w64 GCC

- It ships with the Qt installer as a matching kit, which avoids ABI mismatches between Qt and the compiler.
- A single toolchain on Windows keeps setup short. See [development.md](development.md).
- **Trade-off:** MSVC is the other common choice on Windows. Switching later would require installing the MSVC Qt kit and a fresh build directory.

### Testing: automated tests (planned)

- Parsing bugs show up as crashes, wrong output or out-of-bounds reads, so tests are part of the design rather than an afterthought.
- *Proposed:* Qt Test (already available with Qt) registered with CTest, so `ctest` runs everything from the command line. GoogleTest would work equally well.
- Test inputs are small, deliberately built PE files (valid, truncated, with inconsistent offsets or sizes) kept in the repository. Real malware is never committed.

### Reports

- The engine produces a structured report model; the GUI renders it and the exporter serializes it.
- *Proposed export formats:* JSON for machine-readable output and Markdown for human-readable review. Both contain the evidence behind each finding.

### Version control: Git

- Git for history and branching, hosted on GitHub.
- GitHub Actions to build, run the tests and run static checks.