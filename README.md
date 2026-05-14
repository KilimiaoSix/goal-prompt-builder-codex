# goal-prompt-builder for Codex

Codex skill for drafting audit-friendly `/goal` commands for Codex CLI 0.128+.

[简体中文](README_CN.md) · English

This is a Codex-format adaptation of win4r's `goal-prompt-builder` skill. It keeps the original workflow, scenario templates, project defaults, and examples, while converting the main skill instructions and metadata to Codex's `SKILL.md` + `agents/openai.yaml` format.

## What It Does

- Turns fuzzy long-running task requests into copy-pasteable `/goal` commands.
- Uses the Objective / Scope / Constraints / Done when / Stop if / token budget structure.
- Detects common project types from local files.
- Reads project rules such as `AGENTS.md` and `CLAUDE.md` when present.
- Pushes back on vague goals before rendering a command.
- Includes scenario templates for refactors, SDD features, batch work, archaeology, UI audits, and gatekeeper reviews.

## Install

Clone the repository and copy the skill folder into Codex's skills directory:

```bash
git clone https://github.com/KilimiaoSix/goal-prompt-builder-codex.git
mkdir -p ~/.codex/skills
cp -R goal-prompt-builder-codex/goal-prompt-builder ~/.codex/skills/
```

Restart Codex after installation.

## Use

Examples:

```text
Use $goal-prompt-builder to design a goal for migrating auth from v1 to v2.
```

```text
我要用 /goal 给这个项目新增 Cohere Rerank provider，帮我写一个可审计的 goal。
```

## Repository Layout

```text
goal-prompt-builder/
  SKILL.md
  agents/openai.yaml
  references/examples.md
  references/project-types.md
  references/scenarios.md
```

## Attribution

Adapted from `goal-prompt-builder` by win4r:

<https://github.com/win4r/goal-prompt-builder>

The original project is licensed under the MIT License. This Codex adaptation is also released under the MIT License.
