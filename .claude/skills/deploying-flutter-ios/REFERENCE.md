# Deploying Flutter iOS — Reference Templates

Copy-paste templates for iOS deployment. See `SKILL.md` for decision guidance on when and why to use each.

## PrivacyInfo.xcprivacy Template

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

## Info.plist Common Permissions

```xml
<!-- ios/Runner/Info.plist — add only the permissions your app actually uses -->
<key>NSCameraUsageDescription</key>
<string>Take photos for your workout journal</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>Choose a profile photo from your library</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>Find nearby gyms and training facilities</string>

<key>NSMicrophoneUsageDescription</key>
<string>Record audio for video feedback sessions</string>

<key>NSMotionUsageDescription</key>
<string>Track movement data during exercises</string>

<key>NSHealthShareUsageDescription</key>
<string>Read health data to personalize your training plan</string>

<key>NSFaceIDUsageDescription</key>
<string>Securely sign in with Face ID</string>
```

## Fastlane Fastfile

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

## Fastlane Environment Variables

```bash
# ios/fastlane/.env (add to .gitignore)
FASTLANE_USER="your@email.com"
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD="xxxx-xxxx-xxxx-xxxx"
MATCH_PASSWORD="your-match-encryption-password"
```

Create app-specific password at https://appleid.apple.com → Security.

## Fastlane Setup Commands

```bash
sudo gem install fastlane
cd ios
fastlane init
# Choose option 2: Automate TestFlight distribution

# Run lanes
fastlane beta      # Upload to TestFlight
fastlane release   # Submit to App Store
```

## CI/CD: GitHub Actions Workflow

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
          # Create temporary keychain (CI machines don't have a persistent one)
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

## Encode Secrets for CI

```bash
# Certificate (.p12)
base64 -i Certificates.p12 | pbcopy
# Paste into GitHub Secrets as P12_CERTIFICATE_BASE64

# Provisioning profile
base64 -i profile.mobileprovision | pbcopy
# Paste into GitHub Secrets as PROVISIONING_PROFILE_BASE64
```

**Tip:** For simpler CI setup, use `fastlane match` to manage certificates in a private Git repo — avoids manual Base64 encoding entirely.
