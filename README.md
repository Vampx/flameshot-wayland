# Flameshot - Cross-Monitor Screenshot for GNOME Wayland

Patched version of [Flameshot v14.0.0](https://github.com/flameshot-org/flameshot) that enables cross-monitor screenshot selection under GNOME Wayland with dual/multi monitors.

## The Problem

On GNOME Wayland with dual or multiple monitors, Flameshot:

1. Only shows the selection overlay on a **single monitor** — no cross-monitor selection
2. Requires you to select a monitor first
3. WeChat's screenshot tool handles this correctly, but stock Flameshot doesn't

**Root cause**: The Wayland protocol doesn't allow windows to span across displays, but XWayland does.

## Changes

Only `src/widgets/capture/capturewidget.cpp` was modified. Three changes total:

### 1. Capture all monitors at once

```diff
- m_context.screenshot = grabber.grabEntireDesktop(ok, preSelectedMonitor);
+ m_context.screenshot = grabber.grabFullDesktop(ok);
```

Uses `grabFullDesktop` instead of `grabEntireDesktop` to capture the full desktop in one pass.

### 2. Span window across all monitors

```diff
- QRect screenGeom = selectedScreen->geometry();
- move(screenGeom.topLeft());
- resize(screenGeom.size());
+ QRect totalGeom;
+ for (QScreen* const screen : QGuiApplication::screens()) {
+     totalGeom = totalGeom.united(screen->geometry());
+ }
+ move(totalGeom.topLeft());
+ resize(totalGeom.size());
```

The overlay window covers the entire desktop instead of a single screen.

### 3. Selection area covers all monitors

```diff
- QRect r = screenForAreas ? screenForAreas->geometry() : QRect();
- r.moveTo(0, 0);
- areas.append(r);
+ QRect totalArea;
+ for (QScreen* const screen : QGuiApplication::screens()) {
+     totalArea = totalArea.united(screen->geometry());
+ }
+ totalArea.moveTo(0, 0);
+ areas.append(totalArea);
```

The selection area spans the full desktop, enabling cross-monitor selection.

## Build & Install

### Dependencies

```bash
# Ubuntu/Debian
sudo apt install build-essential cmake extra-cmake-modules \
    libqt5svg5-dev qttools5-dev qttools5-dev-tools \
    libkf5notifications-dev libkf5xmlgui-dev libkf5coreaddons-dev \
    libkf5dbusaddons-dev libkf5windowsystem-dev
```

### Build

Clone the upstream repo and apply the patch:

```bash
git clone https://github.com/flameshot-org/flameshot.git
cd flameshot
git checkout 090033f  # v14.0.0
git apply cross-monitor.patch
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

### Install

```bash
sudo make install
```

This installs the patched Flameshot to `/usr/bin/flameshot`, replacing the system package version. The original binary is backed up as `/usr/bin/flameshot.bak`.

> **Note**: You need to rebuild after system updates that upgrade the flameshot package.

## Usage

### XWayland Wrapper

Flameshot must run under XWayland (`QT_QPA_PLATFORM=xcb`) for cross-monitor selection to work. Create a wrapper script at `/usr/local/bin/flameshot-xcb`:

```bash
#!/bin/bash
export QT_QPA_PLATFORM=xcb
exec flameshot gui
```

```bash
sudo chmod +x /usr/local/bin/flameshot-xcb
```

### GNOME Keyboard Shortcut

```bash
# Add custom shortcut (e.g. Alt+A)
gsettings set org.gnome.settings-daemon.plugins.media-keys custom-keybindings "['/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/']"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ name 'Flameshot'
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ command '/usr/local/bin/flameshot-xcb'
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ binding '<Alt>a'
```

Press the shortcut and the screenshot overlay will span all monitors, allowing free cross-monitor selection.

## Files

| File | Description |
|------|-------------|
| `capturewidget.cpp` | Modified source file (drop-in replacement) |
| `cross-monitor.patch` | Patch against upstream v14.0.0 (commit `090033f`) |
| `flameshot-xcb` | XWayland wrapper script |

[中文文档](README.zh.md)

## Source

- Upstream: [flameshot-org/flameshot](https://github.com/flameshot-org/flameshot)
- Based on: v14.0.0 (commit `090033f`)

## License

GPL-3.0 (same as upstream)
