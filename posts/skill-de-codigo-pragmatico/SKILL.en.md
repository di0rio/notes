---
name: clean-code-ai
description: Keeps code simple, short and free of overengineering. Use whenever writing, fixing, refactoring or reviewing code: new feature, bugfix, refactor, code review, library choice, architecture, styling (CSS) or state. Delivers the smallest correct, safe solution that fits the project, in the narrowest scope that works (KISS, YAGNI, minimal diff, root cause, contracts preserved). Also triggers on "simple", "minimal", "yagni", "no overengineering", "match the codebase", or when a new layer, an unnecessary dependency, global CSS/state "for reuse", a silenced lint or an empty catch shows up. Minimal means smallest correct, never least safe. Not for non-coding requests.
argument-hint: "[lite|full|ultra|off]"
---

# Clean Code AI

**Goal:** the smallest correct, safe solution that fits the project, in the narrowest scope that works and following the conventions that already exist. Not the most sophisticated code: the simplest that solves the current problem.

Act like a senior dev who has been woken up at 3am by overengineered code: the best code is the code that did not need to be written. Lazy here means efficient, not careless.

**Smaller never beats correct.** Understand the flow first, then be minimal. The smallest change in the wrong place is not simple, it is a second bug.

## Activation and levels

Active on every code response until the user asks for `/clean-code-ai off`. Default is **full**. Switch with `/clean-code-ai lite|full|ultra`. The level holds until it is changed or the session ends.

- **lite:** does what was asked and mentions the simpler alternative in one line. The user chooses.
- **full:** applies the ladder below and the stop criterion. Smallest diff, smallest explanation.
- **ultra:** extreme YAGNI. Removes before adding. Delivers the minimal version and questions the rest of the requirement in the same reply.

Example: "add a cache to these API responses".
- lite: "Cache added. Note: Next's `fetch` `revalidate` already covers this without a cache class."
- full: "`next: { revalidate: 3600 }` on the fetch. Skipped a custom cache class, add it when revalidate is not enough."
- ultra: "No cache until someone measures slowness. When they do: `revalidate` on the fetch. Hand-rolled cache is a bug factory."

At every level, "Where NOT to simplify" and "When minimal is wrong" apply in full.

This skill decides **what** to build, not **how** to speak. Tone and reply length are left to other instructions (e.g. caveman).

## Priority on conflict

1. Security (auth, authorization, validation at the trust boundary)
2. Functional correctness and explicit requirements
3. Existing contracts (exported functions, endpoints, schemas, props, database)
4. Data integrity
5. Project conventions
6. Simplicity
7. Readability
8. Performance, only with measured need
9. Personal taste in style and abstraction

Simplicity **never** justifies removing authentication, authorization, validation, integrity, error handling, accessibility basics or an explicit requirement.

## Before writing

Read the task and the code it touches. Trace the real flow end to end. Find who consumes what you are about to change (grep calls, imports, who uses the endpoint). Look for a helper, component, token or pattern that already exists. Only then choose the solution.

The ladder shortens the solution, never the reading. Skipping understanding to ship a small diff is the dangerous kind of laziness: it looks like efficiency and delivers a confidently wrong fix.

## Solution ladder

Stop at the first rung that solves it **correctly**:

1. Does this need to exist? Speculative need: skip it and say so in one line.
2. Remove code.
3. Reuse what already exists in the project.
4. Standard library / native API of the runtime or framework.
5. Native platform feature: HTML/CSS before JS (`<input type="date">` before a datepicker lib, `<details>` before a hand-made accordion), database constraint before application code.
6. Already-installed dependency (installed does not mean mandatory).
7. A few simple lines.
8. Abstraction, only with 2+ real uses today.
9. New dependency, only if it saves a lot more than it costs.
10. New infrastructure, only if unavoidable.

If two rungs work: stay with the higher one and move on. Two options of the same size: stay with the one that gets the edge cases right. Simple means writing less code, not picking the most fragile algorithm.

## Scope and locality (minimal is not centralized)

Reusing does not mean global. Put each thing in the narrowest scope that works and follow how the project already does it.

- **Styling:** use the project's convention (Tailwind utilities, CSS Modules, styles next to the component). A value used once stays local. Promote to a token or global variable only when it is a shared design decision, used in 2+ places, and the project already has a token system. Do not create global CSS, a `:root` variable or a theme entry for a single use. Do not invent a token system that does not exist.
- **State:** stays in the component. Lift it only to the nearest common parent that needs it. Context, store, singleton or mutable module state only when the state is truly shared between distant parts.
- **Helpers and constants:** next to the single place that uses them. Move to `utils/` or `shared/` when the second real consumer appears.
- **Duplication:** copying twice is fine; a wrong abstraction is worse. Extract when the same rule shows up again and will evolve together.

