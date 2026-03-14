---
name: deploying-flutter-ios
description: iOS deployment guide for Flutter apps. Use when configuring code signing, uploading to TestFlight, submitting to App Store, setting up Fastlane automation, or troubleshooting iOS build issues. Covers certificates, provisioning profiles, Info.plist, entitlements, and privacy manifests.
---

# Deploying Flutter iOS

Complete guide for deploying Flutter apps to the Apple App Store, from code signing setup through App Store submission.

## When to Use This Skill

- Setting up iOS code signing (certificates, provisioning profiles)
- Configuring Xcode project settings for release
- Uploading builds to TestFlight
- Submitting apps to App Store
- Setting up Fastlane for iOS CI/CD
- Troubleshooting code signing or build errors
- Configuring Info.plist permissions and entitlements

## Prerequisites

```bash
# Apple Developer Program ($99/year) required
# Xcode 16+ from Mac App Store

xcode-select --install

# Swift Package Manager (recommended, Flutter 3.24+)
flutter config --enable-swift-package-manager

# CocoaPods (legacy fallback)
sudo gem install cocoapods

flutter doctor
```

## Code Signing

### Automatic Signing (Recommended)

1. Open `ios/Runner.xcworkspace` in Xcode
2. Select Runner target → Signing & Capabilities
3. Check "Automatically manage signing"
4. Select your Team
5. Xcode creates certificates and profiles automatically

### Manual Signing

**Certificate types:**

| Type | Purpose | Validity |
|------|---------|----------|
| Development | Testing on devices | 1 year |
| Distribution | App Store & TestFlight | 1 year (max 3 per account) |

**Create certificates:**
1. Xcode → Settings → Accounts → Manage Certificates → "+" → Apple Distribution
2. Or via https://developer.apple.com/account/resources/certificates

**Provisioning profiles:**

| Profile | Use Case | Devices |
|---------|----------|---------|
| Development | Physical device testing | Specific UDIDs |
| Ad Hoc | External testing (up to 100 devices) | Specific UDIDs |
| App Store | App Store & TestFlight | Unrestricted |

Create at https://developer.apple.com/account/resources/profiles or let Xcode manage automatically.

## Xcode Project Configuration

### General Settings

In Xcode → Runner target → General:
- **Display Name**: Your app's user-facing name
- **Bundle Identifier**: `com.yourcompany.appname` (must match App ID)
- **Version**: `1.0.0` (from `pubspec.yaml`)
- **Build**: `1` (increment each upload)
- **Minimum Deployments**: iOS 16.0+

### Info.plist Permissions

```xml
<!-- ios/Runner/Info.plist — add entries for each permission used -->
<key>NSCameraUsageDescription</key>
<string>This app uses the camera to take photos</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>This app accesses photos for your profile</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>This app uses your location to find nearby gyms</string>

<key>NSMicrophoneUsageDescription</key>
<string>This app records audio for video feedback</string>

<key>NSMotionUsageDescription</key>
<string>This app uses motion data to track exercises</string>

<key>NSHealthShareUsageDescription</key>
<string>This app reads health data to personalize workouts</string>
```

**Rule**: Every permission your app requests MUST have a usage description string. Missing descriptions cause App Store rejection.

### Entitlements and Capabilities

Add capabilities in Xcode → Runner → Signing & Capabilities → "+ Capability":

| Capability | When Needed |
|-----------|-------------|
| Push Notifications | FCM or APNs |
| Sign in with Apple | Apple login |
| Associated Domains | Universal links / deep links |
| App Groups | Sharing data with extensions |
| HealthKit | Health data access |
| Background Modes | Background fetch, audio, location |

Entitlements are stored in `ios/Runner/Runner.entitlements`.

### Privacy Manifest (Required)

Since Spring 2024, all apps need `PrivacyInfo.xcprivacy`:

```xml
<!-- ios/Runner/PrivacyInfo.xcprivacy -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>NSPrivacyTracking</key>
  <false/>
  <key>NSPrivacyTrackingDomains</key>
  <array/>
  <key>NSPrivacyCollectedDataTypes</key>
  <array/>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array><string>C617.1</string></array>
    </dict>
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array><string>CA92.1</string></array>
    </dict>
  </array>
</dict>
</plist>
```

## Building for Release

### Version Management

```yaml
# pubspec.yaml
version: 1.0.0+1
# Format: <semver>+<build-number>
# Increment build number for every TestFlight/App Store upload
```

### Build IPA

```bash
flutter clean
flutter pub get
cd ios && pod install && cd ..

# Build release archive
flutter build ipa --release

# Output:
# build/ios/archive/Runner.xcarchive
# build/ios/ipa/app.ipa
```

### Build via Xcode (Alternative)

1. Open `ios/Runner.xcworkspace`
2. Select "Any iOS Device (arm64)" as target
3. Product → Archive
4. Organizer opens → Select archive → Distribute App
5. Choose "App Store Connect" → Upload

## App Store Connect

### Create App

1. Login at https://appstoreconnect.apple.com
2. My Apps → "+" → New App
3. Fill: Name, Primary Language, Bundle ID, SKU

### Required Metadata

| Field | Limit | Notes |
|-------|-------|-------|
| App Name | 30 chars | |
| Subtitle | 30 chars | |
| Description | 4000 chars | |
| Keywords | 100 chars | Comma-separated |
| Support URL | Required | |
| Privacy Policy URL | Required | |
| Copyright | Required | e.g. "2025 Your Company" |

### Screenshots (Required Sizes)

