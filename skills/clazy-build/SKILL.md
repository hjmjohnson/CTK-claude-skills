---
name: clazy-build
description: Build CTK with normal compilers, capturing all compiler warnings and errors to a timestamped log file. Supports incremental (default) and clean rebuilds.
user_invocable: true
---

Build the inner CTK project with the standard compiler, capturing all warnings and errors to a log file for analysis.

## Prerequisites

The superbuild must have completed at least once (via `/superbuild-ctk`) so that all external dependencies are built.

## Build Architecture

CTK uses a **two-level build**:
- **Superbuild** (`~/src/CTK/cmake-build-clazy/`) — builds external dependencies (VTK, DCMTK, PythonQt, etc.) and then the CTK inner build as an ExternalProject
- **Inner build** (`~/src/CTK/cmake-build-clazy/CTK-build/`) — builds CTK itself

For **incremental builds** (changed source files only), building the inner build directly is fine and fast:
```bash
cmake --build ~/src/CTK/cmake-build-clazy/CTK-build -j8
```

For **clean rebuilds** (after `--target clean`), always use the superbuild target:
```bash
cmake --build ~/src/CTK/cmake-build-clazy --target CTK -j8
```

**Why?** The superbuild's `CTK` target re-runs PythonQt wrapper generation (`CMake/ctkWrapPythonQt.py`), which regenerates the `generated_cpp/org_commontk_*/` header and init files from the current source headers. After a clean, these generated files are deleted. If you build the inner build directly after clean, the missing generated files cause link failures.

## Steps

### Step 1: Determine build mode

- **Incremental** (default): builds only changed files — use inner build directly
- **Clean**: if the user passes `clean` or `rebuild` as argument, delete build artifacts and rebuild via superbuild target

### Step 2: Build and capture output

```bash
BLD_DIR=~/src/CTK/cmake-build-clazy/CTK-build
BUILD_LOG="${BLD_DIR}/build-$(date +%Y%m%d-%H%M%S).log"
```

If `clean` or `rebuild` was requested:
```bash
cmake --build "${BLD_DIR}" --target clean 2>&1 > /dev/null
# Then use the superbuild target to rebuild (handles PythonQt wrapper regeneration):
cmake --build ~/src/CTK/cmake-build-clazy --target CTK -j8 2>&1 | tee "${BUILD_LOG}"
```

For incremental builds:
```bash
cmake --build "${BLD_DIR}" -j8 2>&1 | tee "${BUILD_LOG}"
```

**IMPORTANT:** Capture both stdout and stderr to the log file using `2>&1 | tee`.

### Step 3: Parse and summarize results

After the build completes, parse the log:

```bash
# Count errors
ERROR_COUNT=$(grep -c ' error:' "${BUILD_LOG}" || echo 0)

# Count warnings (exclude clazy — those are for /clazy-check)
WARNING_COUNT=$(grep ' warning:' "${BUILD_LOG}" | grep -v '\[-Wclazy' | wc -l | tr -d ' ')

# List unique warnings by category
grep -oE '\[-W[^]]+' "${BUILD_LOG}" | grep -v '\[-Wclazy' | sort | uniq -c | sort -rn

# List files with errors
grep ' error:' "${BUILD_LOG}" | cut -d: -f1 | sort -u
```

Report:
1. **Build result**: success or failure (total targets built, expected: 942)
2. **Error count** and which files have errors
3. **Warning count** by category (excluding clazy)
4. **Log file path** for reference

### Step 4: Offer next steps

- If build failed: show the first few errors and offer to fix them
- If build succeeded with warnings: suggest `/clazy-check` to run static analysis, or `/clazy-fix` to address warnings
- If build succeeded clean (0 errors, 0 warnings): suggest `/clazy-test` to run the test suite

## Known Clean Build State

On the `apply_clazy_skills` branch after all current fixes, a clean rebuild should produce:
- **0 errors**
- **1 warning**: `[-Wunused-result]` in `ctkITKErrorLogModelFileLoggingTest1.cpp` (pre-existing, from `QTemporaryFile::open()` — NOT a clazy warning)
- **942 targets built**

Any deviation from this is a regression to investigate.

## Arguments

Optional:
- `clean` or `rebuild` — perform a clean rebuild via superbuild target (delete artifacts first)
- A target name (e.g., `CTKWidgets`) — append `--target <target>` to the cmake inner build command

Examples:
- `/clazy-build` — incremental build (default)
- `/clazy-build clean` — clean rebuild from scratch
- `/clazy-build rebuild` — same as clean
- `/clazy-build CTKWidgetsCppTests` — build only a specific target (incremental)
