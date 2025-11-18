---
name: devops-deployment
description: Master Flutter DevOps - CI/CD pipelines (GitHub Actions, GitLab CI, Fastlane), code signing, iOS/Android builds, App Store and Google Play submission, beta testing, monitoring, analytics and automated releases.
---

# DevOps & Deployment Automation

## Quick Start - GitHub Actions Pipeline

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
          channel: 'stable'

      - name: Get dependencies
        run: flutter pub get

      - name: Run tests
        run: flutter test

      - name: Build APK
        run: flutter build apk --release

      - name: Upload to Google Play
        run: |
          echo "${{ secrets.PLAY_STORE_KEY }}" > key.json
          bundle exec fastlane android deploy

      - name: Create release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
```

## Android Build & Signing

```gradle
// android/app/build.gradle

android {
  signingConfigs {
    release {
      keyAlias System.getenv("KEYSTORE_ALIAS")
      keyPassword System.getenv("KEYSTORE_PASSWORD")
      storeFile file(System.getenv("KEYSTORE_PATH") ?: "key.jks")
      storePassword System.getenv("KEYSTORE_STORE_PASSWORD")
    }
  }

  buildTypes {
    release {
      signingConfig signingConfigs.release
      minifyEnabled true
      shrinkResources true

      proguardFiles getDefaultProguardFile(
        'proguard-android-optimize.txt'
      ), 'proguard-rules.pro'
    }
  }
}

// Build APK with signing
// flutter build apk --release

// Build App Bundle for Play Store
// flutter build appbundle --release
```

## iOS Build & Signing

```bash
# Export iOS app for App Store
flutter build ios --release --no-codesign

# Sign with Fastlane
bundle exec fastlane ios build_and_upload

# Or manually with Xcode
xcodebuild -workspace ios/Runner.xcworkspace \
  -scheme Runner \
  -configuration Release \
  -derivedDataPath build \
  -archivePath build/Runner.xcarchive \
  -allowProvisioningUpdates \
  archive

xcodebuild -exportArchive \
  -archivePath build/Runner.xcarchive \
  -exportOptionsPlist ios/ExportOptions.plist \
  -exportPath build/ipa
```

## Fastlane Automation

```bash
# Install Fastlane
sudo gem install fastlane -NV

# Initialize for Android
cd android
fastlane init
```

```swift
// android/fastlane/Fastfile

default_platform(:android)

platform :android do
  desc "Build and upload APK to Google Play"
  lane :deploy do
    build_android_app(
      task: "assemble",
      build_type: "Release",
      project_dir: "android/",
      gradle_path: "android/gradlew",
    )

    upload_to_play_store(
      track: "internal",
      aab: "android/app/build/outputs/bundle/release/app-release.aab",
      json_key: ENV["PLAY_STORE_JSON_KEY"],
    )
  end

  desc "Beta release to Firebase Testing Lab"
  lane :beta do
    build_android_app(
      task: "assemble",
      build_type: "Debug",
      project_dir: "android/",
    )

    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID"],
      groups: "beta-testers",
      release_notes: "Beta release",
      android_artifact_path: "android/app/build/outputs/apk/debug/app-debug.apk",
    )
  end
end

# Usage
# fastlane android deploy
# fastlane android beta
```

## App Store Connect Setup

```swift
// ios/fastlane/Fastfile

platform :ios do
  desc "Build and upload to App Store"
  lane :deploy do
    build_ios_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      configuration: "Release",
      export_method: "app-store",
      export_options: {
        provisioningProfileSpecifier: "App Store",
        signingStyle: "automatic",
      },
    )

    upload_to_app_store(
      api_key: app_store_connect_api_key(
        key_id: ENV["APP_STORE_KEY_ID"],
        issuer_id: ENV["APP_STORE_ISSUER_ID"],
        key_content: ENV["APP_STORE_KEY_CONTENT"],
      ),
      skip_screenshots: true,
      skip_metadata: true,
      force: true,
    )
  end

  desc "TestFlight beta release"
  lane :beta do
    build_ios_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      configuration: "Release",
      export_method: "app-store",
    )

    upload_to_testflight(
      api_key: app_store_connect_api_key(...),
      skip_waiting_for_build_processing: true,
    )
  end
end

