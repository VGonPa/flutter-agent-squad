---
name: deploying-flutter-android
description: Android deployment guide for Flutter apps. Use when creating keystores, configuring signing in build.gradle, uploading to Google Play Console, setting up internal/beta/production tracks, configuring ProGuard/R8, or automating with Fastlane. Covers app bundles, staged rollouts, and Play App Signing.
---

# Deploying Flutter Android

Complete guide for deploying Flutter apps to the Google Play Store, from keystore creation through production rollout.

## When to Use This Skill

- Creating upload keystores and configuring signing
- Configuring build.gradle for release builds
- Setting up Google Play Console and release tracks
- Configuring ProGuard/R8 rules
- Building app bundles (AAB) or APKs
- Setting up Fastlane for Android CI/CD
- Troubleshooting signing or build errors

## Prerequisites

```bash
# Google Play Console account ($25 one-time fee)
# https://play.google.com/console/signup
# Processing time: ~48 hours for activation

# Android Studio (for SDK and tools)
flutter doctor
flutter doctor --android-licenses
```

## Keystore and Signing

### Create Upload Keystore

```bash
keytool -genkey -v -keystore ~/upload-keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias upload

# CRITICAL: Back up this file securely. Loss = cannot update app.
# Record passwords in a password manager.
```

### Configure Signing

```properties
# android/key.properties (CREATE THIS FILE — add to .gitignore)
storePassword=your-keystore-password
keyPassword=your-key-password
keyAlias=upload
storeFile=/Users/yourname/upload-keystore.jks
```

```groovy
// android/app/build.gradle

// Load keystore properties
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    namespace "com.yourcompany.appname"
    compileSdk 35

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = '17'
    }

    defaultConfig {
        applicationId "com.yourcompany.appname"
        minSdk 21
        targetSdk 35  // Required for new Play Store submissions (Aug 2025+)
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
        multiDexEnabled true
    }

    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ?
                file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile(
                'proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

### Product Flavors (Optional)

```groovy
android {
    flavorDimensions "environment"
    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"
            versionNameSuffix "-dev"
        }
        staging {
            dimension "environment"
            applicationIdSuffix ".staging"
            versionNameSuffix "-staging"
        }
        prod {
            dimension "environment"
        }
    }
}
```

```bash
# Build specific flavor
flutter build appbundle --flavor prod --release
```

## ProGuard / R8 Configuration

```proguard
# android/app/proguard-rules.pro

# Flutter
-keep class io.flutter.app.** { *; }
-keep class io.flutter.plugin.** { *; }
-keep class io.flutter.util.** { *; }
-keep class io.flutter.view.** { *; }
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }

# Keep application class
-keep class com.yourcompany.yourapp.** { *; }

# Firebase (if used)
-keep class com.google.firebase.** { *; }
-dontwarn com.google.firebase.**

# Gson (if used)
-keepattributes Signature
-keepattributes *Annotation*
-keep class com.google.gson.** { *; }

# Native methods
-keepclasseswithmembernames class * {
    native <methods>;
}
```

**Debugging R8 issues:**
```groovy
// Temporarily disable to isolate problems
buildTypes {
    release {
        minifyEnabled false  // Disable temporarily
        shrinkResources false
    }
}
```

## Building for Release

### Version Management

```yaml
# pubspec.yaml
version: 1.0.0+1
# version: User-facing version string
# +1: versionCode (must increment for every Play Store upload)
```

### Build App Bundle (Recommended)

```bash
flutter clean
flutter pub get

# Build AAB (preferred by Google Play)
flutter build appbundle --release

# Output: build/app/outputs/bundle/release/app-release.aab
```

### Build APK (Alternative)

```bash
# Single APK
flutter build apk --release

# Split by ABI (smaller per-device downloads)
flutter build apk --release --split-per-abi
# Outputs: app-armeabi-v7a-release.apk
#          app-arm64-v8a-release.apk
#          app-x86_64-release.apk
```

### 16KB Page Size Alignment

Since November 2025, Google Play requires 16KB page size support:
- Use NDK r28+, AGP 8.5.1+, Gradle 8.14+
- Flutter 3.22+ handles this automatically with recommended versions

## Google Play Console

### Create App

1. https://play.google.com/console → Create app
2. Fill: App name, language, app/game, free/paid
3. Accept Developer Program Policies

### Store Listing Metadata

| Field | Limit |
|-------|-------|
| App name | 30 chars |
| Short description | 80 chars |
| Full description | 4000 chars |
| App icon | 512 x 512 px, PNG |
| Feature graphic | 1024 x 500 px |
| Phone screenshots | 2-8, min 320px, max 3840px |

### Data Safety Section (Required)

Complete the data safety form thoroughly:
- Declare all data types collected (including by SDKs)
- Declare data sharing with third parties
- Declare encryption and deletion practices
- Privacy policy URL required
- Non-compliance can lead to app removal

### Release Tracks

| Track | Purpose | Review |
|-------|---------|--------|
| Internal testing | Team testing, up to 100 testers | No review |
| Closed testing | Invite-only beta, tester lists | Optional |
| Open testing | Anyone can join (up to 200K) | Required |
| Production | Public release | Required |

### Staged Rollout Strategy

```
Internal → Closed Beta → Production (10%) → 50% → 100%
```

Monitor crash rate and ANR rate at each stage before expanding.

## Play App Signing

Play App Signing is the default for new apps and required for AABs:

- Google manages app signing key in Cloud KMS
- You sign uploads with your upload key only
- Lost upload key can be reset via Play Console
- Smaller APKs (Google generates optimized splits)

### Export Signing Certificate

```bash
# Get SHA fingerprints for Firebase, Maps, etc.
keytool -list -v -keystore ~/upload-keystore.jks -alias upload

