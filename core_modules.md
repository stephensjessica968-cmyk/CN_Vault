# CN Vault — Core Module Interfaces

This document defines the foundational modules for CN Vault and the contracts they expose to each other and to external clients. The intent is to make the vault composable, testable, and resilient while honoring the SEE•N Soul-Tech principles of clarity, stewardship, and traceability.

## Module map

| Module | Purpose | Primary Consumers |
| --- | --- | --- |
| Identity & Consent | Resolves user identities, consent state, and access scopes. | UX surfaces, Vault Ledger, API Gateway |
| Vault Ledger | Writes and reads immutable records of scrolls, rituals, and system events. | Identity & Consent, Ritual Engine, Observability |
| Scroll Registry | Curates structured scroll metadata, versions, and publication flows. | UX surfaces, Vault Ledger, API Gateway |
| Ritual Engine | Orchestrates guided flows (dhikr protocols, healing sessions, validations). | UX surfaces, Scroll Registry |
| UX Surfaces | Web/mobile/UI layer that renders scrolls and rituals to participants. | End users, API Gateway |
| API Gateway | Presents public/private APIs, enforces policy, and brokers inter-module calls. | External services, UX Surfaces |
| Observability | Emits telemetry and compliance trails. | All modules |

## Identity & Consent

**Responsibilities**
- Manage identities (person, practitioner, device) with verifiable attributes.
- Track consent grants, expirations, and revocations per scope.
- Issue signed tokens for internal calls.

**Interfaces**
- `POST /identity` — create/update identity profile with claims payload and provenance.
- `GET /identity/{id}` — resolve identity and current consent scopes.
- `POST /consent/grant` — grant scoped consent; returns consent id and expiry.
- `POST /consent/revoke` — revoke existing consent id; propagates to dependent modules via event bus.
- **Event** `consent.changed` — `{ identity_id, scopes[], status, effective_at }` broadcast to Vault Ledger and API Gateway.

## Vault Ledger

**Responsibilities**
- Maintain an append-only log for scroll access, ritual runs, consent events, and configuration changes.
- Provide read models for audit, integrity proofs, and rollups.

**Interfaces**
- `POST /ledger/event` — ingest signed event envelope `{source, actor, action, payload, signature}`.
- `GET /ledger/events?filter=...` — query events by identity, scope, or ritual id.
- `GET /ledger/proof/{event_id}` — return integrity proof (hash chain or Merkle proof) for an event.
- **Event** `ledger.snapshot.ready` — emitted after scheduled compaction, consumed by Observability.

## Scroll Registry

**Responsibilities**
- Store scroll metadata (title, lineage, tags, sensory assets) and version history.
- Manage publication status (draft, reviewed, sacred-locked) with approver signatures.

**Interfaces**
- `POST /scroll` — create draft scroll with metadata and steward signature.
- `PUT /scroll/{id}` — update metadata or bump version; writes to Vault Ledger.
- `POST /scroll/{id}/publish` — change status to `reviewed` or `sacred-locked`; triggers notification to Ritual Engine.
- `GET /scroll/{id}` — resolve scroll with latest approved assets.
- **Event** `scroll.published` — `{ scroll_id, version, steward_id, checksum }` consumed by UX Surfaces and Ritual Engine.

## Ritual Engine

**Responsibilities**
- Execute guided rituals (dhikr sets, sensory cues, affirmations) sourced from Scroll Registry.
- Validate prerequisites: consent scope, steward signature, and environment safety checks.
- Record completions and reflections to Vault Ledger.

**Interfaces**
- `POST /ritual/session` — start session with `{identity_id, scroll_id, mode}`; returns session token and timeline.
- `POST /ritual/session/{id}/event` — stream participant events (step completed, reflection note, biometrics hash).
- `POST /ritual/session/{id}/complete` — finalize session, compute outcomes, and persist to Ledger.
- **Event** `ritual.session.completed` — `{ session_id, identity_id, scroll_id, outcome }` consumed by Observability and UX Surfaces.

## UX Surfaces

**Responsibilities**
- Render scrolls, rituals, and consent prompts to participants.
- Cache non-sensitive assets; rely on API Gateway for authenticated data.

**Interfaces**
- Consumes `scroll.published`, `ritual.session.completed` events for live updates.
- Calls API Gateway endpoints for identity, consent, scroll, and ritual operations.

## API Gateway

**Responsibilities**
- Fronts all external calls; performs rate limiting, authZ via Identity & Consent, and schema validation.
- Routes to internal services over mTLS; signs inter-module requests.

**Interfaces**
- `POST /api/v1/{resource}` — normalized CRUD entrypoints that forward to respective module services.
- `GET /.well-known/manifest` — advertises available resources, schema versions, and health states.
- **Event** `api.policy.violated` — emitted on blocked requests, logged to Vault Ledger.

## Observability

**Responsibilities**
- Collect metrics, logs, and traces for all modules.
- Enforce retention and redaction policies; surface compliance dashboards.

**Interfaces**
- `POST /telemetry` — ingest structured telemetry with module tag and request correlation id.
- `GET /telemetry/{trace_id}` — retrieve trace with consent-aware redaction applied.
- Subscribes to `ledger.snapshot.ready`, `ritual.session.completed`, and `api.policy.violated` events.

## Data contracts

All inter-module calls share a common envelope to guarantee integrity and traceability:

```json
{
  "id": "evt_123",
  "timestamp": "2024-12-04T19:00:00Z",
  "actor": {"id": "ident_789", "role": "practitioner"},
  "action": "scroll.publish",
  "payload": {"scroll_id": "scr_456", "version": "1.0.0"},
  "provenance": {"module": "Scroll Registry", "signature": "<sig>"}
}
```

- **Transport:** REST over HTTPS for external calls; event bus (NATS/Kafka) for asynchronous fan-out.
- **Security:** mTLS between modules, signed payloads, and consent-scope verification per call.
- **Localization:** All user-facing text carries language codes; rituals specify sensory assets with media hashes.

## Non-functional requirements

- **Reliability:** Each module exposes `/health` and `/readiness` probes; Ledger supports replay for recovery.
- **Privacy:** No PII in events; store references to encrypted blobs managed by Identity & Consent keys.
- **Observability:** Every request carries correlation ids; failures auto-emit `api.policy.violated` or `ledger.snapshot.ready` recovery markers.
- **Extensibility:** New modules must declare their events and required scopes in the manifest served by API Gateway.
