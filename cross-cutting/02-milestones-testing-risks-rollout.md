# 总里程碑、测试、风险与发布计划

## 1. 关键路径

```text
M0 契约/基线
 ├→ M1 2D供应链 → M2 modal-2d → M3 modal-2d-client ┐
 ├→ M4 modal-3D/3D-client加固 ───────────────────────┼→ M5 统一客户端
 └→ M6 EmbodiedGen工作流内核 → M7高价值资产能力 ───┘       │
                                                           ├→ M8 AgentScape方案A
                                                           ├→ M9 AgentScape方案B
                                                           └→ M10生成式世界 → M11环境/房间
```

M1～M4、M6在M0后可并行，但M5必须等待2D/3D公共contract稳定；AgentScape正式接入必须等待单一Connector。

## 2. 里程碑总表

| ID | 目标 | 主要仓库 | 依赖 | 相对规模 | Exit Gate |
|---|---|---|---|---|---|
| M0 | 事实冻结与主契约 | 全部 | 无 | L | schemas/fixtures/baseline通过 |
| M1 | 2D构建供应链 | modal-build | M0 | L | SANA pin、Z-Image SM89 release/offline |
| M2 | Modal 2D云端 | modal-2d | M1 | L | 两Worker/Gateway/lossless artifact canary |
| M3 | 2D客户端与Kaggle迁移 | modal-2d-client | M0/M2 | XL | single/batch/restart/Gallery/import |
| M4 | 3D第一组产品化 | modal-3D/client | M0 | XL | 四模型/direct RGBA/recovery/cache/Windows |
| M5 | 2D/3D客户端统一 | unified client | M3/M4 | XL | 单Host/Connector/DB/cache，2D→3D |
| M6 | EmbodiedGen workflow内核 | modal-build | M0 | XL | Image/Text迁移、stage resume/idempotency |
| M7 | EmbodiedGen高价值资产 | modal-build | M6 | XL | retexture/convert/affordance逐Gate |
| M8 | AgentScape方案A | AgentScape/client | M5 | L | 2D→SAM→3D→Compiler/Admission |
| M9 | AgentScape方案B | AgentScape/modal-build | M5/M6 | L | Text→3D bundle→Compiler/Admission |
| M10 | 生成式世界 | AgentScape/modal-build | M7/M8/M9 | XL | Planner/WorldSpec v2/layout/world admission |
| M11 | 背景/Room/仿真后置 | 全部 | M10 | XL | environment/nav/room可行性与E2E |

相对规模不等于日历时间；未给团队人数、Modal配额和GPU benchmark前不承诺日期。

## 3. M0：事实冻结与契约

交付：

- 六个现有仓库CodeGraph/工作区快照；
- dirty worktree ownership清单，避免覆盖用户改动；
- deployment/model/release/Volume/Secret matrix；
- capability/job/artifact/error/coordinate/evidence schemas；
- Python/TS fixtures；
- 固定prompt/image/RGBA/mesh/URDF/world canaries；
- benchmark记录格式；
-安全威胁模型。

Exit：所有计划仓库能消费至少基础fixtures；没有项目自行发明冲突状态名。

## 4. M1：2D供应链

交付：

- SANA exact model/dependency manifest；
- Z-Image stable-diffusion.cpp exact commit/SM89 immutable release；
- diffusion/LLM/VAE exact revisions/hashes；
- CPU preload/offline GPU proof；
- license/SBOM；
- cold/warm/VRAM/quality canary。

Blocker：未知模型许可、无法固定revision、runtime需要编译/网络，任何一项未解决不进入M2。

## 5. M2：Modal 2D

交付：

- 两独立App/Image/weights；
- CPU Gateway/capabilities；
- strict prompt/size/options/idempotency；
- lossless primary/preview/result manifest；
- artifact hash/validation；
- model-specific canary、disable/rollback。

Shadow：同prompt/seed/profile同时在Kaggle与Modal人工/指标比较，但禁止正式客户端双写。

## 6. M3：2D客户端与Kaggle迁移

交付：