# Usage
# fastlane ios deploy
# fastlane ios beta
```

## Google Play Console Setup

```dart
// pubspec.yaml - Version management
version: 1.0.0+1

// Format: 1.0.0+BUILD_NUMBER
// 1.0.0 = semantic version
// 1 = build number (increment for each release)

// In code
import 'package:package_info_plus/package_info_plus.dart';

final packageInfo = await PackageInfo.fromPlatform();
print('Version: ${packageInfo.version}');
print('Build: ${packageInfo.buildNumber}');
```

## CI/CD with GitHub Actions

```yaml
# .github/workflows/build.yml
name: Flutter Build

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter test
      - run: flutter analyze

  build-apk:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter build apk --release
      - uses: actions/upload-artifact@v3
        with:
          name: app-release.apk
          path: build/app/outputs/apk/release/

  build-ios:
    runs-on: macos-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter build ios --release --no-codesign
      - uses: actions/upload-artifact@v3
        with:
          name: Runner.zip
          path: build/ios/Release-iphoneos/
```

## Release Management

```bash
# Bump version
# pubspec.yaml: 1.0.0+1 → 1.0.1+2

# Generate changelog
# Update CHANGELOG.md with new features/fixes

# Create git tag
git tag -a v1.0.1 -m "Release version 1.0.1"
git push origin v1.0.1

# GitHub Actions triggers on tag and builds/releases automatically
```

## Analytics & Crash Reporting

```dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';

// Initialize Firebase
FirebaseAnalytics.instance.logAppOpen();
FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(true);

// Track custom events
FirebaseAnalytics.instance.logEvent(
  name: 'user_signup',
  parameters: {
    'method': 'email',
  },
);

// Log non-fatal errors
try {
  riskyOperation();
} catch (e, stackTrace) {
  FirebaseCrashlytics.instance.recordError(
    e,
    stackTrace,
    fatal: false,
  );
}

// Crash reporting dashboard
// Firebase Console → Crashlytics tab
```

## Monitoring & Alerts

```dart
// Set up error tracking with Sentry
import 'package:sentry/sentry.dart';

await Sentry.init(
  'YOUR_SENTRY_DSN',
  tracesSampleRate: 1.0,
);

// Capture exceptions
try {
  riskyOperation();
} catch (exception, stackTrace) {
  await Sentry.captureException(
    exception,
    stackTrace: stackTrace,
  );
}

// Performance monitoring
final transaction = Sentry.startTransaction(
  'important-operation',
  'http',
);

try {
  // Do something
} finally {
  await transaction.finish();
}
```

## Pre-commit Hooks

```bash
# .git/hooks/pre-commit
#!/bin/bash

echo "Running linting..."
flutter analyze

if [ $? -ne 0 ]; then
  echo "Linting failed. Commit aborted."
  exit 1
fi

echo "Running tests..."
flutter test

if [ $? -ne 0 ]; then
  echo "Tests failed. Commit aborted."
  exit 1
fi

echo "All checks passed. Proceeding with commit."
```

## Production Deployment Checklist

```
Pre-Release
☐ All tests passing
☐ Code review approved
☐ Test coverage >80%
☐ No crashes in staging
☐ Performance benchmarks met
☐ Security review completed
☐ Privacy policy updated
☐ Release notes prepared

Release Process
☐ Version bumped in pubspec.yaml
☐ CHANGELOG.md updated
☐ Git tag created
☐ Build created (APK/IPA)
☐ Signed correctly
☐ Uploaded to stores
☐ Beta track for staged rollout
☐ Monitoring enabled

Post-Release
☐ Crashes <0.1% of sessions
☐ User reviews monitored
☐ Performance metrics normal
☐ Analytics events tracking
☐ No critical issues reported
☐ Staged rollout 25%→50%→100%
```

## Best Practices

1. **Automate everything** with CI/CD
2. **Sign all release builds** with proper certificates
3. **Use staged rollouts** on Google Play
4. **Monitor crashes immediately** after release
5. **Maintain changelog** for every release
6. **Semantic versioning** (MAJOR.MINOR.PATCH)
7. **Test on real devices** before production
8. **Backup signing keys** securely
9. **Monitor performance metrics** post-release
10. **Quick rollback plan** ready

---

**Master deployment and ship Flutter apps with confidence!**
