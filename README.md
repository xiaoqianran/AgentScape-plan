# Modal 2D/3D × EmbodiedGen × AgentScape 实施计划书索引

本目录只包含计划，不包含任何业务代码实现。

## 阅读顺序

1. [`00-master-roadmap.md`](./00-master-roadmap.md)：总目标、边界、依赖顺序、目标架构与总体验收口径。
2. 2D 前置组：`kaggle-inference-hub → modal-build + modal-2d + modal-2d-client`
   - [`group-0-modal-2d/01-kaggle-inference-hub-migration-map.md`](./group-0-modal-2d/01-kaggle-inference-hub-migration-map.md)
   - [`group-0-modal-2d/02-modal-build-2d-plan.md`](./group-0-modal-2d/02-modal-build-2d-plan.md)
   - [`group-0-modal-2d/03-modal-2d-plan.md`](./group-0-modal-2d/03-modal-2d-plan.md)
   - [`group-0-modal-2d/04-modal-2d-client-plan.md`](./group-0-modal-2d/04-modal-2d-client-plan.md)
   - [`group-0-modal-2d/05-group-contract-and-acceptance.md`](./group-0-modal-2d/05-group-contract-and-acceptance.md)
3. 第一组：`modal-3D + modal-3D-client`
   - [`group-1-modal-3d/01-modal-3D-plan.md`](./group-1-modal-3d/01-modal-3D-plan.md)
   - [`group-1-modal-3d/02-modal-3D-client-plan.md`](./group-1-modal-3d/02-modal-3D-client-plan.md)
   - [`group-1-modal-3d/03-group-contract-and-acceptance.md`](./group-1-modal-3d/03-group-contract-and-acceptance.md)
4. 客户端统一：`modal-2d-client × modal-3D-client`
   - [`client-unification/01-modal-2d-3d-client-unification-plan.md`](./client-unification/01-modal-2d-3d-client-unification-plan.md)
5. 第二组：`modal-build + EmbodiedGen`（EmbodiedGen 只读、不修改）
   - [`group-2-embodiedgen/01-modal-build-plan.md`](./group-2-embodiedgen/01-modal-build-plan.md)
   - [`group-2-embodiedgen/02-EmbodiedGen-readonly-stage-map.md`](./group-2-embodiedgen/02-EmbodiedGen-readonly-stage-map.md)
   - [`group-2-embodiedgen/03-workflow-contract-and-acceptance.md`](./group-2-embodiedgen/03-workflow-contract-and-acceptance.md)
6. 第三组：`AgentScape`
   - [`group-3-agentscape/01-AgentScape-integration-plan.md`](./group-3-agentscape/01-AgentScape-integration-plan.md)
   - [`group-3-agentscape/02-provider-artifact-world-contract.md`](./group-3-agentscape/02-provider-artifact-world-contract.md)
   - [`group-3-agentscape/03-dual-generation-strategy-plan.md`](./group-3-agentscape/03-dual-generation-strategy-plan.md)
   - [`group-3-agentscape/04-live-execution-map.md`](./group-3-agentscape/04-live-execution-map.md)：动态实现账本、下一任务切片、依赖 Gate 与并行 ownership；实施 AgentScape 前优先读取。
   - [`group-3-agentscape/05-execution-task-spec-template.md`](./group-3-agentscape/05-execution-task-spec-template.md)：把 Live Map 中的一个切片固化成可独立交付、可验收的 AI/开发任务。
7. 跨项目治理与落地顺序
   - [`cross-cutting/01-master-contracts.md`](./cross-cutting/01-master-contracts.md)
   - [`cross-cutting/02-milestones-testing-risks-rollout.md`](./cross-cutting/02-milestones-testing-risks-rollout.md)

## 总体关系

```text
Kaggle 2D 原型 → modal-build → modal-2d → modal-2d-client ┐
                                                          ├→ 统一 modal-client
个人图片/RGBA → modal-3D → modal-3D-client ───────────────┘        │
                                                                    ├→ 方案A：2D→SAM→3D
EmbodiedGen（只读）→ modal-build阶段化工作流 ────────────────────────┼→ 方案B：Text→3D bundle
                                                                    ▼
                                                              AgentScape
                                               → Compiler / Asset Admission
                                               → WorldSpec / Runtime / Validation
```

## 本轮 CodeGraph 阅读基线

计划基于 2026-08-23 工作区中的实际代码，而不是只按 README 推断：

| 项目 | CodeGraph 索引 | 主要读取范围 |
|---|---:|---|
| `kaggle-inference-hub` | 62 文件 / 3,440 节点 / 14,834 边 | SANA、Z-Image、TripoSR、SQLite 租约队列、Hub 协议、Prompt Studio、Gallery |
| `modal-3D` | 13 文件 / 243 节点 / 428 边 | Gateway、4 个活动 3D Worker、SAM 3.1、统一结果结构 |
| `modal-3D-client` | 27 文件 / 436 节点 / 845 边 | React、Tauri、Python Agent、四模型、SQLite Job/Project、SAM provider/capability、Artifact、GLB Viewer |
| `modal-build` | 17 文件 / 394 节点 / 740 边 | EmbodiedGen 构建产物、生产 Runtime、Image/Text Job API、实验性 retexture Worker/Job、测试 |
| `EmbodiedGen` | 327 文件 / 6,307 节点 / 12,486 边 | 资产、纹理、场景、布局、房间、转换、affordance、仿真，以及现有 dirty Modal/headless 实验审计 |
| `AgentScape` | 196 文件 / 1,687 节点 / 6,773 边 | Adapter、Asset Library、Compiler、World Pipeline、bounded World retry、Runtime、Skills |

表中数字是计划定稿过程中各仓库最近一次完成的 CodeGraph 索引快照；多个 dirty worktree 正在并行变化，实施时应先 `codegraph sync`，再按差异更新“已有/待做”标签。

## 计划中的状态标签

- **已有**：当前代码已存在且至少有本地测试或文档化验证。
- **已有但需产品化**：主链已通，但契约、恢复、错误语义或 UI 尚不完整。
- **待适配**：EmbodiedGen 上游已有能力，`modal-build` 尚未提供对应 Modal 工作流。
- **后置**：依赖前置契约或成本/可行性基准，不应抢先实现。
- **非目标**：当前阶段明确不做，防止范围失控。

## 一个需要先纠正的现状认知

`modal-build/runtime/embodiedgen_v2_l40s.py` 当前不只支持 Image→3D；它还已经提供 `POST /text-jobs`，以 Kolors Text→Image 接入同一条已验证 Image→3D 流水线。最新工作区还加入了实验性 `RetextureWorker`、`POST /jobs/{source_job_id}/retexture`、几何保持校验与本地契约测试。计划因此不会从零重写这三条链，但 retexture 在通过真实 Modal GPU canary、统一 artifact contract、恢复/幂等与独立部署门禁前，仍不能标为生产适配完成。

同样，`modal-3D-client` 已经出现项目数据库、项目式 RGB→SAM→3D UI，以及本地/云端 SAM capability/settings 的第一条切片；其中本地 SAM 仍明确返回 unavailable。`AgentScape` 也已经有缺失资产的一次有界 World retry 和相同 WorldSpec 防重复门。后续计划是在这些真实基础上收敛契约，不重复制造第二套实现。

`EmbodiedGen` 当前 checkout 含用户已有的未提交 Modal/headless 实验。计划把它们作为只读审计输入：不覆盖、不删除、不继续把正式适配写进上游；可复用内容需迁入 `modal-build` wrapper 或版本化 patch inventory。
