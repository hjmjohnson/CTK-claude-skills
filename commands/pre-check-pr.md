---
description: Pre-flight check for ITK pull requests — identifies and fixes common reviewer concerns before submission.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# ITK Pull Request Pre-flight Check

Analyze the current branch's uncommitted and committed changes (relative to
`upstream/main`) and identify issues that ITK reviewers commonly flag. Fix
what can be fixed automatically; report what needs human attention.

## Scope

$ARGUMENTS

If no argument is given, check all changes on the current branch relative to
`upstream/main`.

---

## Step 1: Gather Branch Diff

```bash
git diff upstream/main --stat
git diff upstream/main --name-only
git log upstream/main..HEAD --oneline
```

Compute the total lines changed:
```bash
git diff upstream/main --shortstat
```

---

## Check 1: PR Size

**Reviewer concern (N-Dekker):** "Pull requests with more than 2000 changes
are very hard to review carefully."

- **WARN** if total insertions + deletions > 2000 lines across changed files.
- **ERROR** if > 5000 lines.
- Report the count and suggest splitting.

---

## Check 2: Commit Message Format

**Reviewer concern (N-Dekker, kw-commit-msg hook):** Subject must start with a
recognized prefix, be ≤ 78 characters, and accurately describe the change.

For every commit in `git log upstream/main..HEAD`:

1. **ERROR** if subject does not start with one of:
   `ENH:`, `BUG:`, `COMP:`, `DOC:`, `STYLE:`, `PERF:`, `WIP:`

2. **ERROR** if subject length > 78 characters.

3. **WARN** for common inaccuracies:
   - Subject says "Add `override`" but diff only removes `virtual` → suggest
     `STYLE: Remove redundant virtual keywords from overrides`
   - Subject says "Convert X to GTest" but diff shows no GTest file created
   - Subject contains typos (run basic spell check on the subject line)
   - Subject still references old internal names (e.g. `ALL_CAPS_TEST_NAMES`)
     that were changed in the diff

**Auto-fix:** If the subject is otherwise correct but exceeds 78 chars,
truncate and report. Do not auto-fix prefix or content.

---

## Check 3: GTest File Conventions

For every `*GTest.cxx` file added or modified in the diff:

### 3a. Include Order

**Read the file.** Check:

- **ERROR** if `#include "itkGTest.h"` is missing and `<gtest/gtest.h>` is
  also missing (no GTest include at all).
- **WARN** if `#include <gtest/gtest.h>` is used directly instead of
  `#include "itkGTest.h"` — prefer the ITK wrapper unless the test uses no
  ITK GTest macros.
- **WARN** if the primary header being tested is NOT the first include.
  The pattern should be:
  ```cpp
  // First include the header file to be tested:
  #include "itkFoo.h"
  #include "itkGTest.h"
  ```

### 3b. Test Suite and Test Case Naming

