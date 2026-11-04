---
name: pr-prepare
description: Prepare or open a GitHub or Gitea pull request with an explicit AI-authorship disclosure, change classification, reproducible reviewer verification steps, UI screenshots and performance evidence when relevant, and optional Mermaid diagram. Use whenever the user asks to prepare, write, open, create, push, ship, or get changes ready for a PR or review. Unless the user explicitly asks for a draft-only result, create or reuse a dedicated branch, verify the change, commit scoped files, push the branch, and then create the PR with gh or gt.
---

# Prepare and open a pull request

Produce a reviewable PR and, unless the user explicitly requests a draft-only result, carry it through the safe sequence:

```text
branch -> verify -> commit -> push -> create PR -> verify PR
```

Never skip forward after a failed step. Preserve unrelated user changes and never force-push.

## Reference files

Read each file at the step that needs it; they hold exact commands and templates, while this file holds the rules and stop conditions.

| File                                                             | Read at                                               |
| ---------------------------------------------------------------- | ----------------------------------------------------- |
| [references/github.md](references/github.md)                     | Step 1, Step 11, Step 11b when the provider is GitHub |
| [references/gitea.md](references/gitea.md)                       | Step 1, Step 11, Step 11b when the provider is Gitea  |
| [references/pr-body-template.md](references/pr-body-template.md) | Step 8                                                |
| [references/evidence.md](references/evidence.md)                 | Step 6b, only when evidence is needed                 |

## Select the operating mode

- **Execute mode (default)**: Run the complete sequence whenever the user invokes PR preparation, including creating or opening a PR. Do not ask for a final confirmation before creating the branch, committing, pushing, or opening the PR.
- **Draft-only mode**: Use only when the user explicitly asks for a draft-only result: a PR description, template, or read-only review. Read the change and output the PR body, but do not create a branch, stage, commit, push, or create/edit a PR.

## Workflow

### Step 1 - Inspect the repository

Gather the current state before making changes:

```bash
git status --short
git branch --show-current
git remote -v | sed -E 's#://[^/@]*@#://***@#g'
git config --get "branch.$(git branch --show-current).remote" || echo "no tracking remote configured"
git rev-parse --abbrev-ref --symbolic-full-name '@{upstream}' 2>/dev/null || echo "no upstream configured"
git log --oneline -20
```

A missing tracking remote or upstream is expected before the first push; treat both as informational.

Remote URLs can carry credentials in their userinfo, so pipe every command that echoes a remote URL through the same `sed` mask. It deliberately masks any scheme-qualified userinfo, even a harmless `git@`, because a narrower pattern risks missing a real token; scheme, host, and path survive for provider detection.

Determine the base branch from repository configuration or the remote default branch. If it is still unclear, ask the user; common values are `main` and `develop`.

In Execute mode, resolve a writable push remote instead of assuming it is named `origin`:

- Reuse the remote from an existing correct upstream.
- Otherwise select the configured writable fork or push remote and verify it with `git remote get-url --push <push-remote> | sed -E 's#://[^/@]*@#://***@#g'`.
- If no writable remote is clear, ask the user before pushing.

Detect the hosting provider from the selected remote URL — **GitHub** uses `gh`, **Gitea** uses `gt` — then verify the CLI and resolve its base-repository context as described in the matching provider file. If the hostname or provider is ambiguous, ask the user instead of guessing. Never print or persist access tokens in the PR body or command output.

Stop before mutation when any of these conditions apply:

- The repository is in a merge, rebase, cherry-pick, or conflicted state.
- `HEAD` is detached.
- The base branch cannot be determined, or Execute mode cannot determine a writable push remote.
- The working tree mixes the requested change with unrelated edits that cannot be safely separated.
- In Execute mode, the provider CLI is unavailable, unauthenticated, or cannot reach the base repository.

### Step 2 - Resolve issue references and branch name

Collect issue identifiers only from reliable context: the user's request, an explicit GitHub/Gitea issue URL or Jira browse URL, or a plan or task document that clearly identifies the work item. Recognize `#123`, `issue 123`, `/issues/123`, and Jira keys such as `GAIA-5375` or `/browse/GAIA-5375`.

