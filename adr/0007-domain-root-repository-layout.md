# ADR-0007 — Domain-root Repository Layout

- Status: Accepted
- Date: 2026-08-29

## Context

仓库合并后，`AgentScape/src/` 同时容纳 Agent、UI、Generation、Artifact、Asset、Compiler、World Runtime、Validation、Persistence 等大量平级技术目录。`src/` 不再表示一个单一 package 的 implementation root，而成为整个产品的总容器；`pipeline/`、`validation/`、`adapters/` 等技术名词又进一步隐藏 ownership。

对 Godot、VS Code、Zed、Bevy、AutoGen、Vercel AI SDK、Isaac Lab、Habitat-Lab 等大型开源仓库的目录设计进行对照后，决定优先让目录反映真实 domain/deployment/release boundary，而不是为了整齐提前引入 `packages/` 或抽象的 `foundation/interfaces/shared`。

## Decision

AgentScape 删除总 `src/`，根目录直接暴露稳定产品 subsystem：

```text
studio/
agent/
generation/
asset/
world/
core/
```

并保留拥有真实工程边界的：

```text
api/
services/
sdk/
tests/
tooling/
docs/
public/
```

具体 ownership：

- `studio`：Human app/editor/UI/local persistence；
- `agent`：Agent loop、LLM gateway、tools、skills、recovery；
- `generation`：Job、Artifact、Connector、Provider-facing orchestration，以及 Artifact→Asset bridge；
- `asset`：Asset truth、manifest、admission、adapter、compiler；
- `world`：World spec、compiler、runtime、verification、content；
- `core`：business-neutral primitives only。

`api/` 保留根级是 Vercel Functions deployment convention；私有 server helper 收敛到 `api/_server/`。`services/` 与 `sdk/` 分别保留独立 runtime/release identity。Repository engineering 收敛到 `tooling/`。

## Dependency constraints

1. `core` 不得依赖任何产品 domain。
2. Asset core 不得依赖 Studio、Agent、Generation 或 World。
3. World 可依赖窄 Asset contract，但不得依赖 Generation/Studio/Agent。
4. Provider implementation 始终在兄弟仓库 `modal-provider`；`generation/providers` 只保存 provider-neutral registry/snapshot semantics。
5. 禁止重新引入根级 `src` 以及技术型总目录作为 ownership shortcut。

## Package policy

当前不引入 `packages/`。只有某个 subsystem 出现真实独立 package API、发布周期、构建边界和 dependency graph 后，才允许通过后续 ADR package 化。

## Verification

AgentScape 的 `npm run architecture:validate` 对旧根目录回归和关键 domain dependency 做机械门禁；完整迁移必须同时通过 architecture validation、asset validation、全量 tests 与 production build。
