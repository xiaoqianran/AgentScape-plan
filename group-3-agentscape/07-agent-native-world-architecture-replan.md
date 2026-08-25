# AgentScape Agent-native World Architecture Replan

> **状态：AUTHORITATIVE FUTURE PLAN / 未来执行权威计划**
> **当前执行基线：AgentScape `main@0729970`（v1.34.2）**
> 本文从 2026-08-25 起取代 `01-AgentScape-integration-plan.md` 中 **AS-11～AS-19 的旧顺序**，并取代 `04-live-execution-map.md` 中旧的 Gate L6～L8 / Phase 5～6 未来排序。
> `04-live-execution-map.md` 仍然是 **当前 Git/测试/实现状态账本**；本文负责“接下来按什么架构继续做”。

---

## 1. 为什么必须重规划

旧计划形成于 AgentScape 主要任务仍是：

```text
Modal / EmbodiedGen
        ↓
Connector / Provider
        ↓
Async Job / Artifact
        ↓
Asset Compiler
        ↓
WorldSpec v2
```

这个阶段的任务是真实且必要的，但随着 `ProviderRegistry`、Connector session、async Job、Artifact Importer、verified asset pipeline、Agent-visible generation、Generation Job Center 已经进入 `main`，继续把未来主线写成：

```text
AS-10 UI
  ↓
AS-11 Planner
  ↓
AS-12 WorldSpec v2
  ↓
AS-13 Layout Adapter
  ↓
AS-14 Environment
  ↓
AS-15 Room
  ↓
AS-16 Persistence
  ↓
AS-17 Security
  ↓
AS-18 Observability
  ↓
AS-19 Tests
```

已经不再适合作为 AgentScape 的长期执行主轴。

主要问题有六个：

1. **Provider integration / Provider 接入被放得过重。** 它是重要支撑层，但不是 AgentScape 的最终产品边界。
2. **WorldSpec v2 太窄。** 最终需要的是 World IR / 世界中间表示，除了 asset/layout，还必须表达 capability、state、interaction、rule、acceptance、physics requirement、revision/provenance。
3. **Interaction & Rule Compiler / 交互与规则编译器没有被当作独立核心系统。** 现有能力散落在 Manifest、Skill、InteractionSystem、Policy 中。
4. **Physics 被写死成 Rapier 永久真值。** Rapier 是当前默认实现，不应该成为最终架构不可替换的 solver 绑定。
5. **Security / Observability / Testing 被排到最后。** 这些必须成为每个 Gate 的横向验收，不允许“功能都做完以后再补”。
6. **Environment / Room 被误当成核心架构阶段。** 它们应该是 World IR + Asset + Runtime + Verification 能力成熟后的内容/场景产品化，而不是驱动核心 contract 的源头。

因此，未来主轴改成：

```text
World Intent / 世界意图
        ↓
World IR / 世界中间表示
        ↓
┌───────────────────────────────┐
│ Asset Compilation / 资产编译 │
│ Behavior Compilation / 行为编译│
└──────────────┬────────────────┘
               ↓
World Runtime / 世界运行时
               ↓
Verification / 验证
               ↓
Repair + Revision / 修复与修订
```

---

## 2. 最终使命 / Mission

AgentScape 的最终产品定义：

> **把自然语言中的世界意图，编译成可生成、可交互、可执行、可验证的三维世界。**

它不是一个生成模型，也不是要重新发明渲染器、刚体 solver 或机器人仿真器。

AgentScape 拥有的是：

> **World Compilation Authority / 世界编译权。**

AI 可以：

- 理解；
- 规划；
- 推断语义；
- 生成 proposal；
- 提议规则；
- 提议修复。

确定性系统必须拥有：

- state mutation / 状态修改；
- collision / 碰撞；
- physics / 物理；
- navigation / 导航；
- execution / 执行；
- verification / 验证；
- transaction / rollback / 事务与回滚。

核心原则不变：

> **Inference is not truth; execution evidence is truth. / 推断不是事实，执行证据才是事实。**

---

## 3. 目标系统总图 / Target Architecture

