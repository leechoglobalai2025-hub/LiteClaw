# LiteClaw Task Execution System Architecture
> Purpose: Feed to Claude Code as system design context  
> Version: 1.0 · February 2026  
> Core Principles: AI-Proactive System · Skill Reuse · Agent Single Lifecycle · Human as External Observer

---

## 0. Fundamental Paradigm: From Human-Proactive to AI-Proactive

```
Past (Human-Proactive):
  Human initiates request → LLM responds → End
  Each conversation is an independent function call. AI is a stateless tool.

Present (AI-Proactive):
  HEARTBEAT triggers periodically → AI perceives environment → Plans autonomously → Executes → Feedback loop
  AI is a stateful, time-continuous agent.
```

**Key Corollaries:**
- Skills are crystallized knowledge — **permanently reusable** across tasks and agents
- Agent instances are execution contexts — **disposed after use**, starting fresh each time
- Humans are not "users" — they are **system stability maintainers** and **external independent observers**

---

## 1. Task Execution System Overview

```
Input Sources
  │
  ├── User Messages (Telegram / Voice / Web UI)
  ├── HEARTBEAT Periodic Triggers
  └── External Events (RSS updates / API callbacks / File changes)
  │
  ▼
┌─────────────────────────────────────────┐
│          Task Analysis Layer             │
│  Intent Recognition → Task Classification│
│  → Complexity Assessment                 │
│  → Output Type Determination             │
│  → Execution Path Planning               │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│          Task Scheduling Layer           │
│  Single-step / Multi-step / Parallel     │
│  Skill Loading / MCP Calls / Agent       │
│  Orchestration                           │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│          Execution Layer                 │
│  Shell / Browser / API / LLM Inference   │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│          Output Layer                    │
│  Text / Image / Video / Voice / GUI Ops  │
└─────────────────────────────────────────┘
  │
  ▼
Result Validation → Memory Write → Agent Instance Destruction
```

---

## 2. Task Analysis Layer: How to Parse User Input

### 2.1 Four Elements of Intent Recognition (Logical Closure)

Upon receiving a task, these four elements must be extracted or completed — missing any one is unacceptable:

```
① Current State Description — What is the current state? (Available information/resources)
② Goal Definition          — What state should be achieved? (Explicit end state)
③ Constraint Boundaries    — What cannot be done? (Permission/time/format limits)
④ Acceptance Criteria      — How to determine success? (Measurable completion conditions)
```

**Missing any element → AI fills with hallucination → Result drift**

Completion methods:
- Prefer inference from `USER.md`, `memory/`, `AGENTS.md`
- Still unclear → Ask user the **single most critical** clarification question (do not list multiple questions)
- Truly indeterminate → Suspend and notify user

### 2.2 Task Classification Matrix

| Dimension | Options | Impact |
|------|------|------|
| **Trigger Mode** | User-initiated / HEARTBEAT periodic / Event-driven | Determines priority and response timing |
| **Execution Steps** | Single-step / Multi-step sequential / Multi-step parallel | Determines Agent orchestration method |
| **Output Type** | Text/Image/Video/Voice/GUI operation | Determines which Skills to invoke |
| **External Dependencies** | Pure local / Requires API / Requires browser | Determines failure rate and retry strategy |
| **Risk Level** | Read-only / Reversible write / Irreversible operation | Determines whether human confirmation needed |

### 2.3 Complexity Assessment and Routing

```python
# Pseudocode: Task routing logic
if single_step and read_only and no_external_dependency:
    → Execute directly, no planning needed
elif multi_step and reversible and has_clear_SOP:
    → Load corresponding Skill, execute step by step
elif involves_irreversible_operation or exceeds_SOUL_coverage:
    → Suspend, notify user via Telegram for confirmation
elif requires_multiple_skill_collaboration or estimated_tool_calls > 10:
    → Suggest user enable Research/Deep Task mode
```

---

## 3. Task Execution Architecture: Skill System

### 3.1 The Nature of Skills

**Skill = Human SOP transformed into AI-executable structured knowledge package**

```
skills/
└── skill-name/
    ├── SKILL.md      ← Core: Description + Steps + Constraints + Acceptance
    ├── package.json  ← Dependencies (optional)
    └── scripts/      ← Execution scripts (optional)
```

**Required fields in SKILL.md:**

