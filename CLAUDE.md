# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CTK (Common Toolkit) is a community-driven C++ toolkit supporting medical image analysis and surgical navigation. It is built on Qt (5 or 6) and provides DICOM support, visualization widgets, an OSGi-based plugin framework, Python scripting, and command-line module infrastructure.

## Build System
Primary build directories are
~/src/CTK/cmake-build-clazy-qt5 for Qt5 build
~/src/CTK/cmake-build-clazy-qt6 for Qt6 build

CMake-based superbuild. By default (`CTK_SUPERBUILD=ON`), external dependencies (DCMTK, etc.) are automatically downloaded and built.

After making changes, verify that builds do not fail for ~/src/CTK/cmake-build-clazy-qt5/CTK-build and ~/src/CTK/cmake-build-clazy-qt6/CTK-build.

**Typical configure + build:**
```bash
export CTK_QT_VERSION=5
export CTK_QT_VERSION=6

mkdir -p ~/src/CTK/cmake-build-clazy-qt${CTK_QT_VERSION} && cmake \
     -DCTK_QT_VERSION:STRING=${CTK_QT_VERSION} \
     -DQt${CTK_QT_VERSION}_DIR:PATH=/opt/homebrew/lib/cmake/Qt${CTK_QT_VERSION} \
     -DCTK_ENABLE_Widgets:BOOL=ON \
     -DCTK_LIB_DICOM/Core:BOOL=ON \
     -DCTK_LIB_DICOM/Widgets:BOOL=ON \
     -DCTK_LIB_Visualization/VTK/Core:BOOL=ON \
     -DCTK_LIB_Visualization/VTK/Widgets:BOOL=ON \
     -DCTK_LIB_ImageProcessing/ITK/Core:BOOL=ON \
     -DCTK_LIB_Scripting/Python/Core:BOOL=ON \
     -DCTK_LIB_Scripting/Python/Widgets:BOOL=ON \
     -DCTK_USE_QTTESTING:BOOL=ON \
     -DCMAKE_C_COMPILER:FILEPATH=/opt/homebrew/opt/llvm/bin/clang \
     -DCMAKE_CXX_COMPILER:FILEPATH=/opt/homebrew/opt/llvm/bin/clang++ \
     -DCMAKE_OSX_SYSROOT:PATH=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
     -DBUILD_TESTING:BOOL=ON \
     -B ~/src/CTK/cmake-build-clazy-qt${CTK_QT_VERSION} \
     -S ~/src/CTK 2>&1 | tail -30
```

Key CMake options:
- `CTK_QT_VERSION`: `5` or `6` (defaults to 5 unless `Qt6_DIR` is set)
- `CTK_SUPERBUILD`: `ON` (default) — fetches/builds dependencies automatically
- `CTK_LIB_*` / `CTK_APP_*` / `CTK_ENABLE_*`: Enable individual modules and apps
- `BUILD_TESTING`: `ON` (default)

Qt auto-tools (`AUTOMOC`, `AUTOUIC`, `AUTORCC`) are enabled globally — no need to manually list `.moc` files.

## Running Tests

```bash
ctest --test-dir CTK-build
# Run a single test by name:
ctest --test-dir CTK-build -R <TestName>
```

Tests are registered via `SIMPLE_TEST(testname [args...])` in per-module `CMakeLists.txt` files. Test source lives under `Testing/Cpp/` inside each library or application directory.

## Code Architecture

### Directory Layout

| Path | Purpose |
|------|---------|
| `CMake/` | Reusable CMake macros for building libs, plugins, apps, and tests |
| `CMakeExternals/` | ExternalProject definitions for third-party dependencies |
| `Libs/` | Core C++ libraries (see below) |
| `Applications/` | Standalone executable applications |
| `Plugins/` | OSGi-based plugin modules |
| `Documentation/` | Doxygen configuration |

### Key Libraries (`Libs/`)

- **Core** — base utilities: factories, logging, settings, threading, dependency graphs, callback framework
- **Widgets** — 100+ custom Qt widgets (sliders, dialogs, panels, tables)
- **DICOM** — DICOM database, query/retrieve, indexing, server echo; split into `Core` and `Widgets` sub-libraries
- **PluginFramework** — OSGi-inspired dynamic plugin system; plugins live under `Plugins/org.commontk.*`
- **Scripting/Python** — PythonQt-based scripting integration
- **Visualization/VTK** — VTK integration and custom VTK-backed widgets
- **CommandLineModules** — Framework for wrapping CLI tools as modules
- **QtTesting** — Qt-based GUI test recorder/player

### CMake Build Macros

All modules use macros defined in `CMake/`:
- `ctkMacroBuildLib` — builds a library target
- `ctkMacroBuildPlugin` — builds an OSGi plugin with manifest
- `ctkMacroBuildApp` — builds an application
- `ctkMacroSimpleTest` / `ctkMacroSimpleTestWithData` — registers CTest tests

