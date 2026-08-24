# J-02 — Job Restart/Reconcile + Event Recovery

## 1. Task Identity

```text
Task ID:        J-02
Parent Plan:    AS-03 / Gate L3
Owner Track:    Connector / Job
Status:         MERGED
Base Commit:    6a08f31
Final Commit:   140acd9
Feature Branch: feat/j02-job-reconcile-events
```

## 2. Goal

Allow AgentScape to recover asynchronous Job truth after browser/process restart, Connector reconnect, duplicate/out-of-order events, or SSE disconnect **without inventing remote state and without depending on signed artifact URLs**.

Connector remains durable Job source of truth. AgentScape reconstructs a safe projection.

## 3. Non-goals

- no Artifact bytes import/cache;
- no Compiler/Admission transition;
- no SQLite/browser persistence implementation;
- no Job Center UI;
- no WorldRuntime mutation;
- no provider-specific poller;
- no fake local completion;
- no arbitrary remote provider IDs as AgentScape identity.

## 4. Critical distinction: remote Job truth vs local transport overlay

There are two different meanings of “connection required” and J-02 must not conflate them.

### A. Connector Job fact

The Connector can authoritatively return:

```text
status = connection_required
sequence = N
```

That is a canonical Job fact and may enter `GenerationJobStore`.

### B. AgentScape cannot reach Connector

If `ConnectorClient.request()` itself fails with `CONNECTION_REQUIRED`, AgentScape has **no new remote Job fact** and therefore must not fabricate:

```text
status = connection_required
sequence = last + 1   // FORBIDDEN
```

Instead maintain a local transport overlay:

```text
canonical Job status = last remote fact
transport state      = connection_required
view phase           = recoverable
```

On successful reconcile the overlay clears. Remote event sequence remains untouched.

## 5. Restart/bootstrap source of truth

On a fresh AgentScape process, recover Jobs from Connector:

```text
GET /connector/v1/jobs
scope = jobs.read
```

The list response is canonical Connector projection data, not local cached provider call IDs.

Bootstrap must:

1. validate every Job with J-01 projection contract;
2. reject duplicate Job IDs / idempotency conflicts;
3. avoid partial store replacement on malformed list responses;
4. preserve terminal remote facts;
5. return current event cursor if Connector supplies one;
6. never require signed artifact URLs to recover identity.

## 6. Reconcile active Jobs

After bootstrap or reconnect:

```text
accepted / queued / running
connection_required
cancel_requested
```

may be refreshed with:

```text
GET /connector/v1/jobs/{id}
scope = jobs.read
```

Remote terminal statuses do not regress.

A failed Connector request marks only the local transport overlay. It does not modify `lastEventSequence`.

## 7. Event stream contract

Endpoint:

```text
GET /connector/v1/events
scope = jobs.read
Accept: text/event-stream
Last-Event-ID: <global Connector sequence>
```

The Connector event cursor is **global stream sequence**, distinct from each Job’s `lastEventSequence`.

Normalized event envelope:

```text
sequence
timestamp
jobId
attempt
type
oldStatus
newStatus
stage
progress
safe message/details
correlationId (safe opaque correlation only)
```

No Secret, prompt text, image bytes, signed URLs, provider traceback, or Authorization data.

## 8. Event handling model

Treat SSE as a notification/evidence channel, not as a second Job truth implementation.

Preferred flow:

```text
SSE event(sequence=N, jobId=J)
          |
          v
validate monotonic global cursor
          |
          v
GET /connector/v1/jobs/J
          |
          v
J-01 GenerationJobStore.apply(canonical snapshot)
          |
          v
advance stream cursor only after successful reconciliation
```

This avoids reconstructing full Job state from partial event details.

## 9. Global event cursor semantics

```text
sequence > cursor    -> process/reconcile then advance
sequence == cursor   -> duplicate/idempotent if same envelope
sequence < cursor    -> stale ignore
same sequence + different envelope -> protocol conflict
```

Cursor must not advance when Job reconciliation fails. This ensures reconnect can replay the notification.

## 10. Poll fallback

SSE disconnect is not Job failure.

J-02 should expose a finite `reconcile()`/poll operation that UI/later scheduler can call. Do not create uncontrolled internal infinite poll loops.

Backoff/jitter scheduling belongs to the caller or later orchestration layer unless a small deterministic helper is needed.

## 11. Suggested modules

```text
src/jobs/GenerationJobReconciler.js
src/jobs/GenerationJobTransportOverlay.js
src/connector/ConnectorJobEventClient.js
```

Possible J-01 extensions:

```text
ConnectorJobClient.list()
GenerationJobStore.replaceFromSnapshotAtomically()
```

## 12. Atomic bootstrap requirement

```text
GET Job list
   |
normalize ALL
   |
validate IDs / idempotency / transitions
   |
all valid ?
 |        |
no       yes
 |         |
reject     atomic store replace/merge
existing store unchanged on failure
```

## 13. Required tests

At minimum:

```text
restart bootstrap list -> store reconstructed
malformed second Job -> no partial bootstrap
list duplicate Job ID rejected
idempotency conflict rejected
active Job reconcile advances canonical sequence
terminal -> running regression rejected
Connector transport failure creates local overlay only
transport failure does not change Job event sequence/status
successful reconnect clears overlay
SSE Last-Event-ID uses global cursor
stale global event ignored
duplicate global event idempotent
same sequence conflicting event rejected
cursor advances only after successful Job GET/apply
SSE disconnect leaves Job truth intact
secret/signed-url/traceback event fields rejected/stripped
poll fallback uses jobs.read only
```

## 14. Merge Gate

- J-01 remains green;
- no Physics/Interaction/WorldRuntime changes;
- full JS regression PASS;
- production build PASS;
- CI/deploy PASS;
- no Artifact-ready semantics introduced;
- no background polling loop created implicitly.

## 15. Completion Evidence

```text
Final Status:       MERGED
Final Commit:       140acd9 feat: add job reconcile and event recovery
Merged into main:   yes
Focused tests:      6 files / 42 tests PASS
Full JS regression: 124 files / 478 tests PASS
Asset validation:   PASS
Production build:   PASS (exit_code=0, 14.87s)
Python smoke:       4/4 PASS
CodeGraph:          Reconciler 7 nodes/11 edges; EventClient 5/7
GitHub Actions run: 32701237783
Test and build:     success
Pages deploy:       success
```

Key correctness boundary preserved: Connector transport failure is represented as a local overlay and does not fabricate a new remote Job status/event sequence. SSE events are notifications; canonical Job truth is refreshed through `/jobs/{id}` before the global event cursor advances.

### Unlocked Gate

A-01 Artifact Descriptor Contract is now ready.
