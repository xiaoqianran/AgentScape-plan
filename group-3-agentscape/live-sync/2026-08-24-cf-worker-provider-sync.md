# AgentScape Live Sync — CF Worker / Runtime Truth / Provider Registry

> 本记录是并发实施期间的只增量状态快照。`04-live-execution-map.md` 当前正在被其它执行轨修改，因此本文件只记录已经有 Git/CI/网络证据的新事实；待主 Live Map 收口后再折叠回总账本。

## 1. AgentScape 当前主线

当前已验证：

```text
main == origin/main
HEAD = 0684af0 feat: add provider capability registry

0684af0 Provider Registry merged
  |
25b2d53 precise approach distance for placement
  |
6668173 switch LLM upstream to CF Worker
  |
f58039d deterministic preset placement workflow
```

工作树在本快照时为 clean。

## 2. CF Worker upstream 已切换

服务端 OpenAI-compatible upstream 已从旧地址切换到新的 CF Worker `/v1` endpoint。

安全规则：

- 真正 API key 只写入 AgentScape ignored `.env.local`；
- `.env.local` 权限为 `0600`；
- tracked `.env.local.example` 只保留 placeholder；
- Git secret scan 未发现 `nvapi-` credential；
- model 保持 `nvidia/nemotron-3.5-lightning-30b-a3b`，没有更换。

### 网络/模型实测

已从 AgentScape 主机直接验证：

- CF Worker CORS preflight 对 `https://xiaoqianran.github.io` 返回 `204`；
- `Access-Control-Allow-Origin: https://xiaoqianran.github.io`；
- 允许 `Authorization, Content-Type`；
- authenticated `/models` 返回 `200`；
- 当前返回 112 个模型，目标 Nemotron 在列表中；
- 最小 `/chat/completions` 返回 `200`，目标模型回复 `OK`；
- AgentScape 本地 `/agent` facade -> CF Worker -> Nemotron 返回 `200`；
- tool calling smoke 返回恰好一个 `ping({value:"hello"})` tool call。

因此：**上游兼容性、CORS、普通 completion、AgentScape tool calling 均已证实可用。**

## 3. GitHub Pages 状态

提交 `6668173` 已推到 `origin/main`。

GitHub Actions run `32695466555`：

```text
build                 success
Test and build        success
deploy                success
Deploy to Pages       success
```

之后 Runtime/Provider main 继续部署成功。

当前 Pages URL 实测：

```text
https://xiaoqianran.github.io/AgentScape/
HTTP 200
```

### 重要边界：CORS 可用 != 应把 API key 放进 Pages

当前前端 `HttpLLMGateway` 使用 AgentScape `/agent` contract：

```text
Browser
   |
   | AgentScape request { messages, tools, context }
   v
/agent facade
   |
   | server-side Authorization + OpenAI translation
   v
CF Worker /v1
```

CF Worker 虽然允许 Pages Origin，但**不能因此把真实 provider key 编译进 GitHub Pages bundle 或默认 localStorage**。否则 public static site 的访问者可以读取 credential。

另外 `/v1/chat/completions` 与 AgentScape `/agent` response contract 不同；仅替换 browser endpoint 也不能直接兼容。

因此安全的下一阶段仍应是 Connector/session 或 server-side `/agent` facade，而不是 client-side secret。

## 4. Runtime truth 修复已收口一批

并发实施过程中，`fix precise approach distance for placement` 一度误提交在 Provider branch 上。已完成无损分离：

```text
Runtime branch:
fix/precise-approach-distance-placement
  -> 25b2d53

Provider branch:
feat/as01-provider-registry
  -> independent Provider commit only
```

Runtime commit `25b2d53`：

- placement approach 在首次 locomotion 后检查实际 actor-position；
- 若剩余误差大于 interaction correction tolerance，则执行精确 arrival correction；
- 新增 place E2E 对 correction 与最终距离的断言。

本地 focused regression：

```text
5 test files
19 tests
PASS
```

GitHub Actions run `32695769192`：

```text
build   success
deploy  success
```