```text
                                      User / 用户
                                          │
                                  Intent / 世界意图
                                          ▼
                              ┌─────────────────────┐
                              │ World Planner       │
                              │ 世界规划器          │
                              └──────────┬──────────┘
                                         │
                                  World IR / 世界 IR
                                         │
                ┌────────────────────────┼────────────────────────┐
                │                        │                        │
                ▼                        ▼                        ▼
      ┌──────────────────┐    ┌────────────────────┐    ┌──────────────────┐
      │ Asset Compiler   │    │ Interaction & Rule │    │ Support Plane    │
      │ 资产编译器       │    │ 交互与规则编译器   │    │ Provider 等支撑  │
      └────────┬─────────┘    └──────────┬─────────┘    └────────┬─────────┘
               │                         │                       │
               └─────────────────────────┼───────────────────────┘
                                         ▼
                               ┌─────────────────────┐
                               │ World Compiler      │
                               │ 世界编译链          │
                               └──────────┬──────────┘
                                          │
                                          ▼
                               ┌─────────────────────┐
                 Human / 人类 ─► WorldRuntime        │◄─ Agent / 智能体
                               │ 世界运行时          │
                               └──────────┬──────────┘
                                          │
       ┌──────────────────────────────────┼─────────────────────────────────┐
       │                                  │                                 │
       ▼                                  ▼                                 ▼
Physics Capability / 物理能力      Navigation / 导航             Interaction / 交互
       │                                  │                                 │
       └──────────────────────────────────┼─────────────────────────────────┘
                                          ▼
                               ┌─────────────────────┐
                               │ Verification        │
                               │ 验证与判定          │
                               └──────────┬──────────┘
                                          │
                               ┌──────────┴──────────┐
                               ▼                     ▼
                       VERIFIED / 已验证        FAILED / 失败
                                                     │
                                                     ▼
                                             Repair / 修复
                                                     │
                                                     ▼
                                         Constrained Revision
                                             受约束 IR 修订
                                                     │
                                                     └────► World IR
```

---

## 4. 五大核心系统 / Five Core Systems

后续架构、ownership、Gate 和测试都围绕五大核心展开。

### Core 1 — World Planner & World IR / 世界规划器与世界 IR

负责：

- natural-language intent → typed World IR；
- entity / spatial / physics / capability / state / interaction / rule / acceptance；
- revision identity；
- provenance；
- planner policy；
- constrained revision；
- canonical compilation input。

不负责：

- 直接写 Runtime state；
- 自己实现碰撞/布局物理；
- 直接调用 provider 私有函数名；
- 把 proposal 当 verified world。

### Core 2 — Physical-Semantic Asset Compiler / 物理语义资产编译器

负责：

- raw asset → geometry normalization；
- part segmentation；
- semantic evidence；
- joint/collider/material proposal；
- capability evidence；
- executable Manifest；
- Asset Admission。

不负责：

- 把 provider 标签直接提升成 truth；
- 世界规则；
- 任务完成判定。

### Core 3 — Interaction & Rule Compiler / 交互与规则编译器

负责：

- capability；
- precondition；
- effect；
- state transition；
- event / condition / effect rule；
- verifier binding；
- executable interaction graph。

不负责：

- 任意脚本执行；
- 绕过 Runtime mutation；
- 直接控制 solver；
- 自己宣布动作成功。

### Core 4 — World Runtime / 世界运行时

负责：

- live world state；
- object lifecycle；
- rendering projection；
- physics backend ownership；
- spatial/navigation；
- locomotion；
- action execution；
- mutation atomicity；
- shared Human + Agent runtime。

### Core 5 — Verification & Repair / 验证与修复

负责：

- geometry verification；
- physics verification；
- interaction verification；
- rule verification；
- task/world acceptance；
- failure attribution；
- recovery proposal；
- bounded local repair；
- re-verification。

---

## 5. World IR / 世界中间表示重新定义

旧 `WorldSpec v2` 主要围绕 asset/layout/environment 展开。新计划把它升级为 **World IR vNext**。

```text
World IR / 世界中间表示
│
├─ metadata / 元数据
│  ├─ schemaVersion
│  ├─ revisionId
│  ├─ parentRevisionId
│  └─ provenance
│
├─ intent / 世界意图
│  ├─ description
│  ├─ task
│  └─ acceptance
│
├─ entities / 实体
│  ├─ identity
│  ├─ assetRef / assetRequest
│  ├─ role / semantic tags
│  ├─ physicsRequirement
│  ├─ capabilities
│  └─ initialState
│
├─ spatial / 空间
│  ├─ pose proposals
│  ├─ bounds
│  ├─ relations
│  └─ global constraints
│
├─ interactions / 交互
│  ├─ capability bindings
│  ├─ preconditions
│  ├─ effects
│  └─ verification target
│
├─ rules / 世界规则
│  ├─ event
│  ├─ condition
│  └─ effect
│
├─ environment / 环境
│  ├─ built-in or asset-backed
│  └─ navigation / spawn hints
│
└─ policy / 策略
   ├─ generation policy
   ├─ cost policy
   ├─ fallback policy
   └─ physics quality policy
```

### 5.1 IR 与 Runtime Truth 的边界

```text
World IR / 世界 IR
    │ proposal / 编译输入
    ▼
Compiler / 编译器
    │
    ▼
Runtime / 运行时事实
    │
    ▼
Observation / 观测
    │
    ▼
Verification / 验证
```