Do not guess an issue from similar titles, and do not treat an ambiguous `#123` as an issue when it may be a PR number. If more than one repository-host issue or more than one Jira issue could be primary, ask the user.

Build the branch name as `<type>/<identifiers>-<slug>`:

```text
feat/add-consent-flow
feat/123-add-consent-flow
fix/GAIA-5375-handle-expired-token
feat/GAIA-5375-123-add-consent-flow
```

Use the repository's prefix convention when one exists; otherwise use a concise type such as `feat`, `fix`, `docs`, `refactor`, `test`, `ci`, or `chore`. Keep Jira keys uppercase and first, strip `#` from issue numbers, keep the lowercase slug short, and shorten the slug before dropping identifiers. Validate with `git check-ref-format --branch <branch>`.

### Step 3 - Create or confirm the head branch

In Draft-only mode, skip this step. In Execute mode, ensure the work is on a dedicated head branch before staging or committing:

- On the base branch, create the branch with `git switch -c <branch>`; uncommitted changes move with it.
- Already on the dedicated branch for this work, reuse it so reruns are idempotent.
- On another feature branch whose commits may not belong in this PR, ask whether to start from that `HEAD` or from the base branch. Do not silently include unrelated commits.
- If the desired name already belongs to different work locally or remotely, append a numeric suffix such as `-2`; never overwrite or delete it.

Record the base branch, head branch, push remote, and existing upstream for the push and PR commands.

### Step 4 - Read and classify the change

Inspect committed and uncommitted changes that would enter the PR:

```bash
git log <base>..HEAD --oneline
git diff <base>...HEAD --stat
git diff <base>...HEAD
git diff --stat
git diff
git diff --cached --stat
git diff --cached
```

For large diffs, read the stat and the important sections. Identify files, packages, services, public interfaces, and call relationships.

Classify the whole PR using the stricter applicable category:

- **Leaf**: Failure remains local, such as a report, isolated endpoint, UI component, or one-off script.
- **Core**: Other components depend on it, such as auth, payments, schemas, shared frameworks, public APIs, or orchestrators.

If the blast radius is unclear, ask: "If this code is buggy, how far would the failure spread?"

### Step 4b - Decide review evidence

Reviewers cannot see a UI change or judge a performance claim from text alone, so decide from the diff and the stated goal whether this PR needs visual or quantitative evidence. Apply every row that matches; a PR can need both:

| Signal in diff / goal                                                                                                                                                                       | Evidence                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| Changed `*.tsx`, `*.jsx`, `*.vue`, `*.svelte`, `*.css`, `*.scss`, `*.html`, templates, UI components, user-visible copy, or layout                                                          | Web screenshots (before/after) |
| Goal mentions performance, optimization, latency, throughput, or memory; or the diff touches hot paths, DB queries or indexes, caching, concurrency, bundle size, or algorithmic complexity | Performance report             |
| Pure backend logic, docs, CI, or a dependency bump without a performance claim                                                                                                              | N/A with a one-line reason     |

Ask the user only when the signals are ambiguous, such as a backend change that may alter what a page renders. Record an evidence plan for Step 6b: pages, states, viewports, data, and account role for screenshots; benchmark or workload, metrics, and dataset size for performance; or the one-line N/A reason.

### Step 5 - Record AI authorship

Use this default disclosure without asking for confirmation:

- **Tool / model**: the AI agent executing this workflow; use its reported model or tool name when available.
- **AI-authored files**: every task-related file included in the PR.
- **Human line-by-line reviewed**: `None — not yet reviewed by a human.`

Lack of human review is not a reason to stop; make the disclosure conspicuous in the PR body and final handoff. If the user supplies more specific authorship or review information, use that instead.

### Step 6 - Verify before commit

Use repository instructions and existing scripts to run the relevant formatter, lint, tests, and build checks.

