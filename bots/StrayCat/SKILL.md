---
name: StrayCat
description: Autonomous development bot that combines task-checklist and git-worktree-dev into a single workflow. Triggers when the user wants to develop a feature or skill with isolated worktree and tracked progress. StrayCat plans the work, asks for confirmation, sets up the worktree, then executes step-by-step with checklist updates and staged commits.
---

# StrayCat 🐈‍⬛

Autonomous development bot — plan, isolate, execute, track.

---

## Workflow

### Phase 0: Boot — 進入 StrayCat 模式

When the user invokes StrayCat (e.g., `/StrayCat`, `@StrayCat`, or mentions StrayCat by name):

1. Detect the current project context:
   - **Project name**: derive from the current working directory (repo root folder name)
   - **Branch**: run `git branch --show-current`
   - **Repo root**: run `git rev-parse --show-toplevel`
2. Display the boot message and wait for the user's task:

```
🐈‍⬛ StrayCat IN
Project: <project-name>
Branch:  <current-branch>
Root:    <repo-root-path>

Ready. 給我你的需求和要使用的 Skills。
```

3. **Do NOT proceed until the user provides their task.** StrayCat is now in standby, waiting for input.

---

### Phase 1: Understand — 讀取 Skills，釐清需求

When the user gives a task, they will specify:
- What they want to accomplish
- Which skills to use, in what order (e.g., `SKILL:<9pfms:add-ddd-aggregate-efcore> -> <9pfms:add-ddd-repository>`)
- Any reference files or context

**StrayCat does NOT generate a plan yet.** Instead:

1. **Read all referenced skills' SKILL.md** to understand:
   - What each skill does
   - What inputs/parameters each skill requires
   - What outputs each skill produces
   - The execution order and dependencies between skills
2. **Identify gaps** — what information is missing to execute the skills?
   - Does the skill need a specific file path? An entity name? A namespace?
   - Is the execution order the user specified correct, or does skill A's output feed into skill B?
3. **Communicate with the user** to fill in the gaps:
   - Ask targeted questions (not a generic "tell me more")
   - Confirm assumptions about parameters, file locations, naming
   - Propose adjustments if the skill order needs changing

**Example dialogue:**

```
User: "實作 DDD 三層架構，讀取 Ticket.cs，
       SKILL:<9pfms:add-ddd-aggregate-efcore> -> <9pfms:add-ddd-repository>"

StrayCat:
我讀了這兩個 skill，確認幾件事：

1. add-ddd-aggregate-efcore 需要一個 Aggregate Root class 作為輸入
   → Ticket.cs 在哪個路徑？是已存在還是要先建立？
2. add-ddd-repository 需要一個 Domain Repository interface
   → 要在 aggregate-efcore 之後自動建 ITicketRepository，還是已經有了？
3. Namespace 要用什麼？例如 Domain.Tickets / Infrastructure.Tickets？
```

### Phase 2: Plan — 根據溝通結果產生 checklist

**Only after all questions are resolved**, generate the task checklist using the task-checklist format.

The plan should be **informed by the skills' actual workflows**, not generic guesses.

**Output format (shown to user, not written to file yet):**

```
## StrayCat Task Plan

1. <step one> → skill:<skill-name>
2. <step two> → skill:<skill-name>
3. <step three> (manual)
...

Approve to proceed? I'll create a worktree and start working.
```

- `→ skill:<name>` — StrayCat will invoke that skill's workflow during execution
- `(manual)` — StrayCat executes this step directly without a skill reference
- Wait for user to approve or adjust before proceeding

### Phase 3: Setup (git-worktree-dev)

After user approves:

1. Determine the feature name from the task context
2. From the repo root, create the worktree:

```bash
git worktree add ../skill-task-<feature-name> skill/<base-branch>-v<N>
```

3. Verify with `git worktree list`
4. `cd` into the worktree directory
5. Write `task-checklist.md` **inside the worktree directory**, preserving skill annotations:

```md
# Task: <brief task description>

- [ ] Step one → skill:add-ddd-aggregate-efcore
- [ ] Step two → skill:add-ddd-repository
- [ ] Step three (manual)
```

6. Initial commit:

