# BrachioGraph Project Assessment

## 1. Quality of Plotted Output

### Calibration Framework

BrachioGraph provides a three-tier calibration system that directly impacts output quality:

1. **Naive mode** — a simple linear mapping from servo angle to pulse width using a
   fixed degrees-per-millisecond ratio. This is adequate for a first test but introduces
   visible distortion because real servos are not perfectly linear.
2. **Polynomial (table-based) mode** — the user supplies measured `[angle, pulse_width]`
   pairs and a cubic polynomial is fitted with `numpy.polyfit`. This compensates for
   non-linear servo response and significantly improves accuracy.
3. **Bidirectional mode** — separate clockwise and anti-clockwise measurements are
   recorded. The library automatically derives a mean curve and a hysteresis correction
   factor, which is applied dynamically based on the direction of travel. This eliminates
   backlash artefacts that are common in cheap hobby servos.

The progressive nature of this system is a real strength: users can start plotting
immediately with naive calibration and incrementally improve quality without changing
any drawing code.

### Pen Control

The `Pen` class offers an easing function (`ease_pen`) that transitions the pen servo in
1 ms increments between its up and down positions. This prevents the abrupt pen drops
that cause ink blots or indentations on the drawing surface. However, the easing code in
`pen_up()` and `pen_down()` is currently commented out, so users only benefit from easing
if they call the method directly or re-enable those lines.

### Line Processing Pipeline

Image-to-line conversion is handled by `linedraw.py`, which implements:

- Contrast enhancement via `PIL.ImageOps.autocontrast`.
- Edge detection using OpenCV Canny when available, falling back to a Sobel-based
  approximation.
- Configurable hatching with multiple angles and adjustable line spacing.
- A line-sorting algorithm (`sortlines`) that reorders segments to minimise pen-up travel
  and reduce total plotting time.
- A `join_lines` function that merges nearby endpoints into continuous strokes.

These steps produce clean vector output from raster images. The main limitation is that
`sortlines` uses an O(n²) nearest-neighbour search, which becomes slow for complex images
with thousands of line segments.

### Drawing Resolution

Two parameters govern output fidelity:

| Parameter | Default | Effect |
|-----------|---------|--------|
| `resolution` | 0.1 cm | Minimum linear step when interpolating straight lines |
| `angular_step` | 0.1° | Minimum servo increment per step |

Lower values produce smoother curves at the cost of slower plotting. The defaults
are a reasonable compromise for the typical 8 cm arm length.

### Summary

The output quality mechanisms are well thought out and the calibration framework is one
of the project's strongest features. The main areas for improvement are:

- Re-enable pen easing by default so all users benefit from smoother pen transitions.
- Optimise the line-sorting algorithm for large line sets (e.g. using a k-d tree).
- Document recommended calibration workflows more prominently in the code itself.

---

## 2. Maintainability of the Codebase

### Architecture

The class hierarchy is clean and easy to follow:

```
Plotter          (base class — movement, calibration, pen control)
├── BrachioGraph (shoulder-elbow kinematics)
└── PantoGraph   (parallel pantograph kinematics)

Pen              (pen servo abstraction)

BaseTurtle       (turtle-graphics visualisation)
├── BrachioGraphTurtle
└── PantoGraphTurtle
```

Adding a new plotter geometry requires subclassing `Plotter` and overriding only
`xy_to_angles()` and `angles_to_xy()`. The template-method pattern used for the
movement pipeline keeps the subclasses small (BrachioGraph is ~228 lines,
PantoGraph ~206 lines).

### Code Style

- Consistent use of section comment markers (e.g. `# ----- trigonometric methods -----`)
  aids navigation.
- `pyproject.toml` configures Black with a 100-character line length, and `.travis.yml`
  runs flake8, so there is basic style enforcement.
- Blank-line ratio is ~19–25 %, which is healthy for readability.

### Docstrings and Type Hints

`plotter.py` is well documented with 48 docstrings across 980 lines. `brachiograph.py`
and `pantograph.py` document their public methods adequately. However, `linedraw.py` has
**zero** docstrings across 18 functions and 521 lines, making it the hardest module to
maintain.

There are no type hints anywhere in the codebase. Adding them — even just to public
method signatures — would improve IDE support and make refactoring safer.

### Error Handling

Error handling is sparse:

- Only ~8 try/except blocks in the 980-line `plotter.py`.
- `linedraw.py` contains a bare `except:` clause that silently swallows all exceptions
  during the OpenCV import fallback.
- Out-of-reach coordinates raise a generic `Exception` with no descriptive message.
- There is no validation of constructor arguments (e.g. negative arm lengths, empty
  calibration tables, pulse widths outside the 500–2500 µs safe range).

### Dead and Commented-Out Code

- The `drive_xy()` method in `plotter.py` (lines ~793–820) is annotated as removed but
  the implementation is still present.
