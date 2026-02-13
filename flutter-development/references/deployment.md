# Flutter Deployment Reference

Consolidated deployment procedures for iOS (App Store) and Android (Google Play Store).

## Table of Contents

- [iOS Deployment](#ios-deployment)
  - [Prerequisites](#ios-prerequisites)
  - [Code Signing](#code-signing)
  - [Build for Release](#ios-build-for-release)
  - [TestFlight](#testflight-beta-testing)
  - [App Store Connect](#app-store-connect)
  - [Fastlane Automation (iOS)](#fastlane-automation-ios)
  - [Common Issues (iOS)](#common-issues-ios)
- [Android Deployment](#android-deployment)
  - [Prerequisites](#android-prerequisites)
  - [App Signing](#app-signing)
  - [ProGuard / R8 Configuration](#proguard--r8-configuration)
  - [Build for Release](#android-build-for-release)
  - [Google Play Console](#google-play-console)
  - [Internal Testing and Staged Rollout](#internal-testing-and-staged-rollout)
  - [Fastlane Automation (Android)](#fastlane-automation-android)
  - [Common Issues (Android)](#common-issues-android)

---

## iOS Deployment

### iOS Prerequisites

**Apple Developer Program** -- $99/year at https://developer.apple.com/programs/. Account types: Individual or Organization.

```bash
xcode-select --install          # Xcode Command Line Tools
sudo gem install cocoapods      # CocoaPods
flutter doctor                  # Verify iOS setup
```

### Code Signing

**Certificates:** Development (testing, 1 yr) and Distribution (App Store/TestFlight, 1 yr, max 3). Create via Xcode (Preferences > Accounts > Manage Certificates > "+" > Apple Distribution) or the Developer Portal (generate CSR in Keychain Access, upload, download and install).

**Bundle Identifier:** Register at https://developer.apple.com/account/resources/identifiers. Use explicit format `com.yourcompany.appname`. Enable needed capabilities. Must match `ios/Runner.xcodeproj`.

**Provisioning Profiles:**

| Type | Purpose | Devices |
|---|---|---|
| Development | Physical device testing | Specific UDIDs |
| Ad Hoc | Outside App Store | Up to 100 |
| App Store | Store + TestFlight | None |

Automatic (recommended): open `ios/Runner.xcworkspace`, Runner target > Signing & Capabilities > check "Automatically manage signing" > select Team.

Manual: create at https://developer.apple.com/account/resources/profiles, select App Store type, choose App ID and Distribution Certificate, download and install.

### iOS Build for Release

```yaml
# pubspec.yaml
version: 1.0.0+1    # <version>+<build-number>
```

```bash
flutter clean
flutter pub get
cd ios && pod install && cd ..
flutter build ipa --release
# Archive: build/ios/archive/Runner.xcarchive
# IPA:     build/ios/ipa/app.ipa
```

Via Xcode: `cd ios && open Runner.xcworkspace`, select "Any iOS Device (arm64)", Product > Archive, then Distribute App > App Store Connect > Upload.

### TestFlight Beta Testing

Upload the IPA via Xcode Organizer or Fastlane, then configure in App Store Connect > TestFlight:

- **Internal** (up to 100): testers need App Store Connect roles, no review, immediate access.
- **External** (up to 10,000/group): add by email, requires Beta App Review (1-2 days). Provide beta description, feedback email, and test instructions.

Export compliance: declare whether encryption is used.

### App Store Connect

Create app at https://appstoreconnect.apple.com > My Apps > "+" > New App. Set platform, name, language, Bundle ID, SKU.

**Required metadata:** app name (30 chars), subtitle (30 chars), description (4000 chars), keywords (100 chars), support URL, privacy policy URL, copyright, age rating, content rights.

**Required screenshots:**

| Device | Resolution |
|---|---|
| 6.7" iPhone 15 Pro Max | 1290 x 2796 px |
| 6.5" iPhone 11 Pro Max | 1242 x 2688 px |
| 5.5" iPhone 8 Plus | 1242 x 2208 px |
| iPad Pro 12.9" | 2048 x 2732 px |

3-10 screenshots per size. Submit for Review after completing all fields. Typical timeline: Waiting (1-3 days) > In Review (1-2 days) > Approved or Rejected.

### Fastlane Automation (iOS)

```bash
sudo gem install fastlane
cd ios && fastlane init    # Choose option 2 for TestFlight
```

**`ios/fastlane/Fastfile`:**

```ruby
default_platform(:ios)

platform :ios do
  lane :beta do
    increment_build_number(xcodeproj: "Runner.xcodeproj")
    build_app(scheme: "Runner", export_method: "app-store",
              output_directory: "./build/Runner", output_name: "Runner.ipa")
    upload_to_testflight(skip_waiting_for_build_processing: true,
                         distribute_external: true, groups: ["Beta Testers"],
                         changelog: "Bug fixes and improvements")
  end

  lane :release do
    ensure_git_status_clean
    increment_build_number(xcodeproj: "Runner.xcodeproj")
    build_app(scheme: "Runner", export_method: "app-store")
    upload_to_app_store(force: true, skip_metadata: false,
                        skip_screenshots: false, submit_for_review: false)
    commit_version_bump(message: "Version Bump", xcodeproj: "Runner.xcodeproj")
    add_git_tag(tag: "v#{get_version_number(xcodeproj: 'Runner.xcodeproj')}")
    push_to_git_remote
  end

  lane :certificates do
    match(type: "appstore", app_identifier: "com.yourcompany.appname", readonly: true)
  end
end
```

```bash
cd ios
fastlane beta              # Upload to TestFlight
fastlane release           # Upload to App Store (draft)
fastlane certificates      # Sync signing certificates
```

**Environment variables** (`ios/fastlane/.env`, add to `.gitignore`):

```bash
FASTLANE_USER="your@email.com"
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD="app-specific-password"
MATCH_PASSWORD="your-match-encryption-password"
```

Create app-specific password at https://appleid.apple.com > Security > App-Specific Passwords.

### Common Issues (iOS)

**"No signing certificate found"** -- Xcode > Preferences > Accounts > Download Manual Profiles, or create a new distribution certificate. Verify it is in Keychain Access.

**"Provisioning profile doesn't include signing certificate"** -- delete old profiles, toggle automatic signing off/on in Xcode, or regenerate in Developer Portal.

**"The executable was signed with invalid entitlements"** -- ensure `Runner.entitlements` matches App ID capabilities. Run `flutter clean` and rebuild.

**CocoaPods / "Module not found":**

```bash
cd ios && pod deintegrate && pod install && cd ..
flutter clean && flutter pub get && flutter build ios
```

**Archive fails:** set target to "Any iOS Device", clean build folder (Cmd+Shift+K), delete DerivedData: `rm -rf ~/Library/Developer/Xcode/DerivedData`.

**Common rejection reasons:** missing permission descriptions in `Info.plist`, incomplete screenshots/metadata, crashes on physical devices.

---

## Android Deployment

### Android Prerequisites

**Google Play Console** -- one-time $25 at https://play.google.com/console/signup. Activation takes ~48 hours. Account types: Personal or Organization (needs D-U-N-S number).

```bash
flutter doctor                     # Verify Android setup
flutter doctor --android-licenses  # Accept SDK licenses
```

### App Signing

**Create upload keystore:**

```bash
keytool -genkey -v -keystore ~/upload-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias upload
# Prompts for passwords, name, org, country. BACK UP this file securely.
```

**Create `android/key.properties`** (add to `.gitignore`):

```properties
storePassword=your-keystore-password
keyPassword=your-key-password
keyAlias=upload
storeFile=/Users/yourname/upload-keystore.jks
```

**Add signing config to `android/app/build.gradle`:**

```groovy
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

**Play App Signing:** enable in Play Console > Release > Setup > App Integrity. Google manages the signing key; you use the upload keystore. Benefits: optimized APKs, recoverable upload key.

```bash
# Export certificate fingerprints for Firebase, Google Maps, etc.
keytool -list -v -keystore ~/upload-keystore.jks -alias upload
```

### ProGuard / R8 Configuration

Create `android/app/proguard-rules.pro`:

```proguard
-keep class io.flutter.app.** { *; }
-keep class io.flutter.plugin.** { *; }
-keep class io.flutter.util.** { *; }
-keep class io.flutter.view.** { *; }
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }
-keep class com.yourcompany.yourapp.** { *; }

# Gson (if used)
-keepattributes Signature
-keepattributes *Annotation*
-keep class com.google.gson.** { *; }

# Firebase (if used)
-keep class com.google.firebase.** { *; }
-dontwarn com.google.firebase.**

# OkHttp / Retrofit (if used)
-dontwarn okhttp3.**
-dontwarn retrofit2.**
-keep class okhttp3.** { *; }
-keep interface okhttp3.** { *; }

-keepclasseswithmembernames class * { native <methods>; }
```

### Android Build for Release

```yaml
# pubspec.yaml
version: 1.0.0+1    # <version>+<build-number>
```

```bash
flutter clean && flutter pub get

# App Bundle (recommended for Play Store)
flutter build appbundle --release
# Output: build/app/outputs/bundle/release/app-release.aab

# APK (alternative)
flutter build apk --release
# Output: build/app/outputs/flutter-apk/app-release.apk

# Split APKs by ABI
flutter build apk --release --split-per-abi
# Outputs: app-armeabi-v7a-release.apk, app-arm64-v8a-release.apk, app-x86_64-release.apk
```

### Google Play Console

Create app at https://play.google.com/console > Create app. Set name, language, type, pricing. Accept policies.

**Setup checklist:** app access, ads declaration, content rating, target audience, data safety (privacy policy URL required).

**Store listing:**

| Field | Limit |
|---|---|
| App name | 30 characters |
| Short description | 80 characters |
| Full description | 4000 characters |
| App icon | 512 x 512 px PNG |
| Feature graphic | 1024 x 500 px |
| Phone screenshots | 2-8, 320-3840 px |

### Internal Testing and Staged Rollout

| Track | Audience | Review |
|---|---|---|
| Internal testing | Up to 100 by email | No |
| Closed testing | Invite-only, up to 100 lists | Optional |
| Open testing | Anyone, up to 200k | Yes |
| Production | All users | Yes |

**Internal testing:** create release, upload AAB, add testers by email, share opt-in URL.

**Staged rollout:** set initial percentage (e.g., 10%), monitor crashes/ANRs, increase gradually (10% > 50% > 100%).

### Fastlane Automation (Android)

```bash
sudo gem install fastlane
cd android && fastlane init
```

**Service account setup:** enable Google Play Android Developer API in Cloud Console, create service account with JSON key, invite its email in Play Console with Release Manager permissions.

**`android/fastlane/Fastfile`:**

```ruby
default_platform(:android)

platform :android do
  lane :internal do
    gradle(task: "bundle", build_type: "Release")
    upload_to_play_store(track: 'internal',
      aab: '../build/app/outputs/bundle/release/app-release.aab',
      skip_upload_apk: true, skip_upload_metadata: true,
      skip_upload_images: true, skip_upload_screenshots: true)
  end

  lane :beta do
    gradle(task: "bundle", build_type: "Release")
    upload_to_play_store(track: 'beta',
      aab: '../build/app/outputs/bundle/release/app-release.aab',
      release_status: 'draft')
  end

  lane :production do
    ensure_git_status_clean
    gradle(task: "bundle", build_type: "Release")
    upload_to_play_store(track: 'production',
      aab: '../build/app/outputs/bundle/release/app-release.aab',
      release_status: 'draft', rollout: '0.1')
    git_commit(path: "app/build.gradle", message: "Version Bump")
    add_git_tag(tag: "android/v#{get_version_name(gradle_file_path: 'app/build.gradle')}")
    push_to_git_remote
  end

  lane :promote_to_production do
    upload_to_play_store(track: 'beta', track_promote_to: 'production', rollout: '0.1')
  end

  lane :increase_rollout do |options|
    upload_to_play_store(track: 'production', rollout: (options[:percentage] || 0.5).to_s,
      skip_upload_apk: true, skip_upload_aab: true, skip_upload_metadata: true,
      skip_upload_images: true, skip_upload_screenshots: true)
  end
end
```

```bash
cd android
fastlane internal                         # Internal testing
fastlane beta                             # Closed beta (draft)
fastlane production                       # Production (10% rollout draft)
fastlane promote_to_production            # Promote beta to production
fastlane increase_rollout percentage:0.5  # Increase rollout to 50%
```

### Common Issues (Android)

**"Failed to sign APK"** -- verify `android/key.properties` paths and passwords. Test: `keytool -list -v -keystore ~/upload-keystore.jks`.

**"App not properly signed":**

```bash
flutter clean && flutter pub get && flutter build appbundle --release
```

Verify `signingConfig signingConfigs.release` is set in the release build type.

**"Execution failed for task ':app:minifyReleaseWithR8'"** -- check `proguard-rules.pro`, add `-keep class your.problematic.Class { *; }`, or temporarily set `minifyEnabled false` to isolate.

**"Duplicate class found"** -- exclude conflicting transitive dependency:

```groovy
implementation('some.library') {
    exclude group: 'duplicate.group', module: 'duplicate-module'
}
```

**"Version code already used"** -- increment the `+N` build number in `pubspec.yaml` and rebuild.

**"Not compliant with Google Play policy"** -- check violation email. Common causes: missing privacy policy, incomplete data safety section, incomplete content rating.

**"Cannot rollout -- does not allow existing users to upgrade"** -- ensure new `versionCode` exceeds previous release and `targetSdkVersion` meets Play Store requirements (33+).
