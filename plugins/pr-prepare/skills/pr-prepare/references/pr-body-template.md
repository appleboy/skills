# PR body template

Default PR body for Step 8 of the `pr-prepare` skill. Use it when the repository has no PR template; when one exists, put the same content in the template's equivalent sections and append any section it lacks. Follow the writing rules in SKILL.md Step 8 while filling it, and never invent evidence.

```markdown
## Summary

<What changed, why, and the mechanism used>

## Related issues

- Jira: <[GAIA-5375](Jira URL), plain key, or N/A>
- GitHub/Gitea: <Closes #123, Related to #123, full cross-repository URL, or N/A>

## Architecture / flow

<Mermaid diagram; omit this section for trivial changes>

## AI authorship

- [ ] No AI was used
- [ ] AI was used
  - **Tool / model**: <value>
  - **AI-authored files**: <list>
  - **Human line-by-line reviewed**: None — not yet reviewed by a human.

## Change classification

- [ ] Leaf change
- [ ] Core change

## Plan reference

<Link or concise goal and scope>

## Verification

### Setup

- **Checkout target / working directory**: <head branch, or exact PR head SHA when already committed, and directory; disclose local-only changes in Draft-only mode>
- **Prerequisites**: <tools, services, configuration names, test data, account roles>
- **Prepare and start**: <exact commands/actions and readiness signal, or setup link plus PR-specific steps>

### Automated checks

| Command (with working directory) | Behavior covered / expected success | Status | Observed result / reason not run |
| --- | --- | --- | --- |
| <copyable command> | <coverage and success signal> | <Passed / Failed / Not run> | <actual evidence or blocker> |

### Behavioral scenarios

Repeat this block for each relevant scenario:

#### <Scenario name: main flow, regression, or relevant edge case>

- **Acceptance condition**: <intended behavior; related issue requirement or flow node where available>
- **Starting state**: <fixture, role, configuration, or other preconditions>

| Step | Action / command / input | Expected observable result |
| --- | --- | --- |
| 1 | <concrete action> | <observable output or state> |
| 2 | <next action> | <observable output or state> |

- **Execution status**: <Passed / Failed / Not run>
- **Observed result**: <actual evidence, or reason not run and required environment/access>
- **Cleanup**: <reset commands/actions, or N/A with reason>

## Evidence

### Web screenshots

<N/A — reason, or:>

| Page / state | Viewport | Before (base) | After (head) | Status |
| --- | --- | --- | --- | --- |
| <route – state> | <1280×800> | <!-- evidence:before-<page>-<state> --> | <!-- evidence:after-<page>-<state> --> | <Passed / Not run> |

### Performance report

<N/A — reason, or:>

- **Environment**: <CPU / OS / runtime / dataset size>
- **Method**: <command, iterations, warm-up>

| Metric | Base | Head | Δ | Status |
| --- | --- | --- | --- | --- |
| <metric> | <measured> | <measured> | <±%> | <Passed / Failed / Not run> |

- **Raw output**: <collapsed `<details>` block of benchstat/hyperfine/Lighthouse output>
- **Interpretation**: <improvement / no regression / inconclusive, and why>

## Security check

- [ ] No secrets in the diff
- [ ] External inputs are validated
- [ ] Permission checks are tested
- [ ] Errors do not leak internals
- [ ] N/A - no external or security-sensitive interface changed

## Risk and rollback

- **Risk**: <what could break>
- **Rollback**: <how to revert safely>

## Reviewer guide

- **Read carefully**: <high-risk files or functions>
- **Spot-check**: <lower-risk generated or mechanical changes>
- **Evidence not captured**: <evidence items marked Not run and what the reviewer should check, or N/A>
```

## Issue link syntax

Use `Closes #123` only when the selected GitHub or Gitea repository supports the closing keyword and the PR should close that issue in the same repository. Otherwise use `Related to #123` or a full URL. Jira keys do not use repository-host closing syntax; link them when the Jira base URL is known and use the plain key otherwise.
