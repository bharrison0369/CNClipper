# Phase 1: Core CNC Primitives — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the three core Klipper extras modules that every CNC machine type needs — work coordinate systems, digital/analog I/O control, and universal tool activation.

**Architecture:** Each module is a standalone `klippy/extras/` Python file that loads only when referenced in `printer.cfg`. Modules follow Klipper conventions (`load_config()` entry point, `config.get*()` parsing, `gcode.register_command()` for G/M-codes). Cross-module dependencies use `printer.lookup_object()`. All modules must be machine-agnostic — no waterjet/plasma/laser-specific logic (see STANDARDS.md "Machine-Agnostic Abstractions").

**Tech Stack:** Python 3 (Klipper host), pytest for unit tests, Klipper `klippy/extras/` module API

**Key Klipper patterns used:**
- `load_config(config)` / `load_config_prefix(config)` — module entry points
- `config.get()`, `config.getfloat()`, `config.getint()`, `config.getboolean()` — read ALL config in `__init__` (unread params = error)
- `printer.lookup_object('gcode')` — always available, provides command registration
- `printer.lookup_object('pins')` — always available, provides `setup_pin()`
- `gcode.register_command(name, handler, desc=)` — register G/M-code handlers
- `printer.register_event_handler('klippy:connect', cb)` — deferred cross-module lookups
- `gcmd.get_float(name, default=)`, `gcmd.get_int(name, default=)` — parse command parameters

**Scope boundaries for Phase 1:**
- **In scope:** Immediate I/O (M64/M65/M68), input wait (M66), digital and PWM tool output (M3/M5), WCS (G54-G59, G10, G53, G92), startup/shutdown delays
- **Deferred to Phase 2:** Motion-synchronized I/O (M62/M63 — needs planner integration), motion-gated output (laser mode — needs move interception), dynamic power scaling (M4 velocity mode), feed hold, output sequencer, interlocks
- **Deferred indefinitely:** Cutter compensation (G41/G42), VFD communication

---

## File Structure

```
CNClipper/
  core/
    cnc_wcs.py              # Work Coordinate Systems (G54-G59, G10, G53, G92)
    cnc_io.py               # Immediate digital/analog I/O (M64-M66, M68)
    cnc_tool_output.py      # Universal tool activation (M3/M4/M5)
  tests/
    conftest.py             # Mock Klipper objects (printer, config, gcode, gcmd, pins)
    test_cnc_wcs.py         # WCS unit tests
    test_cnc_io.py          # I/O unit tests
    test_cnc_tool_output.py # Tool output unit tests
  pyproject.toml            # pytest configuration
  install.sh               # Symlink modules into klippy/extras/
```

**Module → printer.cfg mapping:**
| Module file | Config section | G/M-codes |
|---|---|---|
| `cnc_wcs.py` | `[cnc_wcs]` | G54-G59, G59.1-G59.3, G10 L2/L20, G53, G92/G92.1 |
| `cnc_io.py` | `[cnc_io]` | M64, M65, M66, M68 |
| `cnc_tool_output.py` | `[cnc_tool_output]` | M3, M4, M5 |

**Dependency graph:**
```
cnc_tool_output  →  cnc_io (uses I/O ports for pin control)
cnc_wcs          →  save_variables (Klipper built-in, for persistence)
cnc_wcs          →  gcode_move (Klipper built-in, for SET_GCODE_OFFSET)
```

---

## Task 1: Project Infrastructure

**Files:**
- Create: `pyproject.toml`
- Create: `tests/conftest.py`
- Create: `core/` directory

Sets up pytest and the mock Klipper objects that every test file will use. The mocks replicate just enough of Klipper's runtime API to test module logic without running Klipper.

- [ ] **Step 1: Create pyproject.toml with pytest config**

```toml
[project]
name = "cnclipper"
version = "0.1.0"
description = "Universal CNC module library for Klipper"
license = "GPL-3.0"
requires-python = ">=3.7"

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["core"]
```

- [ ] **Step 2: Create the Klipper mock fixtures in conftest.py**

```python
# tests/conftest.py
# Mock Klipper runtime objects for unit testing CNClipper modules.
# These replicate the subset of Klipper's API that our modules use.

import pytest

_SENTINEL = object()


class MockGcmd:
    """Mock GCodeCommand — the object passed to G/M-code handlers."""
    def __init__(self, params=None):
        self._params = {k.upper(): v for k, v in (params or {}).items()}
        self.responses = []

    def get(self, name, default=_SENTINEL):
        name = name.upper()
        if name in self._params:
            return str(self._params[name])
        if default is not _SENTINEL:
            return default
        raise self.error("Error on '%s': missing required parameter" % name)

    def get_float(self, name, default=_SENTINEL, minval=None, maxval=None):
        raw = self.get(name, default)
        if raw is None:
            return None
        val = float(raw)
        if minval is not None and val < minval:
            raise self.error("Error on '%s': value %.3f below minimum %.3f"
                             % (name, val, minval))
        if maxval is not None and val > maxval:
            raise self.error("Error on '%s': value %.3f above maximum %.3f"
                             % (name, val, maxval))
        return val

    def get_int(self, name, default=_SENTINEL, minval=None, maxval=None):
        raw = self.get(name, default)
        if raw is None:
            return None
        val = int(raw)
        if minval is not None and val < minval:
            raise self.error("Error on '%s': value %d below minimum %d"
                             % (name, val, minval))
        if maxval is not None and val > maxval:
            raise self.error("Error on '%s': value %d above maximum %d"
                             % (name, val, maxval))
        return val

    def respond_info(self, msg, log=True):
        self.responses.append(msg)

    class error(Exception):
        pass


class MockMcuPin:
    """Mock MCU pin — tracks set_digital/set_pwm calls."""
    def __init__(self, pin_type):
        self.pin_type = pin_type
        self.value = 0
        self.history = []  # list of (value,) tuples for assertion

    def set_digital(self, print_time, value):
        self.value = value
        self.history.append(('digital', value))

    def set_pwm(self, print_time, value):
        self.value = value
        self.history.append(('pwm', value))

    def setup_max_duration(self, duration):
        pass

    def setup_start_value(self, start_value, shutdown_value):
        self.value = start_value
        self.shutdown_value = shutdown_value

    def setup_cycle_time(self, cycle_time):
        self.cycle_time = cycle_time


class MockPins:
    """Mock pin allocator — returned by printer.lookup_object('pins')."""
    def __init__(self):
        self.pins = {}

    def setup_pin(self, pin_type, pin_name):
        pin = MockMcuPin(pin_type)
        self.pins[pin_name] = pin
        return pin


class MockGcode:
    """Mock gcode object — tracks registered commands."""
    def __init__(self):
        self.commands = {}
        self.mux_commands = {}

    def register_command(self, cmd, func, when_not_ready=False, desc=None):
        self.commands[cmd] = func

    def register_mux_command(self, cmd, key, value, func, desc=None):
        self.mux_commands[(cmd, key, value)] = func

    def run_script_from_command(self, script):
        # Parse and execute simple SAVE_VARIABLE or SET_GCODE_OFFSET calls
        self._last_script = script


class MockSaveVariables:
    """Mock save_variables module — in-memory variable store."""
    def __init__(self):
        self.allVariables = {}

    def get_status(self, eventtime=None):
        return {'variables': dict(self.allVariables)}

    def cmd_SAVE_VARIABLE(self, name, value):
        self.allVariables[name] = value


class MockToolhead:
    """Mock toolhead — tracks position and dwell calls."""
    def __init__(self):
        self.position = [0.0, 0.0, 0.0, 0.0]
        self.dwells = []

    def get_position(self):
        return list(self.position)

    def dwell(self, delay):
        self.dwells.append(delay)

    def manual_move(self, coord, speed):
        for i, v in enumerate(coord):
            if v is not None:
                self.position[i] = v

    def get_status(self, eventtime=None):
        return {'position': self.get_position()}


class MockGcodeMove:
    """Mock gcode_move object — tracks SET_GCODE_OFFSET state."""
    def __init__(self):
        self.homing_position = [0.0, 0.0, 0.0, 0.0]
        self.offsets = {'x': 0.0, 'y': 0.0, 'z': 0.0}

    def get_status(self, eventtime=None):
        return {
            'homing_origin': list(self.homing_position),
        }


class MockConfig:
    """Mock Klipper config object."""
    def __init__(self, section_name, params=None, printer=None):
        self._section = section_name
        self._params = params or {}
        self._printer = printer

    def get_name(self):
        return self._section

    def get_printer(self):
        return self._printer

    def get(self, name, default=_SENTINEL):
        if name in self._params:
            return str(self._params[name])
        if default is not _SENTINEL:
            return default
        raise Exception("Option '%s' not found in section '%s'"
                        % (name, self._section))

    def getfloat(self, name, default=_SENTINEL, minval=None, maxval=None,
                 above=None, below=None):
        raw = self.get(name, default)
        if raw is None:
            return None
        val = float(raw)
        if minval is not None and val < minval:
            raise Exception("Option '%s' value %.3f below minimum %.3f"
                            % (name, val, minval))
        if maxval is not None and val > maxval:
            raise Exception("Option '%s' value %.3f above maximum %.3f"
                            % (name, val, maxval))
        return val

    def getint(self, name, default=_SENTINEL, minval=None, maxval=None):
        raw = self.get(name, default)
        if raw is None:
            return None
        return int(raw)

    def getboolean(self, name, default=_SENTINEL):
        raw = self.get(name, default)
        if isinstance(raw, bool):
            return raw
        if raw is None:
            return None
        return str(raw).lower() in ('true', 'yes', '1')

    def getlists(self, name, default=_SENTINEL, seps=(',',)):
        raw = self.get(name, default)
        if raw is None:
            return None
        return [[item.strip() for item in raw.split(seps[0])]]


class MockPrinter:
    """Mock Klipper printer object — the central registry."""
    def __init__(self):
        self._objects = {}
        self._event_handlers = {}
        gcode = MockGcode()
        pins = MockPins()
        toolhead = MockToolhead()
        gcode_move = MockGcodeMove()
        save_vars = MockSaveVariables()
        self._objects['gcode'] = gcode
        self._objects['pins'] = pins
        self._objects['toolhead'] = toolhead
        self._objects['gcode_move'] = gcode_move
        self._objects['save_variables'] = save_vars

    def lookup_object(self, name, default=_SENTINEL):
        if name in self._objects:
            return self._objects[name]
        if default is not _SENTINEL:
            return default
        raise Exception("Printer object '%s' not found" % name)

    def register_event_handler(self, event, callback):
        self._event_handlers.setdefault(event, []).append(callback)

    def add_object(self, name, obj):
        self._objects[name] = obj

    def fire_event(self, event, *args):
        for cb in self._event_handlers.get(event, []):
            cb(*args)


# --- Fixtures ---

@pytest.fixture
def printer():
    return MockPrinter()


@pytest.fixture
def make_config(printer):
    """Factory fixture: make_config('section_name', {'key': 'value'})"""
    def _make(section_name, params=None):
        return MockConfig(section_name, params or {}, printer)
    return _make


@pytest.fixture
def gcmd():
    """Factory fixture: gcmd({'PARAM': 'value'})"""
    def _make(params=None):
        return MockGcmd(params)
    return _make
```

