# CTK Claude Skills

This repository contains [Claude Code](https://claude.ai/code) skills, settings, and project context
for working on the [CTK (Common Toolkit)](https://github.com/commontk/CTK) codebase.

CTK is the upstream C++/Qt library for **3D Slicer** and other medical image analysis platforms.
The primary focus of these skills is systematic clazy static analysis — finding and fixing Qt code
quality issues that affect correctness, performance, and Qt6 portability for the whole community.

---

## Installation

```bash
# From your CTK source tree root:
cd ${MY_CTK_SOURCE_TREE}
git clone git@github.com:hjmjohnson/CTK-claude-skills.git ${MY_CTK_SOURCE_TREE}/.claude

# Keep .claude out of CTK's git tracking (it's a separate repo):
echo ".claude" >> ${MY_CTK_SOURCE_TREE}/.git/info/exclude
echo "CLAUDE.md" >> ${MY_CTK_SOURCE_TREE}/.git/info/exclude

# Symlink CLAUDE.md so Claude Code finds it at the project root:
ln -s ${MY_CTK_SOURCE_TREE}/.claude/CLAUDE.md ${MY_CTK_SOURCE_TREE}/CLAUDE.md
```

Then open Claude Code (`claude`) from the CTK source root. The skills load automatically.

---

## Prerequisites

You need [Claude Code](https://claude.ai/code) installed:

```bash
npm install -g @anthropic-ai/claude-code
```

And the clazy toolchain. **macOS (Homebrew):**

```bash
brew install llvm clazy ninja cmake
brew install qt@5    # for Qt5 builds (Slicer's current default)
brew install qt      # for Qt6 builds
xcode-select --install   # ensures CommandLineTools SDK is present
```

See `/clazy-configure` for Linux and Windows setup.

---

## The Clazy Workflow

The five clazy skills form a **pipeline** — run them in order:

```
/clazy-configure  →  /superbuild-ctk  →  /clazy-build  →  /clazy-check  →  /clazy-fix  →  /clazy-test
      │                     │                 │                 │                │               │
  One-time setup       First build       Incremental       Static analysis    Fix one        Verify no
  (cmake -G Ninja)    (~30-60 min)         rebuild         (clazy-standalone)  check/commit   regressions
```

### Quick start for a new environment

```
/clazy-configure qt6        # configure superbuild with Slicer-relevant options
/superbuild-ctk             # first full build (downloads DCMTK, ITK, VTK, PythonQt)
/clazy-check level1         # run level1 static analysis, get a prioritized fix list
/clazy-fix range-loop-detach   # fix one check category, one commit
/clazy-test                 # confirm no regressions (36 known failures are expected)
```

### Quick start for an existing environment

```
/clazy-build                # incremental build to confirm clean state
/clazy-check level2         # run level2 analysis (10,000+ warnings, several minutes)
/clazy-fix function-args-by-ref   # example: fix a high-volume check
/clazy-test                 # verify
```

---

## Skill Reference

| Skill | Purpose | When to use |
|-------|---------|-------------|
| `/clazy-configure` | CMake superbuild configuration with all Slicer-relevant CTK libs, Ninja generator, correct compiler | Once per new build environment |
| `/superbuild-ctk` | Full superbuild (downloads + builds all external deps: DCMTK, ITK, VTK, PythonQt) | Once after configure, or after dep version changes |
| `/clazy-build` | Inner CTK incremental or clean rebuild, captures warnings to timestamped log | After each source change |
| `/clazy-check` | Run `clazy-standalone` at level0/1/2, produce log file with all warnings categorized by check | Before deciding what to fix |
| `/clazy-fix` | Fix one check category end-to-end: read docs → edit source → clean rebuild → run tests → commit | One check per invocation |
| `/clazy-test` | Run CTest suite, classify failures as known vs new regressions | After every fix before committing |

### /clazy-fix usage

`/clazy-fix` does the complete fix cycle for one check:

```
/clazy-fix                          # auto-pick highest-frequency fixable check
/clazy-fix range-loop-detach        # fix a specific check
/clazy-fix -a connect-not-normalized  # use clazy auto-fix YAML (when available)
```

Each invocation:
1. Reads the clazy documentation for the check
2. Extracts all warnings from the most recent clazy log
3. Fixes every instance (uses parallel agents for high-volume checks)
4. Does a **clean rebuild** via the superbuild target (required for PythonQt wrapper regeneration)
5. Runs the full test suite
6. Commits with a `STYLE:` or `BUG:` prefix message
7. Appends an entry to the rationale report at `cmake-build-clazy/CTK-build/clazy-fix-report.md`

---

## Current Progress (`apply_clazy_skills` branch)

As of 2026-03, the branch is 35 commits ahead of `master`, fixing:

| Check | Level | Category | Files | Notes |
|-------|-------|----------|-------|-------|
| `container-anti-pattern` | level2 | STYLE | ~10 | Remove `.values()`, `.keys()` temp lists |
| `unused-non-trivial-variable` | level1 | STYLE | ~5 | Remove dead Qt objects |
| `qmap-with-pointer-key` | level2 | STYLE | ~3 | QMap→QHash for pointer keys |
| `strict-iterators` | level2 | STYLE | ~8 | Use constFind/cbegin/cend |
| `use-static-qregularexpression` | level2 | STYLE | ~6 | Static QRegularExpression |
| `connect-by-name` | level2 | STYLE | ~4 | Rename on_*_* slots |
| `const-signal-or-slot` | level0 | STYLE | ~5 | Remove const from signals |
| `incorrect-emit` | level0 | STYLE | ~4 | Fix missing/wrong emit |
| `detaching-temporary` | level2 | STYLE | ~3 | Avoid non-const on temporaries |
| `range-loop-detach` | level1 | STYLE | ~6 | `std::as_const()` in range-for |
| `global-const-char-pointer` | level2 | STYLE | 1 | `const char* const` |
| `returning-void-expression` | level2 | **BUG** | 7 | 4 real bugs: signals never emitted |
| `missing-qobject-macro` | level2 | STYLE | 2 | Fix Python scripting breaks |
| `non-pod-global-static` | level1 | STYLE | 1 | Q_GLOBAL_STATIC for thread safety |
| `function-args-by-ref` | level2 | STYLE | 78 | Pass Qt types by const-ref |

**Remaining level2 Tier-1 checks to fix:**
- `function-args-by-value` (161 warnings)
- `container-anti-pattern` remaining
- `qproperty-without-notify`

**Build state**: 0 errors, 1 pre-existing warning, 942 targets, 86% tests passing (36 known failures).

---

## Important Notes for Contributors

### Linear history — no merge commits

The `apply_clazy_skills` branch maintains a linear commit history. Each fix is one commit.
**Do not merge** — cherry-pick or rebase onto master when the PR lands.

### Commit message format

```
STYLE: Fix clazy <check-name> warnings
BUG: Fix <what> (found via clazy <check>)
COMP: Fix compilation issue related to <check>
```

Valid prefixes: `ENH:`, `BUG:`, `COMP:`, `DOC:`, `STYLE:`, `PERF:`, `WIP:`

### Slicer downstream impact

Virtual method signature changes in CTK affect Slicer. Known Slicer classes that override CTK virtuals:

| CTK virtual | Slicer override |
|-------------|----------------|
| `ctkLayoutManager::viewFromXML()` | `qMRMLLayoutManager::viewFromXML()` |
| `ctkLayoutViewFactory::createViewFromXML()` | `qMRMLLayoutViewFactory`, `qSlicerSingletonViewFactory` |

When a fix changes public virtual signatures, note it in the commit message. Coordinate with the
Slicer team before merging.

### std::as_const() not qAsConst()

CTK builds with C++17. Always use `std::as_const()` — `qAsConst()` is deprecated in Qt 6.6+.

### Clean rebuilds require the superbuild target

```bash
# CORRECT — triggers PythonQt wrapper regeneration:
cmake --build ~/src/CTK/cmake-build-clazy --target CTK -j8

# WRONG after --target clean — PythonQt wrappers not regenerated:
cmake --build ~/src/CTK/cmake-build-clazy/CTK-build -j8
```

### Known pre-existing test failures

36 tests fail consistently on the `apply_clazy_skills` branch (and on `master`). These are not
regressions. See `/clazy-test` for the full list. Any new failure beyond these 36 is a regression
introduced by the last fix and must be investigated before committing.

---

## Rationale Report

A cumulative rationale document records the approach, references, and verification results for
every fix committed on this branch:

```
~/src/CTK/cmake-build-clazy/CTK-build/clazy-fix-report.md
```

This file lives in the build directory (not tracked in git) but can be shared with reviewers
when submitting the PR. It documents why each fix was made, what false positives were found,
and what incidental bugs were discovered — useful context for code review.

---

## Repository Structure

```
.claude/
├── README.md                    ← This file
├── CLAUDE.md                    ← Project context loaded by Claude Code
├── settings.local.json          ← Pre-approved tool permissions for CTK workflows
├── skills/
│   ├── clazy-configure/         ← /clazy-configure — CMake superbuild setup
│   ├── clazy-build/             ← /clazy-build    — incremental/clean builds
│   ├── clazy-check/             ← /clazy-check    — clazy-standalone analysis
│   │   ├── SKILL.md
│   │   ├── run_clazy_ctk.sh     ← Analysis runner script
│   │   └── split_fixes_by_check.py
│   ├── clazy-fix/               ← /clazy-fix      — fix one check per commit
│   ├── clazy-test/              ← /clazy-test     — CTest with known-failure classification
│   ├── cpp-coding-standards/    ← C++ Core Guidelines reference
│   └── cpp-testing/             ← CTest/GoogleTest helpers
└── commands/
    ├── pre-check-pr.md          ← Pre-flight checks before submitting a PR
    ├── convert-to-gtest.md      ← Convert CTests to GoogleTest format
    └── scan-build-report.md     ← clang scan-build analysis
```

---

## Contributing to This Skills Repo

Found a gap in a skill? Discovered a new CTK-specific pitfall? Improvements welcome:

```bash
cd ~/.claude/projects/CTK-claude-skills   # or wherever you cloned it
# Edit the relevant SKILL.md
git add -A && git commit -m "SKILL: improve clazy-fix with <description>"
git push
```

The skills use the same `STYLE:`/`BUG:`/`ENH:` prefix convention as CTK itself.
