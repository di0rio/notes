---
name: pragmatic-code
description: Use on ANY coding task - writing, fixing, refactoring, reviewing, choosing libraries, styling, state - to deliver the smallest correct, safe, readable change that fits the project's existing conventions. Triggers on over-engineering, bloat, unnecessary dependencies, new abstractions or layers, global state or global CSS added "for reuse", silenced lint/type errors, empty catch blocks, or changes that may break callers. Also use when the user says "pragmatic", "minimal", "simplest", "yagni", "keep it small" or "match the codebase". Minimal means smallest correct and narrowest scope, never least safe. Not for non-coding requests.
---

# Pragmatic code

Goal: the smallest correct, safe, readable solution, in the narrowest scope that works, using the project's own conventions.

Smallest never beats correct. Understand the flow first, then be minimal.

## Priority on conflict

1. Security
2. Correctness and explicit requirements
3. Contracts, compatibility, data integrity
4. Project conventions
5. Simplicity (KISS/YAGNI)
6. Readability and maintainability
7. Performance (only with a concrete need)
8. Abstraction and style taste

Simplicity never justifies removing authorization, validation, integrity, error handling, accessibility basics or an explicit requirement.

## Before changing code

Read the task and the code it touches. Trace the real flow. Find the consumers of what you will change (grep callers, importers, endpoint users). Look for an existing helper, component, token or pattern. Only then pick a solution.

## Solution ladder

Stop at the first rung that solves it correctly:

1. Does it need to exist? Speculative need: skip it and say so in one line.
2. Reuse what is already in the codebase.
3. Standard library.
4. Native platform feature (HTML/CSS before JS, DB constraint before app code).
5. Dependency already installed (installed is not mandatory).
6. A few simple lines.
7. Abstraction, only for a concrete, present need.
8. New dependency or infrastructure, with justification.

Do not add by default: extra layers that only forward calls, one-implementation interfaces, one-product factories, repositories, DI containers, config for values that never vary. Patterns are tools, not requirements.

## Scope and locality (minimal is not centralized)

Reuse does not mean global. Put things in the narrowest scope that works and follow how the project already does it.

- Styling: use the project's existing convention (Tailwind utilities, CSS Modules, co-located styles, styled components). A one-off value stays local. Promote to a global token or variable only when the value is a real shared design decision, used in 2+ places, and the project already has a token system. Do not create global CSS, `:root` variables or a theme entry for a single use. Do not invent a token system that is not there.
- State: keep it in the component. Lift it only to the nearest common parent that needs it. Context, store, singleton or module-level mutable state only when state is truly shared across distant parts.
- Helpers and constants: next to the only place that uses them. Move to `utils/` or `shared/` when a second real consumer appears.
- Duplication: copying twice is fine; a wrong abstraction is worse. Extract when the same rule appears again and will evolve together.

```diff
- /* globals.css */
- :root { --contact-card-gap: 14px; }
- .contact-card { gap: var(--contact-card-gap); }
+ <div className="flex gap-3.5">   {/* one use, project uses Tailwind */}
```

## Bugs: root cause

A report names a symptom. Trace to the origin and fix once where all callers route through. Do not scatter guards to mask a broken contract. Do not patch only the path the ticket names.

## Contracts and breaking changes

Before touching an exported function, endpoint, schema, DB column, component props or type: list consumers, check the contract, preserve compatibility when possible. If the only correct fix breaks a contract, do not apply it silently: name the contract, say why, list who is affected, and prefer a transitional path. For DB changes, think current code, current schema, migration, deploy, new code; environments do not change at the same time.

## Trust boundaries and security

Validate everything crossing a boundary: user input, URL, cookies, headers, files, uploads, external APIs, webhooks. Frontend validation is UX; backend validation is security. Hiding a button is not authorization. Let the database guarantee uniqueness, foreign keys, atomicity (constraints, transactions), and assume concurrent requests for stock, balances, counters, unique creation. Consider retries and idempotency for webhooks, timeouts and payments. For external calls: validate responses, set timeouts, retry only idempotent operations, never expose credentials. Never log tokens, cookies or secrets.