- secure Tauri/Connector基线；
- single/batch parent-child；
- Prompt Pipeline vault/确认/provenance；
- Gallery/Image Detail/Job Center；
- restart/reconcile/local cache；
- Kaggle legacy importer/report；
- 2D→3D neutral handoff contract；
- Windows installer smoke。

Exit：关闭Kaggle也能完成完整2D生成/历史；旧历史不丢。

## 7. M4：3D第一组加固

交付：

- 云端四模型/SAM capability；
- option/input/alpha严格校验；
- direct RGBA与Cloud SAM；
- versioned result/hash/GLB validation；
- SQLite migration/reconcile/cancel/idempotency；
- local artifact cache/Viewer/history；
- Connector pairing/Windows E2E。

Exit：应用/网络重启不重复计费，四模型真实canary。

## 8. M5：统一客户端

交付：

- 中性repository/product ADR；
- 单Tauri/Connector/vault/DB/cache；
- Provider Registry；
- 2D/3D/Jobs/Projects/Pipelines workspaces；
- old DB/cache/vault dry-run/resumable migration；
- 2D primary→SAM→3D零歧义lineage；
- 一个AgentScape pairing facade；
- unified installer与旧app退役计划。

Exit：两旧客户端同时不运行也可完成全部已有能力；旧数据可回滚只读。

## 9. M6：EmbodiedGen工作流内核

交付：

- EmbodiedGen只读commit/release/patch inventory；
- workflow registry；
- stage engine/job events/artifact index；
- idempotency/cancel/resume/cleanup；
- Image/Text现有链迁移；
- ASGI compatibility与Connector RPC；
- fault injection。

Exit：Image/Text输出与性能不回退，control plane重启可恢复，`EmbodiedGen`无适配改动。

## 10. M7：高价值EmbodiedGen资产能力

按独立小Gate：

1. retexture：产品化当前实验性 Worker/派生 Job 切片；
2. convert；
3. affordance/part/grasp evidence。

每项要求独立Image/weights/Secret、artifact schema、canary、cost、AgentScape fixture和rollback。不能等三项全部完成才第一次集成。

## 11. M8/M9：AgentScape双方案

### M8 方案A

- unified Connector；
- modal-2d child、image selection/SAM pause、modal-3D child；
- ArtifactImporter；
- Compiler/Admission；
- retry 3D不重跑2D；
- explicit cost/strategy。

### M9 方案B

- EmbodiedGen Text workflow；
- bundle/hash/dependency；
- URDF/collision/part evidence bridge；
- Compiler/Admission；
- provider-ready与asset-ready分离。

Exit：相同prompt可显式选A/B；不选择不提交；失败不静默跨策略。

## 12. M10：生成式世界

交付：

- Prompt→WorldSpec Planner proposal；
- WorldSpec v2；
- reuse-first/provider policy；
- missing asset parent/child fan-out/dedup/budget；
- EmbodiedGen layout adapter；
- deterministic compose/physics/navigation；
- world admission/rollback/serialization；
- fixed world task evaluation。

Exit：只有world-ready被Agent表述为验证完成；rejected scene回滚，生成资产保留。

## 13. M11：Environment/Room/后置仿真

分开Gate：

- background mesh→budgeted GLB；
- dynamic environment swap/collision/nav rebuild/rollback；
- room Blender/Infinigen可行性；
- per-instance environment bundle；
- 3DGS只保留artifact，renderer另立项；
- SAPIEN/Genesis/softbody仅在明确消费者后上线。

## 14. 测试层级

### T0 静态/供应链

- exact commits/revisions/hashes；
- no-clobber release；
- patch target/apply；
- license/SBOM；
- runtime no compiler/network；
- capability/deployment name uniqueness。

### T1 纯单元

- schema/normalize/state transition；
- path/MIME/hash/alpha/image/mesh；
- idempotency/TTL/lease；
- coordinates/evidence/error；
- DB migration。

### T2 组件/Mock

- Modal Function/Volume/Dict；
- Worker child process；
- Connector provider/reconcile；
- SSE/poll；
- Compiler passes/admission；
- workflow stage/fan-out/failure。

### T3 GPU/Deployment Canary

