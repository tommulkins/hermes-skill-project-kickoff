---
name: project-kickoff
description: "Orchestrate the start of a new project: capture the goal, scaffold a wiki page, break it into sprints, wire GitHub checkpoints, and set up context hygiene (/compress, /new). Use when the user says 'I want to build X' or 'let's start a new project' and you need a durable, resumable workflow — not just a plan file."
version: 1.0.0
author: Mack (adapted from sovthpaw's project-starting workflow on the Nous Research team)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [project, kickoff, workflow, planning, wiki, github, orchestration]
    category: software-development
    related_skills: [plan, llm-wiki, writing-plans, subagent-driven-development, test-driven-development, requesting-code-review, obsidian]
---

# Project Kickoff

A lightweight orchestrator for the first 15 minutes of any new project in Hermes Agent.
sovthpaw's framing: agents drift without scaffolding. The fix is not a better prompt —
it's a fixed sequence of (a) capturing the goal in two places, (b) scaffolding a
wiki page for shared state, (c) breaking the work into sprints with checkpoints,
(d) wiring context hygiene so you can /compress or /new without losing the thread.

This skill does **not** write the implementation plan — `plan` or `writing-plans`
does that. This skill sets up the *scaffolding around* the plan: where the goal lives,
where the wiki page lives, how subgoals become GitHub issues, and how to recover
when context gets long or messy.

## When This Skill Activates

Use this skill when the user:

- Says "I want to build X" / "let's start a new project" / "new project: ..."
- Asks "what things do you have that can work with this?" — discovery + kickoff in one
- Has a goal that is bigger than one session and needs to survive /compress or /new
- Already has a project in flight and wants to re-anchor it (lost the thread)

sovthpaw's framing was: agents drift without scaffolding, so every new project
opens with a recon pass before any plan gets written. Do **not** interpret the
opener as a call to the `hermes-agent` skill (that's for working on the Hermes
codebase itself).

Do **not** use this for:

- A single quick fix or one-shot question
- A pure research task (use `llm-wiki` directly to ingest sources)
- A task that has no code or repo context

## The Five-Step Kickoff

Run these in order. Each step is small, idempotent, and skippable if the artifact
already exists. Ask once if a step is genuinely ambiguous; otherwise default and
report.

### Step 1 — Capture the Goal (two places)

The goal must live in two places that survive a session loss:

1. **The active chat** — restate it in one sentence at the top of the kickoff reply.
2. **A durable note** — the project's wiki page or, if no wiki is configured,
   `~/.hermes/projects/<slug>/GOAL.md`.

Ask the user for the wiki path if it's not already known (`WIKI_PATH` env var,
or check `~/.wiki`, `~/wiki`, or their Obsidian vault via `OBSIDIAN_VAULT_PATH`).
If the user doesn't have a wiki and doesn't want one, default to
`~/.hermes/projects/<slug>/` and skip the wiki steps below.

