---
name: goal-prompt-builder
description: Build high-quality /goal commands for OpenAI Codex CLI 0.128+ that are audit-friendly and resistant to false completion. Use when the user wants to write, draft, generate, improve, or review a /goal prompt, including requests like "help me write a goal", "design a goal for X", "review my goal command", "make a goal for this repo", long-running Codex tasks, persistent agent objectives, Ralph loop style work, or "keep working until done". Produces a copy-pasteable /goal command using Objective, Scope, Constraints, Done when, Stop if, and an optional explicit token budget; supports step-by-step, full-description, and hybrid workflows; detects project type from local files or repo context; reads AGENTS.md/CLAUDE.md when present; and checks audit-friendliness before output.
---

# Goal Prompt Builder

Use this skill to turn a fuzzy task request into a complete, audit-friendly `/goal` command for Codex CLI 0.128+.

The job is not to run `/goal`. The job is to help the user write a goal that Codex can pursue safely, verify mechanically, and stop when boundaries are crossed or explicit stop conditions are hit.

## Output Contract

Final output should contain:

1. A fenced code block with the complete `/goal` command.
2. A short "key design choices" list with no more than 8 bullets.
3. An optional one-line audit-friendliness verdict, such as `审计友好度：优秀 · 7 项验收 · 0 风险标记`.

Do not provide a long tutorial after generating the command. The user needs copy-pasteable text and the minimum rationale needed to trust it.

## Golden Template

Every generated goal should follow this structure and order:

```text
/goal <objective>.

[Optional: First action: read X, Y, Z and report counts in the first progress update. Continue without waiting unless the user explicitly asks for a manual approval gate.]

Scope: <files / subsystem / feature area>.

Constraints:
  - <what not to change>
  - <compatibility / permission boundaries>
  - <project-specific rules from AGENTS.md / CLAUDE.md>

Done when:
  1. <verifiable artifact 1: cite file, command, test name, or measurable output>
  2. <verifiable artifact 2>

Stop if:
  - <mechanically detectable condition 1>
  - <mechanically detectable condition 2>

[Optional: Use a token budget of <N> tokens for this goal. Include only when the user explicitly supplied a budget or explicitly asked for a budget recommendation.]
```

This order matches the way goal continuation audits work: objective first, then scope to bound the search, constraints to prune unsafe paths, acceptance criteria to define success, and stop-if bullets as runtime guards.

Do not render bracketed `[Optional: ...]` notes verbatim. Either omit the optional section entirely or render it as normal `/goal` text only when its condition applies.

## Workflow

### 1. Choose Interaction Mode

Ask once at the start unless the user already provided enough detail:

```text
你希望用哪种方式生成 /goal？
A. 询问式：我一段一段问你，最稳
B. 全描述式：你一次描述需求，我只追问缺口
C. 混合式：先选场景模板，再问关键问题，默认推荐
```

If the user does not choose, use C. Do not block progress on this choice when enough information is already available.

### 2. Detect Project Type Before Asking

Inspect available context first:

- Conversation has repo URL, file path, or snippets: infer from those.
- `package.json`: Node / TypeScript.
- `pyproject.toml`, `requirements.txt`, or `setup.py`: Python.
- `*.xcodeproj/` or `Package.swift`: Swift / iOS.
- `Cargo.toml`: Rust.
- `go.mod`: Go.
- `astro.config.*`, `next.config.*`, `_config.yml`, or `mkdocs.yml`: static / docs project.

If a GitHub URL is provided and internet access is available, inspect README and top-level config files. Read `AGENTS.md` and `CLAUDE.md` when present; these project rules override defaults.

When detection succeeds, state it in one sentence so the user can correct it. Then load the relevant section of `references/project-types.md`.

If detection fails, ask one concise question:

```text
我没法自动判断项目类型：这是 Node / Python / Swift / Go / Rust / 静态文档 / 其他？
```

### 3. Select Scenario Template

Ask which scenario fits, then load the matching section from `references/scenarios.md`:

