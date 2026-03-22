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

Example:

```
/loop 5m Check PR #36891 (go-gitea/gitea) for Copilot review comments.
If there are new unresolved review comments:
1. Read the comment content
2. Fix the code accordingly
3. Run tests to confirm they pass
4. Commit & push
5. Re-trigger review with: gh pr edit 36891 --repo go-gitea/gitea --add-reviewer "@copilot"
If there are no new comments, report "Review passed, ready for human review" and stop.
```

---

## gh Command Reference

### Trigger Copilot Review

```bash
gh pr edit <PR_NUM> --repo <OWNER/REPO> --add-reviewer "@copilot"
```

### Check Review Status

```bash
gh pr view <PR_NUM> --repo <OWNER/REPO> --json reviews -q '
  [.reviews[]
    | select(.author.login | test("copilot"; "i"))]
  | sort_by(.submittedAt)
  | .[-1].state // "PENDING"
'
```

### Fetch Copilot Review Comments

`gh pr view --json comments` does **not** include inline review comments. Use the REST API:

```bash
OWNER="go-gitea"
REPO="gitea"
PR_NUM=36891

gh api "repos/${OWNER}/${REPO}/pulls/${PR_NUM}/comments" \
  --jq '.[] | select(.user.login | test("copilot"; "i"))
    | "### \(.path):\(.line // .original_line)\n\(.body)\n"'
```

### Fetch Unresolved Review Threads (GraphQL)

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
        line: .comments.nodes[0].line,
        comment: .comments.nodes[0].body
      }'
```

### Resolve a Thread

```bash
gh api graphql -f query='
  mutation($threadId: ID!) {
    resolveReviewThread(input: {threadId: $threadId}) {
      thread { isResolved }
    }
  }
' -F threadId="<THREAD_NODE_ID>"
```

---

## Important Notes

- **Always run tests before pushing.** A fix that breaks other things is worse than the original issue.
- **Do not blindly accept every suggestion.** Copilot may repeat resolved comments or suggest changes that conflict with project architecture.
- **Cap at 10 iterations.** Beyond that usually signals an architectural disagreement — stop and review manually.
- **Copilot reviews are "Comment" type** — they never Approve or Block merge.
- **Free for open-source repos.** No Copilot subscription required.
- **Add `.github/copilot-instructions.md`** to guide Copilot's review behavior (coding style, conventions, etc.).
