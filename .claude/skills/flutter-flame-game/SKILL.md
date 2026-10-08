---
name: flutter-flame-game
description: Rules for building the interactive game surface of a Flutter app with the Flame engine — game loop and lifecycle, the BLoC ⇄ Flame bridge, component layering, board rendering, tap and drag input, move/capture/celebration animations, sound and haptics, performance budgets and testing (flame_test, goldens). Extends flutter-clean-mobile; use for any Flame, board, piece, animation, effect or game-sound work in these projects.
---

# Flutter + Flame Game Rules

These rules **extend `flutter-clean-mobile`**. Everything there still applies: feature-first Clean Architecture, BLoC, get_it, gen_l10n, the brand kit, tokens, RTL, accessibility and the test bar. This skill only adds the rules for the part of the screen that is a *game surface* drawn by Flame — a chess board first, and any later mini-games.

The reference product is a turn-based board game for kids (chess). The rules are written so they also fit other turn-based or puzzle games. Real-time action games need more (physics, fixed-step loops) and are out of scope.

Placeholders: `{app}` = Dart package, `{feature}` = feature folder, `{Game}` = PascalCase game class (e.g. `BoardGame`).

---

## 0. The one big rule: Flame is a view, not a brain

```
            intents (MoveAttempted, SquareTapped…)
   ┌──────────────┐  ───────────────────────────▶  ┌──────────────────┐   use cases   ┌───────────────────────┐
   │  Flame game  │                                 │   {Feature}Bloc  │ ────────────▶ │ domain (pure Dart)    │
   │ (renderer +  │  ◀───────────────────────────   │ single source of │               │ e.g. packages/        │
   │ input adapter)│        state (position,        │      truth       │ ◀──────────── │ chess_core: rules,    │
   └──────────────┘    lastMove, highlights, mode)  └──────────────────┘    results    │ legality, game end    │
                                                                                      └───────────────────────┘
```

- **Game rules never live in Flame components.** Legality, check, game end, scoring and turn order come from the domain (for chess: `packages/chess_core`). A component that needs to know "is this move legal" reads the legal-move set the bloc already put in state.
- **The bloc is the single source of truth.** Flame components never change game state. They send *intents* to the bloc and render what the bloc emits.
- **Every visual change is driven by a state diff with a cause.** The state carries what happened (`lastMove` with flags: capture, castle, en passant, promotion, check), so the game animates the exact event instead of guessing it by diffing two boards.
- **Flame owns only visual and transient input state:** drag position, hover square, running effects, particles. If it would matter after the app restarts, it belongs in the bloc.
- Anything that is not a canvas — promotion picker, dialogs, coach bubble, move list, buttons, timers — is **regular Flutter UI** (as a Flame overlay or around the `GameWidget`), so it gets l10n, RTL, Semantics and theme for free.

---

## 1. Tech stack additions (one package per job)

| Concern | Choice |
|---|---|
| Game engine | `flame` (pinned exact version in `pubspec.yaml`; upgrade deliberately) |
| Bloc bridge | `flame_bloc` (`FlameBlocProvider`, `FlameBlocListenable`) |
| Sound | `flame_audio` (wraps `audioplayers`; `AudioPool` for short SFX). No `just_audio`/`soundpool` next to it |
| Haptics | `HapticFeedback` from Flutter services. No haptics package |
| Vector art | Pieces and icons authored as SVG; rasterized once per size with `flutter_svg` (already in the base stack) into a cached atlas — see §5 |
| Particles / effects | Flame `ParticleSystemComponent` and `Effect`s. No Lottie inside the canvas (Lottie stays in Flutter UI for empty states and big celebrations) |
| Tests | `flame_test` (`testWithGame`, `testGolden`) + the base stack's `bloc_test`, `mocktail`, `alchemist` |

Do not add a second game engine, physics engine (`forge2d`) or tween library for a turn-based game.

---

## 2. Folder layout inside a feature

The Flame code is part of a feature's `presentation` layer. It never appears in `domain` or `data`.

