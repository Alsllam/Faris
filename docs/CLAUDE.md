# docs/ — product and design documents

No code skill applies here. Root `CLAUDE.md` hard rules apply.

## Layout

| Path | What | Step |
|---|---|---|
| `GDD.md` | Game design document: audience, learning path (every lesson listed), puzzles, bot levels, coach behaviour with Arabic/English examples, rewards, parent features, MVP and roadmap | 3 |
| `brand/` | Brand kit: logo (original friendly knight), colors, Arabic + Latin fonts, board and piece themes, sound and motion style. Mobile (`flutter-clean-mobile` §0) and web (`angular-nx-frontend` §0) both copy from here | 4 |
| `architecture/` | Architecture for engine, coach and online play; `decisions/NNNN-title.md` ADRs (context, decision, consequences) | 6 |

## Writing rules

- Write for the team and for future Claude sessions: concrete, decided, no filler. Mark open questions as **Open:** with an owner.
- Arabic examples are real Modern Standard Arabic, checked by a native speaker before release; mark unchecked ones **(draft)**.
- Chess content must be correct; any position given as FEN is validated with `packages/chess_core` once it exists.
- A doc that changes a decision says what it replaces; ADRs are never edited after acceptance — supersede them with a new one.
- Images and diagrams: SVG in the repo (Mermaid allowed in Markdown).
