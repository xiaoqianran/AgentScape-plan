# Runtime Backend Plane

本文是 AgentScape Runtime backend ownership 的架构权威视图，对应 ADR-0010。

## 1. Current Runtime Map

```text
                         WorldRuntime
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
      PhysicsSystem      NavigationSystem    InteractionSystem
          │                   │                   │
    PhysicsBackend      NavigationBackend         │
   ┌──────┼──────┐            │                   │
   ▼      ▼      ▼            ▼                   │
Rapier   Jolt  Transform   Recast/Detour          │
          │                   │                   │
          └──────────┬────────┘                   │
                     ▼                            │
               LocomotionSystem ◄────────────────┘
```

箭头表达 capability/state consumption，不表达传统技术分层。

## 2. Physics Ownership

### PhysicsSystem owns

- Object/Part ↔ opaque body/collider binding；
- provenance；
- semantic articulation state；
- held/pending pose semantics；
- navigation obstacle projection；
- counterfactual orchestration；
- effective capability/execution profile。

### PhysicsBackend owns

- native world/body/collider/joint/controller/query；
- native memory/ref-count lifecycle；
- native collision/filter behavior；
- native shape/subshape mapping；
- solver-specific evidence collection。

### Current Backends

| Backend | Role |
|---|---|
| Rapier | default full solver |
| Jolt | second full solver / portability proof |
| Transform | render-only / no solver truth |

Rapier/Jolt full capabilities：

```text
rigid-body
articulated-body
character-controller
collision
joints
scene-query
```

Runtime composite capabilities：

```text
transform-state
articulation-pose
counterfactual-query
```

## 3. Navigation Ownership

### NavigationSystem owns

- static/dynamic World scope；
- invalidation/build semantics；
- route normalization；
- blocker provenance/grouping；
- action-aware diagnostics；
- counterfactual obstacle suppression policy；
- World-facing status/profile。

### NavigationBackend owns

```text
build
syncObstacles
queryRoute
debugGeometry
native lifecycle
```

Current Recast backend capabilities：

```text
static-routing
route-query
dynamic-obstacles
obstacle-suppression
debug-geometry
```

Runtime composite：

```text
action-aware-diagnostics
counterfactual-routing
```

## 4. Locomotion Ownership

`LocomotionSystem` owns only movement task lifecycle：

```text
route
waypoint
elapsed/timeout
noProgress
verticalVelocity
completion
```

It consumes：

```text
Navigation.findPath()
Physics.moveCharacter()
```

No Rapier/Jolt/Recast native object.

## 5. Interaction Ownership

Current main owns in one `InteractionSystem`：

```text
humanHeldId
agentHeld
recoveryHeld
articulationTasks/results
settleTasks
pickup/carry/place/recovery orchestration
```

Independent lifecycle extraction is a pressure-based next slice, not Current Truth until merged.

## 6. Cross-system Flow

```text
Navigation
  “where can I theoretically go?”
        │ route
        ▼
Locomotion
  “what is current movement task progress?”
        │ desired movement
        ▼
Physics
  “can this exact step happen in current collision world?”
```

Therefore:

```text
route reachable ≠ execution guaranteed
```

## 7. Rendering Boundary

```text
Native Physics
   ↓
PhysicsSystem semantic pose
   ↓
Object3D
   ↓
Renderer
```

Renderer never reads native solver handles.

## 8. Evidence Boundary

```text
Backend evidence
   ↓ semantic + quality projection
System evidence
   ↓
Interaction / Validation / Agent Observation
```

Evidence quality is part of the contract.

## 9. Testing Gates

Physics：

- capability conformance；
- public-surface conformance；
- Rapier/Jolt shared parity；
- Jolt articulation/character/locomotion E2E。

Navigation：

- capability conformance；
- Recast tests；
- direct/fake backend portability；
- WorldRuntime factory injection。

## 10. Forbidden Drift

```text
WorldRuntime → RAPIER/Jolt/Recast native operation
LocomotionSystem → native character controller
InteractionSystem → native raycast/joint API
Renderer → native body handle
Backend contract → copied upstream API schema
```