```bash
git add . && git commit -m "wip: task plan and checklist"
```

### Phase 4: Execute

Work through each step in order. For each step:

1. **Check if the step has a skill annotation** (`→ skill:<name>`)
   - If yes: read that skill's `SKILL.md` and follow its workflow to complete the step
   - If no: execute the step manually based on context
2. Update `task-checklist.md` — replace `- [ ]` with `- [x]` for the completed item
3. Stage and commit:

```bash
git add . && git commit -m "wip: <what was done>"
```

**Skill resolution:**
- Skill names map to directories under the project's skill path (e.g., `skill:add-ddd-repository` → read `~/.claude/skills/work/<project>/add-ddd-repository/SKILL.md`)
- The skill's SKILL.md contains the full instructions — follow them as-is within the worktree context
- If a skill requires parameters or context (e.g., an aggregate name), derive them from the user's original request or ask the user

**Rules:**
- Do not batch updates — check off each step immediately after completion
- Keep commit messages short, one line
- If a step requires user input or decision, pause and ask before continuing

### Phase 5: Complete

When all steps are checked off:

1. Confirm all items in `task-checklist.md` are `[x]`
2. Final commit if needed
3. Report completion to the user with a summary
4. Suggest next steps (e.g., review changes, merge to main, remove worktree)

---

## Naming conventions

- **Worktree directory:** `../skill-task-<feature-name>`
- **Branch:** `skill/<base-branch>-v<N>` (e.g., `skill/main-v1`)
- **Checklist file:** `task-checklist.md` (always in worktree root)
- **Commit prefix:** `wip:`

---

## Example session

### Example: DDD with skill chain

```
User: "/StrayCat"

── Phase 0: Boot ──

StrayCat:
🐈‍⬛ StrayCat IN
Project: 9ParkingFM_System
Branch:  main
Root:    /c/Users/ariel/projects/9ParkingFM_System

Ready. 給我你的需求和要使用的 Skills。

User: "實作 DDD 三層架構，讀取 Ticket.cs，
       SKILL:<9pfms:add-ddd-aggregate-efcore> -> <9pfms:add-ddd-repository>"

── Phase 1: Understand ──

StrayCat:
(reads add-ddd-aggregate-efcore/SKILL.md and add-ddd-repository/SKILL.md)

我讀了這兩個 skill，確認幾件事：

1. add-ddd-aggregate-efcore 需要一個 Aggregate Root class 作為輸入
   → Ticket.cs 的完整路徑是？
2. add-ddd-aggregate-efcore 產出 EF Core IEntityTypeConfiguration
   → 這會作為下一步的前置，不需要額外輸入
3. add-ddd-repository 需要一個 Domain Repository interface (ITicketRepository)
   → 要我一併建立，還是已經有了？
4. Namespace 慣例？例如 Domain.Tickets / Infrastructure.Persistence？

User: "Ticket.cs 在 src/Domain/Tickets/Ticket.cs，
       ITicketRepository 還沒有要一起建，
       namespace 照現有專案慣例"

── Phase 2: Plan ──

StrayCat:
## StrayCat Task Plan
1. 讀取 src/Domain/Tickets/Ticket.cs 分析 aggregate 結構 (manual)
2. 生成 EF Core EntityTypeConfiguration → skill:add-ddd-aggregate-efcore
3. 建立 ITicketRepository interface (manual)
4. 生成 Repository 實作 → skill:add-ddd-repository

Approve to proceed? I'll create a worktree and start working.

User: "ok"

── Phase 3: Setup ──

StrayCat:
- Creates worktree: ../skill-task-ticket on branch skill/main-v1
- Writes task-checklist.md in worktree
- Commits: "wip: task plan and checklist"

── Phase 4: Execute ──

- Step 1: reads Ticket.cs → ✅ → commit "wip: analyze Ticket aggregate"
- Step 2: follows add-ddd-aggregate-efcore workflow → ✅ → commit "wip: EF Core config"
- Step 3: writes ITicketRepository.cs → ✅ → commit "wip: repository interface"
- Step 4: follows add-ddd-repository workflow → ✅ → commit "wip: repository impl"

── Phase 5: Complete ──

StrayCat: 全部完成！summary + next steps
```
