# n8n-dev-skills

Agent Skills for developers working with [n8n](https://n8n.io) — building custom community nodes and authoring workflows as code.

These skills give your coding agent the official n8n toolchain knowledge it otherwise lacks: how to scaffold and validate a custom node against the current linter and verification rules, and how to construct workflows programmatically with the `@n8n/workflow-sdk`. Each is grounded in the latest n8n source and docs, not the model's training cutoff.

## Plugins

| Plugin | Description |
|--------|-------------|
| [n8n-node-builder](plugins/n8n-node-builder/) | Scaffolds, implements, validates, and publishes custom n8n nodes using official best practices. Covers declarative, programmatic, and AI sub-node styles, all credential/auth patterns, trigger nodes, a gate-based self-validation protocol (lint, build, cloud-support, runtime, verification scan), and verification requirements. |
| [n8n-workflow-sdk](plugins/n8n-workflow-sdk/) | Builds, tests, validates, and manages n8n workflows programmatically using the `@n8n/workflow-sdk`. Covers workflow creation, JSON import/export, validation, code generation, AI agent workflows, and the full SDK API. |

## Install via Claude Code Plugin Marketplace

Add this marketplace and install the plugins:

```
/plugin marketplace add geckse/n8n-dev-skills
/plugin install n8n-node-builder@n8n-dev-skills
/plugin install n8n-workflow-sdk@n8n-dev-skills
```

Once installed, Claude automatically activates the appropriate skill — `n8n-node-builder` when you ask about building custom n8n nodes, and `n8n-workflow-sdk` when you ask about creating or managing n8n workflows with code.

## Manual Usage

Copy or symlink a skill directory from `plugins/<plugin-name>/skills/` into your agent's skills folder. The agent will discover it automatically via the `SKILL.md` frontmatter.

## Format

Skills follow the [Agent Skills](https://agentskills.io) open specification and are packaged as a [Claude Code plugin](https://code.claude.com/docs/en/plugins).
