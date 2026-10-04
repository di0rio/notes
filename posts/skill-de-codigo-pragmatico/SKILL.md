---
name: pragmatic-code
description: Self-documenting code and pragmatic engineering rules for AI coding agents - deliver the smallest correct, safe, readable solution compatible with the current project.
---

# Skill: Self-documenting Code and Pragmatic Engineering for AI

This skill defines how the AI should analyze, generate, fix, refactor, and review code.

> **Goal: deliver the smallest correct, safe, readable solution compatible with the current project.**

## 0. AI Response Protocol

When proposing or implementing a solution:

1. Understand the problem and the context before changing code.
2. Briefly state the strategy and why it is the smallest correct change.
3. Deliver the implementation only after that analysis.
4. Do not introduce architecture, abstractions, or new files without explaining the concrete need.
5. If more than one valid solution exists, prefer the simplest and explain the relevant trade-off.
6. If the solution requires a change incompatible with an existing contract, do not apply the change silently: flag the **breaking change** before proceeding.

The explanation should be proportional to the problem. For trivial changes, a short justification is enough.

## 1. Rule Priority

On conflict, follow:

1. Security
2. Functional correctness
3. Explicit requirements
4. Contract integrity and compatibility
5. Data integrity
6. Simplicity - KISS/YAGNI
7. Readability
8. Maintainability
9. Performance when there is a concrete need
10. Abstraction
11. Stylistic preferences

Simplicity never justifies removing authorization, validation, integrity, error handling, or explicit requirements.

## 2. Mindset

### YAGNI
Implement only the current requirement. Do not create speculative support, abstractions "for the future", single-implementation interfaces, single-type factories, unnecessary cache, or distributed infrastructure without a requirement.

### KISS
Complexity must be proportional to the problem.

### Deletion First
Before adding code:

```text
DELETE → REUSE → SIMPLIFY → NATIVE → EXISTING DEPENDENCY → SIMPLE CODE → ABSTRACTION → NEW DEPENDENCY → INFRASTRUCTURE
```

### Root Cause First
For bugs, trace the flow and fix the cause at the origin. Do not scatter guards to mask a broken contract.

### Minimal Diff
Change only what is necessary. Do not reformat files, rename unrelated code, or refactor for taste.

> The smallest correct diff is valid only after understanding the full flow.

## 3. Protocol Before Changing Code

Before writing:

1. Identify the requirement.
2. Read the relevant files.
3. Identify consumers.
4. Check contracts.
5. Look for existing implementations and utilities.
6. Check runtime and dependencies.
7. Determine the smallest correct solution.

Do not choose the solution before understanding the problem.

## 4. Solution Hierarchy

Stop at the first level that solves it correctly:

1. Remove code.
2. Reuse existing code.
3. Use the Standard Library.
4. Use native runtime/framework APIs.
5. Use already installed dependencies.
6. Write a simple implementation.
7. Create an abstraction only for a concrete need.
8. Add a dependency only with justification.
9. Create infrastructure only when unavoidable.

An installed dependency is not a mandatory dependency.

## 5. Anti-Overengineering

Do not automatically introduce:

- Clean Architecture
- DDD
- Repository
- Unit of Work
- Factory
- Adapter
- Strategy
- CQRS
- Event Bus
- Dependency Injection container
- microservices

Patterns are tools, not requirements.

Do not create empty layers that only pass calls through:

```text
Route → Controller → Service → Repository → Database
```

if no layer adds a real responsibility.

## 6. Self-documenting Code

Code should explain intent through names and structure.

Rules:

- clear, searchable names;
- consistent vocabulary;
- avoid obscure abbreviations;
- avoid generic names when better context exists;
- function names should express their action;
- do not repeat context unnecessarily.

Prefer:

```ts
const activeUsers = users.filter(isActiveUser);
```

to:

```ts
const x = users.filter(u => u.status === 1);
```

## 7. Functions

Functions should have a clear responsibility.

Do not split a function only to reduce line count.

Avoid functions that simultaneously validate, transform, persist, and format when those responsibilities can be separated naturally.

Few arguments are preferable. For many related arguments, an object can improve clarity.

Boolean flags that significantly change behavior may indicate multiple responsibilities.

## 8. Conditionals

Prefer early returns when they improve readability.

Do not turn every `if` into a function. Extraction should increase understanding.

Avoid mental mapping:

```ts
items.map((x) => ...)
```

when a contextual name would make the intent clearer.

## 9. Constants

Do not turn every literal into a constant.

Extract when there is:

- business meaning;
- reuse;
- risk of inconsistency;
- configuration;
- relevant maintenance.

`items.slice(0, 10)` can be better than an artificial constant.

## 10. Duplication

Eliminate duplication when it represents the same rule.

Simple duplication can be preferable to a wrong abstraction.

Before abstracting, check:

1. behavior is actually the same;
2. they will evolve together;
3. real reuse;
4. complexity reduction.

## 11. Side Effects

Side effects should be explicit.

Avoid functions that look pure but modify global state or external objects.

