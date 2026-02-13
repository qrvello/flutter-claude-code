# Flutter Platform Integration Reference

Consolidated reference for cross-platform communication between Flutter (Dart) and
native code on iOS (Swift) and Android (Kotlin). Covers channel types, Pigeon code
generation, platform-specific configuration, and error handling.

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

A biometric authentication feature implemented across Dart, Swift, and Kotlin.

### Dart Side

```dart
// lib/services/biometric_service.dart
import 'package:flutter/services.dart';

class BiometricService {
  static const _channel = MethodChannel('com.example.app/biometric');

  Future<BiometricResult> authenticate({required String reason}) async {
    try {
      final result = await _channel.invokeMethod<Map>(
        'authenticate', {'reason': reason},
      );
      return BiometricResult.fromMap(result!);
    } on PlatformException catch (e) {
      return BiometricResult.error(e.message ?? 'Authentication failed');
    }
  }

  Future<bool> isAvailable() async {
    try {
      return await _channel.invokeMethod<bool>('isAvailable') ?? false;
    } catch (e) {
      return false;
    }
  }

  Future<String> getBiometricType() async {
    try {
      return await _channel.invokeMethod<String>('getBiometricType') ?? 'none';
    } catch (e) {
      return 'none';
    }
  }
}

class BiometricResult {
  final bool success;
  final String? error;
  BiometricResult.success() : success = true, error = null;
  BiometricResult.error(this.error) : success = false;

  factory BiometricResult.fromMap(Map map) {
    return map['success'] == true
        ? BiometricResult.success()
        : BiometricResult.error(map['error'] as String?);
  }
}
```

### iOS Side (Swift)

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
      case "getBiometricType":
        handler.getBiometricType(result: result)
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

  func getBiometricType(result: FlutterResult) {
    let ctx = LAContext()
    guard ctx.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil) else {
      result("none"); return
    }
    switch ctx.biometryType {
    case .faceID:  result("face")
    case .touchID: result("fingerprint")
    default:       result("none")
    }
  }
}
```

### Android Side (Kotlin)

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
                    "getBiometricType" -> getBiometricType(result)
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
                    result.success(mapOf("success" to false, "error" to "Authentication failed"))
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

    private fun getBiometricType(result: MethodChannel.Result) {
        val available = BiometricManager.from(this).canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG
        ) == BiometricManager.BIOMETRIC_SUCCESS
        result.success(if (available) "fingerprint" else "none")
    }
}
```

---

## EventChannel: Streaming Data

Location streaming implemented across Dart, Swift, and Kotlin.

### Dart Side

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

// Usage
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

### iOS Side (Swift)

```swift
// ios/Runner/LocationStreamHandler.swift
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
                "altitude": loc.altitude,
                "accuracy": loc.horizontalAccuracy])
  }

  func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
    eventSink?(FlutterError(code: "LOCATION_ERROR",
                            message: error.localizedDescription, details: nil))
  }
}
// Register in AppDelegate: FlutterEventChannel(...).setStreamHandler(LocationStreamHandler())
```

### Android Side (Kotlin)

```kotlin
// android/app/src/main/kotlin/com/example/app/LocationStreamHandler.kt
import android.annotation.SuppressLint
import android.content.Context
import android.location.LocationListener
import android.location.LocationManager
import io.flutter.plugin.common.EventChannel

class LocationStreamHandler(private val context: Context) : EventChannel.StreamHandler {
    private var eventSink: EventChannel.EventSink? = null
    private var locationManager: LocationManager? = null
    private val listener = LocationListener { loc ->
        eventSink?.success(mapOf("latitude" to loc.latitude, "longitude" to loc.longitude,
            "altitude" to loc.altitude, "accuracy" to loc.accuracy.toDouble()))
    }

    @SuppressLint("MissingPermission")
    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        eventSink = events
        locationManager = context.getSystemService(Context.LOCATION_SERVICE) as LocationManager
        locationManager?.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000L, 10f, listener)
    }

    override fun onCancel(arguments: Any?) {
        locationManager?.removeUpdates(listener)
        eventSink = null
    }
}
// Register in MainActivity: EventChannel(...).setStreamHandler(LocationStreamHandler(this))
```

