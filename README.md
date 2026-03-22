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
```

## Available Skills

| Skill                 | Description                                                            |
| --------------------- | ---------------------------------------------------------------------- |
| `commit-message`      | Generate a conventional commit message by analyzing staged git changes |
| `copilot-review`      | Single-pass Copilot review check-fix cycle, use with `/loop` to repeat |