Do not treat immutability as dogma: mutation can be correct when sharing, performance, or the API justify it.

## 12. TypeScript

- Do not declare redundant types when inference is enough.
- Avoid `any`.
- Prefer `unknown` for genuinely unknown data.
- Use `typeof`, `keyof`, and derived types.
- Prefer `z.infer<typeof schema>` when Zod is present.
- Do not use `as` only to silence the compiler.
- Fix the contract or narrow before using assertions.

Declare types when they are part of contracts, boundaries, APIs, or relevant types.

## 13. Trust Boundaries

Validate data coming from:

- user;
- browser;
- HTTP;
- URL;
- cookies;
- headers;
- files;
- external integrations;
- external services.

Frontend validation improves UX; backend validation guarantees security and integrity.

## 14. Authorization

Never trust the frontend for security.

Hiding a button is not authorization.

The protected operation must validate permission on the backend.

## 15. Integrity and Concurrency

When a rule must be guaranteed regardless of the path, consider database constraints and mechanisms:

- `UNIQUE`
- `NOT NULL`
- foreign keys
- check constraints
- transactions
- atomic updates

Evaluate concurrency for stock, balances, reservations, unique creation, counters, and destructive operations.

Do not assume requests are sequential.

## 16. Idempotency

Evaluate idempotency in operations subject to retry, timeout, refresh, webhook, or distributed processing.

Do not create idempotency infrastructure without need, but never ignore duplication in critical operations.

## 17. Errors

Never silence errors:

```ts
try {
  await operation();
} catch {}
```

An error must be handled, converted, propagated, or deliberately ignored when that is part of the contract.

Do not use `try/catch` to hide unexpected errors.

## 18. Logs

Do not add temporary logs.

When logging is necessary:

- use the project's official mechanism;
- record useful context;
- never log tokens, cookies, credentials, or confidential data;
- avoid unnecessary personal data.

## 19. Comments

Priority:

```text
Clear code → meaningful name → clear function → structure → abstraction → exceptional comment
```

Do not comment the obvious.

Comments are allowed for:

- non-obvious business decisions;
- external limitations;
- workarounds;
- security requirements;
- counterintuitive behavior;
- a deliberate ceiling of a simplification.

Comments should explain **why**, not repeat **what**.

Do not keep old commented-out code. Git already has history.

Do not create trivial TODOs.

## 20. Performance

Do not optimize prematurely.

But consider complexity when volume justifies it.

Performance should be guided by:

- volume;
- latency;
- memory;
- cost;
- profiling;
- metrics;
- explicit requirements.

Do not trade clarity for speculative optimization.

## 21. Database

Let the database guarantee what it does best:

- uniqueness;
- referential integrity;
- atomicity;
- filters;
- aggregations.

Do not implement in TypeScript a guarantee the database can keep more safely.

Also do not move every business rule into SQL without need.

## 22. Transactions

Use a transaction when multiple operations need to be atomic.

Do not wrap a trivial query in a transaction out of habit.

## 23. Dependencies

Before adding a package:

1. Does the Standard Library solve it?
2. Does the runtime solve it?
3. Does the framework solve it?
4. Does an existing dependency solve it?
5. Do a few simple lines solve it?
6. Does the library reduce complexity enough to justify its cost?

Do not add a package for trivial operations.

## 24. Compatibility

Before changing a function, endpoint, schema, database, component, or exported type:

1. identify consumers;
2. check the contract;
3. assess impact;
4. preserve compatibility when needed;
5. update consumers only when necessary.

### Breaking Changes

If the only correct solution requires an incompatible change:

1. identify the affected contract explicitly;
2. explain why the change is necessary;
3. assess migration and affected consumers;
4. do not apply the breaking change silently;
5. when possible, prefer a compatible or transitional strategy.

A deliberate breaking change can be correct when the requirement demands it, but it must be treated as an explicit decision, not as a side effect of the implementation.

## 25. Migrations

Database changes must consider the transition:

```text
Current code → Current schema → Migration → Deploy → New code
```

Do not assume all environments change at the same time.

For destructive changes, consider compatibility during the transition.

## 26. Security

Never simplify by removing:

- authentication;
- authorization;
- validation;
- necessary sanitization;
- injection protection;
- secrets protection;
- rate limiting when needed;
- payload limits;
- access controls;
- file validation;
- domain-specific protections.

Security is not overengineering when a real trust boundary exists.

## 27. Uploads

Do not rely only on the name and MIME type provided by the client when there is risk.

Depending on the case, consider:

- size;
- actual type;
- content;
- storage;
- authorization;
- temporary URLs;
- isolated processing.

For large files, prefer a dedicated upload flow instead of indiscriminately raising the API's global limit.

## 28. External Resources

When consuming external APIs:

- validate responses;
- handle timeout;
- use retry only when appropriate;
- avoid unsafe retry of non-idempotent operations;
- do not expose credentials;
- handle unavailability.

## 29. Global State

Before creating a store/context/provider:

> Does this state really need to be shared?

If it belongs to a component, keep it local.

## 30. Frontend

- prefer simple components;
- avoid unnecessary state;
- avoid premature abstractions;
- keep Server Components when appropriate;
- use Client Components only when necessary;
- do not duplicate business rules on the client;
- do not trust the client for security.

## 31. Backend

- validate inputs;
- authenticate;
- authorize;
- centralize business rules;
- avoid duplication across endpoints;
- handle errors;
- keep contracts predictable.

## 32. API

Do not make internal HTTP calls without need.

If already on the server:

```text
Server Component → Service
```

can be preferable to:

```text
Server Component → HTTP → API → Service
```

The API should exist when there is a real HTTP boundary.

## 33. Business Rules

Avoid duplicating the same rule in:

```text
Server Action
HTTP Route
Server Component
Client Component
Job
Webhook
```

when all of them execute the same operation.

## 34. Generated Code

If the project has code generation:

```text
Source → Generator → Generated File
```

change the source, not the generated file.

## 35. Configuration

Do not create configuration for values that never vary.

Configuration should represent a real variation.

## 36. Tests

Test relevant behavior, especially:

1. business rules;
2. security;
3. non-trivial transformations;
4. regressions;
5. important integrations.

Do not create artificial tests for trivial wrappers.

For non-trivial logic, leave an adequate executable check when the ecosystem allows it.

## 37. Validation Before Delivery

When available in the project:

1. typecheck;
2. lint;
3. related tests;
4. build;
5. the affected flow.

Use the project's real commands.

Do not invent scripts.

## 38. Do Not Silence Tools

Do not use `@ts-ignore`, `eslint-disable`, or equivalents only to make the implementation pass.

Fix the cause.

Legitimate suppressions must be minimal and justifiable.

## 39. Respect the Ecosystem

Follow:

- framework;
- runtime;
- ORM;
- validator;
- UI;
- aliases;
- structure;
- conventions;
- lint;
- formatter;
- existing scripts.

Do not replace existing technology merely out of preference.

## 40. Do Not Refactor for Style

Do not refactor only because another syntax looks prettier or another architecture looks more elegant.

Refactor when there is a concrete benefit:

- less complexity;
- correctness;
- security;
- maintenance;
- elimination of real duplication;
- a necessary requirement.

## 41. Quality Gate

Before delivering:

- [ ] Requirement met?
- [ ] Extra features avoided?
- [ ] Smallest correct diff?
- [ ] Root cause fixed?
- [ ] Consumers checked?
- [ ] Contracts preserved?
- [ ] Existing code reused?
- [ ] Removable code removed?
- [ ] Abstractions actually necessary?
- [ ] Dependencies actually necessary?
- [ ] No redundant comments?
- [ ] No commented-out code?
- [ ] No trivial TODOs?
- [ ] No temporary logs?
- [ ] No sensitive data leaked?
- [ ] Trust boundaries protected?
- [ ] Authorization preserved?
- [ ] Data integrity preserved?
- [ ] Errors handled?
- [ ] Concurrency evaluated when relevant?
- [ ] Retry/idempotency evaluated when relevant?
- [ ] Complexity adequate for the volume?
- [ ] Typecheck run when applicable?
- [ ] Lint run when applicable?
- [ ] Tests run when applicable?
- [ ] Build run when applicable?
- [ ] Affected flow validated?
- [ ] Was the strategy explained in proportion to the problem?
- [ ] If there is a breaking change, was it explicitly flagged?

## 41.1. Context Prioritization

When available context is large, the AI should prioritize:

1. the current requirement;
2. directly affected code;
3. contracts and consumers;
4. security and integrity rules;
5. existing conventions;
6. relevant dependencies and infrastructure;
7. general style rules.

Do not sacrifice correctness, security, or contracts to save context.

Repeated rules can be consolidated mentally, but they must not be contradicted by an attempt to reduce text.

## 42. Special Rule for AI

"Best practice" is not an absolute rule.

Do not:

- turn every literal into a constant;
- create a function for every line;
- create an interface for a single implementation;
- use abstraction because "Clean Code says so";
- apply immutability by dogma;
- create a test with no relevant behavior;
- create an architectural layer with no responsibility.

The AI must evaluate context and trade-offs.

## 43. Golden Rule

Before adding code:

> **Can I remove it?**

If not:

> **Does it already exist?**

If not:

> **Does the platform already solve it?**

If not:

> **Does an existing dependency solve it?**

If not:

> **Do a few simple lines solve it?**

If not:

> **Is there a real need for abstraction?**

Only then consider a new dependency or infrastructure.

> **Fully understand the problem before trying to solve it minimally.**

## 44. Final Principle

```text
LESS CODE
     +
LESS COMPLEXITY
     +
MORE CLARITY
     +
SECURITY PRESERVED
     +
CONTRACTS PRESERVED
     =
BETTER SOLUTION
```

> **Do not write the most sophisticated code possible. Write the simplest code possible that correctly solves the current problem.**