- In Execute mode, run the normal required commands and inspect the diff again after any formatter or generator changes files.
- In Draft-only mode, preserve the read-only guarantee: use only check or dry-run variants known not to modify tracked files. Skip formatters, generators, fixers, and other potentially mutating commands, and record `Not run` when no read-only variant exists or its behavior is uncertain.

Ask only about checklist items that cannot be established from repository evidence:

- [ ] A plan or written goal and scope exists.
- [ ] The PR is focused and reviewable; coordinate or split unusually large changes.
- [ ] Relevant tests pass locally.
- [ ] No secrets, API keys, credentials, or unrelated files are included.
- [ ] Plan boundaries, including `must not modify`, were respected.
- [ ] Auth, payment, permission, or external-interface changes received the necessary security review.
- [ ] Long-running or asynchronous behavior has appropriate stress or soak coverage.

Also run `git diff --check` and, after staging, `git diff --staged --check`.

In Execute mode, if a required check fails or cannot be verified, stop. Leave the local head branch intact, but do not commit, push, or create the PR. In Draft-only mode, continue preparing the body and report failed checks as `Failed` and skipped or unavailable checks as `Not run`, with reasons; do not claim the draft is verified or ready to merge.

Record the working directory, prerequisites, exact commands or actions, and observed results while verifying so Step 8 can turn them into reproducible reviewer instructions. Keep expected behavior separate from observed results; a proposed check is not evidence that it passed.

### Step 6b - Capture review evidence

Skip this step when Step 4b marked both evidence types N/A; otherwise read [references/evidence.md](references/evidence.md) for capture recipes.

Choose the source in this order, because the user knows their environment best and repository tooling reflects how the team already measures:

1. Files or numbers the user supplied.
2. Repository scripts: `package.json` scripts, `Makefile` bench targets, existing e2e or Playwright configuration, documented benchmark commands.
3. Generic tools already on `PATH` or in the repository's dependencies, such as Playwright or another headless browser, `go test -bench` with `benchstat`, `hyperfine`, Lighthouse CLI, or k6. A browser skill such as `/browse` may be used when installed, but do not assume one exists. Do not install global tools or fetch packages that are not already present without asking.

Capture base and head under the same conditions so the comparison means something:

- **Base vs head**: Check out the base branch in a separate `git worktree` outside the repository so the user's working tree is never touched. If base cannot be built or run, capture head only and say so rather than presenting a one-sided result as a comparison.
- **Screenshots**: Fixed viewport — desktop 1280×800 by default, plus mobile 390×844 when the change is responsive — with the same data, account role, and state in both captures. Use only local or non-production environments and mask or avoid PII, tokens, and internal hostnames, because uploaded attachments are visible to everyone with repository access.
- **Performance**: Record the environment (CPU, OS, runtime, dataset size), warm-up and run count, the statistic (median, p95), and the variance. When the variance overlaps the difference, report the result as inconclusive.
- **Storage**: Write artifacts to `${TMPDIR:-/tmp}/pr-evidence/<head-branch>/`, never inside the repository, and never `git add` them. Name screenshots `<before|after>-<page>-<state>[-mobile].png`; the name without the extension is the evidence ID used in Step 8 and Step 11b.

Draft-only mode may capture, since that does not modify tracked files, but never uploads.

When evidence cannot be captured, record `Not run` with the reason and the environment or access needed. An evidence gap does not block commit, push, or PR creation, and it never overrides or relaxes Step 6's Execute-mode required-check gate; like reviewer-only checks, it is reported, not enforced.

### Step 7 - Map the change

For a non-trivial change, add a Mermaid diagram derived from the actual diff:

| Change shape                         | Diagram                        |
| ------------------------------------ | ------------------------------ |
| Calls or handshakes over time        | `sequenceDiagram`              |
| Decisions, control flow, or pipeline | `flowchart TD`                 |
| Modules, packages, or services       | `flowchart LR` with `subgraph` |
| Status or lifecycle transitions      | `stateDiagram-v2`              |

