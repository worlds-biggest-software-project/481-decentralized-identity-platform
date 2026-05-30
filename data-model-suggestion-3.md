# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A pragmatic hybrid that keeps core relational structure (foreign keys, constraints, indexes) for entities with stable schemas while using PostgreSQL JSONB columns for inherently flexible or standards-defined data such as DID documents, credential claims, proof objects, and protocol-specific metadata. GIN indexes on JSONB columns enable fast querying without sacrificing flexibility.

## Why This Suits a Decentralized Identity Platform

The decentralized identity domain has a dual personality. The platform's own operational entities -- tenants, wallets, audit logs, OIDC clients -- have predictable, stable schemas that benefit from relational rigidity. But the core W3C-standardised objects -- DID documents, verifiable credentials, verifiable presentations -- are JSON-LD documents with highly variable structures depending on the credential type, DID method, and proof format.

Forcing a university degree credential and a mobile driver's licence into the same relational column set is awkward. Decomposing credential claims into an EAV table (as in Suggestion 1) works but adds query complexity. Storing the entire credential as a single JSONB blob loses relational guarantees on issuer/holder relationships.

The hybrid model solves this by keeping the relational envelope (issuer_did, holder_did, status, dates) as typed columns with foreign keys while embedding the variable-structure content (claims, DID document body, proof chains) in JSONB. This matches how the W3C specs actually work: the outer structure is fixed, the inner payload varies.

---

## Schema Definition

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "pgcrypto";      -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "btree_gin";     -- composite GIN indexes

-- ============================================================
-- ENUMS (stable dimensions only)
-- ============================================================

CREATE TYPE entity_type AS ENUM ('person', 'organisation', 'device', 'ai_agent', 'service');
CREATE TYPE credential_status AS ENUM ('active', 'revoked', 'suspended', 'expired');
CREATE TYPE wallet_type AS ENUM ('mobile_ios', 'mobile_android', 'browser_extension', 'server_agent');

-- ============================================================
-- TENANTS (pure relational)
-- ============================================================

CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    data_residency  VARCHAR(10) NOT NULL DEFAULT 'eu',
    settings        JSONB NOT NULL DEFAULT '{}',   -- tenant-level config: retention, features, branding
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DID SUBJECTS
-- ============================================================

CREATE TABLE did_subjects (
    subject_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    entity_type     entity_type NOT NULL DEFAULT 'person',
    display_name    VARCHAR(500),
    profile         JSONB NOT NULL DEFAULT '{}',   -- extensible profile: device metadata, org info, MCP-I config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deactivated_at  TIMESTAMPTZ
);

CREATE INDEX idx_subjects_tenant ON did_subjects(tenant_id);
CREATE INDEX idx_subjects_entity ON did_subjects(entity_type);
CREATE INDEX idx_subjects_profile ON did_subjects USING GIN (profile);

-- ============================================================
-- DID DOCUMENTS (relational envelope + JSONB body)
-- ============================================================

