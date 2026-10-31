# kwin-gestures — Custom Build for Kubuntu 26.04 (Plasma 6.6)

Fork of [gbytedev/kwin-gestures](https://github.com/gbytedev/kwin-gestures) patched for KWin 6.6 API compatibility.

## Problem

The upstream `kwin-gestures` plugin targets KWin ≤ 6.2 API where gesture filter methods used:
```cpp
bool holdGestureBegin(int fingerCount, std::chrono::microseconds time);
bool swipeGestureUpdate(const QPointF &delta, std::chrono::microseconds time);
```

KWin 6.3+ (Plasma 6.1+) changed to event structs:
```cpp
bool holdGestureBegin(PointerHoldGestureBeginEvent *event);
bool swipeGestureUpdate(PointerSwipeGestureUpdateEvent *event);
```

This causes build failures on Kubuntu 26.04 (KWin 6.6.6).

## Solution

This fork adds conditional compilation for KWin ≥ 6.3 using `KWIN_6_3_OR_GREATER` define (detected via `KWin_VERSION` in CMake).

### Files Patched

| File | Changes |
|------|---------|
| `src/kwin/inputfilter.h` | Added `#ifdef KWIN_6_3_OR_GREATER` overloads for all 12 gesture methods using event struct pointers |
| `src/kwin/inputfilter.cpp` | Implemented both API versions; new API extracts `fingerCount`, `delta`, `scale`, `angleDelta`, `time` from event structs; uses member function pointers for `processSpies/processFilters` |
| `src/kwin/CMakeLists.txt` | Adds `KWIN_6_3_OR_GREATER` compile definition when KWin version ≥ 6.2.90 |

### Key Implementation Details

**Event struct mapping** (from `/usr/include/kwin/input_event.h`):
```cpp
// PointerHoldGestureBeginEvent
int fingerCount;
std::chrono::microseconds time;

// PointerSwipeGestureUpdateEvent
QPointF delta;
std::chrono::microseconds time;

// PointerPinchGestureUpdateEvent
qreal scale;
qreal angleDelta;
QPointF delta;
std::chrono::microseconds time;
```

**processSpies/processFilters** now use member function pointers:
```cpp
// Old (lambda) - doesn't work with template member function pointer API
KWin::input()->processSpies([&cancelEvent](auto &&spy) {
    spy->holdGestureCancelled(&cancelEvent);
});

// New (member function pointer)
KWin::input()->processSpies(&KWin::InputEventSpy::holdGestureCancelled, &cancelEvent);
```

## Build Instructions (Kubuntu 26.04 / Plasma 6)

### Dependencies
```bash
sudo apt install -y git cmake g++ extra-cmake-modules qt6-tools-dev \
  kwin-wayland kwin-dev libkf6configwidgets-dev gettext \
  libkf6kcmutils-dev libyaml-cpp-dev libxkbcommon-dev libdrm-dev
```

### Build & Install
```bash
git clone https://github.com/gbytedev/kwin-gestures
cd kwin-gestures

# Apply patches from this repo
cp -r ../customized_apps_ubuntu/kwin-gestures/patches/* .

mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr
make -j$(nproc)
sudo make install
```

### Installed Files
- `/usr/lib/x86_64-linux-gnu/qt6/plugins/kwin/effects/plugins/kwin_gestures.so`
- `/usr/lib/x86_64-linux-gnu/qt6/plugins/kwin/effects/configs/kwin_gestures_kcm.so`

## Enable

1. **System Settings → Desktop Effects → Tools → Gestures** (enable)
2. **System Settings → Input Devices → Touchpad → Gestures** (configure)
3. Log out/in or run `kwin_wayland --replace &`

## Upstream Tracking

| Upstream | Version | KWin API |
|----------|---------|----------|
| gbytedev/kwin-gestures | 0.4.0 | ≤ 6.2 |
| This fork | 0.4.0+kubuntu26.04 | 6.3+ (conditional) |

Check upstream for updates: `git log --oneline upstream/main`