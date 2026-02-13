# Flutter Device Management Reference

Consolidated reference for managing iOS simulators and Android emulators in Flutter
development. Covers device discovery, launch, configuration, app installation,
screenshot capture, and multi-device testing workflows.

## Table of Contents

1. [iOS Simulator Management](#ios-simulator-management)
2. [Android Emulator Management](#android-emulator-management)
3. [Multi-Device Testing](#multi-device-testing)
4. [Flutter Device Integration](#flutter-device-integration)
5. [Troubleshooting](#troubleshooting)

---

## iOS Simulator Management

All iOS simulator operations use `xcrun simctl`.

### Listing Simulators

```bash
xcrun simctl list devices                        # all simulator devices
xcrun simctl list devices | grep Booted          # only running simulators
xcrun simctl list devicetypes                    # available device types
xcrun simctl list runtimes                       # installed iOS runtimes

# JSON output for a specific runtime
xcrun simctl list devices available --json \
  | jq '.devices["com.apple.CoreSimulator.SimRuntime.iOS-17-2"]'
```

### Launching and Creating Simulators

```bash
# Boot a simulator by device ID
xcrun simctl boot <DEVICE_ID>

# Open the Simulator GUI
open -a Simulator

# Boot and open in one step
xcrun simctl boot <DEVICE_ID> && open -a Simulator

# Create a new simulator (name, device type, runtime)
xcrun simctl create "iPhone 15 Pro Test" "iPhone 15 Pro" "iOS17.2"

# Boot by device name -- common form factors
xcrun simctl boot "iPhone SE (3rd generation)"                    # compact
xcrun simctl boot "iPhone 15 Pro"                                 # standard
xcrun simctl boot "iPhone 15 Pro Max"                             # large
xcrun simctl boot "iPad Pro (12.9-inch) (6th generation)"         # tablet

# Lifecycle commands
xcrun simctl shutdown <DEVICE_ID>    # stop a running simulator
xcrun simctl erase <DEVICE_ID>       # reset to factory state
xcrun simctl delete <DEVICE_ID>      # remove simulator entirely
```

### Configuring Simulators

```bash
# Light / dark mode
xcrun simctl ui <DEVICE_ID> appearance light
xcrun simctl ui <DEVICE_ID> appearance dark

# Override status bar for clean screenshots
xcrun simctl status_bar <DEVICE_ID> override \
  --time "9:41" --dataNetwork wifi --wifiBars 3 \
  --cellularMode active --cellularBars 4 \
  --batteryState charged --batteryLevel 100

# Clear status bar overrides
xcrun simctl status_bar <DEVICE_ID> clear

# Set GPS location (latitude longitude)
xcrun simctl location <DEVICE_ID> set 37.7749 -122.4194

# Trigger a push notification
xcrun simctl push <DEVICE_ID> <BUNDLE_ID> notification.json
```

### iOS Screenshots and Video

```bash
xcrun simctl io <DEVICE_ID> screenshot screenshot.png          # by ID
xcrun simctl io booted screenshot screenshot.png               # current device
xcrun simctl io <DEVICE_ID> screenshot ~/Desktop/shot.png      # custom path
xcrun simctl io <DEVICE_ID> recordVideo video.mov              # Ctrl+C to stop
```

### iOS Logs

```bash
xcrun simctl spawn <DEVICE_ID> log stream                      # all logs
xcrun simctl spawn <DEVICE_ID> log stream \
  --predicate 'processImagePath contains "Runner"'             # Flutter only
xcrun simctl spawn <DEVICE_ID> log show --last 1h              # last hour
```

---

## Android Emulator Management

Android uses three CLI tools: `avdmanager` (create/list AVDs), `emulator`
(launch AVDs), and `adb` (interact with running devices).

### Listing Emulators

```bash
emulator -list-avds                              # all created AVDs
avdmanager list avd                              # AVDs with full details
adb devices                                      # running emulators/devices
sdkmanager --list | grep system-images           # downloadable system images
avdmanager list device                           # device definitions
```

### Creating Emulators

```bash
# Download system images first
sdkmanager "system-images;android-34;google_apis;x86_64"
sdkmanager "system-images;android-33;google_apis;x86_64"

# Create an AVD (name, system image, device definition)
avdmanager create avd \
  --name "Pixel_8_API_34" \
  --package "system-images;android-34;google_apis;x86_64" \
  --device "pixel_8"

# With an SD card
avdmanager create avd \
  --name "Pixel_Tablet_API_34" \
  --package "system-images;android-34;google_apis;x86_64" \
  --device "pixel_tablet" \
  --sdcard 512M

# Overwrite existing AVD
avdmanager create avd \
  --name "Pixel_8_Test" \
  --package "system-images;android-34;google_apis;x86_64" \
  --device "pixel_8" --force

# Common device definitions:
#   pixel_8       standard phone     pixel_8_pro    large phone
#   pixel_tablet  tablet             pixel_fold     foldable
#   pixel_7       previous-gen phone
```

### Launching Emulators

```bash
emulator -avd Pixel_8_API_34                     # basic launch

# Launch with options
emulator -avd Pixel_8_API_34 \
  -no-snapshot-load -no-audio -gpu swiftshader_indirect

emulator -avd Pixel_8_API_34 -no-window          # headless (CI)
emulator -avd Pixel_8_API_34 -memory 4096         # extra RAM
emulator -avd Pixel_8_API_34 -skin 1080x2400      # custom resolution
emulator -avd Pixel_8_API_34 -wipe-data            # factory reset on launch

# Wait for emulator to finish booting
adb wait-for-device
adb shell getprop sys.boot_completed              # "1" when ready
```

### Configuring Emulators

```bash
# GPS location (longitude latitude -- note order differs from iOS)
adb emu geo fix -122.4194 37.7749

# Battery
adb shell dumpsys battery set level 50
adb shell dumpsys battery set status 3            # 3 = discharging

# Airplane mode
adb shell cmd connectivity airplane-mode enable
adb shell cmd connectivity airplane-mode disable

# Brightness, text input, key events
adb shell settings put system screen_brightness 255
adb shell input text "Hello%sWorld"               # %s = space
adb shell input keyevent KEYCODE_HOME
adb shell input keyevent KEYCODE_BACK
```

### Android Screenshots and Video

```bash
# One-line screenshot
adb exec-out screencap -p > screenshot.png

# Two-step (works on all API levels)
adb shell screencap /sdcard/screenshot.png
adb pull /sdcard/screenshot.png ./screenshot.png
adb shell rm /sdcard/screenshot.png

# From a specific emulator when multiple are running
adb -s emulator-5554 exec-out screencap -p > screenshot.png

# Screen recording (Ctrl+C to stop, max 3 min)
adb shell screenrecord /sdcard/video.mp4
adb pull /sdcard/video.mp4 ./video.mp4
adb shell rm /sdcard/video.mp4
```

### Android Logs

```bash
adb logcat                                        # all logs
adb logcat | grep flutter                         # Flutter logs
adb logcat -s "FlutterActivity"                   # by tag
adb logcat -c                                     # clear buffer
adb logcat > logcat.txt                           # save to file

# Device properties
adb shell getprop ro.build.version.release        # Android version
adb shell getprop ro.product.model                # device model
```

---

## Multi-Device Testing

### Managing Multiple Devices

```bash
# See all connected devices (iOS + Android + desktop)
flutter devices

# Example output:
#   iPhone 15 Pro (mobile) . <UDID>        . ios
#   Pixel 8 (mobile)       . emulator-5554 . android
#   macOS (desktop)         . macos         . darwin-arm64

# Run on specific devices in separate terminals
flutter run -d <iOS_UDID>
flutter run -d emulator-5554
```

### Multi-Device Setup Script

```bash
#!/bin/bash
# Launch one iOS simulator and one Android emulator, run the app on both.
set -e

IOS_DEVICE="iPhone 15 Pro"
ANDROID_AVD="Pixel_8_API_34"

xcrun simctl boot "$IOS_DEVICE" && open -a Simulator
emulator -avd "$ANDROID_AVD" -no-snapshot-load &

sleep 10 && adb wait-for-device
echo "Devices ready"

IOS_ID=$(flutter devices | grep "iPhone 15 Pro" | awk '{print $5}' | tr -d '.')
ANDROID_ID=$(flutter devices | grep "Pixel 8" | awk '{print $5}' | tr -d '.')

flutter run -d "$IOS_ID" &
flutter run -d "$ANDROID_ID" &
wait
```

### Screenshot Comparison Workflow

```bash
#!/bin/bash
# Capture screenshots from all running iOS simulators and Android emulators.
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DIR="screenshots/$TIMESTAMP"
mkdir -p "$DIR"

# iOS -- iterate over booted simulators
for id in $(xcrun simctl list devices | grep "Booted" \
    | awk -F'[()]' '{print $(NF-1)}'); do
  name=$(xcrun simctl list devices | grep "$id" | awk -F'(' '{print $1}' | xargs)
  xcrun simctl io "$id" screenshot "$DIR/ios_${name// /_}.png"
done

# Android -- iterate over connected emulators
for id in $(adb devices | grep "emulator" | awk '{print $1}'); do
  name=$(adb -s "$id" shell getprop ro.product.model | tr -d '\r')
  adb -s "$id" exec-out screencap -p > "$DIR/android_${name// /_}.png"
done

echo "Screenshots saved to: $DIR"
```

---

## Flutter Device Integration

### Device Discovery and Running Apps

```bash
flutter devices                                   # list recognized devices
flutter doctor -v                                 # full diagnostics

# Run on a device
flutter run -d <DEVICE_ID>
flutter run -d ios                                # default iOS simulator
flutter run -d android                            # default Android emulator
flutter run -v                                    # verbose logging
flutter run --uninstall-first                     # clean install

# Install without running
flutter install -d <DEVICE_ID>

# Manual iOS build and install
flutter build ios --simulator
xcrun simctl install <DEVICE_ID> build/ios/iphonesimulator/Runner.app

# Manual Android build and install
flutter build apk
adb install build/app/outputs/flutter-apk/app-debug.apk

# Android app management
adb uninstall com.example.app
adb shell pm clear com.example.app
```

### Hot Reload

When `flutter run` is active, these keyboard shortcuts are available:

| Key | Action                         |
|-----|--------------------------------|
| `r` | Hot reload (preserves state)   |
| `R` | Hot restart (resets state)     |
| `p` | Toggle performance overlay    |
| `q` | Quit                           |

Hot reload applies Dart code changes without restarting the app, preserving
in-memory state. Hot restart recompiles and restarts the Dart VM.

---

## Troubleshooting

### iOS Simulator Issues

```bash
# Simulator not appearing -- force restart
killall Simulator && open -a Simulator

# Simulator stuck or frozen
xcrun simctl shutdown all
xcrun simctl erase all
xcrun simctl boot <DEVICE_ID>

# Clear Xcode derived data (stale build artifacts)
rm -rf ~/Library/Developer/Xcode/DerivedData

# Verify Xcode version
xcodebuild -version
```

### Android Emulator Issues

```bash
# Emulator won't start -- check hardware acceleration:
#   macOS:   HAXM (Intel) or Hypervisor.framework (Apple Silicon)
#   Linux:   KVM
#   Windows: WHPX or Hyper-V

# Kill all running emulators
adb devices | grep emulator | cut -f1 | xargs -I {} adb -s {} emu kill

# Factory reset
emulator -avd Pixel_8_API_34 -wipe-data

# ADB server stuck
adb kill-server && adb start-server

# Check version
emulator -version

# Slow emulator -- use x86_64 images, enable HW accel, increase RAM
emulator -avd Pixel_8_API_34 -memory 4096
```

### Flutter Connection Issues

```bash
flutter devices                                   # refresh device list
flutter doctor -v                                 # full diagnostics
flutter clean                                     # clear build cache
flutter run --uninstall-first                     # reinstall from scratch
lsof -i :8080                                    # check port conflicts
flutter run -v                                    # verbose logging
```

---

## Recommended Test Device Matrix

| Platform | Device              | Form Factor | Config Name                               |
|----------|---------------------|-------------|-------------------------------------------|
| iOS      | iPhone SE (3rd gen) | Compact     | `iPhone SE (3rd generation)`              |
| iOS      | iPhone 15 Pro       | Standard    | `iPhone 15 Pro`                           |
| iOS      | iPhone 15 Pro Max   | Large       | `iPhone 15 Pro Max`                       |
| iOS      | iPad Pro 12.9"      | Tablet      | `iPad Pro (12.9-inch) (6th generation)`   |
| Android  | Pixel 8             | Standard    | `pixel_8` / API 34                        |
| Android  | Pixel 8 Pro         | Large       | `pixel_8_pro` / API 34                    |
| Android  | Pixel Tablet        | Tablet      | `pixel_tablet` / API 34                   |
| Android  | Pixel Fold          | Foldable    | `pixel_fold` / API 34                     |
| Android  | Pixel 7 (legacy)    | Standard    | `pixel_7` / API 33                        |