# For Play App Signing, also get the app signing cert from:
# Play Console → Setup → App Integrity → App signing tab
```

## Fastlane Automation

### Setup

```bash
sudo gem install fastlane
cd android
fastlane init
```

### Service Account for API Access

1. Google Cloud Console → Enable "Google Play Android Developer API"
2. IAM & Admin → Service Accounts → Create
3. Create JSON key → Download as `api-key.json`
4. Play Console → Users and permissions → Invite service account email → Grant "Release manager"

### Fastfile

```ruby
# android/fastlane/Fastfile
default_platform(:android)

platform :android do
  lane :internal do
    gradle(task: "bundle", build_type: "Release")

    upload_to_play_store(
      track: 'internal',
      aab: '../build/app/outputs/bundle/release/app-release.aab',
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )
  end

  lane :beta do
    gradle(task: "bundle", build_type: "Release")

    upload_to_play_store(
      track: 'beta',
      aab: '../build/app/outputs/bundle/release/app-release.aab',
      release_status: 'draft',
    )
  end

  lane :production do
    ensure_git_status_clean

    gradle(task: "bundle", build_type: "Release")

    upload_to_play_store(
      track: 'production',
      aab: '../build/app/outputs/bundle/release/app-release.aab',
      release_status: 'draft',
      rollout: '0.1',  # 10% staged rollout
    )

    git_commit(path: "app/build.gradle", message: "Version Bump")
    add_git_tag(tag: "android/v#{get_version_name(
      gradle_file_path: 'app/build.gradle')}")
    push_to_git_remote
  end

  lane :promote do |options|
    percentage = options[:percentage] || '0.5'
    upload_to_play_store(
      track: 'production',
      rollout: percentage,
      skip_upload_apk: true,
      skip_upload_aab: true,
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )
  end
end
```

### Run Fastlane

```bash
cd android
fastlane internal                   # Internal testing
fastlane beta                       # Closed beta
fastlane production                 # Production (10% rollout)
fastlane promote percentage:0.5     # Expand to 50%
fastlane promote percentage:1.0     # Full rollout
```

## CI/CD with GitHub Actions

```yaml
# .github/workflows/android-deploy.yml
name: Android Deploy
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: '17'
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.41.0'

      - run: flutter pub get
      - run: flutter test

      - name: Decode keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 --decode > android/upload-keystore.jks

      - name: Create key.properties
        run: |
          cat > android/key.properties <<EOF
          storePassword=${{ secrets.STORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=../upload-keystore.jks
          EOF

      - run: flutter build appbundle --release

      - name: Deploy to Play Store
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
      - run: gem install fastlane
      - run: cd android && fastlane internal
        env:
          SUPPLY_JSON_KEY_DATA: ${{ secrets.PLAY_STORE_CONFIG_JSON }}
```

### Encode Keystore for CI

```bash
base64 -i upload-keystore.jks | pbcopy
# Paste into GitHub Secrets as KEYSTORE_BASE64
```

## Pre-Submission Checklist

- [ ] Version and versionCode updated in `pubspec.yaml`
- [ ] `key.properties` configured and in `.gitignore`
- [ ] Signing config in `build.gradle` correct
- [ ] ProGuard rules cover all required classes
- [ ] App icons generated (all densities)
- [ ] AAB builds successfully with `flutter build appbundle --release`
- [ ] Tested on physical device
- [ ] Data safety section complete in Play Console
- [ ] Privacy policy URL configured
- [ ] Content rating questionnaire complete
- [ ] Store listing metadata and screenshots uploaded
- [ ] Target SDK meets Play Store requirements (35+)

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Failed to sign APK" | Verify `key.properties` paths and passwords; test keystore with `keytool -list -v` |
| "App not properly signed" | `flutter clean && flutter build appbundle --release`; check signingConfig |
| R8 minification errors | Add `-keep` rules to `proguard-rules.pro`; temporarily set `minifyEnabled false` |
| "Duplicate class found" | Exclude conflicting transitive dependencies in `build.gradle` |
| "Version code already used" | Increment `+N` in `pubspec.yaml` version |
| Policy violation | Review email; fix data safety, privacy policy, or content rating |
| "Cannot rollout — no upgrade path" | Ensure new versionCode > previous; check targetSdk requirements |
