# mobile/ — Faris Flutter app

**Load these skills first:** `flutter-clean-mobile` (architecture, stack, l10n, security, tests) and `flutter-flame-game` (everything drawn on the board: Flame components, input, animation, sound, performance). Root `CLAUDE.md` hard rules apply.

The app is not scaffolded yet. Create it with `fvm flutter create --org sa.faris --project-name faris --platforms ios,android .` inside this folder, then follow `flutter-clean-mobile` §14.1. Screens wait for the architecture approval (step 6).

## Product values for the skill placeholders

| Placeholder | Value |
|---|---|
| `{app}` | `faris` |
| Flavors | `dev`, `staging`, `prod` (no `uat` for now) |
| Rules engine | path dependency `chess_core: { path: ../packages/chess_core }` — never re-implement rules in the app |

## Planned features (`lib/features/`)

`onboarding` (parent sign-up, consent, child profiles) · `learn` (learning path, lessons) · `puzzles` (puzzles, daily challenge) · `play` (vs computer, local hot-seat, online friend) · `coach` (move/game explanations, hints) · `rewards` (stars, badges, streaks) · `friends` (parent-approved friends, preset messages) · `parent` (PIN-gated area: progress, time limits, online controls) · `settings`.

The board, pieces and effects live in `lib/core/game/` because `learn`, `puzzles` and `play` all use them (`flutter-flame-game` §2).

## Overrides of the base skill for a kids' product

- **Analytics and crash reporting:** Crashlytics yes (no personal data). `firebase_analytics` only with advertising-ID collection disabled, no ad SDKs, and child-account events limited to aggregated, first-party learning events. No `google_maps_flutter`, `geolocator`, media pickers or `record` unless a feature is approved that needs them.
- **Push notifications** go to parent devices; child-device notifications are limited to friendly reminders the parent allowed.
- **Offline-first** (skill §6) applies to lessons, puzzles, progress, stars and games vs the computer: a child can learn and play the computer with no connection; progress syncs later.
- **Touch targets:** at least 56×56 dp for kid-facing controls (the skill's 48 dp is the minimum for parent screens). Board squares at least 40 dp.
- **Parent area** is behind a parent PIN / biometric gate and a simple adult check.
- **Time limits** set by parents are enforced on the device (and synced), with a gentle "time to rest" screen, never mid-move.
- **Coach** text comes from the backend/AI service; the app never calls an LLM. Engine play vs the computer follows the decision in `docs/architecture/` (step 6).

## Commands (once scaffolded)

```
fvm flutter pub get
fvm flutter analyze
dart format --set-exit-if-changed lib test
fvm flutter test --coverage
fvm flutter run --flavor dev -t lib/main_dev.dart --dart-define-from-file=dart_defines/dev.json
```
