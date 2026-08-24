# 跨项目主契约

## 1. 适用范围与权威顺序

本文是以下项目的协议总纲：

- `modal-2d`；
- `modal-3D`；
- `modal-build` EmbodiedGen workflows；
- 统一 `modal-client`；
- `AgentScape`。

权威顺序：

1. 本文的跨项目语义；
2. provider自己的versioned capability；
3. 各项目详细计划；
4. UI文案/README。

出现冲突时不能靠客户端猜；必须升级/修订contract和fixtures。

## 2. 契约层级

```text
Provider Capability
  └── Operation Request
       └── Remote Execution / Local Job Projection
            └── Events / Result
                 └── Artifact Descriptor / Bundle
                      └── Local Project / Pipeline Lineage
                           └── AgentScape Compiler / Admission / World
```

身份、状态、存储位置和readiness分层，不能用一个`path`或`status=done`替代。

## 3. 命名规范

### Provider ID

- `modal-2d`；
- `modal-3d`；
- `embodiedgen`；
- `kaggle-legacy`（只读迁移）；
- `local-catalog`。

### Operation/Workflow ID

格式：`<provider>.<domain>.<operation>.v<major>`。

示例：

- `modal-2d.image.text_to_image.v1`；
- `modal-3d.image.segment.v1`；
- `modal-3d.asset.image_to_3d.v1`；
- `embodiedgen.asset.text_to_3d.v1`；
- `embodiedgen.asset.retexture.v1`；
- `embodiedgen.scene.layout.v1`。

模型ID与operation分开。模型升级不改变operation ID，除非输入/输出语义破坏兼容。

### Job/Artifact/Project ID

- 本地ID由统一Connector生成，opaque且全局唯一；
- remote call/job ID只保存在location；
- artifact ID不等于Volume path；
- content hash不直接当用户可变label；
- parent/child使用显式relation，不在ID字符串编码层级。

## 4. 版本规则

每个契约记录：

- contract name/version；
- producer implementation revision；
- capability hash；
- compatibility range；
- generated timestamp。

规则：

1. 增加可选字段可升minor。
2. 增加必需字段、删除字段、改义、改变状态语义升major。
3. Consumer遇到未知可选字段保留/忽略；遇到未知主版本拒绝操作。
4. Job保存提交时的contract/capability hash，恢复时用原effective options解释。
5. Artifact成功后schema/revision不可变；衍生处理生成新artifact/bundle。

## 5. Capability Contract

### 服务级

- provider/service ID；
- contract/implementation/deployment revision；
- health/status；
- supported Connector range；
- generated/expires/cache policy。

### Operation级

- stable ID/display/category/status；
- input schemas/limits；
- option JSON Schema/profiles/defaults；
- output roles/required/optional；
- stages；
- async/idempotency/cancel/resume support；
- duration/cost/resource class；
- Secret/license prerequisites；
- retention；
- consumer hints/warnings/deprecation。

### Model/backend级

- stable model ID/display；
- upstream/model/release/weights revision；
- input/output特性；
- profile映射；
- availability；
- historical performance class（非SLA）。

只有部署并通过canary的能力可`available`。只有preload、Notebook或代码路径存在时应`experimental/disabled`。

## 6. Connector Session Contract

两类会话：

### Desktop主会话

- Tauri每次启动生成高熵token；
- 只通过invoke交给主WebView；
- full product scope；
- 生命周期绑定sidecar/desktop。

### AgentScape配对会话

- 用户在desktop确认；
- 独立短期token；
- origin/client identity绑定；
- scopes：capabilities/jobs/artifacts等；
- issued/expires/revoke/audit；
- 不含credential read/write与任意Modal调用。

共同规则：只监听loopback、每请求认证、CORS不替代认证、token不进DB普通字段/log/scene/localStorage。

## 7. Request Envelope

通用字段：

- local request ID；
- idempotency key；
- provider/operation/version；
- typed inputs；
- requested profile/options/seed；
- required output roles；
- retention；
- parent request/job/project；
- capability/contract hash；
- safe user metadata。

Provider计算：

- canonical request hash；
- effective profile/options；
- selected backend/revision；
- accepted timestamp。

幂等规则：相同key+相同canonical payload返回同一执行；相同key+不同payload返回冲突。Retry默认新Job并带`retry_of`，stage resume例外。

## 8. Job Contract

### 本地Job是用户身份