World IR 不能反向覆盖 Runtime truth。

### 5.2 IR Revision / 修订是第一等公民

旧的 `WorldRetry` 只解决 missing asset 的有界 retry；未来必须支持 finding 驱动的受约束修订。

```text
World Revision N / 世界修订 N
        │
        ▼
Compile + Execute + Verify / 编译执行验证
        │
        ▼
Finding / 问题证据
        │
        ▼
Revision Proposal N+1 / 修订提议 N+1
        │
        ├─ changed nodes / 变更节点
        ├─ evidence refs / 证据引用
        ├─ reason / 原因
        └─ bounded scope / 有界范围
        │
        ▼
Changed-plan Gate / 变更计划门
        │
        ▼
Canonical Recompile / 标准重编译
```

禁止：finding handler 直接 patch live scene 作为永久修复。

---

## 6. Physics Capability Layer / 可替换物理能力层

这是本次重规划新增的长期架构边界。

### 6.1 当前与未来

当前：

```text
AgentScape
    │
    ▼
PhysicsSystem
    │
    ▼
Rapier
```

目标：

```text
PhysicsRequirement / 物理需求
        │
        ▼
Physics Capability Router / 物理能力路由器
        │
        ├─────────────────┬─────────────────┐
        ▼                 ▼                 ▼
Rapier Adapter      Genesis Adapter     PhysX / Other Adapter
Rapier 适配器       Genesis 适配器      PhysX / 其他适配器
        │                 │                 │
Rigid / Joint       Soft / Cloth         Advanced / Native
刚体 / 关节         柔体 / 布料候选      高级仿真候选
```

Genesis / PhysX 在此是 **candidate backend / 候选后端类别**，不是当前已承诺的生产依赖。是否成为实时后端，必须由部署、性能、状态同步、授权/构建、验证能力等后续 Gate 决定。

### 6.2 AgentScape 不发明 Solver

AgentScape 负责：

- 描述 PhysicsRequirement；
- 发现 backend capabilities；
- 按 policy 选择后端；
- 把 Entity/Manifest 编译成 backend-neutral command；
- 管理 authority scope；
- snapshot / restore / rollback；
- 统一 observation；
- 把结果交给 Verification。

AgentScape 不负责重新实现：

- 刚体积分器；
- 布料 solver；
- FEM；
- 高级机器人动力学 solver。

### 6.3 PhysicsRequirement Contract

World IR 应表达能力，不应写死 engine 名：

```text
PhysicsRequirement
│
├─ bodyClass
│  ├─ rigid / 刚体
│  ├─ articulated / 关节体
│  ├─ soft / 柔体
│  └─ cloth / 布料
│
├─ requiredCapabilities
│  ├─ collision
│  ├─ contacts
│  ├─ joint-limit
│  ├─ impulse/force
│  ├─ snapshot-restore
│  ├─ scene-query
│  └─ counterfactual-query
│
├─ executionMode
│  ├─ realtime / 实时
│  └─ validation-only / 仅验证
│
└─ qualityPolicy
   ├─ realtime-required
   ├─ deterministic-required
   └─ fallback-policy
```

### 6.4 PhysicsBackend Contract

目标接口概念：

```text
PhysicsBackend
│
├─ identity() / 后端身份
├─ capabilities() / 能力声明
├─ initDomain() / 初始化物理域
├─ attachEntity() / 接入实体
├─ detachEntity() / 移除实体
├─ execute(command) / 执行物理命令
├─ step(dt) / 推进一步
├─ query() / 场景与接触查询
├─ observe() / 读取权威状态
├─ snapshot() / 快照
├─ restore() / 恢复
└─ dispose() / 清理
```

具体方法名不在本计划冻结；冻结的是能力和 truth boundary。

### 6.5 Authority Scope / 真值范围

多后端存在时，必须先明确“谁对哪一块物理事实负责”。

```text
WorldRuntime
    │
    ▼
Physics Capability Router
    │
    ├─ Physics Domain A / 物理域 A
    │      └─ Rapier Adapter = authoritative / 权威
    │
    └─ Physics Domain B / 物理域 B
           └─ Soft-body Adapter = authoritative / 权威
```

规则：

- 一个物理域同一时刻只有一个 authoritative backend；
- validation-only backend 只产生独立 evidence，不自动接管 live truth；
- Renderer 不能反向成为 physics truth；
- backend switch 必须有 snapshot/import/admission/verification；
- 不满足能力时 fail-closed，不静默伪装支持。

### 6.6 Cross-backend Coupling / 跨后端耦合

