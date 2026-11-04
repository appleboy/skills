---
name: plan-feature
description: Plan a feature before writing any code. Use this skill at the START of non-trivial feature work — when the user says they want to "add", "build", "implement", "create", "develop", or "ship" a feature, endpoint, component, page, service, or capability. Especially trigger when the user's prompt is short or under-specified ("add a dashboard", "build an auth flow", "let's do user activity reports") — those need planning, not direct execution. The skill walks through requirements clarification, codebase exploration, sketches a Mermaid architecture/flow diagram of the proposed design, and produces a plan.md document — with issue links, leaf/core classification, reproducible verification, and an evidence plan that pr-prepare later turns into the PR body — that the user can hand off to a fresh AI session for execution. Skip only when the change is genuinely trivial (typo fixes, single-line tweaks, renames). When in doubt, plan.
---

# plan-feature

Most failed AI-coding sessions don't fail because the model is incapable. They fail because the human didn't give it enough context. This skill turns the user into Claude's product manager for ~15 minutes before any code gets written, dramatically increasing the success rate of the actual implementation.

The output is a `plan.md` file that captures everything a competent engineer would need to ship the feature without ambiguity — including a diagram of the proposed design, because an implementer grasps the intended shape far faster from a picture than from a prose description.

`plan.md` is also the input the `pr-prepare` skill reads when the work is ready for review. It looks there for issue identifiers (to name the branch and link the PR), the leaf/core classification, the acceptance conditions and setup a reviewer needs to reproduce verification, and the evidence plan (screenshots, performance report). Deciding these now is cheap; reconstructing them from a finished diff is guesswork. The template below is shaped so each section maps onto a section of the PR body.

## When to use

Use whenever the user asks to build something non-trivial. Triggers include:

- "Add a feature for X"
- "Build / implement / create / ship X"
- "I want to add X to the app"
- "Let's do X"
- Any prompt that names a feature without giving a file path

Do not use for: typo fixes, single-line config changes, simple renames, or when the user has already provided a complete spec.

## The eight-step workflow

Follow these in order. Stop at the user-approval step before drafting code.

### Step 1 — Clarify the goal

Ask the user, in 1-3 questions:

- Who is the user / consumer of this feature? (internal team, end user, system)
- What does "done" look like? (one or two sentences)
- Are there constraints already known? (deadlines, tech stack, must-not-touch areas)

Also capture any tracking issue: a GitHub/Gitea issue (`#123` or an issue URL) or a Jira key (`GAIA-5375`). Take identifiers only from what the user says or pastes — never guess one from a similar title — and if there is none, record `N/A` rather than asking repeatedly. `pr-prepare` builds the branch name and PR links from this field.

Do not move on until you have a one-paragraph problem statement that the user has confirmed.

### Step 2 — Explore the codebase

Look around before designing. Use Read, Grep, and Glob to:

- Find the directory likely to host the new code
- Identify 1-2 existing features that are structurally similar (these will be the patterns to follow)
- Note conventions: naming, file layout, test framework, error-handling style
- Identify shared utilities the feature will likely reuse
- Identify the boundaries of `core` code (auth, payment, schema, shared frameworks) vs `leaf` code (the feature itself)

If the codebase is large or unfamiliar, ask the user pointed questions: "Where does auth happen?", "Show me an existing feature similar to X."

### Step 3 — Identify scope

Produce two lists, validated with the user:

