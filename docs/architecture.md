# Architecture (post-v0.5.0)

This document describes the current runtime shape of `notebooklm-py`
after the v0.5.0 refactor program closed. The historical proposal and
migration tables are preserved in [`docs/refactor-history.md`](./refactor-history.md);
this file is the canonical current map.

## Layered Overview

```text
+------------------------------------------------------------------+
| Public Surface                                                    |
|   notebooklm.__init__ re-exports the stable Python API.           |
|   Public modules: client.py, auth.py, types.py, exceptions.py,    |
|   config.py, paths.py, research.py, log.py, io.py, migration.py,  |
|   urls.py, utils.py, notebooklm_cli.py.                           |
|   The rpc/ and cli/ subpackages are public-looking but internal   |
|   implementation packages; see ADR-012 and docs/stability.md.     |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
| CLI Layer                                                         |
|   notebooklm_cli.py assembles the Click root command.             |
|   cli/*_cmd.py modules own command wiring and rendering.          |
|   cli/services/ modules own reusable command-domain logic.        |
|   CLI code calls the public client/features; it does not encode    |
|   raw NotebookLM RPC payloads directly.                           |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
| Client Composition Layer                                          |
|   NotebookLMClient owns the concrete composition root.             |
|   It constructs Session, SourceUploadPipeline, NoteService,        |
|   NoteBackedMindMapService, and the namespaced feature APIs:       |
|     .notebooks  .sources  .artifacts  .chat                       |
|     .notes      .research  .settings  .sharing                    |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
| Feature API + Domain-Service Layer                                |
|   Feature facades depend on narrow capability Protocols or         |
|   feature-local runtime Protocols, not on broad Session internals. |
|   Larger domains are split into services such as source add/list/  |
|   poll/content/upload and artifact list/generate/poll/download.   |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
| Session + Transport Layer                                         |
|   Session is the internal facade around focused collaborators:     |
|   RpcExecutor, AuthRefreshCoordinator, ClientLifecycle, Kernel,    |
|   MiddlewareChainBuilder, TransportDrainTracker, ClientMetrics,    |
|   ReqidCounter, CookiePersistence, and AuthedTransport.            |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
| RPC Protocol Layer                                                |
|   rpc/types.py       method IDs, enums, endpoint helpers           |
|   rpc/encoder.py     batchexecute request encoding                 |
|   rpc/decoder.py     response parsing and safe_index               |
|   rpc/overrides.py   environment/test overrides for method IDs     |
+------------------------------------------------------------------+
```

## Public/Internal Module Boundary

ADR-012 is the load-bearing rule:

- Non-underscore top-level modules are the stable public surface only when
  documented and exported through `__all__`.
- Single-underscore modules and the `_auth/` subpackage are internal
  implementation seams.
- `notebooklm.rpc` and `notebooklm.cli` are historical public-looking
  subpackages with internal contents. `notebooklm.rpc.RPCMethod` is the
  documented exception; the stable CLI surface is the `notebooklm` console
  script, not the command implementation modules.

This lets implementation seams move while `notebooklm.__init__`,
`notebooklm.auth`, `notebooklm.types`, `notebooklm.exceptions`, and the
console script preserve the user-facing contract.

## Client Composition Root

[`NotebookLMClient`](../src/notebooklm/client.py) is the only production
composition root that knows the concrete internal objects:

```text
NotebookLMClient
  |
  +-- Session(auth, retry/concurrency/lifecycle options)
  |
  +-- SourceUploadPipeline(Session, Session.kernel, Session.auth, ...)
  |     +-- SourcesAPI(Session as RpcCaller, uploader=SourceUploadPipeline)
  |
  +-- NotebooksAPI(Session as RpcCaller, sources_api=SourcesAPI)
  |
  +-- NoteService(Session as RpcCaller)
  |     +-- NoteBackedMindMapService(NoteService)
  |     +-- ArtifactsAPI(Session as ArtifactsRuntime,
  |                     notebooks=NotebooksAPI,
  |                     mind_maps=NoteBackedMindMapService,
  |                     note_service=NoteService)
  |
  +-- ChatAPI(Session as ChatRuntime, notebooks=NotebooksAPI)
  |
  +-- NotesAPI(notes=NoteService,
  |            mind_maps=NoteBackedMindMapService,
  |            save_chat_answer=ChatAPI.save_answer_as_note)
  |
  +-- ResearchAPI(Session as RpcCaller)
  +-- SettingsAPI(Session as RpcCaller)
  +-- SharingAPI(Session as RpcCaller)
```

