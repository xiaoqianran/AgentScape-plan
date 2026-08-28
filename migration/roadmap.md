# Architecture Migration Roadmap

目标不是一起重写全部仓库，而是按 **事实验证 → 拓扑纠正 → Capability 稳定 → Caller 建立 → Core 提纯 → Legacy 删除** 迁移。每一步必须有独立 Gate。

# R0 — Architecture Authority

**状态：DONE。**

权威由 `System Landscape / Repository Cards / Shared Contracts / Runtime Views / Integration Ledger / Repository Migrations / Verified Baseline / ADR` 组成。

# R1 — Provider Reality Verification

**状态：DONE。**

```text
040-modal-2d-provider   PASS
041-modal-3d-provider   PASS
```

真实结论：

```text
modal-2D: 2/2 models verified
modal-3D: 4/4 models verified
BiRefNet/current canonicalization verified
Gateway idempotency verified for all 4 models
GLB integrity verified for all 4 models
```

这些结果是后续 3D public-input migration 的 parity baseline。

# R2 — Repository Topology Correction

## R2A — modal-inference-hub

原 GitHub `modal-3D-client` 已原地 rename 为 `modal-inference-hub`，完整历史保留。

目标身份：Human UI / Project / semantic selection / workflow composition。

## R2B — new modal-3D-client

新独立仓已建立并推送，目标身份：纯 3D Reference Sidecar。

## R2C — AgentScape submodule topology

目标：

```text
providers/modal/inference-hub   → modal-inference-hub
providers/modal/object3d-agent  → new modal-3D-client
```

Gate：Hub full tests + new 3D Client tests + Connector/root integration tests。

# R3 — Capability Boundary Stabilization

## R3A — modal-2D / modal-2D-client

**状态：DONE。** Shared Artifact identity、Volume-first fetch、recovery、real GPU Gate 已完成。

## R3B — modal-3D / modal-3D-client

**状态：CODE DONE / PARITY GATE PENDING。**

先固定：capability/model identity、Provider Artifact identity、Sidecar durable projection、GLB verification。

### InputConditioner 迁移

```text
CURRENT public:
canonical 1024 RGBA

TARGET public:
image + optional mask/alpha
        ↓
Provider InputConditioner
        ↓
internal model canonical
```

**代码已落地（2026-08-28）。** `modal-3D@487b661 feat: condition source images inside modal 3d`
已在 Provider 内部实现输入 Conditioning：

```text
modal_3d/conditioning.py       condition_image(data, predicted_mask=None)
                               ├─ 已有 meaningful alpha → preserve-alpha
                               └─ opaque source → RemBgWorker 预测 mask
                                  → refine_mask（BiRefNet tail，与 041 同一策略）
                                  → letterbox 到 canonical 1024 RGBA

modal_3d/capabilities.py       PUBLIC_IMAGE_INPUT
                               ├─ mediaTypes  image/png, image/jpeg, image/webp
                               ├─ maxBytes    20 MiB
                               ├─ alpha       optional
                               ├─ conditioning  provider
                               └─ pathPrefix  source-inputs/

modal_3d/rembg_gateway.py      condition() 远程入口
                               └─ client-inputs/ 走 legacy pass-through
                                  source-inputs/ 走 condition_image()

modal_3d/gateway.py            source-inputs/ → conditioned_generation
                               client-inputs/ → spawn_generation（原路径）
```

**双前缀过渡设计。** 关键点是 `041` parity 被显式保护：
`client-inputs/` 下已验证的 canonical 输入走 `_legacy_canonical()`
pass-through，字节原样保留（`source_sha256 == canonical_sha256`）；
只有 `source-inputs/` 走新的 `condition_image()`。因此**新路径可用，老路径不变**，
`041` 至今仍是严格的 parity gate。

**Worker 内部契约未变。** `capabilities.py:71` 仍强制
`worker input contract must be canonical 1024x1024 RGBA PNG`。
Conditioning 发生在 Gateway/Provider 边界内，Worker 不感知。
这正是 CARD 09 要求的 `PUBLIC: image/* + optional mask/alpha` /
`INTERNAL: model-required canonical image/mask` 分离。