- 每模型/工作流固定输入；
- cold/warm；
- output independent parse/hash；
- OOM/timeout/missing weights/commit failure；
- disabled/rollback。

### T4 Desktop E2E

- 干净Windows；
- vault/sidecar；
- single/batch/restart；
- 2D/3D/Prompt/Viewer；
- DB/cache migration；
- installer upgrade/uninstall；
- non-ASCII/non-admin。

### T5 AgentScape E2E

- pairing/revoke；
- strategy A/B；
- artifact bytes/hash；
- compiler/admission；
- world pipeline/rollback；
- offline restore。

### T6 Security/Fault/Performance

- Secret scan；
- session scope/origin/path traversal/redirect；
- malformed/oversize/bundle bomb；
- process/network/restart/cancel race；
- GPU/cost/disk/memory/Web budget；
- long workflow cleanup race。

## 15. 必需测试矩阵

| 交付 | T0 | T1 | T2 | T3 | T4 | T5 | T6 |
|---|---:|---:|---:|---:|---:|---:|---:|
| modal-build 2D | ✓ | ✓ | ✓ | ✓ |  |  | ✓ |
| modal-2d | ✓ | ✓ | ✓ | ✓ | via client | fixture | ✓ |
| modal-3D | ✓ | ✓ | ✓ | ✓ | via client | fixture | ✓ |
| unified client |  | ✓ | ✓ | via provider | ✓ | ✓ | ✓ |
| EmbodiedGen kernel | ✓ | ✓ | ✓ | ✓ | via connector | fixture | ✓ |
| AgentScape |  | ✓ | ✓ | fixture | browser E2E | ✓ | ✓ |

## 16. 固定评测集

### 2D

中文/英文、人物/物体/室内、细线、文字约束、透明/反射意图、不同aspect；固定prompt/seed/profile。

### Image→3D

普通RGB、canonical RGBA、薄结构、对称、复杂纹理、空alpha、全opaque、超大/损坏。

### EmbodiedGen

官方image/text/texture/affordance demo；tiny/standard scene；最小layout；minimalist room。

### AgentScape

prop、support/container、furniture、articulated-looking、multi-asset ON/NEAR、navigation blocker、environment swap。

每个结果保存versions/seeds/hashes/raw metrics/admission/reason和人工评分。

## 17. 风险登记

| 风险 | 影响 | 预警 | 缓解/退出 |
|---|---|---|---|
| 范围同时扩到2D/3D/world | 关键链长期不落地 | 多个experimental无E2E | 按M0～M11 Gate；每次只开放通过canary能力 |
| 用户dirty worktree被覆盖 | 丢失工作 | status有大量modified/untracked | 实施前ownership清单；小patch；不reset/checkout |
| Z-Image SM75误用于SM89 | crash/错误结果 | binary架构不明 | modal-build重建；manifest/hash/clean runtime canary |
| HF/上游revision浮动 | 不可复现/供应链风险 | snapshot无commit | exact revision/hash/marker/offline |
| 模型许可不允许分发 | 发布阻断 | license未知/gated | SBOM/license Gate；权重用户workspace下载 |
| 2D有损图伤害3D | 几何/边缘质量 | preview被作为input | lossless primary role；类型/contract/UI门 |
| 中性Volume迁移困难 | 跨服务复制/路径漂移 | `modal-3d-artifacts`硬编码 | descriptor identity；多location；分阶段新Volume |
| 两客户端重复基础设施 | credential/job分叉 | 两sidecar/DB运行 | M5硬Gate；单Connector先于AgentScape |
| DB迁移损坏历史 | 用户数据丢失 | migration异常/ID冲突 | dry-run/source backup/resume/report/只读rollback |
| Secret从Kaggle遗留泄露 | 安全事件 | 默认token/localStorage/Tunnel | rotate/revoke；不导入token；vault/session；secret scan |
| Modal结果过期 | 成功后无法下载 | OutputExpired | 立即本地cache/hash；remote/local状态分开 |
| cancel语义虚假 | 计费/误导 | UI立即显示cancelled | cancel_requested/ack/final race tests |
| EmbodiedGen大runtime继续膨胀 | 冷启动/回归 | 每能力加同一文件/Image | workflow kernel + capability family拆App/Image |
| EmbodiedGen上游patch漂移 | 升级失败/算法变化 | patch fuzzy apply | exact commit/file hash/minimal patch/re-audit |
| Room/Blender不适合Modal | 成本/磁盘/超时 | child process/volume benchmark失败 | 独立可行性Gate；不阻塞单资产/world core |
| Provider evidence被过度信任 | Agent动作虚假 | semantic变action | evidence等级；Compiler/Runtime验证；fail-closed |
| AgentScape Web budget超限 | crash/低性能 | Pixal/room大纹理网格 | ResourceBudget、provider low-budget profile、reject/decimate |
| World fan-out失控 | 高额费用 | 每对象双策略生成 | reuse-first、global budget、显式strategy、concurrency limit |
| Cleanup误删长任务 | Job不可恢复 | heartbeat/TTL竞态 | active lease/stalled状态/fault test/分级TTL |
| Prompt隐私泄露 | 敏感文本外泄 | logs/errors记录全文 | vault、hash/length logs、retention、diagnostic脱敏 |
| 缺客户端/worker测试 | 回归频繁 | CodeGraph找不到覆盖 | M0 fixtures；T1/T2先于重构；GPU canary |