必需字段：

- local Job ID；
- provider/operation/kind；
- request hash/idempotency；
- remote kind/ID/deployment；
- status/stage/progress semantics；
- attempt；
- parent/children/relations；
- created/submitted/started/updated/completed；
- error；
- result/artifact summary；
- project/owner/source app；
- contract/capability versions。

### 基础状态

```text
draft → preparing/uploading → queued → running → downloading → validating → succeeded
                                  │         │             │
                                  ├→ cancel_requested → cancelled
                                  ├→ connection_required
                                  ├→ failed
                                  └→ expired
```

Workflow可增加`awaiting_user_selection`、`stalled`，但必须定义可恢复条件。

### 状态事实

- `connection_required`不是远端失败；
- `cancel_requested`不是cancelled；
- provider `succeeded`只代表结果可取；本地`succeeded`要求required artifact校验/cache完成；
- AgentScape另有asset/world admission，不能复用Job status；
- 超时轮询不直接变failed；
- success/cancel竞态按远端最终事实记录并写event。

## 9. Job Event Contract

事件追加式：

- monotonic sequence；
- local/remote timestamp；
- job/stage/attempt；
- type；
- old/new status；
- progress语义；
- safe message/details；
- correlation IDs。

SSE断线用last sequence恢复；poll是fallback。事件日志不放Secret、prompt全文、图片bytes或远端traceback。

## 10. Parent/Child 与 Pipeline

Relation类型：

- `contains`（batch）；
- `stage-child`；
- `retry-of`；
- `fallback-of`；
- `derived-from`；
- `world-asset-request`。

Parent状态由policy聚合，不隐式取消已成功children。Pipeline记录：stage ID、input/output artifact IDs、child Jobs、interaction pause、retry boundary、status和lineage。

2D→3D示例：Prompt→2D child→primary→SAM selection/canonical→3D child→GLB。3D失败从3D stage重试，不重跑前面成功且兼容的checkpoint。

## 11. Artifact Descriptor

### Identity与内容

- opaque artifact ID；
- role/type/schema；
- display name（不用于path）；
- MIME/format；
- bytes/SHA-256；
- producer job/stage/attempt/revision；
- created/expires；
- validation/readiness/warnings。

### Location

一个artifact可有多个location：

- remote provider/Volume path；
- local cache path；
- AgentScape IndexedDB compiled key；
- legacy source path。

location有scope、verified time、expiry和state。Path不成为artifact identity，也不跨信任边界直接暴露。

### 发布规则

1. 写attempt临时位置；
2. 格式/结构验证；
3. 计算hash/bytes；
4. commit/原子rename；
5. 发布descriptor/index；
6. 才允许Job success。

### Retention/Lease

- remote TTL与local cache policy分开；
- active Job/project/favorite/open Viewer/AgentScape transfer持有lease；
- cleanup不删除leased/active；
- eviction有报告，不静默破坏scene；
- 删除location不等于删除artifact lineage。

## 12. Prompt Record

字段：

- source text；
- transformation chain：mode/provider/model/revision；
- processed candidates；
- final confirmed text；
- edited/stale adapter；
- target capability hash；
- timestamps/privacy/retention；
- prompt hash。

Prompt Pipeline Secret独立保存。普通日志只记录长度/hash/operation，不记录全文。AI优化后必须确认才产生GPU request。

## 13. 2D Image Contract

### Primary image

- role=`primary-image`；
- lossless PNG或声明为lossless的格式；
- width/height/color mode/alpha/ICC policy；
- model/revision/seed/effective options；
- downstream compatibility。

### Preview

- role=`preview-image`；
- 可有损；
- `derived_from` primary hash；
- 只供UI；
- 默认禁止作为Image→3D输入。

### Legacy

Kaggle WebP quality 90标`legacy-lossy`，可由用户显式使用，但不能伪装lossless。

## 14. SAM/Canonical Image Contract

Selection：scene/selection IDs、source hash、concept/boxes、candidate list、bbox/mask stats、model revision、expiry。

Canonical：source/selection/candidate、RGBA descriptor、foreground bounds/alpha stats、output size、request hash、expiry。

`awaiting_user_selection`是合法pipeline状态。Candidate过期返回可执行`resegment`，不模糊为internal error。

## 15. 3D Visual Artifact

GLB descriptor扩展：

