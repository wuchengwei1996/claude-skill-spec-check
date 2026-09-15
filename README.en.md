# spec-check

English · [繁體中文](README.md)

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg) ![Claude Code skill](https://img.shields.io/badge/Claude_Code-skill-blue)

**Before a spec is finalized, let several independent AI agents pick it apart, then you decide. The output is a change plan — the skill never edits your spec.**

> The skill body (`SKILL.md`) is written in Traditional Chinese. Claude reads it fine regardless of the language you chat in; this README tells you what it does and how to install it.

## What is a skill?

A Claude Code skill is just a folder with a `SKILL.md` that tells Claude *when* to use it and *what steps* to follow.
Drop it into `~/.claude/skills/` and Claude loads it automatically whenever the situation matches its description; you can also invoke it by typing `/spec-check`.
This repo *is* that folder — clone it and you are done; nothing else to install.

## The picture

![spec-check overview](docs/spec-check-overview.svg)

Three swim lanes. **You** appear at exactly two points (confirm the inputs, make the final call). The **commander** (the Claude you are talking to) inventories, dispatches and merges. The **agents** are throw-away independent reviewers that can read but never write.

## The problem it solves

**Scenario 1: you finish a spec and ask an AI to "take a look".**
It returns 11 suggestions. All 11 say "consider adding X". Zero say "cut Y". You accept them, the spec grows from 200 to 350 lines, and half of it turns out to be over-engineering once you build it.
(Not hypothetical — that was the measured baseline before this skill existed.)

**Scenario 2: you spend an hour with the AI, settle three decisions, then ask it to review the spec.**
First finding: "Reconsider decision A." You decided that ten minutes ago.
Worse: one section of the spec was drafted by the AI and you never approved it — the review treats it as settled fact and skips it.

spec-check puts a hard constraint on each of these failures: a **mandatory "cut scope" reviewer**, **every finding must answer "what concretely breaks if we don't fix this"**, **settled decisions are off-limits**, and **unapproved sections are listed separately**.

## Why not just use X

| Alternative | What it does | What spec-check adds | What you lose without it |
|---|---|---|---|
| Asking the AI "review this spec" | A list of suggestions | Mandatory cut-scope reviewer; every finding must state the concrete failure; settled decisions are not re-litigated; produces a change plan instead of editing the spec | All findings are additions; freshly-made decisions get reopened; the spec gets edited with no decision trail |
| superpowers `brainstorming` Spec Self-Review | The author checks 4 items and fixes inline | Fresh-context agents that **never saw the conversation**; per-aspect premortem ("assume the project failed after building this — why?"); the commander spot-checks evidence; a human decides | You cannot see your own blind spots; there is no "why did it fail" scenario |
| superpowers `writing-plans` self-review / `requesting-code-review` | Checks the plan covers the spec / reviews the code diff | Questions **the spec itself** | A wrong spec gets implemented faithfully |

In one line: other tools check "did we follow the spec"; spec-check checks "should the spec say this at all".

## Install

The folder name must be `spec-check` (it has to match `name` in `SKILL.md`), otherwise Claude will not find it.

**Claude Code — global (available in every project)**

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git ~/.claude/skills/spec-check
```

**Claude Code — one project only**

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git .claude/skills/spec-check
```

**Codex CLI**

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git ~/.codex/skills/spec-check
```

**Check it works**: type `/` in Claude Code — `spec-check` should appear in the menu. Or just say "check this spec" and see whether it takes over.

**Update**: `cd ~/.claude/skills/spec-check && git pull`

## How to trigger it

Any of these will do: "check this spec", "audit this spec", "run a premortem on this spec", or simply `/spec-check`.

A typical exchange:

> **You**: /spec-check docs/specs/refund-flow-design.md
>
> **Claude**: Read it. Here is what I inventoried (list)…
> Here is the current state as I understand it (an ASCII flow chart) — did I misread anything?
> This spec is process-heavy; I suggest the "user scenarios" and "data & integration" aspects, plus the mandatory "cut scope" one. Anything I missed?
>
> **You**: Nothing missed, just run.
>
> **Claude**: (dispatches 3 agents, each reading the context-pack (the digest file from step 1) and the spec; a few minutes later)
> Merged. Change plan written to `docs/specs/2026-09-15-refund-flow-design-change-plan.md`.
> 2 items need your decision (listed first):
> 🔴 §3.2 "auto-approve refunds after 7 days" contradicts the acceptance criteria in §5 — if left as is, the implementer picks one and the tests for the other side stay red forever. Before → after is in the plan.
> 🔴 Cut: §6 "multi-currency refunds" is not used by any scenario in v1 — keeping it costs 3 tables and 2 endpoints nobody calls.
> 3 🟡 recommendations and 1 🟢 note follow. Once you approve, I will apply the changes to the spec one by one.

Note the last line: **it never edits the spec on its own** — it waits for your go.

## How it works (matches the diagram)

1. **Inventory → context-pack**: read the spec; list settled decisions, sections that exist in the spec but you have not approved, open items, and input file paths. Write it to a `<spec>-audit-context.md`. This pre-digested file is shared by all agents so 3–5 of them do not each re-read everything (measured: one agent re-reading everything on its own costs ~100K tokens; sharing is the main cost saver).
2. **Single confirmation point**: one message with three things — the inventory, an ASCII picture of the current state, and the aspect menu — and it waits for you exactly once. The picture exists so you can tell in 10 seconds whether Claude misread the spec; a misread means the wrong agents get dispatched. You can say "just run" to skip.
3. **Parallel dispatch**: completeness reviewer ×1 (contradictions, ambiguous sentences, untestable acceptance criteria), premortem per aspect ×1–3 ("assume the project failed after building exactly this spec — explain why from your aspect"), cut-scope reviewer ×1 mandatory ("what does v1 not need? what happens if we delete it?"). Every agent gets the same context-pack, is read-only, must cite section and line for every finding, max 5 findings. Prompt templates are in `SKILL.md` Appendix A.
4. **Merge and decide**: done by the commander, never delegated — deduplicate, verify evidence for every 🔴 and at least one 🟡/🟢, and for every "add X" ask "what breaks if v1 does not have it".
5. **Change plan**: `YYYY-MM-DD-<spec>-change-plan.md`, 🔴 decisions first, each with a before → after side-by-side. Format in `SKILL.md` Appendix B.

## Optional integrations

- **superpowers**: spec-check is designed to sit after `brainstorming` (write the spec) and before `writing-plans` (write the plan), but it does not depend on them — it works standalone.
- **Model routing**: `SKILL.md` suggests opus for the completeness reviewer and sonnet for the rest. That is a suggestion; use whatever models your environment has.

## Customize

- **Aspect menu**: step 2's list (feasibility / user scenarios / frontend / backend / data & integration / operations / cost & schedule) — add or remove for your domain.
- **Agent count**: step 3's sizing rule (spec < 200 lines → 3 agents) is adjustable.
- **Change-plan format**: Appendix B is the author's house style (human layer on top, machine layer below, 😣/🎯 opener). Swap in your own.

## FAQ

**Will it edit my spec?**
No. It produces a change plan; edits happen only after you approve, item by item, with a read-back check afterwards.

**Does it need superpowers?**
No. superpowers is mentioned in the description only to say "writing the plan is not this skill's job".

**Why write a context-pack first instead of dispatching agents directly?**
Two reasons: token cost (agents share one pre-digested file instead of each re-reading everything) and protecting settled decisions (the context-pack lists them as "settled", and agents may not overturn them).

**Is it worth running on a short spec?**
A few dozen lines usually needs only 3 agents (completeness + cut-scope + one aspect). Shorter than that, asking Claude directly may be faster — the skill pays off once a spec is too big for one reader to hold in their head.

**Which models do the agents use? Is opus required?**
Not required. The templates in Appendix A are model-agnostic; opus/sonnet is just the author's suggested routing.

## Series

Two more skills by the same author:

- [claude-skill-explain](https://github.com/wuchengwei1996/claude-skill-explain): find the comprehension gap first, then pick the explanation method — not "say it again, longer".
- [claude-skill-show](https://github.com/wuchengwei1996/claude-skill-show): understand the information first, decide how to show it, then actually produce and read back the result.

## License

MIT
