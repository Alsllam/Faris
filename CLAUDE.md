# Faris · فارس — monorepo guide

Faris is a chess app for kids (6–14, the main audience), teens, adult beginners and their families. Players learn through short lessons and puzzles, play the computer or family and friends, and get help from a patient coach that explains moves in simple Arabic or English. Parents see progress and control time limits and online play.

This file holds the rules that apply everywhere. Each top-level folder has its own `CLAUDE.md` that names the skill to use there and the product-specific values that override the skill's placeholders.

---

## Repository layout

| Folder | What it is | Skill(s) to load first |
|---|---|---|
| `mobile/` | Flutter app (iOS + Android): learn, play, coach, parent-lite | `flutter-clean-mobile` + `flutter-flame-game` (board, animation, sound) |
| `packages/chess_core/` | Pure Dart rules engine: board, move generation, legality, game end, FEN/PGN/SAN | none — see `packages/chess_core/CLAUDE.md` |
| `backend/` | .NET 9 modular backend: Identity & Families, Games & Matchmaking (SignalR), Ratings, Lessons & Progress, Puzzles | `dotnet-modular-backend` |
| `ai-service/` | Python FastAPI coach: turns engine analysis into kid-friendly explanations; RAG over Faris lesson content | `python-azure-rag-service` |
| `frontend/` | Angular + Nx: parent dashboard, admin for lessons and puzzles, later a web version of the game | `angular-nx-frontend` |
| `docs/` | GDD, brand kit, architecture, decisions (ADRs) | none — see `docs/CLAUDE.md` |
| `.claude/skills/` | Project copies of the skills above. **The copy in this repo is the source of truth for Faris**; change it here, in its own PR | — |

Before working in a folder, read that folder's `CLAUDE.md` and load its skill(s). When a folder's `CLAUDE.md` and its skill disagree, the folder's `CLAUDE.md` wins.

---

## Hard rules (every folder, every PR)

1. **Chess rules are exact.** Castling (all conditions), en passant, promotion, check, checkmate, stalemate, threefold repetition, the 50-move rule and insufficient material follow FIDE Laws. There is **one** implementation: `packages/chess_core`. Mobile and the future web game use it; the backend validates moves with a server-side engine that is tested against the same perft and position suites. No other code decides legality.
2. **Playing strength comes from a chess engine, never from an LLM.** The LLM never chooses, evaluates or validates a move. The coach gets engine facts (best move, evaluation, mistake class, threats, principal variation) and only *explains* them. If the engine did not say it, the coach does not claim it.
3. **Engine licensing is decided deliberately.** GPL engines (e.g. Stockfish) are not linked into or shipped inside the closed-source apps. See `docs/architecture/` (step 6) for the decision; until it is approved, do not add any engine binary or package.
4. **Kids' safety and privacy (Saudi PDPL, and COPPA/GDPR-K style practice):**
   - no free-text chat between players — only a fixed list of preset friendly messages and emoji, localized;
   - online play only with friends a parent approved (friend codes / family links), never open matchmaking with strangers for child accounts unless a parent turns it on;
   - child accounts are created by a parent; parental consent is recorded and revocable; data minimization (no real names required for kids, nickname + avatar);
   - no ads, no third-party ad SDKs, no advertising IDs, no behavioural profiling; analytics are first-party, aggregated and opt-in for child accounts;
   - personal data is stored in a Saudi/approved region, with export and delete on request;
   - anything the LLM writes to a child passes content safety and is short, kind and age-appropriate.
5. **Arabic first, RTL everywhere** except the board itself, which keeps standard orientation (a1 bottom-left for white; flipped only when playing black). Notation can be shown in Arabic or Latin letters and digits as a user setting.
6. **No secrets in the repo** (`.env.example` only; keys live in Key Vault / CI secrets). No AI keys in any client.

---

## Shared vocabulary

Use these names in code, APIs, docs and UI keys so every layer matches:

| Concept | Name |
|---|---|
| A family account (parent + kids) | `Family`, `ParentProfile`, `ChildProfile` |
| One learning unit | `Lesson` (in a `Chapter` of the `LearningPath`) |
| A puzzle | `Puzzle` (FEN + solution line + themes + rating) |
| A game | `Game` (mode: `VsComputer`, `LocalHotSeat`, `OnlineFriend`) |
| Computer difficulty | `BotLevel` 1–10 (names and behaviour in `docs/GDD.md`) |
| Coach output | `CoachComment` (kind: `Praise`, `Inaccuracy`, `Mistake`, `Blunder`, `Hint`, `Summary`) |
| Engine facts given to the coach | `MoveAnalysis` / `GameAnalysis` |
| Rewards | `Star`, `Badge`, `Streak` |

---

## Workflow

- **One branch and one PR per step or feature.** Branch names: `claude/step-{n}-{slug}` for the setup steps, `feature/{area}-{slug}` afterwards. PRs target `main`.
- Commits are small and descriptive. Generated files are committed only where the folder's `CLAUDE.md` says so.
- Every PR keeps its folder's checks green (see each folder's "Commands" section). CI is added per folder when that folder gets real code.
- Docs that change a product decision update `docs/GDD.md` or add an ADR in `docs/architecture/decisions/` in the same PR.
- New user-facing text is added in Arabic **and** English in the same commit.
