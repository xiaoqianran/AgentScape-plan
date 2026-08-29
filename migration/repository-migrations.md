# Repository Migrations

## Completed consolidation map

| Former repository | Current home | Result |
|---|---|---|
| `AgentScape-agent` | `AgentScape` | Agent/Skill/LLM-VLM orchestration merged into core monorepo |
| `modal-inference-hub` | `AgentScape` | Human workflow/UI responsibility absorbed; standalone boundary retired |
| `modal-gen-client` | `modal-provider/modal-gen-client` | merged |
| `modal-2D-client` | `modal-provider/modal-2D-client` | merged |
| `modal-2D` | `modal-provider/modal-2D` | merged |
| `modal-3D-client` | `modal-provider/modal-3D-client` | merged |
| `modal-3D` | `modal-provider/modal-3D` | merged |
| `kaggle-inference-hub` | none | removed from target architecture |
| `modal-build` | `modal-provider/modal-EmbodiedGen` + provider build tooling | merged/retired as standalone repo |
| `EmbodiedGen` | external upstream pinned by Provider integration | no standalone AgentScape repo |
| `modal-lab` | owning repo/package tests/experiments | standalone repo retired |

## Current cleanup state

AgentScape repository/runtime convergence 已完成：

1. active docs 已不把旧 Provider package 当独立 repository；
2. AgentScape 已删除 recursive submodule/repo sync tooling；
3. LegacyAuthoring/direct Provider path 已删除；
4. Python SDK 已收敛为 Connector consumer；
5. tests/CI 已按当前 boundary 重组。

剩余工作主要是 `modal-provider` 自己的 package-level 持续维护与历史文档整理，不需要恢复任何 standalone repository boundary。
