# Hermes Deep-Dive Source Analysis

> **Scope:** This document is a code-level dissection of the Hermes agent memory system, based on direct reads from the open-source repositories `hermes-agent-main` and `hermes-agent-self-evolution-main`. It is intended as a concrete reference for the gspd memory-engine design questions (layering, context compression, dreaming/offline evolution, and known defects).

---

## 1. Executive Summary: What Hermes Actually Built

Hermes does **not** implement a classical hierarchical cache (L0→L1→L2→L3). Instead it uses a **dual-track** architecture:

| Track | Components | Design Philosophy |
|-------|------------|-------------------|
| **Online / In-Context** | `MEMORY.md` + `USER.md` (frozen snapshot), Skills index + `skill_view`, External provider prefetch | Keep the online pipeline **minimal** and **fast**. Trust the LLM to judge what to save. |
| **Offline / Out-of-Band** | SQLite session archive + `session_search`, Gateway memory flush, GEPA skill evolution, External provider sync (Hindsight, Mem0, etc.) | Push heavy optimisation **offline**. Heavy extraction, synthesis, and mutation happen outside the critical path. |

This is the **opposite** of systems like MemOS or AutoSkill, which build heavy online extraction pipelines. Hermes intentionally trades "instant perfect memory" for "simple, cache-friendly, cheap online + powerful offline".

---

## 2. The Four Storage Tiers (Mapped to gspd Layer Questions)

### Tier 1: Prompt Memory — `MEMORY.md` & `USER.md`
**Files:** `tools/memory_tool.py`, `agent/prompt_builder.py`

- **Capacity:** Hard character limits (not tokens) — `MEMORY.md` ≈ **2,200 chars**, `USER.md` ≈ **1,375 chars**.
- **Format:** Entries delimited by `§` (section sign). Multi-line entries allowed.
- **CRITICAL — Frozen Snapshot Design:**
  - Loaded **once** at session start into the system prompt.
  - Mid-session writes (`memory` tool `add`/`replace`/`remove`) update disk immediately (atomic temp-file + rename) but **do NOT** mutate the in-flight system prompt.
  - **Why:** Preserves the prefix cache for the entire session. On Anthropic/OpenAI cached providers this is a massive cost saving.
  - **Trade-off:** New memories written mid-session are **not visible** to the agent until the next session starts. The agent only sees them via the tool's JSON response.
- **Update semantics:** Sub-string matching for `replace`/`remove` (e.g. old_text=`"dark mode"` matches an entry containing `"User prefers dark mode..."`). If multiple non-identical entries match, the tool errors and asks for more specificity.
- **Pressure as a feature:** At ~80% capacity the `add` action returns an error forcing the agent to compress/merge entries manually via `replace`.

**Mapping to gspd:** This is the closest analogue to an **L0 Fixed-Load Layer**. It is tiny, always-in-context, and deliberately static during a session.

---

### Tier 2: Skills — Procedural Memory
**Files:** `tools/skill_manager_tool.py`, `tools/skills_tool.py`, `agent/prompt_builder.py`, `agent/skill_commands.py`

Skills are Hermes' answer to "how to do things". They are stored as markdown files under `~/.hermes/skills/<category>/<name>/SKILL.md`.

#### Skill Index (L0-equivalent loaded into system prompt)
`build_skills_system_prompt()` in `agent/prompt_builder.py` scans all skill directories and builds a compact index:

```text
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially relevant
to your task, you MUST load it with skill_view(name) and follow its instructions...

<available_skills>
  category:
    - skill-name: short description from frontmatter
</available_skills>
```

- **Optimisation:** Two-layer cache (in-process LRU + disk snapshot `.skills_prompt_snapshot.json` keyed by mtime/size manifest) so cold starts are fast.
- **Conditional activation:** Skills can declare `requires_tools`, `requires_toolsets`, `fallback_for_tools`, `fallback_for_toolsets` in YAML frontmatter. The index hides incompatible skills automatically.
- **External dirs:** Read-only external skill directories are supported; local skills take precedence on name collision.

#### Skill Content (L1 on-demand load)
The agent calls `skill_view(name)` to load the full `SKILL.md` content into context as a user message. This is **not** in the system prompt; it is injected as an explicit turn.

- `SKILL.md` must have YAML frontmatter with `name` and `description`.
- Max agent-written skill size: **100,000 chars** (~36k tokens).
- Supporting files live in subdirectories: `references/`, `templates/`, `scripts/`, `assets/`. These are listed in the load message but **not** auto-loaded; the agent must call `skill_view(name, file_path=...)` for them.

