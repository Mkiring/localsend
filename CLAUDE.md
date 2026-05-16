# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LocalSend is a cross-platform file/message sharing app that works over local networks without internet. Built with Flutter (Dart) for UI and Rust for performance-critical networking/crypto, connected via flutter_rust_bridge.

## Build & Development Commands

Flutter version is pinned to **3.38.10** (managed via FVM, see `.fvmrc`).

```bash
# Get dependencies (must run for both packages)
cd app && flutter pub get
cd common && dart pub get

# Also needed after pub get in app:
cd app/rust_builder/cargokit/build_tool && flutter pub get

# Code generation (from app/)
dart run build_runner build

# Formatting (150-char line width, preserved trailing commas)
dart format lib test                    # app directory
dart format lib test                    # common directory

# Analysis
flutter analyze                         # app directory
dart analyze                            # common directory

# Tests
flutter test                            # app directory (from app/)
flutter test test/unit/some_test.dart   # single test file
dart test                               # common directory (from common/)

# Run the app
flutter run                             # from app/
flutter run -d linux                    # target specific platform
```

## Monorepo Structure

```
app/          # Flutter application (UI, state management, platform integration)
common/       # Shared Dart library (models, utilities) - used by app/ and cli/
core/         # Rust library (crypto, HTTP, WebRTC) with feature flags
server/       # Standalone Rust server (Axum-based)
rust_builder/ # Flutter Rust Bridge build tooling
scripts/      # Platform-specific build/packaging scripts
```

**Dependency graph:** app → common, app → core (via rust_builder/flutter_rust_bridge), server → core

## Architecture

### Dart Side (app/)

- **State management:** Refena (Redux-style providers + notifiers). See `app/lib/provider/`.
- **Routing:** Routerino with tab-based navigation (receive/send/settings tabs). Entry: `app/lib/pages/home_page.dart`.
- **i18n:** Slang with JSON files in `app/assets/i18n/`. Base locale: English. Code generation via slang_build_runner.
- **Serialization:** dart_mappable for models, freezed for Rust bridge DTOs.
- **Generated code** lives in `lib/gen/` and `lib/rust/` — do not edit these files.

### Rust Side (core/)

- **API surface** is in `core/src/api/` — these are the functions exposed to Dart via flutter_rust_bridge.
- **Feature flags:** `crypto`, `http`, `webrtc`, `webrtc-signaling`. `full` enables all.
- **Bridge config:** `app/flutter_rust_bridge.yaml` maps `crate::api` → `app/lib/rust/`.

### Key Providers (app/lib/provider/)

- `settings_provider.dart` — user preferences
- `server/server_provider.dart` — local HTTPS server lifecycle
- `network/nearby_devices_provider.dart` — device discovery
- `network/send_provider.dart` — file sending logic
- `persistence_provider.dart` — data persistence
- `local_ip_provider.dart` — network interface detection

### Generated Files

These directories are auto-generated and should not be edited:
- `app/lib/gen/` (i18n, assets)
- `app/lib/rust/frb_generated*.dart` (Rust bridge)
- `app/lib/rust/api/*.freezed.dart` (Freeze DTOs)
- `**/*.g.dart`, `**/*.freezed.dart` (build_runner output)

The `lib/gen/` directory is deleted before format checks in CI. Regenerate with `dart run build_runner build`.

## Coding Conventions

- **Line width:** 150 characters
- **Trailing commas:** preserved
- **Quotes:** single quotes (`prefer_single_quotes`)
- **Imports:** always use package imports (`always_use_package_imports`)
- **Dependencies:** sorted alphabetically in pubspec.yaml
- **Widget keys:** `use_key_in_widget_constructors` is disabled
- **Futures:** `unawaited_futures` and `discarded_futures` are enforced
- **Analyzer plugins:** `custom_lint` is enabled in the app

## CI Checks (must pass)

CI runs three jobs: format, test, packaging. The format job removes `lib/gen/` before checking formatting. All `dart format`, `flutter analyze`, `flutter test`, and `dart test` commands must pass cleanly.
