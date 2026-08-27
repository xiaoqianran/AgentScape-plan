# ADR-0005 — Experiment-oriented Modular Monolith

**Status:** Accepted
**Date:** 2026-08-28

## Context

AgentScape 正在同时演进 Provider、Sidecar、Human Caller、Agent Caller、Asset 与 World。当前最大风险不是“文件太大”，而是过早按 Controller / Service / Repository / Manager / Factory 等技术层横向拆分，制造接口、协调代码、重复状态和远距离知识依赖。

本系统需要同时满足：

- 实验可以快速验证真实能力；
- production 迁移沿一个业务变化轴推进；
- 高内聚代码优先保持物理接近；
- 副作用边界清晰，可测试；
- 只有真实压力出现时才增加进程/服务/模块边界。

## Decision

采用以下统一实现策略：

```text
Experiment
    │
    ▼
Vertical Slice
    │
    ▼
Single-file First
    │
    ▼
Functional Core / Imperative Shell
    │
    ▼
Pressure Evidence?
   /              no             yes
  │               │
  ▼               ▼
keep together   Extract by Pressure
```

### 1. Experiment-oriented Modular Monolith

实验优先回答“这条能力真的可用吗”，production 则保持模块化单体作为默认部署形态。

```text
modal-lab experiment
       │ real evidence
       ▼
Architecture decision
       │
       ▼
Production Vertical Slice
       │
       ▼
Independent smoke / parity
```

实验代码不是 production runtime dependency；实验结果可以改变 Architecture Card、Contract 或 Migration Gate。

### 2. Vertical Slice

代码沿**变化轴**组织，一起变化的知识放一起；slice 之间低耦合。

优先：

```text
source_3d_asset.py
  plan candidates
  call image capability
  evaluate candidates
  call 3D capability
  publish asset
```

而不是先拆成：

```text
controllers/
services/
repositories/
managers/
factories/
```

只有相邻两部分提供不同抽象、隐藏不同知识时，边界才成立。

### 3. Single-file First

默认一个高内聚文件承载一个 slice。行数只能作为气味，不能作为拆分依据。

允许 300–500+ 行，只要：

- 单一 State Owner；
- 单一失败/重试生命周期；
- 单一部署生命周期；
- 共享大量领域知识；
- 测试矩阵一致；
- 物理接近明显降低理解成本。

### 4. Functional Core / Imperative Shell

纯决策放 Core，副作用推到 Shell。

```text
Functional Core
  validate
  normalize
  select
  rank
  transition
  compile
  decide retry

Imperative Shell
  Modal
  HTTP
  SQLite
  filesystem / Volume
  LLM / VLM
  subprocess
  WorldRuntime
```

Core 应尽量：输入值 → 输出值，无隐式 I/O；Shell 只负责读取世界、调用 Core、执行决定、保存结果。

### 5. Extract by Pressure

不按“看起来像服务”拆分，只按真实压力拆分。

允许 Extract 的证据包括：

```text
独立 State Owner
独立 failure/retry lifecycle
独立 GPU / deployment lifecycle
独立 scaling pressure
独立 security/trust boundary
独立测试矩阵
反复 merge conflict / 变化节奏明显分叉
profiling 证明的性能瓶颈
```

若没有压力证据，默认保持合并。

## Deep Module Rule

拆分必须隐藏显著知识。一个新模块/服务如果只是把参数原样转发给下一层，就是浅接口，应删除或合并。

```text
bad:
Controller → Service → Manager → Provider
(each layer mostly forwards arguments)

better:
source_3d_asset()
  ├─ pure planning / selection
  └─ thin adapters for real side effects
```

## Repository-specific Consequences

### `modal-lab`

一个实验一个变化边界；普通实验默认单 `app.py`，Provider/integration 验证默认单 `run.py`。只有独立领域 workflow 形成后才拆。

### `modal-3D`

`source-inputs → condition → worker` 是一个 Vertical Slice。旧 canonical fast path 不为“统一”被强制包进 conditioner。只有 conditioning 出现独立性能/部署压力后才继续提取。

### `modal-3D-client` / `modal-2D-client`

保持 Reference Sidecar 高内聚：transport + durable execution mirror + artifact verify/cache。不要增加 Manager/Repository 层，除非状态或失败生命周期真的分裂。

### `modal-inference-hub`

按 Human workflow slice 演进：Project / candidate selection / generate 2D / generate 3D。先在现有应用内形成清晰 Application Boundary，不先拆微服务。

### `AgentScape-agent`

按 Skill slice 组织：`source_3d_asset`、`build_world`。Agentic decision 与 deterministic workflow 可以同文件高内聚实现；只有 Run state、provider adapters 或某个 Skill 形成独立压力后再拆。

### `AgentScape`

Asset 与 World 是稳定领域模块；内部 compiler/runtime 继续按真实领域状态与生命周期拆，不按技术层重写。

## Rejected Alternatives

### Layer-first architecture

拒绝默认 Controller / Service / Repository / Manager 分层。它会让同一变化跨多个目录传播，并增加浅接口。

### Microservices first

拒绝把 Provider/Agent/Hub 内部 helper 提前服务化。分布式边界增加部署、重试、认证、观测和一致性成本。

### File-size-driven extraction

拒绝以 300/500/1000 行作为硬拆分阈值。

## Migration Rule

每个新切片在提交前回答：

```text
1. 变化轴是什么？
2. State Owner 是谁？
3. 哪些函数可以保持纯？
4. 哪些副作用必须留在 Shell？
5. 现有模块是否已经隐藏了足够知识？
6. 是否存在真实 Pressure 支持进一步拆分？
```

没有第 6 条证据时，不新增服务。

## References

- Jimmy Bogard — Vertical Slice Architecture：按 feature/变化轴组织，而不是先横向分层。
- John Ousterhout — *A Philosophy of Software Design*：优先 deep modules 与 information hiding；拆分本身也会制造复杂度。
- Functional Core / Imperative Shell：纯业务决策与 I/O 副作用分离。
- Monolith First / Strangler：先形成自然边界，再在有证据时物理提取。
