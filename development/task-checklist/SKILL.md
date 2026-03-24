---
name: task-checklist
description: Use this skill when executing a multi-step task that requires tracking progress. Triggers when the user asks Claude to complete a task that can be broken down into steps, or when the user wants a checklist maintained during execution. Claude automatically derives the task list from the request, writes it to a .md file, and updates each item as it is completed.
---

# task-checklist

When given a task, break it down into steps, write them to a checklist file, and check off each item as it is completed.

---

## Workflow

### 1. Derive the task list

Before starting any work, analyze the user's request and break it down into concrete steps.

- Steps should be actionable and specific
- Order them by execution sequence
- Aim for 3–10 steps; split or merge if needed

### 2. Write the checklist file

Write the checklist to `task-checklist.md` in the current working directory before starting execution.

**File format:**

```md
# Task: <brief task description>

- [ ] Step one
- [ ] Step two
- [ ] Step three
```

### 3. Execute and update

Execute each step in order. After completing each step, update the checklist file immediately — replace `- [ ]` with `- [x]` for that item.

```md
# Task: <brief task description>

- [x] Step one
- [x] Step two
- [ ] Step three
```

Do not batch updates. Each step gets checked off right after it is done.

### 4. Completion

When all steps are checked off, inform the user the task is complete.

---

## File location

Always write `task-checklist.md` to the project root unless the user specifies otherwise.