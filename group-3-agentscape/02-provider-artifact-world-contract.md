# AgentScape Provider、Artifact 与 World 联合契约

## 1. 契约层级

```text
Connector Session
  └── Provider Capability
       └── Generation Request / Job
            └── Result Artifact Bundle
                 ├── Asset Import Contract
                 │    └── Compiler / Manifest / Admission
                 └── World Bundle Contract
                      └── WorldSpec / Runtime / Admission
```

各层都有独立版本。不能用一个无版本“EmbodiedGen payload”承担全部语义。

## 2. Connector Session Contract

会话至少包含：

- connector identity/instance；
- contract version；
- client identity=`agentscape`；
- short-lived token ID；
- scopes；
- issued/expires；
- allowed origins；
- capability hash；
- revoke endpoint/状态。

首批 scope：

- `capabilities.read`；
- `jobs.submit`；
- `jobs.read`；
- `jobs.cancel`；
- `artifacts.read`。

Credential read/write、任意 Volume path、任意 Modal function 调用不在 scope 中。

## 3. Provider Capability Contract

AgentScape 需要的标准化视图：

| 字段 | 含义 |
|---|---|
| provider | `modal-2d` / `modal-3d` / `embodiedgen` |
| operation | stable operation/workflow ID |
| version/status | contract 和可用性 |
| input | text/image/rgba/mesh/bundle/world task |
| output | asset/world/environment artifact roles |
| profiles/options | UI/Agent 可合法提交的 schema |
| execution | async、stage、duration/cost class |
| prerequisites | connection/Secret/license |
| support | cancel/resume/idempotency |
| consumption | AgentScape importer/adapter requirement |

Provider Registry 对外按“能力”而非 Modal App/Function 名查询。

## 4. Generation Request Contract

AgentScape 提交：

- local request ID/idempotency key；
- provider/operation/version；
- typed inputs 或已授权 artifact refs；
- profile/options/seed；
- desired output roles；
- parent world request/asset request ID；
- retention；
- safe label/intent metadata。

Connector 返回稳定 local Job ID。AgentScape 不需要也不持久化 Modal FunctionCall ID。

## 5. Generation Job Projection

字段：

- Job ID/provider/operation；
- status/stage/progress semantics；
- created/updated/completed；
- parent/children；
- effective options/model/workflow revision；
- error；
- result manifest ID；
- cancellation state；
- event sequence。

AgentScape 状态映射：

| Connector | AgentScape Job UI | 资产/世界语义 |
|---|---|---|
| accepted/queued/running | pending | 尚无资产事实 |
| connection_required | recoverable | 不标失败 |
| cancel_requested | cancelling | 不标 cancelled |
| failed/cancelled/expired | terminal non-success | 不进入 Compiler |
| succeeded | result available | 仍需 import/compile/admission |

## 6. Artifact Transport Contract

AgentScape 只使用 opaque artifact ID；Connector 负责内部 path。响应提供：

- MIME；
- content length；
- SHA-256；
- artifact/bundle ID；
- role；
- range/expiry headers；
- producer revision。

规则：

1. 只允许当前 session scope + job ownership 的 artifact。
2. redirect 默认拒绝，除非 Connector contract 明确签名目标。
3. declared 和实际 bytes/hash 必须一致。
4. 编译后持久身份是 compiled storage key + source hash，不是 artifact URL。
5. token/URL 不进入 scene serialization。

## 7. Asset Bundle Contract

### 7.1 必需层

- bundle identity/version/readiness；
- primary visual GLB descriptor；
- coordinates/unit/transform；
- producer lineage；
- validation report；
- warnings。

### 7.2 可选 evidence

- URDF；
- visual/collision mesh descriptors；
- mass/friction/inertia；
- part segmentation；
- part semantics；
- joint definitions；
- grasp proposals/validated grasps；
- texture/material summary；
- preview/GS artifacts。

### 7.3 Evidence source

每项需声明：

- `measured`；
- `generated`；
- `inferred`；
- `default`；
- `provider-validated`；
- `simulator-validated`；
- `unknown`。