| Option | Scenario |
| --- | --- |
| A | Refactor: one file, module, or subsystem |
| B | Feature: especially with SDD/OpenSpec style specs |
| C | Batch: tests, bug fixes, renames, repeated work |
| D | Archaeology: read-only research/reporting |
| E | UI Audit: compare claimed behavior with implementation |
| F | Gatekeeper: evaluate whether a PR/change is mergeable |
| G | Custom: use the bare golden template |

Each scenario has different audit pressure. Feature goals should prefer a `First action: read specs and report counts` guard that does not wait for confirmation by default. Archaeology goals should be explicitly read-only. Batch goals need a numeric target.

### 4. Gather the Required Inputs

Ask only for missing information. In hybrid mode, batch the missing fields into one concise question.

Objective:
- Use one sentence describing the end state.
- Turn noun phrases into verb phrases.
- Reject vague verbs like `improve`, `optimize`, `clean up`, `提升`, `优化`, `全部`, `彻底`.

Scope:
- Name the files, directories, subsystems, or feature area in play.
- Identify what should not be touched.
- For brownfield work, ask whether there is a MUST NOT modify list.

Constraints:
- Pull hard rules from `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, or user instructions.
- Add project-type defaults from `references/project-types.md`.
- Prefer mechanically checkable constraints over vague quality statements.

Done when:
- Require at least 3 verifiable items before rendering a final goal.
- Each item should cite a file, command, test name, count, report path, or measurable artifact.
- Replace "tests pass" with an exact command plus expected exit code and summary.
- Aim for 5-8 items for medium tasks.

Stop if:
- Use mechanically detectable conditions.
- Include dependency, schema, destructive operation, and out-of-scope file guards when relevant.
- For tested code, include a no-test-rewriting guard: existing tests failing is a regression; do not make them pass by weakening tests.

Token budget (optional):
- Do not invent or add a token budget by default.
- Include a token budget line only when the user explicitly supplied a budget or explicitly asked for a budget recommendation.
- If the user asks for a recommendation, propose a conservative budget based on scope:
  - 20K-40K for narrow single-file or prompt/report work.
  - 60K-100K for medium refactors or bounded feature work.
  - 120K-180K for spec-driven implementation across several modules.
  - Above 300K usually means split into multiple goals.
- If the user did not mention budget, omit the entire `Use a token budget...` line from the final `/goal`.

## Audit-Friendliness Check

Before outputting the final command, internally check:

- Acceptance count: 0 bad, 1-2 weak, 3-5 good, 6-8 strong.
- Vague verbs: `improve`, `optimize`, `clean up`, `全部`, `彻底`, `all`, `everything`.
- Stop-if specificity: concrete file/command/state beats "if unclear".
- Token budget: absent by default; if present, it must be explicitly user-supplied or explicitly requested.
- Mechanical verifiability: every Done when item should have a concrete evidence target.

If audit-friendliness is below roughly 70%, do not render the final command. Surface the weak spots and ask for refinement. Be specific about which Done when or Stop if item needs tightening.

## References

Load only the reference needed for the current case:

- `references/project-types.md`: project defaults, likely commands, and traps for Node/TypeScript, Python, Swift/iOS, Go, Rust, static/docs, and unknown projects.
- `references/scenarios.md`: fillable skeletons and rationale for refactor, feature, batch, archaeology, UI audit, gatekeeper, and custom goals.
- `references/examples.md`: worked examples, including when to push back instead of rendering.

## Hard Rules

1. Do not write Stop if as "if unclear, stop"; ask for concrete conditions.
2. Do not let `all`, `everything`, `全部`, or `彻底` pass without converting them into numbers or enumerable sources.
3. Do not include a token budget unless the user explicitly supplied one or explicitly asked for a recommendation.
4. For goals touching tested code, include a no-test-rewriting regression guard.
5. For SDD/OpenSpec feature goals, include a first action to read spec files and report counts before implementation, but do not require user confirmation unless the user explicitly requests an approval gate.
6. For brownfield projects, ask about MUST NOT modify lists.
7. If the user only asks for a blank template, provide the template and skip the interview.

## What This Skill Does Not Do

- Does not start or manage the `/goal` run.
- Does not validate the current repository state unless needed to draft the prompt.
- Does not support Codex versions older than 0.128.
- Does not generate `/plan`, `/compact`, or unrelated Codex command prompts.
