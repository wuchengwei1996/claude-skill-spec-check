# spec-check: a Claude Code skill that lets independent AI agents pick a spec apart before it gets built

English · [繁體中文](README.md)

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg) ![Claude Code skill](https://img.shields.io/badge/Claude_Code-skill-blue)

First published 2026-09-15 · Last synced 2026-09-15

spec-check is a Claude Code skill that also runs in Codex CLI. You hand it a spec, a design document or a requirements document. It dispatches 3 to 5 AI agents that never saw your conversation to review it in parallel, then hands back a change plan. The spec itself is never edited; changes wait for your approval and are applied one item at a time.

The skill body (`SKILL.md`) is written in Traditional Chinese. Claude reads it fine whatever language you chat in. This page explains what it does and how to install it.

## The thirty second version

- 3 to 5 independent agents review in parallel. Each returns at most 5 findings, each finding cites a section and line and states what breaks if it is ignored.
- One agent is always assigned to cut scope: it looks for things version 1 does not need.
- You show up twice: once to confirm the inputs, once to decide.
- All agents share one context-pack (the digest written in step 1) instead of re-reading everything. The author measured a single agent re-reading on its own at roughly 100K tokens.
- The output is a change plan with before and after for each item, not an edited spec.

## What is a Claude Code skill?

A skill is a folder with a `SKILL.md` that tells Claude when to use it and what steps to follow. Drop it into `~/.claude/skills/` and Claude loads it whenever a conversation matches its description; you can also invoke it by typing `/spec-check`. This repository is that folder. Clone it and you are done. Anthropic documents the format in the [Agent Skills docs](https://docs.claude.com/en/docs/claude-code/skills).

## What does spec-check do? One picture

![spec-check overview: inputs on the left (the spec and decisions already settled), four processing steps in the middle (inventory into a context-pack, one confirmation with you, 3 to 5 independent agents reviewing in parallel, merge and verify), outputs on the right (change plan and audit context). The spec itself is untouched.](docs/spec-check-overview.en.svg)

Left is what you give it, middle is the four steps it runs, right is what you get back. The amber boxes are where you appear.

## What problem does it solve?

You finish a spec and ask an AI to take a look. It returns 11 suggestions, every one of them "consider adding X", none of them "this can go". You accept them all, the spec grows from 200 to 350 lines, and halfway through the build you find that half of it was over-engineering. That was the author's measured baseline in August 2026, before this skill existed: 11 suggestions, zero cuts.

The other case: you spend an hour with the AI, settle three decisions, then ask it to review the spec. The first finding is "reconsider decision A", which you made ten minutes ago. Worse, one section of the spec was drafted by the AI and you never approved it, and the review treats it as settled.

spec-check puts a hard rule on each failure: a cut-scope agent is mandatory; every finding must state what breaks if it is ignored; settled decisions are off limits; sections you have not approved are listed separately.

## How is it different from asking Claude to review the spec?

| Alternative | What it does | What spec-check adds | What you lose without it |
|---|---|---|---|
| Asking the AI "review this spec" | A list of suggestions | A mandatory cut-scope agent; every finding states the concrete failure; settled decisions stay settled; a change plan instead of an edited spec | Every finding is an addition; fresh decisions get reopened; the spec gets edited with no decision trail |
| The brainstorming self-review in superpowers ([obra/superpowers](https://github.com/obra/superpowers)) | The author checks 4 items and fixes inline | Agents that never saw the conversation; a premortem per aspect (assume the project shipped and failed, why); Claude spot-checks the evidence; a human decides | You cannot see your own blind spots; nobody asks why it would fail |
| The writing-plans self-review and requesting-code-review in superpowers | Checks that the plan covers the spec; reviews the code diff | Questions the spec itself | A wrong spec gets implemented faithfully |

Other tools check whether the work follows the spec. spec-check checks whether the spec should say this at all.

## How do I install it?

The folder name must be `spec-check` to match `name` in `SKILL.md`, otherwise Claude will not find it.

Claude Code, global (available in every project):

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git ~/.claude/skills/spec-check
```

Claude Code, one project only:

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git .claude/skills/spec-check
```

Codex CLI:

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git ~/.codex/skills/spec-check
```

To check it works, type `/` in Claude Code and look for `spec-check` in the menu, or just say "check this spec" and see whether it takes over. Update with `cd ~/.claude/skills/spec-check && git pull`.

## How do I trigger it?

Say "check this spec", "audit this spec" or "run a premortem on this spec", or type `/spec-check`. A typical exchange:

> **You**: /spec-check docs/specs/refund-flow-design.md
>
> **Claude**: Read it. Here is what I inventoried (list). Here is the current state as I understand it, as an ASCII flow chart. Did I misread anything? This spec is process heavy; I suggest the "user scenarios" and "data and integration" aspects, plus the mandatory cut-scope one. Anything I missed?
>
> **You**: Nothing missed, just run.
>
> **Claude**: (dispatches 3 agents, each reading the context-pack and the spec; a few minutes later)
> Merged. The change plan is at `docs/specs/2026-09-15-refund-flow-design-change-plan.md`. Two items need your decision, listed first:
> 🔴 §3.2 "auto-approve refunds after 7 days" contradicts the acceptance criteria in §5. If left as is, the implementer picks one and the tests for the other side stay red. Before and after are in the plan.
> 🔴 Cut: §6 "multi-currency refunds" is not used by any scenario in v1. Keeping it costs 3 tables and 2 endpoints nobody calls.
> 3 🟡 recommendations and 1 🟢 note follow. Once you approve, I apply the changes to the spec one by one.

The last line matters: it does not edit the spec on its own. It waits for you.

## How does it work inside?

The four steps in the picture above.

1. Inventory the material into a context-pack. Read the spec, list the settled decisions, the sections you have not approved, open items and file paths, and write them to an audit-context file. Every agent reads this one file instead of re-reading everything.
2. Show you once, wait once. One message with three things: the inventory, an ASCII picture of the current state, and the aspect menu. The picture exists so you can tell in ten seconds whether Claude misread the spec, because a misread dispatches the wrong reviewers. You can say "just run" to skip.
3. Dispatch 3 to 5 independent agents in parallel. One completeness reviewer (contradictions, vague sentences, untestable acceptance criteria), one to three premortem reviewers (assume the project shipped from exactly this spec and failed, explain why from your angle), one cut-scope reviewer (what does v1 not need, what happens if we drop it). Every agent is read only, cites section and line for every finding, and returns at most 5. The prompt templates are in `SKILL.md`, appendix A.
4. Merge and verify. Claude does this itself. Deduplicate, check the evidence for every item that needs your decision and at least one of the rest, and ask "what if v1 skips this" for every suggested addition. Then write the change plan, decisions first, each with before and after. The format is in `SKILL.md`, appendix B.

## When should I not use it?

- You want code reviewed, not a spec: use a code review tool.
- You are still deciding whether the project is worth doing: spec-check assumes you have decided to build it.
- You want an implementation plan: that is a planning tool's job; spec-check only audits the spec.
- The spec is a few dozen lines: asking Claude directly is probably faster. The skill pays off once a spec is too big for one reader to hold in their head.

## What can I customize?

The aspect menu in step 2 of `SKILL.md` (feasibility, user scenarios, frontend, backend, data and integration, operations, cost and schedule) can be edited for your domain. The sizing rule in step 3 (specs under 200 lines get 3 agents) is adjustable. Appendix B is the author's house format for change plans; swap in your own. The model routing (opus for completeness, sonnet for the rest) is a suggestion; use whatever models your environment has.

## Frequently asked questions

Will it edit my spec? No. The output is a change plan. Edits happen after you approve, one item at a time, and each one is read back to verify.

Does it need superpowers? No. The description mentions superpowers only to say that writing the plan is somebody else's job.

Why write a context-pack first instead of dispatching agents directly? Two reasons: token cost, since agents share one digest instead of each re-reading everything, and protecting settled decisions, since the context-pack lists them and agents may not overturn them.

Which models do the agents use? Is opus required? No. The templates in appendix A are model agnostic.

How long does a run take? It depends on the spec length and the agent count. The author's test on a 30 line spec with 3 agents took about 5 minutes.

## Other skills by the same author

- [claude-skill-explain](https://github.com/wuchengwei1996/claude-skill-explain): finds which kind of comprehension gap you are stuck on before choosing how to explain; when you say "still unclear" it switches method instead of writing the same thing longer.
- [claude-skill-show](https://github.com/wuchengwei1996/claude-skill-show): reads the data first, decides between a chart, a diagram and a table, then reads its own output back before calling it done.

## License and sources

MIT. The skill format follows Anthropic's [Agent Skills docs](https://docs.claude.com/en/docs/claude-code/skills). The brainstorming and writing-plans skills mentioned in the comparison table are from [obra/superpowers](https://github.com/obra/superpowers).
