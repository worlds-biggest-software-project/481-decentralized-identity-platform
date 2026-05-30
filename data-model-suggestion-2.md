# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourcing architecture where every state change is captured as an immutable domain event. The write side appends events to an append-only event store. The read side materialises one or more projections optimised for specific query patterns (verification lookups, admin dashboards, compliance reporting). Commands are validated before producing events, and projections are rebuilt from the event stream on demand.

## Why This Suits a Decentralized Identity Platform

Decentralized identity is inherently an audit-trail domain. Every credential issuance, revocation, presentation, and verification is a discrete, meaningful event that regulators, issuers, and holders may need to inspect months or years later. Event sourcing makes this audit trail a first-class architectural element rather than a bolt-on logging table.

The W3C VC lifecycle is naturally event-driven: a credential is issued (event), presented (event), verified (event), revoked (event). Each event is cryptographically signed and timestamped, mirroring the append-only, tamper-evident properties of the event store itself. CQRS also allows the verification API (read-heavy, latency-sensitive) to scale independently from the issuance API (write-heavy, consistency-critical).

The main cost is operational complexity: developers must reason about eventual consistency between projections and the event store, handle projection rebuilds, and manage event schema evolution over time.

---

## Event Store Schema (PostgreSQL)

The event store is the single source of truth. All other tables are projections.

```sql
-- ============================================================
-- EVENT STORE (append-only, immutable)
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,   -- 'DID', 'Credential', 'Presentation', 'Wallet', 'Device'
    aggregate_id    UUID NOT NULL,           -- the identity of the domain aggregate
    sequence_num    BIGINT NOT NULL,          -- monotonically increasing per aggregate
    event_type      VARCHAR(200) NOT NULL,   -- e.g. 'CredentialIssued', 'DIDCreated'
    event_version   INT NOT NULL DEFAULT 1,  -- schema version for this event type
    payload         JSONB NOT NULL,          -- the full event data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, actor_did, tenant_id, ip
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_id, sequence_num)
);

-- Immutability enforced at application layer and via REVOKE UPDATE, DELETE on role
CREATE INDEX idx_events_aggregate ON event_store(aggregate_id, sequence_num);
CREATE INDEX idx_events_type ON event_store(event_type);
CREATE INDEX idx_events_created ON event_store(created_at);
CREATE INDEX idx_events_tenant ON event_store USING GIN ((metadata->'tenant_id'));

-- Projection checkpoint tracking
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(200) PRIMARY KEY,
    last_event_id   UUID REFERENCES event_store(event_id),
    last_sequence   BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Domain Events

Each event type has a well-defined JSON payload schema. Below are the core events.

### DID Aggregate Events

```jsonc
// DIDCreated
{
  "event_type": "DIDCreated",
  "aggregate_type": "DID",
  "payload": {
    "did": "did:web:example.com:users:alice",
    "method": "did:web",
    "subject_id": "uuid",
    "entity_type": "person",
    "tenant_id": "uuid",
    "controller_did": null,
    "verification_methods": [
      {
        "key_id": "did:web:example.com:users:alice#key-1",
        "key_type": "Ed25519VerificationKey2020",
        "public_key_jwk": { "kty": "OKP", "crv": "Ed25519", "x": "..." },
        "purposes": ["authentication", "assertionMethod"]
      }
    ],
    "service_endpoints": []
  }
}

// DIDKeyRotated
{
  "event_type": "DIDKeyRotated",
  "payload": {
    "did": "did:web:example.com:users:alice",
    "old_key_id": "...#key-1",
    "new_key_id": "...#key-2",
    "new_public_key_jwk": { ... },
    "reason": "scheduled_rotation"
  }
}

// DIDDeactivated
{
  "event_type": "DIDDeactivated",
  "payload": {
    "did": "did:web:example.com:users:alice",
    "reason": "user_request"
  }
}
```

### Credential Aggregate Events

```jsonc
// CredentialSchemaRegistered
{
  "event_type": "CredentialSchemaRegistered",
  "aggregate_type": "Credential",
  "payload": {
    "schema_id": "uuid",
    "tenant_id": "uuid",
    "name": "UniversityDegree",
    "version": "1.0",
    "json_schema": { ... },
    "context_urls": ["https://www.w3.org/ns/credentials/v2"],
    "credential_type": ["VerifiableCredential", "UniversityDegree"]
  }
}

// CredentialIssued
{
  "event_type": "CredentialIssued",
  "payload": {
    "credential_id": "uuid",
    "schema_id": "uuid",
    "issuer_did": "did:web:university.edu",
    "holder_did": "did:key:z6Mk...",
    "issuance_date": "2026-05-26T10:00:00Z",
    "expiration_date": "2031-05-26T10:00:00Z",
    "proof_format": "json-ld",
    "claims": {
      "degree": "Bachelor of Science",
      "field": "Computer Science",
      "gpa": "3.8"
    },
    "credential_hash": "sha256:abcdef...",
    "revocation_registry_id": "uuid",
    "status_index": 42
  }
}