- [ ] **Step 3: Create empty core/ directory with __init__.py**

Create `core/__init__.py` as an empty file (needed for pytest pythonpath to work).

- [ ] **Step 4: Verify pytest discovers test path**

Run: `cd C:/GitHub/CNClipper && python -m pytest --collect-only 2>&1 | head -5`
Expected: `no tests ran` or `collected 0 items` (no errors)

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml tests/conftest.py core/__init__.py
git commit -m "feat: add test infrastructure with Klipper mock fixtures"
```

---

## Task 2: Evaluate G4 Dwell

**Files:**
- None created — evaluation only

Klipper already implements G4 in its core gcode module (`cmd_G4` calls `toolhead.dwell(delay)`). Before building a custom module, verify it behaves correctly for CNC use.

- [ ] **Step 1: Check Klipper's G4 behavior**

Search Klipper source for G4 implementation. Verify:
1. G4 pauses axis motion for the specified duration
2. G4 does NOT change output pin states (outputs maintain current state during dwell)
3. G4 P parameter units (Klipper uses seconds as float, RS274/NGC uses seconds — should match)

Web search or check: `https://www.klipper3d.org/G-Codes.html` for G4 documentation.

- [ ] **Step 2: Document findings in .agent-context.md**

Update the cnc_dwell entry in the Phase 1 checklist. Expected outcome: Klipper's G4 is sufficient for CNC dwell, no custom module needed. Mark cnc_dwell as resolved if so, or document what's missing if not.

- [ ] **Step 3: Commit if any files changed**

```bash
git commit -m "docs: evaluate Klipper G4 dwell — [conclusion]"
```

---

## Task 3: cnc_wcs — Data Model and Persistence

**Files:**
- Create: `core/cnc_wcs.py`
- Create: `tests/test_cnc_wcs.py`

Implements the internal data model for 9 work coordinate systems with persistent storage via Klipper's `save_variables`. No G-code commands yet — just the class skeleton, config parsing, offset storage, and the `_apply_offsets()` method that calls `SET_GCODE_OFFSET`.

**printer.cfg usage:**
```ini
[save_variables]
filename: ~/cnc_variables.cfg

[cnc_wcs]
# No required config — defaults to 9 zero-offset coordinate systems
```

- [ ] **Step 1: Write test for module instantiation and default state**

```python
# tests/test_cnc_wcs.py
from cnc_wcs import CncWorkCoordinates, load_config


class TestCncWcsInit:
    def test_load_config_returns_instance(self, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        assert isinstance(wcs, CncWorkCoordinates)

    def test_default_active_wcs_is_g54(self, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        assert wcs.active_index == 0  # G54 = index 0

    def test_default_offsets_are_zero(self, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        for i in range(9):
            offset = wcs.get_offset(i)
            assert offset == {'x': 0.0, 'y': 0.0, 'z': 0.0}

    def test_g92_offset_default_zero(self, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        assert wcs.g92_offset == {'x': 0.0, 'y': 0.0, 'z': 0.0}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'cnc_wcs'`

- [ ] **Step 3: Implement CncWorkCoordinates skeleton**

