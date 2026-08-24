# AS-09A — Generation Provider / Job Skill Facade

## 1. Task Identity

```text
Task ID:        AS-09A
Parent Plan:    AS-09 Agent Skills 与权限语义
Status:         READY
Base Commit:    5202f6f
Feature Branch: feat/as09a-generation-job-skills
```

## 2. Goal

Expose existing ProviderRegistry + Connector capability/job contracts to Agent skills without duplicating state or collapsing long-running Job truth into synchronous tool success.

First slice:

```text
listGenerationProviders
listGenerationCapabilities
submitGenerationJob
getGenerationJob
cancelGenerationJob
```

Artifact import/compile orchestration is AS-09B.

## 3. Permission Contract

Introduce policy scopes:

```text
generation.read
generation.write
```

Defaults:

```text
viewer  -> generation.read
builder -> generation.read
admin   -> *
```

`generation.write` is deliberately not granted to default builder because submit may create billable/external side effects. Deployments may define an explicit custom profile that grants it.

## 4. World Mutation Boundary

Submit/cancel are external Connector side effects, not scene/world mutations:

```text
mutates = false
batchable = false
```

They must not create CommandHistory snapshots or pretend to be rollbackable world edits.

## 5. Long Job Outcome Semantics

Tool call returns immediately with canonical Job identity/projection.

```text
job-accepted / queued / running -> requested/pending
job-connection-required        -> unverified/recoverable
job-cancel-requested           -> requested
job-succeeded                  -> unverified, ARTIFACT_NOT_IMPORTED
job-failed / expired           -> failed
job-cancelled                  -> unverified/cancelled
```

`job-succeeded` is never `asset-ready`.

## 6. Service Composition

Add a narrow `GenerationServices` object that owns no duplicate Provider state. It reuses `AssetLibrary.providerRegistry` and optionally composes:

```text
ConnectorClient
ConnectorCapabilityAdapter
ConnectorJobClient
```

No Connector configured => cached/local provider listing still works; remote refresh/job operations return `CONNECTION_REQUIRED`.

## 7. Merge Gate

- permission tests prove builder cannot submit by default;
- admin/custom generation.write can submit;
- submit/cancel do not call runtime.mutate/history;
- list/get read paths work with generation.read;
- job-succeeded outcome stays unverified;
- full regression/build/Python/CI green.
