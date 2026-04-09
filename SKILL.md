---
name: memory-keeper
description: >
  解决 AI /new 后失忆问题的记忆管理 skill。三层加载机制（热/温/冷），session 启动时只取当前需要的记忆，省 token。
  包含：任务状态恢复、每日日记、项目索引、Dream 定期整理。纯文件系统，无需外部服务。
  用户说"先这样"、"暂停"、"记住这个"、里程碑完成时触发。

  Memory management skill that prevents AI amnesia after /new resets. 3-tier loading (hot/warm/cold) loads only
  what's needed per session — saving tokens. Includes: task state recovery, daily journal, project index,
  Dream consolidation. Pure filesystem, no external services. Triggers on "pause", "remember this", milestone completion.
---

# Memory Keeper

**Core goal: After /new, the AI picks up exactly where you left off — no re-explaining needed.**

## Part 0: First-Run Initialization

**Trigger: `memory/tasks.md` exists but is empty.**

1. Tell the user: "Your tasks.md is empty. Let me scan your recent work history and build your memory from scratch."
2. Read `MEMORY.md` + most recent session history (`openclaw sessions list --limit 1`)
3. Extract in-progress tasks, statuses, next steps
4. **Present to user for confirmation — never write without approval**
5. On confirmation → write to `memory/tasks.md`
6. If nothing found → ask user what they're working on

## Post-Install Setup (one-time)

See `references/install-snippets.md` for the three things to set up:
1. Append memory management section to `AGENTS.md`
2. Append daily journal check to `HEARTBEAT.md`
3. Initialize `memory/tasks.md`

> These three configs form a complete memory loop: HEARTBEAT.md creates journals, AGENTS.md restores context on startup, tasks.md persists work state.

## Part 1: Task State

See `references/formats.md` for format specs, quality bar for "Next", and the 3-file rule.

**When to update tasks.md — trigger immediately, don't wait:**

| Trigger | Action |
|---------|--------|
| Milestone reached (version published, bug fixed, module done) | Update status/next, or move to Completed |
| Pause signal ("that's it", "pause", "done for today") | Self-check gaps → fill → update Next → tell user "State saved" |

**Don't trigger for:** Pure Q&A, reviewing files, discussing plans without executing.

**Multiple in-progress tasks:** List all on startup, let user choose. Don't auto-select.

## Part 2: Daily Journal

See `references/formats.md` for journal template and when-to-write triggers.

- Group by topic, not chronological
- Skip empty sections entirely
- Watch List is the most important section

## Part 3: Project Index

See `references/formats.md` for MEMORY.md entry format.

Update immediately after: project added/removed, version released, Git remote/path changed.

## Part 4: Dream Consolidation

See `references/dream-guide.md` for full details: four phases, drift correction, state file management.

**Trigger (during heartbeat, both required):**
- `lastDream` > 7 days ago
- `sessionsSinceLastDream` >= 3

## Rules

- Never record secrets, tokens, or credentials in tasks.md or journals
- Keep project index at the top of MEMORY.md for fast scanning
- **Absolute dates only**: always `YYYY-MM-DD` — never "next week", "tomorrow"
- **Archive journals by age**: move journals older than 30 days to `memory/archive/`; startup loads last 7 days only
- **MEMORY.md under 200 lines**: Dream consolidation handles trimming — never hard-truncate