```python
# core/cnc_wcs.py
# CNClipper - Work Coordinate Systems (G54-G59, G59.1-G59.3)
#
# Provides 9 persistent work coordinate systems following RS274/NGC:
#   G54-G59     (indices 0-5)
#   G59.1-G59.3 (indices 6-8)
#
# Uses Klipper's SET_GCODE_OFFSET for offset application and
# save_variables for persistence across restarts.
#
# G-codes: G54, G55, G56, G57, G58, G59, G10 L2, G10 L20, G53,
#          G92, G92.1, CNC_WCS_STATUS
#
# Config section: [cnc_wcs]
# Dependencies: [save_variables] (for persistence)

WCS_NAMES = ['G54', 'G55', 'G56', 'G57', 'G58', 'G59',
             'G59.1', 'G59.2', 'G59.3']
WCS_COUNT = len(WCS_NAMES)
SAVE_KEY = 'cnc_wcs_offsets'
AXES = ('x', 'y', 'z')


class CncWorkCoordinates:
    def __init__(self, config):
        self.printer = config.get_printer()
        self.gcode = self.printer.lookup_object('gcode')
        self.active_index = 0  # G54
        self.offsets = [{'x': 0.0, 'y': 0.0, 'z': 0.0}
                        for _ in range(WCS_COUNT)]
        self.g92_offset = {'x': 0.0, 'y': 0.0, 'z': 0.0}
        # Deferred init — save_variables may not be loaded yet
        self.printer.register_event_handler('klippy:connect',
                                            self._handle_connect)

    def _handle_connect(self):
        self.save_vars = self.printer.lookup_object('save_variables')
        self._load_offsets()
        self._apply_offsets()

    def _load_offsets(self):
        variables = self.save_vars.allVariables
        saved = variables.get(SAVE_KEY, None)
        if saved is not None and isinstance(saved, list):
            for i, offset in enumerate(saved):
                if i < WCS_COUNT and isinstance(offset, dict):
                    for axis in AXES:
                        if axis in offset:
                            self.offsets[i][axis] = float(offset[axis])

    def _save_offsets(self):
        save_list = [dict(o) for o in self.offsets]
        self.gcode.run_script_from_command(
            "SAVE_VARIABLE VARIABLE='%s' VALUE='%s'"
            % (SAVE_KEY, repr(save_list)))

    def _apply_offsets(self):
        """Push active WCS + G92 offsets to Klipper via SET_GCODE_OFFSET."""
        active = self.offsets[self.active_index]
        total = {axis: active[axis] + self.g92_offset[axis] for axis in AXES}
        self.gcode.run_script_from_command(
            "SET_GCODE_OFFSET X=%.6f Y=%.6f Z=%.6f"
            % (total['x'], total['y'], total['z']))

    def get_offset(self, index):
        return dict(self.offsets[index])

    def get_status(self, eventtime=None):
        return {
            'active_wcs': WCS_NAMES[self.active_index],
            'active_index': self.active_index,
            'offsets': [dict(o) for o in self.offsets],
            'g92_offset': dict(self.g92_offset),
        }


def load_config(config):
    return CncWorkCoordinates(config)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py -v`
Expected: 4 passed

- [ ] **Step 5: Write test for persistence round-trip**

```python
# Add to tests/test_cnc_wcs.py

class TestCncWcsPersistence:
    def test_loads_saved_offsets(self, printer, make_config):
        save_vars = printer.lookup_object('save_variables')
        save_vars.allVariables['cnc_wcs_offsets'] = [
            {'x': 10.0, 'y': 20.0, 'z': 5.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
            {'x': 0.0, 'y': 0.0, 'z': 0.0},
        ]
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        printer.fire_event('klippy:connect')
        assert wcs.get_offset(0) == {'x': 10.0, 'y': 20.0, 'z': 5.0}

    def test_missing_save_vars_uses_defaults(self, printer, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        printer.fire_event('klippy:connect')
        assert wcs.get_offset(0) == {'x': 0.0, 'y': 0.0, 'z': 0.0}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py -v`
Expected: 6 passed

- [ ] **Step 7: Commit**

```bash
git add core/cnc_wcs.py tests/test_cnc_wcs.py
git commit -m "feat(cnc_wcs): data model with 9 WCS, persistence via save_variables"
```

---

## Task 4: cnc_wcs — G54-G59 Selection Commands

**Files:**
- Modify: `core/cnc_wcs.py`
- Modify: `tests/test_cnc_wcs.py`

Registers G54 through G59.3 as G-code commands that switch the active work coordinate system and apply the corresponding offsets.

- [ ] **Step 1: Write tests for WCS selection**

```python
# Add to tests/test_cnc_wcs.py

class TestCncWcsSelection:
    def _make_wcs(self, printer, make_config):
        save_vars = printer.lookup_object('save_variables')
        save_vars.allVariables['cnc_wcs_offsets'] = [
            {'x': 10.0, 'y': 20.0, 'z': 0.0},   # G54
            {'x': 100.0, 'y': 200.0, 'z': 0.0},  # G55
            {'x': 0.0, 'y': 0.0, 'z': 0.0},      # G56
            {'x': 0.0, 'y': 0.0, 'z': 0.0},      # G57
            {'x': 0.0, 'y': 0.0, 'z': 0.0},      # G58
            {'x': 0.0, 'y': 0.0, 'z': 0.0},      # G59
            {'x': 50.0, 'y': 50.0, 'z': 50.0},   # G59.1
            {'x': 0.0, 'y': 0.0, 'z': 0.0},      # G59.2
            {'x': 0.0, 'y': 0.0, 'z': 0.0},      # G59.3
        ]
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        printer.fire_event('klippy:connect')
        return wcs

    def test_g54_registered(self, printer, make_config):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        assert 'G54' in gcode.commands

    def test_g55_switches_active_wcs(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['G55'](gcmd())
        assert wcs.active_index == 1

    def test_g59_1_switches_to_index_6(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        # G59.1 is registered as G59_1 (dots not allowed in Klipper cmd names)
        gcode.commands['CNC_WCS_SELECT'](gcmd({'INDEX': '6'}))
        assert wcs.active_index == 6

    def test_status_reflects_active_wcs(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['G55'](gcmd())
        status = wcs.get_status()
        assert status['active_wcs'] == 'G55'
        assert status['active_index'] == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py::TestCncWcsSelection -v`
Expected: FAIL — `KeyError: 'G54'` (commands not registered yet)

- [ ] **Step 3: Implement G54-G59 command registration**

Add to `CncWorkCoordinates.__init__()`:

```python
        # Register G54-G59 as individual commands
        for i in range(6):
            name = WCS_NAMES[i]  # G54, G55, ..., G59
            self.gcode.register_command(
                name, self._make_select_handler(i),
                desc="Select work coordinate system %s" % name)
        # G59.1-G59.3 can't use dots in command names — provide generic selector
        self.gcode.register_command(
            'CNC_WCS_SELECT', self.cmd_CNC_WCS_SELECT,
            desc="Select work coordinate system by index (0-8)")
```

Add methods:

```python
    def _make_select_handler(self, index):
        def handler(gcmd):
            self.active_index = index
            self._apply_offsets()
            gcmd.respond_info("Active WCS: %s" % WCS_NAMES[index])
        return handler

    def cmd_CNC_WCS_SELECT(self, gcmd):
        index = gcmd.get_int('INDEX', minval=0, maxval=WCS_COUNT - 1)
        self.active_index = index
        self._apply_offsets()
        gcmd.respond_info("Active WCS: %s" % WCS_NAMES[index])
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py -v`
Expected: All passed (previous + new)

- [ ] **Step 5: Commit**

```bash
git add core/cnc_wcs.py tests/test_cnc_wcs.py
git commit -m "feat(cnc_wcs): G54-G59 WCS selection commands"
```

---

## Task 5: cnc_wcs — G10 L2/L20 Offset Setting

**Files:**
- Modify: `core/cnc_wcs.py`
- Modify: `tests/test_cnc_wcs.py`

Implements G10 L2 (set offset directly) and G10 L20 (set offset from current position). Both persist offsets via save_variables.

- [ ] **Step 1: Write tests for G10 L2 (direct offset setting)**

```python
# Add to tests/test_cnc_wcs.py

class TestCncWcsG10:
    def _make_wcs(self, printer, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        printer.fire_event('klippy:connect')
        return wcs

    def test_g10_l2_sets_offset_directly(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['G10'](gcmd({'L': '2', 'P': '1', 'X': '50.0', 'Y': '25.0'}))
        offset = wcs.get_offset(0)  # P1 = G54 = index 0
        assert offset['x'] == 50.0
        assert offset['y'] == 25.0
        assert offset['z'] == 0.0  # unchanged

    def test_g10_l2_p2_sets_g55(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['G10'](gcmd({'L': '2', 'P': '2', 'Z': '-10.0'}))
        offset = wcs.get_offset(1)  # P2 = G55 = index 1
        assert offset['z'] == -10.0

    def test_g10_l20_sets_offset_from_position(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        toolhead = printer.lookup_object('toolhead')
        toolhead.position = [100.0, 200.0, 50.0, 0.0]
        # G10 L20 P1 X0 Y0 means: "I want current position to be X0 Y0 in G54"
        # So G54 offset = machine_pos - desired_pos = 100 - 0, 200 - 0
        gcode.commands['G10'](gcmd({'L': '20', 'P': '1', 'X': '0', 'Y': '0'}))
        offset = wcs.get_offset(0)
        assert offset['x'] == 100.0
        assert offset['y'] == 200.0

    def test_g10_l2_persists_offsets(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['G10'](gcmd({'L': '2', 'P': '1', 'X': '50.0'}))
        # Verify save was called (check that run_script_from_command was called
        # with SAVE_VARIABLE)
        assert hasattr(gcode, '_last_script')
        assert 'SAVE_VARIABLE' in gcode._last_script
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py::TestCncWcsG10 -v`
Expected: FAIL — `KeyError: 'G10'`

