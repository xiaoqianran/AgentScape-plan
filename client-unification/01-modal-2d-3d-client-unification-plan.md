# `modal-2d-client` × `modal-3D-client` 彻底统一计划

> **状态：历史迁移计划，产品拓扑已被 `../01-product-architecture-replan.md` 覆盖。** 其中“一个中央 Local Modal Connector 作为 AgentScape 必经层”的目标已废止。仍可复用：单一桌面产品、2D/3D UX 合并、Local Library/Project/Cache、凭据安全、migration 与 contract fixtures；这些能力现在归属 **Local Companion**。

## 1. 最终目标

“打通”不只是一张图片卡片跳转到另一个本地端口。最终只能有一个用户级产品边界：

- 一个 Tauri桌面Host；
- 一个 Python Local Modal Connector进程；
- 一份 Modal credential；
- 一份 SQLite Job/Event/Artifact/Project数据库；
- 一个内容寻址cache；
- 一套capability/provider/错误/配对contract；
- 2D、3D、Pipeline、AgentScape四个工作区；
- 一个安装、升级、诊断和卸载流程。

文中用中性逻辑名 `modal-client` 表示最终产品。物理仓库可由现有 `modal-3D-client` 演进并重命名，或在 Gate U1 建新中性仓；无论选择哪种，部署结果不能保留两个sidecar和两份DB。

## 2. 为什么不能长期“双客户端互调”

两个完整客户端会导致：

- 两份Windows credential entry与不一致删除语义；
- 两个随机loopback Agent与token；
- 两份SQLite Job和artifact cache；
- 同一Modal FunctionCall在不同历史中不可追溯；
- 2D→3D必须跨进程复制bytes；
- 安装升级/DB migration相互独立；
- AgentScape不知道连接哪一个Connector；
- Prompt Pipeline/API key重复；
- 清理/lease可能删除另一个客户端正在用的artifact。

因此跨客户端HTTP deep-link只允许作为短过渡，不是最终架构。

## 3. 最终模块

```text
modal-client/
  apps/desktop-tauri/
  connector/
    auth/
    capabilities/
    providers/
      modal_2d
      modal_3d
      embodiedgen
    jobs/
    artifacts/
    projects/
    prompt_pipeline/
    agentscape_bridge/
  ui/
    shell/
    workspaces/2d/
    workspaces/3d/
    workspaces/pipelines/
    workspaces/jobs/
    workspaces/projects/
    settings/
  contracts/
  migrations/
```

这只是责任结构，不要求采用monorepo工具。关键是模块只有一份事实实现。

## 4. 统一实体模型

### Provider

`modal-2d`、`modal-3d`、`embodiedgen`、未来local provider；各自capability但同一registry。

### Operation

示例：

- `modal-2d.image.text_to_image.v1`；
- `modal-3d.image.segment.v1`；
- `modal-3d.asset.image_to_3d.v1`；
- `embodiedgen.asset.text_to_3d.v1`；
- `embodiedgen.asset.retexture.v1`。

### Job

统一local ID、provider/operation、request hash、remote identity、status/stage/error、parent/children、timestamps。

### Artifact

统一opaque ID、role、hash/MIME/bytes、remote/local location、lineage、lease/retention。

### Project/Pipeline

有向无环衍生关系：Prompt→2D image→SAM selection→canonical RGBA→3D GLB→compiled AgentScape asset。每个node引用immutable artifact/job，不复制业务真值。

## 5. 数据库合并

目标表族：

- providers/capability_cache；
- jobs/job_events/job_relations；
- artifacts/artifact_locations/artifact_leases；
- projects/project_nodes/project_edges；
- connector_sessions/audit；
- metadata/migrations/import_reports。

迁移规则：

1. 先给两客户端现有DB定义source schema/version。
2. 新DB生成新中性ID；旧ID保存alias，避免碰撞。
3. remote call ID只作为provider location，不当primary key。
4. artifact按hash去重，但保留多个Job lineage/location。
5. terminal Job不重新查询也可导入；non-terminal需用户credential后reconcile。
6. result JSON先schema normalize，未知字段保留raw provenance。
7. 迁移事务可resume；源DB只读并备份。
8. migration report可核对jobs/artifacts/missing/corrupt/duplicate。

