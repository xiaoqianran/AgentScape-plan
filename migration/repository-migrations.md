# Repository Migrations

所有迁移使用 **小切片 + parity + 独立验证**。禁止同时大改多个跨仓 Contract。

# 1. AgentScape

## Target

Asset + World Domain Core。

## Slices

```text
A1 expose stable Asset API
A2 establish Asset Repository
A3 move Agent/Skills out to AgentScape-agent
A4 caller-driven publish_asset(Artifact)
A5 remove AssetLibrary; AssetCatalog is the single read API
A6 remove WorldRuntime → Connector/Generation
A7 move GenerationOrchestrator out of Core; retire after external Caller parity
A8 ProviderRegistry leaves domain path
A9 preserve World Compiler/Runtime
```

Gate：真实苹果链持续通过；Core 可在零 Provider 配置下运行 Asset/World/Runtime。

# 2. AgentScape-agent

**新仓。**

```text
agent.py     Agent Run / LLM tool loop
skills.py    source_3d_asset / build_world
 tools.py    stable tool surface
runs.py      checkpoint / durable run state
adapters.py  AgentScape / 2D / 3D / VLM
```

第一旗舰 Skill：

```text
source_3d_asset
→ search
→ generate N images
→ VLM rank/select/retry
→ generate 3D
→ verify
→ publish Asset
```

Gate：Text → 3D → Asset → World placement 自动闭环。

# 3. AgentScape-client

1. 移除万能 Provider Client 定位。
2. 只暴露 Asset/World/Runtime domain contract。
3. 保留 CLI/reference examples/contract validation。
4. 无业务 DB。

# 4. modal-inference-hub

原 `modal-3D-client` 演化而来。

```text
H1 rename identity + preserve legacy data dir
H2 Project/UI/History 保留
H3 Human semantic selection/mask 保留
H4 3D execution → new modal-3D-client
H5 image candidate generation → modal-2D-client
H6 workflow history only references sidecar jobs/artifacts
H7 optional publish → AgentScape
H8 delete duplicate Modal Job/Artifact transport implementation
```

Gate：Hub full tests；历史 Project/Job 数据无损；2D/3D Provider 可脱离 Hub 独立 smoke。

# 5. modal-gen-client

1. 冻结 pairing/origin/scope 安全测试。
2. 删除业务 composition/routing policy。
3. 只做 mechanical forwarding 到 2D/3D Sidecar。
4. Server-side Agent/Hub 不强制经过 Gateway。

# 6. modal-2D-client

**当前边界基本稳定。**

保持：durable mirror、request identity、Volume-first Artifact fetch、integrity verify/cache、legacy fallback。

禁止新增 Project/UI/Asset/World。

# 7. modal-2D

**当前边界基本稳定。**

保持：image.generate、两模型 runtime、PNG Artifact、named Volume。

Gate：`040-modal-2d-provider`。

# 8. modal-3D-client

新独立 Reference Sidecar。

```text
3C1 models/capability cache
3C2 request identity + durable remote projection
3C3 input upload
3C4 GLB Volume fetch/verify/cache
3C5 restart/unknown-submit recovery
3C6 public input contract 从 canonical RGBA 放宽为 image + optional mask
```

最后一项只能在 modal-3D InputConditioner 完成后切换。

# 9. modal-3D

当前多模型 Gateway/Registry/Worker 架构保留。

```text
3D1 stabilize capability/artifact identity
3D2 add Provider InputConditioner
3D3 preserve valid alpha/mask
3D4 auto segmentation/rembg when absent
3D5 bbox/crop/center/normalize
3D6 keep model-private canonical rules inside workers/adapters
3D7 loosen public input contract
```

Gate：迁移前后都跑 `041-modal-3d-provider`；4/4 model parity 必须保持。

# 10. kaggle-inference-hub

```text
K1 Task Core
K2 Worker API
K3 Consumer API
K4 Artifact Layer
K5 Prompt Pipeline becomes consumer helper
K6 UI/WebSocket only consume Consumer API
K7 add Agent/Hub adapter only after K1-K6 stable
```

# 11. modal-build

```text
B1 Build Manifest
B2 Build Verification
B3 Runtime Composition Root
B4 split by GPU lifecycle
B5 shared Job/Artifact mechanics
B6 explicit evidence levels
B7 remove runtime full-build path
```

# 12. EmbodiedGen

不重写。固定 revision；patch/compatibility 全归 modal-build。

# 13. modal-lab

不 productionize。

新增职责：Provider migration baseline verification。

```text
experiment definition
→ real run
→ evidence
→ PASS/FAIL
→ architecture/promotion decision
```

Production runtime 永不 import modal-lab。

# 14. AgentScape-plan

保持单一 Architecture Authority。

- Card 随 architecture-significant evidence 更新。
- Ledger 始终区分 Current 与 Target。
- 真实实验结果进入 Verified Baseline。
- ADR 只记录真正架构决策。
