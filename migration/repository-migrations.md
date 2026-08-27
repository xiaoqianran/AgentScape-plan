# Repository Migrations

每个仓库使用 **小切片 + parity + 独立验证** 迁移。禁止同时大改多个跨仓 Contract。

# 1. AgentScape

## Current

`WorldRuntime` 直接构造 Connector/Generation；`GenerationOrchestrator` 同时 discovery、execution、composition、artifact import、compile；`AssetLibrary` 同时 search/resolve/generate。

## Target

Architecture Card 01：模块化领域核心，Provider 只通过 Asset Sourcing Port 进入。

## Slices

1. **A1 Composition Root**：把 Connector/Generation 创建从 `WorldRuntime` 移到外部 composition root；先保持行为不变。
2. **A2 Asset Sourcing Port**：引入一个高内聚 Asset Sourcing 边界；LegacyGenerationAdapter 暂接旧 Orchestrator。
3. **A3 Artifact-first**：旧路径返回规范化 Artifact；显式 verify → AssetCompiler；禁止 Orchestrator 直接“成功即 Asset”。
4. **A4 AssetLibrary Purification**：移除 generation ownership，只保留 list/search/resolve existing assets。
5. **A5 Catalog Shrink**：ProviderRegistry/Catalog 只 discover/resolve，不 execute workflow。
6. **A6 Retire Orchestrator**：所有 parity/E2E 通过后删除 GenerationOrchestrator 作为架构中枢。
7. **A7 Skill Surface**：Agent 默认只见高层 source/generate/build/interact；低层 submit/get/cancel 进入 debug/operator 面。

## Gate

真实苹果链必须持续通过：real provider → Artifact → AssetCompiler → WorldPipeline → table support verify → Agent navigate → pickup verified。

# 2. AgentScape-client

1. 对齐 Shared Contracts 语义，不创建万能 Provider model。
2. 把 transport 明确成 direct/HTTP/gateway adapter。
3. Artifact verifier 成为公共稳定能力。
4. CLI 提供最小 image/3d/verify/smoke 调用。
5. 保持无业务 DB。

Gate：可以脱离 AgentScape UI 独立调用 Provider 并验证结果。

# 3. modal-gen-client

1. 冻结 pairing/origin/scope 安全语义和现有安全测试。
2. 将 Provider 业务选择与 generation composition 从 Gateway 移出。
3. Provider adapter 只做 contract mapping + transport。
4. Job/Artifact Store 明确为 local projection，字段保留 provider execution/artifact identity。
5. 删除任何只为“global generation platform”存在的状态/接口。

Gate：浏览器 secret isolation、pairing、scope、2D/3D forwarding 全通过；无业务 workflow regression。

# 4. modal-2D-client

低改动：

1. 明确 local mirror 与 provider execution identity。
2. 对齐 ArtifactDescriptor 输出。
3. 保留 request_id/idempotency/restart recovery。
4. 不新增 Project/UI/多 Provider 能力。

Gate：submit → restart → recover → PNG verify。

# 5. modal-2D

极低改动：

1. 固定最小 input/output contract。
2. readiness/models 与 inference 分离。
3. 输出 PNG descriptor/digest 可独立验证。
4. 保持模型加载/推理高内聚；不引入业务 DB。

Gate：真实 GPU smoke + PNG verifier。

# 6. modal-3D-client

## Current

ProjectStore、Jobs、GenerationIntent、Preprocess、ConnectorStore、UI/Server 多 ownership 混合。

## Slices

1. **3C1 Application Boundary**：先在现有实现外建立 Project/Preprocess/Generation 三个稳定应用入口，行为不变。
2. **3C2 Project Domain**：ProjectStore 只拥有 source/files/history/selection；移出 generation 状态。
3. **3C3 Preprocess Domain**：rembg/component/canonicalization/verification 只产生 local artifacts，不修改 generation store。
4. **3C4 Generation Saga**：统一 intent/idempotency/remote_created/uncertain/recovery/provider binding；合并重复 job truth。
5. **3C5 Artifact Layer**：source/matte/canonical/GLB 使用统一 local descriptor/location/digest。
6. **3C6 Connector Adapter**：connector/* 只调用 Application Boundary，不直接碰 Store。
7. **3C7 UI Adapter**：UI 只消费 Application API。
8. **3C8 Delete Legacy Cross-links**：所有 recovery/E2E 通过后删除重复 store/shortcut。

Gate：raw image → preprocess → canonical → real modal-3D → GLB；进程重启后 uncertain/remote job 可恢复；Connector compatibility 仍通过。

# 7. modal-3D

1. 固化 Provider API 与 model/profile registry。
2. Gateway 只 validation/readiness/dispatch。
3. 各 Model Worker 独立拥有模型依赖/GPU/pre/postprocess。
4. GLB Artifact normalization/digest 统一。
5. preprocess 仅在出现真实跨仓复用后才提升边界。

Gate：每个 enabled model 独立 smoke；gateway capabilities/readiness 不被单个模型冷启动长期阻塞。

# 8. kaggle-inference-hub

1. **K1 Task Core**：从 `app.py` 提炼 queue/lease/task state truth，保持现有 DB 行为。
2. **K2 Worker API**：register/claim/heartbeat/input/complete/fail/upload 独立路由与测试。
3. **K3 Consumer API**：submit/status/cancel?/artifact；不暴露 lease 细节。
4. **K4 Artifact Layer**：completion 与 immutable artifact identity 分开。
5. **K5 Prompt Pipeline**：降级为 Consumer helper。
6. **K6 UI/WebSocket**：只通过 Consumer API/View model 观察。
7. **K7 AgentScape Adapter**：只有 K1-K6 稳定后才加入 AgentScape consumer adapter。

Gate：worker crash → lease/reclaim 正确；consumer 不依赖 Worker API；artifact 可独立验证。

# 9. modal-build

1. **B1 Build Manifest**：记录 upstream revision、patch digest、dependency/runtime identity、weights、output digest。
2. **B2 Build Verification**：import/ABI/runtime smoke 成为 release gate。
3. **B3 Runtime Composition Root**：保留现有 route，先把 3000+ 行 runtime 的 wiring 集中到 app/composition root。
4. **B4 Split by GPU Lifecycle**：依次迁 image、image3d、retexture、affordance；每迁一块保持 endpoint parity。
5. **B5 Jobs/Artifacts**：抽出共享 durable job/artifact mechanics，但不统一不同 capability 的业务输入。
6. **B6 Evidence Levels**：segmentation/raw grasp/semantic/SAPIEN/AgentScape findings 永久分级。
7. **B7 Delete Full-build Runtime Path**：production runtime 不再在线 build 本应离线固定的依赖。

Gate：Build 可复现；runtime 无动态 CUDA build；每个 capability real GPU smoke；AgentScape evidence ingestion 不回退。

# 10. EmbodiedGen

不重写。

- 固定 revision。
- patch 只存在 `modal-build`。
- compatibility test 属于 `modal-build`。
- 不接受 AgentScape-specific upstream architecture patch。

# 11. modal-lab

不做 production refactor。

- 保留一实验一目录。
- 每个实验记录 upstream/revision/run/result/cost/benchmark。
- 成功实验只能通过 Promotion Gate 进入 production repo，禁止 runtime import。

# 12. AgentScape-plan

本次重建即目标结构。

后续只做：

- Card/Contract/Runtime View 随 architecture-significant change 更新。
- Integration Ledger 随跨仓箭头更新。
- Migration 文档记录切片与 Gate，不记录零散 debug TODO。
- 新决策需要 ADR 时才新增 ADR。
