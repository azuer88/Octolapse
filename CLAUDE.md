# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Octolapse is an OctoPrint plugin that creates stabilized timelapses of 3D prints by moving the print bed/extruder to ideal positions before capturing snapshots. It has Python plugin code, a Knockout.js web UI, and performance-critical C++ extension modules.

## Commands

### Install (development)
```bash
pip install -e .
```

### Build C++ extensions
```bash
python setup.py build_ext --inplace
```

### Run all tests
```bash
python -m unittest discover -s octoprint_octolapse/test -p "test_*.py"
```

### Run a single test file
```bash
python -m unittest octoprint_octolapse.test.test_position
```

### Run a single test case
```bash
python -m unittest octoprint_octolapse.test.test_position.TestPosition.test_method_name
```

Dependencies are defined in `setup.py` (not `requirements.txt`). Development dependency: `mock>=2.0.0`. External tools required at runtime: FFmpeg, mjpg-streamer.

## Architecture

### Plugin Entry Point
`octoprint_octolapse/__init__.py` (~3900 lines) — OctoPrint plugin class, all Flask API routes, event handlers, and plugin lifecycle. This is the integration layer between OctoPrint and Octolapse's internal logic.

### Core Logic Flow
1. **Settings & Profiles** (`settings.py`, ~3500 lines) — All configuration schemas, profile types (printer, camera, trigger, stabilization, rendering), and JSON serialization. Profile data flows into all other components.
2. **Preprocessing** (`settings_preprocessor.py`, ~2000 lines) — Before a print starts, parses the full GCode to generate a `SnapshotPlan`: a list of optimal snapshot positions. This uses the C++ extension for performance.
3. **Timelapse State Machine** (`timelapse.py`, ~1400 lines) — Coordinates all components during a print. Receives GCode commands, consults triggers, executes snapshots, manages threading.
4. **Triggers** (`trigger.py`, ~900 lines) — Determines *when* to take snapshots (layer change, timer, GCode command, smart/quality-based).
5. **Stabilization** (`stabilization_gcode.py`, ~1250 lines) — Generates the GCode to move bed/extruder to the snapshot position and back.
6. **Snapshot** (`snapshot.py`, ~900 lines) — Executes the actual capture workflow (runs before-scripts, captures image, runs after-scripts).
7. **Camera** (`camera.py`, ~1400 lines) — Controls webcam capture, mjpg-streamer interaction, image processing.
8. **Rendering** (`render.py`, ~2650 lines) — FFmpeg-based video composition from snapshots, including overlay/watermark support.

### C++ Extensions
Located in `octoprint_octolapse/data/lib/c/`. Compiled as the Python extension `octoprint_octolapse.gcode_parser`. Key files:
- `gcode_parser.cpp` — High-performance GCode tokenizer
- `gcode_position_processor.cpp` — Position state calculation
- `stabilization_*.cpp` — Stabilization algorithm variants
- `snapshot_plan.cpp` — Snapshot plan generation engine
- `trigger_position.cpp` — Trigger point calculation

### Web UI
- Jinja2 templates in `octoprint_octolapse/templates/` (40+ files)
- Knockout.js + jQuery in `octoprint_octolapse/static/js/`
- Settings dialogs, profile editors, and the main plugin tab are separate template files

### Supporting Modules
- `gcode_processor.py` / `gcode_commands.py` — GCode parsing utilities and command definitions (Python-level, wraps C++ parser)
- `position.py` — Python-level position state tracking
- `migration.py` — Settings migration between plugin versions
- `script.py` — Before/after snapshot script execution
- `utility.py` — File operations, calculations, helpers
- `error_messages.py` — Centralized error definitions
- `log.py` — Logging configuration
- `messenger_worker.py` — Message queue for async UI notifications

### Version Management
Uses `versioneer` — version is derived from git tags. Do not manually edit `_version.py`.

### Tests
`octoprint_octolapse/test/` — 19 `unittest`-based test modules. No pytest or tox config; use `unittest` discovery directly.
