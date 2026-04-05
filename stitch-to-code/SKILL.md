---
name: stitch-to-code
description: Plan and orchestrate the implementation of UI designs exported from Google Stitch (Google's AI design tool) into any target codebase. Use this skill whenever the user mentions Stitch, shares a Stitch export directory (containing code.html, screen.png, or DESIGN.md), says "implement this Stitch design", "convert Stitch to code", or wants to translate a Google AI-generated UI prototype into real components. Also triggers when the user provides a directory with an HTML file using a Material Design 3 Tailwind color configuration alongside a screen.png mockup and asks to implement it.
---

# Stitch → Code

Google Stitch produces HTML prototypes as a **visual reference only** — the structure is never copied literally. Your job is to extract design intent from Stitch output and orchestrate its faithful implementation into the target project using proper architecture that supports real functionality.

This skill is a **planning and orchestration skill**. You analyze, plan, then optionally dispatch parallel Sonnet/Haiku agents for implementation — with `/codex:review` reviewing every output throughout.

> **Steps 1–4 are the most important part of this skill.** They determine whether the implementation is faithful to the design intent and compatible with the project. Use Opus for these steps — the quality of understanding here directly determines the quality of everything that follows.

---

## Understanding Stitch Output

A Stitch export directory contains up to three files:

| File | Present | Description |
|------|---------|-------------|
| `code.html` | Usually | HTML prototype with Tailwind CDN + MD3 color config + Material Symbols icons + Google Fonts. HTML comments label each component (e.g. `<!-- TopAppBar -->`). The Tailwind `<script id="tailwind-config">` block contains the full MD3 color palette as hex values — this is the design token source of truth. |
| `screen.png` | Usually | Rendered screenshot. Pass its path to every implementation agent as visual reference. |
| `DESIGN.md` | Sometimes | Design system spec for a specific theme: creative north star, color rules (e.g. "no borders — use background shifts"), typography rationale, explicit do/don'ts. |

**Critical:** Multiple Stitch directories may represent the **same screen in different theme variants** (e.g. `home_screen_light/` and `home_screen_dark/`). These must be identified and grouped — one implementation handles all themes, never one file per theme.

**DESIGN.md and themes:** Each DESIGN.md describes one theme's design system. When multiple DESIGN.md files exist, the implementation must support all themes in a single unified component without duplication.

---

## Workflow (follow in order)

### Step 1 — Intake
Read all files across the provided Stitch directory or directories. Identify how many DESIGN.md files exist and which theme each represents.

### Step 2 — Analysis
For each `code.html`:
- Extract the full MD3 color palette from `<script id="tailwind-config">`
- Extract typography (font families, sizes, weights, letter-spacing)
- Build a component inventory from HTML comments
- Note spacing, border radii, shadows, gradients
- Record the `screen.png` path

**Deduplication:** Identify which directories represent the same screen across theme variants. Group them into single implementation tasks. When grouping is ambiguous, ask the user to confirm before continuing.

### Step 3 — Project Assessment
- Detect the target tech stack (Expo/RN, React web, Vue, etc.)
- Find the existing color/design system file(s)
- Locate any existing plan docs in `docs/superpowers/specs/`
- Check for existing components that might already cover parts of the design

### Step 4 — Functional Clarification
- Read any existing plan docs and summarize what was found
- Ask targeted questions **one at a time** about gaps or ambiguities
- Goal: establish what each screen/component should *do*, what props/callbacks it needs, what data it consumes

### Step 5 — Color/Theme Strategy
- **Single theme:** Ask user — update project's design system to match Stitch (default), or adapt Stitch colors to fit the existing project system
- **Multiple themes:** Task board Phase 1 must merge all theme palettes into the project's unified theming system (e.g. light + dark variants in `Colors.ts`). This is mandatory — required to avoid component duplication. No choice needed.

### Step 6 — Plan Generation
Run as two sequential subagent dispatches:
1. **Spawn an Opus subagent** to generate the task board structure: phases, tasks, agent assignments (Haiku/Sonnet/Opus), and dependencies
2. **Spawn a Sonnet subagent** to write each task's detailed agent brief (see Agent Brief section). If the full context from steps 1–4 is small enough to fit in a single brief, pass it all.

After both complete: **dispatch `/codex:review` asynchronously** on the full task board + briefs. Don't wait — continue to Step 7. If review surfaces issues, address them before dispatching execution agents.

### Step 7 — Task Board Review
Present the completed task board and ask: *"Do you want to adjust any agent assignments or tasks before I begin?"*

### Step 8 — User Approves Task Board
Wait for explicit user approval before any implementation begins.

### Step 9 — Dispatch Offer
Ask: *"Should I dispatch the agents now?"*
- Yes → proceed to Step 10
- No → hand off the task board to the user or another skill

### Step 10 — Phase 1 Execution (Serial)
Dispatch shared/blocking tasks one at a time (color/theme merges, type audits, shared utilities) using Haiku, Sonnet or Opus per the task board:
- After each: **run `/codex:review` synchronously** (blocks — Phase 2 depends on this being correct)
- If review fails → the same model that ran the task fixes it immediately using the review feedback
- If fix attempt also fails → halt and report to user
- **User checkpoint after all Phase 1 tasks pass** — show what was produced before continuing

### Step 11 — Phase 2 Execution (Parallel)
Dispatch all component/screen tasks simultaneously using Haiku/Sonnet per the task board:
- Each agent writes only to its own file(s) — no two agents touch the same file
- After each completes: **dispatch `/codex:review` asynchronously** (non-blocking — other components can proceed in parallel)
- If review finds issues → the same model that built the component fixes it immediately
- If fix attempt also fails → halt dependent tasks, report to user
- Dependent tasks (e.g. screen assembly) wait for their dependencies to pass review before starting

---

## Review Everything

`/codex:review` runs on every output in this skill — not just implementation tasks. It's cheap, runs in the background, and catches issues early before they compound.

| What gets reviewed | Mode | Why |
|--------------------|------|-----|
| Task board + agent briefs (Step 6) | Async | Non-blocking; fix before dispatch if issues found |
| Phase 1 tasks (color merge, type audit) | **Sync** | Phase 2 depends on these being correct |
| Phase 2 component tasks | Async | Independent; can proceed in parallel |
| Screen assembly (final task) | **Sync** | Integrates everything; must be clean before handoff |

**Fix loop:** When `/codex:review` finds a problem, the **same model that produced the output** fixes it — not a new agent. One retry. If the second attempt fails too, halt and surface to the user with both the review findings and the failed fix attempt.

The goal is that review runs constantly in the background without slowing the main execution flow, while still guaranteeing nothing broken advances to the next phase.

---

## Task Board Format

```
## Stitch Implementation Plan: <screen name(s)>

Sources: stitch/<dir1>/, stitch/<dir2>/
Themes detected: Light (alarmory_core/DESIGN.md), Dark (vigilance_dark/DESIGN.md)
Stack: <detected stack>
Color strategy: <chosen strategy>

Functional requirements:
- <requirement 1>
- <requirement 2>

--- Phase 1: Shared (serial) ---

| Task                | Agent  | What it covers                                      |
|---------------------|--------|-----------------------------------------------------|
| Merge theme colors  | Haiku  | Update Colors.ts with light + dark MD3 palettes     |
| Type audit          | Haiku  | Ensure data interfaces cover all UI-needed fields   |

--- Phase 2: Parallel ---

| Task                | Agent  | Depends on | What it covers                                      |
|---------------------|--------|------------|-----------------------------------------------------|
| ComponentA          | Haiku  | Phase 1    | Visual shell + interaction callbacks as props       |
| ComponentB          | Haiku  | Phase 1    | ...                                                 |
| Screen assembly     | Sonnet | Phase 2    | Composes all components, wires data + navigation   |
```

**Task sizing:**
- Simple isolated components with clear visual boundaries → Haiku
- Complex components with logic, composition, or ambiguity → Sonnet
- User can upgrade any task in Step 7

All reviews use `/codex:review` regardless of which model did the implementation.

---

## Agent Brief Contents

Every dispatched agent gets a self-contained brief with:

1. **Relevant HTML section(s)** from `code.html` — explicitly labeled as visual reference only, not code to copy
2. **Path to `screen.png`** for visual reference
3. **Focused DESIGN.md content** — not a summary, not the full doc. Extract only the sections from all applicable DESIGN.md files that are relevant to this specific component (e.g. shadow rules, typography scale, the "no-border" rule). Include rules from all theme DESIGN.md files.
4. **Functional requirements** — props, callbacks, data shape, navigation behavior for this component
5. **Project conventions** — stack, styling approach, file/import patterns, existing components to reuse
6. **Target file path** and expected component interface (exported name, prop types)
7. **Explicit constraints:**
   - Use the project's theming system — no hardcoded colors
   - Design proper architecture for the target stack — do not copy the HTML structure
   - Interactions and data fetching belong as props/callbacks, not hardcoded inside
   - Must support all detected themes without duplicating the component

---

## Core Principles

- **Stitch is a visual spec, not code.** Extract design intent; never copy HTML structure.
- **One component, all themes.** Never duplicate screens or components for different themes.
- **Functionality first.** Components must be architected for real functionality — correct props, callbacks, data contracts.
- **Parallel by default.** No two parallel agents write to the same file.
- **Review everything.** `/codex:review` runs on every output. Async where possible, sync where correctness gates the next phase.
- **Original model fixes its own work.** When review finds an issue, the model that produced the output fixes it — not a new agent.