**Goal statement format** (use this exact shape — it's parseable later):

```markdown
# Goal: <one-sentence description>

**Status:** active | paused | done
**Started:** YYYY-MM-DD
**Owner:** <user>

## What
<2-3 sentences. What we're building and for whom.>

## Why
<1-2 sentences. Why this matters / what changes when it works.>

## Done looks like
<3-5 concrete, verifiable outcomes. Not "it works" — specific signals.>
```

### Step 2 — Discover Available Tools (one pass)

sovthpaw's opener: *"What things do you have that can work with this?"* This is
explicit reconnaissance. Before writing any plan, list what's actually available:

- **Skills** — `search_files` for `SKILL.md` under `~/.hermes/skills/`, filter by
  tags relevant to the goal. Tell the user which ones apply.
- **MCP tools** — check which MCP servers are configured in the Hermes config
  (`mcp.servers`) and list any that could help (GitHub, Cursor, FAL, etc.).
- **Project context** — if a repo path is implied or given, `search_files` for
  common project instruction docs (e.g. agent rules files, `README.md`) to
  learn the conventions.
- **Prior work** — `session_search` for past sessions mentioning the same topic.
  Don't reinvent what was already attempted.

**Report in this shape** (3-6 lines, no fluff):

```
Available for this project:
- Skills: <list 2-5 most relevant, with one-line purpose each>
- MCP: <list servers, e.g. "github (PRs, issues, code review)">
- Past sessions: <N matches on "<topic>", see <links>>
- Conventions: <project instruction docs if found, else "none found">
```

### Step 3 — Scaffold the Wiki (if configured)

If `WIKI_PATH` is set (or the user just created a project folder):

Create **one** wiki page at `<wiki>/projects/<slug>.md` with this structure.
This is the shared state both human and agent read at session start to re-anchor.

```markdown
---
title: <Project Name>
type: project
status: active
started: YYYY-MM-DD
owner: <user>
tags: [project, <domain-tags>]
---

# <Project Name>

> Single source of truth. Read this first when resuming the project.

## Goal
<copy from GOAL.md — kept in sync>

## Current state
<one paragraph: where we are RIGHT NOW. Update this every session.>

## Architecture / approach
<2-5 sentences on the chosen approach. Update as decisions get made.>

## Sprints
| # | Sprint | Status | Started | GitHub |
|---|--------|--------|---------|--------|
| 1 | <name> | active | YYYY-MM-DD | [#N](url) |
| 2 | <name> | pending | — | — |

## Decisions log
- YYYY-MM-DD: <decision and rationale>
- YYYY-MM-DD: <decision and rationale>

## Open questions
- <question>
- <question>

## Bad habits to avoid
<!-- Things the user has explicitly flagged. Filter these out of the agent's behavior. -->
- <habit>: <why it's bad, what to do instead>
- <habit>: <why it's bad, what to do instead>
```

The **"Bad habits to avoid"** section is load-bearing. sovthpaw's full quote was
*"...keep context I want and filter out the bad habits."* When the user mentions
something the agent has been doing wrong — circling, over-asking, suggesting tools
that don't exist, etc. — write it here. Read it back at the start of every session.

### Step 4 — Break Into Sprints (subgoals → GitHub issues)

The "subgoal" unit is a sprint, not a step. Each sprint:

- Has a clear verifiable outcome (binary: works or doesn't)
- Maps to one or more GitHub issues (one repo per project, or a project board)
- Lasts 1-3 days of focused work, not 1-2 hours
- Has a checkpoint commit / PR at the end

**Sprint shape:**

```markdown
### Sprint N: <verb-phrase name>
**Outcome:** <one sentence, verifiable>
**Issues:** #X, #Y
**Checkpoint:** PR #Z or commit <sha>
**Done when:** <observable signal>
```

For each sprint, ask: "Does this need a GitHub issue? An issue? A PR?"
Default: **yes for code work, no for pure research/design.**

If the project is in a Git repo with no `gh` configured, ask once whether to set
it up (`gh auth login`) or skip GitHub entirely. Don't silently bounce.

### Step 5 — Set Up Context Hygiene

This is what makes the workflow survive `/compress` and `/new`:

1. **Tell the user the recovery command.** When context gets long or they say
   "I want to start fresh," the recovery is:

   ```
   /new
   > Read <wiki>/projects/<slug>.md before doing anything else.
   > Continue from sprint N, checkpoint <sha or PR>.
   ```

   Make this a literal block they can copy-paste. Pre-fill it for them.

2. **Periodic wiki reviews.** Suggest the user run `/bg review the wiki with me`
   weekly (or per sprint) to catch drift. The wiki is the contract — if it
   doesn't match reality, fix the wiki, not the code.

3. **Save often.** After every meaningful change (sprint boundary, decision,
   pivot), update the wiki page's "Current state" section and append to
   "Decisions log." This is the `commit -m` equivalent for project state.

## Output Format for the Kickoff Reply

After all five steps, reply with a tight summary in this shape:

```
# Project: <name>

**Goal:** <one sentence>

**Wiki:** <path to project page>
**Repo:** <path or "none yet">
**Sprint 1:** <name> — <one-line outcome>

**Recovery command (if you /compress or /new):**
> <pre-filled block from Step 5>

**Next action:** <one specific thing the user should say to start sprint 1>
```

Then **stop**. Don't auto-execute sprint 1. The user said "I want to build X" —
they didn't say "build X." Wait for the green light.

## Re-Anchoring a Lost Project (recovery path)

If the user comes back to a project after a gap and says "where are we on X?" or
"let's pick up <project>":

1. `read_file` the wiki project page
2. Check "Current state" against reality (read the code, check git log)
3. If they drift, surface the gap explicitly: "Wiki says sprint 2, repo shows
   sprint 1 work merged. Sync the wiki or the code?"
4. Resume from the last checkpoint. Don't re-plan from scratch.

## Composition with Other Skills

- **`plan` / `writing-plans`** — runs *after* kickoff. The kickoff sets up the
  scaffolding; `plan` writes the actionable implementation plan for sprint 1.
- **`llm-wiki`** — kickoff creates one project page; `llm-wiki` ingests external
  research sources into the same wiki.
- **`subagent-driven-development`** — once a sprint is planned, dispatch subagents
  per task with the wiki page as context.
- **`obsidian`** — if the wiki *is* the Obsidian vault, this skill and `obsidian`
  share the same filesystem; the project page just lives under `projects/`.

## Pitfalls

- **Don't over-scaffold.** Five steps is enough. If the project is one file, skip
  the wiki, skip the sprints, just write the goal and start. The point is
  matching scaffolding weight to project weight.
- **Don't auto-execute sprint 1.** The kickoff ends with "Next action: ..." and
  waits. Sovthpaw's workflow is collaborative; the user drives the pace.
- **Don't lose the bad-habits filter.** When the user says "stop doing X,"
  it goes in the wiki. The next /new session must see it. Re-reading the project
  page is the only reliable way.
- **Don't pick the wiki path silently.** If `WIKI_PATH` is unset and the user
  has multiple plausible places (Obsidian vault, `~/wiki`, `~/notes`),
  ask once, then remember the answer in memory.
- **Don't skip the recovery command.** It's the most-skipped step and the
  highest-leverage one. The whole workflow collapses without it.
- **Don't copy-paste user quotes into the trigger list.** This skill is
  inspired by sovthpaw's project-starting workflow; their canonical opener
  (`/hermes-agent I want to do a thing with this`) is a *casual rhetorical
  opener*, not a literal invocation of the `hermes-agent` skill (which is
  specifically for working on the Hermes codebase). The trigger list above
  uses descriptive intent ("I want to build X", "let's start a new project")
  rather than verbatim quotes. Same rule applies if you fork this skill: any
  quoted phrase in "When this skill activates" must read as an unambiguous
  trigger in isolation, not as context that only made sense in the original
  conversation.

## Verification

A good kickoff leaves the user able to answer:

- [ ] What's the goal, in one sentence?
- [ ] Where does the project state live (wiki path)?
- [ ] What's sprint 1, and what's the verifiable outcome?
- [ ] What does the agent do badly that we want filtered out?
- [ ] If I /new right now, what command brings the agent back up to speed?

If any answer is missing, the kickoff isn't done.