| Device | Resolution |
|--------|-----------|
| 6.9" (iPhone 16 Pro Max) | 1320 x 2868 px |
| 6.7" (iPhone 15 Pro Max) | 1290 x 2796 px |
| 6.5" (iPhone 11 Pro Max) | 1242 x 2688 px |
| 5.5" (iPhone 8 Plus) | 1242 x 2208 px |
| iPad Pro 13" (M4) | 2064 x 2752 px |

Upload 3-10 screenshots per device size.

## TestFlight

### Internal Testing
- Up to 100 testers (must have App Store Connect access)
- No review required, immediate access
- Add testers: App Store Connect → TestFlight → Internal Testing

### External Testing
- Up to 10,000 testers per group
- Requires Beta App Review (1-2 days)
- Add testers by email
- Provide: Beta description, feedback email, "What to Test"

### Upload Flow

```bash
# Build and upload
flutter build ipa --release
# Then upload via Xcode Organizer or Fastlane (see below)
```

## Fastlane Automation

### Setup

```bash
sudo gem install fastlane
cd ios
fastlane init
# Choose option 2: Automate TestFlight distribution
```

### Fastfile

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  lane :beta do
    increment_build_number(xcodeproj: "Runner.xcodeproj")

    build_app(
      scheme: "Runner",
      export_method: "app-store",
      output_directory: "./build/Runner",
    )

    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      distribute_external: true,
      groups: ["Beta Testers"],
      changelog: "Bug fixes and improvements",
    )
  end

  lane :release do
    ensure_git_status_clean

    increment_build_number(xcodeproj: "Runner.xcodeproj")

    build_app(scheme: "Runner", export_method: "app-store")

    upload_to_app_store(
      force: true,
      submit_for_review: false,
      automatic_release: false,
    )

    commit_version_bump(
      message: "Version Bump",
      xcodeproj: "Runner.xcodeproj",
    )
    add_git_tag(tag: "v#{get_version_number(xcodeproj: 'Runner.xcodeproj')}")
    push_to_git_remote
  end

  lane :certificates do
    match(type: "appstore", app_identifier: "com.yourcompany.appname", readonly: true)
  end
end
```

### Environment Variables

```bash
# .env (add to .gitignore)
FASTLANE_USER="your@email.com"
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD="xxxx-xxxx-xxxx-xxxx"
MATCH_PASSWORD="your-match-encryption-password"
```

Create app-specific password at https://appleid.apple.com → Security.

### Run Fastlane

```bash
cd ios
fastlane beta      # Upload to TestFlight
fastlane release   # Submit to App Store
```

## CI/CD with GitHub Actions

```yaml
# .github/workflows/ios-deploy.yml
name: iOS Deploy
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.41.0'

      - run: flutter pub get
      - run: flutter test

      - name: Install certificates
        env:
          P12_BASE64: ${{ secrets.P12_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          PROVISION_PROFILE_BASE64: ${{ secrets.PROVISIONING_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          # Create temporary keychain
          security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          security default-keychain -s build.keychain
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain

          # Import certificate
          echo "$P12_BASE64" | base64 --decode > certificate.p12
          security import certificate.p12 -k build.keychain -P "$P12_PASSWORD" -T /usr/bin/codesign
          security set-key-partition-list -S apple-tool:,apple: -s -k "$KEYCHAIN_PASSWORD" build.keychain

          # Install provisioning profile
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          echo "$PROVISION_PROFILE_BASE64" | base64 --decode > ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision

      - name: Build IPA
        run: flutter build ipa --release --export-options-plist=ios/ExportOptions.plist

      - name: Upload to TestFlight
        run: |
          gem install fastlane
          cd ios && fastlane beta
        env:
          FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
          FASTLANE_USER: ${{ secrets.APPLE_ID }}

      - name: Cleanup keychain
        if: always()
        run: security delete-keychain build.keychain
```

### Encode Secrets for CI

```bash
# Certificate (.p12)
base64 -i Certificates.p12 | pbcopy
# Paste into GitHub Secrets as P12_CERTIFICATE_BASE64

# Provisioning profile
base64 -i profile.mobileprovision | pbcopy
# Paste into GitHub Secrets as PROVISIONING_PROFILE_BASE64
```

**Tip:** For simpler setup, use `fastlane match` to manage certificates in a private Git repo, avoiding manual Base64 encoding.

## Pre-Submission Checklist

- [ ] Version and build number updated in `pubspec.yaml`
- [ ] App icons generated (all sizes)
- [ ] Launch screen configured
- [ ] Info.plist permissions have descriptive strings
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) present
- [ ] Required capabilities enabled in entitlements
- [ ] Tested on physical device
- [ ] TestFlight beta testing complete
- [ ] All App Store Connect metadata filled
- [ ] Screenshots uploaded for all required device sizes
- [ ] Privacy policy URL configured
- [ ] Content rating questionnaire completed
- [ ] Export compliance answered

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "No signing certificate found" | Xcode → Settings → Accounts → Download Manual Profiles, or create new certificate |
| "Provisioning profile doesn't include signing certificate" | Toggle Automatic Signing off/on, or regenerate profile |
| "Invalid entitlements" | Verify Runner.entitlements matches App ID capabilities, then `flutter clean` |
| CocoaPods module errors | `cd ios && pod deintegrate && pod install && cd .. && flutter clean` |
| Archive fails in Xcode | Target must be "Any iOS Device"; clean build folder (Cmd+Shift+K); delete DerivedData |
| Missing privacy manifest | Create `PrivacyInfo.xcprivacy` and declare required API reasons |
| App Store rejection | Read rejection message carefully; fix issues; increment build number; resubmit |