AgentScape Adapter 根据 source 决定进入 Compiler input、provenance 还是仅 warning。

## 8. Provider Bundle → Compiler 映射

| Provider artifact/evidence | AgentScape 入口 | 门禁 |
|---|---|---|
| visual GLB bytes | `AssetCompiler.compile(bytes)` | GLTF parse、budget、geometry |
| source unit/axis | normalization evidence | 与 GLB 实际 bounds/transform 对照 |
| URDF visual origin | part/transform proposal | 不覆盖 GLTF truth，冲突则 warning/reject |
| collision mesh | collider candidate | 形状、vertices、budget、frame 校验 |
| part face IDs | partSegmentation | face coverage/index range |
| part semantics | partProposal/provenance | 不是 joint/action proof |
| URDF joint | joint candidate | node/axis/anchors/limits/targets + runtime verify |
| affordance labels | tags/evidence | 只映射 Runtime 支持且有执行证据的 action |
| grasp poses | provenance | 不等于 pickup/grasp verified |
| provider validation | provenance/admission input | 不替代 Compiler checks |

## 9. Manifest/Admission Contract

### 9.1 Manifest source

生成资产完成 Compiler 后使用 `source.kind='compiled'`，保存：

- compiled store key；
- source SHA-256；
- provider bundle/job lineage；
- compiler version/report；
- evidence summary。

不长期使用 `source.kind='glb'` 指向 Connector 临时 URL。

### 9.2 Admission 聚合

```text
provider readiness
  + artifact integrity
  + compiler quality
  + optional runtime verification
  = AgentScape asset admission
```

优先级：rejected 胜过 provisional，provisional 胜过 ready。只有 required evidence 的验证范围一致时才可 ready。

### 9.3 动作准入

- `move` 可由 runtime object 基础能力提供；
- `pickup/place` 需要 dynamic/kinematic carry contract、collider、body 与 Runtime preflight；
- `open/close` 必须有 part node、joint、collider、limits、explicit targets，并通过 articulation verification；
- semantic affordance 不满足上述条件时只保留 provenance。

## 10. WorldSpec v2 概念契约

### 顶层

- schema/version；
- name/description/task intent；
- generation policy；
- environment proposal；
- assets/relations/constraints；
- provenance/seed。

### Asset request

- instance ID；
- existing catalog asset ID 或 provider result ref；
- query/prompt/type；
- required/optional；
- generate/provider policy；
- position；
- rotation（统一 quaternion）；
- scale；
- static/dynamic intent；
- source coordinate metadata；
- tags/role。

### Environment

- built-in environment ID 或 environment bundle ref；
- bounds、ground、up axis；
- visual/collision artifacts；
- spawn/camera/navigation hints；
- resource/admission policy。

### Relations

首批保留 ON/NEAR；后续只增加 Runtime 有真实验证的 INSIDE/ATTACHED 等。每个 relation 可有 tolerance/surface/reference，但 Planner 不能发明 unsupported predicate。

## 11. EmbodiedGen Layout → WorldSpec 映射

| EmbodiedGen | WorldSpec/AgentScape | 处理 |
|---|---|---|
| layout object identity | instance ID + asset ref | sanitize/unique/hash linkage |
| generated asset directory | compiled catalog asset | 逐 bundle import/admission |
| translation | position | unit/axis transform后再验证 |
| orientation | rotation quaternion | 明确 source convention |
| scale | scale | bounds/collider同步检查 |
| background mesh/GS | environment proposal | 首版 mesh GLB，GS unsupported |
| scene graph relation | supported relation | unsupported 保留 warning |
| robot pose | agent/entity proposal | 只在 Runtime 有对应 asset时采用 |
| layout.json physics | provenance/intent | Rapier 为最终真值 |

转换结果不是 `world-ready`，只是一份可提交给 canonical pipeline 的 proposal。

## 12. Environment Bundle Contract

必需：

- visual GLB；
- collision representation；
- bounds/ground/unit/axis；
- primary transform；
- validation/resource summary；
- producer lineage。

可选：

- spawn points；
- camera preset；
- navmesh hints（不能作为最终 navmesh truth）；
- per-instance editable objects；
- pano/preview；
- 3DGS reference。