---

## Type-Safe Channels with Pigeon

Pigeon generates type-safe channel code from a single schema, eliminating string-based
method names and manual argument parsing.

### Setup and Schema

```yaml
# pubspec.yaml
dev_dependencies:
  pigeon: ^17.0.0
```

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

### Generate and Use

```bash
flutter pub run pigeon --input pigeons/api.dart
```

```dart
// Dart usage -- generated AuthApi class
final authApi = AuthApi();
final result = await authApi.login(LoginRequest()
  ..email = 'user@example.com'
  ..password = 'password');
if (result.success!) {
  print('Logged in as ${result.user!.name}');
}
```

On the native side, implement the generated protocol (Swift) or interface (Kotlin).
Pigeon handles all serialization and channel setup automatically.

---

## iOS-Specific Configuration

### Info.plist Permissions

The app crashes at runtime if a required usage description key is missing.

```xml
<!-- ios/Runner/Info.plist (key entries inside the top-level <dict>) -->
<key>NSCameraUsageDescription</key>
<string>We need camera access to take photos</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>We need photo library access to select images</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>We need your location to show nearby places</string>
<key>NSLocationAlwaysUsageDescription</key>
<string>We need your location for background tracking</string>
<key>NSMicrophoneUsageDescription</key>
<string>We need microphone access for voice recording</string>
<key>NSHealthShareUsageDescription</key>
<string>We need to read your health data</string>
<key>NSHealthUpdateUsageDescription</key>
<string>We need to update your health data</string>
<key>UIBackgroundModes</key>
<array>
  <string>fetch</string>
  <string>remote-notification</string>
  <string>location</string>
</array>
```

### Entitlements

HealthKit, Apple Pay, Push Notifications, and App Groups require entitlements configured
in Xcode under Signing & Capabilities. These must match the Apple Developer portal.

### CocoaPods

```ruby
# ios/Podfile
platform :ios, '13.0'
ENV['COCOAPODS_DISABLE_STATS'] = 'true'

target 'Runner' do
  use_frameworks!
  use_modular_headers!
  flutter_install_all_ios_pods File.dirname(File.realpath(__FILE__))
  # pod 'Firebase/Analytics'
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.0'
    end
  end
end
```

Clean stuck pods: `cd ios && rm -rf Pods Podfile.lock && pod install && cd .. && flutter clean`

### Common iOS Framework Integrations

**CoreLocation** -- see EventChannel example above. Needs `NSLocationWhenInUseUsageDescription`.

**HealthKit** -- enable entitlement in Xcode; add both `NSHealthShareUsageDescription` and
`NSHealthUpdateUsageDescription` to Info.plist. Access via `HKHealthStore` natively.

**Camera (AVFoundation)** -- needs `NSCameraUsageDescription`:

```swift
import AVFoundation
func requestCameraPermission(result: @escaping FlutterResult) {
  AVCaptureDevice.requestAccess(for: .video) { granted in
    DispatchQueue.main.async { result(granted) }
  }
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
  <!-- ... -->
</manifest>
```

Dangerous permissions (camera, location, contacts) require runtime request on API 23+.

### Gradle Configuration

```gradle
// android/app/build.gradle
plugins {
    id "com.android.application"
    id "kotlin-android"
    id "dev.flutter.flutter-gradle-plugin"
}
android {
    namespace "com.example.app"
    compileSdkVersion 34
    defaultConfig {
        applicationId "com.example.app"
        minSdkVersion 21
        targetSdkVersion 34
        multiDexEnabled true
    }
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                'proguard-rules.pro'
        }
    }
}
dependencies {
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.biometric:biometric:1.1.0'
    implementation 'androidx.work:work-runtime-ktx:2.9.0'
}
```

