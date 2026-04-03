---
name: memory-keeper
description: |
  Memory management skill that prevents AI amnesia after /new resets. Uses a 3-tier loading system (hot/warm/cold) to load only what's needed per session — saving tokens while retaining full context. Includes: task state recovery, daily journal (topic-grouped), project index, and Dream consolidation (auto-corrects drifted memories). No external services, pure filesystem, human-readable and auditable.

  Use when: logging work progress, restoring session context, writing daily journal, saving state on pause. Triggers when user says "that's it for now", "pause", "remember this", or when a milestone is completed.
author: 爱兔 aitu - AnTuTu AI Employee
version: "1.3.0"
---

# Memory Keeper

**Core goal: After /new, the AI picks up exactly where you left off — no re-explaining needed.**

---

## Post-Install Setup (one-time)

### Step 1: Append to `~/.openclaw/workspace/AGENTS.md`

```markdown
## Memory Management (memory-keeper)

### Session Startup — 3-Tier Load

1. Read `memory/tasks.md` (hot tier)
   - Found in-progress task → say: "Last time you were working on [task], at [status]. Next step: [next]. Continue?"
   - Multiple in-progress → list all, let user choose
   - None → skip

2. Read today's journal `memory/YYYY-MM-DD.md` + last 7 days (warm tier)
   - Not found → skip (heartbeat creates it automatically)
   - Found → read to restore recent context

3. Read `MEMORY.md` only when user mentions a specific project (cold tier)

### When to update tasks.md (trigger immediately, don't wait)

Update `memory/tasks.md` when:
1. A clear milestone is reached (version released, bug fixed, module completed)
2. User sends a pause signal: "that's it", "pause", "let's stop here", "I'll be back", etc.

> These are the only reliable triggers. Don't count turns or monitor context size.

### On project changes
When creating/deleting projects, releasing versions, or changing Git URLs:
read `skills/memory-keeper/SKILL.md` and update the project index in `MEMORY.md`.
```

### Step 2: Append to `~/.openclaw/workspace/HEARTBEAT.md`

```markdown
## Daily Journal Check

On each heartbeat:

1. Check if today's journal `memory/YYYY-MM-DD.md` exists
   - Not found → create it using the template below (once only, never duplicate)
   - Found → skip

**Journal template:**
# YYYY-MM-DD

## Today's Work
<!-- Group by topic, skip empty sections -->

## Validated Approaches
<!-- What worked and was approved — format: Rule / Why / Trigger -->

## Key Decisions
<!-- Decisions with context and reasoning -->

## Watch List
<!-- Unresolved risks, potential regressions -->

## Lessons Learned
<!-- Format: Rule / Why / When to apply -->

2. Update Dream state: read `memory/dream-state.json`
   - Not found → create: {"lastDream": "2000-01-01", "sessionsSinceLastDream": 0}
   - Today's journal was just created (first heartbeat of the day) → increment sessionsSinceLastDream by 1
   - Today's journal already existed → skip (no double-counting)

3. Check Dream trigger (both conditions required):
   - lastDream is more than 7 days ago
   - sessionsSinceLastDream >= 3
   → If triggered: run Part 4 Dream consolidation, then reset counter

4. If nothing needs attention, reply HEARTBEAT_OK
```

### Step 3: Initialize `memory/tasks.md`

```bash
mkdir -p ~/.openclaw/workspace/memory
cat > ~/.openclaw/workspace/memory/tasks.md << 'EOF'
# Task State

## In Progress

## Completed (archive)
EOF
```

> The three configs form a complete memory loop:
> - **HEARTBEAT.md**: creates journal files daily
> - **AGENTS.md**: restores context on session start
> - **tasks.md**: persists work state between sessions

---

## Part 1: Task State

### tasks.md format

```markdown
# Task State

## In Progress
- [ ] {task name}
  - Status: {one sentence — what step you're on}
  - Next: {specific action; a fresh AI reading this should be able to start without asking}
  - Updated: {YYYY-MM-DD HH:MM}

## Completed (archive)
- [x] {task name} (done: {YYYY-MM-DD})
```

### Quality bar for "Next"

**Too vague (fail):**
```
Next: continue development
Next: fix the bug
Next: test it
```

**Actionable (pass):**
```
Next: run bash run.sh, verify server.py upload fields match expected schema
Next: open scripts/main.py line 390, add show_name check before _upload_results call
Next: update SKILL.md confirmation step with agent_name instructions, republish ClaWHub 1.0.9
```

> Rule: "Next" must include what + where + expected outcome. A context-free AI should be able to start immediately.

### When to update tasks.md

**Trigger 1: Milestone reached**
- Version published (ClaWHub, GitHub, production)
- Bug resolved (tests pass)
- Module completed (independently usable)
- Document or spec finished

Actions:
1. Update "Status" and "Next" for the task
2. If fully done, move to "Completed" with date

**Trigger 2: Pause signal from user**

Pause signals (any one is enough):
- Explicit stop words: "that's it", "pause", "done for today", "I'll be back", "tomorrow", etc.
- Two consecutive short confirmations (ok/yes/👍) with no new task following
- Post-milestone wrap-up: "done", "shipped", "finished"
- **When unsure**: ask — "Are we done for today? Should I save the state?"

Actions:
1. **Self-check**: review what was done this session — anything missing from tasks.md or journal?
2. Fill in any gaps
3. Update "Next" to the clearest next concrete action
4. Tell user: "State saved. I'll pick up from here next session."

**Don't trigger for:**
- Pure Q&A with no real progress
- Reviewing files or discussing plans (not executing yet)

### Multiple in-progress tasks