```markdown
---
name: skill-name
description: One-line description (for progressive disclosure indexing)
triggers: [keyword1, keyword2]  # AI determines when to load this Skill
output_type: text|image|video|audio|gui
risk_level: readonly|reversible|irreversible
---

# Skill Name

## Prerequisites
- Required permissions/tools/environment

## Execution Steps
1. Step one
2. Step two (each step includes: what to do + what to verify)

## Failure Handling
- When step N fails: Degradation path or suspension condition

## Acceptance Criteria
- Measurable conditions for success

## Notes
- Known pitfalls and boundary conditions
```

### 3.2 Skill Loading Mechanism: Progressive Disclosure

```
Only inject Skill descriptions into system prompt (lightweight index)
                    ↓
AI identifies that current task requires a specific Skill
                    ↓
Dynamically read full SKILL.md content and inject into context
                    ↓
Execute steps defined in Skill
                    ↓
Task complete, Skill context released (not retained in Session)
```

**Why this design:**
- Loading all Skills at once = Token explosion
- Progressive loading = Context window only contains what current task needs

### 3.3 Skills to Build

#### Information Collection
```
skill-rss-collector      # RSS/Atom subscription collection
skill-web-scraper        # Web content scraping (via BrowserEngine)
skill-api-fetcher        # Third-party API data fetching
skill-dedup-filter       # Deduplication + relevance filtering
```

#### Content Production
```
skill-article-writer     # Article generation (structure + style alignment)
skill-video-script       # Video script (storyboard + narration + timeline)
skill-audio-tts          # Voice synthesis (via Qwen3-TTS)
skill-image-gen          # Image generation (via image models)
```

#### GUI Automation
```
skill-web-login          # Web login (human-like operation)
skill-form-fill          # Form filling
skill-content-publish    # Content publishing (cross-platform)
skill-data-extract       # Web data extraction
```

#### Knowledge Management
```
skill-obsidian-write     # Write to Obsidian (verified)
skill-memory-distill     # Memory distillation (triggers human review)
skill-knowledge-search   # Semantic knowledge base search
```

---

## 4. MCP Interface Architecture

### 4.1 MCP Positioning

MCP (Model Context Protocol) = Standardized interface for AI to call external software

```
AI Agent
    │
    ├── Skill (Custom SOP, Markdown-defined)
    │
    └── MCP Server (External software capabilities)
            ├── File System MCP
            ├── Browser Control MCP (Playwright/Puppeteer)
            ├── Database MCP (SQLite)
            ├── Obsidian MCP (write verified)
            ├── Telegram MCP (communication verified)
            └── Custom MCP (DGX Spark inference interface)
```

### 4.2 Standard MCP Call Flow

```
1. Task Analysis Layer identifies required MCP capability
2. Check MCP Server reachability (health check)
3. Build call parameters (through PromptFirewall egress scan)
4. Execute call
5. Result passes through PromptFirewall ingress scan (prevent malicious return content)
6. Result written to Session / Storage layer
7. On failure: Retry → Degrade → Suspend (by risk level)
```

### 4.3 Key MCP Characteristics and Notes

#### Browser Control MCP (Most Fragile)
```
Characteristics:
- Supports human-like click, input, scroll, screenshot
- Handles dynamically JS-rendered pages

Notes:
- Page DOM structure changes cause selector failure (silent failure!)
- Must snapshot current DOM hash as baseline before execution
- Compare and verify results after execution
- HEARTBEAT must periodically run "Skill availability probe"
- Login state/Cookie expiration detection mechanism required
- Irreversible operations (publish/delete) require human confirmation

Degradation Path:
- Primary selector fails → Try backup selector
- Backup also fails → Screenshot + notify user for manual operation
```

#### File System MCP
```
Characteristics:
- Read/write local files, monitor file changes
- Obsidian Vault real-time write (verified)

Notes:
- Backup important files before writing
- Path permission isolation (cannot escape working directory)
- Large file reads: extract only summary + key sections, do not load entirely into Session
```

#### LLM Inference MCP (DGX Spark Local)
```
Characteristics:
- GPT-OSS 120B local inference, zero API cost
- Suitable for high-volume content production tasks

Notes:
- High VRAM usage, concurrent requests need queue management
- Local model output quality differs from API models, requires tiered routing
- Task routing rule: Simple tasks → local model, High quality output → API model
```

#### Voice MCP (Qwen3 TTS/STT)
```
Characteristics:
- TTS: Text → Voice, verified
- STT: Voice → Text, verified, supports Telegram voice messages

Notes:
- Homophone/proper noun handling requires custom dictionary in Skill
- Long text TTS needs segmented processing (per-call length limit)
- Emotion/prosody markers affect output quality, marking standards must be defined in Skill
```

