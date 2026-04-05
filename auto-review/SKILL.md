---
name: auto-review
description: Use after completing any unit of work that changed files — TodoWrite tasks, Agent subagent results, or finishing a coherent chunk of code. Fires a cheap Codex review as an automatic safety net. Use this skill proactively after every code-changing task, even simple ones. This is NOT a replacement for proper reviews — it's a free bonus accuracy boost that runs alongside normal review workflows.
---

# Auto-Review

A cheap, automatic safety net that fires a Codex review after every completed unit of work. Near-zero Claude tokens by default. Catches obvious issues early before they compound. For major tasks, the user may optionally escalate to a Claude model — this is an explicit paid choice, not the default path.

This skill complements — never replaces — proper reviews like `superpowers:code-reviewer`, `subagent-driven-development` two-stage review, or `verification-before-completion`. When those skills or the user request a review, the normal process applies in full.

## When to Fire

After completing a logical unit of work that changed files:
- A TodoWrite task marked complete (that involved code changes)
- An Agent subagent returning results (for a coding task)
- Finishing a coherent chunk of work before moving to the next planned task or responding to the user

**One review event per unit of work.** If the same work satisfies multiple triggers, only one review event fires. A single event may use multiple mechanisms (primary Codex task + bonus git diff).

**Skip review for:** pure conversation, file reading/exploring, planning/design, or tasks where no files were modified.

## Flow

### Step 1: Determine task criticality

Two levels — use your judgment:

- **Major:** Task touches core logic, or changes something that downstream/dependent tasks rely on (shared interfaces, data models, utility functions used by later plan steps)
- **Minor:** Everything else — isolated changes, UI tweaks, config, self-contained additions with no dependents

### Step 2: Route the review

**For minor tasks:**
Run a Codex background review and continue working. Surface results when ready.

**For major tasks:**
Ask the user:
> "This is a major task, would you like to pause and review? If so, which model would you like to use: Codex, Haiku, Sonnet, or Opus? If no pause, Codex will review in the background like any other task."

- **Codex:** Run `codex-companion.mjs task` in the foreground (blocking)
- **Haiku / Sonnet / Opus:** Dispatch a `superpowers:code-reviewer` subagent with the chosen model
- **No pause:** Run Codex in background like a minor task

### Step 3: Run the review

**Primary mechanism — Codex task (always available, no git needed):**

Locate the script path by globbing on first use:
```bash
~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs
```
Cache the resolved path for the session.

Then run:
```bash
node "<resolved-path>/codex-companion.mjs" task "<review prompt>"
```

Build the review prompt from `review-prompt-template.md` in this skill directory. Fill in:
- What was done (task description, files modified)
- Spec/requirements context (if a plan exists)

**Bonus layer — `/codex:review` (when git is available):**

Also invoke `/codex:review` via the Skill tool for a git-diff perspective. This runs as part of the same review event. Before presenting, deduplicate findings that reference the same file and same issue across both reviews.

**Agent output review:**

When an Agent subagent returns results for a coding task, determine which files were changed by checking `git diff` (if git available) or by comparing the files the agent was instructed to modify. Pass the agent's output summary and the changed files through the same Codex task mechanism.

### Step 4: Handle findings

No auto-fix. All findings are presented to the user.

For each issue or group of issues, present options:
- **Fix** — fix it now
- **Ignore** — move on
- **Defer** — note it for later

If no issues are found, continue silently.

### When Codex is unavailable

Skip the review and emit a brief note so the user knows it was skipped. Do not block progress.

## Backstop

Enable `stopReviewGate` in Codex config to catch anything that slipped through at session end.

## Red Flags — You're About to Skip Review

- "This change is too small to review"
- "I'll review at the end"
- "It's just a config change"
- "The user didn't ask for a review"

This skill fires automatically. The user opted into it by having this skill installed. Small changes still get reviewed — the review is cheap and fast.

## What This Skill Does NOT Do

- Does not replace proper reviews when a skill or the user requests one
- Does not auto-fix anything
- Does not block progress for minor tasks
- Does not review non-code tasks (planning, conversation, file reading)
