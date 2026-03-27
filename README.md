# pyside-frameless

Frameless window toolkit for PySide6. Native Aero Snap on Windows, cross-platform resize fallback, animated drag-and-drop overlay.

![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue)
![PySide6](https://img.shields.io/badge/PySide6-6.5%2B-green)
![License: MIT](https://img.shields.io/badge/license-MIT-brightgreen)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey)

---

## Why

Qt's built-in `FramelessWindowHint` gives you a rectangle with no title bar — but also no resize, no snap, no drag, no visual feedback on drops. You end up reimplementing the same WM_NCHITTEST / mouse-tracking / DPI boilerplate every time.

This library does it once. Subclass `FramelessWindow`, point it at your title bar widget, done.

## Features

- **Windows Aero Snap** — native `WM_NCHITTEST` / `WM_NCCALCSIZE` handling, snap-to-edge, corner resize, Win+Arrow shortcuts. Not emulated — the OS handles it.
- **Cross-platform resize** — mouse-event fallback for macOS / Linux with correct cursors and minimum-size enforcement.
- **DPI-aware** — edge detection and title bar hit-testing scale with `devicePixelRatio`.
- **Title bar drag & double-click maximize** — register any `QWidget` as the drag region.
- **Animated drop overlay** — dashed-border overlay with fade animation, valid/invalid states, configurable colors and icons.
- **Drop zone widget** — file-type validation, directory support, auto-discovery of target files.
- **Zero dependencies** beyond PySide6. No application-specific code, no styling opinions.

## Install

```bash
git submodule add https://github.com/you/pyside-frameless.git pyside_frameless
```
or clone and use as module directly.

## Quick start

### Frameless window

```python
from PySide6.QtWidgets import QApplication, QWidget, QHBoxLayout, QLabel, QPushButton
from pyside_frameless import FramelessWindow


class MyWindow(FramelessWindow):
    def __init__(self):
        super().__init__()
        self.setMinimumSize(800, 500)

        # Build a title bar — any QWidget works
        title_bar = QWidget()
        title_bar.setFixedHeight(40)
        title_bar.setStyleSheet("background: #18181b;")

        layout = QHBoxLayout(title_bar)
        layout.setContentsMargins(16, 0, 16, 0)
        layout.addWidget(QLabel("My App"))
        layout.addStretch()

        close_btn = QPushButton("✕")
        close_btn.clicked.connect(self.close)
        layout.addWidget(close_btn)

        # Register it — this widget becomes the drag region
        self.set_title_bar_widget(title_bar)

        # ... set up your central widget, etc.

    def on_maximize_changed(self, is_maximized: bool):
        # Update your maximize/restore button icon here
        pass


app = QApplication([])
w = MyWindow()
w.show()
app.exec()
```

### Drop zone with overlay

```python
from PySide6.QtWidgets import QVBoxLayout, QLabel
from PySide6.QtGui import QPixmap
from pyside_frameless import DropZoneWidget


class FilePanel(DropZoneWidget):
    def __init__(self):
        super().__init__(
            valid_extensions=['.uproject', '.uplugin'],
            allow_directories=True,
        )
        layout = QVBoxLayout(self)
        layout.addWidget(QLabel("Drop a project here"))

        # Create overlay after layout is ready
        overlay = self.setup_drop_overlay()
        overlay.configure(
            invalid_text="Not a valid project file",
            # valid_pixmap=QPixmap(...),   # optional icon
            # invalid_pixmap=QPixmap(...),
        )

        # Called on successful drop
        self.set_drop_callback(self._on_file)

    def _on_file(self, path: str):
        print(f"Dropped: {path}")
```

## API

### `FramelessWindow(QMainWindow)`

A `QMainWindow` subclass with the frame removed and window management reimplemented.

| Method | Description |
|---|---|
| `set_title_bar_widget(widget)` | Designate a `QWidget` as the drag region. Buttons inside it remain clickable — the hit-test walks up the widget tree and yields to `QPushButton` instances. |
| `toggle_maximize()` | Toggle between maximized and normal state. Uses native `ShowWindow` on Windows, `showMaximized`/`showNormal` elsewhere. Remembers pre-maximize geometry. |
| `on_maximize_changed(is_maximized)` | Override to react to maximize state changes (e.g. swap the maximize/restore icon). Fires on Aero Snap, double-click, and `toggle_maximize()`. |

| Class attribute | Default | Description |
|---|---|---|
| `RESIZE_MARGIN` | `8` | Edge detection zone in logical pixels. |

**Windows behavior**: On `showEvent`, the window style gets `WS_THICKFRAME | WS_MINIMIZEBOX | WS_MAXIMIZEBOX | WS_SYSMENU` so the OS handles snap gestures, Win+Arrow, and taskbar interactions natively. `WM_NCCALCSIZE` returns 0 to remove the default frame. `WM_NCHITTEST` maps mouse positions to `HT*` constants with DPI-scaled borders and title bar region.

**macOS / Linux behavior**: Mouse press/move/release events on edges perform manual resize with `QRect` geometry updates. The title bar double-click handler calls `toggle_maximize()`.

### `DropOverlay(QWidget)`

Animated translucent overlay with dashed border. Typically used through `DropZoneWidget`, not directly.

| Method | Description |
|---|---|
| `configure(*, ...)` | Set any combination of: `valid_bg`, `valid_border`, `invalid_bg`, `invalid_border` (`QColor`), `valid_pixmap`, `invalid_pixmap` (`QPixmap`), `invalid_text` (`str`), `font_family` (`str`). |
| `show_overlay(valid=True)` | Fade in. Pass `valid=False` to show the rejection state. |
| `hide_overlay()` | Fade out and auto-hide when animation completes. |

Animations use `QPropertyAnimation` on a custom `opacity` property with `OutCubic` easing, 150ms duration. Show/hide calls are deferred to the GUI thread via `QTimer.singleShot(0, ...)` for safe use from drag events.

### `DropZoneWidget(QWidget)`

Base widget that handles drag-enter, drag-leave, and drop events with file-type validation.

| Constructor arg | Type | Description |
|---|---|---|
| `valid_extensions` | `list[str]` | Accepted file extensions, e.g. `['.png', '.jpg']`. Empty = accept all. |
| `allow_directories` | `bool` | If `True`, a dropped directory is accepted when it contains a file matching `valid_extensions`. |

| Method / Signal | Description |
|---|---|
| `setup_drop_overlay() → DropOverlay` | Create and attach the animated overlay. Call after your layout is built. |
| `set_drop_callback(fn)` | Register a `Callable[[str], None]` called on successful drop. |
| `file_dropped` | `Signal(str)` — emitted with the resolved file path on valid drop. |

When a directory is dropped with `allow_directories=True`, the widget searches for the first file matching any extension in `valid_extensions` and emits that path.

## How the hit-testing works

The core challenge with frameless windows is telling the OS which part of the window is the title bar (draggable), which is a resize edge, and which is normal content. On Windows this is solved by handling `WM_NCHITTEST`:

```
Mouse position → DPI scaling → edge/corner check → title bar check → HTCLIENT

                 ┌─ within 8px of edge? → HTLEFT / HTTOP / HTBOTTOMRIGHT / ...
global coords →  ├─ within title bar height? → HTCAPTION (drag region)
                 ├─ over a QPushButton? → HTCLIENT (clickable, not draggable)
                 └─ otherwise → HTCLIENT (normal content)
```

The button check walks the widget tree upward from `QApplication.widgetAt()` — any `QPushButton` ancestor wins over the title bar, so close/minimize/maximize buttons work without extra configuration.

## Requirements

- Python ≥ 3.10
- PySide6 ≥ 6.5

## License

MIT
