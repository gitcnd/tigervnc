# TigerVNC Source Code Layout for AI Agents

## Project Identity

- **Name**: TigerVNC
- **Version**: 1.16.80 (as of this source tree)
- **License**: GPL v2+
- **Build system**: CMake (minimum 3.10)
- **Compiler requirement**: MinGW or MinGW-w64 (MSVC explicitly rejected)
- **GUI toolkit**: FLTK 1.3.x
- **Language standard**: C (gnu99), C++ (gnu++11)

## Top-Level Directory Structure

```
tigervnc/
├── CMakeLists.txt          # Root build file; orchestrates all sub-builds
├── config.h.in             # Template for generated config.h
├── BUILDING.txt            # Build instructions for all platforms
├── README.rst              # Project overview
├── LICENCE.TXT             # GPL v2
│
├── cmake/                  # Custom CMake modules (FindFLTK, FindPixman, etc.)
├── common/                 # Platform-independent shared libraries
│   ├── core/               # Low-level utilities: logging, configuration, strings, timers
│   ├── rfb/                # RFB (Remote Framebuffer) protocol library
│   ├── rdr/                # Reader/writer stream abstractions
│   └── network/            # TCP socket abstractions
│
├── vncviewer/              # **THE CLIENT** — cross-platform VNC viewer (FLTK)
│   ├── CMakeLists.txt      # Viewer build; links core, rfb, network, rdr, FLTK
│   └── (many .cxx/.h)      # See detailed breakdown below
│
├── win/                    # Windows-specific SERVER code (winvnc) — mostly irrelevant
│   ├── rfb_win32/          # Win32 server-side RFB helpers
│   ├── winvnc/             # The Windows VNC server (unmaintained)
│   ├── vncconfig/          # Windows server configuration UI
│   └── wm_hooks/           # Windows message hooks for server
│
├── unix/                   # Unix-specific server code (Xvnc, x0vncserver, w0vncserver)
├── java/                   # Java VNC viewer (alternative, lower performance)
├── tests/                  # Unit tests
├── po/                     # Translation files (gettext .po)
├── media/                  # Icons and branding assets
├── contrib/                # Distribution-specific packaging (RPM, DEB)
├── release/                # Installer scripts (Inno Setup for Windows)
└── doc/                    # Documentation
```

## The VNC Viewer (`vncviewer/`) — Detailed File Map

This is the directory we care about. Everything below is the **client** that
renders the remote desktop locally and sends input back to the server.

### Entry Point & Configuration

| File | Role |
|------|------|
| `vncviewer.cxx` | `main()` — CLI parsing, FLTK app init, connection setup, main event loop |
| `vncviewer.h` | Shared declarations for the viewer app |
| `parameters.h/.cxx` | All configurable parameters (BoolParameter, IntParameter, etc.). Includes: `fullScreen`, `viewOnly`, `remoteResize`, `desktopSize`, `shortcutModifiers`, `pointerEventInterval`, `acceptClipboard`, etc. |
| `CConn.h/.cxx` | **Client Connection** — manages the RFB connection lifecycle, framebuffer updates, key/mouse event forwarding to server |

### Window & Rendering Pipeline

| File | Role |
|------|------|
| `DesktopWindow.h/.cxx` | **The top-level FLTK window** (extends `Fl_Window`). Manages scrollbars, fullscreen, overlays, compositing to `offscreen` Surface, and the final `draw()` call that blits to screen |
| `Viewport.h/.cxx` | **The widget displaying the remote desktop** (extends `Fl_Widget`). Holds the `PlatformPixelBuffer`, handles mouse/keyboard events, context menu, cursor rendering |
| `PlatformPixelBuffer.h/.cxx` | **The framebuffer** — extends both `rfb::FullFramePixelBuffer` (protocol data) and `Surface` (renderable bitmap). Tracks damaged regions. |
| `Surface.h` | Abstract rendering surface (platform-specific implementations below) |
| `Surface.cxx` | Cross-platform constructor/destructor (calls `alloc()`, `dealloc()`, `update()`) |
| `Surface_Win32.cxx` | **Win32 implementation**: `alloc()` creates a DIBSection (32-bit BGRA); `draw()` uses `BitBlt(SRCCOPY)` for 1:1 pixel copy; `blend()` uses `AlphaBlend()` for overlays |
| `Surface_X11.cxx` | X11 implementation using XRender |
| `Surface_OSX.cxx` | macOS implementation using CoreGraphics |

#### Rendering Flow (Win32)

