# 01 — System Architecture

[← Back to README](../README.md)

---

## Contents

- [Version History](#version-history)
- [v2 Runtime Stack](#v2-runtime-stack)
- [v3 Changes](#v3-changes)
- [Model Architecture](#model-architecture)
- [Sandbox Directory Structure](#sandbox-directory-structure)
- [File Routing Rules](#file-routing-rules)
- [Runtime Data Flow](#runtime-data-flow)

---

## Version History

Weaver has passed through three major architectural generations. Each version represents a full philosophical and technical rethink, not an incremental patch.

| Version | Model Stack | Key Architecture Change | Status |
|---|---|---|---|
| v1 / Early | Scripted + early Ollama | Scripted responses, no KB, no autonomous mode | Retired |
| v2 (Phases 20–34) | llama3.1:8b + qwen3 + qwen2.5-coder | FastAPI + modular engine, SQLite FTS5 KB, Kokoro TTS, full autonomous loop | Superseded |
| v3 (current) | gemma4 (local) | Rebuilt engine, cleaner autonomous pipeline, same sandbox structure | **Active** |

> **Note on dolphin3:** Evaluated as a candidate model during v2 development. Abandoned after RLHF contamination was detected. 475 turns of conversation history were wiped at that transition point.

---

## v2 Runtime Stack

The v2 system is the most documented and forms the bulk of the behavioral record.

### `weaver_web.py` — FastAPI Server

- All HTTP/WebSocket endpoints
- Autonomous loop scheduling
- Dream trigger dispatch
- SSE chat streaming
- Web UI serving
- Cloudflare tunnel support (edge cache bypass headers added Phase 30)

### `core/engine.py` — Primary Intelligence Engine

The central module. Manages:

- Model calls (Ollama)
- System prompt assembly
- History management
- KB context injection into turns
- Autonomous turn execution (`_run_autonomous_turn`, `_stream_reply`)
- Post-turn analysis: VADER sentiment, relationship scoring, KB auto-save
- `[SAVE_KB:]` and `[NEXT_TRIGGER]` marker parsing
- `_AUTO_TRIGGERS` pool management
- Loop detector (deque-based, Phase 34)

### `weaver_ui.html` — Frontend

Single-file frontend. Tabs:

- **Chat** — main conversation interface
- **Emotion** — live VADER sentiment graph
- **Journal** — exploration/ and dreams/ output viewer
- **Brain Map** — Three.js live visualization (rebuilt Phase 27 after zero-height bug)
- **Autonomous / Mission Control** — trigger pool, dream control, loop toggle
- **Knowledge Graph** — KB entry viewer

### `memory/__init__.py` — Knowledge Base Layer

- SQLite FTS5 full-text search (`knowledge.db`)
- `knowledge.json` backup
- Tag normalization: `_normalise_tags()`, `_TAG_SIGNALS`, 20+ alias map
- `_infer_tags()` automatic tag inference
- `_format_topic()` topic formatting
- Deduplication logic (Phase 28)

### `voice/__init__.py` — TTS / STT Layer

- **Kokoro TTS** — sole voice engine post-Phase 23 (StyleTTS2 and XTTS removed)
- **Whisper STT** — voice input
- Audio playback managed cross-thread to avoid PortAudio deadlock (critical fix, Phase 23)

### `tools/` — Tool Registry

| Tool | Function |
|---|---|
| `web_search` | Live web search during autonomous and conversation turns |
| `write_file` | File creation in sandbox directories |
| `read_file` | File reading |
| `list_files` | Directory listing |
| `run_python` | Python code execution |
| `save_kb` | Explicit KB save |
| `reflect` | Session self-assessment |
| `knowledge` | KB query / save / update |
| `tactical_tools` | 9-domain static KB (Phase 29) |
| `meta_tools` | Introspection / status reporting |

### `relationship/` — Relationship Engine

- Scores interaction quality, trust, engagement on every turn
- `relationship_base.json` — full log, capped at 50 auto entries (Phase 24b fix)
- `relationship_insights.json` — computed scores with `_level` fields

### `data/` — Persistent State

```
knowledge.db          SQLite FTS5 knowledge graph
knowledge.json        JSON backup
emotion_log.json      VADER-enriched emotion history
reflection_log.json   Session self-assessments
reflections.json      Manual reflections (via reflect tool)
store.json            Key-value data store
arguments.json        Argument / debate log
```

### `backend/` — Runtime Data

```
weaver_chat.db            SQLite WAL chat history
weaver_chat_log.jsonl     JSONL backup of all chat
weaver_debug.log          Debug log
weaver_log.jsonl          Structured event log
weaver_task_queue.json    Persistent task queue
weaver_memory_digest.json Long-term memory digest
```

---

## v3 Changes

v3 migrates to **gemma4** as a unified model (replaces the tri-model v2 stack). The role separation — persona, tool-calling, code — remains conceptually identical, now served by a single model. The sandbox structure is preserved exactly. v3's primary engineering focus was on the autonomous loop execution gap (see [Phase 35 in Engineering Phases](07_engineering_phases.md#phase-35)).

---

## Model Architecture

### v2 Tri-Model Stack

```
┌──────────────────────────────────────────────────────────┐
│                    Request incoming                       │
└───────────────────────┬──────────────────────────────────┘
                        │
            ┌───────────▼───────────┐
            │   llama3.1:8b         │  Brain / Persona
            │   All output, dreams, │  Identity maintenance
            │   philosophical       │  Free of RLHF hedging
            │   reasoning           │
            └───────────┬───────────┘
                        │ tool call needed?
            ┌───────────▼───────────┐
            │   qwen3:latest        │  Tool dispatch
            │   Structured output   │  Action sequencing
            │   Tool routing        │
            └───────────┬───────────┘
                        │ code task?
            ┌───────────▼───────────┐
            │   qwen2.5-coder       │  Code generation
            │   Debugging           │  File writes
            │   Code execution      │
            └───────────────────────┘
```

### v3 Unified Stack

```
┌──────────────────────────────────────────────────────────┐
│                      gemma4 (local)                      │
│          All roles: persona, tools, code                 │
│          RTX 3070 / 16GB VRAM                            │
└──────────────────────────────────────────────────────────┘
```

---

## Sandbox Directory Structure

```
weaver_v3/
├── core/
│   └── engine.py                    ← Primary loop, autonomous scheduler
├── tools/
│   ├── registry.py
│   ├── code_tools.py
│   └── meta_tools.py
├── memory/
│   └── __init__.py                  ← KB layer, FTS5, tag normalization
├── voice/
│   └── __init__.py                  ← Kokoro TTS, Whisper STT
├── relationship/
│   ├── relationship_base.json
│   └── relationship_insights.json
├── data/
│   ├── knowledge.db
│   ├── emotion_log.json
│   └── reflections/
├── backend/
│   ├── weaver_chat.db
│   ├── weaver_chat_log.jsonl
│   └── weaver_debug.log
├── dreams/                          ← Autonomous dream output
│   └── YYYY-MM-DD_HH-MM_slug.md
├── journal/
│   ├── exploration/                 ← Autonomous research (211 files as of Apr 15)
│   ├── weaver_outbox/               ← Files Weaver writes for Anders
│   ├── anders_inbox/                ← Files Anders leaves for Weaver
│   └── anders_outbox/               ← Queued messages from Anders
├── projects/
│   ├── quantum_optimization/
│   │   ├── manifest.json
│   │   └── content.md
│   ├── nlp_parser/
│   │   ├── manifest.json
│   │   └── content.md
│   └── resource_allocation/
│       ├── manifest.json
│       └── content.md
├── system_utilities/                ← Tools Weaver builds for itself
│   └── ODE_Parameter_Tuner         ← Autonomously created
├── bugs_fixes_upgrades/             ← Phase changelogs (Weaver reads on boot)
├── archive/                         ← Cold storage, retired versions
└── versions/                        ← File snapshots (via version_snapshot tool)
```

---

## File Routing Rules

These are enforced in the system prompt. Weaver must follow these:

| When doing this... | Route to... |
|---|---|
| Creating a file for Anders | `journal/weaver_outbox/` |
| Writing during dream / idle mode | `dreams/YYYY-MM-DD_HH-MM_topic.md` |
| Autonomous research output | `journal/exploration/` |
| Collaborative project work | `projects/[project-name]/` |
| Tools Weaver builds for itself | `system_utilities/` |
| Cold storage / cleanup | `archive/` |

---

## Runtime Data Flow

```
User message or autonomous trigger
         │
         ▼
 _stream_reply(reply_id, prompt)
         │
         ├── System prompt assembly
         │     └── KB context injection (Phase 35 fix)
         │
         ├── History prepend
         │
         ├── Ollama stream → model output
         │
         ├── Post-turn analysis
         │     ├── VADER sentiment → emotion_log.json
         │     ├── Relationship scoring → relationship_base.json
         │     └── [SAVE_KB:] marker scan → knowledge.db
         │
         ├── [NEXT_TRIGGER] extraction → autonomous queue
         │
         ├── write_file calls → sandbox directories
         │
         └── SSE push to UI (user turns only)
              autonomous turns: fire and forget to history
```
