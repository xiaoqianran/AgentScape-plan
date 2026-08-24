# 第一组执行状态与下一任务：`modal-3D + modal-3D-client`

> 本文件是第一组唯一的**动态执行状态**入口。`01-modal-3D-plan.md`、`02-modal-3D-client-plan.md`、`03-group-contract-and-acceptance.md` 是设计与验收基线，不随每次实现进度反复改写。
>
> 更新规则：代码事实、CI、Modal 实测或 Release 状态发生变化后，先更新真实仓库，再同步本文件。任何“完成”都必须附证据；计划、推测和代码存在本身都不算完成。

## 1. 当前事实快照

快照日期：2026-08-24。

| 仓库 | 当前 HEAD | 事实 |
|---|---|---|
| `modal-3D` | `5fb839b` | 4 个活动 Generation Worker + Cloud SAM 3.1；`modal-3d.capabilities.v1` 已上线；Gateway Worker 路由与 option 校验均消费同一 capability contract |
| `modal-3D-client` | `d436eb7` | Durable Job、Project Workspace、Cloud/Local/Auto SAM、原生 GLB 导出已落地；模型/profile/status/warm reference 已改为云端 capability 驱动并带 last-known-good cache |
| `AgentScape-plan` | 更新本文件前 `9f9fc24` | 第一组动态 tracker 已存在；本次同步 P0-B capability 单一事实源的真实完成证据 |

已验证证据：

- `modal-3D-client` HEAD `d436eb7` 的 GitHub Windows run `32697639923`：completed / success；
- `modal-3D` commit `5fb839b`：新增 `modal-3d.capabilities.v1` 与 11 个纯 CPU contract tests；
- `modal-3d-gateway` 已重新部署并通过真实 Modal SDK `capabilities()` canary；4 个模型顺序、recommended profile、SAM operations 均与 contract 一致；
- Gateway 真实负向 canary：Pixal3D 未声明的 `texture_size` 在 CPU router 层被拒绝，未进入 Worker spawn；
- `modal-3D-client` 已有 23 个 Python unit/lifecycle tests；真实 canary 验证 remote capability -> 原子 cache -> 主动断开 Modal -> last-known-good cache 仍可投影四模型/profile options；
- Windows CI 已覆盖 frontend、PyInstaller Agent smoke、Tauri、NSIS installer 与 installer upload；
- Durable Job P0-A 已有 14 个 Python unit/lifecycle tests，并在 Windows CI 的 Python Agent 步骤中执行；
- 真实 Modal CPU cancel canary：`cancel_requested -> cancelled`，以及“远端已先完成 + 随后 cancel -> succeeded”两种 race 均通过；
- 真实 Modal reconnect canary：本地 Modal client 断开后 Job 进入 `connection_required`，恢复同一 client 后使用原 local Job ID + 原 `remote_call_id` 得到 `succeeded`，无二次提交；
- `local-sam-runtime-v1` 是正式 GitHub Release，bootstrap `34,709,811` bytes；
- bootstrap SHA256：`1392402accd8985cbecabe62f766a847aee613c868e3dfb5191253bb4db0d73d`；
- SAM 3.1 checkpoint 来自 `modal-3d-sam31-weights/sam31/sam3.1_multiplex.pt`，`3,502,755,717` bytes，SHA256 `0567debeec80ba4ac6369540c6c248025283cb3ff2b92827509e57e2b3541cb6`；
- exact client `local_sam_runtime/engine.py` 已在 Modal L40S 完成 `segment -> positive-box refine -> materialize`：peak allocated `4.377 GiB`、canonical RGBA `1024 x 1024`；
- 同一 canonical / seed / Agent recommended profile 的四模型真实矩阵全部产出有效 GLB：FastSAM3D++、Hermit-TRELLIS2++、Hunyuan2.1++、Pixal3D；
- Local SAM 尚缺唯一组合环境证据：真实 Windows x86_64 + NVIDIA GPU 的完整端到端推理。

### 当前必须承认的代码事实

这些事实直接决定下一阶段优先级：