### Q_PROPERTY Conventions

Properties must have either `NOTIFY <signal>` or `CONSTANT`. Use the `CTK_SET_CPP_EMIT` macro for setter implementations that emit a change signal. Classes with properties must have `Q_OBJECT`.

## Commit Message Format

Subject lines are enforced by a commit-msg hook. Required format:

```
<PREFIX>: Short description (≤ 78 characters)
```

Valid prefixes: `ENH:`, `BUG:`, `COMP:`, `DOC:`, `STYLE:`, `PERF:`, `WIP:`

## Code Formatting

```bash
# Format all modified C++ files:
Utilities/Maintenance/clang-format.bash --modified

# Or run all pre-commit hooks:
pre-commit run --all-files
```

Pre-commit checks include large-file detection, merge-conflict markers, trailing whitespace, and line-ending consistency.

## VTK Visualization

When enabling `CTK_LIB_Visualization/VTK/Widgets`, VTK must be configured with:
- `VTK_MODULE_ENABLE_VTK_ChartsCore=YES`
- `VTK_MODULE_ENABLE_VTK_GUISupportQt=YES`
- `VTK_MODULE_ENABLE_VTK_ViewsQt=YES`

## Qt6 Migration Status

The `fix-test-suite-qt6` branch contains 16 test fixes validated on macOS ARM64 (Qt6). These fixes need cross-platform validation on Linux with both Qt5 and Qt6.

### Qt5 vs Qt6 Behavioral Differences (known)

| Area | Qt5 | Qt6 | Pattern |
|------|-----|-----|---------|
| `QSignalMapper` signal | `mapped(int)` | `mappedInt(int)` | `#if QT_VERSION` guard |
| `QFont::Normal` weight | 50 | 400 (CSS-aligned) | `#if QT_VERSION` guard |
| Slider `valueChanged` (tracking off) | 1 signal on release | 2 signals (press + release) | `#if QT_VERSION` guard |
| `QApplication::quit()` | Closes modal dialogs | Does NOT close modal dialogs | Use `activeModalWidget()->close()` or connect timer to dialog's `reject()` directly |
| `QRegExp` | Available | Removed | Use `QRegularExpression` (works in both) |

### Modal Dialog Test Pattern (Qt5+Qt6 safe)

Tests that call `dialog.exec()` must not rely on `QApplication::quit()` to dismiss the modal. Use one of:
```cpp
// Option A: When you have a reference to the dialog
QTimer::singleShot(delay, &dialog, [&dialog]() { dialog.reject(); });

// Option B: When the dialog is created internally (e.g., static getColor())
QTimer::singleShot(delay, &app, [&app]() {
  if (QWidget* modal = app.activeModalWidget())
    modal->close();
});

// Option C: QTimer parented to the dialog (fires within modal event loop)
QTimer* closeTimer = new QTimer(&dialog);
closeTimer->setSingleShot(true);
closeTimer->setInterval(delay);
QObject::connect(closeTimer, &QTimer::timeout, &dialog, &QDialog::reject);
closeTimer->start();
```

### Known Pre-Existing Failures (unfixed, present on master)

These are NOT regressions from the Qt6 branch — they fail on master too:
- **ctkWorkflowWidgetTest1/2** — Qt6 `QStateMachine` async transitions; widgets not visible in time
- **ctkVTKThresholdWidgetTest1** — Widget logic bug ("19 19" vs "11 19")
- **ctkWidgetsUtilsTestGrabWidget** — OpenGL/display infrastructure dependency
- **ctkLanguageComboBoxTest** — Qt6 `QLocale::name()` format change; needs both impl and test updates
- **ctkPathListWidgetWithButtonsTest** — `QRunnable` thread accessing GUI from worker thread
- **VTK OpenGL SEGFAULTs** — `vtkLightBoxRendererManager`, `ctkVTKVolumePropertyWidget`, `ctkVTKDiscretizableColorTransferWidget`, `ctkVTKMagnifyView` variants — GPU/GLSL infrastructure (no display in CI)

### Recommended Test Invocation

```bash
# Run all tests with a timeout to catch modal dialog hangs
ctest --test-dir CTK-build --timeout 30 --output-on-failure
```

## CI

GitHub Actions (`.github/workflows/ci.yml`) builds on Ubuntu, tests both Qt 5 and Qt 6 configurations. A pre-commit configuration (`.pre-commit-config.yaml`) and commit-message lint workflow are also active.

## TODO:
- [ ] Could probably consider dropping VTK8 support here and even possibly early VTK9. Specifically in the context of 3D Slicer, latest preview for Slicer supports VTK 9.4+ (aka 9.6, 9.5, 9.4)

