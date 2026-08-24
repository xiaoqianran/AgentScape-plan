# 第二组统一工作流契约与验收

## 1. 契约目标

第二组必须把 EmbodiedGen 的“目录式 CLI 结果”转换成 Modal 上可组合的工作流事实：

```text
Capability → Request → Job → Stage Events → Artifact Manifest → Validation → Result
```

目录和文件名仍可作为实现细节，但 Connector 与 AgentScape 只能依赖版本化 contract。

## 2. Workflow Identity

每次执行记录：

- stable workflow ID；
- workflow schema version；
- implementation revision；
- EmbodiedGen upstream commit；
- release tag/hash；
- patch set hash；
- weight model/revision；
- option profile/effective values。

同一 stable ID 的主版本不改变输入/输出语义。算法、模型或 release 更新可通过 implementation revision 表达；破坏 contract 才升 workflow 主版本。

## 3. Capability Entry

每个 workflow 发布：

| 字段组 | 内容 |
|---|---|
| identity | ID、display name、version、status |
| inputs | artifact/text/schema、limits、bundle requirements |
| options | JSON Schema、profiles、defaults、seed |
| stages | ordered/DAG stage IDs、粗进度权重 |
| outputs | required/optional artifact roles |
| execution | duration/resource/cost class、cancel/resume support |
| requirements | Secret、license acceptance、network |
| retention | result/intermediate TTL |
| compatibility | Connector/AgentScape notes、deprecated revision |

`status=available` 只能在真实 canary 通过后发布；只有权重预加载函数不算 available。

## 4. Request Envelope

统一请求包含：

- client request ID/idempotency key；
- workflow ID/version；
- input descriptors 或 typed text；
- requested profile/options/seed；
- retention class；
- parent job/child key（layout fan-out）；
- safe user metadata；
- contract/capability hash。

服务端生成：request hash、effective options、selected execution profile。Secret 永远不进入 request body/artifact/lineage；它由部署配置注入。

## 5. Job 状态机

```text
accepted → queued → running → validating → succeeded
                    │  │             │
                    │  ├→ cancel_requested → cancelled
                    │  ├→ failed
                    │  └→ stalled → resumed/running 或 failed
                    └→ rejected（尚未执行，输入/策略错误）
```

定义：

- `rejected`：没有开始付费阶段；请求无效/能力不可用。
- `failed`：已接受后某阶段终止。
- `stalled`：lease/heartbeat 过期但尚无失败证据；control plane 要恢复判定。
- `succeeded`：所有 required artifact 已发布并通过 validation。
- `cancel_requested`：意图已记录，不等同 Worker 已停止。

Job state 至少含 stage、attempt、created/updated、progress estimate、error、artifact summary、parent/children 和 deployment lineage。

## 6. Stage Contract

每个 stage 定义：

- stable stage ID；
- required inputs及其 hash；
- effective options/revision；
- resource class/worker target；
- timeout/heartbeat；
- retry policy/max attempts；
- cancellation boundary；
- required/optional outputs；
- validation；
- checkpoint reuse policy。

### 6.1 Stage 发布协议

1. claim stage attempt；
2. 验证 inputs/revision；
3. 写入 attempt-scoped 临时 artifact；
4. 运行 stage validation；
5. 计算 bytes/hash；
6. commit Volume；
7. 原子更新 active artifact index 与 stage success；
8. 发布 event。

任何一步失败都不能让半成品出现在 result manifest。

### 6.2 Retry 规则

- deterministic input/revision 的成功 checkpoint 可复用；
- model stochastic retry 必须记录新 seed/attempt；
- artifact commit failure 可以重试发布，但不能重新计费执行模型，前提是临时结果仍完整；
- 上游/weights/release revision 变化时 checkpoint 默认不复用；
- layout child Job retry 不重复成功 child。

## 7. Artifact Manifest

每个 Job 最终有 `embodiedgen.artifacts.v1`：

### Artifact 字段