**Sidecar 已跟上。** `modal-3D-client` 侧 `SOURCE_PATH_PREFIX = "source-inputs/"`、
`SOURCE_MEDIA_TYPES = (png, jpeg, webp)`、`SOURCE_MAX_BYTES = 20 MiB`，
`jobs.py` 的 `_CONDITIONING_EVIDENCE_FIELDS` 已包含
`strategy / engine / source_sha256 / canonical_sha256 / mask_elapsed_ms` 等
conditioning evidence 字段。

### 已完成 Gate

```text
modal-3D unittest discover -s tests     87/87 PASS   （与 CI 一致）
modal-3D python -m compileall           PASS
```

注：CI (`.github/workflows/ci.yml`) 使用 `unittest discover -s tests`，
不是 pytest；`pytest tests/` 同样 87 PASS。仓库 `archive/sam3_1/` 下的
`test_sam3_materialize.py` 导入已删除的 `modal_3d.sam3_1` 模块，
collect 即失败——这是 487b661 之前就存在的 archive 残留，不属于本次 Gate。

### 仍待完成 — PARITY GATE

**R3B 不能宣布 DONE，直到重跑 `041` 四模型矩阵并保持 parity。**

```text
[ ] 用 source-inputs/ 路径重跑 041 四模型矩阵
      FastSAM3D++ / Hermite-TRELLIS2++ / Hunyuan2.1++ / Pixal3D
[ ] 证明 conditioning 后的 GLB 与 legacy canonical 路径 digest 一致或差异已解释
[ ] 确认 records 中 conditioning.strategy 分布（preserve-alpha vs birefnet）
[ ] 四模型 GLB magic/version/declared bytes/SHA 全匹配
[ ] parity 通过后，才可将 public contract 从 canonical-only 放宽并标记 verified
```

理由：`condition_image()` 引入了新的 `refine_mask` / letterbox 实现。
本地 87 个单测证明的是**纯函数正确性**，不是**真实 GPU 四模型输出 parity**。
在 parity 证据出现前，文档只能写 `CODE DONE`，不能写 `verified`。

# R4 — Human Caller Purification

`modal-inference-hub`：

```text
H1 Project Domain 保留
H2 Human semantic preprocess 保留
H3 3D execution → new modal-3D-client
H4 image generation → modal-2D-client
H5 candidate workflow 只引用 Sidecar job/artifact
H6 optional publish → AgentScape Asset API
H7 删除 Hub 内重复 Provider Job/Volume implementation
```

Human semantic selection 留 Hub；model-required automatic rembg 下沉 `modal-3D`。

# R5 — AgentScape-agent

**状态：DONE（旗舰迁移切片）。** 独立仓库 `xiaoqianran/AgentScape-agent` 已建立；它是与 `modal-inference-hub` 平级的 Caller，不作为 AgentScape Core 的运行时 submodule 依赖。

```text
A1 Agent Run / tool loop + checkpoint    DONE (cross-process resume verified)
A2 AgentScape / 2D / 3D / VLM adapters  DONE (real one-shot verified)
A3 source_3d_asset Skill                 DONE
A4 build_world                           DONE inside one-shot slice; no extraction pressure
```

当前实现继续遵循 Experiment-oriented Modular Monolith：生产代码保持少量高内聚文件，不建立 service/repository/manager/factory 横向层。`runs.js` 只有因为 Agent Run 具备独立持久化/故障恢复生命周期才被抽出；`source_3d_asset.js` 保持 Text → candidates → VLM selection → 3D → Asset 的单文件 Vertical Slice；`run_text_to_world.js` 只组合 source Asset 与 World/Runtime verification。真实旗舰链已经完成 4 × SANA-Sprint 1.6B → `stepfun-ai/step-3.7-flash` Vision ranking → FastSAM3D++ → verified GLB → ArtifactRegistry → AssetCompiler → WorldPipeline → Runtime `ON table` verification，最终 `completed / verified`。2D candidate generation 已从 4 个独立 GPU Job 收敛为一个 Sidecar batch Job / 一个 `SanaSprintWorker`，warm 4 图实测 9.075s。2D/3D Adapter 继续通过 Sidecar `/v1/models` 做 lazy capability preflight；未知 model/profile 在 GPU submit 前 fail-closed。Cancellation Ownership 已贯穿 Agent Run → high-level Tool → `source_3d_asset` → 2D/VLM/3D Adapter → Sidecar DELETE。Agent step 支持原子 checkpoint，跨进程 Tool resume 已通过 crash/restart 实验并以稳定 `executionId` rebind Sidecar Job。在没有独立 State Owner、failure lifecycle、deployment 或真实性能压力前，不抽独立 `build_world` service。

