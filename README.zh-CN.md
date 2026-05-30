# openspec-schemas

[English](./README.md) · [简体中文](./README.zh-CN.md)

社区贡献的 [OpenSpec](https://github.com/Fission-AI/OpenSpec) schema 集合。每个 schema 都是一个自包式 bundle —— 把它复制到你项目的 `openspec/schemas/` 目录,然后用 `--schema <name>` 在每次 change 时单独选择。

## 本 repo 提供的 bridges

| Bridge | 用途 | 状态 |
|--------|------|------|
| [`superpowers-bridge`](./superpowers-bridge/) | 把 OpenSpec 的 artifact 治理流程与 [obra/superpowers](https://github.com/obra/superpowers) 的执行技能(brainstorming、writing-plans、TDD-via-subagents、code review、finishing)串接成一个工作流。额外加上 evidence-first 的 `retrospective` artifact,补上 Superpowers 没有的 retro 能力。 | v1 |

## 为什么要另开一个 repo?

[OpenSpec PR #970](https://github.com/Fission-AI/OpenSpec/pull/970) 原本提议把 `sdd-plus-superpowers` 收进 OpenSpec 内置 schema。维护者 review 后建议改成社区 repo —— 这跟 [github/spec-kit 的 community extension catalog](https://speckit-community.github.io/extensions/) 处理第三方工具整合的模式相同:让它们待在社区层,不进 core。

好处:

- OpenSpec core 不需要跟 Superpowers 的发布节奏绑在一起
- bridge 可以独立迭代、发版
- 其他社区 schema 之后可以以 sibling 的方式加入本 repo

## 安装

每个 bridge 子目录下都有自己的 `README.md`,附**复制粘贴到 Claude Code 一键安装**的 prompt,以及手动 bash 替代方案。例如 [`superpowers-bridge/README.md#install`](./superpowers-bridge/README.md#install)。

## Roadmap

未来规划见 [`docs/roadmap.md`](./docs/roadmap.md)。

## License

MIT —— 详见 [LICENSE](./LICENSE)。