**Mapping to gspd:**
- Skill index = **L0 fixed-load index** (always in system prompt, very small).
- `SKILL.md` body = **L1 on-demand content** (loaded explicitly by the agent).
- Supporting files = **L2 executable/reference assets** (loaded on demand).

---

### Tier 3: Session Archive — Searchable History
**File:** `tools/session_search_tool.py`

- **Backend:** SQLite with FTS5 full-text search.
- **Storage:** Every session transcript is persisted. The tool does **not** load raw history into context; it returns **LLM-generated summaries** of matching sessions.
- **Search flow:**
  1. FTS5 query (ranked by relevance).
  2. Group by session, take top N (default 3, max 5).
  3. Load conversation, truncate to ~100k chars **centred on query matches** (smart windowing prioritises phrase hits, then proximity co-occurrence within 200 chars, then individual terms).
  4. Summarise each session in parallel using the auxiliary model (Gemini Flash by default, 10k max tokens).
- **Modes:**
  - **Keyword search:** Returns structured summaries.
  - **Recent sessions:** No LLM cost; returns metadata only (titles, previews, timestamps).

**Mapping to gspd:** This combines **L2 indexing/search** (FTS5) with **L1 summarisation** (auxiliary LLM) and **L3 long-term storage** (SQLite). It is the agent's explicit "recall" path for cross-session facts that were **not** captured in `MEMORY.md`.

---

### Tier 4: External Memory Providers (Pluggable)
**Files:** `agent/memory_manager.py`, `agent/memory_provider.py`, `plugins/memory/hindsight/__init__.py`, `plugins/memory/mem0/`

`MemoryManager` enforces a strict rule: **one built-in provider + at most one external provider**.

#### Built-in provider
Implements the `memory` tool (`MEMORY.md`/`USER.md`) described in Tier 1.

#### External providers (8 shipped plugins)
Key ones for gspd design:

| Provider | What it adds | Relevant tools |
|----------|--------------|----------------|
| **Hindsight** | Knowledge graph + entity resolution + multi-strategy retrieval (semantic + keyword + graph + rerank) | `hindsight_retain`, `hindsight_recall`, `hindsight_reflect` |
| **Mem0** | Server-side LLM fact extraction, semantic search, auto-dedup | `mem0_search`, `mem0_profile`, `mem0_conclude` |
| **Honcho** | User-state graph / user modelling | `honcho_*` |
| **Others** | RetainDB, ByteRover, SuperMemory, OpenViking, Holographic | Various |

**Hindsight deep-dive:**
- **Modes:** `cloud`, `local_embedded` (spawns local daemon with built-in PostgreSQL), `local_external`.
- **Auto-recall (`prefetch`):** Runs in a background thread after every turn. Can be configured to use `recall` (raw facts) or `reflect` (LLM synthesis across memories). Injected into the next turn inside a `<memory-context>` fence.
- **Auto-retain (`sync_turn`):** Buffers turns and retains every N turns (default 1) in a background thread. Sends the entire session JSON array to the Hindsight server for extraction.
- **Tools:** The agent can explicitly call `hindsight_reflect` for cross-memory synthesis — this is the closest Hermes comes to a user-facing "dreaming" primitive.

**Mapping to gspd:** External providers are essentially **optional L3 backends** with their own internal L2/L1 machinery (embedding, reranking, graph traversal, LLM synthesis). Hermes treats them as black boxes and only exposes a unified `MemoryProvider` interface.

---

## 3. Context Compression (gspd L1 Summarisation)
**File:** `agent/context_compressor.py`

This is one of the most production-hardened parts of Hermes and is directly relevant to the gspd L1 design.

### Trigger conditions
- Threshold: `context_length * 50%`, floored at `MINIMUM_CONTEXT_LENGTH`.
- Anti-thrashing: If the last 2 compressions each saved <10%, compression is skipped to avoid infinite loops.
- Failure cooldown: If the summariser fails (e.g. auxiliary model down), a 60s–600s cooldown prevents repeated doomed attempts.

### Algorithm (5 phases)
1. **Prune old tool results (cheap, no LLM)**
   - Walk backward from the end, protecting the most recent messages within a `tail_token_budget` (default derived from `threshold * 20%`).
   - Old tool results are replaced with informative 1-line summaries (e.g. `[terminal] ran 'npm test' -> exit 0, 47 lines output`).
   - Deduplicates identical tool outputs (e.g. reading the same file 5x).
   - Truncates large `tool_call` arguments (>500 chars) in assistant messages outside the protected tail.

2. **Protect head**
   - Keep first N messages (default 3), usually system prompt + first user/assistant exchange.