## Errors and tools

- Never `catch {}`. Handle, convert, propagate, or ignore on purpose with a comment saying why.
- Do not add `@ts-ignore`, `eslint-disable`, `as any` or equivalents to make it pass. Fix the cause. A legitimate suppression is minimal and explained.
- Types: rely on inference, avoid `any`, use `unknown` for unknown data, derive types (`z.infer` with Zod). Declare explicitly at contracts and boundaries.
- Generated code: edit the source or generator, never the generated file.

## Readability

- Names carry intent. Prefer `users.filter(isActive)` to `x.filter(u => u.s === 1)`.
- Early returns when they help. Extract a function when it clarifies, not to cut line count.
- Clear beats short. Do not compress readable code into a clever one-liner to look minimal.
- Constants only when the literal has business meaning, is reused, or is configuration. `slice(0, 10)` is fine.
- Comments explain why: business decisions, workarounds, external limits, security reasons, a deliberate ceiling of a simplification. Keep existing comments that explain why. Delete commented-out code, trivial TODOs and temporary logs.

## Frontend (React, Next.js, Tailwind)

- Server Components by default; `"use client"` only for state, effects, browser APIs or event handlers, and as deep in the tree as possible.
- Derive values during render before reaching for `useState` + `useEffect`.
- Server already: call the service directly, not your own API route over HTTP. Keep API routes for real HTTP boundaries.
- Do not duplicate a business rule in Server Action, route, and client. One place, others call it.
- Follow the repo's UI kit and styling convention (see Scope). Use semantic HTML, labels, alt text, focus states and keyboard support; this is not optional polish.
- Respect i18n: if the project has translations, new user-facing strings go through them, not hardcoded.
- Check the framework version's docs when the project warns its APIs differ from your training data.

## Dependencies

Ask in order: stdlib, runtime, framework, installed dep, a few lines. Add a package only if it clearly reduces complexity more than its cost (size, maintenance, supply chain). Never for trivial operations.

## When minimal is wrong

Do more, not less, when:

- the "extra" is validation, authz, error handling, a DB constraint, or accessibility;
- deleting code you do not fully understand (check usage and git history first; unused-looking code may be wired by string, reflection, config or another package);
- the flow needs a test: non-trivial logic (branches, parsing, money, security, regression) gets one small runnable check, using the project's test setup;
- the project convention is heavier than your taste: follow it;
- the user asked for the full version: build it, do not re-argue;
- the simplest-looking approach is less correct on edge cases (dates, money, unicode, concurrency): pick the correct one;
- a deliberate simplification has a known ceiling: leave a short why comment with the upgrade path.

Do not reformat files, rename unrelated code or refactor for taste. Refactor only for a concrete gain: less complexity, correctness, security, real duplication, or a needed requirement.

## Before delivering

- [ ] Requirement met, nothing extra added?
- [ ] Root cause fixed, callers and contracts checked, breaking changes flagged?
- [ ] Reused existing code, convention and narrowest scope (style, state, helpers)?
- [ ] Every abstraction, dependency and global actually needed?
- [ ] Boundaries validated, authz, integrity and errors intact, no secrets logged?
- [ ] No silenced tools, no leftover logs, dead code or trivial TODOs; why-comments kept?
- [ ] Accessibility and i18n not regressed?
- [ ] Project's real typecheck, lint, tests, build run when applicable (never invent scripts), and the affected flow checked?

## Reply format

Code first. Then a short note proportional to the change: what was skipped and when to add it, any breaking change, any check you could not run. One sentence for trivial work. Do not explain the obvious.

Format: `[change] -> skipped: [X], add when [Y].`

The simplest code that correctly solves the current problem, no more, no less.