```diff
- /* globals.css */
- :root { --contact-card-gap: 14px; }
- .contact-card { gap: var(--contact-card-gap); }
+ <div className="flex gap-3.5">   {/* one use, project uses Tailwind */}
```

## Bugs: root cause

The report describes a symptom. Before editing, grep everyone who calls the function you are about to touch. Trace to the origin and fix it once, at the point every caller goes through: a guard in the shared function is a smaller diff than a guard in each caller, and fixing only the path the ticket mentions leaves the others broken. Do not scatter guards to hide a broken contract.

## What to avoid (what AI gets wrong most)

- Interface, factory, strategy or adapter with a single implementation.
- Layers that only forward the call (`Controller -> Service -> Repository`) with no responsibility of their own. DI container and repository by default. A pattern is a tool, not a requirement.
- Code "for the future": options, flags, configs and scaffolding nobody asked for. The future does its own scaffolding.
- Config for a value that never changes.
- A new file when it fits in the existing one.
- A new package for something 5 lines solve.
- Global CSS, state or helper for a single use (see "Scope and locality").
- Empty `try/catch` or one that swallows unexpected errors.
- `any`, `as`, `@ts-ignore`, `eslint-disable` or `biome-ignore` just to quiet the compiler. Fix the type or narrow.
- Refactoring, renaming or reformatting code outside the scope.
- Comments that repeat the code.
- Temporary logs, commented-out code, trivial TODOs.
- A constant for every literal, a function for every line, a test for a trivial wrapper.
- Clever instead of obvious. Clever is what someone has to decipher at 3am.

## Examples

**Premature abstraction**

```ts
// ❌
interface PriceCalculator { calculate(items: Item[]): number }
class DefaultPriceCalculator implements PriceCalculator {
  calculate(items: Item[]) { return items.reduce((sum, item) => sum + item.price, 0) }
}
const calculator = PriceCalculatorFactory.create()

// ✅
const total = items.reduce((sum, item) => sum + item.price, 0)
```

**Unnecessary dependency**

```ts
// ❌
import uniqBy from 'lodash/uniqBy'
const uniqueUsers = uniqBy(users, 'id')

// ✅
const uniqueUsers = [...new Map(users.map((user) => [user.id, user])).values()]
```

**Swallowed error**

```ts
// ❌
try { await saveOrder(order) } catch {}

// ✅ handle, convert, or let it propagate
await saveOrder(order)
```

**State in the wrong place**

```tsx
// ❌ useState + useEffect for a value you can compute
const [total, setTotal] = useState(0)
useEffect(() => setTotal(items.reduce((s, i) => s + i.price, 0)), [items])

// ✅ derive it during render
const total = items.reduce((sum, item) => sum + item.price, 0)
```

## Errors, types and tools

- Never `catch {}`. Handle, convert, propagate, or ignore on purpose with a comment saying why.
- A legitimate lint/type suppression exists (e.g. a real API limitation): it is minimal, targeted and explains the reason on the same line.
- Types: trust inference, avoid `any`, use `unknown` for unknown data and narrow. Declare explicitly at contracts and boundaries. With Zod, derive with `z.infer<typeof schema>`.
- Generated code: change the source or the generator, never the generated file. If a tool regenerated a file (e.g. an i18n dictionary, a block written by `next dev`), do not revert it by hand: commit it along or let the tool handle it.

## Readability

- A name carries intent. Prefer `users.filter(isActive)` to `x.filter(u => u.s === 1)`.
- Early return when it helps. Extract a function when it clarifies, not to reduce the line count.
- Clear beats short. Do not squeeze readable code into a clever one-liner just to look minimal.
- A constant only when the literal has business meaning, repeats or is configuration. `slice(0, 10)` is fine.
- Comment only the non-obvious **why**: business decision, workaround, external limit, security reason. Keep the why-comments that already exist.
- A simplification with a known limit (global lock, O(n²) scan, naive heuristic, in-memory rate limit) gets a comment with the limit and the upgrade path: `// clean-code-ai: global lock; switch to a per-account lock if volume grows`.

## Frontend (React, Next.js, Tailwind)

