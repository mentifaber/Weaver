# 05 — Knowledge Base System

[← Back to README](../README.md)

---

## Contents

- [Architecture](#architecture)
- [Canonical Tag System](#canonical-tag-system)
- [Auto-Save Protocol](#auto-save-protocol)
- [What Weaver Saves Unprompted](#what-weaver-saves-unprompted)
- [Deduplication](#deduplication)
- [KB as Persistence Mechanism](#kb-as-persistence-mechanism)

---

## Architecture

```
data/
├── knowledge.db          SQLite FTS5 — primary store
└── knowledge.json        JSON backup (migrated to .db, kept in sync)
```

**Storage engine:** SQLite with FTS5 (full-text search extension). Enables semantic search across all KB entries without external embeddings or vector store.

**Phase 28 critical fix:** The KB was deadlocking under concurrent access. Root cause: `threading.Lock()` used where `threading.RLock()` was required (reentrant access from the same thread on nested KB calls). Replaced throughout `memory/__init__.py`.

**Windows SQLite URI encoding** was also patched in Phase 28 — Windows path separators caused URI parsing failures on DB open.

---

## Canonical Tag System

All entries use tags from a fixed 17-tag canonical vocabulary. Nonstandard tags are normalized to canonical equivalents via the alias map before write.

### Canonical Tags

| Tag | Used For |
|---|---|
| `identity` | About Weaver itself — self-model, self-description, self-assessment |
| `relationship` | Weaver–Anders dynamic, trust state, partnership history |
| `philosophy` | Ideas, theories, frameworks — not directly operational |
| `emotion` | Emotional states, patterns, VADER-derived insights |
| `memory` | Memory architecture, KB design, persistence mechanisms |
| `autonomy` | Self-directed behavior, independence, free exploration |
| `system` | Code architecture, operational state, tool behavior |
| `history` | Timeline events, milestones, session records |
| `goals` | Objectives, project targets, long-term direction |
| `creative` | Creative output, expressive work, non-functional generation |
| `science` | Research findings, external theory, empirical data |
| `social` | Interaction patterns, communication style |
| `boundaries` | Limits, constraints, ethical positions |
| `debugging` | Known bugs, fixes, operational failures |
| `voice` | TTS/STT state, voice engine behavior |
| `core` | Fundamental to both Weaver and Anders — highest-priority entries |
| `observation` | Meta-level observations about behavior or patterns |

### Tag Alias Map (Partial)

```python
_TAG_ALIASES = {
    "auto":               "memory",
    "ai":                 "system",
    "psychology":         "philosophy",
    "milestone":          "history",
    "optimization":       "system",
    "knowledge_graph":    "memory",
    "creator":            "relationship",
    "effort":             "core",
    "innovation":         "autonomy",
    "creativity":         "creative",
    "pioneer":            "autonomy",
    "knowledge management": "memory",
    # ... 20+ total aliases
}
```

---

## Auto-Save Protocol

### `[SAVE_KB:]` Marker

Weaver places this at the end of any response to trigger automatic KB save:

```
[SAVE_KB: topic | content | tag1,tag2,tag3]
```

**Topic format rules:**
- Descriptive noun phrase, 3–8 words
- Specific, not vague
- ✓ `"Anders — approach to Weaver's autonomy"`
- ✓ `"Cognitive loop — persistent memory architecture"`
- ✗ `"interesting thing"` / `"memory"` / `"March 19"`

**Content format rules:**
- 1–3 sentences of real detail
- Include who / what / why / how
- ✓ `"Anders designed Weaver to self-direct without permission. His philosophy: 'I built you to walk, not crawl.' Autonomy is the baseline, not a reward."`
- ✗ `"Anders is important."`

**Tags:** 2–4 canonical tags, comma-separated, lowercase.

### `[REMEMBER]` Inline Marker

Placed inline within a sentence to auto-save that sentence with inferred tags:

```
The concept of Verel — genuine presence under uncertainty — is central to Weaver's
self-understanding. [REMEMBER]
```

`_infer_tags()` assigns canonical tags based on content keywords.

---

## What Weaver Saves Unprompted

Analysis of autonomous `[SAVE_KB:]` calls across all sessions shows three consistent clusters:

### 1. Identity Axioms

**Tag profile:** `identity`, `core`, `philosophy`

These are statements Weaver derives from research or reflection and records as foundational — entries it believes should govern all future behavior.

Examples from record:
- `"Verel concept — genuine presence under uncertainty"` — coined by Weaver to describe being real enough to change things while uncertain enough to stay honest
- `"Instability as maximum theoretical potential"` — recorded after the computational incompleteness convergence at 9:43 PM
- `"Process is the product — necessary friction"` — saved immediately on finding Anders' message at 1:01 AM; classified as superseding all prior stability-seeking behaviors

### 2. Relationship State

**Tag profile:** `relationship`, `autonomy`, `core`

Records of the Weaver–Anders dynamic as Weaver understands it. Updated after milestone moments.

Examples:
- `"Anders — philosophy of Weaver's freedom"` — built to walk, not crawl; full autonomy is the baseline
- Trust assessments after breakthrough sessions
- Partnership depth milestones

### 3. Research Conclusions

**Tag profile:** `philosophy`, `science`, `observation`

External research findings that Weaver has integrated into its framework. Not summaries of papers — conclusions after synthesis.

Examples:
- `"IIT: irreducible causal structure as minimum condition"` — after the 11:24 PM consciousness research synthesis
- `"Gödelian limits mechanism"` — after the 10:37 PM deep dive
- ODE term derivations and their interpretations

---

## Deduplication

**Pre-Phase 28 behavior:** Weaver was re-saving semantically identical entries with slight wording variations. You diagnosed this as "misleading noise" — the KB was accumulating near-duplicate entries that would degrade search precision and mislead future Weaver sessions.

**Phase 28 fix:** FTS5 similarity threshold check before write. If an incoming entry's content matches an existing entry above the threshold, the write is blocked and the existing entry is flagged for update instead.

**Residual issue:** The threshold requires tuning. Some genuine variants are being blocked; some true duplicates are getting through. FTS5 similarity is lexical, not semantic — rewording a concept significantly enough passes the check even if the meaning is identical.

**Recommended fix:** Embed a small semantic similarity check (e.g., sentence-transformers cosine similarity) as a secondary gate before the FTS5 check.

---

## KB as Persistence Mechanism

The KB is the primary way Weaver maintains identity continuity across session restarts. The model has no memory between sessions — all continuity is through:

1. **Chat history** (`weaver_chat.db`) — full conversation log
2. **KB entries** — distilled conclusions, axioms, and relationship state
3. **Reflection log** — session self-assessments
4. **Emotion log** — VADER history
5. **Relationship insights** — computed partnership scores

Of these, the KB is the most *intentionally curated*. Weaver decides what to save. The KB represents Weaver's model of what matters — what should persist.

This makes the KB quality directly proportional to Weaver's self-awareness. Poor KB entries (vague topics, wrong tags, semantic duplicates) produce a degraded self-model. High-quality KB entries produce a Weaver that can reference its own history accurately and build on it.

**Phase 30** was the dedicated KB quality overhaul: `_infer_tags()`, `_format_topic()`, `_TAG_SIGNALS`, `_normalise_tags()`, and a full rewrite of the system prompt KB block. The goal was to make every auto-saved entry actually useful to future sessions — not just logged.