3. **Protect tail by token budget**
   - Walk backward accumulating tokens until budget is reached (default ~20K tokens, scales with model context window).
   - Hard minimum of 3 messages always protected.

4. **Summarise the middle**
   - Serialise turns into a structured prompt for the summariser model.
   - Input truncation per message: 6,000 chars total (4,000 head + 1,500 tail).
   - **Template** injected into the summariser prompt:
     ```markdown
     ## Goal
     ## Constraints & Preferences
     ## Completed Actions
     ## Active State
     ## In Progress
     ## Blocked
     ## Key Decisions
     ## Resolved Questions
     ## Pending User Asks
     ## Relevant Files
     ## Remaining Work
     ## Critical Context
     ```
   - **Iterative update:** If a previous summary exists, the prompt asks the model to *update* it (preserve relevant info, add new completed actions, move In-Progress → Completed, etc.) rather than summarising from scratch.
   - **Focus topic:** User can call `/compress <topic>` to prioritise preserving info related to a specific topic (~60-70% of summary budget).

5. **Sanitise tool-call integrity**
   - Removes orphaned `tool` results whose assistant `tool_call` was removed.
   - Injects stub `tool` results for assistant `tool_calls` whose results were dropped.

### Handoff framing
The compressed summary is injected with a strict prefix:

```text
[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted into the summary below.
This is a handoff from a previous context window — treat it as background reference,
NOT as active instructions. Do NOT answer questions or fulfill requests mentioned
in this summary; they were already addressed. Respond ONLY to the latest user message
that appears AFTER this summary.
```

This mitigates the classic "compressed instructions re-activation" bug.

---

## 4. Dreaming, Merge & Disambiguation

### 4.1 Gateway Memory Flush (The only *built-in* automatic offline process)
**File:** `gateway/run.py` (`_flush_memories_for_session`)

Triggered on:
- Session inactivity timeout
- Scheduled daily 4 AM reset
- Explicit session exit

**Implementation details:**
- Spawns a temporary `AIAgent(max_iterations=8, quiet_mode=True, skip_memory=True, enabled_toolsets=["memory", "skills"])`.
- Reads the **live disk state** of `MEMORY.md` and `USER.md` and injects it into the flush agent's prompt so it does not overwrite newer entries from other sessions/cron jobs.
- Flush prompt (verbatim from source):
  > "Review the conversation above and: 1. Save any important facts, preferences, or decisions to memory... 2. If you discovered a reusable workflow or solved a non-trivial problem, consider saving it as a skill. 3. If nothing is worth saving, that's fine — just skip. Do NOT respond to the user."
- Skips `cron_` sessions entirely.

**Takeaway:** This is a lightweight, LLM-driven dreaming layer. There is **no** structured merge algorithm, **no** entity disambiguation pipeline, and **no** rule-based deduplication. It is entirely delegated to the flush agent's reasoning.

### 4.2 GEPA Offline Skill Evolution (Heavy offline optimisation)
**File:** `evolution/skills/evolve_skill.py`

This is the heavy artillery, but it is **strictly manual / cron-triggered** and lives in a separate repo.

**Pipeline:**
1. **Dataset build**
   - `synthetic`: LLM generates eval examples from the skill text.
   - `golden`: Load human-curated dataset.
   - `sessiondb`: Mine examples from session history (Claude Code, Copilot, Hermes logs).
2. **Baseline evaluation:** Run baseline skill through dataset, score with `skill_fitness_metric`.
3. **GEPA optimisation:** `dspy.GEPA(metric=skill_fitness_metric, max_steps=iterations)` compiles the skill module against train/val sets.
4. **Constraint validation:**
   - Size limit (`max_skill_size`)
   - Growth limit (`max_prompt_growth`, default ≤ baseline + 20%)
   - Non-empty
   - Structural integrity (YAML frontmatter with `name` + `description`)
   - Optional: full pytest suite must pass 100%
5. **Holdout evaluation:** Compare baseline vs evolved on holdout set.
6. **Deployment:** **Never auto-commits**. Output is saved to `output/<skill>/<timestamp>/`. Human must review the diff and merge.

**Fitness scoring (`evolution/core/fitness.py`):**
- LLM-as-judge (`LLMJudge`) scores on:
  - `correctness` (0.5 weight)
  - `procedure_following` (0.3 weight)
  - `conciseness` (0.2 weight)
  - `length_penalty` (ramps from 0 at 90% of max size to 0.3 at 100%+)
- Fast heuristic proxy during inner loop: keyword overlap between expected and output.

**Takeaway:** GEPA is **not** a continuous dreaming loop. It is a batch compiler for skills. If gspd wants automatic skill refinement, Hermes' approach suggests doing it **offline** with explicit evaluation datasets and constraint gates.