```text
Rigid Domain / 刚体域                  Soft Domain / 柔体域
       │                                      │
    Rapier                                 Backend B
       │                                      │
       └────── Coupling Adapter / 耦合适配 ───┘
                         │
                         ▼
               contact / force exchange
                 接触 / 力交换契约
```

在 coupling contract 通过验证之前：

- 不允许声称跨 backend 的双向物理是真实已验证；
- 可以强制同一 coupling group 进入同一 backend；
- 可以把高精度 backend 限定为 validation-only；
- 可以显式使用 proxy approximation，但状态必须是 provisional，不是 verified。

### 6.7 Runtime Backend 与 Validation Backend 分离

```text
Physics Capability / 物理能力
        │
        ├───────────────┬─────────────────┐
        ▼               ▼                 ▼
Realtime Runtime    Validation-only    Offline Experiment
实时运行后端        仅验证后端          离线实验后端
        │               │                 │
 live truth         evidence only       benchmark/proposal
 实时真值           只产证据            基准/提议
```

这允许 AgentScape 使用高精度 solver 做验证，而不强迫它承担浏览器逐帧运行。

### 6.8 Physics 抽象迁移策略

不能一边抽象 backend，一边重写全部 Physics 行为。

```text
PHY-0 Current / 当前
WorldRuntime → PhysicsSystem → Rapier

PHY-1 Interface Parity / 接口等价
WorldRuntime → PhysicsBackend Contract → RapierAdapter
             业务行为必须等价

PHY-2 Capability Registry / 能力注册
PhysicsRequirement → Router → RapierAdapter

PHY-3 Validation Backend / 验证后端
Router ─► Rapier realtime
       └► high-fidelity validator

PHY-4 Multi-backend / 多后端
只有 coupling/snapshot/verification Gate 全通过后进入
```

### 6.9 当前耦合债务

当前代码中 Rapier-specific assumptions 不只在 `PhysicsSystem`：Agent recovery、Navigation、测试与部分 content/runtime 路径也直接或间接依赖 Rapier 类型/行为。

因此 `PHY-1` 的 DoD 不是“新增一个接口文件”，而是：

- 上层核心模块不 import Rapier；
- Rapier-specific shape/joint/body type 只存在于 adapter 内或显式 compatibility boundary；
- Navigation/Recovery 只消费 backend-neutral query/evidence；
- 当前 physics/articulation/recovery regression 全部保持行为等价。

---

## 7. Support Plane / 支撑层重新定位

Provider / Connector / Job / Artifact、Editor、Persistence、Environment 不消失，而是从“未来主轴”调整为支撑五大核心。

```text
┌───────────────────────────────────────────────────────────────────┐
│ Support Plane / 支撑层                                            │
├───────────────────────────────────────────────────────────────────┤
│ Provider / Connector / Generation Job / Artifact                 │
│ 外部 Provider / Connector / 生成任务 / Artifact                  │
│                                                                   │
│ Editor / Persistence / History / Content                         │
│ 编辑器 / 持久化 / 历史 / 内容                                    │
│                                                                   │
│ Policy / Security / Observability                                │
│ 策略 / 安全 / 观测                                                │
└─────────────────────────────┬─────────────────────────────────────┘
                              │ serves / 服务
                              ▼
                    Five Core Systems / 五大核心
```

### 7.1 Provider 链已经不再是主架构 blocker

当前 `main@7bbe4b2` 已有：

- Provider Registry；
- Connector scoped session；
- capability snapshot；
- async Generation Job projection/reconcile；
- Artifact identity/import integrity；
- verified artifact → Compiler → Admission；
- Agent-visible generation；
- Generation Job Center core。

剩余 Provider 工作属于产品化/扩展：

- real Connector process E2E；
- richer model/workflow/optionsSchema metadata；
- AS-10B schema-driven UX；
- multi-artifact selection；
- cost classes / confirmation policy；
- upstream provider capability expansion。

这些可以与 World IR / Behavior / Runtime correctness 并行，不再阻塞所有核心架构工作。

### 7.2 双生成策略重新定位

`modal-2d→3D` 与 `EmbodiedGen Text→3D` 是 **Asset Sourcing Strategy / 资产来源策略**。

它们仍然需要：

- 显式选择；
- lineage；
- cost policy；
- no silent double-charge；
- same Compiler/Admission truth。

但它们不决定 World IR、Interaction、Physics、Verification 的架构。

---

## 8. Cross-cutting Invariants / 横向永久约束

以下内容不再排在“后期 hardening”：每个 Gate 都必须满足。

### X-SEC Security / 安全

- secret 不进入 browser persistence / scene / trace；
- provider output 视为不可信；
- artifact bytes/hash/size/MIME fail-closed；
- arbitrary JS / skill injection 禁止；
- external side effect 有 policy scope；
- backend/provider capability 不可自我提升权限。

