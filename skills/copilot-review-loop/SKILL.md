---
name: copilot-review-loop
description: "Automates the iterative PR review loop with GitHub Copilot Review using /loop and gh CLI. Use this skill whenever the user wants to: request a Copilot code review on a PR, run a fix-push-re-review cycle, or automate PR review iteration. Trigger on phrases like 'copilot review', 'review loop', 'fix PR comments', '/loop', 'iterate on PR', 're-review', or when the user mentions 'gh pr' with 'copilot'. This skill is essential for any workflow that involves GitHub Copilot as an automated code reviewer."
---

# Copilot Review Loop

Use Claude Code's `/loop` with `gh` CLI to automate the **Copilot Review → Fix → Re-review** cycle.

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

Before fetching comments, confirm that Copilot has finished its latest review. A review may still be in progress after a re-trigger.

```bash
gh pr view ${PR_NUM} --repo ${OWNER}/${REPO} --json reviews -q '
  [.reviews[]
    | select(.author.login | test("copilot"; "i"))]
  | sort_by(.submittedAt)
  | .[-1].state // "PENDING"
'
```

- If the state is `"PENDING"`, Copilot is still reviewing — **wait for the next loop iteration**.
- If the state is `"APPROVED"`, Copilot has no suggestions — report "Copilot review passed, ready for human review" and **stop the loop**.
- If the state is `"COMMENTED"` or `"CHANGES_REQUESTED"`, proceed to Step 3.

### 3. Fetch unresolved Copilot review comments

Use the GraphQL API to get only unresolved threads authored by Copilot. The REST API (`gh pr view --json comments`) does not include inline review comments.

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

Use the **commit-message** skill (`/commit`) to generate a proper conventional commit message for the fixes. This ensures commit messages follow the repo's conventions and include a meaningful summary.

After committing, push the changes:

```bash
git push
```

### 7. Resolve fixed threads

After pushing, resolve the threads that were successfully addressed:

```bash
gh api graphql -f query='
  mutation($threadId: ID!) {
    resolveReviewThread(input: {threadId: $threadId}) {
      thread { isResolved }
    }
  }
' -F threadId="<THREAD_NODE_ID>"
```

### 8. Re-trigger Copilot review

```bash
gh pr edit ${PR_NUM} --repo ${OWNER}/${REPO} --add-reviewer "@copilot"
```

The loop will check again on the next iteration for new comments from the re-review.

---

## Usage

Run directly in Claude Code:

```
/loop 5m Check PR #<PR_NUM> (<OWNER/REPO>) for Copilot review comments.
If there are new unresolved review comments:
1. Read the comment content
2. Fix the code accordingly
3. Run tests to confirm they pass
4. Commit & push
5. Re-trigger review with: gh pr edit <PR_NUM> --repo <OWNER/REPO> --add-reviewer "@copilot"
If there are no new comments, report "Review passed, ready for human review" and stop.
```

If the user doesn't specify a PR number, auto-detect it from the current branch (see Step 1).

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
