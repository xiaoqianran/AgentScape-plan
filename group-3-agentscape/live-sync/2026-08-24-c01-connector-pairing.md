# AgentScape Live Sync — C-01 Connector Pairing

> Snapshot date: 2026-08-24. This is an append-only execution record created in an isolated plan worktree while other AgentScape-plan files are concurrently edited elsewhere.

## Status

C-01 `Connector Pairing Contract` is now **MERGED**.

```text
branch: feat/c01-connector-pairing
final main commit: fefa495 feat: add scoped connector pairing sessions
base after rebase: 671e1ac feat: bridge EmbodiedGen evidence into compiler
```

The independent EmbodiedGen evidence-bridge track completed first (`671e1ac`), after which C-01 was rebased, focused-tested, fast-forwarded into `main`, and deployed.

## Delivered contract

New modules:

```text
src/connector/ConnectorSession.js
src/connector/ConnectorClient.js
```

Core semantics:

- Connector endpoint must be a bare loopback origin (`127.0.0.1`, `localhost`, `::1`);
- no URL credentials, LAN host, public host, arbitrary path, query or hash;
- client identity is `agentscape`;
- contract version is negotiated explicitly;
- scopes are restricted to:
  - `capabilities.read`;
  - `jobs.submit`;
  - `jobs.read`;
  - `jobs.cancel`;
  - `artifacts.read`;
- `approval_required` is not treated as paired;
- pairing response validates connector id/instance/version;
- short-lived token is held only in a JS private field;
- token is absent from public session snapshots;
- no localStorage/sessionStorage persistence exists;
- session snapshot exposes `paired`, `expired`, `revoked` state;
- network failure maps to recoverable `CONNECTION_REQUIRED`;
- scope escalation, origin mismatch, contract mismatch and expired session fail closed;
- Connector-managed `Authorization` cannot be overridden by a caller;
- authenticated requests must declare an explicit scope;
- authenticated request paths are constrained to `/connector/v1/*` and reject traversal/encoded traversal/query/hash escape;
- pairing session records both `capabilityRevision` and `capabilityHash`;
- revoke clears local authorization state after remote success.

## Verification evidence

Focused:

```text
3 test files / 24 tests PASS
```

Full JS regression:

```text
114 test files / 408 tests PASS
```

Other gates:

```text
Asset validation: PASS
Production Vite build: PASS (exit_code=0, built in 16.85s)
Asset Compiler Python smoke: 4/4 PASS
Tracked secret scan: no nvapi key
```

CodeGraph direct dependency facts:

```text
ConnectorClient production callers: none yet
normalizeConnectorSessionResponse callers: ConnectorClient.pair only
ConnectorSession impact: client + connector tests only
```

The module is intentionally not wired into main UI or Runtime yet. This keeps C-01 a narrow security/session boundary.

## Merge Gate — completed

Completed sequence:

1. current EmbodiedGen dirty WIP on main must be committed or removed;
2. rebase C-01 onto the new main;
3. rerun connector/provider focused tests;
4. full CI/build must pass on the combined main;
5. then update Live Map from `COMMITTED_NOT_MERGED` to `MERGED`.

## Newly unlocked work

C-02 `Capability Discovery Adapter` can be designed and implemented on top of the C-01 feature branch, but must not be merged to main before C-01's merge gate is satisfied.

## Final main evidence

GitHub Actions run `32698144927` for `fefa495` completed successfully:

```text
Test and build          success
Deploy to GitHub Pages  success
workflow conclusion     success
```
