# backend/ — Faris .NET modular backend

**Load this skill first:** `dotnet-modular-backend`. Root `CLAUDE.md` hard rules apply.

Not scaffolded yet; follow the skill's §2 layout and §8 workflow when the architecture (step 6) is approved.

## Product values for the skill placeholders

| Placeholder | Value |
|---|---|
| `{Co}` | `Faris` |
| `{Product}` | `Chess` → projects are `Faris.Chess.{Module}.Domain` etc. |
| Solution | `backend/Faris.Chess.sln` |

## Modules

| Module (`{Module}`) | Owns |
|---|---|
| `Families` (Identity & Families) | parent accounts, child profiles (nickname + avatar, no real names required), parental consent records, family links, parent-approved friends, parental controls (time limits, online play on/off, who can invite) |
| `Games` (Games & Matchmaking) | game records (PGN + metadata), online games between approved friends, invitations, clocks, preset friendly messages, local/bot game uploads for coach analysis |
| `Ratings` | per-player rating (Glicko-2), separate pools for bot/online, rating history; kids' ratings shown as levels, not raw numbers |
| `Lessons` (Lessons & Progress) | learning path, chapters, lessons and steps (content authored in the admin app), per-child progress, stars, badges, streaks, daily challenge |
| `Puzzles` | puzzle bank (FEN, solution line, themes, rating), attempts, puzzle rating, daily puzzle selection |

Modules talk only through MassTransit events (e.g. `GameFinished` → Ratings + Lessons badges; `LessonCompleted` → rewards) or Refit clients, as the skill requires.

## Additions to the base skill

- **Real-time:** the `Games` host adds an ASP.NET Core **SignalR** hub (`/hubs/game`) with a Redis backplane. YARP routes WebSockets to it. The server is authoritative: every move received is validated by the server-side rules engine (tested against the same perft and position suites as `packages/chess_core`) before it is broadcast. Clients never trust each other.
- **No free-text chat:** the hub only accepts message ids from the `PresetMessage` list; any other payload is rejected.
- **Child accounts:** the auth host issues tokens with `role=child` or `role=parent` and the family id. Child tokens cannot reach parent-only endpoints (policy permissions, not UI hiding). Parent consent is checked before any child data is stored.
- **PDPL:** personal data in a Saudi/approved region, data export and delete endpoints per family, audit log of consent changes, retention limits on game chat-like events and logs.

## Commands (once scaffolded)

```
dotnet restore Faris.Chess.sln
dotnet build Faris.Chess.sln -warnaserror
dotnet test Faris.Chess.sln --collect:"XPlat Code Coverage"
```
