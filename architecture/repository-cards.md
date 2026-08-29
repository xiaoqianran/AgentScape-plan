# Repository Architecture Cards

本文件描述当前仓库边界。AgentScape 根目录直接表达稳定产品 subsystem；Provider 实现集中在兄弟仓库 `modal-provider`。

# CARD 01 — AgentScape

**Identity**：Agent orchestration + Human Studio + provider-neutral Generation control plane + Asset/World domain core。

```text
AgentScape
├─ studio/        Human app / editor / UI / local persistence
├─ observatory/   Developer Runtime observation surface
├─ agent/         Agent loop / gateway / prompt policies / domain skill packs
├─ generation/    GenerationRuntime / Connector / Job / Artifact / Provider projection
├─ asset/         Asset truth / admission / adapters / compiler
├─ world/         World spec / compiler / runtime / verification / content
├─ core/          business-neutral primitives only
├─ api/           AgentScape-owned deployment capabilities
├─ services/      independently runnable services
├─ sdk/           externally consumed SDKs
├─ tests/         domain + contracts + integration + e2e tests
└─ tooling/       repository validators / scripts / experiments
```

**Owns**：Agent Run、Human workflow、Generation consumer projection、admitted Artifact、Asset、World、Runtime truth。

**Does not own**：remote Provider credential/runtime、GPU lifecycle、Provider-private execution/storage。

## Internal ownership

```text
studio
  → Human composition
  → owns GenerationJobCenter UI

observatory
  → Developer observation surface
  → consumes production Runtime/domain only
  → owns no business truth

agent
  → Agent Run
  → domain skill packs
  → composable prompt policies

generation
  → GenerationRuntime
  → Connector session / capability snapshot
  → ProviderRegistry projection
  → Job / Artifact
  → Artifact→Asset publication

asset
  → reusable Asset truth / Compiler / admission

world
  → desired/compiled/live World truth
  → Physics / Navigation / Interaction / Verification

core
  → business-neutral primitive only
```


### Observatory constraints

```text
Observatory → production Runtime/domain   allowed
Production → Observatory                  forbidden
```

Observatory 只消费 production debug/semantic contract；不实现第二套 Physics/Spatial/Asset truth。Synthetic fixture 不得复制 production Manifest contract。

当前 tests 归 `tests/observatory/`。

### Generation constraints

```text
Generation → Studio       forbidden
Provider implementation   forbidden inside AgentScape
remote Provider defaults  forbidden in ProviderRegistry
```

正式主链只允许：

```text
GenerationRuntime
→ Connector capability snapshot
→ Job / Artifact
→ Asset publication
```

`LegacyAuthoringShell`、direct HTTP asset generator 与 `runtime.authoring` 已退役。

### Agent constraints

`registerCoreSkills.js` 只做 skill-pack composition。Tool handler 按 domain 放入 skill packs；Agent system prompt 由 policy modules 组合，Tool schema/description 是参数与结果 contract 的权威。

### Python SDK constraints

Python SDK 只作为 Connector consumer。`agentscape.providers.*`、Kaggle/direct Modal Provider client 与 direct generation pipeline 不属于当前 public surface。

### Test / CI constraints

Tests 分组为：

```text
agent/
asset/
generation/
world/
studio/
observatory/
contracts/
integration/
e2e/
```

PR/push 的统一 AgentScape Check 必须同时执行 Runtime、Python SDK、Asset Compiler Service Gate，不使用会漏掉跨模块 contract 破坏的 path filter。

# CARD 02 — modal-provider

**Identity**：AgentScape 的 Modal Provider monorepo。

```text
modal-provider
├─ modal-gen-client
├─ modal-2D-client
├─ modal-2D
├─ modal-3D-client
├─ modal-3D
└─ modal-EmbodiedGen
```

**Owns**：Provider execution、GPU/runtime、sidecar mirror/cache、security gateway、build/runtime compatibility。

**Does not own**：Agent intent、Human project truth、Asset semantic truth、World truth。

## Package Card — modal-gen-client

可选本机安全网关。拥有 pairing/origin/scope/session 与 transport projection；不拥有 AgentScape business orchestration。

## Package Card — modal-2D-client

Image Provider Reference Sidecar。拥有本地可恢复 Job mirror、Artifact fetch/verify/cache。

## Package Card — modal-2D

Image Generation Provider。拥有模型、GPU 推理与 Provider-private artifact。

## Package Card — modal-3D-client

3D Provider Reference Sidecar。拥有输入上传、可恢复 Job mirror、GLB fetch/verify/cache。

## Package Card — modal-3D

3D Generation Provider。拥有模型选择、GPU 推理与输入 conditioning。

## Package Card — modal-EmbodiedGen

EmbodiedGen build/runtime integration。拥有 upstream pin、兼容 patch、可复现 CUDA/PyTorch 构建产物与 Modal runtime。

# CARD 03 — AgentScape-plan

**Identity**：架构文档权威。

**Owns**：System Landscape、Contracts、Runtime Views、ADR、Migration、Verified Baseline。

**Does not own**：任何产品 runtime 或部署。

# Retired Repository / Runtime Surfaces

以下名称只能出现在 migration/history，不得重新作为当前 architecture surface：

```text
AgentScape-agent
modal-inference-hub
kaggle-inference-hub
modal-build standalone
EmbodiedGen standalone AgentScape repo
modal-lab standalone

LegacyAuthoringShell
runtime.authoring
HttpAssetGenerator
/api/capabilities/asset-generate
agentscape.providers.* direct-provider SDK
```
