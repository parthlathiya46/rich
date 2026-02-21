# CLAUDE.md — Rich Library Codebase Guide

## Project Overview

Rich is a Python library (v14.3.3) for rendering rich text, tables, progress bars, syntax highlighting, markdown, and more to the terminal. It is created by Will McGugan / Textualize and has ~50k GitHub stars.

- **License**: MIT
- **Python**: >=3.8
- **Dependencies**: `pygments` (syntax highlighting), `markdown-it-py` (markdown parsing)
- **Optional**: `ipywidgets` (Jupyter support)
- **Package manager**: Poetry
- **Type checking**: mypy (strict mode)
- **Formatter**: black
- **Test framework**: pytest with pytest-cov

## Quick Commands

```bash
# Install dependencies
poetry install

# Run tests with coverage
make test
# or: TERM=unknown pytest --cov-report term-missing --cov=rich tests/ -vv

# Run tests without coverage
make test-no-cov

# Type checking
make typecheck
# or: mypy -p rich --no-incremental

# Format code
make format          # apply formatting
make format-check    # check only

# Build docs
make docs
```

## Repository Layout

```
rich/                     # Source package (~26.5k lines across ~60 modules)
├── __init__.py           # Public API: get_console(), print(), inspect(), print_json()
├── console.py            # ★ CENTRAL MODULE (2685 lines) - Console class, rendering pipeline
├── text.py               # ★ Text class (1363 lines) - styled text with spans
├── progress.py           # ★ Progress bars (1717 lines) - tasks, columns, live display
├── table.py              # ★ Table rendering (1015 lines) - rows, columns, borders
├── pretty.py             # Pretty-printing of Python objects (1016 lines)
├── syntax.py             # Syntax highlighting via Pygments (985 lines)
├── traceback.py          # Enhanced tracebacks (924 lines)
├── style.py              # Style system (793 lines) - colors, attributes, composition
├── segment.py            # Segment NamedTuple (783 lines) - atomic render unit
├── markdown.py           # Markdown rendering (793 lines)
├── color.py              # Color system (621 lines) - 4/8/24-bit color support
├── layout.py             # Layout system (442 lines) - split-based terminal layouts
├── live.py               # Live updating display (404 lines)
├── prompt.py             # User input prompts (400 lines)
├── cells.py              # Unicode cell width calculation (352 lines)
├── panel.py              # Border panels around content (317 lines)
├── logging.py            # Rich logging handler (297 lines)
├── align.py              # Text alignment (320 lines)
├── tree.py               # Tree rendering (257 lines)
├── markup.py             # [bold red]Console markup[/] parser (251 lines)
├── highlighter.py        # Regex-based text highlighting (232 lines)
├── columns.py            # Column layout (187 lines)
├── json.py               # JSON pretty-printing (139 lines)
├── rule.py               # Horizontal rule (130 lines)
├── spinner.py            # Spinner animations (132 lines)
├── bar.py                # Simple bar renderable (93 lines)
├── box.py                # Box drawing character sets (474 lines)
├── measure.py            # Width measurement system (151 lines)
├── protocol.py           # Renderable protocol (rich_cast, is_renderable)
├── abc.py                # RichRenderable abstract base class
├── _inspect.py           # Object inspection
├── _log_render.py        # Log entry rendering
├── _ratio.py             # Ratio/proportion calculations for layouts
├── _wrap.py              # Text wrapping
├── _emoji_codes.py       # Emoji code mappings (auto-generated, 3610 lines)
├── _spinners.py          # Spinner definitions (auto-generated, 482 lines)
├── _palettes.py          # Color palette data (auto-generated, 309 lines)
├── _win32_console.py     # Windows console API (661 lines)
├── _unicode_data/        # Unicode width tables
└── ...

tests/                    # Test suite (~11.2k lines across ~50 files)
├── conftest.py           # Fixture: resets NO_COLOR/FORCE_COLOR envvars
├── render.py             # Test helper: render() function for snapshot testing
├── test_console.py       # Largest test file (1129 lines)
├── test_text.py          # 1129 lines
├── test_pretty.py        # 759 lines
├── test_progress.py      # 690 lines
├── ...
└── test_json.py          # Thinnest test file (8 lines!) - major gap

examples/                 # Usage examples (~30 files)
benchmarks/               # Performance benchmarks (ASV-based)
tools/                    # Code generation scripts
docs/                     # Sphinx documentation
```

## Architecture & Core Concepts

### The Rendering Pipeline

Everything in Rich flows through this pipeline:

```
User calls console.print(obj)
        │
        ▼
   rich_cast(obj)           # If obj has __rich__(), call it to get a renderable
        │
        ▼
   obj.__rich_console__()   # Yields Segments or nested renderables
        │
        ▼
   Console.render()         # Recursively resolves renderables → flat Segment stream
        │
        ▼
   Segment.split_and_crop_lines()  # Splits into lines, handles wrapping
        │
        ▼
   Console._render_buffer() # Converts Segments to ANSI escape strings
        │
        ▼
   file.write()             # Output to terminal/file
```

