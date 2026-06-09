# project-kickoff (Hermes Agent skill)

Orchestrate the first 15 minutes of any new project in Hermes Agent: capture the
goal, do a tools/skills/MCP discovery pass, scaffold a wiki project page, break
the work into sprints that map to GitHub issues, and pre-fill a recovery command
so the workflow survives `/compress` or `/new`.

Adapted from [sovthpaw](https://nousresearch.com)'s project-starting workflow
on the Nous Research team.

## Install

The skill declares `category: software-development` in its frontmatter, but
Hermes's installer does not read that field — it derives category from the
`--category` flag or an interactive prompt. Pass it explicitly so the skill
lands next to its peers (`plan`, `writing-plans`, `subagent-driven-development`):

```bash
hermes skills install --category software-development \
  https://raw.githubusercontent.com/tommulkins/hermes-skill-project-kickoff/main/SKILL.md
```

Hermes does not accept a bare `https://github.com/owner/repo` URL for
single-file skills. Use the raw `SKILL.md` URL above, or install locally:

```bash
mkdir -p ~/.hermes/skills/software-development/project-kickoff
cp SKILL.md ~/.hermes/skills/software-development/project-kickoff/
hermes skills list | grep project-kickoff
```

Without `--category`, the installer will prompt interactively (TTY only) or
fall back to a flat install at `~/.hermes/skills/project-kickoff/`. The skill
works either way; the category just groups it with related skills in
`hermes skills list` and `hermes skills config`.

Then verify it loaded:

```bash
hermes skills list | grep project-kickoff
```

## When to use it

Use `project-kickoff` when the user:

- Says "I want to build X" or "let's start a new project"
- Asks "what things do you have that can work with this?" (discovery + kickoff in one)
- Has a goal that is bigger than one session and needs to survive `/compress` or `/new`
- Has a project in flight and wants to re-anchor it after losing the thread

**Don't** use it for single quick fixes, pure research, or tasks with no repo context.

## When *not* to use it (and what to use instead)

| Task | Use this instead |
|------|------------------|
| Write an implementation plan for a sprint | [`plan`](https://github.com/NousResearch/hermes-agent) or `writing-plans` |
| Ingest research sources into a knowledge base | `llm-wiki` |
| Dispatch subagents per task | `subagent-driven-development` |
| Configure, extend, or contribute to Hermes Agent itself | `hermes-agent` (the one this opener is often confused with) |

`project-kickoff` is the **scaffolding around** the plan, not a replacement for
it. It runs first, sets up the durable state (goal + wiki page + sprint table +
recovery command), and then hands off to `plan` for the actual implementation
work.

## What it produces

Running `project-kickoff` on a new project leaves you with:

1. **A goal statement** in two places (chat + `GOAL.md` or wiki page) using a
   parseable format with verifiable "done looks like" criteria.
2. **A discovery report** of relevant skills, MCP servers, past sessions, and
   project conventions.
3. **A wiki project page** with sections for goal, current state, architecture,
   sprint table, decisions log, open questions, and — critically — a
   **"Bad habits to avoid"** section that the agent re-reads on every `/new`.
4. **A sprint table** mapping each sprint to GitHub issues and a checkpoint
   (PR or commit).
5. **A pre-filled recovery command** that brings the next session up to speed
   after `/compress` or `/new` without losing the thread.

## The five-step kickoff

1. Capture the goal (two places, parseable format)
2. Discover available tools (skills, MCP, conventions, prior sessions)
3. Scaffold the wiki (one project page, with bad-habits section)
4. Break into sprints (subgoals → GitHub issues)
5. Set up context hygiene (recovery command for `/new`)

See [`SKILL.md`](./SKILL.md) for the full workflow, templates, and pitfalls.

## The bit that actually matters

sovthpaw's full quote was:

> *"Set a /goal and also ask that the goal be written into the wiki as well.
> Use /subgoal to break it down into sprints. Used GitHub for checkpoints.
> Save often. ... If we go off rails we can /compress or straight up /new and
> keep context I want and **filter out the bad habits**."*

That last clause is the load-bearing one. The goal, the architecture, the sprint
list — those can be re-derived from the repo. The list of bad habits the user
wants filtered out cannot. That's why the wiki template has a dedicated section
for it, and that's why `project-kickoff` re-reads the project page before doing
anything in a resumed session.

## Files

```
.
├── README.md       # you are here
├── SKILL.md        # the actual skill Hermes loads
└── LICENSE         # MIT
```

## License

MIT — © 2026 Tom Mulkins
