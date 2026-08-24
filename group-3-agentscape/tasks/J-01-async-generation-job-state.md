# J-01 — Async Generation Job Projection / State Machine

## 1. Task Identity

```text
Task ID:        J-01
Parent Plan:    AS-03 / Gate L3
Owner Track:    Connector / Job
Status:         READY
Base Commit:    47470c0
Feature Branch: feat/j01-async-job-state
```

## 2. Goal

Create the first AgentScape-side asynchronous generation Job projection and Connector Job client so AgentScape can submit/query/cancel a versioned provider operation **without equating Connector/provider success with asset readiness or task completion**.

J-01 is a local projection/state-machine slice. Connector remains durable source of truth.

## 3. Non-goals

- no SQLite/browser durable Job DB;
- no SSE reconnect implementation;
- no restart reconcile (J-02);
- no Artifact bytes download/import (A-02);
- no Compiler/admission transition;
- no Job Center UI;
- no World mutation integration;
- no direct Modal FunctionCall identity;
- no provider-specific branching in Job state machine.

## 4. Core truth model

Do not use one `status` to represent remote execution + artifact readiness + AgentScape task success.

```text
Connector status
      |
      v
AgentScape Job Projection
      |
      +--> pending
      +--> recoverable(connection_required)
      +--> cancelling(cancel_requested)
      +--> result_available(provider succeeded)
      +--> terminal_non_success

result_available
      !=
artifact imported
      !=
asset compiled
      !=
asset ready
      !=
world ready
      !=
task verified
```

Connector `succeeded` therefore maps to `result_available`, **not** local asset success.

## 5. Required Job identity

Projection should preserve only stable/canonical fields needed by AgentScape:

```text
id                    Connector local Job ID (opaque)
provider
operation
kind
requestHash
idempotencyKey
contractVersion
capabilityHash
capabilityRevision
effectiveOptions/model/workflow revision (safe projection)
status                 raw Connector status
phase                  AgentScape semantic phase
stage                   server-declared only
progress                semantic object, never invented percentage
attempt
parent/relations
created/submitted/started/updated/completed timestamps
error                   safe structured projection
result                  safe artifact/result summary only
lastEventSequence
```

Do not persist provider remote FunctionCall/call IDs or signed artifact URLs as Job identity.

## 6. Connector statuses v1

Accepted statuses:

```text
accepted
queued
running
connection_required
cancel_requested
cancelled
failed
expired
succeeded
```

Projection mapping:

| Connector | AgentScape phase | Terminal locally? |
|---|---|---|
| accepted/queued/running | `pending` | no |
| connection_required | `recoverable` | no |
| cancel_requested | `cancelling` | no |
| succeeded | `result_available` | no — Artifact work remains |
| failed/cancelled/expired | `terminal_non_success` | yes |

J-01 must not invent `completed` for provider success.

## 7. Transition invariants

For a single Connector Job ID / attempt:

```text
accepted -> queued -> running -> succeeded|failed|expired
                         |
                         +-> cancel_requested -> cancelled|succeeded|failed
                         +-> connection_required -> recover to observed nonterminal/final state
```

Rules:

1. terminal Connector fact cannot regress to running/queued;
2. `connection_required` is recoverable and may return to the last/current remote state after reconnect;
3. `cancel_requested` is not cancelled;
4. success/cancel race accepts whichever final Connector fact is authoritative and records event sequence;
5. stale event sequence must not roll state backward;
6. same sequence with conflicting facts is a protocol conflict;
7. retry defaults to a new Job ID linked by `retry_of`, not attempt mutation in place;
8. Job stage comes from Connector; no client-invented percent progress.

## 8. Event sequence slice

J-01 stores monotonic sequence on the projection even though SSE reconnect is J-02.

Required behavior:

```text
sequence > last    -> apply
sequence == last   -> idempotent only if same fact
sequence < last    -> stale, ignore
same sequence + different status/stage -> conflict
```

No prompt text, Secret, bytes or remote traceback may enter event/projection logs.

## 9. Connector Job API slice

Use C-01 scoped request boundary:

```text
POST /connector/v1/jobs
  scope=jobs.submit

GET /connector/v1/jobs/{id}
  scope=jobs.read

POST /connector/v1/jobs/{id}/cancel
  scope=jobs.cancel
```

List/SSE may be added in J-02 unless needed for basic contract fixtures.

Job IDs must be path-safe encoded opaque IDs; caller cannot inject path traversal.

## 10. Submit contract

Request must contain canonical/explicit fields only:

```text
provider
operation
operationVersion
idempotencyKey
requestHash
inputs (typed refs/value contract)
profile/options
requested output roles
parent relation
retention/safe metadata
capabilityHash
capabilityRevision
```

Rules:

- capability must exist in current ProviderRegistry snapshot;
- discovery `available` does not by itself make a low-level Runtime mutation;
- J-01 may submit only through Connector Job API;
- same idempotency key + different request hash must be treated as conflict;
- do not generate fake remote IDs.

## 11. Suggested modules

```text
src/jobs/GenerationJobProjection.js
src/jobs/GenerationJobStore.js
src/connector/ConnectorJobClient.js
```

Keep transport and state transition logic separated.

## 12. Required tests

At minimum:

```text
normalize accepted/queued/running
succeeded -> result_available, not completed
connection_required is recoverable
cancel_requested is nonterminal
cancel_requested -> succeeded race allowed
terminal -> running regression rejected
stale event ignored
same sequence same fact idempotent
same sequence conflicting fact rejected
remote/signed URL/secret-like fields stripped or rejected
submit uses jobs.submit scope
get uses jobs.read scope
cancel uses jobs.cancel scope
job ID traversal encoded/rejected
idempotency key/request hash retained
unknown Connector status rejected
provider-specific fields do not leak into canonical projection
```

## 13. Merge Gate

Before merge:

- C-01/C-02 remain green in main;
- focused Connector/Provider/Job tests PASS;
- full JS regression PASS;
- production build PASS;
- no changes to Physics/Interaction/WorldRuntime required;
- no Artifact readiness semantics introduced prematurely.
