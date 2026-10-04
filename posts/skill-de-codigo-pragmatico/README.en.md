---
title: the skill I wrote so AI stops overcomplicating my code
description: a SKILL.md with rule priority, a solution hierarchy and a quality gate, so coding agents deliver the smallest correct, safe, readable solution
date: 2026-10-04
---

# the skill I wrote so AI stops overcomplicating my code

## tl;dr

coding agents have a habit: you ask for something small and get back a new layer, an interface with a single implementation, a dependency and an empty `catch {}` "just to be safe". I wrote a `SKILL.md` called **Self-documenting Code and Pragmatic Engineering for AI** that tells the agent how to analyze, generate, fix, refactor and review code with a single goal: **deliver the smallest correct, safe, readable solution compatible with the current project.** The full file is in [`SKILL.md`](./SKILL.md). Here I summarize the parts that matter most. It is a work in progress: I keep refining it.

## the problem

AI-generated code often works and is still hard to maintain. The symptoms that bothered me most:

- abstractions nobody needs: repository, factory, service, all just forwarding calls;
- a new dependency for something the stdlib or the framework already solves;
- lint and TypeScript silenced (`@ts-ignore`, `eslint-disable`) just to pass;
- an empty `try/catch` hiding errors;
- commented-out code and pointless TODOs left in the diff;
- a change that breaks another part's contract and nobody says so.

As a junior front-end dev who uses AI to learn, this hits twice as hard: if I don't notice the excess, it becomes "the right way" in my head. So I wrote down the rules I wanted the agent to follow, and the ones I want to learn to follow myself.

## the core idea

Before writing code, the agent must understand the problem, read the relevant files, find who consumes them and check contracts. Only then does it pick the smallest solution. And it explains the strategy in proportion to the problem: for a trivial change, one sentence is enough.

> **Goal: deliver the smallest correct, safe, readable solution compatible with the current project.**

Note that "smallest" comes bundled with "correct" and "safe". The smallest diff only counts after understanding the whole flow.

## what I think matters most

### 1. rule priority

When two rules clash, there is an order to decide:

1. security
2. functional correctness
3. explicit requirements
4. contract integrity and compatibility
5. data integrity
6. simplicity (KISS/YAGNI)
7. readability
8. maintainability
9. performance, when there is a concrete need
10. abstraction
11. stylistic preferences

Simplicity sits in the middle on purpose. And one sentence exists to prevent the worst side effect of "keep it simple":

> Simplicity never justifies removing authorization, validation, integrity, error handling, or explicit requirements.

Without it, "simplify" turns into "remove the validation".

### 2. solution hierarchy

Before adding code, the agent walks down this ladder and stops at the first step that solves it properly:

```text
remove → reuse → stdlib → native runtime/framework API → installed dependency → simple code → abstraction → new dependency → infrastructure
```

One detail I like: "an installed dependency is not a mandatory dependency". Being in `package.json` is not a reason to use it.

### 3. anti-overengineering

The skill lists what must **not** be introduced automatically: Clean Architecture, DDD, Repository, Unit of Work, Factory, Adapter, Strategy, CQRS, Event Bus, a dependency injection container, microservices. The one-line summary: patterns are tools, not requirements. Empty layers don't count either:

```text
Route → Controller → Service → Repository → Database
```

If none of those layers has a real responsibility, it is just ceremony.

### 4. don't silence tools

`@ts-ignore`, `eslint-disable` and the like don't get added just to make the implementation pass. The right move is fixing the cause. Legitimate suppressions exist, but they must be minimal and justifiable. Same logic for errors: no

```ts
try {
  await operation();
} catch {}
```

An error is handled, converted, propagated, or deliberately ignored when that is part of the contract.

### 5. breaking changes must be flagged

If the only correct solution breaks an existing contract (exported function, endpoint, schema, type), the agent doesn't apply it silently. It names the affected contract, explains why it is necessary, assesses who consumes it and, when possible, proposes a compatible or transitional path. Breaking can be the right call, but it has to be an explicit decision, not a side effect.

### 6. quality gate

A checklist before delivering. Some of the questions:

- requirement met, no extra features?
- smallest correct diff, root cause fixed?
- consumers and contracts checked?
- abstractions and dependencies actually necessary?
- no redundant comments, commented-out code, trivial TODOs or temporary logs?
- trust boundaries, authorization and data integrity preserved?
- errors handled? typecheck, lint, tests and build run when applicable?
- if there is a breaking change, was it flagged?

And a rule against making things up: use the project's real commands, don't invent scripts.

### 7. the golden rule

A cascade of questions, in order:

> Can I remove it? → Does it already exist? → Does the platform already solve it? → Does an existing dependency solve it? → Do a few simple lines solve it? → Is there a real need for abstraction?

Only after that is a new dependency or infrastructure considered. It closes with: fully understand the problem before trying to solve it minimally.

## how to use it

Two ways, the ones I use:

1. **Claude Code:** save the file as `SKILL.md` in a folder under `~/.claude/skills/<name>/` (for example `~/.claude/skills/pragmatic-code/SKILL.md`). The file needs frontmatter with `name` and `description`:

   ```yaml
   ---
   name: pragmatic-code
   description: Self-documenting code and pragmatic engineering rules for AI coding agents - deliver the smallest correct, safe, readable solution compatible with the current project.
   ---
   ```

   The `description` is what the agent reads to decide when to load the skill, so it is worth tuning for your case.

2. **other agents:** paste the content into the project's rules file, like `AGENTS.md`, `CLAUDE.md` or Cursor rules.

The full file, with every section (TypeScript, database, uploads, API, global state, tests and the rest), is at [`./SKILL.md`](./SKILL.md).

## what is still missing

It is a work in progress. Some sections are still too long for a prompt, and I am testing what the agent actually follows and what it ignores. The rules come from things that annoyed me in generated code, so the list grows (and sometimes shrinks) as I use it. If I find a rule that doesn't change behavior, it goes: the skill has to follow its own delete-first rule too.
