---
description: Convert no-argument ITK CTests in a directory to GoogleTest format, one test per commit.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Convert ITK CTests to GoogleTest

Convert all no-argument CTests in the specified directory to GoogleTest format.
Process **one test at a time**, creating a separate git commit for each conversion.

## Target Directory

$ARGUMENTS

If no directory is given, use the current working directory's `test/` subdirectory.

## Step 1: Discover No-Argument Tests

Read `CMakeLists.txt` in the target directory. Find tests that:
- Have their `.cxx` file listed in a `set(*Tests ...)` block
- Have a corresponding `itk_add_test(NAME ... COMMAND ...TestDriver testFunctionName)` block with **no additional arguments** after the test function name

A no-argument test looks like this in CMakeLists.txt:
```cmake
itk_add_test(
  NAME itkFooTest
  COMMAND
    ITKThisModule1TestDriver
    itkFooTest
)
```

Skip tests that pass arguments (files, numbers, paths) after the driver function name.
Skip tests whose `*GTest.cxx` file already exists.

Run this Python snippet to find candidates:
```bash
python3 - <<'EOF'
import re, sys
cmake = open('CMakeLists.txt').read()
# Find itk_add_test blocks with no arguments after the function name
pattern = r'itk_add_test\(\s*NAME\s+(\w+)\s*\n\s*COMMAND\s*\n\s*\S+\s*\n\s*(\w+)\s*\)'
for m in re.finditer(pattern, cmake):
    name, fn = m.group(1), m.group(2)
    if name == fn:
        import os
        gtest = fn.replace('Test', 'GTest') + '.cxx'
        old   = fn + '.cxx'
        if not os.path.exists(gtest) and os.path.exists(old):
            print(f"CANDIDATE: {fn}")
EOF
```

## Step 2: For Each Candidate — Convert One test file at a Time

Work through candidates **one at a time**. For each `itkFooTest`:

### 2a. Read the Old Test File

Read `itkFooTest.cxx` to understand what the test does.

### 2b. Create `itkFooGTest.cxx`

Create the new GTest file. Follow these conventions:
- git mv `itkFooTest.cxx` `itkFooGTest.cxx`
- Include the primary header being tested first (if identifiable)
- Use `#include "itkGTest.h"` (not `<gtest/gtest.h>`)
- Use `ITK_GTEST_EXERCISE_BASIC_OBJECT_METHODS(ptr, ClassName, SuperclassName)` for ITK object boilerplate (requires a named variable, not an expression) in places where `ITK_EXERCISE_BASIC_OBJECT_METHODS` was previously used.
- Wrap helper functions in an anonymous `namespace { }`
- Use `TEST(ClassName, DescriptiveName)` — group by class, use descriptive test names
- Add real value assertions (`EXPECT_EQ`, `EXPECT_NEAR`, `EXPECT_TRUE`, `EXPECT_GT`, etc.) — don't just call functions and hope they don't crash
- Preserve `std::cout` diagnostic output that was in the original
- For legacy API tests: wrap in `#ifndef ITK_FUTURE_LEGACY_REMOVE` / `#endif`
- Prefer to use `EXPECT_GT`, `EXPECT_LT`, `EXPECT_EQ`, and `EXPECT_NEQ` to `EXPECT_TRUE` or `EXPECT_FALSE` when working with comparisons that are not boolean variables.

Template structure:
```cpp
/*=========================================================================
 *
 *  Copyright NumFOCUS
 *
 *  Licensed under the Apache License, Version 2.0 (the "License");
 *  ...
 *=========================================================================*/

// First include the header file to be tested:
#include "itkFoo.h"
#include "itkGTest.h"

namespace
{
// helper functions here
} // namespace

TEST(Foo, BasicFunctionality)
{
  // test body with EXPECT_* assertions
}
```

### 2c. Update `CMakeLists.txt`

Make three edits:

**Remove** the `.cxx` filename from the `ITKThisModule1Tests` (or `ITKThisModule2Tests`) `set(...)` block:
```cmake
# Remove this line:
  itkFooTest.cxx
```

**Remove** the entire `itk_add_test(...)` block:
```cmake
# Remove this block:
itk_add_test(
  NAME itkFooTest
  COMMAND
    ITKThisModule1TestDriver
    itkFooTest
)
```

**Add** the new GTest filename to the `ITKThisModuleGTests` `set(...)` block (before the closing `)`):
```cmake
  itkFooGTest.cxx
  )   # <-- closing paren
```

The `ITKThisModuleGTests` set feeds `creategoogletestdriver(ITKThisModuleGTests ...)` which builds `ITKThisModuleGTestDriver`.

### 2d. Delete the Old File

```bash
git rm Modules/Core/Common/test/itkFooTest.cxx
```

### 2e. Commit

Stage and commit with message format:
```
ENH: Convert itkFooTest to itkFooGTest
```

Requirements enforced by hooks:
- Subject line must start with `ENH:`, `BUG:`, `COMP:`, `DOC:`, `PERF:`, `STYLE:`, or `WIP:`
- Subject line must be ≤ 78 characters
- Always run pre-commit run -a on the entire source tree before committing
- The clang-format pre-commit hook **will reformat** staged C++ files; if the first commit attempt fails, re-stage the reformatted files and commit again

```bash
git add Modules/Core/Common/test/itkFooGTest.cxx \
        Modules/Core/Common/test/CMakeLists.txt
git commit -m "ENH: Convert itkFooTest to itkFooGTest"
# If hook reformats files, re-stage and recommit:
# git add Modules/Core/Common/test/itkFooGTest.cxx
# git commit -m "ENH: Convert itkFooTest to itkFooGTest"
```

After a successful commit, move to the next candidate.

## Step 3: Augmenting Existing GTest Files

If a `*GTest.cxx` already exists (e.g., `itkArrayGTest.cxx`), **append** new `TEST()` blocks rather than creating a new file. Read the existing file first to avoid duplicating coverage. Then follow steps 2c–2e treating it as a new conversion.

## Common Pitfalls

- **`ITK_GTEST_EXERCISE_BASIC_OBJECT_METHODS`** requires a named pointer variable in scope — not an inline `FooType::New()` expression
- **Double braces** for aggregate-initialized ITK structs: `itk::Size<3> sz{ { 10, 10, 10 } };`
- **`[[maybe_unused]]`** on variables that are only used in diagnostics/output
- **`std::hash`** behavior is implementation-defined — never assert `hash(x) == x`
- **Platform-specific assertions**: avoid assuming `std::hash<int>` is identity; avoid assuming specific numeric output values that differ across platforms
- **Do not** add a `main()` function — GTest provides its own
- **Do not** use `EXIT_SUCCESS`/`EXIT_FAILURE` returns — use `EXPECT_*`/`ASSERT_*`
- **Do not** combine multiple ctest files into a single GTest.cxx file.
- The old test driver called `itkFooTest(int argc, char* argv[])` as a function — the new file is standalone and should not define that signature