```
Server sends framebuffer update
  → CConn decodes into PlatformPixelBuffer (raw pixel data in DIBSection)
  → Viewport::updateWindow() marks FLTK damage region
  → FLTK schedules redraw
  → DesktopWindow::draw() is called:
      1. Composites to offscreen Surface (if needed)
      2. Calls viewport->draw(offscreen) which calls:
         frameBuffer->draw(dst, src_x, src_y, dst_x, dst_y, W, H)
           → This is Surface::draw(Surface*,...) in Surface_Win32.cxx
           → Uses BitBlt(SRCCOPY) — a strict 1:1 pixel copy, no scaling
      3. Renders overlays (stats graph, tips) via blend()
      4. Blits offscreen to actual window via offscreen->draw(X, Y, X, Y, W, H)
           → Another BitBlt(SRCCOPY)
```

**Key insight for scaling**: The entire rendering pipeline uses `BitBlt` with
`SRCCOPY` which is strictly 1:1. To add integer scaling (e.g. 2x), we would
replace the final `BitBlt` with `StretchBlt` (or `StretchDIBits`) using
`COLORONCOLOR` stretch mode (nearest-neighbor) and adjust the window/viewport
sizing math accordingly.

### Keyboard & Input Handling

| File | Role |
|------|------|
| `Keyboard.h` | Abstract keyboard interface: `handleEvent()`, `translateToKeySyms()`, `getLEDState()`, `setLEDState()` |
| `KeyboardWin32.h/.cxx` | **Win32 keyboard implementation**: Processes `WM_KEYDOWN/WM_KEYUP/WM_SYSKEYDOWN/WM_SYSKEYUP` messages. Maps Windows virtual keys → X11 KeySyms via `vkey_map` tables. Handles AltGr detection, dead keys, shift tracking. |
| `KeyboardX11.h/.cxx` | X11 keyboard implementation |
| `KeyboardMacOS.h/.mm` | macOS keyboard implementation |
| `keysym2ucs.c/.h` | Unicode ↔ KeySym conversion tables |
| `keyucsmap.h` | Auto-generated KeySym-to-Unicode map |

#### Key Event Flow (Win32)

```
Windows WM_KEYDOWN message
  → Fl::add_system_handler routes to Viewport::handleSystemEvent()
  → keyboard->handleEvent(msg) — KeyboardWin32::handleEvent()
      1. Extracts vKey, scanCode, isExtended from MSG
      2. AltGr detection logic
      3. Fixes up scan codes (fixSystemKeyCode)
      4. Translates scan code → RFB key code (translateSystemKeyCode)
      5. Translates vKey → X11 KeySym (translateVKey) using ToUnicode() + vkey_map tables
      6. Calls handler->handleKeyPress(systemKeyCode, keyCode, keySym)
  → Viewport::handleKeyPress() — checks for shortcuts first, then:
  → cc->sendKeyPress(systemKeyCode, keyCode, keySym)
  → CMsgWriter::writeKeyEvent() — sends over the wire
```

**Key insight for key remapping**: There is already a `rfb::KeyRemapper` class
(in `common/rfb/KeyRemapper.h/.cxx`) with a `RemapKeys` parameter that accepts
comma-separated `0xFrom->0xTo` hex keysym mappings. This is a server-side
feature though. For client-side remapping, the best hook point is in
`Viewport::handleKeyPress()` or `Viewport::sendKeyPress()` where we can
intercept the `keyCode`/`keySym` before it's sent to the server.

### Shortcut / Context Menu System

| File | Role |
|------|------|
| `ShortcutHandler.h/.cxx` | State machine for detecting keyboard shortcuts. Uses modifier mask (Ctrl, Shift, Alt, Super/Win). States: Idle → Arming → Armed → Firing. Detects modifier-then-key combos. |

#### Shortcut Mechanism

The shortcut system works as follows:
1. `shortcutModifiers` parameter (default: `Ctrl,Alt`) defines which modifier keys arm the shortcut
2. User holds all modifier keys → state becomes `Armed`
3. User presses a non-modifier key → state becomes `Firing`, returns `KeyShortcut`
4. `Viewport::handleKeyPress()` checks the KeySym of the fired shortcut:
   - `Space` → bypass mode (send raw keys)
   - `G/g` → grab keyboard
   - `M/m` → **popup context menu** (`popupContextMenu()`)
   - `Enter/KP_Enter` → toggle fullscreen
5. On release of all modifier keys while `Armed` → `KeyUnarm` → ungrab keyboard

**The old F8 behavior** was replaced by this modifier-based system. The current
default is `Ctrl+Alt+M` to show the context menu.

#### Context Menu (`Viewport::popupContextMenu()`)

Items: Disconnect, Full screen, Minimize, Resize window to session, Ctrl (toggle),
Alt (toggle), Send Ctrl-Alt-Del, Refresh screen, Options..., Connection info...,
About TigerVNC...

