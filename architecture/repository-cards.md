# Repository Architecture Cards

本文件描述 **当前仓库边界**。旧的 14-repository card 已作废。

# CARD 01 — AgentScape

**Identity**：Agent orchestration + Human workflow + Asset/World domain core。

```text
AgentScape
├─ src/agent            Agent / LLM / VLM / Tools
├─ src/skills           reusable capability workflows
├─ src/ui               Human task/run/editor surfaces
├─ src/connector        provider-neutral connector client
├─ src/jobs             generation job projection/reconcile
├─ src/artifacts        artifact admission/integrity
├─ src/providers        provider registry/snapshot
├─ src/assets           reusable asset domain
├─ src/compiler         asset/world compilation
├─ src/runtime          world runtime
├─ src/validation       verification
└─ sdk/python           first-party SDK/CLI
```

**Owns**：Agent Run、Human workflow、admitted Artifact、Asset、World、Runtime truth。

**Does not own**：Modal credential/runtime、GPU model lifecycle、Provider-private Job/Artifact storage。

**Public integration**：只依赖稳定 Connector/Provider capability、Job、Artifact contract。

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