---

## 5. Output Type System

### 5.1 Output Type Decision Tree

```
After receiving a task, first determine output type:

Does the user need to "read/understand" something?
  → Text output (Markdown / Plain text / Structured report)

Does the user need to "see" something?
  → Image output (Screenshot / AI-generated image / Data visualization)

Does the user need to "hear" something?
  → Voice output (Qwen3-TTS synthesis)

Does the user need to "publish/distribute" content?
  → Multi-modal combination (Text + Image + Voice combined per platform requirements)

Does the user need AI to "operate" a website/software on their behalf?
  → GUI control output (BrowserEngine human-like operation)
```

### 5.2 Technical Specifications by Output Type

#### Text Output
```
Format: Markdown / Plain text / JSON / HTML
Storage: Obsidian write (with timestamp, verified)
Style Anchor: Core tone defined in SOUL.md Chinese
Quality Control: Structural integrity check + human final review (important content)
Token Cost: Directly affects cost, long text generated in segments
```

#### Image Output
```
Sources:
  - Web screenshot (BrowserEngine)
  - AI-generated (Image model API)
  - Data visualization (locally generated)
Format: PNG / JPEG / WebP
Storage: Local file system + Obsidian attachments
Note: AI-generated images require copyright attribution
```

#### Video Output
```
Current Phase: Video script generation (text), no direct video file generation
Script Format:
  - Storyboard descriptions
  - Narration script (with speech rate/pause annotations)
  - Timeline (duration per segment)
  - Music suggestions
Future Extension: Connect to video generation APIs (Sora/Kling etc. MCP)
```

#### Voice Output
```
Engine: Qwen3-TTS (verified)
Trigger: Text content → TTS Skill → Audio file
Delivery: Telegram voice message
Notes:
  - Per-call limit: ~1000 characters
  - Long content: Segmented generation + merge
  - Proper nouns: Maintain custom pronunciation dictionary
```

#### GUI Control Output
```
Essence: AI observes screen/page → Plans operation sequence → Executes human-like operations
Operation Types:
  - Click (buttons/links/menus)
  - Input (text boxes/search boxes)
  - Scroll (load more content)
  - Wait (dynamic loading completion)
  - Screenshot (verify operation results)
Safety Rules:
  - Snapshot DOM hash before execution
  - Irreversible operations require human confirmation
  - Verify expected state after each step
  - Screenshot for evidence on failure
```

---

## 6. Task Startup Mechanisms

### 6.1 Three Startup Methods

#### Method 1: User-Initiated Trigger
```
Input Channels:
  - Telegram messages (Text/Voice/Image)
  - Web UI (Gradio 9-tab interface)
  - Voice input (STT converts to text first)

Processing Flow:
  User input → STT (if voice) → Intent recognition
  → Four-element extraction → Task planning → Execution → Output
```

#### Method 2: HEARTBEAT Periodic Trigger
```
Trigger Sources (defined in HEARTBEAT.md):
  - interval: Fixed interval (every N minutes/hours)
  - cron: Precise time point (daily at 08:00)
  - threshold: Condition-based trigger (information count exceeds N)
  - event: External event (file change/API callback)

Debounce Mechanism: 250ms merge window (multiple simultaneous triggers execute only once)
Repeat Suppression: Same task does not re-execute within cooldown period
```

#### Method 3: Cascade Trigger (Task Chain)
```
Upstream task completes → Check if downstream task should trigger

Example:
  RSS collection complete (50 new items)
    → Trigger filter Skill
    → 15 highly relevant items remain after filtering
    → Trigger summary generation Skill
    → Generate daily report
    → Trigger TTS Skill (if voice push configured)
    → Send Telegram voice message
```

### 6.2 Task Queue Management

```
Priority:
  P0 - Direct user input (real-time response)
  P1 - Human confirmation callbacks for irreversible operations
  P2 - HEARTBEAT periodic tasks
  P3 - Cascade-triggered subsequent tasks
  P4 - Background knowledge organization tasks

Serialization Rules:
  - Operations within same Session must be serial (prevent race conditions)
  - Read-only operations across different Sessions may run in parallel
  - Write operations targeting same external resource must be serial + locked
```

---

## 7. Critical Notes During Task Execution

### 7.1 Signal-to-Noise Ratio Principle (Most Important)

**Human input quality directly determines AI output quality**

