# Integration Ledger

2026-08-29 起，Ledger 以收敛后的仓库边界为准。

| Old integration | Target integration | Decision |
|---|---|---|
| `AgentScape-agent → AgentScape` | Agent logic lives in `AgentScape/src/agent` | MERGED |
| `modal-inference-hub → providers` | Human workflow lives in `AgentScape`; uses same provider boundary | MERGED/RETIRED |
| `AgentScape → modal-gen-client repo` | `AgentScape → modal-provider` connector contract | MERGED |
| `modal-gen-client → modal-2D-client` | internal `modal-provider` package edge | MERGED |
| `modal-2D-client → modal-2D` | internal `modal-provider` package edge | MERGED |
| `modal-gen-client → modal-3D-client` | internal `modal-provider` package edge | MERGED |
| `modal-3D-client → modal-3D` | internal `modal-provider` package edge | MERGED |
| `AgentScape → kaggle-inference-hub` | none | REMOVE |
| `modal-build → EmbodiedGen` | `modal-provider/modal-EmbodiedGen → pinned upstream` | MERGED |
| `AgentScape workspace → EmbodiedGen submodule/repo` | provider-owned upstream pin/clone | REMOVE REPO BOUNDARY |
| `modal-lab → migration evidence` | experiments stay with owning repo/package | REMOVE |

## Current legal repository edges

```text
AgentScape
   │ provider-neutral contract
   ▼
modal-provider
   │ source/model/runtime upstreams
   ▼
external services / pinned upstream source

AgentScape-plan
   └─ documentation only; no runtime edge
```

## Forbidden reintroductions

- 不得让 AgentScape 依赖 `modal-provider` 内部文件路径。
- 不得把 `modal-2D*` / `modal-3D*` 再写成顶层仓库依赖。
- 不得恢复 Kaggle 作为平级生产 Provider，除非先新增 ADR。
- 不得恢复独立 Human Hub/Agent 仓库，除非出现新的独立 state/security/deployment owner 并通过 ADR。