1. `modal-3D-client/agent/models.py` 已不再保存任何具体模型事实；它现在只负责 `modal-3d.capabilities.v1` 校验、last-known-good cache、UI projection 与 profile 展开。React 默认模型也不再写死具体 model ID。
2. `modal-3D/modal_3d/gateway.py` 已提供 `modal-3d.capabilities.v1`，并由同一 contract 驱动 Worker 路由与 options 校验；`submit()` 仍只返回 `call_id + model`，尚缺 request envelope、idempotency 与 input descriptor。
3. `modal-3D/modal_3d/common.py::generation_result()` 已统一外层 result shape，但 artifact 只有 `path/bytes/mime`，没有 hash、opaque ID、producer/revision/validation manifest。
4. `agent/jobs.py` 已完成 Durable Job correctness slice：DB `user_version=1` migration，状态含 `running/connection_required/cancel_requested/succeeded/failed/cancelled/expired`，并暴露 stable `error_code/retryable/updated_at`；临时认证/网络/Modal service 错误不再 terminal failed。仍未完成的是 stage/event log、artifact relation 和 submit idempotency。
5. `agent/artifacts.py` 已有安全 relative path、内容寻址上传与 SHA-256，但 read API 仍以 Volume path 为身份，没有 job-scoped opaque artifact ID。
6. Project Workspace 已持久化 source/SAM/canonical/model/job/GLB lineage，但 `POST /v1/projects` 仍是 `await file.read()` 后直接创建；MIME magic、像素、alpha、EXIF、输入 hash/limits 尚未形成统一 input descriptor。
7. Native GLB export 已经是流式远端读取 -> 本地临时文件 -> glTF v2/hash/bytes 验证 -> Tauri 原生保存；但“成功结果的长期本地 verified cache”尚未建立。
8. React 主工作流仍集中在 `App.tsx` 多个 state/while polling；在 Job/Artifact 契约稳定前不应先做大型前端状态机重构。

## 2. 当前架构判断

第一组现在不缺“再接一个模型”，而缺**稳定产品契约**。

```text
已经证明可计算
    Cloud SAM / Local SAM / 4 x 3D Worker
             |
             v
现在必须证明可长期使用
    Job 可恢复
    Capability 不漂移
    Input 可验证
    Artifact 可寻址/可缓存
             |
             v
最后才能稳定暴露给 AgentScape
    Connector pairing
    opaque Job ID
    opaque Artifact ID
    capability / events / cancel
```

因此当前关键路径固定为：

```text
P0-A Job 正确性
  -> P0-B Capability 单一事实源
  -> P0-C Input / Result / Artifact 契约
  -> P0-D Verified Local Artifact Cache
  -> P1-A AgentScape Minimal Connector
  -> P1-B React 状态机与诊断体验收口
```

**禁止反序**：Connector 不能早于 Durable Job + Artifact identity/cache 对外承诺稳定；否则 AgentScape 会依赖当前内部 Volume path、过简 Job state 和硬编码模型事实。

## 3. `modal-3D` 工作包真实状态

状态定义：`DONE` = 已有可重复证据；`PARTIAL` = 主链已存在但未满足原计划 Gate；`TODO` = 尚未形成该契约。