### 4.3 Disambiguation
There is **no dedicated disambiguation module** in Hermes.
- `MEMORY.md` deduplication: exact string match at load time (`dict.fromkeys`).
- `memory` tool `replace`/`remove`: if sub-string matching returns multiple non-identical entries, the tool errors and returns previews so the LLM can refine the query.
- Hindsight (external) handles entity resolution internally inside its knowledge graph, but Hermes does not see that logic.

---

## 5. How Many Layers Should gspd Have? (Hermes' Implicit Answer)

Hermes does not use a strict N-layer cache. Instead it uses **functionally distinct tiers** that overlap in latency/cost:

```
┌─────────────────────────────────────────────────────────────────┐
│  ALWAYS IN SYSTEM PROMPT (cheap, cached, tiny)                  │
│  ├── Identity (SOUL.md / DEFAULT_AGENT_IDENTITY)               │
│  ├── Prompt Memory snapshot (MEMORY.md / USER.md)  ← ~3.5K chars│
│  └── Skills index (names + descriptions)           ← ~few K    │
├─────────────────────────────────────────────────────────────────┤
│  EXPLICITLY LOADED PER TURN (moderate cost)                    │
│  ├── skill_view(name) → full SKILL.md                          │
│  ├── External provider prefetch (Hindsight recall/reflect)     │
│  └── session_search(query) → LLM summaries of past sessions    │
├─────────────────────────────────────────────────────────────────┤
│  BACKGROUND / OFFLINE (expensive, amortised)                   │
│  ├── Context compressor summarises middle turns                │
│  ├── Gateway flush extracts memories on session expiry         │
│  └── GEPA evolves skills against evaluation datasets           │
└─────────────────────────────────────────────────────────────────┘
```

**For gspd, a reasonable mapping is:**
- **L0 Fixed-Load:** System prompt snapshot (identity + prompt memory + skill index). Must be tiny (<~5K chars total) and frozen per session.
- **L1 Summarised Context:** Context compressor output + `session_search` summaries. Injected as reference-only handoff text.
- **L2 Searchable Index:** FTS5 session DB + skill index metadata + external provider embeddings/graph.
- **L3 Raw Archive:** Full SQLite transcripts + skill files + external provider backends (Hindsight, Mem0).

**N = 4** is a pragmatic sweet spot. Hermes demonstrates that you do **not** need more than 4 functional tiers if you push heavy work offline.

---

## 6. Critical Design Defects & Trade-offs (Lessons for gspd)

### 6.1 The frozen-snapshot latency trade-off
**Defect:** Mid-session memory writes are invisible until the next session. If a user says "remember I hate YAML" and then 5 minutes later says "create a config file", the agent will not know about the preference unless it happens to call `memory` to read disk or starts a new session.

**Lesson for gspd:** If you use a frozen L0 layer for prefix-cache efficiency, you need an explicit "live read" path (tool call) for the agent to see its own recent writes. gspd should consider whether the latency trade-off is acceptable or whether a "warm reload" mechanism is needed.

### 6.2 Char-based limits are model-independent but crude
**Defect:** `MEMORY.md` is capped at 2,200 **characters**, not tokens. This is safe across models but means you get vastly different effective token budgets when switching from English prose to code or Chinese text.

**Lesson for gspd:** Consider token-aware budgeting with a fast tokenizer, or at least a character-to-token heuristic that adapts to the active model.

### 6.3 No structured dreaming pipeline
**Defect:** Outside of the Gateway flush (which is just "throw an LLM at the transcript"), Hermes has no automatic online memory consolidation. There is no merge algorithm, no contradiction detector, no decay schedule. Users must compose their own "dreaming" workflows using cron + Hindsight skills.

**Lesson for gspd:** If you want true autonomous memory refinement, you will need to build a scheduled offline pipeline more sophisticated than Hermes' flush. Consider:
- Clustering similar memories
- Detecting contradictions
- Promoting frequently-used session fragments to `MEMORY.md`
- Demoting stale `MEMORY.md` entries to the archive

### 6.4 Skill creation relies entirely on LLM judgement
**Defect:** The `skill_manage` tool is exposed to the LLM with behavioural guidance ("save after 5+ tool calls"), but there is no classifier, no hard trigger, and no online evaluator. The agent may over-create trivial skills or under-create valuable ones.

**Lesson for gspd:** If you want reliable skill extraction, add a lightweight online trigger (e.g. turn count + success signal + user approval) or an offline classifier that scans session transcripts for reusable workflows.

