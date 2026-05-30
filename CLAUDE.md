# CLAUDE.md

> 给 Claude Code 在这个 repo 工作时的脉络指引。维持简体中文书写。
>
> 关于这个 repo「是什么、为什么存在、有哪些 bridges」 → 看 [README.md](./README.md)(英文)或 [README.zh-CN.md](./README.zh-CN.md)(简中)。
> 本文件聚焦 Claude 在这个 repo 工作时需要知道的**惯例与红旗**。

---

## 结构约定

```
openspec-schemas/                     ← 本 repo
├── README.md                         ← 英文,GitHub 默认 render
├── README.zh-CN.md                   ← 简中,有切换链接
├── CLAUDE.md                         ← 你正在读的(简中,给 Claude 看)
├── LICENSE                           ← MIT
├── .gitignore
├── .github/workflows/
│   ├── validate-schemas.yml          ← CI 对每个 bridge 跑 openspec schema validate
│   └── version-check.yml             ← 每周校验 upstream OpenSpec / Superpowers,落后就开 issue
├── docs/
│   ├── roadmap.md / .zh-CN.md        ← 公开 roadmap
│   └── superpowers/
│       ├── specs/                    ← 设计 spec(brainstorming 产出)
│       └── plans/                    ← 实现 plan(writing-plans 产出)
└── superpowers-bridge/                ← 第一个 bridge,自包式 schema bundle
    ├── README.md / .zh-CN.md         ← 完整 bridge 文档(含 install + integration runbook)
    ├── schema.yaml                   ← OpenSpec 读的 schema 定义
    └── templates/                    ← artifact 模板
        ├── brainstorm.md
        ├── proposal.md
        ├── design.md
        ├── spec.md
        ├── tasks.md
        ├── plan.md
        ├── verify.md
        └── retrospective.md
```

未来新增 bridge:在 repo 根加一个 `<new-bridge>/` 子目录,内含与 `superpowers-bridge/` 相同结构。CI matrix 在 `.github/workflows/validate-schemas.yml` 的 `matrix.bridge` 加一行即可。

## 命名约定

- **Repo / 目录 / schema name**:lowercase + hyphen + 对的单复数
  - repo:`openspec-schemas`(复数,可长多个 bridge)
  - bridge dir / schema name:`superpowers-bridge`(单数)
  - 不用 PascalCase(虽然 OpenSpec 自身 repo 用 `OpenSpec`,但他们的 CLI / npm package 都是 lowercase,我们对齐功能性命名)
- **Locale 编码**:用 `zh-CN`(简中)、`zh-TW`(繁中);避免裸写 `zh`

## 双语策略

| 文件类型 | 语言 |
|---------|------|
| 入口 `README.md` | 英文 canonical + `README.zh-CN.md` 翻译 + 顶部切换链接 |
| `CLAUDE.md`(这份) | 简中(给维护者 + Claude;国际读者从 README 入口进来) |
| `docs/roadmap.md` | 英文 canonical + `.zh-CN.md` 翻译 + 切换链接 |
| `superpowers-bridge/README.md` | 英文 canonical + `.zh-CN.md` 翻译 + 切换链接 |
| `schema.yaml`、`templates/*.md` | 英文(机器读 + 国际读者) |
| Commit message | 英文(国际惯例) |
| Code comment | 英文 |

**翻译同步原则**:英文 canonical,翻译版可能滞后。修改英文版时若 schema / 工作流发生实质变动,要同步更新简中版。小改动允许先英文后简中。

## Schema 修改流程

1. 编辑 `<bridge>/schema.yaml` 或 `<bridge>/templates/*.md`
2. 本地验证:
   ```bash
   mkdir -p /tmp/test-project/openspec/schemas
   cp -R <bridge>/ /tmp/test-project/openspec/schemas/
   cd /tmp/test-project
   openspec schema validate <bridge-name>
   openspec schemas
   ```
3. 若有时序错位 / 错误行为更动,**也要同步更新 `superpowers-bridge/README.md` 的「六个值得记住的设计触点」段**(尤其是 verify/retrospective 时序错位那段),并同步简中版。
4. Commit message 用英文,符合 conventional commits(`feat:`、`fix:`、`refactor:`、`chore:`、`docs:`、`ci:`)
5. push 触发 CI

## 三个 alfred-openspec 顾虑的应对(内化记忆)

PR #970 review 提出三个顾虑,本 schema 在 v1 已具体应对。Claude 在这个 repo 修任何 schema 行为前都要记住:

| 顾虑 | 应对 |
|------|------|
| #3 主动 commit 使用者 git | **完全移除**。Step 0 改为 skill PRECHECK,只验 skill 不动 git |
| #1 与 Superpowers 强耦合无 capability detection | **Layer 1**:每个 invoke skill 的 instruction 开头跑 PRECHECK,缺失就 STOP。**Layer 2**:对 verify / retrospective 加 evidence-based PRECHECK(`git log`、`grep` 检查可观察状态) |
| #2 verify 时序错位(以及 retrospective 同型) | 已知限制,在 bridge README 的「设计触点 #6」文档化。完整修法等 OpenSpec 引擎引入 `post_apply` phase。Layer 2 evidence-based PRECHECK 是当前缓解 |

**修 schema 时的红旗** —— 以下行为**不要做**(会违反 PR #970 的应对):

- ❌ 在 instruction 写「主动 git add / git commit」
- ❌ 拿掉某个 PRECHECK 但没换更强的替代品
- ❌ 把 verify / retrospective 从 artifact 拉掉但没在 README「设计触点」段同步更新限制
- ❌ 改 schema name 但没同步改 bridge 内所有文件 + 顶层 README 的 bridge 索引

## 相关链接

- 设计 spec:[`docs/superpowers/specs/2026-05-02-openspec-schemas-monorepo-design.md`](./docs/superpowers/specs/2026-05-02-openspec-schemas-monorepo-design.md)
- 实现 plan:[`docs/superpowers/plans/2026-05-02-phase-1-implementation.md`](./docs/superpowers/plans/2026-05-02-phase-1-implementation.md)
- PR #970 review:<https://github.com/Fission-AI/OpenSpec/pull/970>
- 既有 spec-kit superpowers bridges 参考:
  - [RbBtSn0w/spec-kit-extensions/superpowers-bridge](https://github.com/RbBtSn0w/spec-kit-extensions/tree/main/superpowers-bridge)
  - [WangX0111/superspec](https://github.com/WangX0111/superspec)