- opaque ID、role、display name；
- MIME/format；
- bytes/SHA-256；
- job/stage/attempt producer；
- relative dependencies；
- validation status/report reference；
- coordinate system、unit、up/front axis、transform；
- geometry/texture/physics summary（适用时）；
- evidence level；
- created/expires。

### Bundle 字段

- bundle ID/type/version；
- primary artifact；
- members/dependency graph；
- required roles；
- provenance；
- warnings/readiness；
- consumer hints。

### 不可变性

成功 artifact 不覆盖。retexture、convert、affordance 产生新 derived bundle，通过 `derived_from` 指向输入 hash。

## 8. Error Contract

错误包含 code/category/stage/attempt/retryable/message/correlation/safe details。

| Code family | 例子 |
|---|---|
| `REQUEST_*` | schema、option、idempotency conflict |
| `INPUT_*` | image/mesh/URDF/bundle 无效、路径逃逸 |
| `CAPABILITY_*` | workflow disabled、license/Secret missing |
| `WEIGHT_*` | marker/revision/files missing |
| `MODEL_*` | OOM、inference、quality exhausted |
| `STAGE_*` | timeout、lease lost、retry exhausted |
| `ARTIFACT_*` | commit、missing、hash/dependency mismatch |
| `CONVERT_*` | target parse/reference/scale invalid |
| `SIM_*` | simulator load/physics/grasp validation |
| `CANCEL_*` | cancel acknowledged/too late |
| `INTERNAL_*` | 未分类脱敏错误 |

LLM/外部 API 返回内容不得直接作为安全 error message。

## 9. 统一坐标、单位与物理声明

每个 3D bundle 必须明确：

- length unit，默认目标为 meter；
- up axis/front axis/handedness；
- source→canonical transform；
- visual/collision local transforms；
- bounds/center/ground offset；
- scale 是否 inferred/default/measured；
- mass/friction/inertia 的 source/evidence；
- static/dynamic intent；
- joint/part/grasp reference frame。

缺失这些信息时仍可输出视觉 GLB，但 readiness 必须 provisional；不得用 `[1,1,1]` 或 mass=1 的默认值冒充测量值。

## 10. Readiness/Evidence

第二组 result readiness 与 AgentScape admission 不相同，但可提供上游证据：

| 状态 | 第二组含义 |
|---|---|
| `ready` | workflow 必需输出和本工作流验证均通过 |
| `provisional` | 可消费但存在明确 fallback/unknown，例如 visual collision、unknown scale |
| `rejected` | 结果不能安全消费 |

AgentScape 必须重新编译和 admission，不能因为 provider `ready` 自动变成 runtime `ready`。

## 11. 各工作流必需 Artifact

| Workflow | 必需 | 可选/后置 |
|---|---|---|
| Image→3D | visual GLB、URDF、validation、lineage | OBJ、collision、GS PLY、video |
| Text→3D | generated image、Image→3D bundle、validation | prompt/quality reports |
| Retexture | textured GLB、texture、validation、derived lineage | OBJ bundle、multiview、video |
| Convert | target format bundle、dependency report、validation | multiple targets |
| Affordance | part GLB、affordance JSON、enriched URDF、evidence report | grasps、sim report |
| Background | mesh/GLB、pano、validation | GS PLY/config、video |
| Layout | layout.json、asset refs/bundles、background ref、validation | scene tree、images、video |
| Room | instance/environment manifest、URDF bundle、validation | Blender、USD/textures |

## 12. 工作流验收

### WF-01 Image→3D

- 上传真实图片；
- API job 不使用 sample fallback；
- 五个 stage 各有 event/timing；
- GLB/OBJ/MTL/texture/URDF 依赖可解析；
- validation report 与 artifact hash 一致；
- control process 重启后可从 checkpoint 恢复；
- cancel 在 stage boundary 停止；
- 同 idempotency key 不重复生成。

### WF-02 Text→3D

- 中英文各 canary（按模型支持）；
- fixed Kolors revision 与 seed 写入 lineage；
- generated image 可独立下载；
- reconstruction retry 不重复 Text→Image；
- prompt 不出现在普通日志；
- 输出进入与 Image→3D 相同 bundle contract。