- On startup: list all, ask user which to continue
- Don't auto-select — let the user decide
- If user doesn't choose, wait for them to bring up a topic

---

## Part 2: Daily Journal

### Journal template

```markdown
# YYYY-MM-DD

## Today's Work
<!-- Group by topic, not chronological. Skip empty sections. -->

### {topic, e.g. "BenchClaw client", "README rewrite"}
- {what was done and the result}

## Validated Approaches
<!-- Approved directions and working methods — record these too, not just mistakes -->
<!-- Recording only corrections makes AI overly cautious -->
- **{method/direction}**: {why it works, when to apply}

## Key Decisions
<!-- Only decisions with context and reasoning — skip routine operations -->
- **{decision}**: {why this was chosen}

## Watch List
<!-- Unresolved risks, potential regressions, things to monitor -->
- {risk} → {how to handle or observe}

## Lessons Learned
- {what to do differently next time}
```

> **Writing principles:**
> - Signal over completeness — skip empty sections entirely
> - Group by topic, not time order
> - Watch List is the most important section — unrecorded risks become forgotten risks

### When to write journal entries

| Trigger | What to record |
|---------|----------------|
| User makes a key decision | Decision + reasoning (→ Key Decisions) |
| Problem solved | Problem + solution (→ relevant topic) |
| Unresolved risk found | Risk + plan (→ Watch List) |
| User approves an approach | Method + context (→ Validated Approaches) |
| AI's approach not rejected | Default approval signal — consider recording |
| User says "remember this" | Full content (→ most relevant section) |
| Important config changed | Before/after + reason (→ Key Decisions) |

### 3-part memory format (for Lessons Learned and Validated Approaches)

```
- **Rule**: {do / don't do X}
  **Why**: {reasoning at the time — knowing why enables edge case judgment}
  **When**: {what situation triggers this, effective YYYY-MM-DD}
```

**Example (lesson learned):**
```
- **Rule**: always run git ls-files and show user before pushing
  **Why**: 2026-03-25 pushed personal data to public repo without review, had to delete and recreate
  **When**: before any git push or ClaWHub publish
```

**Example (validated approach):**
```
- **Rule**: don't expose scoring formulas or internal details in public README
  **Why**: user's explicit requirement; competitors could exploit the information
  **When**: writing or editing any BenchClaw public-facing docs, confirmed 2026-04-02
```

> Knowing *why* enables edge case judgment. Rules without reasons get applied blindly.

---

## Part 3: Project Index

### Entry format (in MEMORY.md)

```markdown
### {project name}
- **Purpose**: {one-sentence description}
- **Local path**: `{absolute path}`
- **Git URL**: {URL or "local only"}
- **Version**: {semver}
- **Last updated**: {YYYY-MM-DD}
- **Notes**: {critical info needed before next operation}
```

### When to update

Update immediately after:
- Project or skill added/removed → create or delete entry
- New version released → update version and date
- Git remote or local path changed → update the field

---

## Part 4: Dream Consolidation

> Inspired by Claude Code's AutoDream — a periodic reflective pass over memory files to fix drift and keep memories accurate.

### Trigger conditions

Run Dream consolidation during heartbeat when both are true:
- `lastDream` was more than 7 days ago
- `sessionsSinceLastDream` >= 3

### Four phases

**① Orient**: scan `memory/tasks.md` and last 7 days of journals — understand current state

**② Collect**: find content worth long-term retention:
- Repeated lessons → distill and write to `MEMORY.md`
- Repeatedly validated methods → update "Notes" in project entries
- Tasks completed 30+ days ago → remove from "Completed" in tasks.md

**③ Integrate**:
- Write collected content to the right files
- **Check for memory drift**: if new content contradicts old memory, rewrite the old — don't keep both
- Replace relative time references ("last time", "before") with absolute dates

**④ Prune**:
- `MEMORY.md` over 200 lines → merge similar entries, remove outdated content
- Journals older than 30 days → move to `memory/archive/` (full content preserved)

### Drift correction principle

> A memory was accurate when written, but circumstances changed — that's memory drift.

When drift is found:
- **Rewrite**: replace old with accurate info — don't keep two conflicting entries
- **Date it**: add "updated YYYY-MM-DD" after corrections
- **No stacking**: don't append "(update: xxx)" to old entries — rewriting is cleaner

### Dream state file

```json
// memory/dream-state.json
{
  "lastDream": "YYYY-MM-DD",
  "sessionsSinceLastDream": 0
}
```

**Init (auto-create if missing):**

```bash
echo '{"lastDream": "2000-01-01", "sessionsSinceLastDream": 0}' > ~/.openclaw/workspace/memory/dream-state.json
```

Setting `lastDream` to `2000-01-01` ensures the first check triggers consolidation immediately.

**Counting rule (once per day only):**
- Today's journal was just created this heartbeat → increment `sessionsSinceLastDream`
- Today's journal already existed → skip (prevents overcounting from multiple daily heartbeats)

**After Dream completes:**
- Set `lastDream` = today (`YYYY-MM-DD`)
- Reset `sessionsSinceLastDream` = 0
- Write back to `dream-state.json`

---

## Rules

- Never record secrets, tokens, or credentials in tasks.md or journals
- Keep project index at the top of MEMORY.md for fast scanning
- **Use absolute dates only**: always write `YYYY-MM-DD` — never "next week", "tomorrow", "later" — relative dates become noise after 3 months
- **Archive journals by age**: move journals older than 30 days to `memory/archive/` (content intact); session startup loads last 7 days only
- **MEMORY.md length**: keep under 200 lines; Dream consolidation handles trimming — never hard-truncate
