---
name: pr-prepare
description: Prepare or open a GitHub or Gitea pull request with an explicit AI-authorship disclosure, change classification, verification checklist, and optional Mermaid diagram. Use whenever the user asks to prepare, write, open, create, push, ship, or get changes ready for a PR or review. For explicit open/push/ship requests, create or reuse a dedicated branch, verify the change, commit scoped files, push the branch, and only then create the PR with gh or tea. For description-only requests, remain read-only and output the PR body without changing git or remote state.
---

# Prepare and open a pull request

Produce a reviewable PR and, when explicitly requested, carry it through the safe sequence:

```text
branch -> verify -> commit -> push -> create PR -> verify PR
```

Never skip forward after a failed step. Preserve unrelated user changes and never force-push.

## Select the operating mode

- **Execute mode**: The user explicitly asks to open/create a PR, push for review, ship the change, or otherwise perform the workflow. Run the complete sequence.
- **Draft mode**: The user asks only for a PR description, template, or review of proposed PR text. Read the change and output the PR body, but do not create a branch, stage, commit, push, or create/edit a PR.
- If the request is ambiguous about remote writes, default to Draft mode and state that no repository state was changed.

## Workflow

### Step 1 - Inspect the repository

Gather the current state before making changes:

```bash
git status --short
git branch --show-current
git remote -v
git config --get "branch.$(git branch --show-current).remote" || echo "no tracking remote configured"
git rev-parse --abbrev-ref --symbolic-full-name '@{upstream}' 2>/dev/null || echo "no upstream configured"
git log --oneline -20
```

A missing tracking remote or upstream is expected before the first push; treat both as informational, not as failures.

Determine the base branch from repository configuration or the remote default branch. If it is still unclear, ask the user; common values are `main` and `develop`.

In Execute mode, resolve a writable push remote instead of assuming it is named `origin`:

- Reuse the remote from an existing correct upstream.
- Otherwise select the repository's configured writable fork or push remote and verify it with `git remote get-url --push <push-remote>`.
- If no writable remote is clear, ask the user before pushing.

Detect the hosting provider from the selected remote URL and record the matching CLI context:

- **GitHub**: Use `gh`. Verify authentication with `gh auth status`.
- **Gitea**: Use `tea`. Inspect `tea --version` and `tea pulls create --help`, verify that a matching login exists with `tea logins list`, and confirm repository access with a read-only `tea pulls list` query.
- If the hostname or provider is ambiguous, ask the user instead of choosing a CLI by guesswork.

For Gitea, resolve `<gitea-remote>` to the base/upstream repository that should receive the PR, then prefer `--remote <gitea-remote>` so `tea` discovers its login and repository. This may differ from the writable push remote in a fork workflow. If the mapping is unavailable or ambiguous, use `--login <gitea-login>` and `--repo <base-owner>/<base-repo>` explicitly. Never print or persist access tokens in the PR body or command output.

Stop before mutation when any of these conditions apply:

- The repository is in a merge, rebase, cherry-pick, or conflicted state.
- `HEAD` is detached.
- The base branch cannot be determined, or Execute mode cannot determine a writable push remote.
- The working tree mixes the requested change with unrelated edits that cannot be safely separated.

In Execute mode, stop before mutation if the selected provider CLI is unavailable or unauthenticated.

### Step 2 - Resolve issue references and branch name

Collect issue identifiers only from reliable context:

1. The user's request.
2. An explicit GitHub or Gitea issue URL, or a Jira browse URL.
3. A plan or task document that clearly identifies the work item.

Recognize:

- GitHub or Gitea issue numbers such as `#123`, `issue 123`, or `/issues/123`.
- Jira issue keys such as `GAIA-5375` or `/browse/GAIA-5375`.

Do not guess an issue from similar titles. Do not treat an ambiguous `#123` as an issue when it may be a PR number. If more than one repository-host issue or more than one Jira issue could be primary, ask the user which one to use.

Build the branch name as `<type>/<identifiers>-<slug>`:

```text
feat/add-consent-flow
feat/123-add-consent-flow
fix/GAIA-5375-handle-expired-token
feat/GAIA-5375-123-add-consent-flow
```

Use the repository's established prefix convention when one exists; otherwise use a concise type such as `feat`, `fix`, `docs`, `refactor`, `test`, `ci`, or `chore`. Preserve Jira keys in uppercase, strip `#` from GitHub or Gitea issue numbers, and keep the lowercase slug short. When both Jira and repository-host identifiers exist, put the Jira key first. Preserve identifiers before shortening an overly long slug.

Validate the result with `git check-ref-format --branch <branch>`.

### Step 3 - Create or confirm the head branch

