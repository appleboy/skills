# Evidence capture recipes

Detailed recipes for Step 6b of the `pr-prepare` skill. SKILL.md holds the decision rules, template, and safety constraints; this file only shows how to produce the artifacts. Prefer user-supplied files and repository scripts over everything here, and use a generic tool only when it is already available — do not install global tools or fetch new packages without asking.

## Artifact directory

Keep every artifact outside the repository so it can never be staged by accident:

```bash
EVIDENCE_DIR="${TMPDIR:-/tmp}/pr-evidence/<head-branch>"
mkdir -p "$EVIDENCE_DIR"
```

## Base checkout in a separate worktree

Build and run the base branch without touching the user's working tree:

```bash
git worktree add --detach "$EVIDENCE_DIR/base-tree" <base>
# install dependencies and start the app from "$EVIDENCE_DIR/base-tree" as the repository documents
# ... capture base artifacts ...
git worktree remove "$EVIDENCE_DIR/base-tree"
```

Run head from the user's current checkout (or a second worktree when the dev server cannot run twice on the same port). Use different ports for base and head when both run at once. If base cannot be built or started, capture head only and say so in the Evidence section.

## Web screenshots

Use the same viewport, data, account role, and UI state for base and head. Only point tools at local or non-production URLs, and avoid pages that show PII, tokens, or internal hostnames.

### Repository e2e or Playwright configuration

If the repository already has Playwright or another e2e setup, prefer adding a throwaway script outside the repository that reuses its configuration, or run an existing visual test with its documented command, and copy the produced images into `$EVIDENCE_DIR`.

### Playwright CLI

When Playwright is already installed in the repository:

```bash
npx playwright screenshot \
  --viewport-size "1280, 800" \
  --wait-for-selector "<selector that signals the page is ready>" \
  "http://localhost:<port>/<route>" \
  "$EVIDENCE_DIR/after-<page>-<state>.png"
```

Add a mobile capture with `--viewport-size "390, 844"` and the `-mobile` suffix when the change is responsive. Add `--full-page` only when the change sits below the fold, and use it for both base and head.

### Headless Chrome

When a Chrome or Chromium binary is available:

```bash
<chrome-binary> --headless=new --hide-scrollbars \
  --window-size=1280,800 \
  --screenshot="$EVIDENCE_DIR/before-<page>-<state>.png" \
  "http://localhost:<port>/<route>"
```

Headless Chrome cannot log in or click through states by itself; use it only for pages reachable by URL.

### Browser skills

A browser skill such as `/browse` may be used when it is installed in the current agent. Do not assume it exists in Claude Code or Codex.

## Performance report

Always record the environment (CPU, OS, runtime version, dataset size), the warm-up and run count, the statistic, and the variance. Run base and head on the same machine, back to back, with no other heavy load.

### Go benchmarks with benchstat

```bash
# in "$EVIDENCE_DIR/base-tree"
go test -run '^$' -bench '<BenchmarkName>' -benchmem -count 10 ./<pkg>/... > "$EVIDENCE_DIR/base-bench.txt"
# in the head checkout
go test -run '^$' -bench '<BenchmarkName>' -benchmem -count 10 ./<pkg>/... > "$EVIDENCE_DIR/head-bench.txt"

benchstat "$EVIDENCE_DIR/base-bench.txt" "$EVIDENCE_DIR/head-bench.txt"
```

`benchstat` reports `~` when the difference is not statistically significant; report that as inconclusive.

### Command-line workloads with hyperfine

```bash
hyperfine --warmup 3 --runs 20 \
  --export-markdown "$EVIDENCE_DIR/hyperfine.md" \
  --export-json "$EVIDENCE_DIR/hyperfine.json" \
  -n base "<base command>" \
  -n head "<head command>"
```

### Page performance with Lighthouse

When the Lighthouse CLI is available:

```bash
lighthouse "http://localhost:<port>/<route>" \
  --chrome-flags="--headless=new" \
  --output=json \
  --output-path="$EVIDENCE_DIR/after-lighthouse.json"
```

Lighthouse scores vary between runs; run each side several times and report the median.

### HTTP load with k6

When k6 and a repository load script exist:

```bash
k6 run --summary-export "$EVIDENCE_DIR/head-k6.json" <script.js>
```

Compare p95 latency, error rate, and request rate between base and head runs with identical virtual users, duration, and dataset.

### Bundle size

Use the repository's build command for base and head, then compare the output sizes (for example with `du -k` on the build directory or the bundler's own stats output). Report the files compared, not just a total.

## Reporting

- Put raw tool output in a collapsed `<details>` block in the Performance report.
- When the variance overlaps the difference, write "inconclusive" and explain why.
- When a capture fails, record `Not run`, the reason, and the environment or access needed.
