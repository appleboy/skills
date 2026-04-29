# Skills

[![Goreleaser](https://github.com/appleboy/skills/actions/workflows/goreleaser.yml/badge.svg)](https://github.com/appleboy/skills/actions/workflows/goreleaser.yml)
[![Trivy Security Scan](https://github.com/appleboy/skills/actions/workflows/security.yml/badge.svg)](https://github.com/appleboy/skills/actions/workflows/security.yml)

Agent Skills Collection.

## Installation

Add this marketplace in Claude Code:

```bash
/plugin marketplace add appleboy/skills
```

Then install the skills you need:

```bash
/plugin install commit-message
/plugin install copilot-review
/plugin install prompt-audit
/plugin install pr-prepare
/plugin install plan-feature
/plugin install code-review
/plugin install classify-change
```

## Available Skills

| Skill                 | Description                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ |
| `commit-message`      | Generate a conventional commit message by analyzing staged git changes               |
| `copilot-review`      | Single-pass Copilot review check-fix cycle, use with `/loop` to repeat               |
| `prompt-audit`        | Audit a prompt against the 6 essential elements and produce an improved rewrite      |
| `pr-prepare`          | Prepare a PR description with AI-authorship disclosure and pre-submit checklist      |
| `plan-feature`        | Plan a feature before coding and produce a `plan.md` handoff document                |
| `code-review`         | Review a PR with rigor calibrated to leaf vs core classification                     |
| `classify-change`     | Classify a change as leaf or core to decide AI involvement and review rigor          |

## References

- [Building an AI-Driven Development Workflow with Claude Code + GitHub Copilot Review](https://blog.wu-boy.com/2026/03/ai-driven-development-with-claude-code-and-github-copilot-review-en/)
- [What Is Agent Skill? How It Changes the Software Industry](https://blog.wu-boy.com/2026/03/what-is-agent-skill-and-impact-on-software-industry-en/)
