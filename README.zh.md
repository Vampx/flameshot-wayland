# Flameshot - GNOME Wayland 跨屏截图增强版

基于 [Flameshot v14.0.0](https://github.com/flameshot-org/flameshot) 修改，解决在 GNOME Wayland 双屏环境下无法跨屏截图的问题。

## 问题

在 GNOME Wayland 双屏（或多屏）环境下，Flameshot 截图时：

1. 只能在**单个显示器**上显示选区，无法跨屏框选
2. 需要先选择显示器，操作繁琐
3. 微信截图可以跨屏选区，Flameshot 原版不行

**原因**：Wayland 协议不允许窗口跨显示器，但 XWayland 可以。

## 改动

仅修改了 `src/widgets/capture/capturewidget.cpp`，共三处改动：

### 1. 截取全桌面（不选择显示器）

```diff
- m_context.screenshot = grabber.grabEntireDesktop(ok, preSelectedMonitor);
+ m_context.screenshot = grabber.grabFullDesktop(ok);
```

使用 `grabFullDesktop` 替代 `grabEntireDesktop`，一次性截取所有显示器的画面。

### 2. 窗口覆盖全桌面

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

窗口大小覆盖所有显示器，而非仅限单个屏幕。

### 3. 选区覆盖全桌面

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

选区范围覆盖全桌面，允许跨屏框选。

## 编译安装

### 依赖

```bash
# Ubuntu/Debian
sudo apt install build-essential cmake extra-cmake-modules \
    libqt5svg5-dev qttools5-dev qttools5-dev-tools \
    libkf5notifications-dev libkf5xmlgui-dev libkf5coreaddons-dev \
    libkf5dbusaddons-dev libkf5windowsystem-dev
```

### 编译

克隆上游仓库并应用补丁：

```bash
git clone https://github.com/flameshot-org/flameshot.git
cd flameshot
git checkout 090033f  # v14.0.0
git apply cross-monitor.patch
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

### 安装

```bash
sudo make install
```

这会将修改版 flameshot 安装到 `/usr/bin/flameshot`，覆盖系统包版本。原版会备份为 `/usr/bin/flameshot.bak`。

> **注意**：系统更新 flameshot 包后需要重新编译安装。

## 使用

### XWayland 包装脚本

截图时需要强制使用 XWayland 模式（`QT_QPA_PLATFORM=xcb`），否则 Wayland 下窗口仍然无法跨屏。

创建包装脚本 `/usr/local/bin/flameshot-xcb`：

```bash
#!/bin/bash
export QT_QPA_PLATFORM=xcb
exec flameshot gui
```

```bash
sudo chmod +x /usr/local/bin/flameshot-xcb
```

### GNOME 快捷键设置

```bash
# 添加自定义快捷键（例如 Alt+A）
gsettings set org.gnome.settings-daemon.plugins.media-keys custom-keybindings "['/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/']"
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ name 'Flameshot'
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ command '/usr/local/bin/flameshot-xcb'
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/ binding '<Alt>a'
```

按下快捷键后，截图覆盖层会横跨所有显示器，可以自由跨屏框选。

## 文件说明

| 文件 | 说明 |
|------|------|
| `capturewidget.cpp` | 修改后的源文件（直接替换） |
| `cross-monitor.patch` | 针对 v14.0.0 (commit `09033f`) 的补丁 |
| `flameshot-xcb` | XWayland 包装脚本 |

## 来源

- 上游仓库：[flameshot-org/flameshot](https://github.com/flameshot-org/flameshot)
- 基于版本：v14.0.0 (commit `090033f`)

## License

GPL-3.0（与上游一致）
