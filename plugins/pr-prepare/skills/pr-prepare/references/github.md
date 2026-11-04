# GitHub (`gh`) commands

Provider-specific details for the `pr-prepare` skill when the base repository is on GitHub. SKILL.md defines when each step runs and when to stop; this file defines the exact commands.

## Contents

- [Step 1 - Verify the CLI](#step-1---verify-the-cli)
- [Pin the base repository](#pin-the-base-repository)
- [Step 11 - Look up an existing PR](#step-11---look-up-an-existing-pr)
- [Step 11 - Create the PR](#step-11---create-the-pr)
- [Step 11 - Verify the PR](#step-11---verify-the-pr)
- [Step 11b - Evidence placeholders](#step-11b---evidence-placeholders)

## Step 1 - Verify the CLI

```bash
gh auth status
```

In Execute mode, stop before mutation if `gh` is missing or unauthenticated.

## Pin the base repository

Pass `--repo <base-owner>/<base-repo>` to every `gh` call below. Without it `gh` infers the repository from the local remotes, which in a fork clone resolves to the fork: the lookup misses an existing upstream PR, creation targets the wrong repository or prompts interactively, and a PR number can resolve to a different repository's PR and report a false pass.

## Step 11 - Look up an existing PR

```bash
gh pr list \
  --repo <base-owner>/<base-repo> \
  --head <head-branch> \
  --base <base-branch> \
  --state all \
  --json number,state,url,title,baseRefName,headRefName,headRepositoryOwner
```

- Request only these fields; omit `body`, which is irrelevant to matching and floods the transcript.
- Pass `--head` a bare branch name. `gh pr list --head` does not support `<owner>:<branch>` and silently returns an empty list, which reads as "no existing PR" and causes a duplicate. For a fork head, or when the branch name could exist across forks, filter `headRepositoryOwner.login` for the expected owner instead.

## Step 11 - Create the PR

```bash
gh pr create \
  --repo <base-owner>/<base-repo> \
  --base <base-branch> \
  --head <head-branch> \
  --title "<title>" \
  --body-file <body-file>
```

When the head branch is in a fork, use `--head <head-owner>:<head-branch>`; here the owner is required for fork-based creation. `--head` does not accept an organization as the owner, so for an org-owned fork run `gh` against that repository context instead. `--repo` fixes where the PR is opened and `--head` fixes where the commits come from; a fork workflow needs both.

## Step 11 - Verify the PR

```bash
gh pr view <pr-number> \
  --repo <base-owner>/<base-repo> \
  --json number,title,url,state,baseRefName,headRefName,body
```

## Step 11b - Evidence placeholders

`gh` has no image-upload API, and committing images to the repository is not allowed, so never claim an automatic upload. Keep the `<!-- evidence:<id> -->` placeholders, append this note to the Evidence section of the body file, and update the PR:

```markdown
> Screenshots captured locally — to be attached by the author:
> - `<id>.png`
```

```bash
gh pr edit <pr-number> \
  --repo <base-owner>/<base-repo> \
  --body-file <body-file>
```

List the absolute local paths in the final handoff and ask the user to drag the images into the PR description.
