# Gitea (`gt`) commands

Provider-specific details for the `pr-prepare` skill when the base repository is on Gitea. SKILL.md defines when each step runs and when to stop; this file defines the exact commands. `gt` is Gitea CLI 1.0+, the successor to the archived `tea`.

## Contents

- [Step 1 - Verify the CLI and resolve the context](#step-1---verify-the-cli-and-resolve-the-context)
- [Step 11 - Look up an existing PR](#step-11---look-up-an-existing-pr)
- [Step 11 - Create the PR](#step-11---create-the-pr)
- [Step 11 - Verify the PR](#step-11---verify-the-pr)
- [Step 11b - Upload evidence attachments](#step-11b---upload-evidence-attachments)

## Step 1 - Verify the CLI and resolve the context

Resolve the Gitea context to the base/upstream repository that should receive the PR. In a fork workflow it may differ from the writable push remote.

- Prefer `--remote <gitea-remote>` so `gt` discovers instance, account, and repository from that remote.
- If the mapping is unavailable or ambiguous, use `--host <gitea-url> --repo <base-owner>/<base-repo>`, adding `--user <gitea-user>` to pick a non-active account.

Every `gt` command in this file shows `--remote <gitea-remote>`; substitute the explicit `--host`/`--repo`/`--user` context when that is what was resolved. Without either, `gt` falls back to `GITEA_CLI_REPO`, `gt repo set-default`, or the local remotes, so a probe can pass against a fork or another server while PR creation fails against the intended base.

Probe in that same context:

```bash
gt --version
gt pr create --help
gt auth status
gt pr list --remote <gitea-remote> --limit 1
```

`gt doctor` checks instance, account, credential, and repository in one pass and may replace the last two commands. In Execute mode, stop before mutation if `gt` is missing, unauthenticated, or cannot read the base repository. Never print or persist access tokens.

## Step 11 - Look up an existing PR

```bash
gt pr list \
  --remote <gitea-remote> \
  --head <head-branch> \
  --base <base-branch> \
  --state all \
  --json number,state,url,title,base,head,head_repo
```

Request only these fields; omit `body`. Confirm the exact head and base in the JSON result. For a fork head, pass `--head <head-owner>:<head-branch>` and still check `head_repo` for the expected owner.

## Step 11 - Create the PR

```bash
gt pr create \
  --remote <gitea-remote> \
  --base <base-branch> \
  --head <head-branch> \
  --title "<title>" \
  --body-file <body-file> \
  --no-input \
  --json number,url,state
```

- Explicit flags skip the interactive questions; `--no-input` guarantees no prompt or editor stalls an unattended run.
- Do not add `--yes`: it lets `gt` push on its own, bypassing the push remote resolved in Step 1. The branch is already pushed.
- If a PR already exists for the same branches, `gt pr create` links it instead of creating a duplicate, but run the lookup first so the workflow can report it explicitly.
- For a fork head, keep the context pointed at the base repository and use `--head <head-owner>:<head-branch>`.

## Step 11 - Verify the PR

```bash
gt pr view <pr-number> \
  --remote <gitea-remote> \
  --json number,state,url,title,body,base,head,head_repo
```

## Step 11b - Upload evidence attachments

Upload in the same context used to create the PR. On reruns, list existing attachments first and reuse the URL of any attachment with the same name instead of uploading it twice:

```bash
gt pr attachment list <pr-number> \
  --remote <gitea-remote> \
  --json name,url

gt pr attachment upload <pr-number> <file> \
  --name <id>.png \
  --remote <gitea-remote> \
  --json url
```

Replace each matching `<!-- evidence:<id> -->` in the body file with `![<id>](<url>)`, update the PR, and verify that no placeholders remain:

```bash
gt pr edit <pr-number> \
  --remote <gitea-remote> \
  --body-file <body-file>

gt pr view <pr-number> \
  --remote <gitea-remote> \
  --json body
```

Attachments are visible to everyone with access to the repository.