Two ordering details are intentional:

- `ChatAPI` is constructed before `NotesAPI` so `NotesAPI` can receive the
  `save_chat_answer` callback from chat.
- `SourceUploadPipeline` receives the concrete `Kernel` and `AuthMetadata`
  surfaces directly from `Session`; `SourcesAPI` owns source operations but
  does not reach back into `Session` for upload internals.

## Capability Protocol Model

ADR-013 ("Composable Session Capabilities") is the design rationale.
Feature APIs depend on narrow Protocols rather than the concrete
[`Session`](../src/notebooklm/_session.py) class.

Shared Protocols live in
[`_session_contracts.py`](../src/notebooklm/_session_contracts.py):

| Protocol | Responsibility |
| --- | --- |
| `RpcCaller` | `rpc_call(method, params, ...)`; the chokepoint every pure-RPC feature API uses for batchexecute calls. |
| `LoopGuard` | `assert_bound_loop()`; cross-loop affinity check for features that may touch async runtime state. |
| `OperationScopeProvider` | `operation_scope(label)`; async context manager for drain-tracked feature-owned work. |
| `AsyncWorkRuntime` | Composition of `LoopGuard` and `OperationScopeProvider`. |
| `AuthMetadata` | Selected-account routing metadata: `authuser` and `account_email`. Used by the upload pipeline. |
| `Kernel` | Upload-facing pure transport Protocol: `post()`, `cookies`, and `aclose()`. The concrete `_kernel.Kernel` also powers the main authed transport path. |

Feature-local Protocols stay beside their consumers:

| Protocol | Module | Responsibility |
| --- | --- | --- |
| `ChatRuntime` | [`_chat.py`](../src/notebooklm/_chat.py) | Chat capability union: `RpcCaller`, `LoopGuard`, `transport_post()`, and `next_reqid()`. |
| `ArtifactsRuntime` | [`_artifacts.py`](../src/notebooklm/_artifacts.py) | Artifact runtime union: `RpcCaller`, `AsyncWorkRuntime`, and `DrainHookRegistration`. |
| `UploadRuntime` | [`_source_upload.py`](../src/notebooklm/_source_upload.py) | Upload pipeline runtime: `RpcCaller`, `OperationScopeProvider`, and `LoopGuard`. |
| `DrainHookRegistration` | [`_artifacts.py`](../src/notebooklm/_artifacts.py) | `register_drain_hook(name, hook)` for artifact polling cleanup on close. |

Production satisfies these Protocols structurally through `Session`.
Tests should use [`make_fake_core(...)`](../tests/_fixtures/fake_core.py)
for lightweight feature cores or target the owning collaborator directly.

## Session Collaborator Graph

```text
                     +---------------------+
                     |  NotebookLMClient   |
                     +----------+----------+
                                |
                                v
                       +--------+--------+
                       |     Session     |
                       +--------+--------+
                                |
   +-----+-----+-----+-----+----+----+-----+-----+-----+
   |     |     |     |     |         |     |     |     |
   v     v     v     v     v         v     v     v     v
 Rpc  Auth  Client Middle Trans   Metrics Reqid Cookie Kernel
 Exec Coord Life   Chain  Drain   Tracker Count Persist
   |           |      |      |       |
   |           |      |      |       +-- telemetry callback + counters
   |           |      |      +---------- graceful drain state
   |           |      +---------------- ADR-009 middleware chain
   |           +----------------------- open/close, keepalive, save-on-close
   +----------------------------------- encode -> chain -> decode
```

The logical RPC path is:

```text
Session.rpc_call
  -> RpcExecutor.execute_with_telemetry
  -> RpcExecutor.execute
  -> Session._perform_authed_post
  -> ADR-009 middleware chain
  -> Session._authed_post_chain_terminal
  -> AuthedTransport.perform_authed_post
  -> Kernel.post
  -> httpx.AsyncClient.stream("POST", ...)
```