### Platform-Specific Helpers

| File | Role |
|------|------|
| `win32.h/.c` | Low-level keyboard grab via `SetWindowsHookEx(WH_KEYBOARD_LL)`. Creates a separate thread for the hook. Translates low-level vkeys to generic ones. |
| `x11.h/.cxx` | X11 keyboard/pointer grab, window property queries |
| `cocoa.h/.mm` | macOS-specific window management |

### Other Files

| File | Role |
|------|------|
| `EmulateMB.h/.cxx` | Middle-button emulation (chord left+right) |
| `GestureHandler.h/.cxx` | Touch gesture recognition (X11) |
| `Win32TouchHandler.h/.cxx` | Windows touch input |
| `BaseTouchHandler.h/.cxx` | Base touch handler |
| `touch.h/.cxx` | Touch initialization |
| `AuthDialog.h/.cxx` | Authentication dialog (username/password) |
| `ServerDialog.h/.cxx` | Server address input dialog |
| `OptionsDialog.h/.cxx` | Options/preferences dialog |
| `MonitorIndicesParameter.h/.cxx` | Multi-monitor fullscreen selection |
| `fltk/` | Custom FLTK widgets (monitor arrangement, suggestion input, navigation, theming, event dispatch) |
| `i18n.h` | Internationalization macros |
| `gettext.h` | Gettext stub for non-NLS builds |

## Common Libraries (`common/`)

### `common/core/`
Low-level utilities shared by all components:
- `Configuration.h/.cxx` — Parameter system (`BoolParameter`, `IntParameter`, `StringParameter`, `EnumParameter`, `EnumListParameter`)
- `LogWriter.h/.cxx` — Logging infrastructure
- `string.h` — String utilities (UTF-8 validation, format, etc.)
- `Region.h/.cxx`, `Rect.h` — Geometry primitives
- `Timer.h/.cxx` — Timer utilities
- `Exception.h` — Exception types (including `win32_error`)
- `xdgdirs.h` — XDG directory resolution

### `common/rfb/`
The RFB protocol implementation:
- `CConnection.h/.cxx` — Client-side connection state machine
- `CMsgWriter.h/.cxx` — Writes RFB messages (key events, pointer events, framebuffer requests)
- `CMsgReader.h/.cxx` — Reads RFB messages from server
- `PixelBuffer.h/.cxx` — Pixel buffer abstractions (`FullFramePixelBuffer`)
- `PixelFormat.h/.cxx` — Pixel format negotiation (RGB ordering, depth, etc.)
- `ClientParams.h/.cxx`, `ServerParams.h/.cxx` — Connection parameters
- `Cursor.h/.cxx` — Remote cursor image
- `KeyRemapper.h/.cxx` — **Existing key remapping** (server-side, `RemapKeys` parameter, `0xFrom->0xTo` syntax)
- `Decoder.h`, `*Decoder.cxx` — Encoding decoders (Raw, RRE, Hextile, Tight, ZRLE, H264, JPEG, CopyRect)
- `DecodeManager.h/.cxx` — Multi-threaded decoding
- `Security*.h/.cxx` — Authentication and encryption
- `ScreenSet.h` — Multi-monitor screen layout
- `keysymdef.h` — X11 KeySym definitions

### `common/network/`
- `TcpSocket.h/.cxx` — TCP networking

### `common/rdr/`
- Stream reader/writer abstractions for the protocol

## Build System Notes

### Building the Windows Viewer with MinGW

```bash
cd {build_directory}
cmake -G "MSYS Makefiles" [flags] {source_directory}
make
```

### Required Dependencies (Windows)
- CMake >= 3.10
- MinGW or MinGW-w64 (GCC)
- zlib
- pixman
- FLTK 1.3.3+
- libjpeg-turbo
- Optional: GnuTLS (for TLS), Nettle (for RSA-AES), gettext (for NLS)

### Key Build Variables
- `-DCMAKE_BUILD_TYPE=Debug` for debug build
- `-DBUILD_STATIC=1` for portable/semi-static build
- `-DBUILD_VIEWER=ON` (default) to build the viewer
- `-DBUILD_WINVNC=1` (default on Windows) to build the server

## Modification Points for Our Goals

### 0. Mouse Wheel Scrolls 1 Pixel Instead of 1 Line (BUG)

**Complexity: Low** — Touches 1 file

The bug is in `Viewport::handle()` (Viewport.cxx lines 498-515). When an
`FL_MOUSEWHEEL` event arrives:

```cpp
if (Fl::event_dy() < 0)
    wheelMask |= 1 << 3;   // scroll up = button 4
if (Fl::event_dy() > 0)
    wheelMask |= 1 << 4;   // scroll down = button 5
```

