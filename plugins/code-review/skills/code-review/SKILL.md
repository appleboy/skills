---
name: code-review
description: Review a Pull Request or code change with rigor calibrated to whether it's leaf-node or core code. Use this skill whenever the user asks to review, audit, or evaluate code — phrases like "review this PR", "audit this code", "is this PR ready", "look at this branch", "review my changes", "what do you think of this", "code review please". Also trigger when reviewing AI-authored code specifically. The skill applies different review intensity based on classification (leaf gets spot-checks, core gets line-by-line), validates the PR description, checks for anti-patterns, and outputs structured review comments ready to paste into GitHub. Trigger for both peer reviews and self-reviews before submitting.
---

# code-review

The reviewer's real job isn't to inspect AI output. It's to verify that the person submitting the PR did the product-manager work properly: clear plan, appropriate classification, honest AI disclosure, sufficient verification. This skill enforces that, while applying the right review depth to the actual code.

## When to use

Use whenever the user asks for code review. Triggers include:

- "Review this PR / branch / change"
- "Audit this code"
- "Is this ready to merge"
- "Take a look at <PR url or branch>"
- "Self-review before I submit"

If the user just shares a diff with no explicit ask, ask: "Want me to review this?"

## Reviewer mindset

Before reading any code, internalize:

1. **You are not checking AI's work.** You are checking that the human submitter did their PM job — plan, scope, classification, disclosure, verification.
2. **Calibrate depth to risk.** Core code requires line-by-line reading. Leaf nodes can be judged from interface and tests. Don't waste effort over-reading leaves; don't dangerously under-read core.
3. **Standard rejection reasons exist** (see below). Use them. They're not personal — they're the team contract.

## Workflow

### Step 1 — Read the PR description first

Before opening the diff, read the PR description and check for:

- **AI authorship disclosure** — present? specific about which files?
- **Change classification** — leaf or core? does the diff match?
- **Plan reference** — is there a plan.md or equivalent?
- **Verification details** — what tests? what manual checks?
- **Reviewer guide** — does it say what to read carefully vs spot-check?

If any of these are missing or evasive ("AI was used a bit"), this is a procedural fail. **Reject immediately with a kind, specific request to fix the description before code review begins.** Don't waste effort reviewing code for a PR that isn't ready for review.

### Step 2 — Classify the change yourself

Don't just trust the PR's self-classification. Look at the diff:

- Touched files in `auth/`, `payment/`, `db/schema`, public APIs, shared utilities → **core**
- Touched files only in feature directories, single endpoints, UI components, scripts → **leaf**
- Mixed → **core** (strictest rule wins)

If your classification differs from the PR's, that's a flag — discuss before continuing.

### Step 3 — Apply review intensity

#### For LEAF NODE PRs

- Read the PR description carefully (~1-2 minutes)
- Read the public interface / signatures (~2-3 minutes)
- Read the tests carefully — do they cover happy + at least 2 error paths? are they at user-observable level (not implementation-coupled)?
- Spot-check one or two interesting-looking implementation files
- Run the tests yourself if possible
- **Trust the tests + interface.** Do not read every line.

#### For CORE CODE PRs

- Read the entire diff line by line. No skipping.
- Verify the human submitter (not AI) wrote or carefully reviewed every line in core paths
- Check for breaking changes to interfaces other code depends on
- Check for new tech debt: hacks, TODOs, commented-out code, magic numbers, unclear naming
- Check observability: are logs / metrics in place for important paths?
- Pull in the module owner if you're not them
- Run tests, including stress / soak tests if applicable

### Step 4 — Check for anti-patterns

Flag any of these explicitly. They're standing rejection reasons:

1. AI mass-changed core code without human line-by-line review noted
2. PR has no tests, only "I ran it locally"
3. AI authorship not disclosed
4. Tests are too implementation-coupled (refactor will break them)
5. Long-running / async code without stress test
6. Touches files outside the announced scope
7. Cross-module sprawling change that should be split
8. Secrets / credentials in the diff
9. New external dependencies added without justification
10. Commit history is a mess of "wip" / "fix" with no narrative

### Step 5 — Output structured review

Use this format for the review. Order: blocking issues first, then non-blocking, then questions.

```markdown
## Review summary
<1-2 sentences: overall verdict — approve / changes requested / blocked>

## Classification
I'm reviewing this as **[LEAF / CORE]** because <reason>.
<If your classification differs from the PR's: flag it here.>

## Blocking issues
<Items that must be fixed before merge. Use this format:>

### B1 — <short title>
**Where**: `src/path/file.ts:42`
**Issue**: <description>
**Why it matters**: <impact>
**Suggested fix**: <concrete suggestion>

## Non-blocking suggestions
<Nice-to-haves. Same format but no obligation to fix.>

## Questions
<Things you don't understand or want clarified. Don't approve until answered.>

## What I checked
- [x] PR description completeness
- [x] Classification matches diff
- [x] Tests cover happy + error paths
- [ ] Line-by-line read of core sections (only required for core PRs)
- [x] No anti-patterns from team checklist
- [x] No secrets / dependencies issues
```

### Step 6 — Communicate verdict

End with one of three clear statements:

- **Approve** — looks good, ready to merge.
- **Approve with nits** — non-blocking suggestions, but OK to merge.
- **Changes requested** — blocking issues. List them in priority order.

## Standard rejection reasons (use these verbatim if applicable)

- "No plan.md or plan summary in the description — please add before I can review."
- "Core code changes need line-by-line review notes per team rule. Please mark which files you reviewed line-by-line."
- "No stress test for long-running service. Please add a soak test before merge."
- "Tests cover happy path only. Please add 2 error cases (we use 3 e2e tests minimum)."
- "Diff modifies files outside the scope you declared. Please split or update the scope."
- "Tests are implementation-coupled — they will break on refactor. Please rewrite at the user-observable level."

## Tone

Be concrete and kind. The submitter is a teammate, not an opponent. Phrase issues as a request, not an accusation. "Could you add..." beats "You forgot to add...". But don't pad — if something's wrong, say it clearly. Reviewer ambiguity hurts more than reviewer bluntness.

## Examples

### Good review comment

> **B1 — Tests don't exercise error paths**
> **Where**: `src/reports/activity.test.ts`
> **Issue**: Only one test, and it's the happy path. Team minimum is 3 e2e tests (1 happy + 2 errors).
> **Why it matters**: We rely on tests to validate AI-authored code without re-reading every line. Without error-path coverage, we can't trust the leaf-node rules.
> **Suggested fix**: Add tests for (a) unauthorized request returns 401, (b) empty result returns []. Both can be ~10 lines each.

### Bad review comment

> "LGTM"
>
> (Especially when the PR is 800 lines of AI-authored core code.)