### X-OBS Observability / 可观测

任何失败必须能区分：

```text
planner
→ IR normalization
→ asset resolution
→ provider/job
→ artifact import
→ compiler/admission
→ behavior compile
→ physics/runtime
→ navigation
→ verifier
→ repair/revision
```

### X-TEST Testing / 测试

每个 contract 必须至少覆盖：

- positive path；
- malformed input；
- stale version；
- partial failure；
- rollback；
- restart/recovery（适用时）；
- truth boundary regression。

### X-COMPAT Compatibility / 兼容

- 现有 WorldSpec/Manifest/Scene 不能被一次性破坏；
- 新 schema 需要 normalize/migrate；
- backend abstraction 第一阶段必须 zero intentional behavior change / 不主动改变现有行为。

---

## 9. 多 AI Ownership / 并行开发所有权

推荐 7 个领域 AI + 1 个 Integration Guardian。

```text
                              ┌─────────────────────────────┐
                              │ AI-8 Integration Guardian  │
                              │ 集成 / 契约 / 架构守门     │
                              └──────────────┬──────────────┘
                                             │
        ┌──────────────────┬─────────────────┼──────────────────┐
        ▼                  ▼                 ▼                  ▼
┌──────────────┐   ┌──────────────┐  ┌──────────────┐   ┌──────────────┐
│ AI-1 IR      │   │ AI-2 Assets  │  │ AI-3 Behavior│   │ AI-4 Runtime │
│ 世界IR/规划  │   │ 资产编译     │  │ 交互/规则    │   │ 运行时/物理  │
└──────┬───────┘   └──────┬───────┘  └──────┬───────┘   └──────┬───────┘
       │                  │                 │                  │
       └──────────────────┴─────────┬───────┴──────────────────┘
                                    ▼
                           ┌──────────────────┐
                           │ AI-5 Verification│
                           │ 验证 / 修复      │
                           └────────┬─────────┘
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                ┌────────────────┐    ┌────────────────────┐
                │ AI-6 Provider  │    │ AI-7 Human/Content │
                │ 生成支撑       │    │ 编辑/持久化/内容   │
                └────────────────┘    └────────────────────┘
```

### AI-1 — World IR & Planner / 世界 IR 与规划

Single-owner contracts：

- World IR schema；
- WorldSpec compatibility normalization；
- revision/provenance；
- planner output contract。

### AI-2 — Asset Compiler / 资产编译

Single-owner contracts：

- Asset Manifest；
- part/joint/collider evidence；
- Asset Admission；
- executable promotion。

### AI-3 — Interaction & Rules / 交互与规则

Single-owner contracts：

- capability schema；
- precondition/effect；
- state transition；
- interaction graph；
- rule graph。

### AI-4 — Runtime & Physics / 运行时与物理

Single-owner contracts：

- WorldRuntime mutation；
- PhysicsBackend contract；
- Physics Capability Router；
- physics authority scope；
- spatial/navigation/runtime execution boundary。

### AI-5 — Verification & Repair / 验证与修复

Single-owner contracts：

- evidence/finding；
- verifier semantics；
- world acceptance；
- recovery eligibility；
- repair proposal contract。

### AI-6 — Provider / Generation Support / Provider 与生成支撑

Single-owner contracts：

- Connector protocol；
- provider capability；
- Generation Job；
- Artifact transport；
- Job Center provider schema UX。

### AI-7 — Editor / Persistence / Content / 编辑器、持久化与内容

Single-owner contracts：

- SceneSerializer；
- history/undo-redo integration；
- editor UX；
- environments/world packs；
- demos/benchmarks。

### AI-8 — Integration Guardian / 集成守门

负责：

- contract compatibility；
- cross-domain tests；
- ownership collision detection；
- merge sequencing；
- architecture invariants；
- 禁止新建第二份 Truth。

---

## 10. 新 Gate 顺序 / New Execution Gates

旧 AS 编号保留历史意义，但不再作为未来线性队列。

### Gate G0 — Runtime Truth & Baseline Freeze / Runtime 真值与基线冻结

目标：先保证后续所有抽象建立在可靠地基上。

必须完成：

- 当前 `main` baseline 可重现；
- `WorldRuntime.mutate()` atomicity 明确并有 regression；
- partial mutation throw 可 restore before snapshot；
- no duplicate truth ownership；
- 现有 Runtime/Physics/Interaction tests 可在安装依赖的干净环境复现；
- architecture ownership 文件/契约表冻结。

**状态：COMPLETE。** `R-ATOMIC-01` 已由 `3a956dc` 闭合；`npm ci` 后 `145 files / 630 tests PASS`，production build PASS。下一核心 Gate 为 G1 / IR-01。

