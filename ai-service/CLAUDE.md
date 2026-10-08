# ai-service/ — Faris coach (Python, FastAPI)

**Load this skill first:** `python-azure-rag-service`. Root `CLAUDE.md` hard rules apply, especially rule 2: **engine facts in, explanations out**.

Not scaffolded yet; follow the skill's §3 layout once the architecture (step 6) is approved.

## Product values for the skill placeholders

| Placeholder | Value |
|---|---|
| `{product}` | `faris` |
| Package | `src/ai_service/` (as in the skill) |
| Gateway prefix | `/ai-api/**` through the backend BFF |

## What this service does for Faris

1. **Coach explanations.** Input is a `MoveAnalysis` or `GameAnalysis` produced by the chess engine (best move, evaluation before/after, mistake class, tactical motif, threats, short principal variation, the player's age band, language and level). Output is a `CoachComment` (structured output, JSON schema in `llm/schemas.py`): a short explanation, an optional hint ladder (nudge → stronger hint → answer only on request), and the squares/arrows to highlight.
2. **RAG over Faris's own lesson content** (lesson texts, glossary, puzzle themes) so explanations use the same words and examples the child learned, with a link back to the lesson.

## Coach rules (overrides and additions to the skill)

- The LLM **never** proposes, evaluates or validates moves. Every move, square and evaluation it mentions must come from the input facts; a post-check rejects any output that names a move or square that is not in the facts, and falls back to a template.
- Templates first: common cases (hanging piece, missed fork, back-rank mate, good capture, castling) have hand-written Arabic and English templates. The LLM is used to adapt tone and combine facts, not to invent analysis.
- **Age bands** (6–8, 9–11, 12–14, teen/adult) control length (1–3 short sentences for the youngest), vocabulary and tone. Always kind: praise effort, never shame a mistake.
- **Arabic is Modern Standard Arabic, simple and warm**, with chess terms from the Faris glossary (e.g. شاه، وزير، فيل، حصان، رخ، بيدق، كش، كش مات). English is plain and friendly.
- **Hints, not answers** by default: the first hint names an idea ("look for a piece that is not protected"), the second narrows it ("your knight can attack two pieces"), the answer only when asked.
- **Safety:** child-facing output goes through content safety; there is no free-text chat with the coach for child accounts in the MVP (only buttons such as "why?", "hint", "show me"). No personal data is sent to the model (nickname, age band and level only).
- **Cost:** cache explanations by (position hash, move, mistake class, language, age band); templates cover most calls without an LLM.
- Evaluation sets (`evals/datasets/faris_{ar,en}.jsonl`) include factual-accuracy checks against the engine facts, tone checks per age band and Arabic quality reviewed by a native speaker.

## Commands (once scaffolded)

```
uv sync
uv run ruff check . && uv run ruff format --check .
uv run mypy --strict src
uv run pytest
```
