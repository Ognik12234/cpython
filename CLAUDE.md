# CPython Codebase Guide for AI Assistants

This document provides a comprehensive overview of the CPython repository structure, development workflows, and conventions. It is intended to help AI assistants (like Claude) navigate and contribute to CPython effectively.

## Repository Overview

CPython is the reference implementation of the Python programming language, written in C and Python. This repository is at **Python 3.13.0 alpha 5** (main development branch).

- **Source**: https://github.com/python/cpython
- **Issue tracker**: https://github.com/python/cpython/issues
- **Documentation**: https://docs.python.org
- **Developer's Guide**: https://devguide.python.org/

---

## Directory Structure

```
cpython/
├── Android/          # Android-specific build configuration
├── Doc/              # Documentation source (Sphinx/reStructuredText)
├── Grammar/          # PEG grammar definitions (python.gram, Tokens)
├── Include/          # C header files for the Python C API
│   └── internal/     # Internal (private) C API headers
├── iOS/              # iOS/WebAssembly cross-compilation support
├── Lib/              # Python standard library
│   └── test/         # Standard library test suite (496+ test_*.py files)
├── Mac/              # macOS-specific build tools and resources
├── Misc/             # Miscellaneous files (ACKS, NEWS.d/, docs)
├── Modules/          # C extension modules
├── Objects/          # Core object type implementations in C
├── Parser/           # PEG parser, tokenizer, AST generation
├── PC/               # Windows-specific C code (launcher, registry)
├── PCbuild/          # Windows Visual Studio project files
├── Programs/         # Executable program sources (python.c, etc.)
├── Python/           # Core interpreter (bytecode, GC, import, compiler)
├── Tools/            # Development utilities
│   ├── clinic/       # Argument Clinic (C argument parsing automation)
│   ├── peg_generator/# PEG parser generator
│   ├── cases_generator/ # Bytecode case generator
│   ├── c-analyzer/   # Static analyzer for C code
│   ├── gdb/          # GDB helpers for debugging CPython
│   └── jit/          # JIT compiler tooling
└── .github/
    ├── workflows/    # GitHub Actions CI/CD workflows
    └── CONTRIBUTING.rst
```

### Key C Source Directories

| Directory | Purpose |
|-----------|---------|
| `Python/` | Core interpreter: `ceval.c` (eval loop), `compile.c` (bytecode compiler), `import.c`, `gc.c` |
| `Objects/` | Object implementations: `typeobject.c`, `unicodeobject.c`, `listobject.c`, etc. |
| `Modules/` | C extension modules: `_asynciomodule.c`, `_csv.c`, `_datetimemodule.c`, etc. |
| `Parser/` | PEG parser and tokenizer |
| `Include/` | Public C API headers; `Include/internal/` for private headers |
| `Programs/` | `python.c` (main entry point), `_freeze_module.c`, `_testembed.c` |

---

## Build System

### Unix/Linux/macOS

```bash
# Basic build
./configure
make
make test
sudo make install   # installs as python3; use altinstall for non-primary

# Debug build (recommended for development)
mkdir debug && cd debug
../configure --with-pydebug
make

# Optimized build (PGO + optional LTO)
./configure --enable-optimizations
make
```

**Out-of-tree builds** are fully supported and recommended for development:
```bash
mkdir build-debug && cd build-debug
../configure --with-pydebug && make
```

### Windows

See `PCbuild/readme.txt` for Visual Studio build instructions and `Tools/msi/README.txt` for installer builds.

### Configure Options

| Option | Effect |
|--------|--------|
| `--with-pydebug` | Full debug build (enables `Py_DEBUG`, `Py_REF_DEBUG`, assertions) |
| `--enable-optimizations` | PGO (Profile Guided Optimization) |
| `--with-lto` | Link Time Optimization |
| `--with-trace-refs` | Heavy reference debugging (`Py_TRACE_REFS`) |
| `--with-address-sanitizer` | AddressSanitizer support |

### Special Debug Builds (via `EXTRA_CFLAGS`)

```bash
make EXTRA_CFLAGS="-DPy_REF_DEBUG"   # Reference count tracking
make EXTRA_CFLAGS="-DLLTRACE"         # Low-level interpreter tracing
```