- **Files / modules that may be modified**
- **Files / modules that must NOT be modified** (especially core code that shouldn't accumulate AI-written tech debt)

If the feature would naturally touch core code, surface that explicitly and discuss whether the change should be split into a human-led core change plus a leaf-node feature change.

Record the resulting classification in the plan, using the stricter category that applies:

- **Leaf**: a failure stays local — a report, an isolated endpoint, a UI component, a one-off script.
- **Core**: other components depend on it — auth, payments, schemas, shared frameworks, public APIs, orchestrators.

If the blast radius is unclear, ask: "If this code is buggy, how far would the failure spread?" The classification decides how many reviewers the PR needs, so it belongs in the plan rather than being re-derived later. Also flag when the feature changes auth, payment, permission, or an external interface — those need a security review before merge, and the reviewer should know that from the start.

### Step 4 — Decide the verification strategy

Before any code is written, agree on how correctness will be checked. Write it so that a reviewer who has never seen this conversation could reproduce it — that is exactly what the PR will later need to show.

- **3 acceptance scenarios** (1 happy path, 2 error or boundary cases), each with a starting state (data, role, config), a concrete action (request, command, UI navigation with real inputs), and an observable expected result. Keep them at the user-observable level, not implementation details. For a bug fix, one scenario is the original trigger; for a refactor, one is the behavior that must stay unchanged.
- **Setup**: the prerequisites (services, config names, test data, account roles) and the commands to start the system, taken from the repository's README, scripts, or Makefile — not invented.
- **Automated checks**: the repository's actual test / lint / build commands that will cover this feature, found during Step 2. If a check doesn't exist yet (e.g. a new test file), say what will be added.
- **Manual verification**, if any.
- For long-running or async features: what stress / soak test is needed, and what signals completion.
- How the feature will be observable in production (logs, metrics).

Then decide the **evidence plan** — what a reviewer will need to *see* beyond passing tests, because a UI change can't be judged from text and a performance claim can't be judged without numbers. Apply every row that matches:

| Signal in the goal / scope | Evidence to plan |
| --- | --- |
| UI components, templates, styles, user-visible copy, or layout | Before/after screenshots: which pages, which states, desktop 1280×800 (plus mobile 390×844 if responsive), and the data/role used |
| A performance goal or constraint (latency, throughput, memory, bundle size), or work on hot paths, queries/indexes, caching, concurrency | Performance report: which benchmark or workload, which metrics (median, p95), dataset size, base vs head |
| Pure backend logic, docs, CI, or a dependency bump with no performance claim | `N/A` with a one-line reason |

Any numeric constraint you put in the plan (say, "latency < 500ms") is a claim someone will have to prove, so pair it with how it will be measured. If no measurement method exists, either plan one or drop the number.

Verification design comes BEFORE implementation design — this is the "verifiable checkpoints" principle.

### Step 5 — Sketch the design as a diagram

By now you know the goal, the surrounding code, and the scope — enough to draw _how the feature will fit together_ before any of it exists. Drawing it makes the proposed design reviewable: a wrong boundary or a missing call is easy to catch in a diagram, easy to miss in a paragraph. Draw it as a **Mermaid** diagram so it renders natively inside `plan.md` on GitHub/Gitea, travels with the plan, and diffs cleanly as the design evolves.

Pick the diagram type from what the feature mainly _is_:

| The feature is mainly about…                                                                                 | Use               | Mermaid header                                |
| ------------------------------------------------------------------------------------------------------------ | ----------------- | --------------------------------------------- |
| Interactions/handshakes over time (auth flow, request lifecycle, calls between services / client↔server)     | Sequence diagram  | `sequenceDiagram`                             |
| Control flow, decision logic, or a pipeline (branching, data transform, job steps)                           | Flowchart         | `flowchart TD`                                |
| New or rewired modules / packages / services (structural dependencies)                                       | Component diagram | `flowchart LR` with one `subgraph` per module |
| A state machine (status transitions, token / job lifecycle)                                                  | State diagram     | `stateDiagram-v2`                             |

Rules that keep the diagram honest and renderable:

- **Anchor every node to the plan's scope, not to imagination.** Each box must map to a file/module the "May modify" list will create or touch, or to an existing component it integrates with. A diagram of an aspirational architecture the plan won't actually build misleads the implementer — the same failure mode as a vague goal section.
- **Mark what's new vs existing.** Give new elements a distinct node style, e.g. `style NewSvc fill:#dff0d8,stroke:#3c763d`, and draw the existing code the feature plugs into in the default style so the boundary is obvious. Per-node `style` lines work; `%%{init: ...}%%` theme directives do **not** — GitHub strips them.
- **Stay under ~15–20 nodes.** GitHub falls back to plain text (or times out) on very large/complex diagrams. If the design won't fit in that budget, that's a signal the feature should probably be split into smaller plans.
- **Prefer `TD` over `LR` when wide.** GitHub renders inside the markdown column; wide diagrams force horizontal scroll. Group related nodes in `subgraph` blocks so they wrap.
- **No `click` handlers / interactivity** — GitHub renders a static SVG and ignores them.

If the feature is genuinely trivial, skip the diagram and say so — don't draw a one-box graph. The user confirms the diagram during plan review (Step 7).

### Step 6 — Draft the plan.md

Write the plan to `plan.md` in the working directory using this template. Be concrete, not abstract.

`plan.md` is a working document for the author and the executing session, not part of the change: do not commit it or add it to the scope list. `pr-prepare` summarizes its goal and scope into the PR body's "Plan reference" section instead. If the repository's `.gitignore` doesn't already cover it, mention that to the user so it isn't staged by accident.

````markdown
# Plan: <feature name>

## Goal
<1-paragraph problem statement, who uses it, what done looks like>

## Related issues
- Jira: <GAIA-5375, or N/A>
- GitHub/Gitea: <#123 or issue URL, or N/A>

## Architecture / flow
<Mermaid diagram of the proposed design. Omit this section for trivial features.>

```mermaid
flowchart TD
    Req[GET /reports/active-users] --> Auth[admin auth middleware]
    Auth --> Svc[reports/active-users service.ts]
    Svc --> Q[query: logins last 7d by channel]
    Q --> DB[(users / sessions tables)]
    Svc --> Resp[JSON: channel, count, last_seen]
    style Req fill:#dff0d8,stroke:#3c763d
    style Svc fill:#dff0d8,stroke:#3c763d
    style Q fill:#dff0d8,stroke:#3c763d
    style Resp fill:#dff0d8,stroke:#3c763d
```

## Scope

### May modify
- src/path/to/file.ts
- src/path/to/dir/

### Must not modify
- src/auth/*
- src/db/schema.ts

## Change classification
- Leaf | Core: <one, with a one-line blast-radius reason>
- Security review needed: <yes — what auth/payment/permission/external interface changes | no>

## Existing patterns to follow
- Mirror the structure of `src/reports/revenue.ts`:
  - `service.ts` for queries
  - `route.ts` for HTTP handler
  - `*.test.ts` for tests

## Constraints
- No new third-party dependencies
- Query latency < 500ms (measured as described in Evidence plan)
- Must use existing admin auth middleware

## Verification

### Setup
- Prerequisites: <services, config/env var names, test data, account roles>
- Start: <exact commands from the repo, and the signal that it's ready>

### Automated checks
- `<repo command>` — <what it covers>
- `<repo command>` — <what it covers>

### Acceptance scenarios
1. Happy path — <name>
   - Starting state: <data, role, config>
   - Action: <concrete request / command / UI steps with real inputs>
   - Expected: <observable result>
2. Error case — <name>
   - Starting state: …
   - Action: …
   - Expected: …
3. Error case — <name>
   - Starting state: …
   - Action: …
   - Expected: …

- Stress / soak test: <if applicable, or N/A>
- Manual verification: <if applicable, or N/A>

## Evidence plan
- Screenshots: <pages, states, viewports, data/role — or N/A with reason>
- Performance report: <benchmark/workload, metrics, dataset size, base vs head — or N/A with reason>

## Done definition
- [ ] All automated checks and acceptance scenarios pass
- [ ] Evidence in the Evidence plan is captured (or marked Not run with a reason)
- [ ] No changes outside the "may modify" list
- [ ] <other domain-specific criteria>
- [ ] PR opened with `pr-prepare` (AI authorship disclosed, classification and verification carried over from this plan)

## Risks & rollback
- Risk: <what could go wrong>
- Rollback: <how to revert>

## Open questions
- <anything still ambiguous>
````

### Step 7 — Get explicit user approval

Show the plan to the user and ask: "Does this look right? Anything to add or remove?" Walk through the diagram with them explicitly — a wrong arrow or a missing module is far cheaper to fix here than after the code is written.

Do not skip this step. The plan is only useful if the user has actually read it and corrected any drift.

If the user wants to write the code right now in this conversation, that's fine — but the plan is still the contract. Refer back to it.

### Step 8 — Recommend handoff

After approval, recommend the user:

1. Compact the current conversation (or open a new one)
2. Start the next conversation with: "Execute the plan in plan.md. Do not deviate from the scope. Stop and ask if you find ambiguity."
3. When the implementation is done, run `/pr-prepare`. It reads `plan.md` for the issue identifiers, classification, verification, and evidence plan, so the PR body reflects what was agreed here instead of what's reconstructable from the diff.

This separation matters: the planning conversation often fills 80k+ tokens of exploration. Compacting drops that down to a few thousand and gives the executing conversation a clean focus.

Remind the user that `plan.md` stays out of the commit.

## Anti-patterns to avoid

- **Over-constraining the prompt.** Don't dictate every variable name. Give the model the constraints that matter and let it choose the rest. Treat the plan like onboarding documentation for a junior engineer, not a fill-in-the-blanks form.
- **Skipping codebase exploration.** A plan written without reading the existing code will produce code that doesn't fit the codebase. Always look first.
- **Vague verification.** "It works" is not verification. Three concrete scenarios — starting state, action, observable result — plus the repo commands that run them is verification.
- **Unbacked numbers.** A latency or memory target without a measurement method becomes an unprovable claim in the PR. Plan the measurement or drop the number.
- **Letting the plan touch core code without flagging.** If the feature requires changing core, that needs human ownership and a separate decision — surface it, don't bury it.
- **Drawing a diagram that doesn't match the scope.** A picture of an aspirational design the plan won't build is worse than no picture — re-anchor every node to the scope (Step 5), or split the feature if it won't fit.

## Examples

### Good plan goal section

```
## Goal
Internal ops team needs a daily report of users who logged in within
the last 7 days, broken out by signup channel. Done means: an admin-
only endpoint at GET /reports/active-users that returns JSON with
{channel, count, last_seen} rows, sortable by count.
```

### Bad plan goal section

```
## Goal
Active users report.
```

(Too short. The implementer cannot tell who uses it, what "active" means, or what fields to return.)

## Stopping conditions

End the skill when:

1. `plan.md` exists, the user has approved it, and the handoff recommendation has been given, OR
2. The user pivots to wanting to do the work directly in this conversation (in which case the plan still serves as the contract — refer to it during execution).
