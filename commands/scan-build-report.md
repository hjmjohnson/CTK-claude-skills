---
description: Run clang analyze-build (scan-build) on the ITK build directory, filter out ThirdParty findings, and post a structured report to a GitHub issue.
allowed-tools: Bash, Read, Write, Agent
---

# ITK clang analyze-build Report

Run LLVM's `analyze-build` static analyzer against an existing ITK build, exclude
`Modules/ThirdParty`, parse the SARIF output, and post a formatted summary to a
GitHub issue.

## Arguments

$ARGUMENTS

Expected forms (all optional — defaults shown):
- Build directory: path to an ITK CMake build with `compile_commands.json`
  (default: `cmake-build-clangtidy-22` relative to the repo root)
- LLVM root: path to the LLVM installation
  (default: `/opt/homebrew/Cellar/llvm/22.1.0`)
- GitHub issue number to update
  (default: `1261` on `InsightSoftwareConsortium/ITK`)

Parse `$ARGUMENTS` to extract these three values. Accept them positionally or as
`--build-dir`, `--llvm-root`, `--issue` flags.

## Step 1: Resolve Paths

```bash
# Defaults
REPO_ROOT=$(git rev-parse --show-toplevel)
BLDDIR="${REPO_ROOT}/cmake-build-clangtidy-22"
LLVM_ROOT="/opt/homebrew/Cellar/llvm/22.1.0"
ISSUE=1261
REPO="InsightSoftwareConsortium/ITK"
```

Verify required binaries exist:
- `${LLVM_ROOT}/bin/analyze-build`
- `${LLVM_ROOT}/bin/clang`
- `${BLDDIR}/compile_commands.json`

Abort with a clear error message if any are missing.

## Step 2: Run analyze-build

```bash
OUTDIR=$(mktemp -d /tmp/itk-scan-build-XXXXXXXX)
CLANG_VERSION=$("${LLVM_ROOT}/bin/clang" --version | head -1)

"${LLVM_ROOT}/bin/analyze-build" \
  --cdb "${BLDDIR}/compile_commands.json" \
  --use-analyzer "${LLVM_ROOT}/bin/clang" \
  --exclude "${BLDDIR}/Modules/ThirdParty" \
  --exclude "${REPO_ROOT}/Modules/ThirdParty" \
  --output "${OUTDIR}" \
  --sarif \
  --no-failure-reports \
  2>&1 | tee "${OUTDIR}/analyze-build.log"
```

This may take 10–30 minutes on a full ITK build. Run in the background if
the task framework supports it.

## Step 3: Parse SARIF Output

Locate the merged SARIF file:
```bash
SARIF=$(find "${OUTDIR}" -name "results-merged.sarif" | head -1)
```

Parse it with Python. For each result in the SARIF:
- Extract: `ruleId`, `level`, artifact URI, start line, message text
- **Skip** any result whose URI contains `ThirdParty` (belt-and-suspenders
  in addition to the `--exclude` flags)
- Strip the repo root prefix from URIs to produce short relative paths

Produce these aggregations:
1. Total finding count
2. Count by checker (`ruleId`)
3. Count by top-level module group (first two path components, e.g. `Modules/Core`)
4. Two categorized lists:
   - **High-priority**: all non-`deadcode.DeadStores` findings (table: checker, file, line, message)
   - **Dead stores**: grouped by file (collapsible details block)

Example Python parsing skeleton:
```python
import json
from collections import Counter

with open(sarif_path) as f:
    data = json.load(f)

rows = []
for run in data.get("runs", []):
    rules = {r["id"]: r for r in run["tool"]["driver"].get("rules", [])}
    for result in run.get("results", []):
        rule_id = result.get("ruleId", "")
        msg = result.get("message", {}).get("text", "")
        for loc in result.get("locations", []):
            uri = loc["physicalLocation"]["artifactLocation"]["uri"]
            line = loc["physicalLocation"]["region"]["startLine"]
            if "ThirdParty" in uri:
                continue
            short = uri.replace(f"file://{repo_root}/", "")
            rows.append((rule_id, short, line, msg))
```

## Step 4: Format the GitHub Comment

Build a Markdown comment with these sections:

```markdown
## clang-{VERSION} analyze-build report ({DATE})

**Tool:** `analyze-build` from {CLANG_VERSION}
**Build:** `{BLDDIR}` (all modules, BUILD_TESTING=ON)
**Exclusions:** `Modules/ThirdParty`

---

### Summary

| Checker | Count |
|---------|------:|
| `deadcode.DeadStores` | N |
| `core.NullDereference` | N |
| ... | ... |
| **Total** | **N** |

| Module Group | Count |
|-------------|------:|
| `Modules/Core` | N |
| ... | ... |

---

### High-priority findings (non-dead-store)

| Checker | File | Line | Message |
|---------|------|-----:|---------|
| `core.DivideZero` | `Modules/...` | 55 | Division by zero |
| ... | | | |

---

### Dead store findings (N total)

<details>
<summary>Full dead-store list</summary>

| File | Line | Variable |
|------|-----:|---------|
| `Modules/...` | 55 | `myVar` |

</details>
```

## Step 5: Post to GitHub Issue

```bash
gh issue comment "${ISSUE}" \
  --repo "${REPO}" \
  --body "${COMMENT_BODY}"
```

Print the resulting comment URL.

## Step 6: Report Summary to User

Print to stdout:
- Total findings (excl. ThirdParty)
- Count of high-priority vs dead-store findings
- The GitHub comment URL
- Path to the raw SARIF file for further inspection

## Error Handling

- If `compile_commands.json` is missing: suggest running
  `cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON .` from the build directory.
- If `analyze-build` binary is missing: suggest the LLVM version and
  homebrew install path.
- If the SARIF file is not found after the run: print the last 50 lines
  of `analyze-build.log` to diagnose the failure.
- If `gh issue comment` fails: print the full comment body to stdout so
  it can be posted manually.