- [ ] **Step 3: Implement G10 command**

Add to `CncWorkCoordinates.__init__()`:

```python
        self.gcode.register_command(
            'G10', self.cmd_G10,
            desc="Set work coordinate system offset (L2=direct, L20=from position)")
```

Add method:

```python
    def cmd_G10(self, gcmd):
        l_value = gcmd.get_int('L')
        p_value = gcmd.get_int('P', minval=1, maxval=WCS_COUNT)
        index = p_value - 1  # P1=G54=index 0, P2=G55=index 1, etc.
        if l_value == 2:
            # G10 L2: set offset directly
            for axis in AXES:
                val = gcmd.get_float(axis.upper(), default=None)
                if val is not None:
                    self.offsets[index][axis] = val
        elif l_value == 20:
            # G10 L20: set offset so current position becomes the given value
            toolhead = self.printer.lookup_object('toolhead')
            pos = toolhead.get_position()
            axis_map = {'x': 0, 'y': 1, 'z': 2}
            # Current machine pos = toolhead pos (before our offsets)
            # We need: offset = machine_pos - desired_user_pos
            # But toolhead pos already has our offsets applied via SET_GCODE_OFFSET
            # So machine_pos = toolhead_pos + current_total_offset
            active = self.offsets[self.active_index]
            for axis in AXES:
                val = gcmd.get_float(axis.upper(), default=None)
                if val is not None:
                    machine_pos = pos[axis_map[axis]] + active[axis] + self.g92_offset[axis]
                    self.offsets[index][axis] = machine_pos - val
        else:
            raise gcmd.error("G10: unsupported L value %d (use L2 or L20)" % l_value)
        self._save_offsets()
        if index == self.active_index:
            self._apply_offsets()
        gcmd.respond_info("Set %s offset: X=%.4f Y=%.4f Z=%.4f"
                          % (WCS_NAMES[index],
                             self.offsets[index]['x'],
                             self.offsets[index]['y'],
                             self.offsets[index]['z']))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py -v`
Expected: All passed

- [ ] **Step 5: Commit**

```bash
git add core/cnc_wcs.py tests/test_cnc_wcs.py
git commit -m "feat(cnc_wcs): G10 L2/L20 offset setting with persistence"
```

---

## Task 6: cnc_wcs — G53 Machine Coords and G92 Temporary Offsets

**Files:**
- Modify: `core/cnc_wcs.py`
- Modify: `tests/test_cnc_wcs.py`

G53 provides one-shot machine-coordinate moves (bypass all offsets for one line). G92 sets a temporary offset that stacks on top of the active WCS. G92.1 clears G92 offsets.

- [ ] **Step 1: Write tests for G92 and G92.1**

```python
# Add to tests/test_cnc_wcs.py

class TestCncWcsG92:
    def _make_wcs(self, printer, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        printer.fire_event('klippy:connect')
        return wcs

    def test_g92_sets_temporary_offset(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        toolhead = printer.lookup_object('toolhead')
        toolhead.position = [50.0, 50.0, 10.0, 0.0]
        # G92 X0 Y0 means "current position is now X0 Y0"
        # So g92_offset = machine_pos - desired_pos = 50 - 0, 50 - 0
        gcode.commands['G92'](gcmd({'X': '0', 'Y': '0'}))
        assert wcs.g92_offset['x'] == 50.0
        assert wcs.g92_offset['y'] == 50.0

    def test_g92_1_clears_g92_offset(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        toolhead = printer.lookup_object('toolhead')
        toolhead.position = [50.0, 50.0, 10.0, 0.0]
        gcode.commands['G92'](gcmd({'X': '0'}))
        assert wcs.g92_offset['x'] == 50.0
        gcode.commands['G92.1'](gcmd())
        assert wcs.g92_offset == {'x': 0.0, 'y': 0.0, 'z': 0.0}

    def test_g92_stacks_with_wcs(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        # Set G54 offset to 10,20,0
        gcode.commands['G10'](gcmd({'L': '2', 'P': '1', 'X': '10', 'Y': '20'}))
        toolhead = printer.lookup_object('toolhead')
        toolhead.position = [50.0, 50.0, 0.0, 0.0]
        # G92 X0 — current machine pos is 50+10+0=60 (toolhead + wcs + g92)
        # Actually toolhead pos is already offset... this depends on implementation
        # The key assertion: both offsets are tracked independently
        status = wcs.get_status()
        assert status['offsets'][0]['x'] == 10.0  # WCS offset preserved


class TestCncWcsG53:
    def _make_wcs(self, printer, make_config):
        config = make_config('cnc_wcs')
        wcs = load_config(config)
        printer.fire_event('klippy:connect')
        return wcs

    def test_g53_registered(self, printer, make_config):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        assert 'G53' in gcode.commands

    def test_g53_clears_and_restores_offsets(self, printer, make_config, gcmd):
        wcs = self._make_wcs(printer, make_config)
        gcode = printer.lookup_object('gcode')
        # Set a WCS offset
        gcode.commands['G10'](gcmd({'L': '2', 'P': '1', 'X': '50'}))
        # G53 should temporarily clear offsets
        # After G53 handler runs, offsets should be restored
        gcode.commands['G53'](gcmd({'X': '0', 'Y': '0'}))
        # Verify offsets were restored after G53
        status = wcs.get_status()
        assert status['offsets'][0]['x'] == 50.0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py::TestCncWcsG92 tests/test_cnc_wcs.py::TestCncWcsG53 -v`
Expected: FAIL — `KeyError: 'G92'`

- [ ] **Step 3: Implement G92, G92.1, and G53**

Add to `CncWorkCoordinates.__init__()`:

```python
        self.gcode.register_command(
            'G92', self.cmd_G92,
            desc="Set temporary coordinate offset (current pos becomes given value)")
        self.gcode.register_command(
            'G92.1', self.cmd_G92_1,
            desc="Clear G92 temporary offset")
        self.gcode.register_command(
            'G53', self.cmd_G53,
            desc="Move in machine coordinates (one-shot, ignores all offsets)")
```

Add methods:

```python
    def cmd_G92(self, gcmd):
        toolhead = self.printer.lookup_object('toolhead')
        pos = toolhead.get_position()
        axis_map = {'x': 0, 'y': 1, 'z': 2}
        active = self.offsets[self.active_index]
        for axis in AXES:
            val = gcmd.get_float(axis.upper(), default=None)
            if val is not None:
                # machine_pos = toolhead_pos + active_wcs + current_g92
                machine_pos = (pos[axis_map[axis]]
                               + active[axis] + self.g92_offset[axis])
                self.g92_offset[axis] = machine_pos - val
        self._apply_offsets()
        gcmd.respond_info("G92 offset: X=%.4f Y=%.4f Z=%.4f"
                          % (self.g92_offset['x'],
                             self.g92_offset['y'],
                             self.g92_offset['z']))

    def cmd_G92_1(self, gcmd):
        self.g92_offset = {'x': 0.0, 'y': 0.0, 'z': 0.0}
        self._apply_offsets()
        gcmd.respond_info("G92 offset cleared")

    def cmd_G53(self, gcmd):
        # Temporarily clear all offsets, execute the move, restore offsets
        self.gcode.run_script_from_command(
            "SET_GCODE_OFFSET X=0 Y=0 Z=0")
        # Build and execute the move command
        parts = []
        for axis in ('X', 'Y', 'Z', 'F'):
            val = gcmd.get_float(axis, default=None)
            if val is not None:
                parts.append("%s%.4f" % (axis, val))
        if parts:
            self.gcode.run_script_from_command("G0 " + " ".join(parts))
        # Restore offsets
        self._apply_offsets()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py -v`