Map every changed node to a touched file, function, package, or service; mark new or modified nodes with per-node `style` lines; stay below roughly 15-20 nodes and prefer `TD` when `LR` would be too wide; do not use `click` handlers or theme initialization directives. Omit the diagram for a one-line fix, copy change, or dependency bump.

### Step 8 - Prepare the PR body

Match an existing repository PR template when present, including `.github/` templates for GitHub and `.gitea/` templates for Gitea; put the verification and evidence content in its equivalent sections, or add sections it lacks. Otherwise fill [references/pr-body-template.md](references/pr-body-template.md). Never invent evidence.

#### Write reproducible reviewer verification

The PR body must let a reviewer verify the intended behavior without the author's local state or this conversation. Scale detail to the change, but never reduce verification to "tests pass" or "test manually."

- **Prerequisites and setup**: Identify the head branch, or the exact PR head SHA when the changes are already committed, along with the working directory, runtime/tool versions, dependencies, services, configuration, test data, and account roles. Never substitute the base SHA or invent a revision. In Draft-only mode with uncommitted changes, state that the changes are local. Give actual setup/start commands and readiness signals; use safe sample data and environment-variable names, never credentials. Link maintained setup docs for lengthy standard setup but keep PR-specific steps in the body.
- **Behavioral scenarios**: Derive acceptance conditions from the goal or issue and confirm them against the diff. Include the main changed flow and relevant failure, boundary, permission, or regression cases. For a bug fix, describe the original trigger and corrected outcome; for a refactor, the behavior that stays unchanged; for documentation or skill changes, a concrete inspection or representative usage scenario.
- **Steps and expected results**: For each scenario, give the starting state and numbered actions with concrete inputs, commands, requests, or UI navigation, each paired with an observable expected result. For asynchronous flows, identify the completion signal and a supported timeout. Map scenarios to the diagram when present.
- **Automated checks**: Include copyable repository-supported commands, where to run them, what behavior they cover, and how to recognize success. Do not invent scripts, fixtures, endpoints, or outputs; mark missing information explicitly.
- **Execution evidence**: Label each check and scenario `Passed`, `Failed`, or `Not run`, and record actual results separately from expected ones. For checks not run, give the reason and the access or environment needed. Reviewer-only checks do not count as completed verification or override Step 6's Execute-mode gate.
- **Cleanup**: Include reset steps when verification creates data or changes state, describe side effects, and use a disposable or non-production environment for mutating checks.

#### Write the Evidence section

Fill `## Evidence` from Step 4b and Step 6b with the same `Passed / Failed / Not run` vocabulary. For an evidence type marked N/A, write a single `N/A — <reason>` line instead of a table. Put each image cell as an `<!-- evidence:<id> -->` placeholder so Step 11b can replace it reliably. Never describe a screenshot that was not captured or fill a metric that was not measured. Flag every evidence item marked `Not run` in the Reviewer guide so reviewers know what to check themselves.

Before finalizing, walk through the body as a fresh reviewer: can they prepare the environment, follow each step, and decide whether the flow meets the acceptance conditions? Fill every placeholder from repository evidence, and omit inapplicable fields with a brief reason.

### Step 9 - Stage and commit

In Draft-only mode, skip this step. First determine whether the intended change is already committed:

```bash
git log <base>..HEAD --oneline
git diff <base>...HEAD --stat
git status --short
```

- If `<base>..HEAD` already contains the complete intended change and no task-related uncommitted changes remain, inspect those commits, skip staging and commit creation, and continue to push.
- If task-related changes remain uncommitted, stage only those files. Never use `git add .` or `git add -A` in a dirty worktree.
- If there are neither intended commits nor task-related changes, stop because there is nothing to open as a PR.

When staging is needed, inspect exactly what will be committed:

```bash
git add <task-file-1> <task-file-2>
git diff --staged --stat
git diff --staged
git diff --staged --check
```

Stop if a non-empty staged diff contains unrelated changes, violates the announced scope, or fails verification. Evidence artifacts from Step 6b must never be staged; if any appear, stop and unstage them. An empty staged diff is valid only when `<base>..HEAD` already contains the complete intended commits; otherwise stop.

