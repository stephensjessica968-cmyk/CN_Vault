# CN Vault — Core Module Interfaces

This document defines the foundational modules for CN Vault and the contracts they expose to each other and to external clients. The intent is to make the vault composable, testable, and resilient while honoring the SEE•N Soul-Tech principles of clarity, stewardship, and traceability.

## Module map

| Module | Purpose | Primary Consumers |
| --- | --- | --- |
| Identity & Consent | Resolves user identities, consent state, and access scopes. | UX surfaces, Vault Ledger, API Gateway |
| Vault Ledger | Writes and reads immutable records of scrolls, rituals, and system events. | Identity & Consent, Ritual Engine, Observability |
| Scroll Registry | Curates structured scroll metadata, versions, and publication flows. | UX surfaces, Vault Ledger, API Gateway |
| Ritual Engine | Orchestrates guided flows (dhikr protocols, healing sessions, validations). | UX surfaces, Scroll Registry |
| State Regulation Engine (SEE•N) | Classifies autonomic state, predicts transition risk, and emits adaptive UI cues. | Ritual Engine, UX Surfaces, Observability |
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

## State Regulation Engine (SEE•N)

**Responsibilities**
- Classify each session window into one of four validated autonomic states: `ventral_vagal`, `sympathetic`, `high_alert_sympathetic`, `dorsal_vagal`.
- Apply temporal smoothing across physiological and cognitive channels before state assignment.
- Compute transition probabilities with a predictive delta layer and publish anticipatory UI adaptation signals.
- Enforce non-diagnostic safety cutoffs and emit hard-stop alerts when telemetry exceeds safe operating bounds.

**Interfaces**
- `POST /state/classify` — classify a telemetry window and return state, confidence, and color encoding.
- `POST /state/transition/probability` — compute weighted transition probabilities from current state + metric deltas.
- `GET /state/config` — retrieve active thresholds, smoothing windows, and transition rules.
- `POST /state/config/calibration` — store individualized baseline calibration profile bound to `identity_id`.
- **Event** `state.changed` — `{ session_id, previous_state, current_state, confidence, effective_at }` consumed by Ritual Engine and UX Surfaces.
- **Event** `state.transition.predicted` — `{ session_id, from_state, to_state, probability, horizon_sec }` consumed by UX Surfaces.
- **Event** `state.safety.cutoff` — `{ session_id, metric, measured_value, threshold, severity }` consumed by Ritual Engine and Observability.

### SEE•N state schema