**Reviewer concern (N-Dekker):** Every GTest naming violation gets flagged.
See [GoogleTest FAQ: Why should test suite names and test names not contain underscore?](https://google.github.io/googletest/faq.html#why-should-test-suite-names-and-test-names-not-contain-underscore)

For every `TEST(SuiteName, TestName)` and `TEST_F(FixtureName, TestName)`:

- **ERROR** if `SuiteName` contains an underscore → must fix.
- **ERROR** if `TestName` contains an underscore → must fix.
- **WARN** if `SuiteName` starts with `itk` (redundant prefix) → suggest removing.
- **WARN** if `SuiteName` ends with `Test` (redundant suffix) → suggest removing.
- **WARN** if `TestName` ends with `Test` or `Tests` (redundant suffix) →
  suggest renaming to describe the behavior (e.g. `SupportsSubregions`,
  `ExercisesBasicObjectMethods`, `ConvertedLegacyTest`).
- **WARN** if `TestName` is all uppercase (e.g. `RANDOM_WALK_TESTS`) →
  must rename to `CamelCase`.

**Good examples from merged ITK PRs:**
```cpp
TEST(ImageRandomNonRepeatingIteratorWithIndex, SupportsSubregions)
TEST(AutoPointer, OwnershipTransfer)
TEST(StdStreamStateSave, RestoresCoutState)
TEST(BoundingBox, EmptyBoxDefaults)
TEST(SobelOperator, ExerciseBasicObjectMethods)
TEST(Foo, ConvertedLegacyTest)    // acceptable for mechanical conversions
```

**Auto-fix:** Replace underscores in test names with camelCase equivalents
where unambiguous. Remove `itk` prefix from suite name. Remove `Test`/`Tests`
suffix from suite name. Report all renames clearly.

### 3c. ITK Object Filter Boilerplate

**Reviewer concern (dzenanz):** "`ITK_EXERCISE_BASIC_OBJECT_METHODS` is
missing."

If the test creates an ITK object (detectable by `::New()` call or `itkNew`
macro), check:

- **WARN** if neither `ITK_GTEST_EXERCISE_BASIC_OBJECT_METHODS` nor
  `ITK_EXERCISE_BASIC_OBJECT_METHODS` appears in the test, AND the test was
  converted from an old CTest that DID call `ITK_EXERCISE_BASIC_OBJECT_METHODS`.

The macro requires a **named pointer variable**, not an inline expression:
```cpp
// CORRECT — named variable:
auto filter = FilterType::New();
ITK_GTEST_EXERCISE_BASIC_OBJECT_METHODS(filter, FilterType, Superclass);

// WRONG — expression:
ITK_GTEST_EXERCISE_BASIC_OBJECT_METHODS(FilterType::New(), FilterType, Superclass);
```

### 3d. Assertion Quality

- **ERROR** if `EXPECT_TRUE(a > b)` — use `EXPECT_GT(a, b)`.
- **ERROR** if `EXPECT_TRUE(a < b)` — use `EXPECT_LT(a, b)`.
- **ERROR** if `EXPECT_TRUE(a >= b)` — use `EXPECT_GE(a, b)`.
- **ERROR** if `EXPECT_TRUE(a <= b)` — use `EXPECT_LE(a, b)`.
- **ERROR** if `EXPECT_TRUE(a == b)` — use `EXPECT_EQ(a, b)`.
- **ERROR** if `EXPECT_TRUE(a != b)` — use `EXPECT_NE(a, b)`.
- **ERROR** if `EXPECT_TRUE(ptr == nullptr)` — use `EXPECT_EQ(ptr, nullptr)`.
- **ERROR** if `EXPECT_TRUE(ptr != nullptr)` — use `EXPECT_NE(ptr, nullptr)`.
- **WARN** if `EXPECT_*` is inside a `while` loop body — the assertion may
  never execute if the loop condition fails before reaching it. Consider
  `ASSERT_*` or restructuring.
- **WARN** if ITK array-like objects are compared element-by-element with
  `EXPECT_NEAR` in a loop — use `ITK_EXPECT_VECTOR_NEAR(v1, v2, tol)` instead.

**Auto-fix:** Replace `EXPECT_TRUE(a OP b)` with the specific macro equivalent
where the operator is unambiguous.

### 3e. Portability: Float Space Precision

- **WARN** if `itk::GTest::MakePoint(...)` or `itk::GTest::MakeVector(...)` is
  used — these always return `double`, which breaks builds with
  `ITK_USE_FLOAT_SPACE_PRECISION=ON`. Use aggregate initialization instead:
  ```cpp
  // Prefer:
  PointType point{ 1.0, 2.0, 3.0 };
  // Over:
  auto point = itk::GTest::MakePoint(1.0, 2.0, 3.0);
  ```

### 3f. No `main()` Function

- **ERROR** if the GTest file defines a `main()` function — GTest provides its
  own entry point.

### 3g. No File I/O / DATA{} References

- **ERROR** if the GTest file references `DATA{`, `${ITK_DATA_ROOT}`,
  `${ITK_TEST_OUTPUT_DIR}`, or `argc`/`argv` parameters — these cannot be
  passed to GTest and indicate the test should remain as a CTest, not be
  converted.

---

## Check 4: CMakeLists.txt Completeness

For every `CMakeLists.txt` modified in the diff, verify the GTest migration
is complete. If a `*GTest.cxx` was added to the branch:

Let `BaseName = itkFooTest` (derived from the GTest file name `itkFooGTest.cxx`).

- **ERROR** if `itkFooTest.cxx` still appears in any `set(*Tests ...)` block.
- **ERROR** if `itk_add_test(NAME itkFooTest ...)` block still exists.
- **ERROR** if `itkFooGTest.cxx` does NOT appear in a `set(*GTests ...)` block.
- **WARN** if a `creategoogletestdriver(...)` call does not reference the GTests
  set that `itkFooGTest.cxx` was added to.

---

## Check 5: Old Test File Deletion

**Reviewer concern (dzenanz):** "This file was not deleted from git tracking."

For every `*GTest.cxx` added in the diff:

Derive the old filename: `itkFooGTest.cxx` → `itkFooTest.cxx`.

- **ERROR** if `itkFooTest.cxx` still exists in the working tree (`git status`
  shows it as untracked or modified, rather than deleted).
- **ERROR** if `itkFooTest.cxx` was not staged as a deletion (`git rm` was not
  run).

**Auto-fix:** Run `git rm <old-file>` if the file exists and is untracked.

---

## Check 6: clang-format Compliance

**Reviewer concern (N-Dekker, dzenanz):** clang-format violations are caught
by the pre-commit hook but sometimes slip through.

```bash
# Check if any staged C++ files would be reformatted:
pre-commit run clang-format --files $(git diff upstream/main --name-only | grep -E '\.(cxx|hxx|h)$' | tr '\n' ' ')
```

- **ERROR** if clang-format would modify any file in the diff.

**Auto-fix:** Run `Utilities/Maintenance/clang-format.bash --modified` and
re-stage affected files.

---

## Check 7: YAML CI Files (ccache patterns)

If any `.github/workflows/*.yml` or `Testing/ContinuousIntegration/Azure*.yml`
files are modified:

- **WARN** if a GitHub Actions workflow uses `hendrikmuhs/ccache-action` —
  the project has migrated to `actions/cache@v4` with a manual install step.
- **WARN** if `CCACHE_BASEDIR`, `CCACHE_COMPILERCHECK`, or `CCACHE_NOHASHDIR`
  env vars are missing from an Azure Pipelines file that uses ccache.
- **WARN** if `ccache --evict-older-than 7d` step is missing after a cache
  restore task in any CI file.

---

## Check 8: Modern C++ / ITK Style (Advisory)

These are WARN-only — not blocking, but N-Dekker or blowekamp will note them:

- `image->Allocate(); image->FillBuffer(0);` → prefer `image->AllocateInitialized();`
- `size[0] = n; size[1] = n; size[2] = n;` → prefer `SizeType::Filled(n)`
- `region.SetIndex(index); region.SetSize(size);` → prefer `RegionType region = { index, size };`
- `std::rand()` → prefer `std::mt19937` with `std::uniform_*_distribution`
- `push_back(SomeStruct{a, b})` → prefer `emplace_back(a, b)`
- `#include "vnl_sample.h"` → flag for removal (deprecated in ITK)

---

## Output Format

Print a structured report:

```
=== ITK PR Pre-flight Check ===

Branch: <branch-name>
Commits: <N> commits ahead of upstream/main
Lines changed: +<ins> -<del> (<total> total)

ERRORS (must fix before merging):
  [CMake]  itkFooTest.cxx still listed in ITKCommon1Tests set (CMakeLists.txt:38)
  [GTest]  TEST(itkFooTest, SOME_THING) — underscore in test name (itkFooGTest.cxx:47)
  [GTest]  EXPECT_TRUE(a > b) should be EXPECT_GT(a, b) (itkFooGTest.cxx:92)
  [Git]    itkFooTest.cxx not deleted from git tracking

WARNINGS (reviewers will request changes):
  [GTest]  Suite name 'itkFooTest' has redundant 'itk' prefix and 'Test' suffix
           → suggest: TEST(Foo, ConvertedLegacyTest)
  [Size]   1847 lines changed — approaching reviewer discomfort threshold (2000)
  [CI]     CCACHE_NOHASHDIR missing from AzurePipelinesLinux.yml variables

ADVISORY (style; N-Dekker may note):
  [Style]  image->Allocate(); FillBuffer(0) → use AllocateInitialized() (itkFooGTest.cxx:23)

AUTO-FIXED:
  EXPECT_TRUE(ptr == nullptr) → EXPECT_EQ(ptr, nullptr) (itkFooGTest.cxx:55)
  Test name MY_TEST_CASE → MyTestCase (itkFooGTest.cxx:47)

PASSED:
  ✓ Commit messages (2/2 valid)
  ✓ clang-format clean
  ✓ Old test file deleted
  ✓ CMakeLists.txt: itkFooGTest.cxx in GTests set
```

If no errors or warnings are found, print: `✓ All checks passed. Ready for review.`

---

## Reviewer Quick Reference (from 3 years of PR analysis)

**N-Dekker** flags: underscore in test names, `itk`/`Test` prefix/suffix in
suite names, PR size > 2000 lines, `EXPECT_TRUE(comparison)` instead of
specific macro, direct `<gtest/gtest.h>` include, commit message inaccuracy,
`itk::GTest::MakePoint` with float-precision builds.

**dzenanz** flags: old `*Test.cxx` not deleted from git, missing
`ITK_GTEST_EXERCISE_BASIC_OBJECT_METHODS`, CMakeLists.txt incomplete
(test still registered as CTest), Windows-specific CI failures.

**blowekamp** flags: `EXPECT_*` inside loops (unreachable assertions), numeric
instability in formulas, CMake option placement, test failure reporting quality.

**All reviewers:** clang-format violations, DATA{}/file-I/O in GTest files,
`main()` function in GTest files.
