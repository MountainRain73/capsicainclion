# Capsicain (Windows)

A keyboard remapping tool for Windows that rewires keys and creates powerful key combinations — without changing how you type.

Uses the **Interception** driver to capture and transform keyboard input at the lowest level, before any application sees it.

> **Also available**: [Capsicain for Linux](https://github.com/yourname/capsicainlinux) — native evdev/uinput port without external dependencies.

## Features

- **Key Rewiring** — Remap any key to any other key. Tap vs hold can produce different keys (e.g., CapsLock tap = Esc, hold = Ctrl).
- **Combo System** — Define modifier + key combinations with AND/OR/NOT logic. Support for virtual modifiers (MOD9–MOD15) that don't interfere with real modifier keys.
- **Alt+HJKL Cursor Navigation** — Move cursor with Alt+HJKL (arrows), Alt+Shift+HJKL (select), Alt+CapsLock+HL (word jump), plus Home/End/PgUp/PgDown.
- **Alt+Tab Passthrough** — Native Alt+Tab window switching works through capsicain's modifier proxy via transparent passthrough mode.
- **Multiple Config Layers** — Toggle between up to 10 different key mapping configurations at runtime (ESC+1/2/3...).
- **System Tray Icon** — Minimize to system tray with tray icon showing active config layer.
- **Macro Recording** — Record and play back up to 20 macros, including obfuscated secret macros for passwords.
- **LED Control** — CapsLock/NumLock/ScrollLock LED reflects capsicain's internal state.
- **AutoHotkey Integration** — Auto-launch companion AHK scripts on startup.
- **Dead Keys** — Compose accented characters with dead key sequences.
- **Alpha Remapping** — Swap keyboard layouts (QWERTY ↔ DVORAK, etc.) without changing Windows settings.

## Quick Start

### Prerequisites

- **Windows** (7/8/10/11)
- **Interception driver** — [Download and install](https://github.com/oblitum/Interception) `install-interception.exe`
  - Or use the included installer in `install-interception/`
- **Administrator privileges** — Required for Interception to capture keyboard input

### Option A: Pre-built Binary

1. Install the Interception driver (one-time setup)
2. Download `capsicain.exe`, `interception.dll`, and `capsicain.ini` from the latest release
3. Place all files in the same directory
4. Run `capsicain.exe` as Administrator

### Option B: Build from Source

```bash
# Prerequisites: CLion with Visual Studio toolchain, CMake 3.20+
git clone https://github.com/yourname/capsicainclion.git
cd capsicainclion

# Open in CLion, configure Visual Studio toolchain, then:
# Build > Build Project

# Or from command line (Visual Studio Developer Command Prompt):
mkdir cmake-build-release
cd cmake-build-release
cmake .. -G "Ninja" -DCMAKE_BUILD_TYPE=Release
cmake --build .

# Copy interception.dll alongside the executable
copy ..\src\interception.dll bin\
```

## Runtime Controls

| Shortcut | Action |
|----------|--------|
| `ESC+X` | Exit capsicain |
| `ESC+0` | Disable capsicain (passthrough all keys) |
| `ESC+1..9` | Switch to config layer 1–9 |
| `ESC+Q` | Toggle debug output |
| `ESC+Backspace` | Reset all state |

## Command Line Options

| Flag | Description |
|------|-------------|
| (none) | Run in foreground with console window. |
| `-h`, `--help` | Show help. |

Capsicain runs as a console application. You can minimize the console window — it will continue running. Use the system tray icon for quick access.

## Configuration

Keyboard mappings are defined in `capsicain.ini`. The default configuration provides:

```ini
REWIRE CAPS LCTRL ESC     # CapsLock: tap=Esc, hold=Ctrl
REWIRE LALT MOD9 NOP      # LAlt → virtual MOD9 (prevents app shortcuts)
COMBO LSHF [RSHIFT] > key(CAPS)       # Shift+Shift = CapsLock toggle
COMBO TAB  [MOD9] > alttab()          # Alt+Tab passthrough
COMBO H    [MOD9] > moddedKey(LEFT)    # Alt+H = Left Arrow
COMBO J    [MOD9] > moddedKey(DOWN)    # Alt+J = Down Arrow
COMBO K    [MOD9] > moddedKey(UP)      # Alt+K = Up Arrow
COMBO L    [MOD9] > moddedKey(RIGHT)   # Alt+L = Right Arrow
# ... and many more
```

For a complete reference of all mappings, features, and the full configuration syntax, see the **[capsicain-mappings-reference.md](capsicain-mappings-reference.md)**.

## Project Structure

```
capsicainclion/
├── src/                              # Source code
│   ├── capsicain.h / .cpp            # Main program logic, config parsing
│   ├── configUtils.h / .cpp          # INI config file parser
│   ├── scancodes.h / .cpp            # PS/2 Set 1 scancode ↔ virtual keycode tables
│   ├── modifiers.h / .cpp            # Modifier bitmask definitions
│   ├── constants.h                   # Shared constants
│   ├── utils.h / .cpp                # Utility functions
│   ├── led.h / .cpp                  # Keyboard LED control
│   ├── traybar.h / .cpp              # System tray icon
│   ├── interception.h                # Interception driver API header
│   ├── interception.lib / .dll       # Interception driver library
│   ├── capsicain.rc / resource.h     # Windows resources (icon, version info)
│   ├── icons/                        # Tray icon files
│   └── *.ini                         # Configuration files
├── CMakeLists.txt                    # Build configuration (CLion + Visual Studio)
├── capsicain-mappings-reference.md   # Full technical reference
├── README.md
├── LICENSE
└── .gitignore
```

## How It Works

1. **Capture** — Interception driver installs a low-level keyboard filter, capturing all keystrokes before any application.
2. **Process** — The capsicain engine runs: modifier tracking, key rewiring, COMBO matching, tap detection.
3. **Send** — Modified key events are sent back through Interception to the Windows input stack.

The core processing logic is identical to the Linux version — only the platform abstraction layer differs. See [capsicain-mappings-reference.md](capsicain-mappings-reference.md) for detailed implementation documentation.

## System Requirements

- Windows 7 or later (x86 or x64)
- [Interception driver](https://github.com/oblitum/Interception) installed
- Administrator privileges
- CLion 2022+ with Visual Studio 2017+ toolchain (for building from source)

## License

MIT License — see [LICENSE](LICENSE) for details.

## Acknowledgments

- Keyboard driver powered by [Interception](https://github.com/oblitum/Interception) by Francisco Lopes
- Originally created by [cajhin](https://github.com/cajhin/capsicain)