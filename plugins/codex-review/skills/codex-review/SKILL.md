---
name: codex-review
description: "Runs a single Codex (ChatGPT/OpenAI) review check-fix cycle on a PR using gh CLI. Use this skill whenever the user wants to: check and fix Codex review comments, request a Codex code review on a PR, or iterate on PR feedback from the chatgpt-codex-connector bot. Trigger on phrases like 'codex review', '@codex review', 'fix codex comments', 'iterate on PR', 're-review with codex', or when the user mentions 'gh pr' with 'codex'. Combine with /loop for automated repeated cycles, e.g. '/loop 2m /codex-review'."
---

# Codex Review

A single-pass check-fix cycle for Codex (the `chatgpt-codex-connector` GitHub bot) review comments. Designed to be called repeatedly via `/loop`:

```
/loop 2m /codex-review
```

Each invocation runs one iteration: check for Codex's verdict → fix unresolved comments → push → resolve → re-trigger review. The loop terminates ONLY when Codex's latest result (for the latest commit) says `"Didn't find any major issues"`.

## How Codex differs from other PR reviewers

Codex stamps every result with the commit it reviewed — `**Reviewed commit:** \`<sha>\``. This is the authoritative signal for "did it review my latest push?", which makes the timing logic far more reliable than timestamp guessing. Two more quirks to internalize:

- Codex is triggered by an **issue comment** (`@codex review`), not by adding a reviewer.
- Codex's "all clear" verdict arrives as a plain **issue comment** with a 👍 (`"Didn't find any major issues"`), while its "found issues" verdict arrives as a **PR review** with inline threads. So you must look in **both** places and pick the most recent by time.

## Prerequisites

- `gh` CLI v2.88.0+
- Authenticated: `gh auth status`
- Codex enabled for the repo (team-level setting at chatgpt.com/codex). Free tiers/forks may not have it.

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

### 2. Check Codex review status

Before acting on comments, confirm two things: (a) Codex has produced a result, and (b) that result covers the **latest** commit. The danger without the commit check is a race: you fix, push, and re-trigger, but the next cycle reads Codex's *previous* result, sees its old threads resolved, and falsely concludes "all clear" — stopping the loop before Codex's fresh review lands.

Codex's verdict can be either a **review** (when it found issues) or an **issue comment** (its 👍 "all clear"). Fetch both in one GraphQL query, along with the latest commit SHA and all unresolved Codex threads:

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviews(last: 20) {
          nodes { author { login } submittedAt body }
        }
        comments(last: 30) {
          nodes { author { login } createdAt body }
        }
        commits(last: 1) {
          nodes { commit { oid } }
        }
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            comments(first: 10) {
              nodes { body author { login } path line originalLine }
            }
          }
        }
      }
    }
  }
