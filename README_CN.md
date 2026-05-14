# goal-prompt-builder for Codex

> 一个用于生成高质量 Codex `/goal` 命令的 Codex Skill，目标是让长任务可审计、可停止、不容易跑偏。

简体中文 · [English](README.md)

## 这个仓库是什么

这是 [win4r/goal-prompt-builder](https://github.com/win4r/goal-prompt-builder) 的 Codex 版本适配。

原仓库是 Claude Skill 格式，面向 Claude Code / Claude Desktop / Claude.ai 的 skills 目录；这个仓库把它转换成 Codex Skill 格式：

- 使用 Codex 的 `SKILL.md` frontmatter 和触发描述。
- 新增 `agents/openai.yaml`，用于 Codex UI 元数据。
- 保留原来的 3 个 reference 文件：项目类型默认值、场景模板、完整示例。
- 安装路径改为 `~/.codex/skills/`。

核心目标不变：把一句模糊的长期任务需求，转换成可以直接粘贴到 Codex CLI 的 `/goal` 命令。

## 为什么需要它

Codex CLI 0.128+ 引入了 `/goal`：一个持久化目标机制，带运行时延续、完成审计、token budget 和软停止。

它很强，但也有一个明显失败模式：目标写得太糊，Codex 就会长时间朝错误方向推进，最后还可能误判完成。

例如这种 goal 风险很高：

```text
/goal 修一下所有 flaky 测试，顺便整理一下代码
```

问题在于：

- “所有”没有可枚举来源。
- “整理一下”没有可验证状态。
- “测试通过”容易退化成代理信号。
- 没有 Stop if，跑偏时没有边界。
- 没有 token budget，成本不可控。

这个 skill 会把目标改写成更适合 `/goal` 审计机制的结构：

```text
/goal <一句话目标>.

Scope: <允许修改的文件 / 子系统 / 功能范围>.

Constraints:
  - <不能做什么>
  - <兼容性和项目规则>

Done when:
  1. <可验证产物：文件、命令、测试名、数量或报告路径>
  2. <可验证产物>

Stop if:
  - <可机械识别的停止条件>
  - <越界或破坏性操作时停止>

Use a token budget of <N> tokens for this goal.
```

## 一句话安装

```bash
git clone https://github.com/KilimiaoSix/goal-prompt-builder-codex.git
mkdir -p ~/.codex/skills
cp -R goal-prompt-builder-codex/goal-prompt-builder ~/.codex/skills/
```

安装后重启 Codex。

## 怎么使用

可以显式调用：

```text
Use $goal-prompt-builder to design a goal for migrating auth from v1 to v2.
```

也可以直接用中文描述：

```text
我要用 /goal 给这个项目新增 Cohere Rerank provider，帮我写一个可审计的 goal。
```

典型触发场景：

- “帮我写一个 /goal”
- “给这个 repo 设计一个 goal”
- “review 一下我的 goal command”
- “我要让 Codex 持续跑到完成”
- “这个任务要无人值守跑几个小时”
- “我想用 `/goal` 做一次重构 / 批量修复 / SDD 实现”

## 工作流程

触发后，这个 skill 会按 6 步工作：

1. 选择交互模式：询问式、全描述式、混合式。
2. 自动检测项目类型：Node / Python / Swift / Go / Rust / 静态文档等。
3. 读取项目规则：优先读取 `AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md`。
4. 选择场景模板：重构、新功能、批量任务、代码考古、UI audit、守门员 review、自定义。
5. 收集五段输入：Objective、Scope、Constraints、Done when、Stop if。
6. 输出最终 `/goal`，并给出简短设计理由和审计友好度判断。

## 内置场景

| 场景 | 适合什么任务 |
| --- | --- |
| Refactor | 单文件、模块、子系统重构 |
| Feature | 新功能实现，尤其是 OpenSpec / SDD 驱动 |
| Batch | 批量补测试、修 bug、重命名 |
| Archaeology | 只读代码考古和架构梳理 |
| UI Audit | 对照文档审查实际 UI / 行为 |
| Gatekeeper | 判断 PR / 变更是否可以合并 |
| Custom | 不适合上述模板的自定义目标 |

## 项目类型默认规则

这个 skill 会根据项目类型自动补一些 Stop if 和 Constraints：

- Node / TypeScript：避免新增 npm 依赖，使用具体 `npm test` / `tsc` 命令。
- Python：避免新增 `requirements.txt` / `pyproject.toml` 依赖，明确 `pytest` 命令。
- Swift / iOS：默认保护 `project.pbxproj`，处理 simulator 不可用等情况。
- Go：使用 `go test ./...`、`go vet ./...`，避免不必要的 module 变更。
- Rust：使用 `cargo test`、`cargo clippy`、`cargo fmt --check`。
- Static / docs：明确只读或文档输出边界，避免无意义构建检查。

## 硬规则

这个 Codex 版本保留了原仓库的关键规则：

1. 不放过 “all / everything / 全部 / 彻底” 这类不可枚举表达。
2. 不把 “测试通过” 当成验收项，必须写成具体命令、退出码和输出摘要。
3. 每个 goal 都必须有 token budget。
4. 任何涉及测试代码的 goal，都要加“不许通过削弱测试来让测试通过”的 Stop if。
5. SDD / OpenSpec 类任务必须先读规格文件并报告计数，再开始实现。
6. brownfield 项目必须询问 MUST NOT modify 清单。
7. 审计友好度明显不够时，不直接输出最终 goal，而是先指出缺口。

## 仓库结构

```text
goal-prompt-builder-codex/
├── README.md
├── README_CN.md
├── LICENSE
└── goal-prompt-builder/
    ├── SKILL.md
    ├── agents/
    │   └── openai.yaml
    └── references/
        ├── examples.md
        ├── project-types.md
        └── scenarios.md
```

## 和原仓库的区别

| 项目 | 原仓库 | 本仓库 |
| --- | --- | --- |
| Skill 格式 | Claude Skill | Codex Skill |
| 安装目录 | `~/.claude/skills/` | `~/.codex/skills/` |
| UI 元数据 | 无 Codex `openai.yaml` | 包含 `agents/openai.yaml` |
| 主说明对象 | Claude | Codex |
| references | 保留 | 保留并适配 |
| 许可证 | MIT | MIT |

## 致谢和授权

本仓库基于 win4r 的 `goal-prompt-builder` 适配：

<https://github.com/win4r/goal-prompt-builder>

原项目采用 MIT License。本 Codex 适配版本同样采用 MIT License，并在 `LICENSE` 中保留原作者版权信息。