High-quality input = Logically closed four elements → AI output space converges → Predictable results  
Low-quality input = Missing elements/ambiguity/contradictions → AI fills with hallucination → Result drift

**Design Principles:**
- User interaction interfaces should guide structured input
- Telegram message templates (optional) reduce input noise
- When ambiguous, ask the single most critical question — do not list multiple

### 7.2 Session Memory Token Management

```
Session is a Token black hole — must be actively managed:

Hot Layer (retain fully): Most recent 5-8 conversation turns
Warm Layer (compressed): Middle sections retain only "action + conclusion", remove reasoning process
Cold Layer (archived summary): Early content summarized then written to memory/, deleted from Session

Special Notes:
- Do not retain full tool call results in Session (keep only key values)
- Collapse failed retry reasoning into a single line
- Enable Prompt Caching for SOUL/IDENTITY (fixed content not re-billed)
- HEARTBEAT triggers proactive compression when exceeding N turns
```

### 7.3 Idempotency Design

**All Skills are designed to be idempotent by default (repeated execution produces no side effects)**

```
Non-Idempotent Operations (require special handling):
  - Sending messages/emails → Add send record, check before resending
  - Creating files → Check if already exists
  - API write operations → Use idempotency key
  - Database writes → upsert instead of insert

Irreversible Operations (require human confirmation):
  - Deleting files/data
  - Publishing to public platforms
  - Sending messages to real users
  - Payment/order operations
```

### 7.4 Stability Risks of Continuous Learning

```
Risk 1: Coverage Blind Spots
  SOUL only covers anticipated scenarios
  Mitigation: Add "meta-rule" to SOUL — suspend on uncovered scenarios

Risk 2: Information Entropy Growth
  memory/ rules become outdated over time
  Mitigation: Add timestamp + expiration to each memory, periodic freshness checks

Risk 3: Self-Reinforcing Instability
  AI fixates on a past successful path, forcefully reuses it in new scenarios
  Mitigation: Add confidence rating to memory writes, successful paths require human confirmation to solidify

Risk 4: Agent Drift
  Long-running Agents drift from SOUL settings due to Session accumulation
  Mitigation: Agent single lifecycle — start from clean state every time
```

---

## 8. Result Validation and Task Closure

### 8.1 Three Levels of Result Validation

```
Level 1: Format Validation (Automatic)
  Does output match expected format?
  Was file successfully created?
  Did API return 200?

Level 2: Semantic Validation (LLM-Assisted)
  Does output content answer the original question?
  Does it meet acceptance criteria from the four elements?

Level 3: Business Validation (Human)
  Is the result truly useful?
  Are there factual errors?
  Does it need further iteration?
```

### 8.2 Task Closure Flow

```
1. Result validation passes
2. Output delivered (sent to Telegram / written to Obsidian / archived in Storage)
3. Valuable insights written to memory/ (with timestamp + confidence)
4. Update task execution statistics (cost/time/tool call count)
5. Session Memory cleanup (cold layer summary archived)
6. Agent instance destroyed
```

### 8.3 Four-Level Repair Response

```
L1 Auto-repair: Skill built-in retry, idempotent operations auto re-execute
L2 Degradation: Primary path fails → Backup path, complex → simple, output degradation notification
L3 Suspension: High-risk / exceeds SOUL scope → AI proactively pauses + Telegram notification
L4 Global Rollback: Systemic drift → Restore to most recent human-confirmed stable state (Git)
```

---

## 9. HEARTBEAT.md Design Template

```markdown
# HEARTBEAT Configuration

## Scheduled Tasks
- cron: "0 8 * * *"     → skill-daily-briefing      # Daily briefing at 08:00
- cron: "0 */6 * * *"   → skill-rss-collector        # Collect every 6 hours
- cron: "0 9 * * 1"     → skill-knowledge-freshness  # Weekly Monday knowledge freshness check
- cron: "0 */1 * * *"   → skill-health-check         # Hourly health check

## Threshold Triggers
- when: new_items > 20  → skill-dedup-filter         # Trigger filtering when new items exceed 20
- when: session_turns > 15 → skill-session-compress  # Trigger compression when Session exceeds 15 turns

## Event Triggers
- on: obsidian_change   → skill-knowledge-index      # Vault changes trigger index update
- on: skill_failure > 3 → notify_user               # Notify after 3 consecutive Skill failures

## Debounce Settings
- merge_window: 250ms   # Multi-source trigger merge window
- cooldown: 300s        # Same-type task cooldown period
```

---

## 10. Task Type Examples: End-to-End Flows

