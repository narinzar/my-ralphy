# my-ralphy

Autonomous AI coding loop — give it a plan, watch it build.

Forked from [michaelshimeles/ralphy](https://github.com/michaelshimeles/ralphy) and maintained by [@narinzar](https://github.com/narinzar).

---

## How it works

You create two files in your project:

1. **`PRD.json`** — your task list with parallelism instructions
2. **`CLAUDE.md`** — rules injected into every AI agent session

Then run one command. my-ralphy spawns Claude Code agents, assigns each a task in an isolated git worktree, merges everything back when done, and checks tasks off as they complete.

---

## Step 1 — Install (one time only)

```powershell
# Node.js 18+ required
node --version

# Claude Code CLI
npm install -g @anthropic-ai/claude-code
claude --version

# tsx (required to run my-ralphy)
npm install -g tsx

# bun (required to build my-ralphy)
npm install -g bun

# Clone and build my-ralphy globally
git clone https://github.com/narinzar/my-ralphy.git C:\tools\my-ralphy
cd C:\tools\my-ralphy\cli
npm install
bun run build
npm link

# Verify
ralphy --version
```

> To update my-ralphy in the future:
> ```powershell
> cd C:\tools\my-ralphy\cli
> git pull
> npm install
> bun run build
> npm link
> ```

---

## Step 2 — Create your project (every new project)

```powershell
mkdir D:\vscode_projects\my-app
cd D:\vscode_projects\my-app

git init
git config user.email "you@email.com"
git config user.name "Your Name"
New-Item -Name ".gitkeep" -ItemType File
git add .
git commit -m "init"

uv venv .venv
.venv\Scripts\Activate.ps1
uv pip install customtkinter pystray pillow pygame
```

---

## Step 3 — Create PRD.json

> **CRITICAL: Always create and edit PRD.json in VSCode — never with PowerShell
> Out-File or Set-Content.** PowerShell adds a BOM character that breaks Node.js
> JSON parsing. VSCode saves UTF-8 without BOM by default.

Open VSCode, create a new file called `PRD.json`, paste your tasks, save with `Ctrl+S`.

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
    },
    {
      "title": "Add Space bar keyboard shortcut to toggle Start/Stop",
      "completed": false,
      "parallel_group": 2,
      "description": "Bind space key to the same toggle function"
    },
    {
      "title": "Add a Reset button that sets display back to 00:00:00 and stops clock",
      "completed": false,
      "parallel_group": 3,
      "description": "Reset clears time and halts updates"
    }
  ]
}
```

**Parallel group rules:**
- Same group number = run simultaneously
- Groups run in order — group 2 starts only after group 1 finishes
- Tasks that touch the same file = put in different groups

**Task status:** `"completed": false` = pending, `"completed": true` = done and skipped. Ralphy updates this automatically.

**Titles must be unique** across all tasks.

---

## Step 4 — Create CLAUDE.md

Open VSCode, create a new file called `CLAUDE.md`, paste this, save with `Ctrl+S`.

```markdown
# CLAUDE.md — Autonomous Build Instructions

## Your job
Work through every unchecked task in the PRD from top to bottom.

For each task:
1. Implement it fully — no stubs, no placeholders
2. Mark it done in the PRD
3. Move immediately to the next task
4. If a task fails twice, skip it with a one-line note

Do not stop until every task is checked off.

## Rules
- Never ask for permission before editing or creating files
- Never explain what you are about to do — just do it
- All code must run on Windows with Python 3.10+
- Never touch: .env, *.lock files

## CRITICAL: How to add UI features (prevents merge conflicts)
Every new UI feature must go in its own file (e.g. src/history.py, src/laps.py).
Each feature file exposes a single attach(app) function.
The only change allowed in ui.py is one import + one attach call per feature.
Never add widget code directly into ui.py.

## Done when
Every task is checked off.
```

---

## Step 5 — Run

```powershell
# Preview what tasks will run without executing
ralphy --json PRD.json --dry-run

# Sequential — one task at a time (small PRDs, under 8 tasks)
ralphy --json PRD.json

# Parallel — multiple agents at once (large PRDs, 8+ tasks)
ralphy --json PRD.json --parallel --max-parallel 3

