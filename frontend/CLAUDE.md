# frontend/ — Faris web (Angular + Nx)

**Load this skill first:** `angular-nx-frontend`. Root `CLAUDE.md` hard rules apply.

Not scaffolded yet. When it is, create the Nx workspace **inside this folder** (the skill's `{product}-frontend` root is `frontend/` here), following the skill's §2 layout.

## Product values for the skill placeholders

| Placeholder | Value |
|---|---|
| `{scope}` | `@faris` |
| Workspace root | `frontend/` |
| Brand kit | `docs/brand/` (step 4) is the source; the skill's §0 brand-kit steps copy from there, never invent a second identity |

## Apps (`apps/`)

| App | For | Notes |
|---|---|---|
| `parents` | parent dashboard | progress reports (ECharts), time limits, online-play controls, approved friends, consent and data export/delete. Arabic-first, mobile-friendly |
| `admin` | Faris content team | lesson and chapter authoring, puzzle bank (import, tag themes, set ratings), preset friendly messages, badges, coach templates review. Permission-gated |
| `play` (later) | web version of the game | uses the same rules as `packages/chess_core` (via a shared TS port or a WASM/JS build — decided in step 6). Not in the MVP |

## Feature libraries (`libs/`)

`families` · `progress` · `parental-controls` · `lessons` · `puzzles` · `messages` · `badges` — each with `config` and `ui-common` as in the skill, and one proxy lib per backend host (`shared/families-proxy`, `shared/games-proxy`, `shared/lessons-proxy`, `shared/puzzles-proxy`, `shared/ratings-proxy`).

A board widget for lesson/puzzle authoring and previews goes in `libs/shared/board` (type `ui`). It renders positions and records moves; legality comes from the rules engine, not from Angular code.

## Overrides

- Kids do not use this web app in the MVP; the `parents` app still follows the kids'-privacy rules for any child data it shows (no third-party trackers, no ad scripts).
- Charts use the brand palette from `docs/brand/`.

## Commands (once scaffolded)

```
npx nx run-many -t lint test build
npx nx affected -t lint test build
npx nx serve parents
```