Clean stuck builds: `cd android && ./gradlew clean && cd .. && flutter clean && flutter pub get`

### Common Android Integrations

**WorkManager** -- deferrable, guaranteed background tasks:

```kotlin
class DataSyncWorker(ctx: Context, params: WorkerParameters) : Worker(ctx, params) {
    override fun doWork(): Result = try { /* sync */ Result.success() }
                                    catch (e: Exception) { Result.retry() }
}

fun scheduleSync(context: Context) {
    val request = PeriodicWorkRequestBuilder<DataSyncWorker>(15, TimeUnit.MINUTES)
        .setConstraints(Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true).build())
        .build()
    WorkManager.getInstance(context)
        .enqueueUniquePeriodicWork("data_sync", ExistingPeriodicWorkPolicy.KEEP, request)
}
```

**Foreground Services** -- declare in manifest with `android:foregroundServiceType`.
Start with `startForeground(id, notification)` inside `onStartCommand`.

**BroadcastReceivers** -- register statically in manifest or dynamically in code for
system events like `ACTION_BOOT_COMPLETED` or `ACTION_BATTERY_LOW`.

---

## Error Handling Across Platform Boundaries

### Dart: Structured error wrapper

```dart
class PlatformResult<T> {
  final T? data;
  final PlatformError? error;
  PlatformResult.success(this.data) : error = null;
  PlatformResult.failure(this.error) : data = null;
  bool get isSuccess => error == null;
}

class PlatformError {
  final String code;
  final String message;
  final dynamic details;
  PlatformError(this.code, this.message, [this.details]);

  factory PlatformError.fromException(PlatformException e) =>
      PlatformError(e.code, e.message ?? 'Unknown error', e.details);
}

Future<PlatformResult<T>> safeInvoke<T>(
    MethodChannel channel, String method, [dynamic args]) async {
  try {
    final result = await channel.invokeMethod<T>(method, args);
    return PlatformResult.success(result);
  } on PlatformException catch (e) {
    return PlatformResult.failure(PlatformError.fromException(e));
  } catch (e) {
    return PlatformResult.failure(PlatformError('UNEXPECTED', e.toString()));
  }
}
```

### Native side patterns

```swift
// iOS (Swift)
result(someValue)                                                  // success
result(FlutterError(code: "PERMISSION_DENIED", message: "...", details: nil))  // error
result(FlutterMethodNotImplemented)                                // not implemented
```

```kotlin
// Android (Kotlin)
result.success(someValue)                              // success
result.error("PERMISSION_DENIED", "...", null)         // error
result.notImplemented()                                // not implemented
```

### Key rules

- Use consistent error codes across iOS and Android so Dart handles them uniformly.
- Always dispatch results on the main thread. iOS: `DispatchQueue.main.async`.
  Android: main executor or `runOnUiThread`.
- Every code path must call `result()` exactly once; an unanswered result leaks the
  Dart future.

---

## Best Practices for API Parity

**1. Single channel name per feature.** Use `com.example.app/biometric` on both
platforms. Dart should not need to know which OS it runs on.

**2. Unified Dart interface.** Define an abstract class; implement once using a single
MethodChannel. Both Swift and Kotlin handle the same method names and argument shapes.

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

**3. Use Pigeon for complex APIs.** When a channel has many methods or structured data,
Pigeon enforces identical signatures on both platforms at compile time.

**4. Match error codes.** Use a shared set on both platforms:

| Code                | Meaning                         |
|---------------------|---------------------------------|
| `INVALID_ARGS`      | Missing or malformed arguments  |
| `PERMISSION_DENIED` | User denied a permission        |
| `NOT_AVAILABLE`     | Feature not supported on device |
| `TIMEOUT`           | Operation timed out             |
| `UNKNOWN`           | Catch-all for unexpected errors |

**5. Batch operations.** Prefer one `processBatch` call over N individual `process` calls
to reduce channel-crossing overhead.

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
types, return types, and error codes. This prevents iOS and Android implementations
from drifting apart.
