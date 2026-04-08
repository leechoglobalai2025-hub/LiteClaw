# LiteClaw — Build Guide (AI Programming Startup Protocol)

> **This is the third and final document in the Open Architecture trilogy.**
> Without this guide, AI will produce architecturally dead code. With it, AI produces LiteClaw.

---

## The Three Documents

| # | Document | Role |
|:---|:---|:---|
| 1 | `LiteClaw_Task_Execution_Architecture.md` | **What to build** — system design, principles, layer definitions |
| 2 | `LiteClaw_Task_Card_System.md` | **How to build** — 29 task cards, class signatures, test cases |
| 3 | `BUILD_GUIDE.md` (this file) | **How to start** — AI startup protocol, anti-pattern warnings |

**All three documents must be provided to the AI. Missing any one produces failure.**

---

## Why This Guide Exists

### Two real failures (March 28, 2026)

**Failure #1:** AI was given Document 1 first. It immediately began coding based on its own interpretation. When Document 2 was provided later, AI rebuilt everything from scratch — producing 35 Python files that all existed but never referenced each other. 61 tests passed, but the system had no Agent Loop connecting the layers. Dead code.

**Failure #2:** AI was given Documents 1 and 2 simultaneously with the instruction "build the complete LiteClaw software." AI launched 6 parallel agents to build all layers at once. Result: template-based UI disconnected from actual functionality, missing two entire architectural layers (L1 Channel, L4 LLM), no inter-layer integration. The author's verdict: *"屎山搬运"* (code landfill).

**Root cause in both cases:** The AI chose its own execution strategy instead of following the sequential task card order defined in Document 2.

---

## Recommended Environment

| Environment | Recommendation | Reason |
|:---|:---|:---|
| **IDE** (Google IDX, Cursor, Windsurf, etc.) | ✅ **Recommended** | Single-agent execution, visual file tree, real-time feedback, naturally sequential |
| **CLI** (Claude Code, terminal-based agents) | ⚠️ **Use with caution** | Multi-agent parallelism, black-box execution, results only visible after completion |

**The original zero-error build was achieved using Google IDE + Claude Opus 4.5.** The IDE environment naturally constrains AI to single-threaded, sequential execution — exactly what this architecture requires.

Both CLI failures occurred because CLI agents have the freedom to spawn parallel sub-agents. The AI interpreted "build the system" as "build everything simultaneously" — which violates the layer-by-layer integration principle.

**If you must use CLI:** The startup prompt below includes constraints to prevent parallelism, but IDE enforcement is inherently more reliable than prompt-level enforcement.

---

## Correct Startup Protocol

### Step 1: Load Both Documents as Context

Provide **both** Document 1 and Document 2 to the AI in the same session. Do NOT start coding yet.

### Step 2: Use This Exact Startup Prompt

Copy and paste the following prompt to begin:

```
I am providing you with two architecture documents for a project called LiteClaw.

Document 1 (Architecture Blueprint) defines WHAT to build.
Document 2 (Task Card System) defines HOW to build it.

CRITICAL RULES — violating ANY of these will produce broken code:

1. SEQUENTIAL EXECUTION ONLY: Execute task cards in strict order:
   TASK-00 → TASK-01 → TASK-02 → ... → TASK-28.
   NEVER skip ahead. NEVER parallelize. NEVER batch multiple tasks.

2. ONE TASK PER SESSION: Each task card = one independent coding session.
   Complete one task fully, then STOP and report what you built.
   Wait for my verification before proceeding to the next task.

3. LAYER INTEGRATION: Each new task MUST import and reference code
   from previously completed tasks. No isolated modules.
   Every new file must connect to the existing codebase.

4. NO TEMPLATE DUMPING: Do not generate boilerplate or placeholder code.
   Every line of code must implement actual functionality as specified
   in the task card. If a task card says "80 lines max," respect it.

5. VERIFY BEFORE PROCEEDING: After each task, run the specified tests.
   Report: files created, lines of code, test results, integration points
   with previous layers. Then WAIT for my confirmation.

6. DO NOT REORGANIZE: Follow the exact directory structure and file names
   specified in the task cards. Do not rename, restructure, or "improve"
   the organization.

Start with TASK-00 now. Show me what you build. Stop after TASK-00
and wait for my verification.
```