### Gate G1 — World IR vNext Contract / 世界 IR 契约

目标：先定义语言，再让多个系统同时消费。

必须完成：

- schemaVersion / revision identity；
- entity / asset ref；
- spatial relation / constraints；
- PhysicsRequirement；
- capability intent；
- initial state；
- interaction/rule intent；
- acceptance criteria；
- provenance；
- v1/vCurrent normalize/migration；
- schema/serializer tests。

此 Gate **不要求一次实现 Planner LLM**。

### Gate G2A — Physics Interface Parity / 物理接口等价抽象

目标：把 Rapier 从永久架构绑定降为首个 backend，不改变现有行为。

必须完成：

- PhysicsBackend contract；
- RapierAdapter；
- WorldRuntime 依赖 contract，不直接依赖 Rapier 实现；
- Recovery/Navigation 不依赖 Rapier-specific API；
- snapshot/restore/query semantics；
- physics/articulation/recovery parity tests。

禁止：

- 在 G2A 同时引入 Genesis/PhysX；
- 为了“泛化”删除 Rapier 已验证能力；
- 抽象后测试下降。

### Gate G2B — Interaction & Rule Contract / 交互与规则契约

可与 G2A 并行。

已完成第一层 contract：

- capability；
- precondition；
- effect；
- state transition；
- verifier target；
- Runtime articulation request contract；
- no arbitrary JS execution。

下一层 G3 才负责 interaction intent → executable Runtime command，并加入 rule graph 的真实运行语义。

### Gate G3 — Executable Behavior Vertical Slice / 可执行行为纵向切片

目标：至少一组实体行为完全经过新 contract。

首批建议：

```text
Door / 门
  OPEN / CLOSE

Container / 容器
  PICKUP / PLACE

Switch / 开关
  SWITCH_ON / SWITCH_OFF
```

验收：

```text
World IR behavior intent
        ↓
Behavior Compiler
        ↓
Runtime command
        ↓
Physics/State execution
        ↓
Verifier
        ↓
VERIFIED / FAILED
```

### Gate G4 — Planner + Canonical World Compilation / 世界规划与标准编译闭环

这是旧 AS-11/12 的真正替代。

必须完成：

- prompt → strict World IR proposal；
- reuse-first；
- async missing asset resolution；
- Asset Admission；
- Behavior Compilation；
- deterministic composition；
- PhysicsRequirement admission；
- Runtime instantiate；
- world validation；
- rejected rollback；
- finding → constrained IR revision；
- changed-plan Gate；
- bounded recompile。

### Gate G5 — Semantic Asset Automation / 语义资产自动化

与 G3/G4 部分并行，但 executable promotion 必须服从核心 contract。

优先链：

```text
Segmentation Evidence / 分割证据
        ↓
Semantic Evidence / 语义证据
        ↓
Joint Proposal / 关节提议
        ↓
Physics Requirement Proposal / 物理需求提议
        ↓
Capability Proposal / 能力提议
        ↓
Compiler Verification / 编译验证
        ↓
Executable Promotion / 可执行提升
```

### Gate G6 — World-level Acceptance & Local Repair / 世界级验收与局部修复

目标：不仅验证单动作，还验证“用户要求的世界整体成立”。

```text
Geometry / 几何
 + Physics / 物理
 + Navigation / 导航
 + Interaction / 交互
 + Rules / 规则
 + Task Acceptance / 任务验收
             ↓
World Verdict / 世界判定
```

失败：

```text
Finding
  ↓
Root Cause
  ↓
Affected IR subgraph
  ↓
Bounded Revision
  ↓
Incremental or bounded Recompile
  ↓
Re-verify
```

### Gate G7 — Multi-backend Physics / 多物理后端

只有 G2A/G6 稳定后进入。

分两步：

**G7A Validation Backend**

- 接入一个高精度或不同物理能力的 validation-only backend；
- capability discovery；
- import/export snapshot/evidence；
- 不接管 live Runtime；
- verifier 能对比证据。

**G7B Runtime Backend / Coupling**

只有出现真实需求且具备：

- realtime latency；
- state sync；
- snapshot/restore；
- query contract；
- failure semantics；
- coupling contract；
- parity/acceptance benchmark；

才允许新增第二个 live authoritative backend。

### Gate G8 — Scale, Environment, Persistence, Multi-Agent / 规模化

在核心编译链稳定后扩展：

- environment / room bundle；
- large-world streaming；
- dynamic nav rebuild；
- offline restore；
- richer grasp/IK；
- soft-body world entities；
- multi-agent navigation/coordination；
- benchmark/world packs。

Environment/Room 现在属于 G8 产品化，不再反过来决定 Core Contract。

---