- geometry count/bounds；
- materials/textures/dimensions/estimated VRAM；
- unit/axis/source transform；
- parse/non-empty/finite validation；
- Web budget status；
- model/seed/input artifact lineage。

GLB有效不代表physics/affordance ready。

## 16. Asset Bundle

Bundle包含：

- bundle ID/type/version/readiness；
- primary visual GLB；
- optional OBJ/MTL/texture/collision/URDF/MJCF/USD/GS/video；
- part segmentation/semantics/grasp evidence；
- coordinate/unit/transform；
- dependency graph；
- validation；
- provenance/warnings。

成功member不可覆盖。Retexture/convert/affordance产生新derived bundle。

## 17. Evidence Contract

每项声明source：

- measured；
- generated；
- inferred；
- default；
- provider-validated；
- simulator-validated；
- runtime-verified；
- unknown。

示例：

- semantic part不证明joint；
- SAPIEN grasp验证不等于AgentScape/Rapier抓取验证；
- visual mesh fallback不等于accurate collision；
- default mass=1不等于测量质量；
- provider ready不等于AgentScape asset ready。

## 18. 坐标、单位与物理

所有3D bundle/environment声明：

- length unit，canonical目标meter；
- handedness/up/front axes；
- source→canonical transform；
- visual/collision local transform；
- bounds/center/ground offset；
- scale evidence；
- mass/friction/inertia evidence；
- body intent；
- part/joint/grasp frames。

旋转跨项目使用明确quaternion顺序；若接Euler必须记录order和degree/radian。未知坐标可保留视觉asset但admission provisional。

## 19. Environment/World Bundle

Environment必需：visual GLB、collision representation、bounds/ground/unit/axis、validation/resource summary、lineage。

可选：spawn/camera/navigation hints、per-instance objects、pano/preview、GS reference。Hints不是Runtime truth。

World bundle：layout proposal、asset refs/hashes/poses、environment ref、relations、robot/entity、coordinate metadata、validation/provenance。

进入AgentScape后必须转WorldSpec并执行Runtime pipeline。

## 20. Error Contract

字段：

- stable code；
- category；
- stage/attempt；
- retryable；
- user-safe message；
- correlation ID；
- safe details；
- optional cause chain codes。

类别：auth、capability、request/input、queue/network、model/weight、stage/workflow、artifact、convert/sim、cancel、internal。

原则：远端traceback/HTTP body/Secret/绝对path不直接回UI；UI同时显示稳定code和下一步。

## 21. Readiness/Admission

不同层独立：

- Provider workflow：ready/provisional/rejected；
- Artifact integrity：valid/invalid；
- AgentScape Compiler quality：ready/provisional/rejected；
- Runtime verification；
- World admission。

聚合优先级：任一required层rejected→rejected；无hard但有unknown/fallback→provisional；要求的所有层通过→ready。

禁止把`HTTP 200`、`Job succeeded`、`provider ready`当作最终world-ready。

## 22. 安全与隐私

- Browser不保存Modal/HF/Tencent/LLM Secret；
- OS vault→Tauri→Python memory，不回传React；
- loopback/session/scope/expiry/revoke；
- Artifact path/job ownership/range/hash；
- 输入MIME magic、size/pixels、bundle traversal/JSON depth；
- prompt/label/error安全渲染；
- GPU runtime只获得最小Secret且尽量offline；
- diagnostics脱敏；
- external submit/cancel/import有audit；
- Kaggle legacy token不导入新vault。

## 23. 兼容 Fixtures

主fixtures至少包括：

- modal-2d SANA/Z-Image result；
- modal-3D四模型result；
- SAM selection/canonical；
- EmbodiedGen Image/Text bundle；
- retexture/convert/affordance bundle；
- background/layout/room bundle；
- malformed/unknown-major/hash mismatch/path traversal；
- local Job restart/cancel race/expired；
- AgentScape admission ready/provisional/rejected。

Python与TypeScript consumer对同一JSON fixture round-trip；CI检查required字段和向后兼容。

## 24. Contract 变更流程

1. 提交问题/用例和受影响consumer；
2. 更新本文与schema；
3. 添加new/old/malformed fixtures；
4. provider与consumer contract tests；
5. capability compatibility window；
6. shadow/canary；
7. 发布minor/major；
8. deprecated deadline与rollback；
9. 旧Job/artifact继续可读。

不能先改某个UI字段，再让其他仓库追着猜。