In Draft mode, skip this step.

In Execute mode, ensure the work is on a dedicated head branch before staging or committing:

- If currently on the base branch, create the new branch with `git switch -c <branch>`; uncommitted changes move with it.
- If already on the dedicated branch for this work, reuse it so rerunning the skill is idempotent.
- If currently on another feature branch whose commits may not belong in this PR, ask whether the new branch should start from that `HEAD` or from the base branch. Do not silently include unrelated commits.
- Check local and remote names before creation. If the desired name belongs to different work, append a numeric suffix such as `-2`; never overwrite or delete it.

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

For large diffs, read the stat and the important changed sections. Identify files, packages, services, public interfaces, and call relationships.

Classify the whole PR using the stricter applicable category:

- **Leaf**: Failure remains local, such as a report, isolated endpoint, UI component, or one-off script.
- **Core**: Other components depend on it, such as auth, payments, schemas, shared frameworks, public APIs, or orchestrators.

If the blast radius is unclear, ask: "If this code is buggy, how far would the failure spread?"

### Step 5 - Record AI authorship

Ask explicitly; do not infer:

> Did AI tools write any of this change? If yes, which tool/model and files, and which files did you review line by line yourself?

Record:

- Tool and model.
- AI-authored files.
- Human line-by-line reviewed files.

Stop before commit when AI-authored core code has not received a human line-by-line review or when the author cannot identify what was AI-authored.

### Step 6 - Verify before commit

Use repository instructions and existing scripts to run the relevant formatter, lint, tests, and build checks.

- In Execute mode, run the normal required commands and inspect the diff again after any formatter or generator changes files.
- In Draft mode, preserve the read-only guarantee: use only check or dry-run variants known not to modify tracked files. Do not run formatters, generators, fixers, or other potentially mutating commands; skip them and record `Not run` when no read-only variant exists or their behavior is uncertain.

Ask only about checklist items that cannot be established from repository evidence:

- [ ] A plan or written goal and scope exists.
- [ ] The PR is focused and reviewable; coordinate or split unusually large changes.
- [ ] Relevant tests pass locally.
- [ ] No secrets, API keys, credentials, or unrelated files are included.
- [ ] Plan boundaries, including `must not modify`, were respected.
- [ ] Auth, payment, permission, or external-interface changes received the necessary security review.
- [ ] Long-running or asynchronous behavior has appropriate stress or soak coverage.

Also run `git diff --check` and, after staging, `git diff --staged --check`.

If a required check fails or cannot be verified, stop. In Execute mode, leave the local head branch intact, but do not commit, push, or create the PR.

### Step 7 - Map the change

For a non-trivial change, add a Mermaid diagram derived from the actual diff:

| Change shape                         | Diagram                        |
| ------------------------------------ | ------------------------------ |
| Calls or handshakes over time        | `sequenceDiagram`              |
| Decisions, control flow, or pipeline | `flowchart TD`                 |
| Modules, packages, or services       | `flowchart LR` with `subgraph` |
| Status or lifecycle transitions      | `stateDiagram-v2`              |

Keep the diagram honest and renderable:

- Map every changed node to a touched file, function, package, or service.
- Distinguish new or modified nodes with per-node `style` lines.
- Stay below roughly 15-20 nodes and prefer `TD` when `LR` would be too wide.
- Do not use `click` handlers or theme initialization directives.
- Omit the diagram for a one-line fix, copy change, or dependency bump.

### Step 8 - Prepare the PR body

Match an existing repository PR template when present, including `.github/` templates for GitHub and `.gitea/` templates for Gitea. Otherwise fill this template without inventing evidence:

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
  - **Human line-by-line reviewed**: <list>

## Change classification

- [ ] Leaf change
- [ ] Core change

## Plan reference

<Link or concise goal and scope>

## Verification

- **Automated**: <commands and results>
- **Manual**: <steps and results>
- **Not run**: <checks and reason>

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
```

Use `Closes #123` only when the selected GitHub or Gitea repository supports the closing keyword and the PR should close that issue in the same repository. Otherwise use `Related to #123` or a full URL. Jira keys do not use repository-host closing syntax; link them when the Jira base URL is known and use the plain key otherwise.

### Step 9 - Stage and commit

In Draft mode, skip this step.

First determine whether the intended change is already committed:

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

Stop if a non-empty staged diff contains unrelated changes, violates the announced scope, or fails verification. An empty staged diff is valid only when `<base>..HEAD` already contains the complete intended commits; otherwise stop.

Match recent repository commit style. Prefer a focused conventional commit when no stronger convention exists:

```text
<type>(<scope>): <imperative summary>

<important change bullets, if useful>
```

