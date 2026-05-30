# openspec-schemas

[English](./README.md) · [简体中文](./README.zh-CN.md)

Community-contributed [OpenSpec](https://github.com/Fission-AI/OpenSpec) schemas. Each schema is a self-contained bundle that you copy into your project's `openspec/schemas/` directory and select per-change with `--schema <name>`.

## Bridges in this repository

| Bridge | Purpose | Status |
|--------|---------|--------|
| [`superpowers-bridge`](./superpowers-bridge/) | Bridges OpenSpec's artifact governance with [obra/superpowers](https://github.com/obra/superpowers) execution skills (brainstorming, writing-plans, TDD-via-subagents, code review, finishing). Adds an evidence-first `retrospective` artifact filling a gap Superpowers does not natively cover. | v1 |

## Why a separate repository?

[OpenSpec PR #970](https://github.com/Fission-AI/OpenSpec/pull/970) originally proposed `sdd-plus-superpowers` as a built-in schema. After maintainer review, the integration moved to a community repository — same pattern as [github/spec-kit's community extension catalog](https://speckit-community.github.io/extensions/), which keeps third-party tool integrations out of core.

Benefits:
- OpenSpec core does not take on Superpowers' release cadence
- Bridge can iterate independently
- Other community schemas can join this repository as siblings

## Install

Each bridge directory has its own `README.md` with a copy-paste Claude Code prompt for one-shot installation, plus a manual bash alternative. See e.g. [`superpowers-bridge/README.md#install`](./superpowers-bridge/README.md#install).

## Roadmap

See [`docs/roadmap.md`](./docs/roadmap.md) for what's planned.

## License

MIT — see [LICENSE](./LICENSE).
