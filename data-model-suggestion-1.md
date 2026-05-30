# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF) relational schema in PostgreSQL. Every entity is its own table with strict foreign keys, check constraints, and appropriate indexes. This is the most familiar pattern for enterprise teams and offers strong ACID guarantees, mature tooling, and well-understood operational characteristics.

## Why This Suits a Decentralized Identity Platform

Decentralized identity has a core set of entities with well-defined relationships: DIDs own key material, issuers create credential schemas, credentials reference both a schema and a holder DID, presentations bundle credentials for a verifier, and revocation registries track credential status. These relationships map cleanly to a relational model. The W3C VC Data Model 2.0 defines a relatively stable structure, so schema rigidity is an asset rather than a liability for the core trust layer.

The main limitation is flexibility: DID documents and credential claims vary widely across use cases. A pure 3NF model handles this through EAV-style claim tables, which work but add query complexity. Nested JSON-LD contexts and linked-data graphs also lose their native shape when forced into rows and columns.

---

## Schema Definition

```sql
-- ============================================================
-- ENUMS
-- ============================================================

CREATE TYPE did_method AS ENUM ('did:web', 'did:ion', 'did:key', 'did:ethr', 'did:hedera', 'did:peer');
CREATE TYPE key_type AS ENUM ('Ed25519VerificationKey2020', 'JsonWebKey2020', 'EcdsaSecp256k1VerificationKey2019', 'X25519KeyAgreementKey2020');
CREATE TYPE key_purpose AS ENUM ('authentication', 'assertionMethod', 'keyAgreement', 'capabilityInvocation', 'capabilityDelegation');
CREATE TYPE credential_status AS ENUM ('active', 'revoked', 'suspended', 'expired');
CREATE TYPE entity_type AS ENUM ('person', 'organisation', 'device', 'ai_agent', 'service');
CREATE TYPE proof_format AS ENUM ('jwt', 'json-ld', 'sd-jwt', 'anoncreds');
CREATE TYPE wallet_type AS ENUM ('mobile_ios', 'mobile_android', 'browser_extension', 'server_agent');

-- ============================================================
-- CORE IDENTITY
-- ============================================================

CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    data_residency  VARCHAR(10) NOT NULL DEFAULT 'eu',  -- ISO 3166-1 alpha-2
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE did_subjects (
    subject_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    entity_type     entity_type NOT NULL DEFAULT 'person',
    display_name    VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deactivated_at  TIMESTAMPTZ,
    CONSTRAINT chk_deactivated CHECK (deactivated_at IS NULL OR deactivated_at > created_at)
);

CREATE INDEX idx_did_subjects_tenant ON did_subjects(tenant_id);

CREATE TABLE did_documents (
    did             VARCHAR(2048) PRIMARY KEY,  -- the full DID URI
    subject_id      UUID NOT NULL REFERENCES did_subjects(subject_id),
    method          did_method NOT NULL,
    controller_did  VARCHAR(2048) REFERENCES did_documents(did),
    also_known_as   TEXT[],                      -- array of alternative identifiers
    service_endpoints JSONB,                     -- service block from DID document
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deactivated_at  TIMESTAMPTZ
);

CREATE INDEX idx_did_documents_subject ON did_documents(subject_id);
CREATE INDEX idx_did_documents_method ON did_documents(method);

-- ============================================================
-- KEY MATERIAL
-- ============================================================

CREATE TABLE verification_methods (
    vm_id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    did                 VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    key_id              VARCHAR(2048) NOT NULL,  -- fragment URI e.g. did:web:example.com#key-1
    key_type            key_type NOT NULL,
    public_key_jwk      JSONB,                   -- JWK representation
    public_key_multibase VARCHAR(500),            -- multibase representation
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at          TIMESTAMPTZ,
    UNIQUE(did, key_id)
);

CREATE INDEX idx_vm_did ON verification_methods(did);

CREATE TABLE verification_relationships (
    relationship_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vm_id           UUID NOT NULL REFERENCES verification_methods(vm_id),
    purpose         key_purpose NOT NULL,
    UNIQUE(vm_id, purpose)
);

-- ============================================================
-- CREDENTIAL SCHEMAS
-- ============================================================

CREATE TABLE credential_schemas (
    schema_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(500) NOT NULL,
    version         VARCHAR(50) NOT NULL DEFAULT '1.0',
    schema_uri      VARCHAR(2048),               -- published JSON-LD context URI
    json_schema     JSONB NOT NULL,              -- JSON Schema for claim validation
    context_urls    TEXT[] NOT NULL,             -- @context array
    credential_type TEXT[] NOT NULL,             -- type array e.g. ['VerifiableCredential','UniversityDegree']
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name, version)
);

CREATE INDEX idx_credential_schemas_tenant ON credential_schemas(tenant_id);

CREATE TABLE schema_attributes (
    attribute_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_id       UUID NOT NULL REFERENCES credential_schemas(schema_id) ON DELETE CASCADE,
    attribute_name  VARCHAR(255) NOT NULL,
    data_type       VARCHAR(50) NOT NULL,         -- string, integer, date, boolean, etc.
    is_required     BOOLEAN NOT NULL DEFAULT false,
    is_pii          BOOLEAN NOT NULL DEFAULT false,
    display_order   INT NOT NULL DEFAULT 0,
    UNIQUE(schema_id, attribute_name)
);

-- ============================================================
-- VERIFIABLE CREDENTIALS
-- ============================================================

CREATE TABLE verifiable_credentials (
    credential_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(tenant_id),
    schema_id           UUID NOT NULL REFERENCES credential_schemas(schema_id),
    issuer_did          VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    holder_did          VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    issuance_date       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expiration_date     TIMESTAMPTZ,
    status              credential_status NOT NULL DEFAULT 'active',
    proof_format        proof_format NOT NULL DEFAULT 'json-ld',
    credential_hash     BYTEA NOT NULL,           -- SHA-256 hash of the signed credential
    signed_credential   TEXT,                     -- the full signed VC (JWT or JSON-LD proof)
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_expiry CHECK (expiration_date IS NULL OR expiration_date > issuance_date)
);

CREATE INDEX idx_vc_tenant ON verifiable_credentials(tenant_id);
CREATE INDEX idx_vc_issuer ON verifiable_credentials(issuer_did);
CREATE INDEX idx_vc_holder ON verifiable_credentials(holder_did);
CREATE INDEX idx_vc_schema ON verifiable_credentials(schema_id);
CREATE INDEX idx_vc_status ON verifiable_credentials(status);
CREATE INDEX idx_vc_issuance ON verifiable_credentials(issuance_date);

CREATE TABLE credential_claims (
    claim_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    credential_id   UUID NOT NULL REFERENCES verifiable_credentials(credential_id) ON DELETE CASCADE,
    attribute_id    UUID NOT NULL REFERENCES schema_attributes(attribute_id),
    claim_value     TEXT NOT NULL,                -- stored encrypted for PII attributes
    UNIQUE(credential_id, attribute_id)
);

CREATE INDEX idx_claims_credential ON credential_claims(credential_id);

-- ============================================================
-- REVOCATION
-- ============================================================

CREATE TABLE revocation_registries (
    registry_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    issuer_did      VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    registry_type   VARCHAR(50) NOT NULL DEFAULT 'StatusList2021',
    status_list_uri VARCHAR(2048),               -- published URI of the bitstring status list
    bitstring       BYTEA,                       -- compressed bitstring for StatusList2021
    max_credentials INT NOT NULL DEFAULT 131072,  -- 2^17 entries per list
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_registry_issuer ON revocation_registries(issuer_did);

CREATE TABLE revocation_entries (
    entry_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    registry_id     UUID NOT NULL REFERENCES revocation_registries(registry_id),
    credential_id   UUID NOT NULL REFERENCES verifiable_credentials(credential_id),
    status_index    INT NOT NULL,                -- position in the bitstring
    revoked_at      TIMESTAMPTZ,
    suspension_at   TIMESTAMPTZ,
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
    requested_schemas UUID[] NOT NULL,           -- schemas the verifier accepts
    requested_attributes TEXT[],                 -- specific attributes needed (selective disclosure)
    zk_predicate    JSONB,                       -- ZK predicate specification if applicable
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE verifiable_presentations (
    presentation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID REFERENCES presentation_requests(request_id),
    holder_did      VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    verifier_did    VARCHAR(2048) NOT NULL REFERENCES did_documents(did),
    proof_format    proof_format NOT NULL,
    signed_presentation TEXT,                    -- the full VP with proof
    is_verified     BOOLEAN,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vp_holder ON verifiable_presentations(holder_did);
CREATE INDEX idx_vp_verifier ON verifiable_presentations(verifier_did);
CREATE INDEX idx_vp_created ON verifiable_presentations(created_at);

CREATE TABLE presentation_credentials (
    presentation_id UUID NOT NULL REFERENCES verifiable_presentations(presentation_id) ON DELETE CASCADE,
    credential_id   UUID NOT NULL REFERENCES verifiable_credentials(credential_id),
    disclosed_attributes TEXT[],                 -- attributes actually disclosed in this presentation
    PRIMARY KEY (presentation_id, credential_id)
);

-- ============================================================
-- WALLET
-- ============================================================

CREATE TABLE wallets (
    wallet_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject_id      UUID NOT NULL REFERENCES did_subjects(subject_id),
    wallet_type     wallet_type NOT NULL,
    device_info     VARCHAR(500),
    biometric_bound BOOLEAN NOT NULL DEFAULT false,
    backup_enabled  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_active_at  TIMESTAMPTZ
);

CREATE INDEX idx_wallets_subject ON wallets(subject_id);

CREATE TABLE wallet_credentials (
    wallet_id       UUID NOT NULL REFERENCES wallets(wallet_id) ON DELETE CASCADE,
    credential_id   UUID NOT NULL REFERENCES verifiable_credentials(credential_id),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_favourite    BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (wallet_id, credential_id)
);

-- ============================================================
-- NON-HUMAN IDENTITY (IoT / AI AGENTS)
-- ============================================================

CREATE TABLE device_registrations (
    device_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject_id      UUID NOT NULL REFERENCES did_subjects(subject_id),
    device_type     VARCHAR(100) NOT NULL,       -- 'iot_sensor', 'ai_agent', 'service_mesh', etc.
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    firmware_version VARCHAR(100),
    attestation_type VARCHAR(100),               -- 'tpm', 'secure_enclave', 'strongbox', 'fido2', 'mcp-i'
    mcp_i_profile   JSONB,                       -- MCP-I identity metadata for AI agents
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_heartbeat  TIMESTAMPTZ
);

CREATE INDEX idx_device_subject ON device_registrations(subject_id);
CREATE INDEX idx_device_type ON device_registrations(device_type);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    actor_did       VARCHAR(2048),
    action          VARCHAR(100) NOT NULL,       -- 'did.create', 'vc.issue', 'vc.revoke', 'vp.verify', etc.
    resource_type   VARCHAR(100) NOT NULL,
    resource_id     VARCHAR(2048) NOT NULL,
    outcome         VARCHAR(20) NOT NULL DEFAULT 'success',
    ip_address      INET,
    metadata        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_action ON audit_log(tenant_id, action);
CREATE INDEX idx_audit_created ON audit_log(created_at);
CREATE INDEX idx_audit_actor ON audit_log(actor_did);

-- ============================================================
-- COMPLIANCE AND RETENTION
-- ============================================================

CREATE TABLE retention_policies (
    policy_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    resource_type   VARCHAR(100) NOT NULL,       -- 'credential', 'presentation', 'audit_log'
    retention_days  INT NOT NULL,
    anonymise       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, resource_type)
);

-- ============================================================
-- OID4VC / OIDC BRIDGE
-- ============================================================

CREATE TABLE oidc_clients (
    client_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    client_secret_hash BYTEA NOT NULL,
    redirect_uris   TEXT[] NOT NULL,
    allowed_schemas UUID[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE oid4vc_sessions (
    session_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES oidc_clients(client_id),
    holder_did      VARCHAR(2048),
    state           VARCHAR(255) NOT NULL UNIQUE,
    nonce           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    credential_offer JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_oid4vc_state ON oid4vc_sessions(state);
```

