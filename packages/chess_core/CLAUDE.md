# packages/chess_core — pure Dart rules engine

No skill applies; these rules are the contract. Root `CLAUDE.md` hard rule 1 applies: **this package is the only place that decides what is legal** for the apps.

Built in step 5. Until then this folder holds only this file.

## Rules

- **Pure Dart** (Dart 3, sound null safety). No Flutter, no `dart:io`/`dart:html`, no platform channels, so it runs in Flutter, on a server, in tests and compiles to JS/WASM for the web game. Runtime dependencies: none (dev: `test`, `lints`).
- Implements the FIDE Laws of Chess exactly: all piece moves, castling (king and rook unmoved, squares between empty, king not in check and not passing through or landing on an attacked square; Chess960 not in scope), en passant (only immediately, and only when it does not expose the king — including the horizontal-pin case), promotion to Q/R/B/N, check, checkmate, stalemate, threefold repetition (same position = same placement, side to move, castling rights and *legal* en-passant possibility), the 50-move rule (claimable) and 75-move rule (automatic), fivefold repetition (automatic), and dead positions by insufficient material (K vs K, K+B vs K, K+N vs K, K+B vs K+B same-coloured bishops).
- **Immutable public API:** `Position` is immutable; `position.play(move)` returns a new position. A `Game` keeps history, repetition counts and results. Internals may use mutable make/unmake for speed, hidden behind the API.
- Formats: FEN in/out (validated, with clear errors), UCI move strings, SAN in/out (disambiguation, `+`/`#`, `=Q`), PGN import/export (tags, comments, NAGs; variations may be skipped in the MVP).
- Localized notation is a **presentation** concern: the package exposes structured moves (piece, from, to, capture, promotion, check) plus SAN; Arabic letters are mapped by the apps.

## Tests (required before merge)

- **Perft** from the start position to **depth 5** (4,865,609 nodes) and the standard test positions (Kiwipete, positions 3–6 from the Chess Programming Wiki perft page) to the depths listed in the test file, with expected node counts. Long-running depths are tagged `slow` and run in CI.
- Unit tests for each special rule, FEN/SAN/PGN round-trips, every draw rule and game-end reason.
- `dart analyze` with zero issues, `dart format` clean, coverage ≥ 95%.

## Commands

```
dart pub get
dart analyze
dart format --output=none --set-exit-if-changed .
dart test                 # fast suite
dart test --run-skipped -t slow   # deep perft
```