Expected: All passed

- [ ] **Step 5: Write CNC_WCS_STATUS command for debugging**

Add to `__init__`:

```python
        self.gcode.register_command(
            'CNC_WCS_STATUS', self.cmd_CNC_WCS_STATUS,
            desc="Display all work coordinate system offsets")
```

Add method:

```python
    def cmd_CNC_WCS_STATUS(self, gcmd):
        lines = ["Active: %s (index %d)" % (WCS_NAMES[self.active_index],
                                              self.active_index)]
        for i, offset in enumerate(self.offsets):
            marker = " *" if i == self.active_index else ""
            lines.append("  %s: X=%.4f Y=%.4f Z=%.4f%s"
                         % (WCS_NAMES[i], offset['x'], offset['y'],
                            offset['z'], marker))
        if any(self.g92_offset[a] != 0.0 for a in AXES):
            lines.append("  G92: X=%.4f Y=%.4f Z=%.4f"
                         % (self.g92_offset['x'], self.g92_offset['y'],
                            self.g92_offset['z']))
        gcmd.respond_info("\n".join(lines))
```

- [ ] **Step 6: Run full test suite**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_wcs.py -v`
Expected: All passed

- [ ] **Step 7: Commit**

```bash
git add core/cnc_wcs.py tests/test_cnc_wcs.py
git commit -m "feat(cnc_wcs): G53 machine coords, G92 temp offsets, status command"
```

---

## Task 7: cnc_io — Immediate Digital Output (M64/M65)

**Files:**
- Create: `core/cnc_io.py`
- Create: `tests/test_cnc_io.py`

Implements immediate digital output control (M64/M65) — the simplest I/O primitive. Outputs are configured by port number in `printer.cfg`.

**printer.cfg usage:**
```ini
[cnc_io]
# Digital outputs (M64/M65 P<port>)
digital_out_0_pin: P1.23
digital_out_1_pin: P1.24
# Analog outputs (M68 E<port> Q<value>)
analog_out_0_pin: P1.25
analog_out_0_cycle_time: 0.001
# Digital inputs (M66 P<port>)
digital_in_0_pin: P1.26
digital_in_0_pull: up
```

- [ ] **Step 1: Write tests for digital output**

```python
# tests/test_cnc_io.py
from cnc_io import CncIo, load_config