| 工作包 | 状态 | 当前证据 | 下一缺口 |
|---|---|---|---|
| M3D-00 部署与事实审计 | PARTIAL | 4 Worker/SAM 文档、benchmark、archive trellis.cpp、统一 Gateway 名称 | 机器可读 deployment matrix + 自动 `from_name()` smoke + canary 输入清单 |
| M3D-01 单一能力清单 | DONE | `modal-3d.capabilities.v1` 已上线：model/profile/input/output/status/options/deployment revision/reference performance/SAM cloud capability 一处定义 | 后续仅随 contract version 演进，不再另建 registry |
| M3D-02 输入契约与内容寻址 | PARTIAL | Worker adapter 已接统一 relative artifact path；client upload 已内容寻址 | Gateway CPU 层 input descriptor、prefix allowlist、MIME/pixels/alpha/hash/limits |
| M3D-03 统一 option 校验 | DONE | Gateway 直接消费 capability option schema；未知字段/类型与 Worker 已有显式 range 在 CPU 层校验，真实负向 canary 已证明 pre-spawn fail | future option 只能先进入 capability schema，再进入 Worker |
| M3D-04 统一 Job Envelope | TODO | submit 仅 `call_id + model` | request_id/idempotency/input/effective options/revision/submitted_at |
| M3D-05 Result/Artifact Manifest | PARTIAL | `generation_result()` 已统一 model/artifact/timing/metrics | artifact hash + opaque ID + validation + lineage + sidecar manifest |
| M3D-06 Worker 收敛 | PARTIAL | 四 Worker 已统一顶层 `generate(input_path, options)`；真实矩阵通过 | 共享 input/result/error helper、结构校验、统一错误类别；不做大爆炸重构 |
| M3D-07 SAM 3.1 产品化接口 | PARTIAL | Cloud segment/refine/materialize 已生产；Local engine 语义对齐 | schema/version/hash、TTL/cleanup、materialize descriptor/idempotency |
| M3D-08 可观测性/成本 | PARTIAL | benchmarks、load/inference/VRAM 已有；GPU policy 仍 1 container | correlation/error/stage/artifact hash 统一日志字段 |
| M3D-09 测试体系 | PARTIAL | 真实 benchmark/canary 很强 | 缺纯 unit/contract/negative canary 自动化 |
| M3D-10 部署回滚 | PARTIAL | Worker 可独立部署；Gateway 路由集中 | capability status=disabled/degraded 与兼容回滚 Gate |

### 后端当前冻结项

除非回归失败或下游明确需求，暂时**不做**：

- 新 3D 模型；
- Worker GPU 并发扩容；
- Pixal3D/FastSAM/Hermite/Hunyuan 新一轮性能调参；
- 公网匿名 API；
- Gateway Web 化；
- 把 Local SAM 逻辑搬进 `modal-3D`。

这些都不能解决当前契约稳定性问题。

## 4. `modal-3D-client` 工作包真实状态

| 工作包 | 状态 | 当前证据 | 下一缺口 |
|---|---|---|---|
| C3D-00 基线/smoke | DONE | Windows CI、PyInstaller smoke、四模型真实矩阵、Project e2e | 只随契约变化扩 smoke，不重做基线 |
| C3D-01 云端 capability | DONE | Agent 远端读取 v1、严格 major compatibility、原子 last-known-good cache；`/v1/models` 只是 projection；运行时代码已无具体 model ID/name/profile/warm hardcode | 后续只处理 contract version migration |
| C3D-02 本地输入管线 | TODO | Project source 已持久化 | magic/bytes/pixels/EXIF/alpha/hash；limits 来自 capability，不另写一套数字 |
| C3D-03 SAM provider | PARTIAL | Cloud/Local/Auto、refine、Local install/update/uninstall 已完成 | direct RGBA `skip`、provider selection/recovery contract；Windows+NVIDIA 实机证据 |
| C3D-04 模型/profile/options | PARTIAL | model/status/profile/options 全由 capability 驱动；新增 model ID 无需 client registry 改动；disabled model 无法提交 | advanced schema/UI 以后再做，不阻塞当前主线 |
| C3D-05 Durable Job Store v1 | PARTIAL | SQLite + remote call ID + restart restore + DB v1 migration + `updated_at/error_code/retryable` 已完成 | stage/event log、artifact relation、后续 DB compatibility/backup Gate |
| C3D-06 恢复/取消/重试/幂等 | PARTIAL | transient/auth/network 可恢复；`cancel_requested` + cancel race 已验证；真实 reconnect 保留同一 remote call | submit idempotency、event/retry policy、更多故障注入 |
| C3D-07 Artifact Cache/Workspace | PARTIAL | Project SQLite、source、native export、Local SAM data 生命周期 | verified content-addressed result cache、opaque artifact ID、远端 expiry 后仍可打开 |
| C3D-08 React 状态机 | TODO | MVP workflow 可用 | 等 C3D-05/07 稳定后再拆；当前不要先重构 UI |
| C3D-09 Viewer/验证 | PARTIAL | GLTFLoader/OrbitControls/GLB preview、原生 save | viewer summary/budget/error 分类；只消费 verified local artifact |
| C3D-10 安全 | PARTIAL | random localhost session、Windows vault、CSP、loopback Agent | connector 独立 scoped token、artifact owner scope、日志诊断脱敏 |
| C3D-11 Local Modal Connector | TODO | 尚未对 AgentScape 开放 | 等 C3D-05/07 完成后实现 pairing/capabilities/jobs/artifacts/events |
| C3D-12 可观测性 | PARTIAL | Agent/runtime 日志与部分状态存在 | correlation chain、job events、诊断包 |
| C3D-13 打包升级 | PARTIAL | Windows CI/NSIS、Rust cache、Local SAM runtime transactional update | sidecar/app/DB compatibility manifest、干净 Windows E2E |
| C3D-14 测试 | PARTIAL | Windows smoke + 14 个 Job/Project 单测 + 多次真实 Modal canary/E2E | artifact/input contract 单测、schema fixtures、expiry/hash/owner-scope 负向测试 |