Two problems:

1. **Magnitude is discarded**: `Fl::event_dy()` returns the number of "notches"
   scrolled (e.g., -3 for three fast notches up). The code only checks the
   sign (`< 0` or `> 0`), then sends exactly ONE button-4/5 press+release
   pair regardless of magnitude. Fast scrolling is completely lost.

2. **Remote server interprets each button 4/5 as 1 pixel**: In the RFB
   protocol, scroll wheel is modeled as button press/release. The remote
   VNC server (e.g., Apple Screen Sharing on macOS) translates each button
   4/5 into a scroll event. Some servers use 1 pixel per event rather than
   1 line. There is nothing the client can do about the server's translation
   except send MORE events to compensate.

**Fix approach**: Loop `abs(event_dy())` times, and optionally multiply by a
configurable `scrollWheelMultiplier` parameter:

```cpp
if (event == FL_MOUSEWHEEL) {
    int dy_count = abs(Fl::event_dy()) * scrollWheelMultiplier;
    int dx_count = abs(Fl::event_dx()) * scrollWheelMultiplier;
    for (int i = 0; i < dy_count; i++) {
        wheelMask = 0;
        if (Fl::event_dy() < 0) wheelMask = 1 << 3;
        if (Fl::event_dy() > 0) wheelMask = 1 << 4;
        handlePointerEvent(pos, buttonMask | wheelMask);
        handlePointerEvent(pos, buttonMask);
    }
    // same for dx...
}
```

Touch points: `Viewport.cxx`, `parameters.h/.cxx` (for the multiplier).

### 1. Integer Pixel Scaling (e.g. 2x)

**Complexity: Medium** — Touches ~4 files

The rendering is strictly 1:1 today. Every pixel from the remote framebuffer
becomes exactly one pixel on screen. The modification points are:

- **`Surface_Win32.cxx`** — `Surface::draw()` method: Replace `BitBlt(SRCCOPY)`
  with `StretchBlt()` using `COLORONCOLOR` stretch mode (nearest-neighbor).
  The `draw(Surface*)` variant (for offscreen compositing) also needs updating.

- **`DesktopWindow.cxx`** — `draw()` method: Adjust coordinate mapping. The
  window is currently sized to match the remote framebuffer or uses scrollbars.
  With 2x scaling, the window content area would be `2 * remote_w` by
  `2 * remote_h` pixels, or scrollbars would appear sooner.

- **`Viewport.cxx`** — `handle()` method: Mouse coordinates sent to the server
  must be divided by the scale factor. Currently `Fl::event_x() - x()` gives
  the remote pixel position directly; with 2x scaling this needs to become
  `(Fl::event_x() - x()) / 2`.

- **`parameters.h/.cxx`** — Add a new `IntParameter` for the scale factor.

- **`DesktopWindow.cxx`** — `repositionWidgets()`, `remoteResize()`,
  `resizeFramebuffer()` — All size calculations need to account for scale.

### 2. Client-Side Key Remapping

**Complexity: Low** — Touches ~2 files

There is already a `KeyRemapper` class but it's server-side. For client-side:

- **`Viewport.cxx`** — `handleKeyPress()` and/or `sendKeyPress()`: Add a lookup
  table that maps `keySym` values before sending. Could reuse the existing
  `KeyRemapper` class or add a new client-side `RemapKeys` parameter.

- **`parameters.h/.cxx`** — Add a client-side remap parameter if separate from
  the server one.

The existing `RemapKeys` parameter format (`0xFrom->0xTo`) is already a good
model.

### 3. Title Bar Click → Context Menu

**Complexity: Low-Medium** — Touches ~2 files, platform-specific

The context menu is currently triggered by `Ctrl+Alt+M`. Making it accessible
from title bar clicks requires intercepting non-client area (NC) messages:

- **Windows approach**: Hook `WM_NCLBUTTONDOWN` / `WM_NCRBUTTONDOWN` etc.
  on the FLTK window. FLTK doesn't natively expose non-client area events,
  so this requires either:
  - Subclassing the Win32 window (via `SetWindowLongPtr(GWLP_WNDPROC)`) to
    intercept NC messages, OR
  - Using `Fl::add_system_handler()` to catch `WM_NC*` messages before FLTK
    processes them

- **`DesktopWindow.cxx`** or a new `win32_titlebar.cxx` — Add the NC message
  handler that calls `viewport->popupContextMenu()` (or the equivalent).

- Note: Right-click on title bar normally shows the Windows system menu
  (Move, Size, Minimize, Maximize, Close). Overriding this is possible but
  may confuse users. Middle-click might be a better choice as it has no
  default behavior. Or we could add our items to the system menu itself.