环境准入：visual/bytes/预算、static collision、world bounds、ground、Rapier attach、Recast rebuild 全部成功；否则替换回滚。

## 13. World Pipeline 扩展顺序

建议 v2 canonical stages：

```text
normalize_spec
→ resolve_environment
→ resolve_assets（reuse-first / async generation）
→ import_compile_assets
→ asset_admission
→ environment_admission
→ compose_layout
→ instantiate
→ apply_relations
→ rebuild_navigation
→ validate
→ bounded_repair
→ finalize
```

具体实现可以把外部长任务拆成 pipeline 之前的 preparation phase；关键要求是 Agent 的 `runWorldPipeline` 不阻塞等待 30 分钟，也不能跳过后半段 admission。

## 14. 世界状态与远端 Job 的事务边界

远端生成不可随浏览器 scene rollback 回滚：

- provider Job 是外部、可能计费且有独立历史；
- imported asset 是 catalog mutation；
- live world instantiate 是可回滚 mutation。

因此：

1. world proposal 创建/等待 Job；
2. 资产成功后编译注册；
3. canonical pipeline 在 scene transaction 内实例化；
4. world rejected 只回滚 scene；
5. Job、artifact 和 compiled asset 保留以便修复/复用，除非用户显式删除。

## 15. End-to-End 契约验收

### P-01 Generic modal-3D

- Connector capability → submit → Job succeeded；
- GLB descriptor/hash → bytes；
- Compiler → manifest；
- admission 明确；
- spawn/serialize/offline restore。

### P-01A Composed modal-2d→modal-3D

- 显式strategy与parent Job；
- lossless 2D primary，禁止preview作为默认input；
- 可恢复SAM selection/canonical artifact；
- 3D child可独立重试/换模型；
- 最终GLB经Compiler/Admission；
- lineage贯穿prompt、2D/3D revisions/seeds与candidate。

### P-02 EmbodiedGen Image/Text asset

- bundle 含 GLB/URDF/validation；
- collision/scale evidence 正确映射；
- fallback 明确 provisional；
- asset 可进入 WorldSpec，world admission 独立。

### P-03 Retexture derivation

- 输入 compiled asset hash → retexture Job；
- 输出新 compiled revision；
- `derived_from` 保留；
- 原资产不被覆盖；
- texture budget 决定 admission。

### P-04 Affordance evidence

- face segmentation 被 Compiler 接受；
- semantics/grasp evidence 保留来源；
- 无 joint 时 open/close 不出现；
- Runtime pickup/open 仍按自身 contract 验证。

### P-05 Layout world

- child asset jobs 可恢复/去重；
- layout bundle refs/hash 有效；
- adapter 生成 WorldSpec v2；
- blocked pose/relation 导致 bounded repair 或 rejected；
- rejected scene rollback，但资产保留。

### P-06 Background/room

- environment bundle通过预算/碰撞；
- environment swap 可回滚；
- navigation rebuild；
- per-instance objects按政策 editable；
- serialization/offline restore；
- GS/USD unsupported 项不冒充已加载。

## 16. 兼容与降级

- Connector 不可达：现有本地世界与 compiled assets继续工作；生成入口禁用。
- Provider workflow disabled：历史 Job/result可读，新提交禁用。
- capability major incompatible：不猜字段，要求升级。
- bundle 可视 GLB 有效但 physics缺失：可编译为 provisional。
- affordance schema未知：忽略该 evidence并 warning，不拒绝有效视觉资产，除非用户明确要求功能性资产。
- environment超预算：拒绝环境导入，可保留 artifact并建议 provider lower-budget profile。

## 17. 安全验收清单

- 浏览器存储和 scene JSON 中搜不到 Modal token；
- Connector token有 scope/expiry/revoke；
- 只访问 loopback；
- artifact path不由 provider直接控制；
- hash mismatch、超限、redirect、bundle traversal均拒绝；
- provider JSON不能注入 skill/action；
- prompt/error/labels安全渲染；
- trace可审计但无 Secret；
- scene rollback不触发未授权远端删除或二次提交。