## 6. Artifact统一与零复制Handoff

同一local cache中的2D primary在3D使用时：

- 不复制本地bytes；增加pipeline reference/lease；
- 云端若共用中性Volume，传descriptor/path；
- 若Volume不同，Connector按hash上传目标namespace并记录第二location；
- 目标上传成功不改变artifact identity；
- downstream完成后释放临时lease，cache仍按project/favorite policy保留；
- preview与primary不同artifact，类型系统防止误选。

## 7. UI统一

### Shell

- connection/provider status；
- project switcher；
- global Job Center；
- disk/cache；
- settings/vault/AgentScape pairing。

### 2D Studio

- Prompt Studio、single/batch、Gallery、Image Detail。

### 3D Studio

- image/RGBA input、SAM candidates、模型profile、GLB Viewer。

### Pipeline Studio

- 可视化但不强制图编辑器：至少展示stage、input/output、lineage、retry boundary；
- `Text→2D→SAM→3D`；
- EmbodiedGen Text→3D；
- retexture/convert/affordance衍生。

### Job Center

- 统一2D/3D/EmbodiedGen parent-child；
- provider/stage/status/cancel/retry；
- connection_required；
- artifact下载/验证。

## 8. Prompt与输入流

统一Prompt record：

- source；
- transformations（mode/provider/revision/time）；
- confirmed final；
- edited flag；
- target capability hash；
- privacy/retention。

同一确认prompt可派生两条方案：2D→3D与EmbodiedGen Text→3D；不得因为共用输入就在后台同时计费提交。

2D image input、用户上传image、canonical RGBA都进入同一artifact系统；工作区只改变展示与可用operation。

## 9. Connector统一

最终Python Agent：

- Tauri主session；
- Modal credential/client；
- capability aggregator；
- provider adapters；
- Job scheduler/reconciler；
- Artifact/cache/project；
- Prompt Pipeline；
- AgentScape scoped pairing；
- diagnostics。

任何UI工作区都只调用产品级Connector API。React不直接import Modal SDK、不读Secret、不访问Volume。

## 10. Tauri与凭据统一

- 保留一个app identity和vault service/account命名；
- 迁移发现旧2D/3D credential entries，用户确认后合并；
- credential值不经过React；
- 删除持久credential、断开当前session、取消Job语义分开；
- 单一sidecar lifecycle/parent PID/log；
- 单一app-data目录；
- 旧app卸载不删除已迁移数据；
- installer检测同时安装的旧客户端并提供迁移，不强制破坏性删除。

## 11. API统一

概念资源：

- `/v1/providers`；
- `/v1/capabilities`；
- `/v1/jobs`；
- `/v1/artifacts`；
- `/v1/projects`；
- `/v1/pipelines`；
- `/v1/prompt-transformations`；
- `/connector/v1/*`（AgentScape scoped facade）。

旧 `/v1/models`、`/v1/generations`、SAM endpoints在compatibility facade保留一个版本窗口，UI新代码只用统一resource。

## 12. 工作包

### U-00：Architecture Decision与freeze

- 选择最终物理仓库/名称；
- 冻结2D/3D现有E2E；
- 列出两边app identity、vault、DB、cache、endpoint；
- 定义不覆盖用户dirty worktree的实施策略。

推荐：以 `modal-3D-client` 已验证的Tauri/Agent安全基线为迁移源，在中性目录/名称完成模块化后再切产品名；不在旧3D命名上永久承载2D语义。

### U-01：Contract包

- JSON schemas/fixtures；
- Python/TypeScript types；
- Job/Artifact/Capability/Error/Pipeline；
- compatibility matrix；
- 两客户端consumer tests。

### U-02：Connector core抽取