// CredentialRevoked
{
  "event_type": "CredentialRevoked",
  "payload": {
    "credential_id": "uuid",
    "registry_id": "uuid",
    "reason": "holder_compromise",
    "revoked_by_did": "did:web:university.edu"
  }
}

// CredentialSuspended / CredentialReinstated
{
  "event_type": "CredentialSuspended",
  "payload": {
    "credential_id": "uuid",
    "reason": "investigation_pending"
  }
}
```

### Presentation Aggregate Events

```jsonc
// PresentationRequested
{
  "event_type": "PresentationRequested",
  "payload": {
    "request_id": "uuid",
    "verifier_did": "did:web:employer.com",
    "nonce": "random-nonce-value",
    "requested_schemas": ["uuid"],
    "requested_attributes": ["degree", "field"],
    "zk_predicate": { "attribute": "gpa", "predicate": ">=", "value": 3.0 },
    "expires_at": "2026-05-26T11:00:00Z"
  }
}

// PresentationSubmitted
{
  "event_type": "PresentationSubmitted",
  "payload": {
    "presentation_id": "uuid",
    "request_id": "uuid",
    "holder_did": "did:key:z6Mk...",
    "verifier_did": "did:web:employer.com",
    "proof_format": "sd-jwt",
    "credential_ids": ["uuid"],
    "disclosed_attributes": { "uuid": ["degree", "field"] }
  }
}

// PresentationVerified
{
  "event_type": "PresentationVerified",
  "payload": {
    "presentation_id": "uuid",
    "is_verified": true,
    "verification_checks": {
      "signature_valid": true,
      "not_revoked": true,
      "not_expired": true,
      "schema_match": true,
      "zk_predicate_satisfied": true
    }
  }
}
```

### Wallet and Device Events

```jsonc
// WalletCreated
{
  "event_type": "WalletCreated",
  "payload": {
    "wallet_id": "uuid",
    "subject_id": "uuid",
    "wallet_type": "mobile_ios",
    "biometric_bound": true
  }
}

// CredentialAddedToWallet
{
  "event_type": "CredentialAddedToWallet",
  "payload": {
    "wallet_id": "uuid",
    "credential_id": "uuid"
  }
}

// DeviceRegistered
{
  "event_type": "DeviceRegistered",
  "payload": {
    "device_id": "uuid",
    "subject_id": "uuid",
    "device_type": "ai_agent",
    "attestation_type": "mcp-i",
    "mcp_i_profile": { "agent_name": "billing-bot", "capabilities": ["invoice.read"] }
  }
}
```

---

## Command Handlers (Pseudocode)

```python
class IssueCredentialCommand:
    tenant_id: UUID
    schema_id: UUID
    issuer_did: str
    holder_did: str
    claims: dict
    proof_format: str
    expiration_date: Optional[datetime]

class IssueCredentialHandler:
    def handle(self, cmd: IssueCredentialCommand) -> list[Event]:
        # 1. Validate issuer DID is active and has assertionMethod key
        issuer = self.did_repository.load(cmd.issuer_did)
        assert issuer.is_active()
        assert issuer.has_purpose('assertionMethod')

        # 2. Validate schema exists and claims conform
        schema = self.schema_repository.load(cmd.schema_id)
        schema.validate_claims(cmd.claims)

        # 3. Allocate revocation index
        registry = self.registry_repository.find_or_create(cmd.issuer_did, cmd.tenant_id)
        status_index = registry.allocate_next_index()

        # 4. Sign credential
        signed_vc = self.crypto_service.sign_credential(issuer, cmd)
        credential_hash = sha256(signed_vc)

        # 5. Produce event
        return [CredentialIssued(
            credential_id=uuid4(),
            schema_id=cmd.schema_id,
            issuer_did=cmd.issuer_did,
            holder_did=cmd.holder_did,
            claims=cmd.claims,
            proof_format=cmd.proof_format,
            credential_hash=credential_hash,
            revocation_registry_id=registry.id,
            status_index=status_index,
            issuance_date=now(),
            expiration_date=cmd.expiration_date
        )]
```

---

## Read Projections (Materialised Views)

```sql
-- ============================================================
-- PROJECTION: Credential Lookup (optimised for verification API)
-- ============================================================

CREATE TABLE proj_credentials (
    credential_id       UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    schema_id           UUID NOT NULL,
    schema_name         VARCHAR(500),
    issuer_did          VARCHAR(2048) NOT NULL,
    holder_did          VARCHAR(2048) NOT NULL,
    issuance_date       TIMESTAMPTZ NOT NULL,
    expiration_date     TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    proof_format        VARCHAR(20) NOT NULL,
    credential_hash     BYTEA NOT NULL,
    revocation_registry_id UUID,
    status_index        INT,
    revoked_at          TIMESTAMPTZ,
    suspension_at       TIMESTAMPTZ
);

