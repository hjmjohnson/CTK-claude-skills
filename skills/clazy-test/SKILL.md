---
name: clazy-test
description: Run the CTK test suite via CTest, capture all results to log files (JUnit XML + JSON), and report a summary of passed/failed/errored/skipped tests.
user_invocable: true
---

Run the CTK test suite and record all results to log files for analysis.

## Prerequisites

The CTK inner build must have completed successfully (via `/clazy-build`).

## Steps

### Step 1: Run CTest

```bash
BLD_DIR=~/src/CTK/cmake-build-clazy-qt6/CTK-build
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
TEST_LOG="${BLD_DIR}/test-${TIMESTAMP}.log"
JUNIT_XML="${BLD_DIR}/test-${TIMESTAMP}.xml"
```

Run CTest with JUnit XML output and a per-test timeout:

```bash
ctest --test-dir "${BLD_DIR}" \
  --timeout 30 \
  --output-on-failure \
  --output-junit "${JUNIT_XML}" \
  2>&1 | tee "${TEST_LOG}"
```

If the user provides a test name filter, append `-R <filter>`.
If the user provides an exclude filter, append `-E <filter>`.

### Step 2: Parse and summarize results

After CTest finishes, extract summary counts:

```bash
grep -E '^[0-9]+% tests passed' "${TEST_LOG}"
grep -E 'Failed|SEGFAULT|SIGTRAP' "${TEST_LOG}" | grep -oE '[0-9]+ - [^ ]+' | sed 's/[0-9]* - //' | sort
```

### Step 3: Classify failures

Compare the failure list against the **Known Pre-Existing Failures** below.

Report:
1. **Total**: X passed, Y failed out of N (percentage)
2. **Known failures**: list pre-existing failures (expected, not regressions)
3. **New failures**: any failure NOT in the known list — these are regressions requiring immediate investigation
4. **Log paths**: test log and JUnit XML

### Step 4: Offer next steps

- If no new failures: report clean run, suggest `/clazy-check` or `/clazy-fix` to continue
- If new failures: identify which tests are regressions, investigate root cause, fix and retest
- Offer to run a filtered test to investigate a specific failure: `/clazy-test DICOM`

## Known Pre-Existing Failures

These tests fail consistently and are NOT regressions. As of 2026-03 on `apply_clazy_skills` branch, the baseline is **36 failures out of 258 tests (86% pass rate)**.

**Widget tests:**
- **ctkWorkflowWidgetTest1/2** — Qt6 QStateMachine async transitions; widgets not visible in time
- **ctkWidgetsUtilsTestGrabWidget** — OpenGL/display infrastructure dependency
- **ctkLanguageComboBoxTest** — Qt6 `QLocale::name()` format change; needs both impl and test updates
- **ctkPathListWidgetWithButtonsTest** — QRunnable thread accessing GUI from worker thread
- **ctkBooleanMapperTest** — platform-dependent
- **ctkDoubleRangeSliderValueProxyTest** — platform-dependent precision
- **ctkDoubleSpinBoxTest / ctkDoubleSpinBoxTest1** — platform-dependent
- **ctkDoubleSpinBoxValueProxyTest** — platform-dependent precision
- **ctkCrosshairLabelTest2** — display-dependent (OpenGL)
- **ctkMaterialPropertyWidgetTest1/2** — OpenGL context
- **ctkRangeWidgetTest** — platform-dependent
- **ctkSliderWidgetTest1/2** — platform-dependent signal counts
- **ctkSliderWidgetValueProxyTest** — platform-dependent

**Settings tests:**
- **ctkSettingsDialogTest1** — `settingChanged` → `oneSettingChanged` rename from prior NOTIFY fix
- **ctkSettingsPanelTest** — same rename
- **ctkSettingsPanelTest1** — same rename

**DICOM tests (all SEGFAULT — pre-existing on master):**
- **ctkDICOMDatabaseTest2** — SEGFAULT
- **ctkDICOMSchedulerTest1** — SEGFAULT
- **ctkDICOMAppWidgetTest1** — SEGFAULT
- **ctkDICOMBrowserTest / ctkDICOMBrowserTest1** — SEGFAULT
- **ctkDICOMVisualBrowserWidgetTest1** — SEGFAULT

**VTK OpenGL tests (all SEGFAULT or display-dependent):**
- **vtkLightBoxRendererManagerTest1** — GPU/GLSL infrastructure (SIGTRAP)
- **ctkVTKPropertyWidgetTest** — SEGFAULT
- **ctkVTKThresholdWidgetTest1** — Widget logic bug ("19 19" vs "11 19")
- **ctkVTKVolumePropertyWidgetTest1** — SEGFAULT
- **ctkVTKDiscretizableColorTransferWidgetTest1** — SEGFAULT
- **ctkVTKSurfaceMaterialPropertyWidgetTest1** — SEGFAULT
- **ctkVTKMagnifyViewTest2OddOdd / EvenEven / OddEven / EvenOdd** — SEGFAULT (4 variants)

When reporting results, clearly mark each failure as `[known]` or `[NEW REGRESSION]`.

## Arguments

Optional: a test name filter (regex passed to `ctest -R`).

Examples:
- `/clazy-test` — run all tests
- `/clazy-test DICOM` — run only DICOM-related tests
- `/clazy-test Widgets` — run only Widget tests
- `/clazy-test ctkLayout` — run only layout-related tests