- modal client/auth；
- durable Job/reconcile；
- artifact/cache；
- capability registry；
- provider adapter interface；
- session middleware；
- 不改变现有3D行为。

### U-03：2D Provider/Workspace接入同一core

- 2D submit/job/result；
- Prompt Pipeline；
- Gallery；
- batch parent/child；
- lossless artifact。

### U-04：3D Workspace迁移

- SAM/Direct RGBA；
- 四模型；
- Viewer；
- 历史Job/artifact；
- 所有endpoint切统一API。

### U-05：Pipeline/Project graph

- 2D→3D parent-child；
- interactive SAM pause/resume；
- retry从失败stage；
- immutable lineage；
- project保存/打开。

### U-06：DB/cache/vault migration

- dry run报告；
- source backup；
- resumable migration；
- hash dedup；
- non-terminal reconcile；
- vault merge；
- rollback到旧客户端只读模式。

### U-07：AgentScape facade

- 一个配对入口；
- provider capabilities包括2D/3D/EmbodiedGen；
- Job/artifact统一；
- scope/audit/revoke；
- 不让AgentScape知道内部有过两个客户端。

### U-08：安装与切换

- unified installer；
- side-by-side detection；
- migrate-first, switch-later；
- desktop shortcuts/file association；
- old apps标只读/迁移完成；
- 明确数据保留/卸载选项。

### U-09：删除重复实现

只有统一E2E和回滚窗口通过后：

- 停止旧sidecar；
- 删除重复vault/DB/cache代码；
- 旧UI只保留迁移提示或归档tag；
- 文档/脚本指向统一client；
- 不删除用户源数据。

## 13. 迁移顺序与门禁

### U0：Schema兼容

两客户端fixtures一致，现有3D与新2D都可用统一Job/Artifact模型。

### U1：单Connector双Provider

同一Agent会话可提交2D和3D，单一DB/cache；UI可暂时两个shell。

### U2：2D→3D Pipeline

lossless primary零歧义handoff、interactive SAM、parent-child恢复。

### U3：单一桌面Shell

2D/3D/Jobs/Projects统一，vault/app-data迁移。

### U4：AgentScape与旧客户端退役

一个Connector facade，旧客户端进入只读/归档。

## 14. 测试矩阵

**Contract**：Python/TS round-trip、old/new fixtures、major incompatible降级。

**DB**：空、仅2D、仅3D、两者、ID冲突、duplicate hash、corrupt row、non-terminal、migration中断/resume/rollback。

**Artifact**：同hash多location、primary/preview、lease/eviction、Volume bridge、offline restore。

**Credential**：两旧entry、一个entry、删除/断开、非Windows、升级不泄露React。

**Pipeline**：2D成功→SAM pause→3D；2D失败；candidate过期；3D失败重试不重跑2D；取消竞态；重启每个stage。

**UI**：workspace切换不丢state；global Job Center；deep link；project restore；无Secret/localStorage token。

**Installer**：干净安装、旧2D、旧3D、两者并存、非ASCII路径、非管理员、卸载保留数据。

**AgentScape**：一个pairing、两种生成方案、artifact bytes、revoke/offline compiled资产。

## 15. 完成定义

- 只有一个安装应用、Tauri Host、sidecar、vault、DB、cache；
- 2D/3D/EmbodiedGen是provider/workspace而非独立基础设施；
- 2D→3D使用immutable primary artifact与完整lineage；
- 3D失败可从3D stage重试，不重新生成2D；
- 两旧客户端数据可dry-run、迁移、核对、回滚；
- AgentScape只配对一个Connector；
- unified app离线可打开历史图片/GLB/项目；
- old apps退役不删除用户数据；
- 无默认token、browser Secret、公开Volume path或重复credential实现。

## 16. 非目标

- 通过iframe或两个localhost页面伪装统一；
- 永久运行两个sidecar；
- 用文件复制代替Artifact identity；
- 一次破坏性搬迁所有旧数据；
- 在统一阶段同时加入大量新模型/编辑器功能。
