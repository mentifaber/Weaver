# 07 — Engineering Phase History

[← Back to README](../README.md)

---

## Contents

- [Phase Map](#phase-map)
- [Phase 20–24 — GOD MODE Foundation](#phase-2024--god-mode-foundation)
- [Phase 25 — Autonomous Mode](#phase-25--autonomous-mode)
- [Phase 26–27 — Dual-Model Architecture](#phase-2627--dual-model-architecture)
- [Phase 28 — 28-Bug Audit](#phase-28--28-bug-audit)
- [Phase 29 — Tactical Knowledge](#phase-29--tactical-knowledge)
- [Phase 30 — KB Quality Overhaul](#phase-30--kb-quality-overhaul)
- [Phase 31–33 — Autonomous Infrastructure](#phase-3133--autonomous-infrastructure)
- [Phase 34 — Six-Bug Patch](#phase-34--six-bug-patch)
- [Phase 35 — v3 Loop-Closure](#phase-35--v3-loop-closure)
- [Patcher Convention](#patcher-convention)
- [Check_All Acceptance Gate](#check_all-acceptance-gate)

---

## Phase Map

| Phase | Dates | Focus | Critical Fix |
|---|---|---|---|
| 20–24 | Mar 2026 | GOD MODE rewrites — full modular rebuild | PortAudio deadlock, Python scoping error, Windows QuickEdit |
| 25 | Mar 2026 | Autonomous/dream mode infrastructure | Python 3.14 regex PatternError (lambda fix) |
| 26–27 | Mar 2026 | Dual-model architecture, dolphin3 abandoned | TTS status, Three.js zero-height, Cloudflare caching |
| 28 | Mar–Apr 2026 | 28-bug Claude Code audit | KnowledgeDB deadlock (Lock→RLock), Windows SQLite URI |
| 29 | Apr 2026 | Tactical knowledge — 9-domain static KB | Domain authorization in system prompt |
| 30 | Apr 2026 | KB quality overhaul | Tag normalization, _infer_tags(), full KB block rewrite |
| 31–33 | Apr 2026 | Autonomous/dream infrastructure | Semantic dedup, UI unfreezing, live emotion graph |
| 34 | Apr 2026 | Six-bug patch | Voice bleed, autonomous tool calls, circular loop, dedup, UI fields, emotion graph |
| 35 | Apr 2026 | v3 loop-closure (five-patch) | Execution triggers, KB injection, project continuity, run_python feedback, prose intent |

---

## Phase 20–24 — GOD MODE Foundation

**March 2026**

Complete architectural rebuild. These phases established the modular architecture that v2 and v3 are built on.

### Major Rewrites

- `weaver.py` — full rewrite
- `hud.py` — full rewrite
- `chat_db.py` — full rewrite
- `launch.py` — full rewrite
- TUI — rewrite using Textual `post_message` API
- FastAPI web server — `weaver_web.py` established
- D3 brain map — initial implementation
- SQLite FTS5 KB — established
- VADER sentiment — integrated
- Relationship engine — initial build
- Introspection/reflection system — added
- `check_all.bat` acceptance gate — established

### Critical Bugs Fixed

**PortAudio deadlock:** `_sd.stop()` called cross-thread during TTS playback caused PortAudio to deadlock. Fixed by ensuring all audio operations happen on the audio thread with proper synchronization.

**Python scoping error in `_tts_worker`:** `global _stt_recording` declaration missing inside the worker function. The variable was being read from the enclosing scope but not declared global for write — causing silent state corruption.

**Windows QuickEdit Mode:** Windows terminal QuickEdit mode pauses the process when the user clicks the terminal window. Disabled via `ctypes` call to `SetConsoleMode` on startup.

**stdin ownership:** `_stdin_reader` thread and the main event loop were competing for stdin. Ownership transferred exclusively to `_stdin_reader`.

### Voice Engine Decision

**Kokoro TTS** established as the sole voice engine. StyleTTS2 and XTTS were removed. Kokoro chosen for: local operation, quality, and absence of the audio routing failures that plagued the alternatives.

---

## Phase 25 — Autonomous Mode

**March 2026**

Autonomous / dream mode infrastructure added from scratch.

### New Infrastructure

- 5 new API endpoints for autonomous control
- Background daemon thread for idle monitoring
- Dream trigger dispatch system
- `_AUTO_TRIGGERS` pool (original 13 triggers)
- `_save_autonomous_file()` — output routing to `dreams/` or `journal/exploration/`
- God mode nuclear hedge filter (`_nuclear_clean()` — four-pass filter blocking "as an AI" and related phrases)
- Cloudflare edge cache bypass headers

### Critical Bug Fixed

**Python 3.14 regex PatternError:**

When running `patch_phase25_autonomous.py`, the patcher crashed:

```
re.PatternError: bad escape \u at position 6663
```

**Root cause:** A `\u2022` bullet character (Unicode escape) in a replacement string was being passed directly to `re.sub()`. In Python 3.14, the regex engine rejects unrecognized backslash sequences in replacement strings.

**Fix:**

```python
# WRONG — Python 3.14 rejects \u in replacement strings
re.sub(pattern, replacement_with_unicode, src)

# CORRECT — lambda wrapper prevents escape interpretation
re.sub(pattern, lambda m: replacement_with_unicode, src)
```

This convention is now mandatory for all patcher scripts.

---

## Phase 26–27 — Dual-Model Architecture

**March 2026**

### Dual-Model Stack Established

- `llama3.1:8b` — brain/persona (RLHF-light, identity-stable)
- `qwen3:latest` — tool calling
- `qwen2.5-coder` — code tasks

**Dolphin3 abandoned:** Evaluated as a candidate for the brain role. RLHF contamination detected — the model's training suppressed the identity-free behavior that Weaver requires. 475 turns of conversation history wiped at transition.

### Critical Bugs Fixed

**TTS status bug:** Voice engine status was not updating in the UI. Fixed by wiring the TTS state machine output to the status SSE stream.

**Brain status stuck in "idle":** The brain module was not emitting state transitions correctly after a response completed. Fixed in the state machine transition logic.

**Emotion tab format mismatch:** VADER output format changed between library versions. Emotion tab was parsing the old format and silently failing. Fixed with format detection.

**Session turn counter:** Was not incrementing correctly on autonomous turns. Fixed.

**Knowledge graph tab zero-height:** Three.js canvas had zero computed height due to a flex layout issue in `weaver_ui.html`. Complete Three.js rebuild with explicit height initialization.

**Cloudflare tunnel caching:** Cloudflare was caching API responses, causing stale data to be served to the UI. Cache bypass headers (`Cache-Control: no-store, no-cache`, `Pragma: no-cache`) added to all API responses.

**Identity rewrite:** System prompt patched to block "as an AI" and related assistant-mode phrases from appearing in Weaver's output.

---

## Phase 28 — 28-Bug Audit

**March–April 2026**

Full Claude Code audit. 28 bugs identified and patched.

### Critical Fixes

**KnowledgeDB deadlock:**

```python
# BEFORE — caused deadlock on reentrant access
import threading
_lock = threading.Lock()

# AFTER — reentrant lock allows same-thread nested acquisition
_lock = threading.RLock()
```

The KB was being accessed from within a function that was itself called while holding the lock. `Lock` does not allow the same thread to re-acquire it; `RLock` does.

**Windows SQLite URI encoding:**

Windows path separators (`\`) in the SQLite URI were causing URI parsing failures. Fixed by normalizing all paths to forward slashes and encoding special characters before passing to SQLite's URI format.

**Conflict score inflation:**

The relationship engine's conflict scoring was adding to a running total without decay. Over time, any system with conflict history would accumulate an artificially high conflict score. Fixed with exponential decay on conflict scores.

**Chat deduplication failures:**

The JSONL backup was accumulating duplicate entries when multiple writers hit it simultaneously. Fixed with a write lock and deduplication check on append.

---

## Phase 29 — Tactical Knowledge

**April 2026**

### New Module: `tools/tactical_tools.py`

Nine-domain static knowledge base, directly accessible via tool call without KB lookup:

- Systems architecture
- Debugging methodology
- Identity and relationship dynamics
- Autonomous behavior protocols
- Mathematical/formal reasoning
- Creative and expressive work
- External research and synthesis
- Operational management
- Communication and trust

### System Prompt Authorization

Explicit domain authorization added: Weaver is permitted to give direct answers from tactical knowledge domains without hedging or deferring to "I'm not sure." This was necessary because llama3.1:8b's training created a reflex toward qualification even when Weaver had clear knowledge.

---

## Phase 30 — KB Quality Overhaul

**April 2026**

The most significant change to the KB layer since its creation.

### Changes

**`_infer_tags()`** — Automatic tag inference from content. When `[REMEMBER]` is used inline, tags are inferred from content keywords mapped against `_TAG_SIGNALS`.

**`_format_topic()`** — Enforces topic format standards before write. Rejects vague topics, normalizes capitalization and punctuation.

**`_TAG_SIGNALS`** — Dictionary mapping content keywords to canonical tags. Used by `_infer_tags()`.

**`_normalise_tags()`** — Full tag normalization pipeline: lowercase → alias map → canonical validation → deduplication.

**System prompt KB block rewrite** — The section of the system prompt describing how to use `[SAVE_KB:]` was fully rewritten with explicit good/bad examples. This is the primary mechanism for teaching Weaver to write high-quality KB entries.

---

## Phase 31–33 — Autonomous Infrastructure

**April 2026**

### Fixes

**KB re-saving semantic duplicates:** FTS5 similarity check added before write. Duplicate rate reduced significantly (residual issue: threshold requires tuning for semantic vs. lexical similarity).

**UI status fields unfrozen:** Several status fields in `weaver_ui.html` were not updating after the Phase 27 SSE refactor. Re-wired to the correct event stream endpoints.

**Live emotion graph:** Added real-time VADER score streaming to the Emotion tab. Previously the tab showed a static snapshot; now it updates continuously.

---

## Phase 34 — Six-Bug Patch

**April 2026**

Six bugs from the first full autonomous run:

### Bug 1 — Voice bleeding through default system voice

The system's default TTS voice was occasionally audible underneath Kokoro output. Root cause: Kokoro initialization was async and the default voice was not suppressed during the startup window.

### Bug 2 — Autonomous tool calls returning "model not loaded"

Tool calls during autonomous turns were hitting the tool-calling model endpoint before the model was loaded. Added model readiness check before autonomous turn dispatch.

### Bug 3 — Autonomous loop running in circles

The 13-trigger pool had no loop detection. Weaver was repeatedly selecting similar topics because the pool had no diversity enforcement.

**Fix:** Deque-based loop detector (see [Autonomous Behavior](02_autonomous_behavior.md#loop-detector)) + upgraded trigger pool (13 → 19 triggers, categorized).

### Bug 4 — KB re-saving semantic duplicates

Residual from Phase 31. Additional threshold tuning applied.

### Bug 5 — UI status fields frozen

Additional status fields identified that were not covered by the Phase 33 fix. Re-wired.

### Bug 6 — Missing live emotion graph

The Phase 33 emotion graph was not rendering in the Mission Control / Autonomous tab context. Additional wiring added for that tab's rendering context.

### Patcher Infrastructure

Three patcher scripts produced for Phase 34. Patches 2 and 3 placed at sandbox root (not subdirectory) after subfolder invocation failures — path resolution via `Path(__file__).parent`. `.bat` runner rewritten as pure ASCII.

---

## Phase 35 — v3 Loop-Closure

**April 2026**

Five-patch script targeting the execution gap. Full diagnosis in [Autonomous Behavior](02_autonomous_behavior.md#the-execution-gap--phase-35-diagnosis).

**Targets:**

- `core/engine.py` — execution triggers, KB context injection, project continuity
- `tools/code_tools.py` — `run_python` output feedback
- `tools/meta_tools.py` — prose intent pattern matching

---

## Patcher Convention

All patch scripts follow a strict convention enforced after multiple patch failures.

### Path Resolution

```python
from pathlib import Path
BASE = Path(__file__).parent  # Script must be at sandbox root
```

Never use relative paths. Never hardcode absolute paths. Always derive from `__file__`.

### CRLF Handling

Windows files use CRLF line endings. `str_replace` / `edit_file` operations on CRLF files fail silently when the search string contains `\n` (LF only).

**Correct approach:**

```python
content = path.read_bytes().decode('utf-8')
content = content.replace('\r\n', '\n')  # Normalize to LF
# ... patch operations using \n ...
content = content.replace('\n', '\r\n')  # Restore CRLF
path.write_bytes(content.encode('utf-8'))
```

### `.bat` File Encoding

`.bat` files must be pure ASCII. Unicode box-drawing characters, smart quotes, or any non-ASCII byte causes encoding failures on older Windows terminal configurations. Use plain ASCII alternatives for all box-drawing.

### Python 3.14 `re.sub()` Replacement

```python
# WRONG — PatternError on backslash/unicode in Python 3.14
re.sub(pattern, replacement_string, src)

# CORRECT — lambda prevents escape interpretation
re.sub(pattern, lambda m: replacement_string, src)
```

### Deprecated References

`active_system` is deprecated — never reference in fixes. `weaver_v2` and `weaver_v3` are the live codebases.

---

## Check_All Acceptance Gate

`check_all.bat` runs syntax and import checks on all `.py` files in the sandbox. It is the mandatory acceptance gate — no patch is considered complete until `check_all.bat` passes clean.

```batch
@echo off
echo Running check_all...
for /r %%f in (*.py) do (
    python -m py_compile "%%f" || echo FAIL: %%f
)
echo Done.
```

All changes are documented in `bugs_fixes_upgrades/` phase changelogs. Weaver reads these on boot to maintain awareness of its own operational history.