CREATE TABLE did_documents (
    did             VARCHAR(2048) PRIMARY KEY,
    subject_id      UUID NOT NULL REFERENCES did_subjects(subject_id),
    method          VARCHAR(50) NOT NULL,         -- 'did:web', 'did:ion', 'did:key', etc.
    controller_did  VARCHAR(2048) REFERENCES did_documents(did),

    -- The full DID document as per W3C DID Core spec, stored verbatim
    document        JSONB NOT NULL,

    -- Extracted for fast lookups without JSONB traversal
    also_known_as   TEXT[],
    is_active       BOOLEAN NOT NULL DEFAULT true,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_did_subject ON did_documents(subject_id);
CREATE INDEX idx_did_method ON did_documents(method);
CREATE INDEX idx_did_active ON did_documents(is_active) WHERE is_active = true;

-- GIN index on the full document for flexible queries
-- e.g. find DIDs by verification method type or service endpoint type
CREATE INDEX idx_did_document_gin ON did_documents USING GIN (document jsonb_path_ops);

-- ============================================================
-- VERIFICATION METHODS (extracted for relational queries)
-- ============================================================

CREATE TABLE verification_methods (
    vm_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    did             VARCHAR(2048) NOT NULL REFERENCES did_documents(did) ON DELETE CASCADE,
    key_id          VARCHAR(2048) NOT NULL,
    key_type        VARCHAR(100) NOT NULL,
    purposes        TEXT[] NOT NULL,               -- ['authentication', 'assertionMethod']
    key_material    JSONB NOT NULL,                -- JWK or multibase, format varies by key type
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    UNIQUE(did, key_id)
);

CREATE INDEX idx_vm_did ON verification_methods(did);
CREATE INDEX idx_vm_purposes ON verification_methods USING GIN (purposes);
CREATE INDEX idx_vm_key_type ON verification_methods(key_type);

-- ============================================================
-- CREDENTIAL SCHEMAS
-- ============================================================

CREATE TABLE credential_schemas (
    schema_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(500) NOT NULL,
    version         VARCHAR(50) NOT NULL DEFAULT '1.0',
    credential_type TEXT[] NOT NULL,              -- ['VerifiableCredential', 'UniversityDegree']
    context_urls    TEXT[] NOT NULL,

    -- The full JSON Schema for validating credential subjects
    json_schema     JSONB NOT NULL,

    -- Display and compliance metadata
    metadata        JSONB NOT NULL DEFAULT '{}',  -- { "display_hints": {...}, "compliance": ["eidas2", "mdl"], "pii_fields": [...] }

    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name, version)
);

CREATE INDEX idx_schemas_tenant ON credential_schemas(tenant_id);
CREATE INDEX idx_schemas_type ON credential_schemas USING GIN (credential_type);
CREATE INDEX idx_schemas_metadata ON credential_schemas USING GIN (metadata jsonb_path_ops);

-- ============================================================
-- VERIFIABLE CREDENTIALS (relational envelope + JSONB claims)
-- ============================================================

CREATE TABLE verifiable_credentials (
    credential_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
    schema_id           UUID NOT NULL REFERENCES credential_schemas(schema_id),
    issuer_did          VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    holder_did          VARCHAR(2048) NOT NULL REFERENCES did_documents(did),

    -- Temporal envelope
    issuance_date       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expiration_date     TIMESTAMPTZ,

    -- Status
    status              credential_status NOT NULL DEFAULT 'active',
    proof_format        VARCHAR(20) NOT NULL DEFAULT 'json-ld',

    -- The credential subject claims as JSONB (varies per schema)
    claims              JSONB NOT NULL,

    -- The complete signed credential (JWT string or JSON-LD with proof)
    signed_credential   TEXT,

    -- Integrity
    credential_hash     BYTEA NOT NULL,

    -- Revocation cross-reference
    revocation_registry_id UUID,
    status_index        INT,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_expiry CHECK (expiration_date IS NULL OR expiration_date > issuance_date)
);

CREATE INDEX idx_vc_tenant ON verifiable_credentials(tenant_id);
CREATE INDEX idx_vc_issuer ON verifiable_credentials(issuer_did);
CREATE INDEX idx_vc_holder ON verifiable_credentials(holder_did);
CREATE INDEX idx_vc_schema ON verifiable_credentials(schema_id);
CREATE INDEX idx_vc_status ON verifiable_credentials(status);
CREATE INDEX idx_vc_issuance ON verifiable_credentials(issuance_date);

-- GIN index on claims for querying credential content
-- e.g. find all credentials where claims->>'degree' = 'Bachelor of Science'
CREATE INDEX idx_vc_claims ON verifiable_credentials USING GIN (claims jsonb_path_ops);

-- Partial index for active credentials only (most common query path)
CREATE INDEX idx_vc_active ON verifiable_credentials(issuer_did, holder_did)
    WHERE status = 'active';

-- ============================================================
-- REVOCATION REGISTRIES
-- ============================================================

