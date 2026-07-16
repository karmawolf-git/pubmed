# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Single-file vanilla JS web app (`index.html`) for Viatris Korea medical representatives to search PubMed, translate abstracts to Korean, and generate AI-powered clinical implications and infographics.

Deployed via **GitHub Pages** on branch `claude/debug-and-fix-R2J0B`. All changes must be committed and pushed to that branch to take effect.

## Architecture

Everything lives in `index.html` (~1700 lines). Sections in order:

1. **CSS** (lines ~15–320) — CSS custom properties, component styles, responsive grid
2. **HTML** (lines ~321–480) — sidebar (search controls) + right panel (results)
3. **JS globals & keyword store** (lines ~480–650) — `CATS`, `kwStore`, `activeCategory`, IF lookup table
4. **Query builder** (lines ~655–713) — `buildQuery()` assembles PubMed search string with `[Title/Abstract]`, `[Author]`, `[Affiliation]`, date range, exclusions
5. **PubMed fetch layer** (lines ~715–755) — `fetchAbstractsBatch()` (efetch XML), `withRetry()`, `pLimit()` (inline concurrency limiter)
6. **API keys** (lines ~751–764) — `OR_KEY` and `GEMINI_KEY` stored as `atob("...")` — **never store as plaintext**
7. **AI layer** (lines ~765–1090):
   - `callGeminiBatch()` — parallel Gemini calls (pLimit 5), tries `GEMINI_MODELS` in order
   - `callOpenRouterBatch()` / `callOpenRouter()` — OpenRouter fallback
   - `buildImplication()` — Gemini-primary + OR fallback, used by infographic
   - `buildPrompt()` — quick mode (3 fields) vs full mode (all infographic fields, 2500 tokens)
   - `sanitizeImp()` — strips brand names and "선생님" from all AI output post-parse
   - `VIATRIS_CONTEXT` — system prompt context injected into every AI call
8. **Cache** (lines ~758–763) — localStorage key `pubmed_imp_v4`; bump version string to bust stale cache
9. **Rendering** (lines ~1090–1440) — `impHTML()`, `cardHTML()`, `buildInfographic()`, `igKeyStats()`, `igBarChart()`, `igTable()`
10. **`toggleInfog()`** (line ~1440) — lazy-loads full infographic data; `papers` must be global (`let papers=[]`) for this to work
11. **`search()`** (line ~1490) — main orchestration: esearch → esummary → efetch → translate → Gemini batch → render

## Key Invariants

- **`papers` is global** (`let papers=[]` at line ~991). It is assigned (not declared) inside `search()`. Never redeclare it with `const`/`let` inside `search()` — `toggleInfog()` accesses it from outside.
- **API keys**: always `atob("base64")` — GitHub push protection blocks plaintext secrets.
- **Cache version**: cache key is `pubmed_imp_v4`. Bump to `v5`, `v6`, etc. when prompt or schema changes require fresh AI calls.
- **Gemini API body**: use merged `contents:[{role:"user",parts:[{text:sys+"\n\n"+usr}]}]` — do **not** use `systemInstruction` field (causes failures on some models).
- **`esc()`**: uses `String(s==null?"":s)` — must handle numbers (Gemini returns numeric `key_stats.value`).

## AI Prompt Rules (in `VIATRIS_CONTEXT`)

- No product trade names in output (리피토®, 쎄레브렉스®, etc.)
- No "선생님" honorifics
- Frame findings positively — lead with the patient group that benefits, not who doesn't
- `sanitizeImp()` enforces brand-name removal client-side as a safety net

## Search Flow

```
buildQuery() → PubMed esearch → esummary (metadata) → efetch (abstracts XML)
  → parallel Korean translation (Google Translate, pLimit 5)
  → render cards (loading spinners for AI)
  → cache lookup → callGeminiBatch(needAI) → OR fallback → render implications
```

Infographic is lazy-loaded on demand via `toggleInfog()` → `buildImplication(..., 'full')`.

## Keyword Categories

Two categories (`cv`, `pain`) defined in `CATS` with presets and defaults. Active category stored in `activeCategory`. `kws()` returns the active keyword list.