| Collaborator | Module | Responsibility |
| --- | --- | --- |
| `RpcExecutor` | [`_rpc_executor.py`](../src/notebooklm/_rpc_executor.py) | Encodes batchexecute requests, resolves RPC method overrides, applies idempotency policy, calls the transport path, decodes responses, and owns decode-time auth refresh/retry. |
| `AuthRefreshCoordinator` | [`_session_auth.py`](../src/notebooklm/_session_auth.py) | Owns auth snapshot serialization, single-flight refresh tasks, and token updates. |
| `ClientLifecycle` | [`_session_lifecycle.py`](../src/notebooklm/_session_lifecycle.py) | Opens/closes the HTTP client through `Kernel`, captures loop affinity, runs optional keepalive, runs drain hooks, saves cookies, and shields final close. |
| `Kernel` | [`_kernel.py`](../src/notebooklm/_kernel.py) | Owns the live `httpx.AsyncClient`, cookie jar, size-capped streaming POST, and `aclose()` target. |
| `MiddlewareChainBuilder` | [`_middleware_chain.py`](../src/notebooklm/_middleware_chain.py) | Constructs the canonical ADR-009 middleware chain. |
| `TransportDrainTracker` | [`_transport_drain.py`](../src/notebooklm/_transport_drain.py) | Tracks in-flight transport operations and gates graceful shutdown. |
| `ClientMetrics` | [`_client_metrics.py`](../src/notebooklm/_client_metrics.py) | Per-client counters, queue-wait accounting, and `on_rpc_event` callback emission. |
| `ReqidCounter` | [`_reqid_counter.py`](../src/notebooklm/_reqid_counter.py) | Lock-protected monotonic request ID allocation for chat transport. |
| `CookiePersistence` | [`_cookie_persistence.py`](../src/notebooklm/_cookie_persistence.py) | Open-time cookie baseline, in-process save serialization, and snapshot advancement after disk writes. |
| `AuthedTransport` | [`_authed_transport.py`](../src/notebooklm/_authed_transport.py) | Single-attempt authenticated POST leaf. It captures an auth snapshot, builds the URL/body/headers for that attempt, calls `Kernel.post`, and translates HTTP 429/5xx/network failures into middleware-consumable transport exceptions. Retry decisions live in middleware. |
| `IdempotencyRegistry` | [`_idempotency.py`](../src/notebooklm/_idempotency.py) | Registry keyed by `(RPCMethod, operation_variant)` that resolves retry policy and injects client tokens for token-deduped operations. |

`Session.poll_registry` is still present as a legacy test-observed
attribute. Production artifact polling state is owned by
`ArtifactsAPI._polling.poll_registry`; dropping the session-level
attribute requires migrating the remaining tests that observe it.

## Middleware Chain

The current chain order is pinned by
[`tests/unit/test_chain_wiring.py`](../tests/unit/test_chain_wiring.py)
and
[`tests/unit/test_middleware_chain_builder.py`](../tests/unit/test_middleware_chain_builder.py).
The order is load-bearing.

The list returned by
[`MiddlewareChainBuilder.build()`](../src/notebooklm/_middleware_chain.py)
is outermost first:

```text
DrainMiddleware              admit and track shutdown-drain operations
   |
MetricsMiddleware            time logical transport attempts
   |
SemaphoreMiddleware          hold one max_concurrent_rpcs slot per logical RPC
   |
RetryMiddleware              retry HTTP 429, HTTP 5xx, and network errors
   |
AuthRefreshMiddleware        refresh and retry auth failures
   |
ErrorInjectionMiddleware     synthetic-error harness for tests/cassette work
   |
TracingMiddleware            structured logging boundary
   |
AuthedTransport leaf         build auth-specific request, call Kernel.post
```

Current request-shape detail: `Session._perform_authed_post()` still
passes an `RpcRequest` whose `url`, `headers`, and `body` fields are
empty and whose `context` carries the actual closure-based contract:
`build_request`, `log_label`, `disable_internal_retries`, and
`rpc_method`. The terminal adapter reads those context keys and calls
`AuthedTransport.perform_authed_post()`. This is the current runtime
shape, not planned documentation: middleware behavior depends on those
context keys today.

