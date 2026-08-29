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

## Remaining migration work

1. 删除 active docs 中把旧组件称为“独立仓库”的表述。
2. 将 CI/脚本从跨仓 checkout 假设改为 `modal-provider` monorepo path filter/matrix。
3. 保留 package-level 独立 test/deploy，但不复制 repository-level architecture authority。
4. 逐步清理旧 repo URL、submodule、workspace inventory 与历史同步脚本。
5. 历史 benchmark/ADR 可保留旧名称，但必须明确标记为 historical。
