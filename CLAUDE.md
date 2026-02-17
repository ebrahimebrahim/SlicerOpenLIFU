# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SlicerOpenLIFU is a 3D Slicer extension for Openwater's OpenLIFU (Low Intensity Focused Ultrasound) research platform. It provides a GUI for focused ultrasound treatment planning, simulation, and hardware control. Licensed under AGPL.

The extension depends on the `openlifu` Python library (`../OpenLIFU-python`), which provides the core computational engine (beamforming, simulation via k-Wave, data model classes). SlicerOpenLIFU wraps `openlifu` objects with Slicer UI, visualization, and persistence.

## Build and Test Commands

### Building

The extension is built against a 3D Slicer superbuild. Let `<slicer-superbuild>` denote the superbuild directory. Key paths within it:
- Slicer build: `<slicer-superbuild>/Slicer-build/`
- Slicer's bundled Python: `<slicer-superbuild>/python-install/bin/PythonSlicer`
- Run Python in Slicer env: `<slicer-superbuild>/Slicer-build/Slicer --python-code "..." --no-splash --no-main-window --exit-after-startup`

```bash
# Configure (from the repo root)
cmake -DSlicer_DIR=<slicer-superbuild>/Slicer-build \
      -DBUILD_TESTING=ON \
      -DDVC_GDRIVE_KEY_PATH=/path/to/gdrive-service-account.json \
      -B build

# Build
cmake --build build
```

`BUILD_TESTING=ON` requires `DVC_GDRIVE_KEY_PATH` pointing to a Google Drive service account JSON key (used for downloading test data via DVC).

### Running Slicer with the Extension

```bash
<slicer-superbuild>/Slicer-build/Slicer
```

The extension modules are automatically loaded from the build directory.

### Running Tests

Tests are integration tests that run inside Slicer's environment using CTest. They use Slicer's `ScriptedLoadableModuleTest` framework (not pytest).

```bash
# Run all tests
cd build && ctest --verbose

# Run a single module's test
cd build && ctest -R py_OpenLIFUHome --verbose

# Run just one module test (e.g. sonication planner)
cd build && ctest -R py_OpenLIFUSonicationPlanner --verbose
```

Test names follow the pattern `py_<ModuleName>`. The main integration test is `py_OpenLIFUHome`, which orchestrates a full workflow through all modules sequentially.

Test data is downloaded at runtime from Google Drive via DVC. The CMake config passes `GDRIVE_CREDENTIALS_DATA` and `DVC_REPO_DIR` as environment variables to CTest.

### Python Version

Slicer 5.10 embeds Python 3.12. This means PEP 701 f-string syntax (nested quotes) is valid, but the project has not yet adopted it widely.

### No Linter/Formatter Configuration

There is currently no configured linter or formatter (no flake8, ruff, black, or pre-commit setup).

## Architecture

### Module Structure

The extension has 9 feature modules plus a shared library, all as Slicer scripted (Python) modules:

| Module | Purpose |
|--------|---------|
| **OpenLIFUHome** | Entry point, guided workflow orchestration, python dependency installation |
| **OpenLIFUDatabase** | Connect to/create local file-based openlifu databases |
| **OpenLIFULogin** | User authentication, role-based access (operator/admin) |
| **OpenLIFUData** | Load subjects, sessions, protocols, transducers, volumes |
| **OpenLIFUPrePlanning** | Target placement, skin segmentation, virtual fit validation |
| **OpenLIFUTransducerLocalization** | Photogrammetry-based transducer tracking |
| **OpenLIFUSonicationPlanner** | Solution computation (beamforming + k-Wave simulation) |
| **OpenLIFUSonicationControl** | Hardware interface, run execution and recording |
| **OpenLIFUProtocolConfig** | Protocol creation/editing (admin only) |
| **OpenLIFULib** | Shared utility library (not a UI module) |

### Workflow Data Flow

```
Home → Database → Login → Data → PrePlanning → TransducerLocalization → SonicationPlanner → SonicationControl
                                       ↕
                                 ProtocolConfig
```

Each module can operate standalone, but in guided mode they follow this sequence.

### Standard Module Pattern (Slicer Convention)