### Local SAM v1 决策

Local SAM 当前进入**功能冻结**：

- bootstrap、cu128 wheel、checkpoint 均有固定 hash；
- install/update/uninstall 为完整生命周期；
- exact engine 已在 L40S 做 GPU 运行验证；
- Auto/Cloud/Local 已接 Project provider lineage；
- 唯一待补证据是 Windows + NVIDIA 组合实机。

在真实 Windows GPU 未暴露 bug 前，不再增加 Local SAM feature、模型参数或新 installer abstraction。下一阶段所有工程时间回到 Durable Job / capability / artifact 主线。

## 5. 下一批具体任务

以下顺序是当前执行顺序，不是建议菜单。

### P0-A：Durable Job correctness slice ✅ DONE (2026-08-24)

对应：C3D-05/C3D-06、联合契约 E2E-04/05/06。

**完成结论**：已修复。`JobManager.poll()` 不再把临时 Modal 网络/认证/服务异常永久写成 `failed`；取消也不再在 SDK `cancel()` 返回后伪造终态。

涉及文件：

- `modal-3D-client/agent/jobs.py`
- `modal-3D-client/agent/modal_client.py`
- `modal-3D-client/agent/main.py`
- `modal-3D-client/src/agent.ts`
- `modal-3D-client/src/App.tsx`
- 新增小型 Python tests；不先引入大型 test framework abstraction

最小目标状态：

```text
running
  |-- remote not ready ----------------------> running
  |-- connection/service temporary ---------> connection_required / recoverable
  |-- auth/permission -----------------------> connection_required
  |-- remote output expired -----------------> expired
  |-- remote user/model exception ----------> failed
  |-- real result ---------------------------> succeeded

cancel:
running -> cancel_requested -> cancelled OR succeeded (cancel race)
```

必须完成：

1. 明确 retryable Modal SDK exception 集，不再 `except Exception => failed`；
2. `connection_required` 是非终态；重新连接后同一 local Job 继续 reconcile，不生成第二个 remote call；
3. cancel 不能在 SDK `cancel()` 返回后立即伪造 `cancelled`，必须处理 remote-complete race；
4. DB 增加最小 migration/version，不用一次把 C3D-05 所有未来字段全部塞进去；
5. UI 能区分“云端仍可能运行，只是当前连接断开”和真正失败；
6. stable error code/message 不直接显示远端 traceback。

验收证据：