## 11. Support Gates / 支撑线 Gate

核心 Gate 之外，Provider/产品线继续并行。

### Support S1 — Real Connector Product E2E

当前 AS-10A 已 merged。下一项不是无条件写 AS-10B，而是先验证真实产品链：

```text
Browser Job Center
   ↓ pair
Real Connector Process
   ↓ capability
Provider
   ↓ submit/restart/reconcile
Generation Job
   ↓ artifact
Importer
   ↓ compile
Asset Admission
```

必须证明：

- restart 不重复计费；
- local Job identity 稳定；
- event cursor/idempotency 正确；
- artifact hash/lineage 正确；
- provider success != asset-ready。

### Support S2 — Schema-driven Job UX

只有 Provider capability 真实提供：

- model metadata；
- workflow metadata；
- optionsSchema/profile；
- cost class；

才进入 AS-10B。

UI 不得自建 provider 私有 catalog。

### Support S3 — Provider Evidence Expansion

EmbodiedGen Layout、Room、Affordance、Grasp 等继续作为：

```text
Provider Evidence / Provider 证据
        ↓
Adapter
        ↓
World IR / Asset Compiler proposal
        ↓
AgentScape verification
```

不再直接定义 AgentScape Core schema。

---

## 12. 旧 AS-00～AS-19 如何迁移

| 旧任务 | 新定位 | 状态/处理 |
|---|---|---|
| AS-00 Runtime baseline | G0 | 保留历史；Runtime correctness 继续 |
| AS-01 Provider Registry | Support Plane | 已 merged foundation |
| AS-02 Connector | Support S1 | 已 merged foundation |
| AS-03 Async Job | Support S1 | 已 merged foundation |
| AS-04 Artifact Importer | Support Plane | 已 merged foundation |
| AS-05/05A single asset / dual source | Support + Asset Compiler | 不再作为核心未来 Gate |
| AS-06/07/08 Evidence/Admission | Core 2 | 保留 contract，继续 evidence-first |
| AS-09 Agent generation skills | Support + Agent UX | 已 merged |
| AS-10 Job Center | Support S1/S2 | 10A merged；10B 条件式 |
| AS-11 Prompt→WorldSpec Planner | G4 | **被新 Planner + World IR 编译闭环取代** |
| AS-12 WorldSpec v2 | G1 | **被 World IR vNext 取代** |
| AS-13 EmbodiedGen Layout | Support S3 | 变成 provider adapter，不再是核心 Gate |
| AS-14 Background/Environment | G8 / Support Content | 后置产品化 |
| AS-15 Room/House | G8 / Support Content | 后置产品化 |
| AS-16 Persistence | AI-7 + G8 | support capability，不再等到最后才考虑 schema compatibility |
| AS-17 Security | X-SEC | **横向永久 Gate，不是后期任务** |
| AS-18 Observability | X-OBS | **横向永久 Gate** |
| AS-19 Testing | X-TEST | **每个 Gate 的 DoD** |

---

## 13. 立即可执行的第一批任务 / Immediate Work Slices

以下任务按照新架构可以并行，但必须遵守 ownership。

### R-ATOMIC-01 — WorldRuntime Mutation Atomicity

**状态：MERGED — `3a956dc` / `main@7bbe4b2`。**

Owner：AI-4 Runtime。

目标：

```text
before snapshot
    ↓
operation partially mutates
    ↓
throws
    ↓
restore(before)
    ↓
no undo entry
mutation owner cleared
runtime == before
```

DoD：最小 reproducer + regression + no caller workaround。**已完成**：partial throw rollback、rollback failure fail-closed、snapshot failure unlock；full suite 601 tests PASS。

### IR-01 — World IR Contract RFC + Compatibility Normalizer

**状态：MERGED — `281e02c` / `main@ca8cab7`.**

Owner：AI-1。

目标：只定义 contract 和 normalize/migrate，不同时写完整 Planner。

最小字段：

- schema/revision/provenance；
- entities；
- spatial；
- PhysicsRequirement；
- capability intent；
- state；
- interaction/rule intent；
- acceptance；
- policy。

### PHY-01 — PhysicsBackend Contract + Rapier Parity Adapter

Owner：AI-4。

前置已满足：R-ATOMIC-01 merged。下一步先做 dependency audit/contract draft，再做 Rapier parity adapter。

DoD：

- Rapier-specific import 收敛；
- current PhysicsSystem behavior parity；
- Navigation/Recovery 消费 backend-neutral queries；
- no Genesis/PhysX implementation in this slice。

### BEH-01 — Capability / State / Rule Contract

Owner：AI-3。

目标：把现有 Manifest actions + Skill preflight + Interaction semantics 整理成 typed contract。