Every module Python file contains four classes:

```python
class OpenLIFU<Name>(ScriptedLoadableModule):           # Module registration/metadata
class OpenLIFU<Name>ParameterNode(@parameterNodeWrapper): # Persistent state via MRML
class OpenLIFU<Name>Widget(ScriptedLoadableModuleWidget): # Qt UI
class OpenLIFU<Name>Logic(ScriptedLoadableModuleLogic):   # Business logic
class OpenLIFU<Name>Test(ScriptedLoadableModuleTest):     # Integration test
```

UI is defined in `Resources/UI/<ModuleName>.ui` (Qt Designer XML) and loaded in the Widget class.

### OpenLIFULib Key Patterns

**Lazy importing** (`lazyimport.py`): The `openlifu` library and its dependencies are heavy and may not be installed yet. All imports go through lazy-import functions:
- `openlifu_lz()` — returns the `openlifu` module, installing it first if needed
- `xarray_lz()`, `bcrypt_lz()`, `threadpoolctl_lz()` — same pattern
- For IDE type-checking, real imports are guarded under `if TYPE_CHECKING:`

**Parameter node wrappers** (`parameter_node_utils.py`): Thin wrapper classes (`SlicerOpenLIFUProtocol`, `SlicerOpenLIFUTransducer`, `SlicerOpenLIFUSession`, etc.) exist solely to enable Slicer's `@parameterNodeWrapper` serialization of `openlifu` types without importing `openlifu` at module load time. Each wrapper has a corresponding `@parameterNodeSerializer` class that serializes to/from JSON via the wrapped object's `to_json()`/`from_json()` methods.

**Cross-module communication**: Modules access each other's state through:
- `get_openlifu_data_parameter_node()` / `get_openlifu_database_parameter_node()` — access parameter nodes from other modules
- `get_cur_db()` — get the currently loaded `openlifu.db.Database`
- Callback registration: `logic.call_on_db_changed(callback)`, `logic.call_on_active_user_changed(callback)`
- VTK observers for MRML scene changes

**Guided workflow** (`guided_mode_util.py`): `GuidedWorkflowMixin` provides Back/Next/Jump navigation. Modules implement this mixin to participate in the guided workflow.

**User account mode** (`user_account_mode_util.py`): Widgets can be tagged with `slicer.openlifu.allowed-roles` Qt property. The Login module enforces visibility based on the current user's role.

### Relationship to openlifu Library

`openlifu` (`../OpenLIFU-python`, installed as `openlifu` package) provides:
- **Data model**: `Protocol`, `Transducer`, `Point`, `Session`, `Solution`, `Run`, `SolutionAnalysis`, `Photoscan`
- **Database**: File-based JSON database (`openlifu.db.Database`)
- **Beamforming**: Delay/apodization calculation (`openlifu.bf`)
- **Simulation**: k-Wave acoustic simulation (`openlifu.sim`)
- **Hardware I/O**: `openlifu.io.LIFUInterface` for device communication
- **Virtual fit**: `openlifu.vf` for transducer virtual fitting

SlicerOpenLIFU wraps these with Slicer MRML nodes, VTK visualization, Qt widgets, and parameter node persistence. The wrapper classes in `parameter_node_utils.py` bridge between openlifu's data classes and Slicer's parameter node system.

The pinned version is in `OpenLIFULib/OpenLIFULib/Resources/python-requirements.txt`.

### Testing Architecture

Tests are integration tests embedded in each module's main `.py` file as a `Test` class. `OpenLIFUHomeTest.runTest()` is the master test that:
1. Installs Python requirements if missing
2. Downloads test database from Google Drive via DVC
3. Calls each module's test methods sequentially (database → data → preplanning → localization → planning → control)

Individual module tests create state needed by subsequent tests — they are not independent.

## Commit Guidelines

- **Every commit must reference a relevant GitHub issue number** in the title or body (e.g. `Fix target placement crash (#42)` or with `Fixes #42` / `Relates to #42` in the body).
- When creating commits, always verify an issue number is included before finalizing.
- When reviewing code or PRs, check that every commit references an issue number and flag any that don't.