CREATE INDEX idx_proj_cred_issuer ON proj_credentials(issuer_did);
CREATE INDEX idx_proj_cred_holder ON proj_credentials(holder_did);
CREATE INDEX idx_proj_cred_status ON proj_credentials(status);

-- ============================================================
-- PROJECTION: DID Resolution Cache
-- ============================================================

CREATE TABLE proj_did_documents (
    did                 VARCHAR(2048) PRIMARY KEY,
    subject_id          UUID NOT NULL,
    method              VARCHAR(50) NOT NULL,
    entity_type         VARCHAR(50) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    document_json       JSONB NOT NULL,          -- full DID document ready to serve
    updated_at          TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- PROJECTION: Verification Audit Trail
-- ============================================================

CREATE TABLE proj_verification_log (
    presentation_id     UUID PRIMARY KEY,
    holder_did          VARCHAR(2048) NOT NULL,
    verifier_did        VARCHAR(2048) NOT NULL,
    credential_ids      UUID[] NOT NULL,
    is_verified         BOOLEAN,
    proof_format        VARCHAR(20),
    requested_at        TIMESTAMPTZ NOT NULL,
    verified_at         TIMESTAMPTZ,
    verification_checks JSONB
);

CREATE INDEX idx_proj_verify_verifier ON proj_verification_log(verifier_did);
CREATE INDEX idx_proj_verify_date ON proj_verification_log(requested_at);

-- ============================================================
-- PROJECTION: Issuance Dashboard (admin console)
-- ============================================================

CREATE TABLE proj_issuance_stats (
    tenant_id           UUID NOT NULL,
    schema_id           UUID NOT NULL,
    date                DATE NOT NULL,
    issued_count        INT NOT NULL DEFAULT 0,
    revoked_count       INT NOT NULL DEFAULT 0,
    verified_count      INT NOT NULL DEFAULT 0,
    PRIMARY KEY (tenant_id, schema_id, date)
);

-- ============================================================
-- PROJECTION: Wallet Contents
-- ============================================================

CREATE TABLE proj_wallet_contents (
    wallet_id           UUID NOT NULL,
    credential_id       UUID NOT NULL,
    schema_name         VARCHAR(500),
    issuer_did          VARCHAR(2048),
    issuance_date       TIMESTAMPTZ,
    expiration_date     TIMESTAMPTZ,
    status              VARCHAR(20),
    added_at            TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (wallet_id, credential_id)
);
```

---

## Projection Rebuild Process

```python
class CredentialProjectionRebuilder:
    """Rebuilds proj_credentials from the event store."""

    def rebuild(self):
        truncate('proj_credentials')
        events = event_store.query(
            aggregate_type='Credential',
            event_types=['CredentialIssued', 'CredentialRevoked', 'CredentialSuspended', 'CredentialReinstated'],
            order_by='created_at ASC'
        )
        for event in events:
            self.apply(event)

    def apply(self, event):
        match event.event_type:
            case 'CredentialIssued':
                upsert_proj_credential(event.payload)
            case 'CredentialRevoked':
                update_proj_credential_status(event.payload['credential_id'], 'revoked', event.created_at)
            case 'CredentialSuspended':
                update_proj_credential_status(event.payload['credential_id'], 'suspended', event.created_at)
            case 'CredentialReinstated':
                update_proj_credential_status(event.payload['credential_id'], 'active', None)
```

---

## Trade-offs

**Strengths:**
- Perfect audit trail by construction -- every state change is an immutable, timestamped event. This aligns with GDPR accountability and eIDAS 2.0 trust framework logging requirements.
- Read and write sides scale independently. The verification API (read-heavy) can run against denormalised projections with sub-millisecond latency while the issuance side maintains full consistency.
- Temporal queries are trivial: "What was the state of this DID at time T?" is answered by replaying events up to T.
- Event replay enables new projections without data migration -- add a new read model and rebuild from the event stream.
- Natural fit for distributed systems: events can be published to Kafka/NATS for cross-service consumption.

**Weaknesses:**
- Operational complexity: teams must manage event versioning, projection lag, and idempotent event handlers.
- Eventual consistency between the event store and projections means a just-issued credential might not appear in the verification projection for a few milliseconds. For most identity use cases this is acceptable, but it must be documented.
- Storage grows monotonically. Snapshotting is needed for aggregates with long event histories.
- GDPR right-to-erasure conflicts with append-only semantics. Requires crypto-shredding (encrypting PII in events with per-subject keys and deleting the key on erasure request).

## Scalability Considerations

- Partition the event store by `aggregate_type` or by `tenant_id` for large multi-tenant deployments.
- Use Kafka or NATS JetStream as the event bus between the event store and projection updaters.
- Snapshot aggregates with more than 100 events to bound replay time.
- Projections can be hosted in different databases (Redis for hot verification cache, PostgreSQL for admin queries).

## Migration Path

This model can be introduced incrementally alongside a relational schema (Suggestion 1) by adding an event store table and publishing events from existing write operations. Over time, the relational tables become projections of the event stream. This "strangler fig" approach avoids a big-bang migration.