旗舰 Gate：

```text
Text
→ N image candidates
→ VLM rank/select
→ 3D
→ Artifact verify
→ Asset Repository
→ World placement
→ Runtime verification
```

# R6 — AgentScape Core Purification

**状态：IN PROGRESS — Asset/World fracture plane 已有 executable evidence。**

| Slice | Status | Evidence |
|---|---|---|
| C1 stable Asset boundary | **DONE** | `9fb7fec` + `9369e12`：`AssetRef`、World Compilation v2、`publishAsset()` public API 已稳定 |
| C2 Asset Repository/state owner | **DONE (module boundary)** | `0a41a93`：AssetManager/Store/Catalog 归 Asset Module；唯一生产构造点为 `createAssetModule()` |
| C3 move Agent/Skills out | **IN PROGRESS elsewhere** | Agent 独立仓已建立；本迁移线不改 Agent |
| C4 caller-driven `publish_asset(Artifact)` | **DONE** | `9369e12`：`assetModule.publishAsset()` 为稳定入口；真实 modal-lab GLB experiment 通过 |
| C5 single Asset read API | **DONE** | `a0b522a` + `86a2232`：AssetLibrary 已删除；AssetCatalog 为唯一 read/search/get/resolveExisting API |
| C6 remove WorldRuntime → Connector/Generation | **DONE** | `bc3be81`；architecture test fail-closed |
| C7 generation authoring leaves Core | **DONE / legacy retirement TODO** | `ef2830d` + `4dab4f7`：Orchestrator 与 Job Center 均移至 `src/authoring/`，`src/generation/` 无生产模块；彻底删除等待外部 Caller parity |
| C8 ProviderRegistry leaves Asset/World domain | **DONE for domain core** | `7bafc7c`；仅 authoring shell 持有 |
| C9 preserve World Compiler/Runtime behavior | **PASS** | 746/746 root tests + independent Asset/World experiments；测试数下降来自删除冗余 facade 测试 |

已验证结构：

```text
Asset Module
  createAssetModule()
  → Manager / Store / Catalog
  → AssetRef
       ↓
World Core
  injected Asset module
  → World Compilation v2
  → execution entities contain AssetRef only
```

最终目标不变：AgentScape Core 在零生成 Provider 配置下也能独立运行 Asset/World/Runtime。

# R7 — Kaggle / modal-build Purification

Kaggle：Consumer API / Task Core / Queue+Lease / Worker API / Artifact Layer。

modal-build：Build Plane → immutable runtime artifact → Runtime Plane。

# R8 — Optional Gateway / SDK Cleanup

`modal-gen-client` 收缩成 pairing/origin/scope/credential isolation/mechanical forwarding。

`AgentScape/sdk/python` 作为 monorepo 内第一方 AgentScape Domain SDK/CLI package；独立 `AgentScape-client` repository 已退役。

# R9 — Legacy Deletion

只在 parity 全绿后删除：

- AgentScape `GenerationOrchestrator`。
- WorldRuntime Provider construction。
- Hub 内旧直接 Modal 3D execution/store。
- Connector 指向旧 3D Hub execution endpoints 的路径。
- duplicate Job truth。
- public canonical-RGBA-only 3D contract（InputConditioner 完成后）。
- Provider payload 直接冒充 ready Asset 的路径。

# 全局迁移规则

1. 一次只改变一个主要 Ownership。
2. Strangler：新边界先接 Legacy Adapter，parity 后删除旧实现。
3. 破坏性 Contract 迁移前先保存真实 baseline experiment。
4. 两个实现不自动意味着要建 framework。
5. 当 identity/state/input-output/failure/retry/smoke/contract 都清晰时停止继续拆。