### Example 1: Information Collection → Article Generation

```
Trigger: HEARTBEAT cron daily at 08:00

Steps:
1. skill-rss-collector
   → Pull subscription sources, obtain raw content list
   → Output: raw_items[]

2. skill-dedup-filter
   → Dedup (SimHash) + Relevance scoring + Timeliness assessment
   → Output: filtered_items[] (highly relevant content)

3. skill-knowledge-search
   → Search historical memory/, supplement background knowledge
   → Output: context[]

4. skill-article-writer
   → Generate article based on filtered_items + context
   → Output: article.md (written to Obsidian)

5. Result Validation:
   → Format check (Markdown structure complete)
   → Word count / quality auto-assessment
   → Telegram notification "Today's article generated, pending review"

6. Human Review Node (cannot be skipped)
   → User replies "publish" → Trigger skill-content-publish
   → User replies "revision notes" → Trigger revision loop
```

### Example 2: Voice Input → Web Operation

```
Trigger: User sends Telegram voice message

Steps:
1. STT (Qwen3)
   → Voice to text
   → Output: "Help me publish this article on XX platform"

2. Intent Recognition
   → Task type: GUI control
   → Risk level: Irreversible (public publishing)
   → Requires: Article content (retrieved from Obsidian)

3. Human Confirmation Node (irreversible operation)
   → Send preview to Telegram: "Will publish the following content, reply 'confirm' to proceed"
   → Wait for user confirmation

4. skill-content-publish (after user confirmation)
   → BrowserEngine starts
   → Snapshot DOM hash before execution
   → Login check (Cookie validity)
   → Human-like operations: Click publish entry → Paste content → Fill tags → Click publish
   → Screenshot after execution to verify publish success
   → Output: Published URL

5. Result Write
   → Record publish time + URL in Obsidian
   → Reply success screenshot to Telegram
```

### Example 3: HEARTBEAT Health Check

```
Trigger: HEARTBEAT hourly

Check Items:
1. Is Gateway port reachable?
2. LLM API response time < 5s?
3. DGX Spark local inference available?
4. Telegram Bot heartbeat normal?
5. Any L3 suspensions unhandled in past hour?
6. Session average Token consumption exceeding threshold?

Result Handling:
  All normal → Silent (do not disturb user)
  Single anomaly → Log, wait for next check to confirm
  Two consecutive anomalies → Telegram notification to user
  Critical service unreachable → Immediate notification + trigger degradation mode
```

---

## 11. Claude Code Implementation Priority

```
P1 (Implement Immediately):
  [ ] HEARTBEAT Scheduler (cron + debounce + repeat suppression)
  [ ] Skill Loader (progressive disclosure + dynamic injection)
  [ ] Task Analyzer (intent recognition + four-element extraction)
  [ ] Session Compressor (three-layer hot/warm/cold trimming)

P2 (Core Features):
  [ ] skill-rss-collector
  [ ] skill-dedup-filter
  [ ] skill-article-writer
  [ ] skill-obsidian-write (existing foundation, wrap as Skill)

P3 (Extended Features):
  [ ] skill-web-scraper (BrowserEngine wrapper)
  [ ] skill-audio-tts (Qwen3-TTS wrapper)
  [ ] skill-content-publish (Web GUI automation)

P4 (Continuous Evolution):
  [ ] Memory distillation flow
  [ ] Knowledge freshness check
  [ ] Filter rule meta-management
```

---

## Appendix: Key Design Principles Quick Reference

| Principle | One-liner | Anti-Pattern |
|------|--------|------|
| Skill Reuse | Skills are function libraries, permanent | Rewriting logic for every task |
| Agent Single-Shot | Agents are processes, disposed after use | Letting Agents run long-term accumulating state |
| Progressive Disclosure | Only load Skills needed right now | Loading all Skills at once |
| Idempotency First | Repeated execution produces no side effects | Write operations without duplicate checks |
| Irreversible = Confirm | Publish/Delete must have human gate | Auto-executing irreversible operations |
| Hot/Warm/Cold Trimming | Session retains no more than N full turns | Session growing infinitely until explosion |
| External Observer | Humans always retain ultimate control | Fully automated with no human gates |
| Native Language SOUL | Define personality in mother tongue for maximum information density | Copy-pasting English behavioral guidelines |
| Confidence Tagging | Successful experience not auto-solidified | AI self-reinforcing incorrect paths |
| Expiring Memory | Memory content has TTL, periodic freshness checks | Trusting outdated memories forever |
```
