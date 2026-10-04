---
title: the skill I wrote so AI stops overcomplicating my code
description: a SKILL.md with rule priority, a solution ladder, minimal scope (no global-by-default) and a quality gate, compared with ponytail
date: 2026-10-04
---

# the skill I wrote so AI stops overcomplicating my code

updated 2026-10-05: I rewrote the skill (from 44 sections to a much smaller version) after comparing it with ponytail. See "what changed / comparing with ponytail" below.

## tl;dr

coding agents have a habit: you ask for something small and get back a new layer, an interface with a single implementation, a dependency and an empty `catch {}` "just to be safe". I wrote a `SKILL.md` called **pragmatic-code** that tells the agent how to analyze, generate, fix, refactor and review code with a single goal: **deliver the smallest correct, safe solution, in the narrowest scope that works, following the project's own conventions.** The full file is in [`SKILL.md`](./SKILL.md). Here I summarize the parts that matter most. It is a work in progress: I keep refining it.

## the problem

AI-generated code often works and is still hard to maintain. The symptoms that bothered me most:

- abstractions nobody needs: repository, factory, service, all just forwarding calls;
- a new dependency for something the stdlib or the framework already solves;
- lint and TypeScript silenced (`@ts-ignore`, `eslint-disable`) just to pass;
- an empty `try/catch` hiding errors;
- commented-out code and pointless TODOs left in the diff;
- a change that breaks another part's contract and nobody says so;
- styles and state pushed into global scope just to "reuse" them (I only noticed this one while testing ponytail, below).

As a junior front-end dev who uses AI to learn, this hits twice as hard: if I don't notice the excess, it becomes "the right way" in my head. So I wrote down the rules I wanted the agent to follow, and the ones I want to learn to follow myself.

## the core idea

Before writing code, the agent must understand the problem, read the relevant files, find who consumes them and check contracts. Only then does it pick the smallest solution. And it explains in proportion to the problem: for a trivial change, one sentence is enough.

> Goal: the smallest correct, safe, readable solution, in the narrowest scope that works, using the project's own conventions.

Note that "smallest" comes bundled with "correct" and "safe". The smallest diff only counts after understanding the whole flow.

## what I think matters most

### 1. rule priority

When two rules clash, there is an order: security, correctness and explicit requirements, contracts and data integrity, project conventions, simplicity, readability, performance (only with a concrete need), and last abstraction and taste. Simplicity sits in the middle on purpose. And one sentence exists to prevent the worst side effect of "keep it simple":

> Simplicity never justifies removing authorization, validation, integrity, error handling, accessibility basics or an explicit requirement.

Without it, "simplify" turns into "remove the validation".

### 2. solution ladder

Before adding code, the agent walks down this ladder and stops at the first step that solves it properly:

```text
needs to exist? → reuse → stdlib → native API → installed dependency → a few lines → abstraction → new dependency
```

"An installed dependency is not a mandatory dependency". Being in `package.json` is not a reason to use it.

### 3. scope and locality (minimal is not centralized)

This is the new step, and the most important one in version 2. The rule: reuse does not mean global. Put things in the narrowest scope that works and follow what the project already does.

- **styling:** use the convention already there (Tailwind, CSS Modules, co-located styles). A one-off value stays local. It becomes a global token only when it is a shared design decision, used in 2+ places, and the project already has a token system.
- **state:** keep it in the component, lift it only to the nearest common parent. Context, store or singleton only when state is truly shared.
- **helpers and constants:** next to whoever uses them. They move to `utils/` when a second real consumer shows up.

```diff
- /* globals.css */
- :root { --contact-card-gap: 14px; }
- .contact-card { gap: var(--contact-card-gap); }
+ <div className="flex gap-3.5">   {/* one use, project uses Tailwind */}
```

### 4. don't silence tools or errors

`@ts-ignore`, `eslint-disable` and the like don't get added just to make the implementation pass. The right move is fixing the cause. Legitimate suppressions exist, but they must be minimal and explained. Same for `catch {}`: an error is handled, converted, propagated, or ignored on purpose, with a comment saying why.

### 5. breaking changes must be flagged

