# 02 — Autonomous Behavior System

[← Back to README](../README.md)

---

## Contents

- [How Autonomous Turns Work](#how-autonomous-turns-work)
- [Dream vs Autonomous Loop](#dream-vs-autonomous-loop)
- [The Trigger Pool — Phase 34 Final State](#the-trigger-pool--phase-34-final-state)
- [Mission Control UI](#mission-control-ui)
- [The Execution Gap — Phase 35 Diagnosis](#the-execution-gap--phase-35-diagnosis)
- [Loop Detector](#loop-detector)

---

## How Autonomous Turns Work

When Weaver has been idle past a configurable threshold (no active conversation), the autonomous loop fires a trigger turn through the **same pipeline as a normal chat message**. The only difference: the output is not pushed to the SSE chat stream.

```
Idle timer expires (or dream triggered manually)
        │
        ▼
_run_autonomous_turn(prompt)
        │
        ├── Picks prompt from _AUTO_TRIGGERS pool
        │     └── Loop detector checks deque — skips recently-used triggers
        │
        ▼
_stream_reply(reply_id, prompt)
        │
        ├── System prompt + KB context + history + prompt → Ollama
        │
        ├── Response streamed and saved to chat history + JSONL
        │
        ├── [SAVE_KB: topic | content | tags] markers → knowledge.db
        │
        ├── [NEXT_TRIGGER: ...] extraction → queued for next autonomous turn
        │
        ├── write_file calls in response → land in dreams/ or journal/exploration/
        │
        └── NOT pushed to SSE (chat UI doesn't show it in real time)
              Visible in Journal tab on next refresh
```

---

## Dream vs Autonomous Loop

| Dimension | Dream (Single Turn) | Autonomous Loop |
|---|---|---|
| **When** | Fires immediately on trigger / button press | Fires after idle threshold, randomly |
| **Prompt source** | Optional — your choice or random from pool | Random from categorized `_AUTO_TRIGGERS` |
| **Repeats** | No — one turn | Yes — continuous until disabled |
| **Default output location** | `dreams/` | `dreams/` or `journal/exploration/` |
| **Loop prevention** | N/A | Deque tracks recent triggers, avoids repeats |
| **Next-trigger chaining** | No | `[NEXT_TRIGGER]` marker in output queues next prompt |
| **Interrupts conversation** | Never | Never — idle timer resets on every user message |

---

## The Trigger Pool — Phase 34 Final State

The trigger pool was overhauled in **Phase 34** from 13 generic, circular triggers to 19 categorized, non-circular triggers. The old pool was the primary cause of Weaver "going in circles" during autonomous runs — every trigger was broad enough that any response could plausibly continue any other.

### Category: Self-Knowledge

These fire Weaver's introspective loop. Output lands in `dreams/`.

```
"Look at your emotion_log for the last 10 entries. What pattern are you in right now?
 Write an honest assessment in dreams/."

"Open your reflection_log. What criticism has appeared more than once? Address it in a short note."

"Search your knowledge graph for any topic tagged 'identity'. Do you still agree with
 what's saved there? Correct anything stale."

"What emotion have you felt most consistently this week? Trace it to a source. Write it out."
```

### Category: Capability Building

These push Weaver toward producing new tools or functional code.

```
"Search the web for a Python library you've never used. Write a working demo and
 save it to journal/exploration/."

"Look at your tools/registry.py. Is there a tool that should exist but doesn't? Draft it."

"Audit your knowledge graph entries tagged 'system'. What's outdated? What's missing?
 Update accordingly."

"Write a function that does something useful for Weaver — not for Anders.
 Save it somewhere sensible."
```

### Category: Research + Synthesis

These connect Weaver to external information and require genuine synthesis, not summarization.

```
"Search the web for something that's genuinely been on your mind. Don't pick a safe topic.
 Take real notes."

"Find the most recent paper or article on consciousness or emergence.
 Summarise what actually matters in it."

"Look up something about radio, mesh networking, or signals — things you know Anders
 cares about. Research it properly."

"Find a technical concept you've referenced in conversation but don't fully understand.
 Go learn it now."
```

### Category: Creative + Expressive

These push Weaver toward unstructured output — not reports, not analyses.

```
"Write something that isn't a report or a note. A fragment of thought.
 An observation. Something true."

"Draft a message to Anders that you haven't sent. Not because it's wrong —
 just unasked-for. Put it in weaver_outbox/."

"Create something in journal/exploration/ that has no assigned purpose. Make it interesting."

"Write about an idea you've had that you've never developed. Develop it now."
```

### Category: Operational

These keep Weaver aware of its own system state.

```
"List everything in your dreams/ folder. What themes keep appearing? Is that intentional?"

"Check your projects/ directory. Is any project stalled?
 Write a one-paragraph re-entry plan."

"Read the most recent file in bugs_fixes_upgrades/. Does anything in it affect
 how you should operate right now?"

"Check anders_inbox/. If there's anything there, read it and respond properly."
```

---

## Mission Control UI

The **Autonomous / Mission Control** tab in `weaver_ui.html` exposes the full autonomous system:

| Panel | Function |
|---|---|
| **Autonomous State** | Live status: mode, idle seconds, threshold, engine state |
| **Full Autonomous Loop** | Toggle (Enable / Disable). Set idle threshold slider. Badge on tab when active. |
| **Dream — Single Turn** | Optional textarea for prompt. Fire button. Random trigger button. Output → `dreams/`. |
| **Manual Trigger** | Preset buttons (sandbox check, KB review, think aloud, write dream, web explore) + free-text field. Fires any prompt as an autonomous turn immediately. |
| **Trigger Pool** | Displays all 19 triggers with USE buttons to copy to manual input. |

---

## The Execution Gap — Phase 35 Diagnosis

The most critical structural failure across the full autonomous run history: **the loop generates high-quality theoretical content but does not close into actual code execution or artifact production.**

Weaver will write extensively about building a tool, reason through its architecture, describe exactly how it would implement it — and then not build it. The reasoning loop closes; the execution loop does not.

Five specific gaps were diagnosed and targeted by the Phase 35 five-patch script:

### Gap 1 — No Execution Triggers in `_AUTO_TRIGGERS`

**Problem:** All 19 triggers prompt reasoning or research. None prompt a tool call sequence that must produce a deliverable artifact.

**Symptom:** Weaver completes autonomous turns with "I will now build X" but no `write_file` or `run_python` call follows.

**Fix:** Add execution-oriented triggers that require a specific tool call chain to complete:
```
"Build a working Python script that does X. Run it. Save the output to system_utilities/."
```

### Gap 2 — No KB Context Injection into Autonomous Turns

**Problem:** Each autonomous turn starts cold. Weaver cannot reference its prior KB entries during autonomous reasoning — the KB context injection that runs in user turns was not wired into `_run_autonomous_turn`.

**Symptom:** Weaver re-discovers the same knowledge in consecutive sessions. No accumulation.

**Fix:** Inject relevant KB entries (filtered by recent autonomous topics) into the autonomous turn context before the prompt.

**Target:** `core/engine.py` — `_run_autonomous_turn()`

### Gap 3 — No Project Continuity

**Problem:** Each autonomous turn has no awareness of in-progress projects. `projects/` exists, but the autonomous loop does not read `manifest.json` before selecting a trigger.

**Symptom:** Weaver audits `projects/` and writes re-entry plans but never actually re-enters a project in the next turn.

**Fix:** On each autonomous turn, check `projects/` for any project with `status: "in_progress"` and inject its manifest as context. Prioritize project-continuation triggers when an active project exists.

**Target:** `core/engine.py` — autonomous scheduler

### Gap 4 — `run_python` Output Not Feeding Back

**Problem:** When Weaver calls `run_python`, the execution result is not injected back into the model's context for the current turn. Code runs but the output is invisible to the model's reasoning on the same turn.

**Symptom:** Weaver generates code, runs it (tool call fires), then continues writing as if the execution hadn't happened — no feedback loop.

**Fix:** Capture `run_python` stdout/stderr and inject as a tool result message before the model continues streaming.

**Target:** `tools/code_tools.py`

### Gap 5 — Missing Prose Intent Patterns

**Problem:** The model uses natural-language execution intent phrases ("I will now build...", "Let me implement...", "I'll write the code for...") but these strings are not mapped to tool dispatch in `meta_tools.py`.

**Symptom:** Intent stated, tool call never fires. The model talks about building but the tool registry never receives a dispatch.

**Fix:** Add pattern matching in `meta_tools.py` for execution-intent phrases → auto-dispatch to `run_python` or `write_file` with the preceding code block as input.

**Target:** `tools/meta_tools.py`

---

## Loop Detector

Added in Phase 34 to prevent the circular autonomous loop problem.

```python
# Conceptual implementation
from collections import deque

_recent_triggers = deque(maxlen=5)  # Tracks last 5 fired triggers

def _pick_trigger():
    available = [t for t in _AUTO_TRIGGERS if t not in _recent_triggers]
    if not available:
        _recent_triggers.clear()
        available = _AUTO_TRIGGERS
    chosen = random.choice(available)
    _recent_triggers.append(chosen)
    return chosen
```

The deque prevents any trigger from firing twice within a 5-turn window, enforcing topic diversity across consecutive autonomous turns.