### Step 3: Verify Each Task Before Proceeding

After each task, check:

```
□ Files created match task card specification?
□ Code line count within task card limit?
□ All specified tests pass?
□ New code imports/references previous layer code?
□ No placeholder or stub functions?
```

If all checks pass, tell the AI: **"TASK-XX verified. Proceed to TASK-XX+1."**

If any check fails: **"TASK-XX failed. Start a new session and redo TASK-XX only."**
Do NOT let the AI fix it in the same session — entropy accumulates.

### Step 4: Phase Checkpoints

At the end of each Phase, do a full integration test:

```
Phase 0 (TASK-00):       pip install -e . works, original nanobot functions available
Phase 1 (TASK-01~04):    Security layer integrated into Agent Loop, keys never leak
Phase 2 (TASK-05~09):    Router selects models by task level, failover works
Phase 3 (TASK-10~13):    Context layer replaces original context.py, token budget enforced
Phase 4 (TASK-14~16):    Audit engine blocks destructive ops, logs to Obsidian
Phase 5 (TASK-17~20):    Execution engines connected through unified ExecutionManager
Phase 6 (TASK-21~23):    All data persists to SQLite, costs tracked accurately
Phase 7 (TASK-24~28):    UI connects to real components, all 5 tabs functional
```

---

## Anti-Patterns (What NOT to Do)

| ❌ Anti-Pattern | Why It Fails | ✅ Correct Approach |
|:---|:---|:---|
| "Build the complete system" | AI parallelizes everything, produces disconnected modules | "Start with TASK-00 only. Stop and wait." |
| Give Document 1 first, Document 2 later | AI begins coding before seeing the construction plan | Give both documents together, then use startup prompt |
| Let AI "improve" the architecture | AI reorganizes files, renames classes, adds frameworks | "Follow the exact specification. No modifications." |
| Fix failed task in the same session | Session context polluted with wrong code, entropy increases | New session, redo the failed task from scratch |
| Skip verification between tasks | Layer N+1 builds on broken Layer N, errors cascade | Verify every task. No exceptions. |
| Use multiple parallel agents | Each agent is isolated, produces code that doesn't integrate | Single sequential agent. One task at a time. |

---

## Verified Results (March 28, 2026)

### Three rounds of testing, same day, same CLI environment, same Opus 4.6:

| Round | Startup Instruction | Execution Mode | Result |
|:---|:---|:---|:---|
| **#1** | Fed Document 1 first, Document 2 later | AI decided its own order | ❌ 61 tests pass but modules isolated, no Agent Loop |
| **#2** | "Build the complete LiteClaw software" | 6 parallel agents | ❌ Template UI, missing layers, code landfill (屎山) |
| **#3** | **"绝对禁止多Agent！按部就班！"** + sequential enforcement | **Strict TASK-00→28, single agent** | ✅ **227 tests pass, all 8 layers integrated, complete product** |

**Same documents. Same model. Same environment. Only the startup instruction changed.**

### Round 3 verified metrics:

- **29 task cards:** All completed in strict sequence
- **227 tests:** All passed, zero regression across phases
- **8 architectural layers:** L0 Security → L7 Audit, fully integrated via Agent Loop
- **Phase checkpoints:** Every phase verified before proceeding to next
- **Test accumulation:** 6 → 34 → 80 → 91 → 118 → 167 → 196 → 227 (monotonically increasing)

### Historical verification:

- **Original build (Feb 2026):** Google IDE + Claude Opus 4.5 → ~5 hours, near-zero errors
- **CLI build (Mar 28, 2026):** Claude Code CLI + Claude Opus 4.6 → all 29 tasks completed, 227 tests pass

---

## The Philosophy

```
Document 1 (Architecture)     = The mind of the system
Document 2 (Task Cards)       = The hands of the system
Document 3 (Build Guide)      = The discipline of the builder

Mind without hands = ideas that never materialize
Hands without mind = code without purpose
Both without discipline = code landfill (屎山)

All three together = industrial-grade AI programming
```

---

*"The blueprint works. The construction sequence is non-negotiable. The startup protocol is the final key."*

**LEECHO Global AI Research Lab · 2026**