- commit：`3ca937e fix: make generation jobs recoverable`；
- Windows CI：run `32695629317`，frontend / Python unit tests / packaged Agent smoke / Tauri / NSIS / installer upload 全绿；
- unit tests：14 个，覆盖 pending、connection、auth、remote timeout、remote exception、expired、cancel retry/race、legacy DB migration、future DB guard、Project active-delete Gate；
- Modal cancel canary：`cancel_requested -> cancelled`，以及完成优先 race -> `succeeded`；
- Modal reconnect canary：`connection_required -> running -> succeeded`，local Job ID 和 remote call ID 均保持不变；
- UI 已区分 `connection_required`、`cancel_requested` 与真实 terminal failure；远端异常不直接把 traceback 作为用户 message。

尚未包含在本 slice：submit idempotency、event log、SSE、artifact relation。这些继续留在 C3D-05/06 后续，不把 P0-A 扩成大重构。

**禁止顺手做**：完整 Job events UI、SSE、Connector、React reducer 大重构。

### P0-B：`modal-3d.capabilities.v1` 单一事实源 ✅ DONE (2026-08-24)

对应：M3D-01/M3D-03/C3D-01/C3D-04。

涉及文件：

- `modal-3D/modal_3d/common.py`
- `modal-3D/modal_3d/gateway.py`
- 必要时新增一个纯 Python `capabilities.py`，但禁止分散成多层 registry class
- `modal-3D-client/agent/models.py`
- `modal-3D-client/agent/generation.py`
- `modal-3D-client/src/agent.ts`

v1 至少包含：

- service/contract revision；
- 四个 model ID/display/status；
- output type；
- recommended profile 与 effective options；
- option schema/min/max/enum；
- input alpha/format requirements；
- SAM cloud capabilities；
- deployment/upstream revision；
- historical performance 只给 `reference`，不能成为 SLA。

迁移策略：

1. backend capability 先上线；
2. client 获取、校验并 cache；
3. `/v1/models` 改由 capability projection 生成；
4. 保留短兼容窗口；
5. 最后删除 `agent/models.py` 里的模型事实，不要求删除 projection helper 本身。

验收证据：

- backend commit：`5fb839b feat: add generation capability contract`；11 个纯 CPU capability tests 全绿；
- Gateway deploy 成功；真实 `Function.from_name("modal-3d-gateway", "capabilities")` 返回 `modal-3d.capabilities.v1`；
- 真实负向调用证明 unknown option 在 Worker spawn 前被拒绝；
- client commit：`d436eb7 feat: consume cloud generation capabilities`；Windows run `32697639923` 全绿；
- client 当前 23 个 tests，包括 remote->cache、offline cache、incompatible major no-fallback、disabled model、duplicate model/profile、新 model ID 无 client registry 改动等；
- 4 个 current model recommended profile 与此前真实四模型矩阵一致；
- React 已移除默认具体 model ID，历史 Project 指向已删除 model 时不会静默切到第一个模型。

关于原计划“schema snapshot/fixture 双仓库共享一组例子”：本 slice 不额外引入第三个 contract package。后端用 canonical contract tests，客户端保留 v1 fixture，并通过真实 remote canary 验证两端；若 P0-C 后需要长期共享 JSON examples，再把 example 放到联合 contract fixtures，而不是现在制造新发布单元。

### P0-C：Input + Result + Artifact identity

对应：M3D-02/M3D-05/C3D-02/C3D-07。

目标不是“多加字段”，而是去掉**path 就是身份**的隐患。

必须形成：

```text
InputDescriptor
  id/hash/mime/bytes/pixels/alpha/source role

GenerationResult
  model/revision/input/options/timing/metrics
  primary_artifact_id

ArtifactDescriptor
  opaque id/role/mime/bytes/sha256/producer/created/expires
  internal remote path 仅 Agent/Connector 内部知道
```

实施顺序：

1. backend 先产生 hash/descriptor/validation；
2. client Project create 按 capability limits 做本地 input validation；
3. Agent DB 保存 descriptor，不把用户绝对路径写远端 lineage；
4. `/v1/assets?path=` 进入兼容期，新增 job/artifact-scoped API；
5. React/Tauri 不再持久化 Volume path。

验收：