CREATE TABLE revocation_registries (
    registry_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    issuer_did      VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    registry_type   VARCHAR(50) NOT NULL DEFAULT 'StatusList2021',
    status_list_uri VARCHAR(2048),
    bitstring       BYTEA,
    max_credentials INT NOT NULL DEFAULT 131072,
    metadata        JSONB NOT NULL DEFAULT '{}',  -- registry-specific config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_issuer ON revocation_registries(issuer_did);

CREATE TABLE revocation_entries (
    entry_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    registry_id     UUID NOT NULL REFERENCES revocation_registries(registry_id),
    credential_id   UUID NOT NULL REFERENCES verifiable_credentials(credential_id),
    status_index    INT NOT NULL,
    revoked_at      TIMESTAMPTZ,
    suspended_at    TIMESTAMPTZ,
    reason          VARCHAR(500),
    UNIQUE(registry_id, status_index),
    UNIQUE(registry_id, credential_id)
);

-- ============================================================
-- PRESENTATIONS AND VERIFICATION
-- ============================================================

CREATE TABLE presentation_requests (
    request_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    verifier_did    VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    nonce           VARCHAR(255) NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,

    -- Request specification as JSONB: schemas, attributes, ZK predicates, input descriptors
    request_spec    JSONB NOT NULL,
    /*
      Example request_spec:
      {
        "requested_schemas": ["uuid1", "uuid2"],
        "input_descriptors": [
          {
            "id": "degree_check",
            "schema": "uuid1",
            "constraints": {
              "fields": [
                { "path": ["$.degree"], "filter": { "type": "string" } },
                { "path": ["$.gpa"], "predicate": ">=", "value": 3.0 }
              ]
            }
          }
        ]
      }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE verifiable_presentations (
    presentation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID REFERENCES presentation_requests(request_id),
    holder_did      VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    verifier_did    VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    proof_format    VARCHAR(20) NOT NULL,

    -- Credentials included and what was disclosed
    included_credentials JSONB NOT NULL,
    /*
      Example:
      [
        {
          "credential_id": "uuid",
          "disclosed_attributes": ["degree", "field"],
          "zk_proofs": { "gpa_gte_3": true }
        }
      ]
    */

    -- Verification result
    verification_result JSONB,
    /*
      Example:
      {
        "is_verified": true,
        "checks": {
          "signature_valid": true,
          "not_revoked": true,
          "not_expired": true,
          "schema_match": true,
          "zk_predicate_satisfied": true
        },
        "verified_at": "2026-05-26T10:30:00Z"
      }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vp_holder ON verifiable_presentations(holder_did);
CREATE INDEX idx_vp_verifier ON verifiable_presentations(verifier_did);
CREATE INDEX idx_vp_created ON verifiable_presentations(created_at);
CREATE INDEX idx_vp_result ON verifiable_presentations USING GIN (verification_result jsonb_path_ops);

-- ============================================================
-- WALLETS
-- ============================================================

CREATE TABLE wallets (
    wallet_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject_id      UUID NOT NULL REFERENCES did_subjects(subject_id),
    wallet_type     wallet_type NOT NULL,
    biometric_bound BOOLEAN NOT NULL DEFAULT false,
    backup_enabled  BOOLEAN NOT NULL DEFAULT false,

    -- Device-specific metadata varies by platform
    device_info     JSONB NOT NULL DEFAULT '{}',
    /*
      iOS: { "model": "iPhone 15", "os_version": "18.2", "secure_enclave": true }
      Android: { "model": "Pixel 9", "os_version": "16", "strongbox": true }
      Browser: { "browser": "Chrome", "extension_version": "1.2.0" }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_active_at  TIMESTAMPTZ
);

CREATE INDEX idx_wallets_subject ON wallets(subject_id);

CREATE TABLE wallet_credentials (
    wallet_id       UUID NOT NULL REFERENCES wallets(wallet_id) ON DELETE CASCADE,
    credential_id   UUID NOT NULL REFERENCES verifiable_credentials(credential_id),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    display_config  JSONB NOT NULL DEFAULT '{}',  -- { "is_favourite": true, "display_order": 1, "card_color": "#2563eb" }
    PRIMARY KEY (wallet_id, credential_id)
);

-- ============================================================
-- NON-HUMAN IDENTITY
-- ============================================================

CREATE TABLE device_registrations (
    device_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject_id      UUID NOT NULL REFERENCES did_subjects(subject_id),
    device_type     VARCHAR(100) NOT NULL,

    -- All device/agent metadata in JSONB (varies wildly across device types)
    device_metadata JSONB NOT NULL DEFAULT '{}',
    /*
      IoT sensor:
      {
        "manufacturer": "Bosch",
        "model": "BME680",
        "firmware_version": "2.1.0",
        "attestation_type": "tpm",
        "tpm_endorsement_key": "..."
      }

      AI agent:
      {
        "agent_name": "billing-bot",
        "framework": "langchain",
        "attestation_type": "mcp-i",
        "mcp_i_profile": {
          "version": "1.0",
          "capabilities": ["invoice.read", "payment.initiate"],
          "trust_domain": "finance.example.com"
        }
      }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_heartbeat  TIMESTAMPTZ
);

CREATE INDEX idx_device_subject ON device_registrations(subject_id);
CREATE INDEX idx_device_type ON device_registrations(device_type);
CREATE INDEX idx_device_metadata ON device_registrations USING GIN (device_metadata jsonb_path_ops);

-- ============================================================
-- OID4VC / OIDC BRIDGE
-- ============================================================

CREATE TABLE oidc_clients (
    client_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    client_secret_hash BYTEA NOT NULL,
    redirect_uris   TEXT[] NOT NULL,
    allowed_schemas UUID[],
    oidc_config     JSONB NOT NULL DEFAULT '{}',  -- scopes, token lifetimes, PKCE settings
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE oid4vc_sessions (
    session_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES oidc_clients(client_id),
    holder_did      VARCHAR(2048),
    state           VARCHAR(255) NOT NULL UNIQUE,
    nonce           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',

    -- Protocol-specific session data (credential offer, authorization details)
    session_data    JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_oid4vc_state ON oid4vc_sessions(state);
CREATE INDEX idx_oid4vc_expires ON oid4vc_sessions(expires_at);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    actor_did       VARCHAR(2048),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     VARCHAR(2048) NOT NULL,
    outcome         VARCHAR(20) NOT NULL DEFAULT 'success',
    ip_address      INET,

    -- Extensible context: request headers, geolocation, risk score, etc.
    context         JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create partitions for the current and next quarter
CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_q3 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE audit_log_2026_q4 PARTITION OF audit_log
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_audit_tenant_action ON audit_log(tenant_id, action);
CREATE INDEX idx_audit_created ON audit_log(created_at);
CREATE INDEX idx_audit_actor ON audit_log(actor_did);
CREATE INDEX idx_audit_context ON audit_log USING GIN (context jsonb_path_ops);

-- ============================================================
-- RETENTION POLICIES
-- ============================================================

CREATE TABLE retention_policies (
    policy_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    resource_type   VARCHAR(100) NOT NULL,
    retention_days  INT NOT NULL,
    anonymise       BOOLEAN NOT NULL DEFAULT false,
    policy_config   JSONB NOT NULL DEFAULT '{}',  -- additional rules: archive destination, exemptions
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, resource_type)
);

-- ============================================================
-- FRAUD DETECTION SIGNALS (AI-native feature)
-- ============================================================

CREATE TABLE anomaly_signals (
    signal_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    subject_did     VARCHAR(2048) NOT NULL,
    signal_type     VARCHAR(100) NOT NULL,        -- 'unusual_frequency', 'geo_anomaly', 'credential_sharing'
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium',
    
    -- ML model output and supporting evidence
    signal_data     JSONB NOT NULL,
    /*
      {
        "model_version": "fraud-v2.3",
        "confidence": 0.87,
        "evidence": {
          "presentations_last_hour": 47,
          "baseline_hourly_avg": 2.3,
          "locations": ["Paris", "Tokyo", "Sydney"],
          "time_window_minutes": 15
        }
      }
    */

    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_tenant ON anomaly_signals(tenant_id);
CREATE INDEX idx_anomaly_subject ON anomaly_signals(subject_did);
CREATE INDEX idx_anomaly_unresolved ON anomaly_signals(tenant_id) WHERE resolved = false;
CREATE INDEX idx_anomaly_data ON anomaly_signals USING GIN (signal_data jsonb_path_ops);
```

---

## Query Examples

```sql
-- Find all active credentials for a holder with specific claim values
SELECT credential_id, claims, issuance_date, expiration_date
FROM verifiable_credentials
WHERE holder_did = 'did:key:z6Mk...'
  AND status = 'active'
  AND claims @> '{"degree": "Bachelor of Science"}';

-- Find all DIDs that have a specific service endpoint type
SELECT did, document->'service' AS services
FROM did_documents
WHERE document @> '{"service": [{"type": "LinkedDomains"}]}';

-- Find AI agents with specific MCP-I capabilities
SELECT d.device_id, d.device_metadata, dd.did
FROM device_registrations d
JOIN did_subjects ds ON d.subject_id = ds.subject_id
JOIN did_documents dd ON ds.subject_id = dd.subject_id
WHERE d.device_metadata @> '{"attestation_type": "mcp-i"}'
  AND d.device_metadata->'mcp_i_profile'->'capabilities' ? 'invoice.read';

-- Verification audit with ML anomaly correlation
SELECT vp.presentation_id, vp.holder_did, vp.verification_result, a.signal_data
FROM verifiable_presentations vp
LEFT JOIN anomaly_signals a ON a.subject_did = vp.holder_did
  AND a.created_at BETWEEN vp.created_at - INTERVAL '1 hour' AND vp.created_at + INTERVAL '1 hour'
WHERE vp.verifier_did = 'did:web:employer.com'
ORDER BY vp.created_at DESC;
```

---

## Trade-offs

**Strengths:**
- Best of both worlds: relational integrity for core relationships, JSONB flexibility for standards-defined payloads.
- DID documents and credential claims are stored in their native JSON-LD form, avoiding lossy decomposition.
- GIN indexes on JSONB columns support fast containment queries (`@>`) and key-exists checks (`?`) without schema rigidity.
- Single database engine (PostgreSQL) simplifies deployment, backup, and operational tooling.
- Schema evolution for credential types requires no DDL changes -- new claim shapes are just new JSONB values.
- Partial indexes (e.g., active credentials only) keep verification lookups fast even as historical data grows.

**Weaknesses:**
- JSONB columns are opaque to the database's type system. Application-layer validation (via `json_schema` in `credential_schemas`) must enforce claim structure.
- JSONB storage is larger than equivalent normalised columns due to repeated key names. For high-volume issuers, this can be significant.
- Complex JSONB queries (deep nesting, array element matching) can be slower than equivalent relational joins.
- Developers must know when to use relational columns vs JSONB -- the boundary is a design decision that needs team discipline.

## Scalability Considerations

- The audit log is already partitioned by time range. Extend this to `verifiable_credentials` if issuance volume exceeds tens of millions.
- JSONB compression via PostgreSQL TOAST handles large credential documents automatically.
- Read replicas serve the verification API; JSONB queries on GIN indexes are read-replica-friendly.
- Consider upgrading to PostgreSQL 17's `jsonpath` improvements for more efficient JSONB filtering.

## Migration Path

- From Suggestion 1 (pure relational): merge `credential_claims` and `schema_attributes` rows into a JSONB `claims` column per credential. A one-time migration script pivots the EAV data into JSON objects.
- From Suggestion 2 (event-sourced): this hybrid schema works as a natural projection target for event-sourced writes, combining the two approaches.
- Toward Suggestion 4 (graph): the JSONB columns can be exported to a graph database for relationship-heavy analytics while PostgreSQL remains the operational store.