因此此 Runtime fix 已进入 `main`。

## 5. P-01 Provider Registry 已完成并进入 main

最终 Provider commit：

```text
0684af0 feat: add provider capability registry
```

状态从：

```text
COMMITTED_NOT_MERGED
```

升级为：

```text
MERGED
```

当前 capability foundation 覆盖：

- provider identity/version；
- health/status；
- stable provider-scoped operation ID；
- input/output capability descriptors；
- execution binding；
- raw provider result consumer；
- disabled/unavailable provider semantics；
- `local-catalog`；
- legacy HTTP generator；
- `modal-2d` placeholder capability；
- `modal-3d` placeholder capability；
- `embodiedgen` capability；
- custom provider contract test。

`AssetLibrary` 不再要求每接一个新 Provider 就增加 provider-specific `if/else`。

### 验证证据

在最新 Runtime 基线上：

```text
Provider/World focused:
7 test files / 37 tests PASS

Asset validation:
PASS
```

在合并前的完整本地 suite：

```text
113 test files / 397 tests PASS
```

最终 main `0684af0` GitHub Actions run `32695868985`：

```text
Test and build          success
Deploy to GitHub Pages  success
workflow conclusion     success
```

因此 P-01 的 production build Gate 已由干净 CI 明确通过。

## 6. 当前 Gate 变化

### 已通过

```text
R-01 current placement truth slice   -> MERGED
P-01 Provider Registry               -> MERGED
CF Worker upstream transport         -> VERIFIED
Pages deployment                     -> VERIFIED
```

### 新解锁

```text
C-01 Connector Pairing Contract
        |
        v
C-02 Capability Discovery Adapter
        |
        v
J-01 Async Generation Job State Machine
```

C-01 现在是真正可执行的下一 slice，不再被 P-01 阻塞。

## 7. C-01 下一任务 contract

目标不是“做一个 Connector 大系统”，而是先建立一个最窄的安全 session boundary：

```text
GitHub Pages / AgentScape Browser
              |
              | pair request
              v
        Local/Remote Connector
              |
              +-- owns provider credential
              +-- owns provider capability refresh
              +-- issues scoped session
              |
              v
        AgentScape session identity
              |
              v
      ProviderRegistry snapshot
```

C-01 首版必须定义：

1. pair request / explicit approval；
2. session ID 是 opaque identity；
3. session scope；
4. expiry；
5. revoke；
6. connector version；
7. capability snapshot revision/hash；
8. `connection_required`；
9. origin binding；
10. browser 永远拿不到 provider credential。

### C-01 明确不做

- 不做 Async Job DB；
- 不做 Artifact Store；
- 不接真实 Modal GPU；
- 不做 Job Center UI；
- 不把 API key 编译进 Pages；
- 不修改 Physics/Interaction；
- 不让 provider capability 自动升级成 Runtime action truth。

### C-01 hard tests

必须至少覆盖：

```text
pair success
approval required
wrong origin
expired session
revoked session
version mismatch
scope escalation denied
capability snapshot revision
credential absent from browser response
```

## 8. 仍未解决的独立 1.35 correctness 工作

以下任务仍保持独立，不因 P-01 完成而冒充已解决：

```text
W-01 WorldSpec constrained revision  -> STASH_PROTOTYPE
B-01 Backend production handoff      -> STASH_PROTOTYPE
R-02 WorldRuntime mutation atomicity -> PLANNED_CORRECTNESS_FIX
```

尤其 R-02 仍需要正式修复 `WorldRuntime.mutate()` exception atomicity；不要因为部分 caller 有显式 restore 就把通用 transaction contract 标为完成。

## 9. 下一同步触发条件

下一次 AgentScape 有以下任一变化时更新 Live Map：

- C-01 branch/commit 创建；
- Connector session contract 发生稳定 schema 变化；
- ProviderRegistry schema 被 C-02 扩展；
- WorldRevision stash 工程化；
- Backend stash 工程化；
- mutation atomicity 修复；
- GitHub Pages `/agent` public facade 有正式部署身份。