## Feature And Domain Services

Feature facades are intentionally thin where a domain has grown beyond a
single file. Current service seams:

| Service / Module | Module | Responsibility |
| --- | --- | --- |
| `SourceAddService` | [`_source_add.py`](../src/notebooklm/_source_add.py) | URL, text, Drive, and YouTube source-add RPC construction and idempotent create probing. |
| `SourceLister` | [`_source_listing.py`](../src/notebooklm/_source_listing.py) | Source list/get row parsing. |
| `SourcePoller` | [`_source_polling.py`](../src/notebooklm/_source_polling.py) | Source readiness and registration polling. |
| `SourceContentRenderer` | [`_source_content.py`](../src/notebooklm/_source_content.py) | Source full-text/content rendering. |
| `SourceUploadPipeline` | [`_source_upload.py`](../src/notebooklm/_source_upload.py) | File upload registration, resumable upload handshake/stream/finalize, queue-wait metrics, and rename/wait orchestration. |
| `NotebookMetadataService` | [`_notebook_metadata.py`](../src/notebooklm/_notebook_metadata.py) | Notebook source ID and metadata helper Protocols used by chat/artifacts. |
| `NoteService` | [`_note_service.py`](../src/notebooklm/_note_service.py) | Note CRUD, note-backed generation persistence, and note row parsing. |
| `NoteBackedMindMapService` | [`_mind_map.py`](../src/notebooklm/_mind_map.py) | Mind-map adapter backed by standard notebook notes. |
| `ArtifactListingService` | [`_artifact_listing.py`](../src/notebooklm/_artifact_listing.py) | Artifact listing, filtering, and selection support. |
| `ArtifactGenerationService` | [`_artifact_generation.py`](../src/notebooklm/_artifact_generation.py) | Artifact generation request construction and generation-result parsing. |
| `ArtifactPollingService` | [`_artifact_polling.py`](../src/notebooklm/_artifact_polling.py) | Leader/follower polling, shared polling futures, close-time drain. |
| `ArtifactDownloadService` | [`_artifact_downloads.py`](../src/notebooklm/_artifact_downloads.py) | Download selection, URL download, interactive-content extraction, and mind-map downloads. |
| `_artifact_formatters` | [`_artifact_formatters.py`](../src/notebooklm/_artifact_formatters.py) | Markdown, HTML, CSV/table, quiz, flashcard, and interactive artifact formatting helpers. |
| `ShareManager` | [`_sharing_manager.py`](../src/notebooklm/_sharing_manager.py) | Legacy `SHARE_ARTIFACT` helper used by `NotebooksAPI.share()` for notebook/artifact share-link composition. |

Artifact helper services are extracted but still collaborate with the
`ArtifactsAPI` facade through a private `_ArtifactsServiceMethods`
Protocol for selected method seams. Direct collaborators such as
`notebooks`, `note_service`, and `mind_maps` are constructor-injected
instead of reached through the facade.

## Auth, Profiles, And Cookie Storage

[`auth.py`](../src/notebooklm/auth.py) is the public auth facade. The
implementation lives under the internal
[`_auth/`](../src/notebooklm/_auth/) subpackage:

| Module | Responsibility |
| --- | --- |
| `_auth/cookie_policy.py` | Required/optional cookie-domain policy and cookie validation. |
| `_auth/cookies.py` | Cookie-shape normalization and storage-state conversion. |
| `_auth/extraction.py` | CSRF/session token extraction from NotebookLM pages. |
| `_auth/refresh.py` | Refresh command execution and refresh-session helpers. |
| `_auth/keepalive.py` | Identity-surface cookie rotation for long-lived clients. |
| `_auth/storage.py` | Atomic storage-state saves, cross-process lock, snapshot/delta merge. |
| `_auth/account.py` | Account metadata read/write and account enumeration. |
| `_auth/paths.py` | Auth path helpers that build on `paths.py`. |

[`paths.py`](../src/notebooklm/paths.py) owns the profile-aware filesystem
contract:

```text
~/.notebooklm/
  config.json
  profiles/
    default/
      storage_state.json
      context.json
      browser_profile/
```