### WF-03 Retexture

- 先冻结当前 `RetextureWorker`、派生 endpoint、geometry-preserved report 与 effective options 作为 regression baseline；
- 上游两个 demo mesh canary；
- GLB 与 self-contained OBJ 输入；
- 无 UV、恶意 OBJ 引用、超大 mesh 负向用例；
- topology/transform policy 可核对；
- texture/GLB 可解析并满足 profile budget；
- RoboAssetGen exact revision、offline load、no nvcc。
- source Job/artifact 有 lease，派生 lineage/hash 完整；复制沿用的 URDF/GS 与本次新验证的 texture/mesh evidence 明确分层；
- 至少一次真实 Modal L40S cold/warm E2E，而不是仅以源码断言测试作为完成证据。

### WF-04 Convert

- URDF bundle → MJCF；
- mesh/URDF → USD（若 capability available）；
- 所有引用存在；
- target parser load；
- bounds/scale 与源一致在容差内；
- physics USD 与普通 USD readiness 不混淆。

### WF-05 Affordance

- 上游 demo URDF；
- 输入不可变；
- segmentation/semantics/grasp/sim 各自 checkpoint；
- face IDs 可读；
- affordance JSON schema 有效；
- simulated grasp 与 proposal 区分；
- enriched URDF 引用 bundle 内文件；
- 没有 LLM Secret 泄露。

### WF-06 Background

- tiny profile 快速结构 canary；
- standard profile 真实质量 canary；
- panorama/mesh/GS 可独立恢复；
- mesh→GLB 可由 AgentScape Compiler 读取；
- 长 stage heartbeat/TTL 不被 cleanup；
- GS artifact 标注 runtime support=false。

### WF-07 Layout

- 单任务、2～3 个对象、已有 background；
- reuse-first 命中已有 asset；
- 缺失对象 child Job fan-out；
- 一个 child 失败可定位/重试；
- layout.json 与所有引用 hash 一致；
- parent cancel 传播到未开始 children；
- 转换后的 AgentScape WorldSpec 仍需 world admission。

### WF-08 Room

- explicit minimalist config，无 GPT；
- 固定 seed 可重现结构 metadata；
- generation 与 export checkpoint 分离；
- whole-room URDF、per-instance meshes 可解析；
- USD 无 physics 时 provisional；
- Blender image/临时磁盘/timeout 有实测；
- environment adapter 可生成受预算 GLB bundle。

## 13. 故障注入验收

每个已上线 workflow 至少覆盖：

- Worker 首次冷启动失败；
- GPU OOM；
- Dict handoff 丢失；
- Volume reload/commit 失败；
- stage 完成后 control plane 崩溃；
- event 写入失败；
- artifact hash mismatch；
- Secret 缺失/外部 API timeout；
- cancel 与 success 竞态；
- cleanup 与 active heartbeat 竞态；
- revision 更新后旧 Job 恢复。

## 14. 性能与成本基线

每条 workflow 保存：

- cold scheduler wall；
- image/container/model load；
- stage execution；
- artifact commit/download；
- GPU peak、CPU/memory、disk peak；
- input/output bytes；
- idle tail cost；
- retry/quality rejection rate。

Image/Text 使用固定小/中输入集；scene/layout/room 分 tiny/standard profile。没有足够样本前只做回归阈值，不承诺用户 SLA。

## 15. 第二组 Gate

### Gate 2A：内核

- contract/workflow registry；
- artifact manifest；
- job/stage/idempotency/recovery；
- Image/Text 迁移不回退。

### Gate 2B：AgentScape 高价值资产

- retexture；
- conversion；
- affordance evidence；
- Connector fixtures。

### Gate 2C：世界能力

- background；
- layout；
- world bundle/coordinate contract。

### Gate 2D：大型环境与仿真

- room；
- optional simulation/soft body。

任何 Gate 只以真实 artifact、validation、恢复和 consumer test 为通过证据，不以“函数已部署”代替。