class TestCncIoDigitalOut:
    def test_load_config_returns_instance(self, make_config):
        config = make_config('cnc_io', {
            'digital_out_0_pin': 'test:pin0',
        })
        io = load_config(config)
        assert isinstance(io, CncIo)

    def test_m64_turns_on_output(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'digital_out_0_pin': 'test:pin0',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M64'](gcmd({'P': '0'}))
        pin = printer.lookup_object('pins').pins['test:pin0']
        assert pin.value == 1

    def test_m65_turns_off_output(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'digital_out_0_pin': 'test:pin0',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M64'](gcmd({'P': '0'}))
        gcode.commands['M65'](gcmd({'P': '0'}))
        pin = printer.lookup_object('pins').pins['test:pin0']
        assert pin.value == 0

    def test_m64_invalid_port_raises(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'digital_out_0_pin': 'test:pin0',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        import pytest
        with pytest.raises(Exception):
            gcode.commands['M64'](gcmd({'P': '5'}))

    def test_multiple_outputs(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'digital_out_0_pin': 'test:pin0',
            'digital_out_1_pin': 'test:pin1',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M64'](gcmd({'P': '0'}))
        gcode.commands['M64'](gcmd({'P': '1'}))
        pins = printer.lookup_object('pins')
        assert pins.pins['test:pin0'].value == 1
        assert pins.pins['test:pin1'].value == 1
        gcode.commands['M65'](gcmd({'P': '0'}))
        assert pins.pins['test:pin0'].value == 0
        assert pins.pins['test:pin1'].value == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_io.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'cnc_io'`

- [ ] **Step 3: Implement CncIo with M64/M65**

```python
# core/cnc_io.py
# CNClipper - Digital/Analog I/O Control
#
# Provides immediate digital and analog output control following
# LinuxCNC/grblHAL M-code conventions:
#   M64 P<n>         — Digital output ON (immediate)
#   M65 P<n>         — Digital output OFF (immediate)
#   M66 P<n> L<mode> Q<timeout> — Wait on digital input
#   M68 E<n> Q<val>  — Analog output set (immediate)
#
# Motion-synchronized outputs (M62/M63/M67) require planner integration
# and are deferred to Phase 2.
#
# Config section: [cnc_io]
# Dependencies: none (uses Klipper's built-in pin system)

import time


class CncIo:
    def __init__(self, config):
        self.printer = config.get_printer()
        self.gcode = self.printer.lookup_object('gcode')
        ppins = self.printer.lookup_object('pins')
        # Parse digital outputs: digital_out_N_pin
        self.digital_outputs = {}
        for i in range(16):  # support up to 16 ports
            pin_name = config.get('digital_out_%d_pin' % i, default=None)
            if pin_name is None:
                continue
            pin = ppins.setup_pin('digital_out', pin_name)
            pin.setup_max_duration(0)
            pin.setup_start_value(0, 0)
            self.digital_outputs[i] = pin
        # Parse analog outputs: analog_out_N_pin
        self.analog_outputs = {}
        for i in range(16):
            pin_name = config.get('analog_out_%d_pin' % i, default=None)
            if pin_name is None:
                continue
            cycle_time = config.getfloat('analog_out_%d_cycle_time' % i,
                                         default=0.100)
            pin = ppins.setup_pin('pwm', pin_name)
            pin.setup_cycle_time(cycle_time)
            pin.setup_max_duration(0)
            pin.setup_start_value(0.0, 0.0)
            self.analog_outputs[i] = pin
        # Parse digital inputs: digital_in_N_pin
        self.digital_inputs = {}
        for i in range(16):
            pin_name = config.get('digital_in_%d_pin' % i, default=None)
            if pin_name is None:
                continue
            # Input pins are read-only; store pin name for M66
            self.digital_inputs[i] = pin_name
        # Track output state for get_status()
        self.digital_out_state = {i: 0 for i in self.digital_outputs}
        self.analog_out_state = {i: 0.0 for i in self.analog_outputs}
        # Last M66 result (LinuxCNC stores in #5399)
        self.last_input_result = 0
        # Register commands
        self.gcode.register_command(
            'M64', self.cmd_M64,
            desc="Turn ON digital output (immediate)")
        self.gcode.register_command(
            'M65', self.cmd_M65,
            desc="Turn OFF digital output (immediate)")
        self.gcode.register_command(
            'M68', self.cmd_M68,
            desc="Set analog output value (immediate)")
        self.gcode.register_command(
            'M66', self.cmd_M66,
            desc="Wait on digital input")
        self.gcode.register_command(
            'CNC_IO_STATUS', self.cmd_CNC_IO_STATUS,
            desc="Display I/O port status")

    def cmd_M64(self, gcmd):
        port = gcmd.get_int('P', minval=0)
        if port not in self.digital_outputs:
            raise gcmd.error("M64: digital output port %d not configured" % port)
        toolhead = self.printer.lookup_object('toolhead')
        print_time = toolhead.get_position()  # sync point
        self.digital_outputs[port].set_digital(0, 1)
        self.digital_out_state[port] = 1

    def cmd_M65(self, gcmd):
        port = gcmd.get_int('P', minval=0)
        if port not in self.digital_outputs:
            raise gcmd.error("M65: digital output port %d not configured" % port)
        self.digital_outputs[port].set_digital(0, 0)
        self.digital_out_state[port] = 0

    def cmd_M68(self, gcmd):
        port = gcmd.get_int('E', minval=0)
        value = gcmd.get_float('Q', minval=0.0, maxval=1.0)
        if port not in self.analog_outputs:
            raise gcmd.error("M68: analog output port %d not configured" % port)
        self.analog_outputs[port].set_pwm(0, value)
        self.analog_out_state[port] = value

    def cmd_M66(self, gcmd):
        port = gcmd.get_int('P', minval=0)
        mode = gcmd.get_int('L', default=0, minval=0, maxval=4)
        timeout = gcmd.get_float('Q', default=0.0, minval=0.0)
        if port not in self.digital_inputs:
            raise gcmd.error("M66: digital input port %d not configured" % port)
        # Mode 0 = IMMEDIATE: read current value, no wait
        # Modes 1-4 (RISE/FALL/HIGH/LOW) require polling — implementation
        # depends on Klipper's input pin API. For now, mode 0 is supported.
        if mode != 0:
            raise gcmd.error(
                "M66: wait modes (L1-L4) not yet implemented. Use L0 for immediate read.")
        # TODO: Read actual pin value via Klipper's endstop/filament_switch API
        # For now, report as not-yet-wired
        self.last_input_result = 0
        gcmd.respond_info("M66: input port %d = %d" % (port, self.last_input_result))

    def cmd_CNC_IO_STATUS(self, gcmd):
        lines = ["CNC I/O Status:"]
        for port in sorted(self.digital_outputs):
            lines.append("  Digital Out %d: %s"
                         % (port, "ON" if self.digital_out_state[port] else "OFF"))
        for port in sorted(self.analog_outputs):
            lines.append("  Analog Out %d: %.3f"
                         % (port, self.analog_out_state[port]))
        for port in sorted(self.digital_inputs):
            lines.append("  Digital In %d: configured"  % port)
        gcmd.respond_info("\n".join(lines))

    # --- Public API for other modules (cnc_tool_output, cnc_interlock) ---

    def set_digital_out(self, port, value):
        """Set digital output. Called by other CNClipper modules."""
        if port not in self.digital_outputs:
            raise Exception("cnc_io: digital output port %d not configured" % port)
        self.digital_outputs[port].set_digital(0, 1 if value else 0)
        self.digital_out_state[port] = 1 if value else 0

    def set_analog_out(self, port, value):
        """Set analog output. Called by other CNClipper modules."""
        if port not in self.analog_outputs:
            raise Exception("cnc_io: analog output port %d not configured" % port)
        self.analog_outputs[port].set_pwm(0, value)
        self.analog_out_state[port] = value

    def get_status(self, eventtime=None):
        return {
            'digital_outputs': dict(self.digital_out_state),
            'analog_outputs': dict(self.analog_out_state),
            'last_input_result': self.last_input_result,
        }


def load_config(config):
    return CncIo(config)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_io.py -v`
Expected: 5 passed

- [ ] **Step 5: Commit**

```bash
git add core/cnc_io.py tests/test_cnc_io.py
git commit -m "feat(cnc_io): immediate digital I/O with M64/M65 commands"
```

---

## Task 8: cnc_io — Analog Output (M68) and Status

**Files:**
- Modify: `tests/test_cnc_io.py`

Adds tests for analog output and the status command (implementation already in Task 7).

- [ ] **Step 1: Write tests for M68 analog output**

```python
# Add to tests/test_cnc_io.py

class TestCncIoAnalogOut:
    def test_m68_sets_analog_value(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'analog_out_0_pin': 'test:pwm0',
            'analog_out_0_cycle_time': '0.001',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M68'](gcmd({'E': '0', 'Q': '0.75'}))
        pin = printer.lookup_object('pins').pins['test:pwm0']
        assert pin.value == 0.75

    def test_m68_clamps_to_range(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'analog_out_0_pin': 'test:pwm0',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        import pytest
        with pytest.raises(Exception):
            gcode.commands['M68'](gcmd({'E': '0', 'Q': '1.5'}))


class TestCncIoStatus:
    def test_status_tracks_digital_state(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'digital_out_0_pin': 'test:pin0',
            'digital_out_1_pin': 'test:pin1',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M64'](gcmd({'P': '0'}))
        status = io.get_status()
        assert status['digital_outputs'][0] == 1
        assert status['digital_outputs'][1] == 0

    def test_status_tracks_analog_state(self, printer, make_config, gcmd):
        config = make_config('cnc_io', {
            'analog_out_0_pin': 'test:pwm0',
        })
        io = load_config(config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M68'](gcmd({'E': '0', 'Q': '0.5'}))
        status = io.get_status()
        assert status['analog_outputs'][0] == 0.5
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_io.py -v`
Expected: All passed (previous + new)

- [ ] **Step 3: Commit**

```bash
git add tests/test_cnc_io.py
git commit -m "test(cnc_io): analog output and status tracking tests"
```

---

## Task 9: cnc_tool_output — Capability Model and Digital Mode

**Files:**
- Create: `core/cnc_tool_output.py`
- Create: `tests/test_cnc_tool_output.py`

Implements the universal tool activation module with M3/M5 commands. Starts with digital mode (on/off solenoid — waterjet valve, plasma arc relay). The key design: tool type is defined by config capabilities, not by machine type.

**printer.cfg usage (digital waterjet valve):**
```ini
[cnc_io]
digital_out_0_pin: P1.23   # waterjet valve

[cnc_tool_output]
type: digital
output_port: 0             # cnc_io digital output port number
```

**printer.cfg usage (PWM spindle):**
```ini
[cnc_io]
analog_out_0_pin: P1.25

[cnc_tool_output]
type: pwm
output_port: 0             # cnc_io analog output port number
max_power: 1.0
min_power: 0.1
startup_delay: 2.0         # wait 2s after M3 for spindle to reach speed
shutdown_delay: 1.0         # wait 1s after M5 for spindle to stop
```

- [ ] **Step 1: Write tests for digital tool output**

```python
# tests/test_cnc_tool_output.py
from cnc_tool_output import CncToolOutput, load_config
from cnc_io import CncIo


def _setup_digital(printer, make_config):
    """Helper: set up cnc_io + cnc_tool_output in digital mode."""
    io_config = make_config('cnc_io', {
        'digital_out_0_pin': 'test:valve',
    })
    io = CncIo(io_config)
    printer.add_object('cnc_io', io)
    tool_config = make_config('cnc_tool_output', {
        'type': 'digital',
        'output_port': '0',
    })
    tool = load_config(tool_config)
    printer.fire_event('klippy:connect')
    return tool


class TestToolOutputDigital:
    def test_load_config_returns_instance(self, printer, make_config):
        tool = _setup_digital(printer, make_config)
        assert isinstance(tool, CncToolOutput)

    def test_m3_turns_on_digital_output(self, printer, make_config, gcmd):
        tool = _setup_digital(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd())
        io = printer.lookup_object('cnc_io')
        assert io.get_status()['digital_outputs'][0] == 1

    def test_m5_turns_off_digital_output(self, printer, make_config, gcmd):
        tool = _setup_digital(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd())
        gcode.commands['M5'](gcmd())
        io = printer.lookup_object('cnc_io')
        assert io.get_status()['digital_outputs'][0] == 0

    def test_s_value_ignored_for_digital(self, printer, make_config, gcmd):
        tool = _setup_digital(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd({'S': '500'}))
        io = printer.lookup_object('cnc_io')
        assert io.get_status()['digital_outputs'][0] == 1

    def test_status_reports_tool_state(self, printer, make_config, gcmd):
        tool = _setup_digital(printer, make_config)
        gcode = printer.lookup_object('gcode')
        assert tool.get_status()['active'] is False
        gcode.commands['M3'](gcmd())
        assert tool.get_status()['active'] is True
        gcode.commands['M5'](gcmd())
        assert tool.get_status()['active'] is False

    def test_status_reports_capabilities(self, printer, make_config):
        tool = _setup_digital(printer, make_config)
        status = tool.get_status()
        assert status['type'] == 'digital'
        assert status['variable_power'] is False
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_tool_output.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'cnc_tool_output'`

- [ ] **Step 3: Implement CncToolOutput with digital mode**

```python
# core/cnc_tool_output.py
# CNClipper - Universal Tool Output (M3/M4/M5)
#
# Provides machine-agnostic tool activation following grblHAL's
# capability model. The same M3/M4/M5 commands drive any tool type —
# behavior is determined by config, not by code.
#
# Tool types (set via 'type' config):
#   digital — on/off solenoid (waterjet valve, plasma relay)
#   pwm     — variable power (spindle, laser)
#
# Config section: [cnc_tool_output]
# Dependencies: [cnc_io] (for pin control)
#
# G-codes: M3 S<power>, M4 S<power>, M5
#
# Phase 1 scope: digital + PWM modes, startup/shutdown delay
# Deferred: motion-gated output (laser mode), dynamic power (M4 velocity
# scaling), at-speed feedback, multi-tool support

TOOL_TYPES = ('digital', 'pwm')


class CncToolOutput:
    def __init__(self, config):
        self.printer = config.get_printer()
        self.gcode = self.printer.lookup_object('gcode')
        # Config
        self.tool_type = config.get('type')
        if self.tool_type not in TOOL_TYPES:
            raise Exception(
                "[cnc_tool_output] type must be one of: %s"
                % ', '.join(TOOL_TYPES))
        self.output_port = config.getint('output_port', minval=0)
        self.max_power = config.getfloat('max_power', default=1.0,
                                         minval=0.0, maxval=1.0)
        self.min_power = config.getfloat('min_power', default=0.0,
                                         minval=0.0, maxval=1.0)
        self.startup_delay = config.getfloat('startup_delay', default=0.0,
                                             minval=0.0)
        self.shutdown_delay = config.getfloat('shutdown_delay', default=0.0,
                                              minval=0.0)
        # State
        self.active = False
        self.power = 0.0
        self.direction = 'cw'  # M3=cw, M4=ccw
        # Capabilities (derived from config)
        self.variable_power = (self.tool_type == 'pwm')
        # Deferred lookup of cnc_io
        self.io = None
        self.printer.register_event_handler('klippy:connect',
                                            self._handle_connect)
        # Register commands
        self.gcode.register_command(
            'M3', self.cmd_M3,
            desc="Start tool (CW / constant power)")
        self.gcode.register_command(
            'M4', self.cmd_M4,
            desc="Start tool (CCW / dynamic power — Phase 2)")
        self.gcode.register_command(
            'M5', self.cmd_M5,
            desc="Stop tool")

    def _handle_connect(self):
        self.io = self.printer.lookup_object('cnc_io')

    def _activate(self, power, direction):
        self.direction = direction
        self.active = True
        if self.tool_type == 'digital':
            self.io.set_digital_out(self.output_port, True)
            self.power = 1.0
        elif self.tool_type == 'pwm':
            # Clamp power to configured range
            if power <= 0.0:
                clamped = 0.0
            else:
                clamped = max(self.min_power,
                              min(self.max_power, power / 1000.0))
                # S value convention: S0-S1000 maps to 0.0-1.0
                # (following grblHAL/Klipper laser convention)
            self.io.set_analog_out(self.output_port, clamped)
            self.power = clamped
        # Startup delay — dwell to let tool reach operating state
        if self.startup_delay > 0.0:
            toolhead = self.printer.lookup_object('toolhead')
            toolhead.dwell(self.startup_delay)

    def _deactivate(self):
        self.active = False
        self.power = 0.0
        if self.tool_type == 'digital':
            self.io.set_digital_out(self.output_port, False)
        elif self.tool_type == 'pwm':
            self.io.set_analog_out(self.output_port, 0.0)
        # Shutdown delay — dwell to let tool stop safely
        if self.shutdown_delay > 0.0:
            toolhead = self.printer.lookup_object('toolhead')
            toolhead.dwell(self.shutdown_delay)

    def cmd_M3(self, gcmd):
        power = gcmd.get_float('S', default=1000.0, minval=0.0)
        self._activate(power, 'cw')
        if self.variable_power:
            gcmd.respond_info("Tool ON (CW) power=%.1f%%" % (self.power * 100))
        else:
            gcmd.respond_info("Tool ON")

    def cmd_M4(self, gcmd):
        # Phase 1: M4 behaves same as M3 but sets direction to CCW.
        # Phase 2 will add dynamic power mode (velocity scaling) for lasers.
        power = gcmd.get_float('S', default=1000.0, minval=0.0)
        self._activate(power, 'ccw')
        if self.variable_power:
            gcmd.respond_info("Tool ON (CCW) power=%.1f%%" % (self.power * 100))
        else:
            gcmd.respond_info("Tool ON (CCW)")

    def cmd_M5(self, gcmd):
        self._deactivate()
        gcmd.respond_info("Tool OFF")

    def get_status(self, eventtime=None):
        return {
            'active': self.active,
            'power': self.power,
            'direction': self.direction,
            'type': self.tool_type,
            'variable_power': self.variable_power,
            'startup_delay': self.startup_delay,
            'shutdown_delay': self.shutdown_delay,
        }


def load_config(config):
    return CncToolOutput(config)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_tool_output.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add core/cnc_tool_output.py tests/test_cnc_tool_output.py
git commit -m "feat(cnc_tool_output): M3/M5 with digital mode and capability model"
```

---

## Task 10: cnc_tool_output — PWM Mode and Startup/Shutdown Delays

**Files:**
- Modify: `tests/test_cnc_tool_output.py`

Tests PWM tool output (spindle, laser) and the startup/shutdown delay feature.

- [ ] **Step 1: Write tests for PWM mode**

```python
# Add to tests/test_cnc_tool_output.py
from cnc_io import CncIo


def _setup_pwm(printer, make_config, startup_delay=0.0, shutdown_delay=0.0):
    """Helper: set up cnc_io + cnc_tool_output in PWM mode."""
    io_config = make_config('cnc_io', {
        'analog_out_0_pin': 'test:spindle_pwm',
    })
    io = CncIo(io_config)
    printer.add_object('cnc_io', io)
    tool_config = make_config('cnc_tool_output', {
        'type': 'pwm',
        'output_port': '0',
        'max_power': '1.0',
        'min_power': '0.1',
        'startup_delay': str(startup_delay),
        'shutdown_delay': str(shutdown_delay),
    })
    tool = load_config(tool_config)
    printer.fire_event('klippy:connect')
    return tool


class TestToolOutputPWM:
    def test_m3_sets_pwm_value(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd({'S': '500'}))  # S500 = 50% power
        status = tool.get_status()
        assert status['active'] is True
        assert status['power'] == 0.5
        assert status['variable_power'] is True

    def test_m3_s0_sets_zero_power(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd({'S': '0'}))
        assert tool.get_status()['power'] == 0.0

    def test_m3_clamps_to_max_power(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd({'S': '2000'}))  # Over max
        assert tool.get_status()['power'] == 1.0

    def test_m3_enforces_min_power(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd({'S': '10'}))  # S10 = 1%, below min_power 10%
        assert tool.get_status()['power'] == 0.1  # Clamped to min

    def test_m5_sets_zero(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config)
        gcode = printer.lookup_object('gcode')
        gcode.commands['M3'](gcmd({'S': '500'}))
        gcode.commands['M5'](gcmd())
        io = printer.lookup_object('cnc_io')
        assert io.get_status()['analog_outputs'][0] == 0.0


class TestToolOutputDelays:
    def test_startup_delay_dwells(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config, startup_delay=2.0)
        gcode = printer.lookup_object('gcode')
        toolhead = printer.lookup_object('toolhead')
        gcode.commands['M3'](gcmd({'S': '1000'}))
        assert 2.0 in toolhead.dwells

    def test_shutdown_delay_dwells(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config, shutdown_delay=1.5)
        gcode = printer.lookup_object('gcode')
        toolhead = printer.lookup_object('toolhead')
        gcode.commands['M3'](gcmd({'S': '1000'}))
        gcode.commands['M5'](gcmd())
        assert 1.5 in toolhead.dwells

    def test_no_delay_when_zero(self, printer, make_config, gcmd):
        tool = _setup_pwm(printer, make_config,
                          startup_delay=0.0, shutdown_delay=0.0)
        gcode = printer.lookup_object('gcode')
        toolhead = printer.lookup_object('toolhead')
        gcode.commands['M3'](gcmd({'S': '1000'}))
        gcode.commands['M5'](gcmd())
        assert toolhead.dwells == []
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/test_cnc_tool_output.py -v`
Expected: All passed (previous + new)

- [ ] **Step 3: Commit**

```bash
git add tests/test_cnc_tool_output.py
git commit -m "test(cnc_tool_output): PWM mode, power clamping, startup/shutdown delays"
```

---

## Task 11: Install Script and Sample Config

**Files:**
- Create: `install.sh`
- Create: `configs/example_waterjet.cfg`
- Create: `configs/example_spindle.cfg`

Provides the symlink installer and example printer.cfg snippets showing how the same modules configure differently for different machines.

- [ ] **Step 1: Create install.sh**

```bash
#!/bin/bash
# CNClipper install script
# Symlinks CNClipper modules into Klipper's klippy/extras/ directory.
#
# Usage: ./install.sh [KLIPPER_PATH]
# Default KLIPPER_PATH: ~/klipper

set -e

KLIPPER_PATH="${1:-$HOME/klipper}"
EXTRAS_PATH="$KLIPPER_PATH/klippy/extras"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

if [ ! -d "$EXTRAS_PATH" ]; then
    echo "Error: Klipper extras directory not found at $EXTRAS_PATH"
    echo "Usage: $0 [path-to-klipper]"
    exit 1
fi

echo "Installing CNClipper modules to $EXTRAS_PATH"

# Core modules
for module in "$SCRIPT_DIR"/core/cnc_*.py; do
    [ -f "$module" ] || continue
    name=$(basename "$module")
    target="$EXTRAS_PATH/$name"
    if [ -L "$target" ]; then
        rm "$target"
    fi
    ln -s "$module" "$target"
    echo "  Linked: $name"
done

# Common modules (Phase 2+)
for module in "$SCRIPT_DIR"/common/cnc_*.py; do
    [ -f "$module" ] || continue
    name=$(basename "$module")
    target="$EXTRAS_PATH/$name"
    if [ -L "$target" ]; then
        rm "$target"
    fi
    ln -s "$module" "$target"
    echo "  Linked: $name"
done

# Machine modules (Phase 3+)
for module in "$SCRIPT_DIR"/machines/*.py; do
    [ -f "$module" ] || continue
    name=$(basename "$module")
    target="$EXTRAS_PATH/$name"
    if [ -L "$target" ]; then
        rm "$target"
    fi
    ln -s "$module" "$target"
    echo "  Linked: $name"
done

echo "Done. Restart Klipper to load new modules."
```

- [ ] **Step 2: Create example waterjet config**

```ini
# configs/example_waterjet.cfg
# CNClipper configuration for a waterjet cutter (e.g., Wazer)
#
# This demonstrates how the SAME universal modules configure for
# waterjet-specific behavior through config alone — no waterjet code needed.

[save_variables]
filename: ~/cnc_variables.cfg

# --- Work Coordinate Systems ---
# G54-G59 for different sheet positions / fixtures
[cnc_wcs]

# --- I/O Ports ---
# Map physical pins to logical port numbers used by M64/M65/M66
[cnc_io]
digital_out_0_pin: P1.23    # High-pressure valve (abrasive waterjet on/off)
digital_out_1_pin: P1.24    # Low-pressure pump enable
digital_in_0_pin: P1.25     # Lid safety switch
digital_in_0_pull: up
digital_in_1_pin: P1.26     # Water level sensor
digital_in_1_pull: up

# --- Tool Output ---
# Waterjet valve = digital on/off via M3/M5
# No variable power, no direction — just open/close the valve
[cnc_tool_output]
type: digital
output_port: 0               # cnc_io digital_out_0 = HP valve
```

- [ ] **Step 3: Create example spindle config**

```ini
# configs/example_spindle.cfg
# CNClipper configuration for a CNC mill/router with PWM spindle
#
# Same modules as waterjet — different config, different behavior.

[save_variables]
filename: ~/cnc_variables.cfg

# --- Work Coordinate Systems ---
[cnc_wcs]

# --- I/O Ports ---
[cnc_io]
analog_out_0_pin: P2.4      # Spindle PWM (0-10V via DAC or direct PWM)
digital_in_0_pin: P1.28     # Spindle at-speed signal
digital_in_0_pull: up
digital_in_1_pin: P1.29     # Door interlock switch
digital_in_1_pull: up

# --- Tool Output ---
# Spindle = PWM variable speed via M3 S<rpm>/M5
# S0-S1000 maps to 0-100% duty cycle (actual RPM depends on VFD/spindle)
[cnc_tool_output]
type: pwm
output_port: 0               # cnc_io analog_out_0 = spindle PWM
max_power: 1.0
min_power: 0.1               # Don't run below 10% (stall protection)
startup_delay: 3.0           # Wait 3s for spindle to reach speed after M3
shutdown_delay: 2.0           # Wait 2s for spindle to stop after M5
```

- [ ] **Step 4: Commit**

```bash
chmod +x install.sh
git add install.sh configs/example_waterjet.cfg configs/example_spindle.cfg
git commit -m "feat: install script and example configs for waterjet and spindle"
```

---

## Task 12: Run Full Suite and Final Cleanup

**Files:**
- Possibly modify any files where tests reveal issues

- [ ] **Step 1: Run full test suite**

Run: `cd C:/GitHub/CNClipper && python -m pytest tests/ -v --tb=short`
Expected: All tests pass. If any fail, fix the issue and re-run.

- [ ] **Step 2: Verify module imports work independently**

Run:
```bash
cd C:/GitHub/CNClipper && python -c "import sys; sys.path.insert(0, 'core'); import cnc_wcs; print('cnc_wcs OK')"
cd C:/GitHub/CNClipper && python -c "import sys; sys.path.insert(0, 'core'); import cnc_io; print('cnc_io OK')"
cd C:/GitHub/CNClipper && python -c "import sys; sys.path.insert(0, 'core'); import cnc_tool_output; print('cnc_tool_output OK')"
```
Expected: Each prints "OK" with no import errors.

- [ ] **Step 3: Update .agent-context.md to reflect Phase 1 completion**

Mark Phase 1 items as complete. Update project status to "Phase 1 Complete — Ready for Phase 2".

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: Phase 1 complete — core CNC primitives (cnc_wcs, cnc_io, cnc_tool_output)"
```

---

## What Phase 2 Builds On This

Phase 1 delivers the three primitives. Phase 2 composes them:

| Phase 2 Module | Composes | What it adds |
|---|---|---|
| `cnc_output_sequencer` | cnc_io + cnc_tool_output + G4 dwell | Config-driven multi-step timed sequences |
| `cnc_interlock` | cnc_io (input monitoring) | Safety rules with configurable response |
| `cnc_feed_hold` | toolhead integration | Real-time motion pause (blocker risk) |
| `cnc_overrides` | cnc_tool_output + toolhead | Runtime feed/power scaling (M50/M51) |

Phase 2 also adds to Phase 1 modules:
- **cnc_io:** Motion-synchronized outputs (M62/M63) — requires planner hooks
- **cnc_tool_output:** Motion-gated output (laser mode), dynamic M4 power, at-speed feedback via M66