```
lib/features/{feature}/presentation/
├── bloc/                         {feature}_bloc.dart  _event.dart  _state.dart
├── pages/                        {feature}_page.dart          # Scaffold + GameWidget + Flutter overlays
├── widgets/                      move_list.dart, coach_bubble.dart, promotion_picker.dart …
└── game/
    ├── {game}.dart               # class {Game} extends FlameGame — wiring only, < 200 lines
    ├── board_geometry.dart       # pure Dart: size → square rects, square ↔ offset, flip  (unit-tested)
    ├── input/
    │   └── move_input_machine.dart  # pure Dart state machine: idle / selected / dragging / awaitingPromotion
    ├── render/
    │   ├── board_theme.dart      # colors, coordinate style, from brand tokens
    │   └── piece_atlas.dart      # SVG → ui.Image cache per (pieceSet, squarePx)
    ├── components/
    │   ├── board_component.dart          # 64 squares + coordinates, one render() call
    │   ├── highlight_layer.dart          # last move, selected, check, hint squares
    │   ├── legal_targets_layer.dart      # dots / rings on legal destinations
    │   ├── piece_component.dart          # one per piece, SpriteComponent
    │   ├── pieces_layer.dart             # owns PieceComponents, applies state diffs
    │   └── arrow_layer.dart              # coach arrows and hint arrows
    ├── effects/                  capture_burst.dart, check_pulse.dart, confetti.dart, shake.dart
    └── audio/
        └── game_sounds.dart      # abstract GameSounds + FlameGameSounds + NoopGameSounds
```

Shared game pieces used by more than one feature (`board_geometry`, `piece_atlas`, `game_sounds`, the board components) move to `lib/core/game/`. Features never import another feature's `game/` folder.

`{feature}_injection.dart` registers `GameSounds` (singleton) and `PieceAtlas` (singleton). The `{Game}` itself is **not** registered in get_it — the page creates it.

---

## 3. Lifecycle and the game loop

- **Create the game once per page**, in `State.initState` (or a `late final` field), never inside `build()`. Rebuilding the `GameWidget` with a new game instance loses all components and reloads assets.
- The page puts the bloc into Flame with `FlameBlocProvider<{Feature}Bloc, {Feature}State>.value(value: bloc, children: [...])` inside `{Game}.onLoad`. Components that react to state mix in `FlameBlocListenable` and override `listenWhen` narrowly (e.g. only when `position` or `lastMove` changed).
- **Intents go out through a callback, not `getIt`:** the page passes `onIntent: bloc.add` into the game constructor. Components call `game.onIntent(MoveAttempted(from, to))`.
- `onLoad` preloads everything the game draws or plays (atlas, sounds). Show a skeleton board in Flutter (`GameWidget.loadingBuilder`) while it loads — never a bare spinner.
- `onGameResize` recomputes `BoardGeometry` and re-lays out components. It is the only place that changes sizes.
- **A board game idles most of the time. Do not burn battery at 60 fps on a still board:**
  - keep `update(dt)` empty for static components;
  - the game pauses its engine (`pauseEngine()`) when no effect is running and no drag is active for 2 s, and resumes (`resumeEngine()`) on any pointer event, state change or effect start. Wrap this in a small `IdleController` with a test;
  - `GameWidget` uses `pauseWhenBackgrounded: true`; the page also pauses when its route is covered (`RouteAware.didPushNext`).
- Clamp `dt` (e.g. `min(dt, 1/20)`) in anything time-based so a resume after a pause does not jump animations.
- Never `await` network or engine calls inside `update`, `render` or input handlers. Those go to the bloc.

---

## 4. Board geometry, orientation and RTL