If the only correct solution breaks an existing contract (exported function, endpoint, schema, props, type), the agent doesn't apply it silently. It names the affected contract, explains why it is necessary, assesses who consumes it and, when possible, proposes a transitional path. Breaking can be the right call, but it has to be explicit.

### 6. when minimal is wrong

A new section, born from the failure modes of "minimalist" skills: deleting code that looks dead but is wired by string or config; skipping validation, error handling or accessibility to shorten the diff; refusing a test the logic needs; squeezing readable code into a clever one-liner; editing generated files; deleting comments that explain why; ignoring i18n. In those cases the skill says do more, not less.

### 7. quality gate

A short checklist before delivering: requirement met with nothing extra, root cause and contracts, minimal scope (styling, state, helpers), abstractions and globals actually needed, trust boundaries and errors intact, nothing silenced, accessibility and i18n not regressed, and the project's real commands run (no invented scripts).

## what changed / comparing with ponytail

[ponytail](https://ponytail.dev/) is a popular skill with the same idea (minimal code). I installed it, used it and read the whole `SKILL.md`. An honest comparison:

**what ponytail does better**

- it is short: you can read it in 2 minutes, and the agent actually follows it. Mine had 44 sections that repeated each other;
- a `description` with clear triggers ("be lazy", "yagni", complaints about over-engineering) and a "when not to use";
- a tight reply format: code first, at most three lines, `skipped: X, add when Y`;
- intensity levels (lite, full, ultra) with short examples of each;
- a `ponytail:` comment marking the ceiling of a simplification, and the rule to leave one runnable check for non-trivial logic.

**what mine does better**

- explicit priority: security, correctness and contracts above simplicity;
- real trust boundaries, authorization, integrity and concurrency (ponytail only mentions validation in passing);
- breaking changes as an explicit decision, with migration;
- not silencing tools, a quality gate, generated files.

**where each one fails**

- ponytail: it pushes "reuse" into the global scope. The agent creates a CSS variable in `:root`, a token, a class in a global stylesheet, a store or a singleton for a one-off value, because "centralizing" looks like less code. In practice it becomes coupling and a bigger diff. It also says little about real accessibility, i18n, the project's style, or keeping comments that explain why, and "one line before fifty" invites unreadable one-liners;
- my old one: too long, repeated sections, few examples, a generic `description` that triggers rarely, and not a word about scope.

**what I did**

- cut from ~2500 to ~1400 words by merging repeated sections;
- a new ponytail-style `description` (triggers, when to use, when not to);
- added "scope and locality" and "when minimal is wrong" (precautions), short before/after examples, a frontend section (React, Next.js, Tailwind, i18n, accessibility) and a short reply format;
- I did **not** copy the intensity levels. For day-to-day use I saw no real gain: asking for "leaner" already works, and three modes are one more thing for the agent to decide. If I miss them, I will bring them back.

The lesson I take: minimal is not a synonym for centralized or for "fewest characters". It is the smallest correct change, in the narrowest scope, in the style the project already uses.

## how to use it

The skill now has its own repo, with install steps and the full benchmark (5 tasks, no rules vs ponytail vs pragmatic-code, judged blind): [github.com/di0rio/pragmatic-code](https://github.com/di0rio/pragmatic-code).

Two ways, the ones I use:

1. **Claude Code:** save the file as `SKILL.md` in a folder under `~/.claude/skills/<name>/` (for example `~/.claude/skills/pragmatic-code/SKILL.md`). The file needs frontmatter with `name` and `description`. The `description` is what the agent reads to decide when to load the skill, so it is worth tuning for your case.

2. **other agents:** paste the content into the project's rules file, like `AGENTS.md`, `CLAUDE.md` or Cursor rules.

The full file is at [`./SKILL.md`](./SKILL.md).

## what is still missing

It is still a work in progress. I still need to test it on real tasks side by side (with and without the skill, and against ponytail) to measure whether the agent really follows it, especially the scope part. The rules come from things that annoyed me in generated code, so the list grows (and sometimes shrinks). If a rule doesn't change behavior, it goes: the skill has to follow its own delete-first rule too.
