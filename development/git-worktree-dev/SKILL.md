---
name: git-worktree-dev
description: Use this skill when developing a new skill using git worktree isolation. Triggers when the user wants to start skill development, create a worktree for a skill task, write staged commit logs during development, or manage a skill-task branch lifecycle.
---

# git-worktree-dev

Develop skills using a git worktree isolated workspace with staged commits as a development log, keeping the main branch history clean.

---

## Create worktree

Run from the repo root:

```bash
git worktree add ../skill-task-<feature-name> skill/<base-branch>-v<N>
```

**Naming convention:**
- Directory: `../skill-task-<feature-name>` — e.g. `../skill-task-git-worktree-dev`
- Branch: `skill/<base-branch>-v<N>` — e.g. `skill/main-v1`, iterate with v2, v3

**Example (developing git-worktree-dev from main branch):**

```bash
git worktree add ../skill-task-git-worktree-dev skill/main-v1
```

Verify status:

```bash
git worktree list
```

---

## Staged commits during development

Commit once per meaningful phase, used as a development log.

**Commit format:**

```
wip: <what was done>
```

**Principles:**
- Keep messages short, one line is enough
- Don't worry about granularity — commit whenever a small chunk is done
- Purpose is to leave a development trail, not a formal record for others

**Examples:**

```bash
git add . && git commit -m "wip: initial SKILL.md structure"
git add . && git commit -m "wip: add examples"
git add . && git commit -m "wip: adjust trigger description"
```