### The Three Protocols

Any object can be rendered by Rich by implementing one or more of these:

1. **`__rich_console__(self, console, options) -> Iterable[Segment | RenderableType]`**
   The primary rendering protocol. Yields `Segment` instances or other renderables.

2. **`__rich_measure__(self, console, options) -> Measurement`**
   Returns min/max width needed. Used by containers (Table, Columns) for layout.

3. **`__rich__(self) -> RenderableType`**
   Convenience protocol — returns another renderable. Called first by `rich_cast()`.

### Key Types

- **`Segment(text, style, control)`** — A NamedTuple. The atomic unit of rendered output. Every renderable ultimately produces Segments.
- **`Style`** — Immutable style object (color, bgcolor, bold, italic, etc.). Supports composition via `+` operator. Has tri-state attributes (True/False/None).
- **`Text`** — Rich text with a list of `Span(start, end, style)` annotations over a plain string. The fundamental text type.
- **`Measurement(minimum, maximum)`** — NamedTuple for width measurement.
- **`ConsoleOptions`** — Dataclass passed to `__rich_console__()` with terminal size, encoding, wrapping settings, etc.

### Module Dependency Hierarchy (simplified)

```
console.py (imports almost everything)
    ├── text.py ← markup.py, highlighter.py
    ├── style.py ← color.py ← color_triplet.py
    ├── segment.py ← cells.py
    ├── measure.py ← protocol.py
    ├── table.py, panel.py, tree.py, progress.py  (renderables)
    └── live.py ← live_render.py, control.py
```

`console.py` is the hub — it imports most other modules. Individual renderables (panel, table, tree, etc.) are relatively independent of each other but all depend on `console`, `segment`, `text`, `style`, and `measure`.

### Console Class (console.py)

The `Console` class is the central API (2685 lines). Key responsibilities:

- **Rendering**: `render()`, `render_lines()`, `render_str()`
- **Output**: `print()`, `log()`, `print_json()`
- **Capture**: `capture()`, `begin_capture()`, `end_capture()`
- **Export**: `export_text()`, `export_html()`, `export_svg()`
- **Terminal**: `size`, `is_terminal`, `color_system`
- **Thread safety**: Uses `threading.RLock` for all output

### Style System (style.py)

- Styles are **immutable** and cached via `lru_cache`
- Parsed from strings: `Style.parse("bold red on white")`
- Composed via `+`: `Style(bold=True) + Style(color="red")`
- Uses bit manipulation (`_Bit` descriptor) for attribute storage
- `StyleType = Union[str, Style]` — styles can be passed as strings throughout

### Text System (text.py)

- `Text` holds a plain `str` plus a list of `Span` objects mapping style ranges
- Supports slicing, concatenation, wrapping, highlighting, alignment
- The `Span` NamedTuple: `Span(start, end, style)` — character index ranges
- `Text.__rich_console__()` converts to `Segment` stream with proper wrapping

### Markup System (markup.py)

Parses `[bold red]Hello[/bold red]` syntax into styled `Text`:
- Tag regex: `r"((\\*)\[([a-z#/@][^[]*?)])"` 
- Supports nesting, closing tags, auto-close with `[/]`
- `render()` function returns `Text` with spans applied
- `escape()` function escapes brackets in user content

## Testing Patterns

### Standard Test Pattern

Most tests follow this pattern — create a Console writing to StringIO, render content, assert on output:

```python
import io
from rich.console import Console

def test_something():
    console = Console(file=io.StringIO(), width=80, legacy_windows=False)
    console.print(SomeRenderable(...))
    result = console.file.getvalue()
    assert result == "expected output\n"
```

### Render Helper

Tests use `tests/render.py` which provides:
```python
def render(renderable, no_wrap=False) -> str:
    # Creates Console(width=100, color_system="truecolor", legacy_windows=False)
    # Returns output with link IDs replaced for reproducibility
```

### Snapshot-Style Tests

Many tests use parametrized expected output strings:
```python
tests = [Panel("Hello"), Panel("Hello", expand=False)]
expected = ["╭──...──╮\n│Hello...│\n╰──...──╯\n", ...]

@pytest.mark.parametrize("panel,expected", zip(tests, expected))
def test_render_panel(panel, expected):
    assert render(panel) == expected
```

### Measurement Tests

```python
def test_measure():
    console = Console(file=io.StringIO(), width=80)
    min_w, max_w = renderable.__rich_measure__(console, console.options)
    assert min_w == expected_min
    assert max_w == expected_max
```

### conftest.py

Auto-use fixture removes `FORCE_COLOR` and `NO_COLOR` env vars to ensure consistent test output.

### Test Coverage Gaps (by test:source line ratio)

These modules have notably thin test coverage:

| Module | Source Lines | Test Lines | Ratio | Notes |
|--------|-------------|------------|-------|-------|
| `json.py` | 139 | 8 | 5.7% | Only 1 test! Missing: constructor, error handling, CLI |
| `palette.py` | 100 | 8 | 8.0% | Nearly untested |
| `screen.py` | 54 | 12 | 22.2% | Minimal tests |
| `layout.py` | 442 | 100 | 22.6% | Complex module, thin tests |
| `filesize.py` | 88 | 20 | 22.7% | Basic utility, under-tested |
| `measure.py` | 151 | 36 | 23.8% | Core module, needs more coverage |
| `status.py` | 131 | 34 | 25.9% | Live status display |
| `markdown.py` | 793 | 206 | 25.9% | Large module with many element types |
| `control.py` | 219 | 62 | 28.3% | Terminal control codes |
| `color.py` | 621 | 187 | 30.1% | Complex color system |
| `prompt.py` | 400 | 126 | 31.5% | User input handling |
| `jupyter.py` | 101 | 32 | 31.6% | Jupyter integration |
| `style.py` | 792 | 267 | 33.7% | Core style system |
| `containers.py` | 167 | 57 | 34.1% | Renderables, Group, Lines |

## Code Conventions

### Type Annotations
- All code is fully typed with mypy strict mode
- `TYPE_CHECKING` blocks for circular import prevention
- Common patterns: `StyleType = Union[str, Style]`, `TextType = Union[str, Text]`

### Naming
- Private modules prefixed with `_` (e.g., `_ratio.py`, `_wrap.py`, `_loop.py`)
- No abbreviations in public APIs
- `RenderableType`, `RenderResult`, `JustifyMethod` — type aliases use CamelCase

### Docstrings
- Google-style docstrings with Args/Returns sections
- All public classes and methods are documented

### Module Structure
- Imports at top, TYPE_CHECKING block for forward refs
- NamedTuples and dataclasses for data containers
- `JupyterMixin` mixed into renderables for notebook support
- `if __name__ == "__main__":` blocks for standalone demos

### Error Handling
- Custom exceptions in `rich/errors.py` (`NotRenderableError`, `MarkupError`, etc.)
- `# pragma: no cover` on unreachable/demo code

### Performance
- `lru_cache` on hot paths (Style.parse, cell width calculation)
- `__slots__` not widely used (except `_Bit` descriptor)
- Segment is a NamedTuple for memory efficiency
- `_is_single_cell_widths()` fast-path optimization in cells.py

## Creating a New Renderable

Follow this pattern to create a new renderable:

```python
from rich.console import Console, ConsoleOptions, RenderableType, RenderResult
from rich.jupyter import JupyterMixin
from rich.measure import Measurement
from rich.segment import Segment
from rich.style import StyleType

class MyRenderable(JupyterMixin):
    """A custom renderable."""

    def __init__(self, content: RenderableType, style: StyleType = "none") -> None:
        self.content = content
        self.style = style

    def __rich_console__(
        self, console: Console, options: ConsoleOptions
    ) -> RenderResult:
        # Yield Segments or other renderables
        yield Segment("Hello ", console.get_style(self.style))
        yield self.content  # Nested renderable — Console.render() handles recursion

    def __rich_measure__(
        self, console: Console, options: ConsoleOptions
    ) -> Measurement:
        # Return min/max width needed
        from rich.measure import measure_renderables
        content_measure = measure_renderables(console, options, [self.content])
        return Measurement(
            content_measure.minimum + 6,  # "Hello " prefix
            content_measure.maximum + 6,
        )
```

## Writing Tests for Rich

```python
import io
import pytest
from rich.console import Console

# Standard test fixture
def make_console(**kwargs) -> Console:
    return Console(
        file=io.StringIO(),
        width=80,
        legacy_windows=False,
        color_system="truecolor",
        **kwargs
    )

# Test rendering output
def test_my_renderable():
    console = make_console()
    console.print(MyRenderable("World"))
    output = console.file.getvalue()
    assert "Hello World" in output

# Test measurement
def test_my_renderable_measure():
    console = make_console()
    r = MyRenderable("Hi")
    measurement = r.__rich_measure__(console, console.options)
    assert measurement.minimum >= 8
    assert measurement.maximum >= 8

# Test with different widths
@pytest.mark.parametrize("width", [10, 40, 80, 120])
def test_my_renderable_widths(width):
    console = make_console(width=width)
    console.print(MyRenderable("Content"))
    output = console.file.getvalue()
    for line in output.splitlines():
        assert len(line) <= width
```

## AI Contribution Policy

This project accepts AI-generated PRs under these conditions:
1. Must fill in the PR template
2. Must identify itself as AI-generated with agent name
3. Must link to an issue/discussion approved by @willmcgugan

## Key Files to Read First

When working on Rich, start with these files in order:
1. `rich/protocol.py` — Understand the rendering protocol (42 lines)
2. `rich/segment.py` — Understand Segments (the atomic unit)
3. `rich/measure.py` — Understand width measurement
4. `rich/style.py` — Understand the style system
5. `rich/text.py` — Understand styled text
6. `rich/console.py` lines 263-277 — The protocol types
7. `rich/console.py` lines 1300-1350 — The `render()` method
8. `rich/panel.py` — A clean example of a renderable implementation