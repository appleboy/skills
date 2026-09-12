# Skills

[![Goreleaser](https://github.com/appleboy/skills/actions/workflows/goreleaser.yml/badge.svg)](https://github.com/appleboy/skills/actions/workflows/goreleaser.yml)
[![Trivy Security Scan](https://github.com/appleboy/skills/actions/workflows/security.yml/badge.svg)](https://github.com/appleboy/skills/actions/workflows/security.yml)

Agent Skills Collection.

## Installation

Claude Code and Codex manage plugins separately. Install or update the plugins in each tool you use. Each plugin includes its corresponding skill.

### Claude Code

In a Claude Code session, add the marketplace:

```text
/plugin marketplace add appleboy/skills
```

Then run the install commands for the plugins you need:

```text
/plugin install commit-message@appleboy-skills
/plugin install copilot-review@appleboy-skills
/plugin install codex-review@appleboy-skills
/plugin install prompt-audit@appleboy-skills
/plugin install pr-prepare@appleboy-skills
/plugin install plan-feature@appleboy-skills
/plugin install classify-change@appleboy-skills
```

You can also install from a terminal:

```bash
claude plugin marketplace add appleboy/skills
claude plugin install pr-prepare@appleboy-skills
```

To update, refresh the marketplace and update each installed plugin you want to upgrade. For example:

```bash
claude plugin marketplace update appleboy-skills
claude plugin update pr-prepare@appleboy-skills
claude plugin list
```

Replace `pr-prepare` with another plugin name as needed. Restart Claude Code to apply updates.

### Codex

Use a Codex CLI version that supports `codex plugin` (check with `codex plugin --help`). Run these commands in a terminal to add the marketplace and install the plugins you need:

```bash
codex plugin marketplace add appleboy/skills

codex plugin add commit-message@appleboy-skills
codex plugin add copilot-review@appleboy-skills
codex plugin add codex-review@appleboy-skills
codex plugin add prompt-audit@appleboy-skills
codex plugin add pr-prepare@appleboy-skills
codex plugin add plan-feature@appleboy-skills
codex plugin add classify-change@appleboy-skills
```

To update, refresh the marketplace and run `plugin add` again for each plugin you want to upgrade. For example:

```bash
codex plugin marketplace upgrade appleboy-skills
codex plugin add pr-prepare@appleboy-skills
codex plugin list --marketplace appleboy-skills --json
```

Replace `pr-prepare` with another plugin name as needed. Refreshing the marketplace alone does not replace the installed plugin cache. Start a new Codex session after updating so it loads the current skill instructions.

## Available Skills

| Skill                 | Description                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ |
| `commit-message`      | Generate a conventional commit message by analyzing staged git changes               |
| `copilot-review`      | Single-pass Copilot review check-fix cycle, use with `/loop` to repeat               |
| `codex-review`        | Single-pass Codex (ChatGPT) review check-fix cycle, use with `/loop` to repeat       |
| `prompt-audit`        | Audit a prompt against the 6 essential elements and produce an improved rewrite      |
| `pr-prepare`          | Prepare a PR description with AI-authorship disclosure and pre-submit checklist      |
| `plan-feature`        | Plan a feature before coding and produce a `plan.md` handoff document                |
| `classify-change`     | Classify a change as leaf or core to decide AI involvement and review rigor          |

## References

- [Building an AI-Driven Development Workflow with Claude Code + GitHub Copilot Review](https://blog.wu-boy.com/2026/03/ai-driven-development-with-claude-code-and-github-copilot-review-en/)
- [What Is Agent Skill? How It Changes the Software Industry](https://blog.wu-boy.com/2026/03/what-is-agent-skill-and-impact-on-software-industry-en/)
