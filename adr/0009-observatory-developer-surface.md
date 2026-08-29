# ADR-0009 — Observatory Developer Product Surface

- Status: Accepted
- Date: 2026-08-29

## Context

AgentScape 已经拥有稳定的 Physics、Spatial、Navigation、Interaction 与 World Runtime，但长期仅靠自动 tests 很难回答以下工程问题：

- solver 当前真实 body/collider/joint/contact 状态是什么；
- Manifest collider 与实际 Physics collider 是否发生 drift；
- Rapier 与 Jolt 在同一 fixed-step scenario 下差异在哪里；
- Spatial raycast / overlap / support / free-space 当前到底返回什么；
- 一个 regression 是 Asset contract、Runtime adapter 还是 backend 的问题。

为此需要一个可以人工单步、可视化、固定步长 replay、对照 backend 的开发界面。但如果这个界面自行实现第二套 Physics/Spatial truth，反而会制造新的架构分叉。

## Decision

AgentScape 新增根级：

```text
observatory/
```

其身份是 **Developer Product Surface**，不是第七个业务 domain，也不是第二个 Runtime。

六个业务 product/domain root 仍然是：

```text
studio/
agent/
generation/
asset/
world/
core/
```

Observatory 与 Studio 平级作为入口 surface，但 ownership 方向不同：

```text
                    production runtime/domain
                         ▲            ▲
                         │            │
                  Studio│   Observatory
                    uses│      observes
                         │            │

production code ─────────────X──────► observatory
```

### 1. One-way dependency

Observatory 可以 import production Runtime/domain contract；任何 production module 都禁止反向 import `observatory/`。

### 2. No second implementation

Observatory 不拥有：

- 第二套 Physics engine wrapper；
- 第二套 SpatialSystem；
- 第二套 Asset/Manifest truth；
- 第二套 Navigation/Interaction/Agent execution semantics。

Scenario 必须驱动生产代码。

Synthetic geometry 可以用于隔离测试，但 production Manifest / Schema / Runtime contract 必须直接复用，不能复制一份实验版 contract。

### 3. Observation contract

Observatory 不允许穿透 solver-private world handle。生产 Runtime 提供 normalized observation contract：

```text
PhysicsSystem.debugSnapshot()
SpatialSystem.debugSnapshot()
```

Physics snapshot 可包含：

```text
bodies
colliders
joints
contacts
backend metrics
optional native debug geometry
```

Spatial snapshot 可包含：

```text
bounds
collision pairs
metrics
```

这些 snapshot 是开发观测面，不是新的业务 source of truth。

### 4. Backend comparison

Physics Lab 可以在相同 Scenario 与 fixed dt 下分别驱动 Rapier / Jolt，然后比较 normalized snapshot。

Backend-specific native geometry 是 optional observation；不同 backend 的 contact evidence strength 不要求完全相同。

### 5. Spatial/BVH ownership

`three-mesh-bvh` runtime setup 归 World/Spatial owner，而不是 Observatory 或 Asset domain。

```text
world/runtime/spatial/ThreeBvhRuntime.js
```

AssetManager 只加载/校验 Asset，不依赖“某个入口已经 patch Three prototype”的隐式前置条件。对象进入 WorldRuntime 时由 World/Spatial 明确准备 BVH bounds tree。

### 6. Entry / tests

Vite production build 同时拥有：

```text
/index.html
/observatory/index.html
```

Observatory tests 归：

```text
tests/observatory/
```

主 architecture validator 必须拒绝 production → Observatory 反向依赖。

## Initial scope

正式第一阶段包含：

```text
Physics Lab
├─ Rapier
├─ Jolt
├─ normalized backend comparison
├─ Gravity / Collision / Stack / Hinge
├─ production Cup / Cabinet truth comparison
└─ checkpoint / deterministic replay

Spatial Lab
├─ BVH Raycast
├─ Bounds / Overlap
├─ Support / FreeSpace
└─ deterministic query replay
```

未来 Navigation / Interaction / Agent Lab 应继续遵循同一 one-way observation principle。

## Consequences

- 工程师可以看到生产 Runtime 的真实执行状态，而不是额外实现一套 demo。
- Debug contract 成为可测试的生产 observation boundary。
- Rapier/Jolt 差异可被 normalized comparison 明确表达。
- Spatial/BVH 初始化 ownership 更清晰。
- 根目录增加 `observatory/`，但业务 domain model 不增加新的 truth owner。

## Verification

AgentScape commit `1bf17a6` 已验证：

```text
architecture validation      PASS
world viability              PASS
Observatory tests          4 files / 28 tests PASS
full JS tests            180 files / 858 tests PASS
production build             PASS
Python SDK tests             130 PASS
Asset Compiler tests           7 PASS
```
