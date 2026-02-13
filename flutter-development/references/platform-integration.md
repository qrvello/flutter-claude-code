# Flutter Platform Integration Reference

Consolidated reference for cross-platform communication between Flutter (Dart) and
native code on iOS (Swift) and Android (Kotlin).

## Table of Contents

1. [Channel Types Overview](#channel-types-overview)
2. [MethodChannel: Complete Cross-Platform Example](#methodchannel-complete-cross-platform-example)
3. [EventChannel: Streaming Data](#eventchannel-streaming-data)
4. [Type-Safe Channels with Pigeon](#type-safe-channels-with-pigeon)
5. [iOS-Specific Configuration](#ios-specific-configuration)
6. [Android-Specific Configuration](#android-specific-configuration)
7. [Error Handling Across Platform Boundaries](#error-handling-across-platform-boundaries)
8. [Best Practices for API Parity](#best-practices-for-api-parity)

---

## Channel Types Overview

| Channel              | Pattern            | Use Case                                   |
|----------------------|--------------------|--------------------------------------------|
| MethodChannel        | Request / Response | One-off calls: CRUD, auth, device info     |
| EventChannel         | Stream             | Continuous data: sensors, location, battery |
| BasicMessageChannel  | Binary messages    | Large binary payloads, custom serialization |

All channels use a unique string name (e.g., `com.example.app/feature`) that must
match exactly between Dart and native code.

```dart
// BasicMessageChannel quick reference (rarely needed)
class ImageChannel {
  static const _channel = BasicMessageChannel<Uint8List>(
    'com.example.app/images', BinaryCodec(),
  );
  Future<Uint8List?> processImage(Uint8List data) => _channel.send(data);
}
```

---

## MethodChannel: Complete Cross-Platform Example

Biometric authentication implemented across Dart, Swift, and Kotlin.

### Dart

```dart
// lib/services/biometric_service.dart
import 'package:flutter/services.dart';

class BiometricService {
  static const _channel = MethodChannel('com.example.app/biometric');

  Future<BiometricResult> authenticate({required String reason}) async {
    try {
      final result = await _channel.invokeMethod<Map>(
        'authenticate', {'reason': reason});
      return BiometricResult.fromMap(result!);
    } on PlatformException catch (e) {
      return BiometricResult.error(e.message ?? 'Authentication failed');
    }
  }

  Future<bool> isAvailable() async {
    try {
      return await _channel.invokeMethod<bool>('isAvailable') ?? false;
    } catch (e) { return false; }
  }
}

class BiometricResult {
  final bool success;
  final String? error;
  BiometricResult.success() : success = true, error = null;
  BiometricResult.error(this.error) : success = false;

  factory BiometricResult.fromMap(Map map) => map['success'] == true
      ? BiometricResult.success()
      : BiometricResult.error(map['error'] as String?);
}
```

### iOS (Swift)

```swift
// ios/Runner/AppDelegate.swift
import UIKit
import Flutter
import LocalAuthentication

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    let controller = window?.rootViewController as! FlutterViewController
    let channel = FlutterMethodChannel(
      name: "com.example.app/biometric",
      binaryMessenger: controller.binaryMessenger)
    let handler = BiometricHandler()

    channel.setMethodCallHandler { (call, result) in
      switch call.method {
      case "authenticate":
        guard let args = call.arguments as? [String: Any],
              let reason = args["reason"] as? String else {
          result(FlutterError(code: "INVALID_ARGS", message: "Missing reason", details: nil))
          return
        }
        handler.authenticate(reason: reason, result: result)
      case "isAvailable":
        handler.isAvailable(result: result)
      default:
        result(FlutterMethodNotImplemented)
      }
    }
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}

class BiometricHandler {
  func authenticate(reason: String, result: @escaping FlutterResult) {
    let context = LAContext()
    var error: NSError?
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                                    error: &error) else {
      result(["success": false,
              "error": error?.localizedDescription ?? "Biometrics unavailable"])
      return
    }
    context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                           localizedReason: reason) { success, authError in
      DispatchQueue.main.async {
        result(success
          ? ["success": true]
          : ["success": false, "error": authError?.localizedDescription ?? "Failed"])
      }
    }
  }

  func isAvailable(result: FlutterResult) {
    let ctx = LAContext()
    result(ctx.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil))
  }
}
```

### Android (Kotlin)

```kotlin
// android/app/src/main/kotlin/com/example/app/MainActivity.kt
package com.example.app

import androidx.biometric.BiometricManager
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import io.flutter.embedding.android.FlutterFragmentActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterFragmentActivity() {
    private val CHANNEL = "com.example.app/biometric"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "authenticate" -> {
                        val reason = call.argument<String>("reason")
                            ?: return@setMethodCallHandler result.error(
                                "INVALID_ARGS", "Missing reason", null)
                        authenticate(reason, result)
                    }
                    "isAvailable" -> isAvailable(result)
                    else -> result.notImplemented()
                }
            }
    }

    private fun authenticate(reason: String, result: MethodChannel.Result) {
        val prompt = BiometricPrompt(this, ContextCompat.getMainExecutor(this),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(r: BiometricPrompt.AuthenticationResult) {
                    result.success(mapOf("success" to true))
                }
                override fun onAuthenticationFailed() {
                    result.success(mapOf("success" to false, "error" to "Failed"))
                }
                override fun onAuthenticationError(code: Int, errString: CharSequence) {
                    result.success(mapOf("success" to false, "error" to errString.toString()))
                }
            })
        prompt.authenticate(BiometricPrompt.PromptInfo.Builder()
            .setTitle("Biometric Authentication").setSubtitle(reason)
            .setNegativeButtonText("Cancel").build())
    }

    private fun isAvailable(result: MethodChannel.Result) {
        val ok = BiometricManager.from(this).canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG)
        result.success(ok == BiometricManager.BIOMETRIC_SUCCESS)
    }
}
```

---

## EventChannel: Streaming Data

Location streaming across Dart, Swift, and Kotlin.

### Dart

```dart
class LocationService {
  static const _channel = EventChannel('com.example.app/location');
  Stream<Map<String, double>>? _stream;

  Stream<Map<String, double>> get locationUpdates {
    _stream ??= _channel.receiveBroadcastStream()
        .map((event) => Map<String, double>.from(event as Map));
    return _stream!;
  }
}

// Widget usage
StreamBuilder<Map<String, double>>(
  stream: LocationService().locationUpdates,
  builder: (context, snapshot) {
    if (snapshot.hasData) {
      return Text('${snapshot.data!['latitude']}, ${snapshot.data!['longitude']}');
    }
    return CircularProgressIndicator();
  },
)
```

### iOS (Swift)

```swift
import CoreLocation

class LocationStreamHandler: NSObject, FlutterStreamHandler {
  private var eventSink: FlutterEventSink?
  private let locationManager = CLLocationManager()

  func onListen(withArguments arguments: Any?,
                eventSink events: @escaping FlutterEventSink) -> FlutterError? {
    eventSink = events
    locationManager.delegate = self
    locationManager.requestWhenInUseAuthorization()
    locationManager.startUpdatingLocation()
    return nil
  }

  func onCancel(withArguments arguments: Any?) -> FlutterError? {
    locationManager.stopUpdatingLocation()
    eventSink = nil
    return nil
  }
}

extension LocationStreamHandler: CLLocationManagerDelegate {
  func locationManager(_ manager: CLLocationManager,
                       didUpdateLocations locations: [CLLocation]) {
    guard let loc = locations.last else { return }
    eventSink?(["latitude": loc.coordinate.latitude,
                "longitude": loc.coordinate.longitude,
                "accuracy": loc.horizontalAccuracy])
  }
  func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
    eventSink?(FlutterError(code: "LOCATION_ERROR",
                            message: error.localizedDescription, details: nil))
  }
}
// Register: FlutterEventChannel(...).setStreamHandler(LocationStreamHandler())
```

### Android (Kotlin)

```kotlin
import android.annotation.SuppressLint
import android.content.Context
import android.location.LocationListener
import android.location.LocationManager
import io.flutter.plugin.common.EventChannel

class LocationStreamHandler(private val context: Context) : EventChannel.StreamHandler {
    private var eventSink: EventChannel.EventSink? = null
    private var locationManager: LocationManager? = null
    private val listener = LocationListener { loc ->
        eventSink?.success(mapOf("latitude" to loc.latitude,
            "longitude" to loc.longitude, "accuracy" to loc.accuracy.toDouble()))
    }

    @SuppressLint("MissingPermission")
    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        eventSink = events
        locationManager = context.getSystemService(Context.LOCATION_SERVICE) as LocationManager
        locationManager?.requestLocationUpdates(
            LocationManager.GPS_PROVIDER, 1000L, 10f, listener)
    }

    override fun onCancel(arguments: Any?) {
        locationManager?.removeUpdates(listener)
        eventSink = null
    }
}
// Register: EventChannel(...).setStreamHandler(LocationStreamHandler(this))
```

---

## Type-Safe Channels with Pigeon

Pigeon generates type-safe channel code from a single schema, eliminating string-based
method names and manual argument parsing.

Add `pigeon: ^17.0.0` to `dev_dependencies` in `pubspec.yaml`.

### Schema Definition

```dart
// pigeons/api.dart
import 'package:pigeon/pigeon.dart';

@ConfigurePigeon(PigeonOptions(
  dartOut: 'lib/pigeon/api.g.dart',
  kotlinOut: 'android/app/src/main/kotlin/com/example/app/Pigeon.kt',
  swiftOut: 'ios/Runner/Pigeon.swift',
  kotlinOptions: KotlinOptions(package: 'com.example.app'),
))

class User { String? id; String? name; String? email; }
class LoginRequest { String? email; String? password; }
class LoginResult { bool? success; User? user; String? error; }

@HostApi()  // Dart calls native
abstract class AuthApi {
  @async LoginResult login(LoginRequest request);
  @async bool logout();
  @async User? getCurrentUser();
}

@FlutterApi()  // Native calls Dart
abstract class AuthCallbackApi {
  void onAuthStateChanged(bool isAuthenticated);
}
```

### Code Generation and Usage

```bash
flutter pub run pigeon --input pigeons/api.dart
```

This produces `api.g.dart`, `Pigeon.swift`, and `Pigeon.kt`. Use in Dart:

```dart
final authApi = AuthApi();
final result = await authApi.login(LoginRequest()
  ..email = 'user@example.com'
  ..password = 'password');
if (result.success!) print('Logged in as ${result.user!.name}');
```

On the native side, implement the generated protocol (Swift) or interface (Kotlin).

---

## iOS-Specific Configuration

### Info.plist Permissions

The app crashes at runtime if a required usage description key is missing.

| Key                                | Used For             |
|------------------------------------|----------------------|
| `NSCameraUsageDescription`         | Camera access        |
| `NSPhotoLibraryUsageDescription`   | Photo library        |
| `NSLocationWhenInUseUsageDescription` | Foreground location |
| `NSLocationAlwaysUsageDescription` | Background location  |
| `NSMicrophoneUsageDescription`     | Microphone           |
| `NSHealthShareUsageDescription`    | HealthKit reads      |
| `NSHealthUpdateUsageDescription`   | HealthKit writes     |

Each key requires a human-readable string explaining why the app needs access.
Add `UIBackgroundModes` array with entries like `fetch`, `remote-notification`, or
`location` if the app performs background work.

### Entitlements

HealthKit, Apple Pay, Push Notifications, and App Groups require entitlements configured
in Xcode under Signing & Capabilities. These must match the Apple Developer portal.

### CocoaPods

Set `platform :ios, '13.0'` in `ios/Podfile`. Add native dependencies inside the
`target 'Runner'` block. Use `post_install` to set deployment target on all pods.

Clean stuck pods: `cd ios && rm -rf Pods Podfile.lock && pod install && cd .. && flutter clean`

### Common iOS Framework Integrations

**CoreLocation** -- see EventChannel example above. Requires
`NSLocationWhenInUseUsageDescription` in Info.plist.

**HealthKit** -- enable entitlement in Xcode; add `NSHealthShareUsageDescription` and
`NSHealthUpdateUsageDescription` to Info.plist. Access via `HKHealthStore` natively.

**Camera (AVFoundation)** -- requires `NSCameraUsageDescription`:

```swift
AVCaptureDevice.requestAccess(for: .video) { granted in
  DispatchQueue.main.async { result(granted) }
}
```

---

## Android-Specific Configuration

### AndroidManifest Permissions

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  <uses-permission android:name="android.permission.INTERNET"/>
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
  <uses-permission android:name="android.permission.CAMERA"/>
  <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
  <uses-feature android:name="android.hardware.camera" android:required="false"/>
  <uses-feature android:name="android.hardware.location.gps" android:required="false"/>
</manifest>
```

Dangerous permissions (camera, location, contacts) require a runtime request on API 23+.

### Gradle Configuration

In `android/app/build.gradle`, key settings: `compileSdkVersion 34`,
`minSdkVersion 21`, `targetSdkVersion 34`, `multiDexEnabled true`. For release builds
enable `minifyEnabled true` and `shrinkResources true` with proguard rules. Add native
dependencies (e.g., `androidx.biometric`, `androidx.work:work-runtime-ktx`) in the
`dependencies` block.

Clean stuck builds: `cd android && ./gradlew clean && cd .. && flutter clean`

### Common Android Integrations

**WorkManager** -- deferrable, guaranteed background tasks:

```kotlin
class DataSyncWorker(ctx: Context, params: WorkerParameters) : Worker(ctx, params) {
    override fun doWork(): Result = try { Result.success() }
                                    catch (e: Exception) { Result.retry() }
}

fun scheduleSync(context: Context) {
    val request = PeriodicWorkRequestBuilder<DataSyncWorker>(15, TimeUnit.MINUTES)
        .setConstraints(Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED).build())
        .build()
    WorkManager.getInstance(context)
        .enqueueUniquePeriodicWork("data_sync", ExistingPeriodicWorkPolicy.KEEP, request)
}
```

**Services** -- declare foreground services in manifest with
`android:foregroundServiceType`. Start with `startForeground(id, notification)`.

**BroadcastReceivers** -- register in manifest or dynamically for system events
(`ACTION_BOOT_COMPLETED`, `ACTION_BATTERY_LOW`).

---

## Error Handling Across Platform Boundaries

### Dart: Structured Error Wrapper

```dart
class PlatformResult<T> {
  final T? data;
  final PlatformError? error;
  PlatformResult.success(this.data) : error = null;
  PlatformResult.failure(this.error) : data = null;
  bool get isSuccess => error == null;
}

class PlatformError {
  final String code, message;
  final dynamic details;
  PlatformError(this.code, this.message, [this.details]);
  factory PlatformError.fromException(PlatformException e) =>
      PlatformError(e.code, e.message ?? 'Unknown error', e.details);
}

Future<PlatformResult<T>> safeInvoke<T>(
    MethodChannel channel, String method, [dynamic args]) async {
  try {
    return PlatformResult.success(await channel.invokeMethod<T>(method, args));
  } on PlatformException catch (e) {
    return PlatformResult.failure(PlatformError.fromException(e));
  } catch (e) {
    return PlatformResult.failure(PlatformError('UNEXPECTED', e.toString()));
  }
}
```

### Native Error Patterns

```swift
// iOS (Swift)
result(someValue)                           // success
result(FlutterError(code: "PERMISSION_DENIED", message: "...", details: nil))  // error
result(FlutterMethodNotImplemented)         // not implemented
```

```kotlin
// Android (Kotlin)
result.success(someValue)                   // success
result.error("PERMISSION_DENIED", "...", null) // error
result.notImplemented()                     // not implemented
```

### Key Rules

- Use consistent error codes across iOS and Android so Dart handles them uniformly.
- Dispatch results on the main thread. iOS: `DispatchQueue.main.async`. Android: main
  executor or `runOnUiThread`.
- Every code path must call `result()` exactly once; an unanswered result leaks the
  Dart future.

---

## Best Practices for API Parity

**1. Single channel name per feature.** Use one name on both platforms. Dart should not
need to know which OS it runs on.

**2. Unified Dart interface.** Abstract class with a single MethodChannel implementation.
Both Swift and Kotlin handle the same method names and argument shapes.

```dart
abstract class PlatformService {
  Future<bool> authenticate(String reason);
  Future<bool> isAvailable();
}

class PlatformServiceImpl implements PlatformService {
  static const _channel = MethodChannel('com.example.app/platform');
  @override
  Future<bool> authenticate(String reason) async =>
      await _channel.invokeMethod<bool>('authenticate', {'reason': reason}) ?? false;
  @override
  Future<bool> isAvailable() async =>
      await _channel.invokeMethod<bool>('isAvailable') ?? false;
}
```

**3. Use Pigeon for complex APIs.** Enforces identical signatures at compile time.

**4. Match error codes across platforms.**

| Code                | Meaning                         |
|---------------------|---------------------------------|
| `INVALID_ARGS`      | Missing or malformed arguments  |
| `PERMISSION_DENIED` | User denied a permission        |
| `NOT_AVAILABLE`     | Feature not supported on device |
| `TIMEOUT`           | Operation timed out             |
| `UNKNOWN`           | Catch-all for unexpected errors |

**5. Batch operations.** One `processBatch` call beats N individual `process` calls.

**6. Minimize data transfer.** Pass file paths instead of raw bytes when possible.

**7. Test with mock channels.**

```dart
setUp(() {
  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockMethodCallHandler(channel, (call) async {
    switch (call.method) {
      case 'isAvailable': return true;
      case 'authenticate': return {'success': true};
      default: return null;
    }
  });
});
```

**8. Document the channel contract.** For each channel, list method names, argument
types, return types, and error codes to prevent platform implementations from drifting.