See `Misc/SpecialBuilds.txt` for full details.

---

## Testing

### Running Tests

```bash
# Standard test run (skips resource-intensive tests)
make test

# Verbose single test module
make test TESTOPTS="-v test_os"

# Multiple test modules
make test TESTOPTS="-v test_os test_gdb"

# Full buildbot test (all resources enabled)
make buildbottest

# Run tests directly via Python
./python -m test test_os -v
./python -m test -j4          # parallel, 4 workers
./python -m test -x test_slow # exclude a test
```

### Test Organization

- **Location**: `Lib/test/` (496+ `test_*.py` files)
- **Runner infrastructure**: `Lib/test/libregrtest/`
- Tests follow the `unittest` framework
- Tests for a module `foo` live in `Lib/test/test_foo.py`
- Tests for C extension modules may be in `Lib/test/` as well

### Writing Tests

- Use `unittest.TestCase` subclasses
- Use `test.support` utilities for common helpers (temp files, platform checks, etc.)
- Mark slow tests with `@unittest.skip` or resource guards
- Use `test.support.import_helper`, `test.support.os_helper` for isolation

---

## Code Style and Conventions

### Editor Configuration (`.editorconfig`)

| File type | Indentation |
|-----------|-------------|
| `.py`, `.c`, `.cpp`, `.h` | 4 spaces |
| `.rst` | 3 spaces |
| `.js`, `.yml` | 2 spaces |

All files: trim trailing whitespace, insert final newline, use spaces (not tabs).

### Python Code Style

- Follow **PEP 8** for Python code in `Lib/`
- **Ruff** is used for linting (`Lib/test/` and `Tools/clinic/`)
- Run pre-commit checks: `pre-commit run --all-files`

### C Code Style

- Follow the conventions in `Misc/indent.pro` (GNU indent profile)
- 4-space indentation, no tabs
- Use `PyObject *` naming convention for object pointers
- Error handling: use `goto error` pattern; always check return values
- Reference counting: `Py_INCREF`/`Py_DECREF`/`Py_XDECREF`, or use `Py_NewRef()`/`Py_XNewRef()`
- New C API functions should use the `Argument Clinic` tool (see `Tools/clinic/`)

### Naming Conventions

| Context | Convention |
|---------|-----------|
| Public Python API (C) | `PyObject_Foo`, `PyLong_FromLong` |
| Internal C functions | `_Py_Foo`, `_PyObject_Foo` |
| Module init functions | `PyInit_modulename` |
| Static/private C functions | lowercase with underscores |

---

## Argument Clinic

Argument Clinic (`Tools/clinic/`) is used to auto-generate argument parsing boilerplate for C extension modules. When modifying C extension modules that use Clinic:

```bash
# Regenerate clinic output
./python Tools/clinic/clinic.py Modules/_mymodule.c

# Or regenerate all
make clinic
```

Clinic input is in `/*[clinic input]` ... `/*[clinic start generated code]*/` blocks in `.c` files.

---

## Grammar and Parser

- **Grammar definition**: `Grammar/python.gram` (PEG-based, 64KB)
- **Tokens**: `Grammar/Tokens`
- The parser is generated from the grammar using `Tools/peg_generator/`
- After changing the grammar, regenerate the parser:
  ```bash
  make regen-pegen
  ```

---

## Documentation

### Structure

```
Doc/
├── c-api/       # C API reference
├── library/     # Standard library reference
├── reference/   # Language reference
├── tutorial/    # Python tutorial
├── howto/       # How-to guides
├── extending/   # Extending/embedding Python
├── whatsnew/    # What's New documents per version
└── conf.py      # Sphinx configuration
```

### Building Docs

```bash
cd Doc
make venv          # set up virtual environment with Sphinx
make html          # build HTML documentation
make check         # check for markup errors
make linkcheck     # check external links
```

Documentation uses **reStructuredText** and **Sphinx** with `python-docs-theme`.

### Changelog Entries

New changelog entries go in `Misc/NEWS.d/next/` in the appropriate subdirectory. File naming follows the `gh-ISSUE.TYPE.rst` convention (e.g., `gh-12345.bugfix.rst`). Use the `blurb` tool or create files manually.

