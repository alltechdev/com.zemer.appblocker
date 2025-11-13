# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Dumbdroid is an Android parental controls application that uses Device Owner APIs to restrict app access and installation. The app requires system-level permissions and must be built as a platform-signed privileged application to function properly.

## Build System

This project uses the Android Open Source Project (AOSP) build system rather than standard Gradle builds:

- **Primary build file**: `Android.bp` - AOSP Soong build configuration
- **Build command**: `m DumbdroidAdmin` (within AOSP build environment)
- The app is built as a privileged system app with platform certificate
- `build.gradle.kts` exists but is not the primary build system

## Architecture Overview

### Core Components

1. **DeviceRestrictionManager** (`src/main/java/eu/dumbdroid/deviceowner/policy/DeviceRestrictionManager.kt`)
   - Central policy enforcement engine
   - Uses `PackageManager.setApplicationHiddenSettingAsUser()` to hide/block apps (requires Device Owner privileges)
   - Manages Play Store blocking via hiding + intercepting app market intents
   - Sets `UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES` restriction
   - Requires Android N+ (Build.VERSION_CODES.N) for hiding functionality
   - Device Owner verification: checks ability to call `getApplicationHiddenSettingAsUser()` on own package

2. **PinStorage** (`src/main/java/eu/dumbdroid/deviceowner/storage/PinStorage.kt`)
   - Handles PIN authentication with SHA-256 hashing + salting
   - Supports optional PIN (can be skipped during setup)
   - Stores encrypted PINs in SharedPreferences with Base64-encoded salt

3. **MainActivity** (`src/main/java/eu/dumbdroid/deviceowner/ui/MainActivity.kt`)
   - Fragment coordinator using callback interfaces
   - Auto-locks on pause/stop if PIN is set
   - Flow: Setup PIN → PIN Entry → Restriction Management

4. **DeviceOwnerReceiver** (`src/main/java/eu/dumbdroid/deviceowner/admin/DeviceOwnerReceiver.kt`)
   - Entry point for device admin policies
   - Extends `DeviceAdminReceiver` for system callbacks

### UI Flow

```
SetupPinFragment → PinEntryFragment → RestrictionFragment
                                   ↔ ChangePinFragment
```

- **SetupPinFragment**: Initial PIN creation (can be skipped)
- **PinEntryFragment**: Authentication gate (shows on pause/stop)
- **RestrictionFragment**: Main UI for managing app restrictions
- **ChangePinFragment**: PIN modification interface

### Play Store Blocking Strategy

When Play Store restrictions are enabled:

1. Hide Play Store package (`com.android.vending`) using Device Owner API
2. Enable `PlayStoreBlockedActivity` as component
3. `PlayStoreBlockedActivity` intercepts:
   - `ACTION_MAIN` with `CATEGORY_APP_MARKET`
   - `market://` scheme URIs
   - `https://play.google.com/store` URLs
4. Set `DISALLOW_INSTALL_UNKNOWN_SOURCES` user restriction

### App Discovery

`DeviceRestrictionManager.getManageableApplications()`:
- Queries all installed apps (including uninstalled/disabled)
- Filters to only apps with launcher activities using `declaresLauncherActivity()`
- Excludes self, Play Store (`com.android.vending`), and Settings (`com.android.settings`)
- Returns sorted list by app label

## Key Manifest Features

- `android:sharedUserId="android.uid.system"` - Runs as system user
- `coreApp="true"` - Core system application
- `QUERY_ALL_PACKAGES` permission required
- MainActivity has no launcher intent (hidden from launcher by design)
- PlayStoreBlockedActivity dynamically enabled/disabled via PackageManager

## Development Notes

- This app requires Device Owner status to function, which means:
  - Must be provisioned via `adb shell dpm set-device-owner`
  - Cannot be set if other accounts exist on device
  - Typically requires factory reset for initial setup
- The app will not have Device Owner permissions when installed as regular APK
- Testing requires either AOSP build environment or custom ROM integration
- Minimum Android version: Android 7.0 (API 24) for core hiding functionality