' -F owner="${OWNER}" -F repo="${REPO}" -F pr="${PR_NUM}" \
  --jq '{
    latestResult: ([
        (.data.repository.pullRequest.reviews.nodes[]
          | select(.author.login == "chatgpt-codex-connector")
          | select(.body | contains("Reviewed commit"))
          | {at: .submittedAt, body}),
        (.data.repository.pullRequest.comments.nodes[]
          | select(.author.login == "chatgpt-codex-connector")
          | select(.body | contains("Reviewed commit"))
          | {at: .createdAt, body})
      ] | sort_by(.at) | .[-1]),
    latestCommit: .data.repository.pullRequest.commits.nodes[-1].commit.oid,
    unresolvedThreads: [.data.repository.pullRequest.reviewThreads.nodes[]
      | select(.isResolved == false)
      | select(.comments.nodes[0].author.login == "chatgpt-codex-connector")
      | {
          threadId: .id,
          path: .comments.nodes[0].path,
          line: (.comments.nodes[0].line // .comments.nodes[0].originalLine),
          comment: .comments.nodes[0].body
        }]
  }'
```

The `contains("Reviewed commit")` filter keeps only Codex's actual review *verdicts* — every verdict (both "found issues" and "all clear") carries a `**Reviewed commit:** \`<sha>\`` line. This deliberately ignores Codex's conversational replies (e.g. an answer to `@codex address that feedback`), which lack that line and would otherwise hijack `latestResult` and leave you with no SHA to compare.

From `latestResult.body`, extract the reviewed commit SHA from that `**Reviewed commit:** \`<sha>\`` line (a short SHA, typically 10 chars). Call it `reviewedSha`.

Apply this decision logic in order:

1. **No Codex result yet** (`latestResult` is null) — report "No Codex review found, triggering one..." → run the trigger from Step 8 → **return (end this cycle; wait for the next `/loop` invocation).**
2. **`reviewedSha` does not match `latestCommit`** — Codex's latest result reviewed an older commit, so it hasn't seen your latest push. (Match by prefix: `latestCommit` starts with `reviewedSha`.) **Report "Waiting for Codex to review latest commit (`<latestCommit short>`)..." and return (end this cycle; do NOT stop the loop).**
3. **`reviewedSha` matches `latestCommit`** — Codex reviewed the current HEAD. Now read `latestResult.body`:
   - Contains `"Didn't find any major issues"` → Codex is done with no issues. Report "Codex review passed (no major issues) — ready for human review" and **STOP THE LOOP (terminate entirely).** This is the ONLY check that may terminate the loop.
   - Contains `"Here are some automated review suggestions"` (or `unresolvedThreads` is non-empty) → Codex found issues. **Proceed to Step 3.**
   - **Neither phrase present** (an overview-only or reworded verdict) → don't guess. Fall back to `unresolvedThreads`: if it's non-empty, **proceed to Step 3** to fix them; if it's empty, re-trigger (Step 8) and return so Codex produces a phrase you can act on. Never terminate the loop from this fallback — only the "Didn't find any major issues" phrase may stop it.

> **Edge case — re-triggered on the same commit.** If you skipped every suggestion last cycle (no code change, no new commit), re-triggering produces a fresh Codex result on the *same* SHA, so the SHA check alone can't tell old from new. In that situation only, fall back to a timestamp guard: record `lastSeenAt = latestResult.at` after Step 8, and at the top of check #3 treat `latestResult.at <= lastSeenAt` as "re-triggered review hasn't arrived yet" → report "Waiting for re-triggered Codex review..." and return (do NOT stop the loop). Once `latestResult.at` is newer, clear `lastSeenAt` and continue. Do NOT terminate the loop from this guard — Codex *was* re-triggered and *will* post a new result; it just takes time.

### 3. Process unresolved Codex review comments

The `unresolvedThreads` array from the Step 2 query already contains all unresolved threads authored by Codex.

If `latestResult.body` says `"Didn't find any major issues"`, Codex found nothing. Report "All clear — ready for human review" and **stop the loop**.

If Codex reported suggestions but `unresolvedThreads` is **empty**, all threads were already resolved — but Codex may surface new issues on re-review. **Do NOT stop.** Proceed to Step 7 (resolve any stragglers) and Step 8 (re-trigger) so Codex confirms with a fresh review.

If `unresolvedThreads` is **not empty**, proceed to fix each comment.

### 4. Fix the code

Codex tags each finding with a severity badge in the comment body — **P1** (high), **P2** (medium), **P3** (low). Address higher-severity findings first.

For each unresolved comment:

1. Read the referenced file and line.
2. Understand the suggestion — evaluate whether it makes sense in the project context.
3. Apply the fix.

**Do not blindly accept every suggestion.** Codex may flag stylistic nits, repeat already-resolved points, or propose changes that conflict with project architecture. If a suggestion is incorrect or irrelevant, skip it and note the reason. A P3 nit that fights the codebase's conventions is fine to decline with a short rationale.

**If no code changes were made** (all suggestions skipped as incorrect or irrelevant), skip Steps 5 and 6 — there are no tests to run and nothing to commit. Proceed directly to Step 7 to resolve the evaluated threads, then Step 8 to re-trigger. Because no new commit is pushed, the reviewed SHA won't change, so the `lastSeenAt` timestamp guard (Step 2 edge case) is what lets the next cycle wait correctly for the re-triggered review instead of acting on stale data.

### 5. Run tests

Run the project's test suite before committing. A fix that breaks other things is worse than the original issue.

If tests fail:
- Diagnose and fix the test failure.
- If the fix conflicts with the Codex suggestion, revert that suggestion and note it as skipped.
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

### 8. Re-trigger Codex review

Codex re-reviews when you post an issue comment containing `@codex review`:

```bash
gh pr comment ${PR_NUM} --repo ${OWNER}/${REPO} --body "@codex review"
```

**After re-triggering, if this cycle made no new commit (Step 4 skip-all case), record `lastSeenAt` = `latestResult.at` from this cycle's Step 2 query.** The next `/loop` cycle uses it in the Step 2 edge-case guard to detect whether the re-triggered review has arrived. When a new commit *was* pushed, you don't need `lastSeenAt` — the reviewed-SHA check handles it.

> **Alternative:** Instead of fixing the code yourself, you can ask Codex to do it by commenting `@codex address that feedback`. This skill defaults to fixing locally so you retain full control and run the tests yourself, but `@codex address` is useful for mechanical changes.

---

## Usage

```
/loop 2m /codex-review
```

Or with a specific PR:

```
/loop 2m Check Codex review on PR #123 (owner/repo) using /codex-review
```

---

## gh Command Reference

### Trigger Codex Review

```bash
gh pr comment <PR_NUM> --repo <OWNER/REPO> --body "@codex review"
```

### Fetch Codex Review Comments (REST)

For quick inspection without thread IDs — useful for a summary view:

```bash
gh api "repos/${OWNER}/${REPO}/pulls/${PR_NUM}/comments" \
  --jq '.[] | select(.user.login | test("codex"; "i"))
    | "### \(.path):\(.line // .original_line)\n\(.body)\n"'
```

---

## Important Notes

- **Cap at 10 iterations.** Beyond that usually signals an architectural disagreement — stop and ask the user to review manually.
- **Codex reviews are "Comment" type** — they never Approve or Block merge.
- **Author login:** Codex posts as `chatgpt-codex-connector` in both REST (`user.login`) and GraphQL (`author.login`). Match on it exactly (or case-insensitively on the substring `codex`) for reliable detection.
- **Reviewed-commit SHA is the source of truth for freshness.** Codex writes `**Reviewed commit:** \`<sha>\`` on every result. Compare it to the PR's latest commit `oid` (prefix match — Codex uses a short SHA). Don't rely on thread counts or timestamps for the "covers latest push?" question when a SHA is available.
- **Verdict can be a review OR an issue comment.** The "found issues" verdict is a PR review with inline threads; the "all clear" 👍 is a plain issue comment. Always merge both sources and take the most recent by time, or you'll miss the all-clear and loop forever.
- **Completion phrase is the only stop condition.** `"Didn't find any major issues"` on the latest commit means done. `"Here are some automated review suggestions"` means there are comments to address. Do NOT stop merely because `unresolvedThreads` is empty — threads from a prior review can all be resolved while a fresh review for the latest commit hasn't arrived yet.
- **Re-trigger without a new commit:** when all suggestions are skipped, no commit is pushed and the reviewed SHA stays the same; the `lastSeenAt` timestamp guard (Step 2 edge case) is the reliable way to wait for the re-triggered review.
- **Severity badges:** Codex labels findings P1/P2/P3 (high/medium/low). Triage by severity; declining a P3 with a rationale is acceptable.
- **Add `AGENTS.md`** at the repo root to steer Codex's review behavior (coding style, conventions, what to ignore), analogous to `.github/copilot-instructions.md` for Copilot.