---

## CI/CD

### GitHub Actions (`.github/workflows/`)

| Workflow | Purpose |
|----------|---------|
| `build.yml` | Main build and test matrix |
| `lint.yml` | Code linting |
| `mypy.yml` | Type checking |
| `jit.yml` | JIT compiler tests |
| `build_msi.yml` | Windows MSI installer build |
| `documentation-links.yml` | Documentation link checking |
| `reusable-ubuntu.yml` | Reusable Ubuntu job |
| `reusable-macos.yml` | Reusable macOS job |
| `reusable-windows.yml` | Reusable Windows job |

### Azure DevOps (`.azure-pipelines/`)

- `ci.yml` — CI definition
- `posix-steps.yml` / `windows-steps.yml` — Platform-specific steps
- `posix-deps-apt.sh` — APT dependency installation

### Pre-commit Hooks (`.pre-commit-config.yaml`)

- **Ruff** — Python linting
- TOML/YAML validation
- End-of-file newline enforcement
- Trailing whitespace removal
- **Sphinx-lint** — documentation markup checks

---

## Contribution Workflow

1. **File an issue** at https://github.com/python/cpython/issues before major changes
2. **Fork and clone** the repository
3. **Create a feature branch** from `main`
4. **Make changes** following the style conventions above
5. **Add a changelog entry** in `Misc/NEWS.d/next/`
6. **Add/update tests** in `Lib/test/`
7. **Run the test suite**: `make test`
8. **Submit a PR** — see the [PR lifecycle guide](https://devguide.python.org/getting-started/pull-request-lifecycle/)
9. **First contribution?** Add yourself to `Misc/ACKS`

All non-code discussions belong in GitHub Issues, not PR comments.

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `Python/ceval.c` | The main interpreter evaluation loop |
| `Python/compile.c` | Bytecode compiler |
| `Objects/typeobject.c` | Python type system implementation |
| `Objects/unicodeobject.c` | Unicode string implementation |
| `Modules/Setup` | Controls which C modules are built static vs. shared |
| `Lib/importlib/` | Python's import system |
| `Lib/test/support/` | Test support utilities |
| `Misc/stable_abi.toml` | Stable ABI symbol definitions |
| `Include/Python.h` | Main public C API header |
| `Grammar/python.gram` | PEG grammar for Python syntax |
| `configure.ac` | Autoconf build configuration |

---

## Common Development Tasks

### Regenerating Auto-generated Files

```bash
make regen-all       # regenerate all auto-generated files
make regen-pegen     # regenerate parser from Grammar/python.gram
make regen-opcode    # regenerate opcode tables
make regen-clinic    # regenerate Argument Clinic code
make regen-typeslots # regenerate type slot tables
```

### Checking for Memory Leaks / Reference Issues

```bash
# Build with reference debugging
./configure --with-pydebug
make
./python -X showrefcount  # show ref count after each statement

# Run with Valgrind
make valgrind
# See Misc/valgrind-python.supp for suppression file
```

### Running a Subset of Tests Quickly

```bash
./python -m test -j0 test_ast test_compile  # no parallelism, specific modules
./python -m test -m "test_something*"       # pattern match test methods
./python -m test --fail-fast                # stop at first failure
```

### Checking the Stable ABI

The stable ABI is defined in `Misc/stable_abi.toml`. After changing exported symbols:
```bash
make check-abidump  # compare against expected ABI dump
```

---

## Platform Notes

- **Linux**: Primary development platform; most CI runs here
- **macOS**: See `Mac/README.rst` for framework and universal build options
- **Windows**: See `PCbuild/readme.txt`; uses Visual Studio
- **Android**: See `Android/` directory
- **iOS/WASI**: Cross-compilation supported; see `iOS/` and `.github/workflows/reusable-wasi.yml`

---

## Useful Links

- [Python Developer's Guide](https://devguide.python.org/) — comprehensive contributor documentation
- [Python Discourse](https://discuss.python.org/) — community discussion
- [Buildbot status](https://buildbot.python.org/all/#/release_status) — CI across many platforms
- [PSF Code of Conduct](https://www.python.org/psf/codeofconduct/)
