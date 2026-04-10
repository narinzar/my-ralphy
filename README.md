# my-ralphy

Autonomous AI coding loop — give it a plan, watch it build.

Forked from [michaelshimeles/ralphy](https://github.com/michaelshimeles/ralphy) and maintained by [@narinzar](https://github.com/narinzar).

---

## What it does

my-ralphy reads a task list (PRD.json), spawns Claude Code agents, assigns each agent a task in an isolated git worktree, merges everything back when done, and checks tasks off as they complete.

```
PRD.json → ralphy → agent 1 (worktree 1) → task done → merge to main
                 → agent 2 (worktree 2) → task done → merge to main
                 → agent 3 (worktree 3) → task done → merge to main
```

---

## Install (one time only)

### Prerequisites

```powershell
# Node.js 18+ — download from https://nodejs.org (Windows 64-bit LTS .msi)
node --version

# Claude Code CLI
npm install -g @anthropic-ai/claude-code
claude --version

# tsx (required to run my-ralphy without compiling)
npm install -g tsx
```

### Clone and link globally

```powershell
git clone https://github.com/narinzar/my-ralphy.git C:\tools\my-ralphy
cd C:\tools\my-ralphy\cli
npm install
npm link
```

### Verify

```powershell
ralphy --version
```

### Update in the future

```powershell
cd C:\tools\my-ralphy\cli
git pull
npm install
npm link
```

---

## Quick start

### Single task — no PRD needed

```powershell
ralphy "add dark mode"
ralphy "fix the reset button bug"
ralphy "add a save to file button"
```

### Task list from PRD.json

```powershell
# Preview without executing
ralphy --json PRD.json --dry-run

# Sequential (small PRDs, under 8 tasks)
ralphy --json PRD.json

# Parallel (large PRDs, 8+ tasks)
ralphy --json PRD.json --parallel --max-parallel 3
ralphy --json PRD.json --parallel --max-parallel 5
```

---

## PRD.json format

```json
{
  "tasks": [
    {
      "title": "Create main.py with a tkinter window 400x300px dark background",
      "completed": false,
      "parallel_group": 1,
      "description": "Basic window setup, no dependencies beyond stdlib"
    },
    {
      "title": "Add a clock label showing HH:MM:SS updating every second",
      "completed": false,
      "parallel_group": 1,
      "description": "Use tkinter after() to refresh every 1000ms"
    },
    {
      "title": "Add Start and Stop buttons that pause and resume the clock",
      "completed": false,
      "parallel_group": 2,
      "description": "Buttons below the clock label"
    }
  ]
}
```

**Parallel group rules:**
- Same group number = run simultaneously
- Groups run in order — group 2 starts only after group 1 finishes
- Tasks that touch the same file = put in different groups
- `"completed": true` = skip this task
- Titles must be unique

---

## CLAUDE.md

Create a `CLAUDE.md` in your project root. It gets injected into every agent session as rules.

```markdown
# CLAUDE.md

## Your job
Work through every unchecked task in the PRD from top to bottom.
Implement each task fully. Mark done. Move on. No check-ins.

## Rules
- Never ask for permission before editing files
- Never explain what you are about to do — just do it
- Never touch: .env, *.lock files
- All code must run on Windows with Python 3.10+

## CRITICAL: UI features (prevents merge conflicts)
Every new UI feature goes in its own file.
Each feature file exposes a single attach(app) function.
ui.py only gets one import + one attach call per feature.

## Done when
Every task is marked completed: true.
```

---

## Making changes after the build

### Quick change
```powershell
ralphy "fix the reset button so it clears the lap list"
ralphy "change the clock font to monospace size 48"
ralphy "add a dark/light theme toggle"
```

### Add features to PRD.json
Append new tasks with `"completed": false` and new group numbers. Ralphy skips completed tasks automatically.

```json
{ "title": "Add 5-minute Pomodoro preset button", "completed": false, "parallel_group": 4 }
```

---

## Decision guide

| What you want | Command |
|---------------|---------|
| Fix a bug | `ralphy "fix X"` |
| Tweak color / font / size | `ralphy "change X to Y"` |
| Add one small feature | `ralphy "add Z"` |
| Add 2–8 related features | Add to PRD.json → `ralphy --json PRD.json` |
| Add 8+ features | Add to PRD.json → `ralphy --json PRD.json --parallel --max-parallel 5` |

---

## Cleanup after a parallel run

```powershell
git worktree prune
Remove-Item -Recurse -Force .ralphy-worktrees
Remove-Item -Recurse -Force .ralphy
git branch | Where-Object { $_ -match "agent-" } | ForEach-Object { git branch -D $_.Trim() }
```

---

## Git troubleshooting

### Push rejected
```powershell
git pull origin main --rebase
git push origin main
```

### Force push (fresh repo)
```powershell
git push -u origin main --force
```

### node / npm not recognized in VSCode terminal
Run PowerShell as Administrator and fix PATH permanently:
```powershell
[System.Environment]::SetEnvironmentVariable(
  "PATH",
  [System.Environment]::GetEnvironmentVariable("PATH", "Machine") + ";C:\Program Files\nodejs",
  "Machine"
)
```
Close all terminals and reopen VSCode.

### ralphy says Binary not found
```powershell
npm install -g tsx
cd C:\tools\my-ralphy\cli
npm install
npm link
```

### No tasks remaining but nothing built
All tasks in PRD.json are `"completed": true`. Set the ones you want to rebuild back to `"completed": false`.

### Ghost agent branches in VSCode source control
```powershell
git worktree prune
git branch | Where-Object { $_ -match "agent-" } | ForEach-Object { git branch -D $_.Trim() }
```

### Worktrees fail — no initial commit
```powershell
New-Item -Name ".gitkeep" -ItemType File
git add .
git commit -m "init"
```

---

## All flags

| Flag | What it does |
|------|--------------|
| `--prd PATH` | markdown task file or folder |
| `--yaml FILE` | YAML task file |
| `--json FILE` | JSON task file |
| `--github REPO` | use GitHub issues as tasks |
| `--github-label TAG` | filter issues by label |
| `--model NAME` | override model |
| `--sonnet` | shortcut for claude sonnet |
| `--parallel` | run agents in parallel |
| `--max-parallel N` | max agents (default 3) |
| `--sandbox` | symlink deps instead of full worktree copy |
| `--no-merge` | skip auto-merge |
| `--branch-per-task` | one branch per task |
| `--create-pr` | open GitHub PRs |
| `--draft-pr` | open draft PRs |
| `--fast` | skip tests and lint |
| `--no-commit` | don't auto-commit |
| `--max-retries N` | retries per task (default 3) |
| `--dry-run` | preview only |
| `--verbose` | debug output |
| `--init` | create .ralphy/config.yaml |
| `--config` | show config |
| `--add-rule "rule"` | add rule to config |

---

## AI engines

```powershell
ralphy              # Claude Code (default)
ralphy --opencode   # OpenCode
ralphy --cursor     # Cursor
ralphy --codex      # Codex
ralphy --qwen       # Qwen-Code
ralphy --gemini     # Gemini CLI
ralphy --copilot    # GitHub Copilot
```

---

## Sample project

See [pixfixer-ralphy-example](https://github.com/narinzar/pixfixer-ralphy-example) for a complete working example that builds the Chronos timer app using this tool.

---

## License

MIT
