# ADR-0010 — Replaceable Runtime Backend Ownership

**Status:** Accepted
**Date:** 2026-08-29

## Context

AgentScape 早期 Runtime 直接依赖 Rapier 与 Recast/Detour。随着 Physics second-solver 与 Navigation portability work 落地，继续把 native engine object model 放在 System 层会导致：

- World/Interaction/Locomotion 被具体 solver schema 污染；
- 第二 backend 被迫模拟第一个 backend API；
- renderer/debug/validator 直接依赖 native handle；
- capability/execution mode 随品牌而不是 consumer semantics 漂移。

## Decision

采用：

```text
Semantic System
      │
      ▼
Deep Backend Contract
      │
      ▼
Concrete Native Backend
```

### Physics

```text
PhysicsSystem
  → PhysicsBackend
     ├─ RapierPhysicsBackend
     ├─ JoltPhysicsBackend
     └─ TransformPhysicsBackend
```

`PhysicsSystem` owns World physics semantics/provenance/composite runtime capability；concrete backend owns native solver lifecycle。

### Navigation

```text
NavigationSystem
  → NavigationBackend
     └─ RecastNavigationBackend
```

`NavigationSystem` owns World navigation semantics/action-aware diagnostics；backend owns NavMesh/TileCache/query/native obstacle lifecycle。

### Locomotion

`LocomotionSystem` remains a single movement-task state owner. It consumes Navigation route semantics and Physics movement semantics. No backend layer is introduced until real extraction pressure exists.

### Interaction

Current main still keeps Interaction as one orchestration/state owner. Independent articulation/settle lifecycle extraction is allowed only after code lands and preserves high-level public semantics.

## Capability Rule

Backend capability names express consumer semantics, not upstream types.

Allowed:

```text
rigid-body
collision
scene-query
joints
character-controller
static-routing
route-query
dynamic-obstacles
```

Forbidden direction:

```text
rapier-cuboid
jolt-body-id
recast-poly-ref
```

## Execution Mode Rule

Execution mode ownership follows execution owner.

Physics native solver modes:

```text
realtime
validation-only
```

Physics runtime-composite mode:

```text
render-only
```

Visual/render success must not be promoted to solver verification.

## Native Leakage Rule

System/World callers must not depend on:

```text
RAPIER.RigidBody / Collider / Joint
Jolt.BodyID / SubShapeID / HingeConstraint / CharacterVirtual
Recast NavMesh / TileCache / poly refs
```

Native types stay behind concrete backend.

## Evidence Rule

A shared semantic result may expose different evidence strength. Missing evidence must remain missing.

Example:

```text
Rapier contact
  solver-contact
  impulseAvailable=true

Jolt contact
  geometric-contact
  impulseAvailable=false
  totalImpulse=null
```

Do not fabricate `0` for unavailable measurement.

## Conformance Rule

Every backend slice must have executable conformance. Capability declaration without implementation fails closed. Concrete backend must not publish non-contract native convenience methods.

## Extraction Rule

This ADR does **not** mandate one backend/interface per System. Extract only when there is:

- native implementation ownership;
- independent state/failure lifecycle;
- deployment/security/scaling boundary;
- divergent test matrix;
- persistent merge conflict;
- measured performance pressure.

Line count alone is not pressure.

## Consequences

Positive:

- Rapier/Jolt can satisfy the same World semantics;
- Recast can be replaced without rewriting navigation policy;
- renderer/agent/validator remain native-engine-neutral;
- backend evidence quality is explicit;
- future experimental backend can validate contract without forking World Domain.

Cost:

- Backend contract itself becomes architecture-significant and must stay small/deep;
- parity/conformance tests become mandatory;
- native optimizations may require opaque backend-private structures rather than public shortcuts.
