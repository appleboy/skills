---
name: copilot-review
description: "Runs a single Copilot review check-fix cycle on a PR using gh CLI. Use this skill whenever the user wants to: check and fix Copilot review comments, request a Copilot code review on a PR, or iterate on PR feedback. Trigger on phrases like 'copilot review', 'fix PR comments', 'iterate on PR', 're-review', or when the user mentions 'gh pr' with 'copilot'. Combine with /loop for automated repeated cycles, e.g. '/loop 2m /copilot-review'."
---

# Copilot Review

A single-pass check-fix cycle for Copilot review comments. Designed to be called repeatedly via `/loop`:

```
/loop 2m /copilot-review
```

Each invocation runs one iteration: check for unresolved comments → fix → push → resolve → re-trigger review. When no comments remain, stop the loop.

## Prerequisites

- `gh` CLI v2.88.0+
- Authenticated: `gh auth status`

---

## Steps

### 1. Detect the PR

If the user provides a PR number and repo, use those directly. Otherwise, auto-detect from the current branch:

```bash
gh pr view --json number,url,headRepository -q '
  "PR #\(.number) — \(.headRepository.owner.login)/\(.headRepository.name)\n\(.url)"
'
```

If no PR is found on the current branch, ask the user for the PR number and repo.

Extract and store:

- `OWNER` — repository owner
- `REPO` — repository name
- `PR_NUM` — pull request number

### 2. Check Copilot review status

Confirm that Copilot has finished its latest review before acting on comments.

```bash
gh pr view ${PR_NUM} --repo ${OWNER}/${REPO} --json reviews -q '
  [.reviews[]
    | select(.author.login | test("copilot"; "i"))]
  | sort_by(.submittedAt)
  | .[-1].state // "PENDING"
'
```

- `"PENDING"` — Copilot is still reviewing. **Report "Copilot review in progress, waiting..." and stop.** The next `/loop` cycle will check again.
- `"APPROVED"` — No suggestions. Report "Copilot review passed, ready for human review" and **stop the loop**.
- `"COMMENTED"` or `"CHANGES_REQUESTED"` — Proceed to Step 3.

### 3. Fetch unresolved Copilot review comments

Use the GraphQL API to get only unresolved threads authored by Copilot. The REST API does not include inline review comments.

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            comments(first: 10) {
              nodes {
                body
                author { login }
                path
                line
                originalLine
              }
            }
          }
        }
      }
    }
  }
' -F owner="${OWNER}" -F repo="${REPO}" -F pr="${PR_NUM}" \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[]
    | select(.isResolved == false)
    | select(.comments.nodes[0].author.login | test("copilot"; "i"))
    | {
        threadId: .id,
        path: .comments.nodes[0].path,
        line: (.comments.nodes[0].line // .comments.nodes[0].originalLine),
        comment: .comments.nodes[0].body
      }'
```

If there are **no unresolved comments**, report "All Copilot comments resolved — ready for human review" and **stop the loop**.

### 4. Fix the code

For each unresolved comment:

1. Read the referenced file and line.
2. Understand the suggestion — evaluate whether it makes sense in the project context.
3. Apply the fix.

**Do not blindly accept every suggestion.** Copilot may repeat already-resolved comments or suggest changes that conflict with project architecture. If a suggestion is incorrect or irrelevant, skip it and note the reason.

### 5. Run tests

Run the project's test suite before committing. A fix that breaks other things is worse than the original issue.

If tests fail:
- Diagnose and fix the test failure.
- If the fix conflicts with the Copilot suggestion, revert that suggestion and note it as skipped.
- Re-run tests until they pass.

### 6. Commit and push

Use the **commit-message** skill (`/commit-message`) to generate a proper conventional commit message for the fixes.

After committing, push the changes:

```bash
git push
```

### 7. Resolve fixed threads

After pushing, resolve each thread that was successfully addressed by inlining its ID:

```bash
gh api graphql -f query='
  mutation {
    resolveReviewThread(input: {threadId: "<THREAD_NODE_ID>"}) {
      thread { isResolved }
    }
  }
'
```

### 8. Re-trigger Copilot review

```bash
gh pr edit ${PR_NUM} --repo ${OWNER}/${REPO} --add-reviewer "@copilot"
```

The next `/loop` cycle will check for new comments from this re-review.

---

## Usage

```
/loop 2m /copilot-review
```

Or with a specific PR:

```
/loop 2m Check Copilot review on PR #123 (owner/repo) using /copilot-review
```

---

## gh Command Reference

### Trigger Copilot Review

```bash
gh pr edit <PR_NUM> --repo <OWNER/REPO> --add-reviewer "@copilot"
```

### Fetch Copilot Review Comments (REST)

For quick inspection without thread IDs — useful for a summary view:

```bash
gh api "repos/${OWNER}/${REPO}/pulls/${PR_NUM}/comments" \
  --jq '.[] | select(.user.login | test("copilot"; "i"))
    | "### \(.path):\(.line // .original_line)\n\(.body)\n"'
```

---

## Important Notes

- **Cap at 10 iterations.** Beyond that usually signals an architectural disagreement — stop and ask the user to review manually.
- **Copilot reviews are "Comment" type** — they never Approve or Block merge.
- **Free for open-source repos.** No Copilot subscription required.
- **Add `.github/copilot-instructions.md`** to guide Copilot's review behavior (coding style, conventions, etc.).