# Push it harder
ralphy --json PRD.json --parallel --max-parallel 5
```

---

## Making changes after the initial build

### Quick change — no PRD editing needed
```powershell
ralphy "fix the reset button so it also clears the lap list"
ralphy "change the clock font to monospace and increase the size"
ralphy "add a dark/light theme toggle button in the top right corner"
ralphy "fix the reset bug and change the font to monospace"
```

### Medium changes — add tasks to PRD.json
Open PRD.json in VSCode, append new tasks with `"completed": false` and new group numbers, save, then run:
```powershell
ralphy --json PRD.json
```

### Large changes — add tasks with parallel groups
```powershell
ralphy --json PRD.json --parallel --max-parallel 5
```

---

## Decision guide

| What you want | What to do |
|---------------|------------|
| Fix a bug | `ralphy "fix the bug in X"` |
| Tweak a color, font, size | `ralphy "change X to Y"` |
| Add one small feature | `ralphy "add a Z button"` |
| Add 2–8 related features | Add to PRD.json, run sequential |
| Add 8+ features | Add to PRD.json with parallel groups, run parallel |
| Rebuild from scratch | New PRD.json, all `completed: false`, run parallel |

---

## Commit after every run

```powershell
git add .
git commit -m "feat: describe what was built"
git push origin main
```

---

## Cleanup after a parallel run

```powershell
git worktree prune
Remove-Item -Recurse -Force .ralphy-worktrees
Remove-Item -Recurse -Force .ralphy
git branch | Where-Object { $_ -match "agent-" } | ForEach-Object { git branch -D $_.Trim() }
git branch
```

---

## Git troubleshooting

### Push rejected — remote has changes you don't have
```powershell
git pull origin main --rebase
git push origin main
```

### Push rejected on fresh repo
```powershell
git push -u origin main --force
```

### npm or node not recognized in VSCode terminal
VSCode terminal launched before Node was installed — PATH is stale.
Run PowerShell as Administrator and fix permanently:
```powershell
Get-Command node | Select-Object -ExpandProperty Source
[System.Environment]::SetEnvironmentVariable(
  "PATH",
  [System.Environment]::GetEnvironmentVariable("PATH", "Machine") + ";C:\Program Files\nodejs",
  "Machine"
)
```
Close all terminals and reopen VSCode.

### PRD.json shows "Unexpected token" or "not valid JSON"
PowerShell Out-File and Set-Content add a BOM character that breaks Node.js.
Fix: always create and edit PRD.json in VSCode. If already broken, rewrite it with Node:
```powershell
node -e "const fs=require('fs'); const data=JSON.parse(fs.readFileSync('PRD.json','utf8').replace(/^\uFEFF/,'')); fs.writeFileSync('PRD.json', JSON.stringify(data, null, 2), 'utf8'); console.log('Fixed')"
```

### ralphy says "No tasks remaining" but nothing was built
PRD.json has all tasks as `"completed": true`. Open in VSCode and reset tasks to `"completed": false`.

### ralphy says "Binary not found" or version errors
```powershell
cd C:\tools\my-ralphy\cli
npm install
bun run build
npm link
ralphy --version
```

### VSCode source control shows ghost agent branches
```powershell
git worktree prune
git branch | Where-Object { $_ -match "agent-" } | ForEach-Object { git branch -D $_.Trim() }
```
Then click refresh in VSCode source control panel.

### Worktrees fail — no initial commit
```powershell
New-Item -Name ".gitkeep" -ItemType File
git add .
git commit -m "init"
```

---

## Merge conflicts

When parallel agents both edit the same file:
```powershell
git status
code src/ui.py

# After resolving in VSCode (Accept Current / Accept Incoming):
git add src/ui.py
git commit -m "resolve merge conflicts"
```

Prevention: each feature in its own file, `ui.py` only gets one import line per feature.

---

## Folder structure

```
my-app/
├── PRD.json            ← task plan (never delete, keep adding to it)
├── CLAUDE.md           ← AI rules (customize per project)
├── main.py             ← built by agents
├── src/                ← feature modules
├── .ralphy/            ← progress tracking (safe to delete after run)
└── .ralphy-worktrees/  ← agent sandboxes (safe to delete after run)
```

---

## All options

| Flag | What it does |
|------|--------------|
| `--prd PATH` | task file or folder (default: PRD.md) |
| `--yaml FILE` | YAML task file |
| `--json FILE` | JSON task file |
| `--github REPO` | use GitHub issues |
| `--github-label TAG` | filter issues by label |
| `--sync-issue N` | sync PRD progress to GitHub issue #N |
| `--model NAME` | override model for any engine |
| `--sonnet` | shortcut for `--claude --model sonnet` |
| `--parallel` | run in parallel |
| `--max-parallel N` | max agents (default: 3) |
| `--sandbox` | use lightweight sandboxes instead of worktrees |
| `--no-merge` | skip auto-merge in parallel mode |
| `--branch-per-task` | branch per task |
| `--base-branch NAME` | base branch |
| `--create-pr` | create PRs |
| `--draft-pr` | draft PRs |
| `--no-tests` | skip tests |
| `--no-lint` | skip lint |
| `--fast` | skip tests + lint |
| `--no-commit` | don't auto-commit |
| `--max-iterations N` | stop after N tasks |
| `--max-retries N` | retries per task (default: 3) |
| `--retry-delay N` | seconds between retries |
| `--dry-run` | preview only |
| `--browser` | enable browser automation |
| `--no-browser` | disable browser automation |
| `-v, --verbose` | debug output |
| `--init` | setup .ralphy/ config |
| `--config` | show config |
| `--add-rule "rule"` | add rule to config |

---

## AI Engines

```powershell
ralphy              # Claude Code (default)
ralphy --opencode   # OpenCode
ralphy --cursor     # Cursor
ralphy --codex      # Codex
ralphy --qwen       # Qwen-Code
ralphy --droid      # Factory Droid
ralphy --copilot    # GitHub Copilot
ralphy --gemini     # Gemini CLI
```

### Model override
```powershell
ralphy --model sonnet "add feature"
ralphy --sonnet "add feature"
```

### Engine-specific arguments
```powershell
ralphy --claude "add feature" -- --no-permissions-prompt
ralphy --copilot --prd PRD.md -- --allow-all-tools --stream on
```

---

## Requirements

**Required:**
- Node.js 18+
- [Claude Code](https://github.com/anthropics/claude-code)
- tsx: `npm install -g tsx`
- bun: `npm install -g bun`

**Optional:**
- `gh` CLI for GitHub issues / `--create-pr`
- [agent-browser](https://agent-browser.dev) for `--browser`

---

## Engine details

| Engine | CLI | Permissions | Output |
|--------|-----|-------------|--------|
| Claude | `claude` | `--dangerously-skip-permissions` | tokens + cost |
| OpenCode | `opencode` | `full-auto` | tokens + cost |
| Codex | `codex` | N/A | tokens |
| Cursor | `agent` | `--force` | duration |
| Qwen | `qwen` | `--approval-mode yolo` | tokens |
| Droid | `droid exec` | `--auto medium` | duration |
| Copilot | `copilot` | `--yolo` | tokens |
| Gemini | `gemini` | `--yolo` | tokens + cost |

---

## License

MIT