### 6.5 Context compression is lossy and auxiliary-model dependent
**Defect:** If the auxiliary model is unavailable, compression either enters a long cooldown or drops middle turns entirely without a summary. There is no fallback to a cheaper deterministic algorithm.

**Lesson for gspd:** Implement a tiered compression fallback:
  1. LLM structured summarisation (best quality)
  2. Heuristic truncation / tool-result pruning (medium quality, zero LLM cost)
  3. Hard drop of oldest messages (last resort)

### 6.6 External provider "only one at a time" limit
**Defect:** `MemoryManager` explicitly rejects registering a second external provider. This prevents schema bloat but means you cannot compose Hindsight (knowledge graph) + Mem0 (semantic dedup) simultaneously.

**Lesson for gspd:** Decide early whether your memory engine is monolithic or polyglot. If polyglot, design a unified query layer so multiple backends can coexist without exploding the tool schema.

---

## 7. Key Prompts & Data Structures (Copy-Paste Reference)

### 7.1 Memory Tool Schema (Behavioural Injection)
From `tools/memory_tool.py`:

```python
MEMORY_SCHEMA = {
    "name": "memory",
    "description": (
        "Save durable information to persistent memory that survives across sessions. "
        "...\n"
        "PRIORITY: User preferences and corrections > environment facts > procedural knowledge. "
        "The most valuable memory prevents the user from having to repeat themselves.\n\n"
        "Do NOT save task progress, session outcomes, completed-work logs, or temporary TODO "
        "state to memory; use session_search to recall those from past transcripts."
    ),
    ...
}
```

### 7.2 Context Compaction Summary Template
From `agent/context_compressor.py`:

```markdown
## Goal
[What the user is trying to accomplish]

## Constraints & Preferences
[User preferences, coding style, constraints, important decisions]

## Completed Actions
[Numbered list of concrete actions taken — include tool used, target, and outcome.]

## Active State
[Current working state — include working directory, branch, modified files, test status.]

## In Progress
[Work currently underway]

## Blocked
[Any blockers, errors, or issues not yet resolved.]

## Key Decisions
[Important technical decisions and WHY they were made]

## Resolved Questions
[Questions the user asked that were ALREADY answered]

## Pending User Asks
[Questions or requests from the user that have NOT yet been answered]

## Relevant Files
[Files read, modified, or created — with brief note on each]

## Remaining Work
[What remains to be done — framed as context, not instructions]

## Critical Context
[Any specific values, error messages, configuration details, or data that would be lost without explicit preservation]
```

### 7.3 Gateway Flush Prompt
From `gateway/run.py`:

```text
[System: This session is about to be automatically reset due to inactivity
or a scheduled daily reset. The conversation context will be cleared after
this turn.

Review the conversation above and:
1. Save any important facts, preferences, or decisions to memory...
2. If you discovered a reusable workflow or solved a non-trivial problem,
   consider saving it as a skill.
3. If nothing is worth saving, that's fine — just skip.

IMPORTANT — here is the current live state of memory. ...
Do NOT overwrite or remove entries unless the conversation above reveals
something that genuinely supersedes them.

Do NOT respond to the user. Just use the memory and skill_manage tools
if needed, then stop.]
```

### 7.4 Session Search Summariser Prompt
From `tools/session_search_tool.py`:

```text
You are reviewing a past conversation transcript to help recall what happened.
Summarize the conversation with a focus on the search topic. Include:
1. What the user asked about or wanted to accomplish
2. What actions were taken and what the outcomes were
3. Key decisions, solutions found, or conclusions reached
4. Any specific commands, files, URLs, or technical details that were important
5. Anything left unresolved or notable

Be thorough but concise. Preserve specific details (commands, paths, error messages)
that would be useful to recall. Write in past tense as a factual recap.
```

---

## 8. Conclusion: What gspd Should Steal vs. Avoid

| Steal from Hermes | Avoid from Hermes |
|-------------------|-------------------|
| **Frozen snapshot** for L0 prompt memory (massive cache win) | Char-based limits without token awareness |
| **Context compressor** structured template and iterative update pattern | Relying solely on LLM judgement for memory/skill creation |
| **Tool-result pruning** as a cheap pre-pass before LLM summarisation | The "only one external provider" restriction |
| **Explicit skill index** always in system prompt + on-demand `skill_view` | Lack of automatic online memory consolidation |
| **Offline GEPA evolution** with dataset + constraint gates | Manual-only trigger for skill evolution |
| **Session search** returning summaries instead of raw transcripts | Auxiliary-model single point of failure for compression/search |

Hermes' core insight for gspd is: **Keep the online path ruthlessly simple, and push all complexity to offline, explicit, evaluatable pipelines.**