- `BoardGeometry` is pure Dart and the only code that maps between squares and pixels: `rectOf(Square)`, `squareAt(Offset) → Square?`, `flipped`, `squareSize`, `origin`. Every component asks it; nobody does `x * squareSize` inline.
- **Board size:** the largest square that fits the available area minus the 16 dp gutter, with `squareSize` **snapped to whole physical pixels** (`(size * dpr).floor() / dpr / 8`) so squares have no seams or blurry edges. On a 360 dp phone this gives ≥ 41 dp squares, which is the minimum for kids; the layout must never shrink the board below 40 dp per square — move side panels below the board instead.
- **The board is never mirrored by RTL.** Wrap the `GameWidget` and any coordinate labels in `Directionality(textDirection: TextDirection.ltr)`. a1 is bottom-left for the player with white, whatever the app language. The only orientation change is **flip** (playing black, or a user setting), and it comes from state, not from locale.
- Everything *around* the board (move list, player cards, coach bubble, buttons) follows the app's RTL normally.
- Coordinate labels and notation script (Latin `a–h`/`1–8` or Arabic letters and digits) are a setting in state, rendered by the board component from l10n-provided strings.
- Tablets and landscape: the board takes the short side; side panels go to the trailing side (`end`) in both directions.

---

## 5. Rendering

**Layers** use explicit `priority` constants from one file (`GameLayers`), never magic numbers:

| Layer | Priority |
|---|---|
| board squares + coordinates | 0 |
| square highlights (last move, selected, check, hint) | 10 |
| legal-target dots / capture rings | 20 |
| pieces | 30 |
| arrows (coach, hint) | 40 |
| dragged piece | 100 |
| effects and particles | 200 |

- **Draw the board in one `render()` call** — 64 `drawRect`s with cached `Paint`s — not 64 components. Same for the highlight and legal-target layers: one component per layer, rebuilt only when its slice of state changes.
- **Pieces** are `SpriteComponent`s from a `PieceAtlas`:
  - pieces are authored as SVG in `assets/pieces/{set}/` (brand kit owns the art);
  - on load and on resize, the atlas rasterizes each SVG once to a `ui.Image` at `squarePx = squareSize × devicePixelRatio`, caches it by `(set, squarePx)`, and disposes the old images;
  - never parse or draw SVG per frame.
- **Cache everything** used in `render`: `Paint`, `Path`, `Rect`, `TextPainter` for coordinates. No allocations in `render` or `update` — check with the DevTools memory view.
- Board and piece themes come from a `BoardTheme`/`PieceSet` value built from the brand tokens (`context.tokens`) and passed into the game, so theme switches and light/dark mode do not need asset reloads. The board stays high-contrast in dark mode.
- Avoid `saveLayer` (opacity on a group, `ColorFilter` on many components). Fade a single sprite with `OpacityEffect` on that sprite only.

---

## 6. Input: tap and drag

Both ways to move are **always on**: tap-tap (select, then tap target) and drag-and-drop. Kids switch between them mid-game.

- Input goes through a **pure Dart `MoveInputMachine`** (unit-tested without Flame):
  `idle → selected(sq) → dragging(sq, pointer) → awaitingPromotion(from, to) → idle`.
  The Flame component only translates pointer events into machine events, and machine outputs into intents (`MoveAttempted`) or visual changes (selection, dots).
- **Touch target is the whole square**, never just the piece's drawn pixels.
- **Tap slop:** a press that moves less than `kTouchSlop` is a tap, more is a drag. A tap on another own piece switches the selection; a tap on an empty non-target square clears it.
- **Dragging:** the piece lifts above the finger (offset up by ~0.6 square so the finger does not hide it), scales to 1.15, and moves at `dragged piece` priority. The square under the piece's center gets a hover ring.
- **Dropping:**
  - on a legal target → send `MoveAttempted`; the piece snaps to the target immediately (no second animation from the drag point);
  - anywhere else → the piece slides back (150 ms) with a small shake and the soft "illegal" sound. **No red flashes, no error text for kids** — the coach can explain later if they keep trying.