---

## Trade-offs

**Strengths:**
- Strong referential integrity across the entire credential lifecycle.
- Mature ecosystem: backups, replication, monitoring, and migration tools are well understood.
- Row-level security in PostgreSQL enables clean multi-tenancy isolation.
- ACID transactions guarantee that credential issuance and revocation are atomic.
- Easily auditable -- every relationship is explicit and queryable via standard SQL.

**Weaknesses:**
- Credential claims stored in an EAV-style table (`credential_claims`) require pivoting for full-credential reconstruction, which is slower than storing the credential as a single document.
- DID documents and service endpoints have variable structure that does not fit cleanly into fixed columns -- hence the `service_endpoints JSONB` escape hatch, which breaks pure 3NF.
- Schema evolution (adding new DID methods, credential formats) requires `ALTER TYPE` for enums and potentially new columns.
- Graph-like queries (e.g., "find all credentials transitively trusted through a chain of delegated issuers") are expensive in relational SQL.

## Scalability Considerations

- Partition `audit_log` by month using PostgreSQL declarative partitioning.
- Partition `verifiable_credentials` by `tenant_id` for large multi-tenant deployments.
- Read replicas can serve verification API traffic, which is read-heavy.
- The `revocation_registries.bitstring` column allows StatusList2021 to be served as a single binary blob, matching the W3C spec's publication model.

## Migration Path

This schema can evolve toward the hybrid JSONB model (Suggestion 3) by adding JSONB columns alongside existing relational columns and gradually migrating queries. The normalized structure also maps directly to an event-sourced model (Suggestion 2) by treating each INSERT/UPDATE as a materialised projection of an underlying event stream.