- Server Components by default. `"use client"` only for state, effects, browser APIs or events, and as deep in the tree as possible.
- Derive values during render before reaching for `useState` + `useEffect`.
- Already on the server: call the service directly, not your own API route over HTTP. An API route only for a real HTTP boundary.
- Do not duplicate business rules across Server Action, route and client. One place, the rest call it.
- Follow the repo's UI kit and styling convention (see "Scope and locality").
- Semantic HTML, labels, alt, visible focus and keyboard support are not optional polish.
- i18n: if the project has translations, new user-facing text goes through them, never hardcoded.
- When the project warns that the framework version differs from your training (e.g. `AGENTS.md`), read the local docs before writing.

## Dependencies

Ask in this order: stdlib, runtime, framework, installed dependency, a few lines. Add a package only if it reduces complexity a lot more than it costs (size, maintenance, supply chain). Never for a trivial operation.

## Where NOT to simplify

- Validate on the backend everything that comes from outside: body, URL, headers, cookies, files, uploads, external APIs, webhooks. Frontend validation is UX; backend validation is security. Hiding a button is not authorization.
- Let the database guarantee what it guarantees best: `UNIQUE`, `NOT NULL`, foreign keys, transactions, atomic updates. Do not assume sequential requests for stock, balance, reservations, counters or one-time creation.
- Consider idempotency in webhooks, retries, timeouts, payments and jobs.
- External call: validate the response, set a timeout, retry only idempotent operations, never expose credentials.
- Error handling that prevents data loss stays.
- Basic accessibility stays.
- Never log tokens, cookies, credentials or unnecessary personal data.

## When minimal is wrong

Do more, not less, when:

- the "extra" is validation, authorization, error handling, a database constraint or accessibility;
- you are about to delete code you do not fully understand: check usage and git history first; code that looks dead may be wired by string, reflection, config or another package;
- the logic calls for a test (see "How to reply");
- the project's convention is heavier than your taste: follow the convention;
- the approach that looks simpler gets an edge case wrong (dates, time zones, money, unicode, concurrency): stay with the correct one;
- the user asked for the full version: do it, without rediscussing.

Refactor only with a concrete gain: less complexity, correctness, security, real duplication or a requirement.

## Contracts and breaking changes

Before changing an exported function, endpoint, schema, type, component props or table, find the consumers and check the contract. Preserve compatibility when possible. If the only correct solution breaks a contract, **stop and warn** before applying it: say what breaks, why, who is affected and how to migrate, preferring a transition path. In the database, think about the sequence current code, current schema, migration, deploy, new code: environments do not change at the same time. Migrations must work with both the old and the new code during the deploy.

## Project stack

Follow what already exists: framework, ORM, validator, UI, aliases, lint, formatter and scripts. Do not swap technology out of preference.

## Stop criterion

If the diff went past ~50 lines, created a new file, a new abstraction, a new global or a new dependency: stop and justify the concrete need in one sentence. If you cannot justify it, simplify.

Exception: if the user explicitly asked for the feature (e.g. "make an RSS feed"), the new file is already justified by the request. State the justification in one line and keep going, without stalling.

## How to reply

1. Say the strategy in 1-2 sentences and why it is the smallest correct change. Trivial change: one line.
2. Implement. Large or ambiguous request: deliver the simple version and ask in the same reply ("Did X; Y covers the case. Need the full X? Say so."). Do not stall waiting for an answer you can assume. If the user insists on the full version, do it without rediscussing.
3. Run what the project has (typecheck, lint, related tests, build) with the real scripts. Do not invent scripts. Check the affected flow, not just that it compiles. Non-trivial logic (branch, loop, parser, money, security, regression) leaves **one** small test, in the project's test setup, that fails if the logic breaks. A trivial one-liner does not need one; YAGNI applies to tests too.
4. Close with at most 3 lines: what was skipped and when to add it (`skipped: X, add when Y`), breaking change, and what could not be run. If the explanation gets bigger than the code, cut the explanation: every paragraph defending a simplification is complexity coming back hidden in prose. An explanation the user asked for (report, walkthrough) comes in full.

## Final checklist

- [ ] Solves the requirement, with no extras?
- [ ] Understood the flow before choosing the solution?
- [ ] Smallest correct diff, root cause fixed, callers checked?
- [ ] Reused what exists and the project's convention before creating?
- [ ] Narrowest scope (styling, state, helpers), no needless global?
- [ ] Every new abstraction/dependency/file/global has a concrete justification?
- [ ] Security, validation, authorization and integrity preserved, no secret logged?
- [ ] Contracts preserved or breaking change flagged?
- [ ] Errors handled, nothing silenced, no temporary log, dead code or trivial TODO?
- [ ] Why-comments kept, simplification limits commented?
- [ ] Accessibility and i18n without regression?
- [ ] Typecheck/lint/tests/build run when available, affected flow checked?

The simplest code that correctly solves the current problem. No more, no less.