- Pen easing calls are commented out in `pen_up()` and `pen_down()`.
- `test_pantograph.py` contains ~40 lines of commented-out test cases.

This kind of code adds noise and should either be restored with clear intent or removed.

### Test Suite

The project has 52 tests across four files:

| Module | Tests | Key Coverage |
|--------|-------|-------------|
| `test_plotter.py` | 18 | Calibration modes, basic movement |
| `test_brachiograph.py` | 17 | Kinematics, patterns, `plot_file` |
| `test_pantograph.py` | 12 | Kinematics (parameterised) |
| `test_turtle.py` | 5 | Smoke tests for visualisation |

Virtual mode allows all tests to run without hardware, which is excellent for CI.
Coverage gaps include:

- `linedraw.py` — completely untested.
- `Pen` class — no unit tests.
- Edge cases — no tests for boundary coordinates, invalid inputs, or large data sets.
- Several commented-out tests reduce effective coverage.

### Dependency Management

`requirements.txt` pins all eight runtime dependencies to specific versions, which is
good for reproducibility. The list is concise and each dependency serves a clear purpose.
OpenCV is handled as an optional dependency with a graceful fallback — a nice pattern.

### Summary

The codebase is well structured and relatively easy to navigate, but maintainability
would benefit from:

- Adding type hints to public APIs.
- Documenting `linedraw.py` with docstrings.
- Replacing bare except clauses with specific exception types.
- Removing dead and commented-out code.
- Expanding the test suite to cover `linedraw.py` and `Pen`.

---

## 3. Simplicity of Setup and Integration with Hardware

### Software Setup

Installation is straightforward for anyone familiar with Python:

```bash
pip install -r requirements.txt
```

The pinned `requirements.txt` avoids version-compatibility surprises. The only
non-trivial prerequisite is the **pigpio daemon**, which must be running on the
Raspberry Pi before the library can control servos:

```bash
sudo pigpiod
```

This step is documented in the tutorial but is not mentioned in `requirements.txt` or
`README.rst`, so new users may miss it.

### Hardware Requirements

The project targets the **Raspberry Pi** exclusively (pigpio is RPi-only). The minimum
hardware list is:

- Raspberry Pi (any model with GPIO header)
- 2× SG90 or similar hobby servos (arm joints)
- 1× SG90 servo (pen lifter)
- Jumper wires, mounting materials, a pen

GPIO pins are hardcoded to sensible defaults (14, 15, 18) but can be overridden in the
constructor.

### Virtual Mode

Setting `virtual=True` in the constructor replaces all pigpio calls with no-ops and
enables turtle-graphics visualisation. This is a significant strength:

- Developers can work on the codebase from any machine, not just a Raspberry Pi.
- CI tests run without hardware.
- Users can preview drawings before committing to paper.

### Configuration

Example configurations live in `bg.py` and `bgt.py`. A typical setup looks like:

```python
from brachiograph import BrachioGraph

bg = BrachioGraph(
    inner_arm=8, outer_arm=8,
    servo_1_parked_pw=1500, servo_2_parked_pw=1500,
    pw_up=1400, pw_down=1100,
)
```

All parameters have sensible defaults, so a bare `BrachioGraph()` call works for initial
experimentation.

### Interactive Calibration

The `capture_pws()` method provides a keyboard-driven calibration workflow that records
angle-to-pulse-width mappings. It outputs formatted Python dictionaries that can be
pasted directly into configuration code. This is well designed and avoids the need for
external calibration tools.

### Documentation

The Sphinx documentation under `docs/` is comprehensive, with ~20 RST files covering
construction, wiring, software installation, and usage. Over 40 images (geometry
diagrams, photographs, annotated screenshots) support the written guides. The main gap is
that some operational details — such as recommended servo warm-up procedures and safe
speed limits — are not covered.

### Areas for Improvement

- Add a `setup.py` or populate `pyproject.toml` with package metadata so the project can
  be installed with `pip install .` or published to PyPI.
- Document the pigpio daemon requirement in `README.rst`.
- Provide a single-command setup script (e.g. `make install` or a shell script) that
  installs dependencies and starts `pigpiod`.
- Consider supporting alternative GPIO libraries (e.g. `gpiozero`) for broader
  hardware compatibility.

---

## Overall Assessment

| Area | Rating | Notes |
|------|--------|-------|
| **Output quality** | Strong | Excellent calibration framework; minor gaps in pen easing and line-sort performance |
| **Maintainability** | Good | Clean architecture, but needs type hints, docstrings in `linedraw.py`, and broader test coverage |
| **Setup & hardware** | Good | Easy for RPi users; virtual mode is a standout feature; packaging and daemon setup could be smoother |

BrachioGraph is a well-designed project that makes complex robotics concepts accessible
through a clean API. Its progressive calibration system and virtual mode are particular
highlights. The main investment areas for future development are test coverage,
documentation of internal modules, and packaging for easier distribution.