- **Promotion** opens a Flutter overlay (`overlays.add('promotion')`) with four big piece buttons near the promotion square, with Semantics labels. Cancel returns the pawn.
- Input is locked (machine ignores pointers) while it is not the player's turn, while a move animation is running, and in replay mode; a locked tap gives a gentle wiggle on the piece, not silence.
- Kid settings that come from state: show legal targets (default on), touch-move rule (default off), confirm move (default off), allow take-back (parent setting).
- Multi-touch: track one pointer id at a time; a second finger cancels the drag back to its square.

---

## 7. Animation and effects

Durations and curves come from the brand motion tokens (`context.tokens.motion`), never literals in components.

| Event | Effect | Default |
|---|---|---|
| Move (computer, opponent, replay) | `MoveEffect.to` with `EffectController(curve: Curves.easeOutCubic)` | 220 ms |
| Player's tap-tap move | same, shorter | 160 ms |
| Capture | captured piece scales to 0.6 + fades, small `CaptureBurst` particles in brand colors | 180 ms |
| Castling | king and rook move **together** | 220 ms |
| En passant | captured pawn fades on **its own square**, not the target square | 180 ms |
| Promotion | pawn scales down, new piece pops in with overshoot (`Curves.easeOutBack`) | 260 ms |
| Check | king square pulses red twice | 2 × 300 ms |
| Illegal drop | slide back + 3-cycle horizontal shake | 150 + 180 ms |
| Win / puzzle solved / star | `Confetti` particles (≤ 120 particles) + Flutter overlay (stars, badge) | 1.2 s |
| Hint | arrow grows from source to target, target square breathes | 400 ms |

- **One animation queue per board.** If a new state arrives while a move is still animating (fast opponent, undo, jump in replay), finish the running effects instantly (`controller.setToEnd()`), sync pieces to the current position, then animate the new event. The visuals must **never** drift from state.
- After every animation completes, `PiecesLayer.syncTo(position)` verifies each piece is exactly on its square (and fixes it silently). This is the safety net against effect bugs.
- Undo and replay-backwards animate the move in reverse; jumping more than one move just syncs with a 120 ms cross-fade.
- **Reduced motion:** when `MediaQuery.disableAnimations` is set (or the in-app "calm mode" for sensitive kids), all durations become 0 except a 100 ms fade, and no particles, shakes or pulses play. The game reads this flag from the page, not from a global.
- Effects that spawn particles take a `Random` from the game (seedable) so tests are deterministic.

---

## 8. Sound and haptics

- All sounds go through an abstract `GameSounds` (`move()`, `capture()`, `castle()`, `check()`, `promote()`, `illegal()`, `win()`, `drawOrLoss()`, `star()`, `tap()`), implemented by `FlameGameSounds` and faked by `NoopGameSounds` in tests. Components never call `FlameAudio` directly.
- `FlameGameSounds.load()` preloads all files in `onLoad`; frequent short sounds (move, capture, tap) use an `AudioPool` (`maxPlayers: 3–4`) so fast sequences do not cut each other off.
- **Assets:** `assets/audio/sfx/*.m4a` (AAC, mono, 44.1 kHz, ≤ 50 KB each, ≤ 600 ms for move/capture), loudness-normalized so no sound jumps out. Optional background music in `assets/audio/music/`, **off by default**, looped, ducked under SFX.
- **Kid-friendly sound design:** soft wooden or plastic clicks, warm tones for success; losing and illegal sounds are gentle and never mocking or alarming.
- Audio context: mix with other audio, respect the iOS silent switch (ambient category), stop on app background.
- Separate settings for **SFX** and **music** volume/mute, stored with the other settings (prefs), read by `GameSounds` on change — not per call.
- Haptics: `HapticFeedback.selectionClick()` on pick-up and drop, `lightImpact()` on capture, `mediumImpact()` on win. Respect the in-app haptics setting.
- Sound is never the only signal: every sound event also has a visual one (accessibility, muted devices).

---

## 9. Accessibility on a canvas

A Flame canvas is invisible to screen readers, so the page adds a **Semantics grid** on top of the `GameWidget`:

- 64 transparent `Semantics` nodes (8×8 `Stack` positioned from `BoardGeometry`), each labeled from l10n, e.g. "حصان أبيض، g1" / "White knight, g1", with `selected`, and a hint "legal move" on legal targets.
- Double-tap on a node sends the same events as a tap on the canvas, so blind or low-vision players can play tap-tap.
- Moves are also announced (`SemanticsService.announce`) in the current language: "الحصان إلى f3، كش" / "Knight to f3, check".
- High-contrast board theme and an optional "show coordinates on every square" mode for kids learning the board.

---

## 10. Performance budgets

| Metric | Budget |
|---|---|
| Frame time on a low-end Android (2 GB RAM, ~2019 GPU), profile mode | ≤ 16 ms during move + capture effects, 0 jank frames in the integration test |
| Idle board | engine paused, ~0% GPU (see §3) |
| `onLoad` (atlas + sounds) | ≤ 300 ms after the page route is visible |
| Peak particles on screen | ≤ 150 |
| Game memory (atlas + audio) | ≤ 25 MB |

- Profile with `fvm flutter run --profile` and DevTools (performance + memory). The debug flavor can show `FpsTextComponent` behind a dev setting; it is never in release builds.
- Rebuild the atlas only on resize or piece-set change; dispose old `ui.Image`s.
- Do not rebuild the Flutter widget tree around the `GameWidget` on every bloc state — pages use `BlocSelector`/`buildWhen` so only the panels that changed rebuild.

---

## 11. Testing

- **Unit tests (pure Dart, fast, most of the coverage):**
  - `BoardGeometry`: square ↔ offset for every square, flipped and not, odd sizes, pixel snapping;
  - `MoveInputMachine`: every transition, including tap-switch-selection, drag-cancel by second finger, promotion cancel, locked input;
  - the animation planner: given (previous state, new state with `lastMove`), it produces the right effect list for normal move, capture, castling both sides, en passant, promotion with and without capture, and multi-move jumps.
- **Component tests with `flame_test`** (`testWithGame<{Game}>` with a fake bloc and `NoopGameSounds`):
  - tapping a square at its geometry offset sends the expected intent;
  - a drag-drop on an illegal square returns the piece and plays `illegal`;
  - a state with `lastMove` castling moves both king and rook; after `game.update(1)` pieces are exactly on their squares;
  - reduced motion: effects finish in one update and no particles are added.
- **Golden tests** (`testGolden` from `flame_test`, plus `alchemist` for the full page): starting position, mid-game with highlights and legal dots, check, flipped board — for each board theme in light and dark, at 360×640 and tablet size. Use the fixed seed and `setToEnd()` on effects so goldens are stable.
- **Bloc tests**: the game feature's bloc with the real domain engine (e.g. `chess_core`), covering every intent.
- **Integration test:** open a game, play a short sequence by tap and by drag (including a capture and castling), check the move list and sound calls, background and resume the app, and verify the board matches state.
- CI adds `flame_test` goldens to the base stack's golden job; golden updates are reviewed as images in the PR.

---

## 12. Workflow rules for Claude

1. **New game surface, in this order:** domain model and rules (pure Dart, tested) → bloc state that includes `lastMove` with event flags → `BoardGeometry` + tests → `MoveInputMachine` + tests → `{Game}` with board and pieces layers → input wiring → animation planner + effects → sounds → Semantics grid → goldens → integration test.
2. Never put a rule of the game inside a component. If you need one, add it to the domain and expose the result in state.
3. Before adding a new effect, add its duration/curve to the motion tokens and its reduced-motion behaviour.
4. Before adding a new sound, add it to `GameSounds`, the asset list, the settings mute path and `NoopGameSounds`.
5. Every new component gets a `flame_test` test; every new visual state gets a golden.
6. Before calling a game screen done: profile mode on a low-end Android device, ar and en, light and dark, flipped and not, reduced motion on, sound muted, a screen reader pass on the Semantics grid, and background/resume mid-animation.