```json
{
  "states": [
    {
      "name": "ventral_vagal",
      "nervous_system_branch": "parasympathetic_social_engagement",
      "physiological_thresholds": {
        "HRV_RMSSD_ms": {"min": 50, "max": 120},
        "HeartRate_bpm": {"min": 55, "max": 75},
        "BreathRate_bpm": {"min": 10, "max": 16},
        "MuscleTone_EMG": {"min": 0, "max": 15},
        "EyeStability_deg": {"max_saccade_velocity": 20},
        "CognitiveBandwidth_index": {"min": 70, "max": 100},
        "ThreatBias_score": {"max": 30}
      },
      "color_encoding": {
        "primary_hex": "#4CAF50",
        "secondary_hex": "#81C784",
        "saturation_percent": {"min": 30, "max": 50},
        "brightness_percent": {"min": 60, "max": 80},
        "contrast_ratio": {"min": 2.5, "max": 3.5}
      },
      "functional_significance": "regulated engagement, social readiness, learning"
    },
    {
      "name": "sympathetic",
      "nervous_system_branch": "sympathetic_mobilized_engagement",
      "physiological_thresholds": {
        "HRV_RMSSD_ms": {"min": 20, "max": 50},
        "HeartRate_bpm": {"min": 76, "max": 100},
        "BreathRate_bpm": {"min": 16, "max": 22},
        "MuscleTone_EMG": {"min": 15, "max": 40},
        "EyeStability_deg": {"max_saccade_velocity": 30},
        "CognitiveBandwidth_index": {"min": 50, "max": 70},
        "ThreatBias_score": {"max": 50}
      },
      "color_encoding": {
        "primary_hex": "#FFC107",
        "secondary_hex": "#FFEB3B",
        "saturation_percent": {"min": 60, "max": 80},
        "brightness_percent": {"min": 65, "max": 85},
        "contrast_ratio": {"min": 4.0, "max": 5.0}
      },
      "functional_significance": "action-oriented, alert, performance"
    },
    {
      "name": "high_alert_sympathetic",
      "nervous_system_branch": "sympathetic_threat_dominant",
      "physiological_thresholds": {
        "HRV_RMSSD_ms": {"min": 5, "max": 20},
        "HeartRate_bpm": {"min": 101, "max": 140},
        "BreathRate_bpm": {"min": 22, "max": 30},
        "MuscleTone_EMG": {"min": 40, "max": 80},
        "EyeStability_deg": {"max_saccade_velocity": 50},
        "CognitiveBandwidth_index": {"min": 30, "max": 50},
        "ThreatBias_score": {"max": 80}
      },
      "color_encoding": {
        "primary_hex": "#F44336",
        "secondary_hex": "#E53935",
        "saturation_percent": {"min": 80, "max": 100},
        "brightness_percent": {"min": 80, "max": 90},
        "contrast_ratio": {"min": 6.0, "max": 8.0}
      },
      "functional_significance": "threat salience, urgency detection, dominance signaling"
    },
    {
      "name": "dorsal_vagal",
      "nervous_system_branch": "parasympathetic_energy_conservation",
      "physiological_thresholds": {
        "HRV_RMSSD_ms": {"min": 0, "max": 15},
        "HeartRate_bpm": {"min": 40, "max": 55},
        "BreathRate_bpm": {"min": 6, "max": 10},
        "MuscleTone_EMG": {"min": 0, "max": 10},
        "EyeStability_deg": {"max_saccade_velocity": 10},
        "CognitiveBandwidth_index": {"min": 0, "max": 50},
        "ThreatBias_score": {"max": 20}
      },
      "color_encoding": {
        "primary_hex": "#1976D2",
        "secondary_hex": "#455A64",
        "saturation_percent": {"min": 10, "max": 30},
        "brightness_percent": {"min": 30, "max": 50},
        "contrast_ratio": {"min": 2.0, "max": 3.0}
      },
      "functional_significance": "shutdown, conservation, withdrawal"
    }
  ]
}
```

### Transition and smoothing contracts

- `ventral_vagal -> sympathetic`: `HRV_RMSSD_ms < 50` OR `HeartRate_bpm > 75` sustained for `>10s` (weight `0.7`).
- `sympathetic -> high_alert_sympathetic`: `HeartRate_bpm > 100` AND `ThreatBias_score > 50` (weight `0.8`).
- `high_alert_sympathetic -> dorsal_vagal`: `HRV_RMSSD_ms < 5` OR `CognitiveBandwidth_index < 30` (weight `0.9`).
- `dorsal_vagal -> ventral_vagal`: `HRV_RMSSD_ms > 50` AND `HeartRate_bpm < 75` sustained for `>20s` (weight `0.6`).

Temporal smoothing windows (seconds):
- `HRV_RMSSD_ms: 8`
- `HeartRate_bpm: 8`
- `BreathRate_bpm: 5`
- `MuscleTone_EMG: 5`
- `CognitiveBandwidth_index: 10`

Safety cutoffs:
- `HeartRate_bpm` hard limits: `35..150`
- `MuscleTone_EMG` hard limits: `0..90`
- `ThreatBias_score` hard limit: `<=100`

Predictive delta layer:
- Enabled with an 8-second sliding window over `HRV_RMSSD_ms`, `HeartRate_bpm`, `BreathRate_bpm`, `MuscleTone_EMG`, and `CognitiveBandwidth_index`.
- Delta alert thresholds: `HRV 5`, `HR 5`, `Breath 2`, `EMG 5`, `CognitiveBandwidth 5`.
- Transition probability families are logistic functions over weighted deltas.
- UI adaptation contract exposes color-ramp anticipation, trend-vector icon support, and a 5-second render smoothing window.
- All outputs must include the explicit note: "Color is a low-bandwidth proxy for arousal. This system does not diagnose or treat."

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
- **State semantics:** State labels and color channels are regulatory proxies and must remain decoupled from medical diagnosis language.

## Non-functional requirements

- **Reliability:** Each module exposes `/health` and `/readiness` probes; Ledger supports replay for recovery.
- **Privacy:** No PII in events; store references to encrypted blobs managed by Identity & Consent keys.
- **Observability:** Every request carries correlation ids; failures auto-emit `api.policy.violated` or `ledger.snapshot.ready` recovery markers.
- **Extensibility:** New modules must declare their events and required scopes in the manifest served by API Gateway.