Legacy top-level `storage_state.json`, `context.json`, and
`browser_profile/` paths remain supported as fallbacks for the default
profile. The CLI sets the active profile at startup; library callers
should pass explicit paths/profile arguments rather than relying on
process-global profile state.

## RPC And Wire-Shape Handling

The RPC layer is intentionally small:

- [`rpc/types.py`](../src/notebooklm/rpc/types.py) is the source of truth
  for `RPCMethod`, enum codes, and endpoint URL helpers.
- [`rpc/overrides.py`](../src/notebooklm/rpc/overrides.py) resolves
  method-ID overrides used by tests and protocol-drift work.
- [`rpc/encoder.py`](../src/notebooklm/rpc/encoder.py) builds
  batchexecute request payloads.
- [`rpc/decoder.py`](../src/notebooklm/rpc/decoder.py) strips XSSI
  prefixes, parses chunked responses, extracts RPC results, and exposes
  `safe_index`.

NotebookLM responses are positional arrays. New parsing code should use
`safe_index` and follow ADR-011's strict-decode policy: default to fail-fast
on schema drift, with the documented environment opt-out for legacy soft
mode.

## CLI Architecture

[`notebooklm_cli.py`](../src/notebooklm/notebooklm_cli.py) owns root
Click option handling, profile setup, Windows runtime setup, logging
verbosity, and command registration.

Command modules under [`cli/`](../src/notebooklm/cli/) are command shells.
Reusable command logic belongs under
[`cli/services/`](../src/notebooklm/cli/services/) per ADR-008. The
current service package includes source operations, download/generate
flows, polling, research import, confirming mutations, and a
multi-module `cli/services/login/` package for browser-cookie login
support.

[`cli/grouped.py`](../src/notebooklm/cli/grouped.py) defines the
sectioned help layout. Tests pin that every top-level command is assigned
to a help section or explicitly marked as miscellaneous.

## Current Architectural Watchpoints

These are current facts about the architecture, not historical migration
notes:

- The middleware request seam is still context-dict based. `RpcRequest`
  has typed `url`/`headers`/`body` fields, but the live path still carries
  `build_request` and related values in `context`.
- Artifact services are physically extracted, but some behavior still
  routes back through `_ArtifactsServiceMethods` so old facade monkeypatch
  seams keep working.
- Raw positional wire parsing is safer than before because `safe_index`
  and strict-decode policy exist, but row-shape knowledge still lives in
  multiple feature/service modules rather than one adapter layer.
- `Session` is no longer a compatibility property-bridge object, but it
  still exposes a test-observed `poll_registry` attribute that production
  artifact polling does not use.

## ADR Cross-References

- [ADR-001](./adr/0001-layered-core-seams-and-property-bridge-policy.md) —
  Layered seams and property-bridge policy. Superseded; property shims are
  retired.
- [ADR-003](./adr/0003-auth-facade-write-through.md) — Auth facade
  write-through. Superseded by the current `_auth/` facade/export shape.
- [ADR-005](./adr/0005-idempotency-taxonomy.md) — RPC idempotency
  taxonomy and retry policy.
- [ADR-008](./adr/0008-cli-services-extraction-pattern.md) —
  `cli/services/` extraction pattern.
- [ADR-009](./adr/0009-middleware-chain.md) — Middleware chain ordering
  and per-request behavior.
- [ADR-010](./adr/0010-session-kernel-split.md) — Session/Kernel split.
  Superseded by ADR-013, but useful historical context.
- [ADR-011](./adr/0011-schema-validation-policy.md) — Strict decode and
  schema-drift policy.
- [ADR-012](./adr/0012-implementation-surface-convention.md) —
  public/internal module boundary.
- [ADR-013](./adr/0013-composable-session-capabilities.md) —
  composable capability Protocol model.

## See Also

- [`CLAUDE.md`](../CLAUDE.md) — high-level navigation map for agents.
- [`docs/development.md`](./development.md) — how to add a feature API.
- [`docs/refactor-history.md`](./refactor-history.md) — historical
  refactor narrative and migration tables.
- [`docs/python-api.md`](./python-api.md) — public Python API surface.
- [`docs/stability.md`](./stability.md) — API stability policy.