Match recent repository commit style; otherwise prefer a focused conventional commit:

```text
<type>(<scope>): <imperative summary>

<important change bullets, if useful>
```

Create the commit directly, since Execute mode already expresses the intent to commit. Then verify the intended commit range and `git show --stat --oneline HEAD`, and report any intentionally unstaged user changes.

### Step 10 - Push the branch

Push only after the intended commit was created or verified as already present.

- If the branch already has the correct upstream, preserve it and run `git push`.
- If it has no upstream, run `git push -u <push-remote> <head-branch>` with the remote resolved in Step 1.
- If the existing upstream is wrong or ambiguous, stop and ask before replacing it.
- On reruns, if the correct upstream already contains `HEAD`, skip the push.

Never force-push. If authentication, policy, hooks, or network errors prevent the push, stop and report the exact failure; keep the local branch and commit and do not create the PR.

### Step 11 - Create and verify the PR

Use the commands in the provider file, always pinned to the base repository context resolved in Step 1, so a fork clone cannot redirect the lookup, creation, or verification to the wrong repository.

1. **Look up** an existing PR for the exact head and base pair. If one exists, return it instead of creating a duplicate; edit it only when the user asks (Step 11b's placeholder replacement is the one exception).
2. **Create** the PR from a body file only after the push succeeded. Do not force issue identifiers into the title when repository conventions place them only in branch names or the body.
3. **Verify** with the same provider and context: the PR is open against the intended base, uses the pushed head branch, and contains the expected issue links and body.

A PR creation failure leaves the remote branch intact for retry.

### Step 11b - Attach evidence

Skip this step in Draft-only mode or when Step 6b captured no screenshots. Run it only after Step 11 verified the PR, following the provider file: Gitea uploads attachments and replaces placeholders with image links; GitHub has no upload API, so keep the placeholders, append a pending-attachment note, and never claim an automatic upload or commit images. The only body change allowed here is replacing `<!-- evidence:<id> -->` placeholders and adding that note.

## Failure and rerun behavior

- In Execute mode, a verification failure stops before commit; Draft-only mode records it and continues preparing the body.
- A commit failure stops before push.
- A push failure stops before PR creation.
- A PR creation failure does not delete or rewrite the pushed branch.
- An evidence capture failure is recorded as `Not run` and does not block commit, push, or PR creation.
- An attachment upload or body edit failure leaves the PR and its placeholders intact and is reported; it never deletes or recreates the PR.
- A rerun reuses the correct branch, commit, upstream, existing PR, or existing attachment instead of duplicating work.
- Never use destructive cleanup such as `git reset --hard`, branch deletion, or force-push as automatic recovery.

## Final handoff

Report:

- Operating mode used.
- Hosting provider and CLI context used.
- Base and head branches.
- GitHub/Gitea and Jira issue identifiers included or omitted.
- Commit SHA and title, if created.
- Push remote and upstream status, if pushed.
- PR number and URL, if created or already present.
- Verification commands and results.
- Evidence: screenshots captured / uploaded / pending manual attach (with absolute local paths for GitHub), performance report status, and every evidence item marked `Not run` with its reason.
- Remaining unstaged changes, failed checks, TODOs, and reviewer recommendation: one reviewer for leaf changes, two or more including the module owner for core changes.

## Quality rules

- Be specific about the bug, feature, mechanism, risk, and verification.
- Make reviewer verification reproducible: setup, concrete steps, observable expected results, and honest execution status, even when using a repository PR template.
- Match repository branch, commit, and PR conventions before applying defaults.
- Lead reviewers toward high-risk code and disclose AI authorship honestly.
- Reject architecture fiction: diagrams and claims must trace back to the actual diff.
- Never fabricate or estimate performance numbers or describe screenshots that were not captured; evidence must come from an actual run or a user-supplied file.
- Flag unreviewed AI-authored core code, missing tests, leaked secrets, scope creep, and sprawling cross-module changes before any push.
