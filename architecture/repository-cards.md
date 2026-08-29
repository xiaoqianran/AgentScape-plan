# Repository Architecture Cards

本文件描述当前仓库边界。`AgentScape` 内部不再使用总 `src/` 容器；根目录直接表达稳定产品 subsystem。

# CARD 01 — AgentScape

**Identity**：Agent orchestration + Human Studio + Generation control plane + Asset/World domain core。

```text
AgentScape
├─ studio/        Human app / editor / UI / local persistence
├─ agent/         Agent / LLM / tools / skills / recovery
├─ generation/    Job / Artifact / Connector / Provider-facing orchestration
├─ asset/         Asset truth / admission / adapters / compiler
├─ world/         World spec / compiler / runtime / verification / content
├─ core/          business-neutral primitives only
├─ api/           Vercel Functions deployment boundary
├─ services/      independently runnable services
├─ sdk/           externally consumed SDKs
├─ tests/         cross-domain integration/regression/e2e
└─ tooling/       repository validators / scripts / experiments
```

**Owns**：Agent Run、Human workflow、provider-neutral Generation projection、admitted Artifact、Asset、World、Runtime truth。

**Does not own**：Modal credential/runtime、GPU model lifecycle、Provider-private Job/Artifact storage。

## Internal ownership rules

```text
studio       → human-facing composition
agent        → agent-facing composition
generation   → provider consumer + Artifact→Asset orchestration
asset        → reusable Asset truth
world        → World truth/runtime/verification
core         → no product-domain dependency
```

禁止重新引入根级 `src/`、`pipeline/`、`validation/`、`adapters/`、`helpers/`、`utils/` 作为技术型总目录。代码必须跟 owner 走。

`api/`、`services/`、`sdk/` 可以占据根目录，是因为它们分别拥有真实 deployment/release boundary，而不是为了技术分类。

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

可选本机安全网关。拥有 pairing/origin/scope/session 与统一 transport projection；不拥有业务编排。

## Package Card — modal-2D-client

Image Provider Reference Sidecar。拥有本地可恢复 Job mirror、Artifact fetch/verify/cache；不拥有全局 provider routing。

## Package Card — modal-2D

Image Generation Provider。拥有模型、GPU 推理和 Provider-private artifact。

## Package Card — modal-3D-client

3D Provider Reference Sidecar。拥有输入上传、可恢复 Job mirror、GLB fetch/verify/cache。

## Package Card — modal-3D

3D Generation Provider。拥有模型选择、GPU 推理和模型输入 conditioning。

## Package Card — modal-EmbodiedGen

EmbodiedGen build/runtime integration。拥有 upstream pin、兼容 patch、可复现 CUDA/PyTorch 构建产物与 Modal runtime。上游 `HorizonRobotics/EmbodiedGen` 仅是 source dependency。

# CARD 03 — AgentScape-plan

**Identity**：架构文档权威。

**Owns**：System Landscape、Contracts、Runtime Views、ADR、Migration、Verified Baseline。

**Does not own**：任何产品 runtime 或部署。

# Retired Repository Cards

以下名称只允许出现在迁移/历史说明，不得作为当前 Repository Card：

```text
AgentScape-agent
modal-inference-hub
modal-gen-client          (standalone repo)
modal-2D-client           (standalone repo)
modal-2D                  (standalone repo)
modal-3D-client           (standalone repo)
modal-3D                  (standalone repo)
kaggle-inference-hub
modal-build               (standalone repo)
EmbodiedGen               (standalone AgentScape workspace repo)
modal-lab
```
