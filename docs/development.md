# Development Guide

How to build BinaryLab locally and work on it day to day.

## Prerequisites

- [Git](https://git-scm.com/downloads)
- [CMake](https://cmake.org/download/)
- [Ninja](https://ninja-build.org/) (the Qt installer provides it under `Tools`)
- A MinGW-w64 GCC compiler (the Qt installer provides it under `Tools`)
- [Qt 6](https://www.qt.io/download-qt-installer) with the MinGW kit that matches the compiler

> Use a Qt kit and a compiler of the same MinGW version. Mixing them leads to link errors that are hard to diagnose.

The examples use Windows PowerShell and an installation of Qt 6.12.0 with MinGW 13.1.0 in `C:\Qt`. Adjust the paths to your installation.

## First-time setup

### 1. Clone the repository

```bash
git clone <repository-url>
cd binary-lab
```

Replace `<repository-url>` with the URL of your GitHub repository.

### 2. Configure

From the repository root:

```powershell
cmake -S . -B build -G Ninja `
  -DCMAKE_PREFIX_PATH="C:/Qt/6.12.0/mingw_64"
```

CMake must find Ninja and a compatible C++ compiler. If they are not available through `PATH`, specify them during the first configuration:

```powershell
cmake -S . -B build -G Ninja `
  -DCMAKE_MAKE_PROGRAM="C:/Qt/Tools/Ninja/ninja.exe" `
  -DCMAKE_CXX_COMPILER="C:/Qt/Tools/mingw1310_64/bin/g++.exe" `
  -DCMAKE_PREFIX_PATH="C:/Qt/6.12.0/mingw_64"
```

| Option | Purpose |
|---|---|
| `CMAKE_PREFIX_PATH` | Directory of the Qt kit, so CMake can find Qt 6 |
| `CMAKE_MAKE_PROGRAM` | Path to `ninja.exe`, if it is not in `PATH` |
| `CMAKE_CXX_COMPILER` | Path to the MinGW `g++.exe`, if it is not in `PATH` |

If you change the compiler or the generator, delete `build/` or use a new build directory.

### 3. Build

```powershell
cmake --build build
```

### 4. Run

```powershell
.\build\binary-lab.exe
```

If Windows cannot locate the Qt DLLs, either add the Qt `bin` directory to `PATH` for the current session:

```powershell
$env:PATH = "C:\Qt\6.12.0\mingw_64\bin;$env:PATH"
```

or copy the required libraries next to the executable with `windeployqt`:

```powershell
windeployqt .\build\binary-lab.exe
```

## Editors

### Qt Creator

1. Open Qt Creator.
2. Select **File → Open File or Project**.
3. Open the repository's `CMakeLists.txt`.
4. Select a kit compatible with the installed Qt and MinGW versions.
5. Configure, build and run the project.

### VS Code

Install the Microsoft **C/C++** and **CMake Tools** extensions, select the MinGW kit when prompted, and set `cmake.configureSettings` to include `CMAKE_PREFIX_PATH` so Qt is found.

## Repository layout

*Proposed; the layout may change as the code is written.*

```text
binary-lab/
├── CMakeLists.txt
├── src/
│   ├── core/          # Analysis engine (standard C++, no Qt)
│   │   ├── file/      # Reading, identification, SHA-256
│   │   ├── pe/        # PE parser
│   │   ├── analysis/  # Rules and findings
│   │   └── report/    # Report model and export
│   └── gui/           # Qt Widgets interface
├── tests/             # Automated tests and small test PE files
└── docs/
```

The engine in `src/core` must never include a Qt header. The interface depends on the engine, never the other way round.

## Day-to-day workflow

Rebuild after changes. Ninja only recompiles what changed:

```powershell
cmake --build build
```

Re-run CMake configuration only when you add or remove files or change `CMakeLists.txt` (this happens automatically on the next build in most setups).

Start from a clean state:

```powershell
Remove-Item -Recurse -Force build
cmake -S . -B build -G Ninja -DCMAKE_PREFIX_PATH="C:/Qt/6.12.0/mingw_64"
cmake --build build
```

## Tests

*Automated tests are planned; this section describes the intended workflow.*

Tests are registered with CTest and run from the build directory:

```powershell
ctest --test-dir build --output-on-failure
```

The test suite covers:

- **Parser tests** for headers, sections and imports.
- **Malformed input tests** for truncated files, invalid offsets and inconsistent sizes.
- **Rule tests**, one per rule, checking that each fires on its intended condition and only there.
- **Regression tests** for any bug fixed after release.

Test PE files are small, purpose-built files kept in `tests/`. Do not commit real malware, and do not commit third-party binaries you do not have the right to redistribute.

## Working with sample files

BinaryLab never executes what it analyzes, but handle samples carefully anyway:

- Start with harmless files: system DLLs, your own builds, or the test files in `tests/`.
- Compile small test programs with different settings (for example with and without ASLR or DEP flags) to see how the headers change.
- If you analyze real malware samples, do it only in an isolated virtual machine without shared folders, and never double-click or run them.
- Do not rely on the application's findings to decide that a file is safe to run.