When staging was needed, create the commit directly because Execute mode already expresses the user's intent to commit and open the PR. Whether the commit was created now or already existed, verify the intended commit range and `git show --stat --oneline HEAD`, then report any intentionally unstaged user changes.

### Step 10 - Push the branch

Push only after the intended commit was created successfully or verified as already present.

- If the branch already has the correct upstream, preserve it and use `git push`.
- If the branch has no upstream, use the writable push remote resolved in Step 1.
- If the existing upstream is wrong or ambiguous, stop and ask before replacing it.
- On reruns, if the correct upstream already contains `HEAD`, skip the redundant push and continue to PR creation.

```bash
git push
git push -u <push-remote> <head-branch>
```

Use only the command that matches the branch state; do not run both.

Never force-push. If authentication, policy, hooks, or network errors prevent the push, stop and report the exact failure. Keep the local branch and commit for retry; do not create the PR.

### Step 11 - Create and verify the PR

Check whether the exact head and base branch pair already has a PR before creating one. If it does, return the existing PR instead of creating a duplicate; edit it only when the user asks.

For GitHub, query existing PRs with `gh`:

```bash
gh pr list \
  --head <head-branch> \
  --base <base-branch> \
  --state all \
  --json number,state,url,title,body,baseRefName,headRefName,headRepositoryOwner
```

Pass `--head` a bare branch name. `gh pr list --head` does not support `<owner>:<branch>`, and it returns an empty list rather than an error when given one — which reads as "no existing PR" and causes a duplicate. When the head branch lives in a fork, or the same branch name could exist across forks, filter the returned `headRepositoryOwner.login` for the expected head owner instead of encoding the owner in `--head`.

For Gitea, query with `tea` and filter the JSON result for the exact head and base. Use the selected `--remote` context, or replace it with the resolved `--login` and optional `--repo` context:

```bash
tea pulls list \
  --remote <gitea-remote> \
  --state all \
  --fields index,state,url,title,body,base,head \
  --output json
```

Save the prepared body to a temporary file, then create the PR only after the push succeeds:

For GitHub:

```bash
gh pr create \
  --base <base-branch> \
  --head <head-branch> \
  --title "<title>" \
  --body-file <body-file>
```

When the GitHub head branch is in a fork, use `--head <head-owner>:<head-branch>`; `gh pr create --head` accepts that form to select a head repo owned by another user. It does not accept an organization as the owner, so for an org-owned fork run `gh` against that repository context instead. Unlike `gh pr list`, an explicit owner here is required for fork-based creation, not optional.

For Gitea, pass the body through `--description` because `tea pulls create` has no body-file option:

```bash
tea pulls create \
  --remote <gitea-remote> \
  --base <base-branch> \
  --head <head-branch> \
  --title "<title>" \
  --description "$(cat <body-file>)"
```

When the Gitea head branch is in a fork, keep `--remote` pointed at the base/upstream repository and use `--head <head-owner>:<head-branch>`. When remote discovery is unsuitable, replace `--remote <gitea-remote>` with `--login <gitea-login> --repo <base-owner>/<base-repo>`.

Do not force issue identifiers into the title when repository conventions place them only in branch names or the body.

Verify the result with the same provider and context used to create it.

For GitHub, pass the PR number returned by creation or found by the exact head/base query:

```bash
gh pr view <pr-number> \
  --json number,title,url,state,baseRefName,headRefName,body
```

For Gitea, pass the PR index returned by creation or found by the exact head/base query:

```bash
tea pulls <pr-index> \
  --remote <gitea-remote> \
  --fields index,state,url,title,body,base,head \
  --output json
```

Confirm that the PR is open against the intended base, uses the pushed head branch, and contains the expected issue links and body. A PR creation failure leaves the remote branch intact for retry.

## Failure and rerun behavior

- A verification failure stops before commit.
- A commit failure stops before push.
- A push failure stops before PR creation.
- A PR creation failure does not delete or rewrite the pushed branch.
- A rerun reuses the correct branch, commit, upstream, or existing PR when already present instead of duplicating work.
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
- Remaining unstaged changes, failed checks, TODOs, and reviewer recommendation: one reviewer for leaf changes, two or more including the module owner for core changes.

## Quality rules

- Be specific about the bug, feature, mechanism, risk, and verification.
- Match repository branch, commit, and PR conventions before applying defaults.
- Lead reviewers toward high-risk code and disclose AI authorship honestly.
- Reject architecture fiction: diagrams and claims must trace back to the actual diff.
- Flag unreviewed AI-authored core code, missing tests, leaked secrets, scope creep, and sprawling cross-module changes before any push.