- 损坏图、伪扩展名、无 meaningful alpha、超 capability limit 在 GPU 前失败；
- 同内容得到稳定 source hash；
- result artifact bytes/hash 与实际 Volume bytes 一致；
- 用户无法凭猜测 path 读取另一个 Job artifact。

### P0-D：Verified Local Artifact Cache

对应：C3D-07/C3D-09、联合契约 E2E-07/08。

当前 native export 已证明“流式 + 校验 + 原生保存”路径可行；下一步把同样原则变成持久 cache，而不是再造下载层。

目标目录：

```text
app-data/cache/sha256/<prefix>/<digest>
```

必须完成：

- remote stream -> `.part` -> bytes/hash/GLB validation -> atomic rename；
- DB artifact record 只在 commit 后标 verified；
- Viewer 优先打开 verified local cache；
- 远端 Volume artifact 过期后，本地已 verified 的历史结果仍可打开/另存；
- cache cleanup 有 size budget，不删除 active/open/leased artifact；
- 不在 React memory 里复制 100 MiB GLB 多份。

验收：E2E-07、E2E-08 完整通过。

### P1-A：AgentScape Minimal Connector

**进入条件**：P0-A/B/C/D 全部过 Gate。

只实现第三组已经依赖的最小接口：

```text
pair/session
capabilities
submit job
get/list job
cancel
artifact bytes
replayable events（若 P0-A 已形成 event log）
```

硬边界：

- AgentScape 只知道 local Job ID；
- 不知道 Modal FunctionCall ID；
- artifact 只用 opaque ID；
- 不知道 Volume path；
- token 独立于 Tauri session，有 scope/expiry/revoke；
- 不暴露 arbitrary Modal app/function/path invocation。

第一条 Connector canary：AgentScape pairing -> generic `modal-3d` FastSAM recommended -> local Job -> verified GLB bytes -> revoke。

## 6. 现在明确不做

直到 P0-A/B/C/D 完成：

- 不做 Connector API；
- 不做 React 全量 reducer/state-machine 重写；
- 不做 Web Runtime；
- 不增加第五个 3D model；
- 不优化 Local SAM 性能/安装 UX；
- 不引入 Redis/Postgres/消息队列；SQLite + Modal FunctionCall 足够当前单用户桌面产品；
- 不把 public FastAPI Gateway 重新变成 Desktop 主协议；Desktop 继续使用用户显式 `modal.Client`；
- 不把 Volume path 公开给 AgentScape；
- 不为了“统一”把四个 GPU Worker 合并成一个大镜像/大类。

## 7. AgentScape-plan 同步协议

以后每次第一组代码推进，按以下顺序执行：

```text
1. git pull / fetch 三个事实仓库
2. 确认 working tree，避免覆盖其他开发者
3. 实现一个可独立验收的 slice
4. 本地只跑轻量 lint/unit/contract
5. 重型 Windows/Tauri -> GitHub Actions
6. GPU/Modal 行为 -> 真实 Modal canary
7. 只有证据通过后更新本文件状态
8. AgentScape-plan 单独 commit/push
```

每次更新本文件至少修改：

- repo HEAD；
- CI/canary evidence；
- 对应 M3D/C3D 状态；
- “下一批具体任务”顶部任务；
- 若发现设计判断错误，记录原因，而不是静默改掉历史结论。

### Done 的判定

以下都不能单独叫 Done：

- “代码写了”；
- “能 import”；
- “计划已经设计”；
- “UI 看起来有按钮”；
- “旧 benchmark 曾经成功”。

Done 必须至少有与任务匹配的证据：unit/contract、packaged Windows CI、Modal real canary、artifact/hash validation 中的一项或多项。

## 8. 下一次执行入口

下一次继续第一组，直接进入 **P0-C Input + Result + Artifact identity**。

第一条执行原则：先定义最小 `InputDescriptor / ArtifactDescriptor`，并让 backend result 产生 bytes + SHA256 + producer/revision；再迁移 Agent DB/API 到 opaque artifact ID。不要先做 verified local cache，也不要先改 Connector——P0-D 依赖 P0-C 的稳定 artifact identity。