DoD：至少 OPEN/CLOSE + PICKUP/PLACE 能表示 precondition/effect/verifier target。

### VER-01 — Unified Finding + Acceptance Contract

Owner：AI-5。

目标：统一 action finding 与 world finding 的核心 envelope：

- category；
- evidence；
- target IR/runtime identity；
- severity；
- retriable；
- repairable；
- stale-evidence identity；
- acceptance linkage。

### GEN-01 — Real Connector Product E2E

Owner：AI-6。

这是当前 Provider 线最重要的产品验证，独立于 IR/Runtime 主线。

### UX-01 — Mission/IR observability surface

Owner：AI-7。

不是重做 UI；只定义未来 World IR revision、finding、verification 的可视化需求，避免核心 contract 做完后 UI 无法解释。

### INT-01 — Architecture Contract Test Matrix

Owner：AI-8。

建立跨域矩阵：

```text
World IR
  × Asset Manifest
  × Behavior Contract
  × PhysicsBackend
  × Verification Finding
  × Persistence
```

每次 schema change 都检查 compatibility。

---

## 14. 第一批并行关系 / Parallelization

```text
                     AI-8 INT-01 / 集成契约矩阵
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
     AI-1 IR-01         AI-3 BEH-01       AI-5 VER-01
             │                │                │
             └────────────┬───┴───────┬────────┘
                          │           │
                          ▼           ▼
                     G1 Contract   G2B Contract
                          │
                          │
AI-4 R-ATOMIC-01 ──► AI-4 PHY-01
      Runtime 真值         物理接口等价
                          │
                          └───────────────┐
                                          ▼
                               G3 Behavior Vertical Slice

AI-6 GEN-01 ──────────────────────────────► Support S1
Provider 产品线                             独立并行

AI-7 UX-01 ───────────────────────────────► observability requirements
```

### 并发禁止

- `WorldRuntime` mutation contract 与 PHY-01 同一高风险区域由 AI-4 串行管理；
- AI-1 不直接修改 PhysicsSystem；
- AI-3 不直接新增 Runtime side-channel state；
- AI-5 不直接 patch IR/Runtime；
- AI-6 不改 core truth contract；
- AI-7 不在 UI 保存第二份可执行 world state。

---

## 15. 新的完成定义 / Definition of Done

AgentScape 的长期 DoD 不再是“接通所有 Provider”。

最终 North-star：

```text
User / 用户：
“生成一个废弃实验室。
柜子可以打开，里面有一个红色盒子。
让 Agent 把盒子拿出来放到桌上。”

Natural Language / 自然语言
        ↓
World IR / 世界 IR
        ↓
Asset Resolve / Generate / 资产解析或生成
        ↓
Asset Compile / 资产编译
        ↓
Behavior Compile / 行为编译
        ↓
Physics Admission / 物理能力准入
        ↓
World Compose / 世界编排
        ↓
Runtime / 真实运行
        ↓
Navigate / 导航
        ↓
Open / 打开
        ↓
Verify / 验证
        ↓
Pickup / 拿起
        ↓
Verify / 验证
        ↓
Place / 放置
        ↓
Settle / 物理稳定
        ↓
World Acceptance / 世界级验收
        ↓
VERIFIED TASK COMPLETE / 任务真实验证完成
```

关键要求：

- Planner 没有 Runtime truth 权；
- Provider 没有 Asset/World truth 权；
- Interaction 必须编译，不执行任意脚本；
- physics backend 可替换，但 authority 明确；
- validation-only backend 不冒充 live authority；
- failure 能归因；
- repair 是 bounded revision，不是无边界重生成；
- Human/Agent 共用一个 Runtime；
- 最终“成功”来自 execution evidence。

---

## 16. 计划文档权威关系 / Document Authority

从本次重规划以后：

```text
07-agent-native-world-architecture-replan.md
  └─ 未来架构 / Gate / ownership / 长期执行顺序

04-live-execution-map.md
  └─ 当前 HEAD / branch / commit / tests / 当前可执行 slice

02-provider-artifact-world-contract.md
  └─ Provider / Job / Artifact / Admission 兼容契约

03-dual-generation-strategy-plan.md
  └─ Asset sourcing strategy / 资产来源策略

06-embodiedgen-evidence-bridge-execution.md
  └─ 已完成/延续的 Provider evidence 专项记录

01-AgentScape-integration-plan.md
  └─ 历史 AS-00～AS-19 分解；AS-11～19 不再作为未来执行顺序
```

更新规则：

- 核心架构变化 → 更新 `07`；
- 当前实现状态 → 更新 `04`；
- Provider transport contract → 更新 `02`；
- 资产生成策略 → 更新 `03`；
- 不把未来愿景写成当前已实现事实。
