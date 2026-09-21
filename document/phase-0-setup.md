# Phase 0 — Environment Setup: Status

Reference: [.claude/implementation-plan.md](../.claude/implementation-plan.md) §1.

## Completed

| Task | Detail |
|---|---|
| Install Flutter SDK (stable channel) | Flutter 3.47.5, stable channel, at `C:\src\flutter` |
| Install Android Studio + Android SDK + AVD emulator | Android SDK 36.0.0 toolchain; AVD **`Pixel_7`** created (Android 14, API 34) |
| Run `flutter doctor`, resolve Android toolchain/license issues | All Android licenses accepted. Only remaining `flutter doctor` warning is missing Visual Studio (Windows desktop dev) — irrelevant, this MVP is Android-only |
| Install VS Code Flutter/Dart extensions | `dart-code.dart-code` and `dart-code.flutter` confirmed installed |
| `flutter create pdf_redactor` scaffold, confirm it builds and runs on the emulator | Scaffold exists (`pubspec.yaml` / `lib/main.dart` still default counter app — expected, dependencies land in Week 1 Task 1.1). Verified: booted `Pixel_7`, ran `flutter run -d emulator-5554`, Gradle built `app-debug.apk`, installed and launched successfully with hot reload active |

## Not yet done

| Task | Detail |
|---|---|
| Register a Syncfusion account, confirm Community License eligibility, generate a license key | Account created, but the key obtained so far is a **7-day trial key**, not the free **Community License**. Community License eligibility (per the plan's decision): org revenue < $1M USD, ≤5 developers, not an existing paying customer — this project qualifies. **Action needed:** check the Syncfusion account dashboard under "License and Downloads" for a distinct Community License option (separate from the trial), since a 7-day key will break the build after a week if used as-is. |

## Notes for later (Week 1 Task 1.1)

- Once the correct (Community) license key is confirmed, it should **not** be hardcoded into a file committed to git. Plan: keep it in an untracked file (e.g. `lib/syncfusion_key.dart`, added to `.gitignore`) or pass via `--dart-define`, then call `SyncfusionLicense.registerLicense(...)` before `runApp()` in `main.dart`.
- The `flutter run` session started during Phase 0 verification may still be active locally (hot reload/restart available) — safe to quit at any time, no state depends on it.

## Gate status

Per the plan: *"Gate: emulator boots the default Flutter counter app before Week 1 Task 1.1 starts."* — **met**. The only open item blocking full Phase 0 close-out is confirming the correct Syncfusion license type.