## 18. 发布阶梯

每个provider/workflow遵循：

1. `development`：本地/CI，不发布capability；
2. `disabled`：部署完成，direct canary；
3. `shadow`：与旧路径对比，不作为默认；
4. `allowlist`：指定用户/客户端；
5. `available`：默认可选；
6. `degraded`：已知问题，限制新提交；
7. `deprecated`：历史可读，提示迁移；
8. `disabled`：停止新提交，旧Job/result可读。

状态必须由capability发布，客户端不硬编码猜测。

## 19. Rollback

### 云端

- 每模型/能力族独立revision；
- Gateway capability disable先于回滚；
- 旧Worker/release保留兼容窗口；
- 新Job停止，旧Job按原revision查询；
- 不覆盖immutable artifacts。

### 客户端

- DB migration前备份；
- old schema read-only fallback；
- installer保留app-data；
- unified client失败可打开旧客户端只读数据，不双写；
- credential migration可撤销mapping但不在React暴露value。

### AgentScape

- provider关闭不影响compiled assets/scene offline；
- rejected world回滚scene；
- generation Job/artifact保留；
- WorldSpec v2 serializer保留v1 reader。

## 20. Kaggle退役

1. Modal shadow达到质量/功能基线；
2. 默认切Modal，Kaggle allowlist；
3. legacy history完整导入；
4. 禁止新Kaggle task；
5. 等待queue/inflight清空；
6. 最终snapshot/report；
7. 撤销Tunnel/token；
8. Notebook/repo标legacy/research；
9. 保留tag，不删除用户outputs。

## 21. 上线观测

必须能按provider/model/workflow/revision查看：

- submit/accept/fail/cancel/expire；
- queue/load/infer/postprocess/commit/download/compile；
- cold/warm、GPU peak、bytes、cost class；
- hash/parse/validation failure；
- local reconcile/cache hit/eviction；
- AgentScape admission reason；
- world ready/provisional/rejected；
- strategy A/B成功/成本/质量对比。

不以总HTTP 200率代替端到端成功率。

## 22. 最终验收清单

- Kaggle SANA/Z-Image已迁移，历史可读，正式链不依赖Tunnel；
- modal-build所有native/model来源固定且runtime offline；
- modal-2d lossless primary与modal-3D四模型可用；
- 统一客户端只有一个Host/Connector/vault/DB/cache；
- 2D→3D parent-child恢复/重试/lineage完整；
- EmbodiedGen只读且Image/Text workflow内核稳定；
- retexture/convert/affordance按Gate上线；
- AgentScape两策略显式并都过Compiler/Admission；
- WorldSpec/world pipeline/rollback/offline restore通过；
- environment/room只在独立可行性Gate后上线；
- security/fault/performance/installer测试有证据；
- 每个能力能独立disable/rollback，旧Job/artifact仍可读。
