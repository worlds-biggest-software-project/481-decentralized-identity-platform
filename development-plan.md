# Decentralized Identity Platform — Phased Development Plan

> Project: 481-decentralized-identity-platform · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` proposals. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** because the platform's core objects (DID documents, verifiable credentials, presentations) are W3C JSON-LD documents with variable inner structure, while the operational envelope (issuer/holder relationships, status, tenancy) is stable and benefits from relational integrity. The hybrid model also carries the AI-native `anomaly_signals` table used in the fraud-detection phase.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **TypeScript (Node 22 LTS)** | The mature SSI ecosystem the platform must interoperate with — Veramo, Sphereon SSI-SDK, `@digitalbazaar/*` VC libraries, `did-jwt`, `jose` — is TypeScript-native. Avoids re-implementing W3C VC / OID4VC primitives. Shared types between API, SDK, and browser-extension wallet. |
| API framework | **Fastify 5** | High-throughput (verification API is read-heavy and latency-sensitive), first-class JSON Schema validation (every route validates request/response), native `@fastify/swagger` for OpenAPI 3.1 generation required by the relying-party SDK. |
| Database | **PostgreSQL 17** | Adopts data-model Suggestion 3. JSONB + GIN indexes store JSON-LD payloads natively; relational columns enforce issuer/holder integrity; declarative partitioning for `audit_log`; row-level security for multi-tenancy. |
| Query layer / migrations | **Drizzle ORM + drizzle-kit** | Typed SQL that maps cleanly onto the hybrid relational+JSONB schema without hiding JSONB operators (`@>`, `?`). Versioned SQL migrations checked into the repo. |
| Task queue | **BullMQ on Redis 7** | Async workloads: status-list publication, webhook delivery to verifiers, OID4VCI credential-offer polling, fraud-scoring jobs, retention sweeps. Redis also serves as the hot DID-resolution / verification cache. |
| Cryptography | **`jose` + `@noble/curves` + `@digitalbazaar/data-integrity`** | `jose` for JWT/SD-JWT (JOSE/COSE securing per W3C `vc-jose-cose`); `@noble/curves` for Ed25519/ES256/secp256k1; `@digitalbazaar` Data Integrity suites for `ecdsa-rdfc-2019`, `ecdsa-sd-2023`, `bbs-2023` JSON-LD proofs. |
| DID resolution | **Local resolvers for `did:web` / `did:key` + DIF Universal Resolver (driver subset) for ledger methods** | Keeps the verification layer ledger-agnostic per research §4. `did:web`/`did:key` need no network; `did:ion`/`did:ethr` proxy to Universal Resolver drivers. |
| Key management | **Pluggable KMS abstraction: local (AES-GCM-wrapped, dev) + AWS KMS / HashiCorp Vault (prod) + WebAuthn/Secure Enclave (wallet)** | Issuer keys must be hardware-backed in production; wallet keys target Secure Enclave / StrongBox / WebAuthn passkeys per standards.md WebAuthn L2. |
| Admin console | **Next.js 16 (App Router) + shadcn/ui + Tailwind** | Server-rendered issuer portal: schema management, issuance monitoring, audit log, fraud review. Reuses shared TS types from the core packages. |
| Browser-extension wallet | **WXT (Vite-based) + React** | Low-friction holder entry point (research risk mitigation). Reuses the holder SDK; keys in extension-managed storage with WebAuthn binding. |
| Mobile wallet | **React Native (Expo) — deferred to Phase 11** | iOS Secure Enclave / Android StrongBox via native modules. Out of the MVP critical path; the browser extension ships first. |
| LLM provider | **Provider-agnostic via Vercel AI SDK (default Anthropic Claude)** | AI-native features: schema-from-description, compliance mapping, natural-language presentation queries. SDK abstracts provider so deployments can swap models. |
| AI fraud scoring | **Python sidecar (FastAPI + scikit-learn) called over HTTP** | Anomaly detection (isolation forest / rolling z-score) is idiomatic in Python; isolates the ML runtime from the Node API. Communicates via the queue + a thin HTTP scoring endpoint. |
| Monorepo | **pnpm workspaces + Turborepo** | Multiple deployables (api, admin, extension, sdk, ml-sidecar) share schema/types packages; Turborepo caches builds and tests. |
| Testing | **Vitest (unit/integration) + Playwright (admin & extension e2e) + Testcontainers (real Postgres/Redis)** | Vitest for TS units; Testcontainers spins ephemeral Postgres/Redis for integration; Playwright for browser flows. pytest for the ML sidecar. |
| Code quality | **ESLint (typescript-eslint) + Prettier + `tsc --noEmit` + ruff/mypy (sidecar)** | Enforced in CI per Definition of Done. |
| Containerisation | **Docker + docker-compose (dev) / Helm chart (prod, Phase 12 backlog)** | Self-hosted, cloud, and hybrid deployment per README. compose brings up api, postgres, redis, ml-sidecar, admin. |
| Conformance | **OID4VC + EUDI ARF test vectors; W3C VC test suite** | Verifies OID4VCI/OID4VP/SIOPv2 interop and VC Data Model 2.0 compliance. |

### Project Structure

```
decentralized-identity-platform/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── tsconfig.base.json
├── .env.example
├── packages/
│   ├── core-types/                   # shared TS types: DID, VC, VP, schema, enums
│   │   └── src/
│   ├── db/                           # Drizzle schema + migrations (Suggestion 3)
│   │   ├── src/schema/               #   tenants, did_documents, verifiable_credentials, ...
│   │   ├── migrations/
│   │   └── src/client.ts
│   ├── crypto/                       # KMS abstraction, signers, JWK/multibase utils
│   │   └── src/
│   │       ├── kms/                  # LocalKms, AwsKms, VaultKms
│   │       ├── suites/               # ed25519, es256, bbs-2023, ecdsa-sd-2023
│   │       └── index.ts
│   ├── did/                          # DID method drivers + resolver gateway
│   │   └── src/
│   │       ├── methods/              # web, key, ion(proxy), ethr(proxy)
│   │       └── resolver.ts
│   ├── vc/                           # VC issuance/verification engine
│   │   └── src/
│   │       ├── issue.ts
│   │       ├── verify.ts
│   │       ├── statuslist.ts         # Bitstring Status List v1.0
│   │       └── sdjwt.ts              # SD-JWT VC (RFC 9901)
│   ├── oid4vc/                       # OID4VCI, OID4VP, SIOPv2, Presentation Exchange
│   │   └── src/
│   ├── ai/                           # LLM-backed schema design, compliance, NL queries
│   │   └── src/
│   └── holder-sdk/                   # holder/wallet client logic (shared by extension+mobile)
│       └── src/
├── apps/
│   ├── api/                          # Fastify server (issuer + verifier + OID4VC endpoints)
│   │   └── src/
│   │       ├── routes/
│   │       ├── plugins/              # auth, tenancy, rate-limit, swagger
│   │       ├── workers/              # BullMQ processors
│   │       └── server.ts
│   ├── admin/                        # Next.js issuer console
│   ├── extension/                    # WXT browser-extension wallet
│   └── ml-sidecar/                   # FastAPI fraud-scoring service (Python)
└── tests/
    ├── fixtures/                     # sample DIDs, VCs, presentations, test vectors
    └── e2e/
```

The structure is grouped by concern (crypto, did, vc, oid4vc, ai) so each phase extends a package without restructuring. `core-types` and `db` are foundational and imported everywhere.

---

## Phase 1: Foundation & Data Layer

### Purpose
Establish the monorepo, the PostgreSQL schema (Suggestion 3), shared types, configuration, and a running Fastify server with health checks and multi-tenant scaffolding. After this phase the team can run `docker compose up`, apply migrations, and hit a `/health` endpoint. Nothing identity-specific works yet, but every later phase builds on this skeleton.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap

**What**: Initialise the pnpm/Turborepo workspace with shared TS config, lint/format, and CI.

**Design**:
- `pnpm-workspace.yaml` declares `packages/*` and `apps/*`.
- `tsconfig.base.json`: `"target": "ES2023"`, `"module": "NodeNext"`, `"strict": true`, `"noUncheckedIndexedAccess": true`.
- `turbo.json` pipeline: `build` (depends on `^build`), `test`, `lint`, `typecheck`.
- Root `.env.example`:
  ```
  DATABASE_URL=postgresql://didp:didp@localhost:5432/didp
  REDIS_URL=redis://localhost:6379
  KMS_PROVIDER=local            # local | aws | vault
  KMS_LOCAL_MASTER_KEY=         # base64 32-byte key (dev only)
  PLATFORM_BASE_URL=http://localhost:3000
  LLM_PROVIDER=anthropic
  ANTHROPIC_API_KEY=
  LOG_LEVEL=info
  ```
- A typed config loader (`packages/core-types/src/config.ts`) parses `process.env` through a Zod schema; missing required vars throw at boot.

**Testing**:
- `Unit: config loader with all required env → typed Config object`
- `Unit: config loader missing DATABASE_URL → throws with "DATABASE_URL" in message`
- `Unit: KMS_PROVIDER=invalid → ZodError listing allowed values`
- `CI smoke: pnpm install && pnpm -r build succeeds`

#### 1.2 — Database schema & migrations (Suggestion 3)

**What**: Implement the hybrid relational+JSONB schema as Drizzle definitions plus the initial SQL migration.

**Design**:
- Port Suggestion 3 verbatim into `packages/db/src/schema/`. Tables: `tenants`, `did_subjects`, `did_documents`, `verification_methods`, `credential_schemas`, `verifiable_credentials`, `revocation_registries`, `revocation_entries`, `presentation_requests`, `verifiable_presentations`, `wallets`, `wallet_credentials`, `device_registrations`, `oidc_clients`, `oid4vc_sessions`, `audit_log` (range-partitioned by `created_at`), `retention_policies`, `anomaly_signals`.
- Enums: `entity_type`, `credential_status`, `wallet_type`.
- Extensions: `pgcrypto`, `btree_gin`.
- GIN indexes on `did_documents.document`, `verifiable_credentials.claims`, `credential_schemas.metadata`, `device_registrations.device_metadata`, `anomaly_signals.signal_data`.
- Row-Level Security: enable RLS on every tenant-scoped table; policy `tenant_isolation USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
- Drizzle client (`packages/db/src/client.ts`) exposes `withTenant(tenantId, fn)` that opens a transaction and `SET LOCAL app.tenant_id`.

**Testing** (Testcontainers Postgres):
- `Integration: run all migrations → all 18 tables + 3 enums exist`
- `Integration: insert credential referencing non-existent issuer_did → FK violation`
- `Integration: claims @> '{"degree":"BSc"}' query uses idx_vc_claims (EXPLAIN shows bitmap index scan)`
- `Integration: withTenant(A) cannot SELECT rows owned by tenant B (RLS)`
- `Integration: insert into audit_log routes to correct quarterly partition`
- `Unit: chk_expiry rejects expiration_date <= issuance_date`

#### 1.3 — Fastify server skeleton, tenancy & observability

**What**: Boot a Fastify app with health/readiness, tenancy resolution, structured logging, and OpenAPI scaffolding.

**Design**:
- `apps/api/src/server.ts` registers plugins: `@fastify/swagger` (OpenAPI 3.1 at `/openapi.json`, Swagger UI at `/docs`), `@fastify/rate-limit` (backed by Redis), `pino` logging, error handler emitting RFC 9457 Problem Details.
- Tenancy plugin resolves tenant from API key (`Authorization: Bearer <key>`) → `tenants.tenant_id`, attaches `req.tenant`, and runs handlers inside `withTenant`.
- Routes: `GET /health` (process up), `GET /ready` (DB + Redis reachable).
- Error contract (`packages/core-types`):
  ```ts
  interface ProblemDetails { type: string; title: string; status: number; detail?: string; instance?: string }
  ```

**Testing**:
- `Integration: GET /health → 200 {status:"ok"}`
- `Integration (mocked deps): GET /ready with DB down → 503`
- `Integration: request without API key on tenant route → 401 ProblemDetails`
- `Integration: request with valid key → req.tenant populated, RLS scoped`
- `Integration: exceed rate limit → 429 with Retry-After`

---

## Phase 2: Cryptography & Key Management

### Purpose
Build the cryptographic core every credential and DID operation depends on: a pluggable KMS, signature suites for the formats the platform must emit (Ed25519, ES256, secp256k1; BBS+ for selective disclosure), and JWK/multibase utilities. This phase ships no user-facing feature but unblocks DID creation (Phase 3) and credential signing (Phase 4).

### Tasks

#### 2.1 — KMS abstraction

**What**: A `KeyManagementService` interface with local, AWS KMS, and Vault implementations.

**Design**:
```ts
type KeyAlg = 'Ed25519' | 'ES256' | 'ES256K' | 'BLS12381G2';
interface ManagedKey { kid: string; alg: KeyAlg; publicKeyJwk: JsonWebKey; }
interface KeyManagementService {
  generateKey(alg: KeyAlg): Promise<ManagedKey>;
  sign(kid: string, data: Uint8Array): Promise<Uint8Array>;
  getPublicKey(kid: string): Promise<ManagedKey>;
  rotate(kid: string): Promise<ManagedKey>;        // returns new key, keeps old for verification
  destroy(kid: string): Promise<void>;
}
```
- `LocalKms`: private keys AES-256-GCM-wrapped under `KMS_LOCAL_MASTER_KEY`, stored in a `kms_keys` table (added by this phase's migration). Dev/self-host default.
- `AwsKms`: maps `kid` to a KMS key ARN; `sign` calls `KMS:Sign`. Only asymmetric specs KMS supports (Ed25519 via ECC, ES256, ES256K).
- `VaultKms`: HashiCorp Vault Transit engine.
- Factory selects implementation from `config.kms.provider`.

**Testing**:
- `Unit (LocalKms): generateKey('Ed25519') → kid + valid JWK; sign/verify round-trips`
- `Unit (LocalKms): stored private key is ciphertext, not plaintext (no raw d in DB)`
- `Unit (LocalKms): rotate keeps old kid resolvable for verification`
- `Integration (mocked AWS SDK): sign → KMS:Sign called with correct ARN; signature verifies against returned pubkey`
- `Unit: destroy(kid) then sign(kid) → KeyNotFoundError`

#### 2.2 — Signature suites & encoding utilities

**What**: Implement the proof suites and JWK ↔ multibase ↔ multicodec conversions.

**Design**:
- `packages/crypto/src/suites/`:
  - `ed25519` (`Ed25519Signature2020` / `eddsa-jcs-2022`)
  - `es256` and `es256k` JOSE signers (for JWT/SD-JWT VCs per `vc-jose-cose`)
  - `ecdsa-sd-2023` (selective disclosure)
  - `bbs-2023` (BBS+ derive-proof for ZK selective disclosure per `vc-di-bbs`)
- Encoding utils: `publicKeyJwkToMultibase`, `multibaseToJwk`, `didKeyFromJwk` (multicodec prefixes: `0xed01` Ed25519, `0x1200` P-256).
- Each suite implements:
  ```ts
  interface ProofSuite {
    readonly type: string;
    sign(doc: object, key: ManagedKey, kms: KeyManagementService): Promise<object>; // returns doc + proof
    verify(doc: object, resolver: DidResolver): Promise<boolean>;
    deriveProof?(doc: object, disclose: string[]): Promise<object>; // SD/BBS only
  }
  ```

**Testing** (fixture-based, W3C test vectors):
- `Unit: ed25519 sign then verify → true; tamper one byte → false`
- `Unit: didKeyFromJwk(Ed25519 jwk) → did:key:z6Mk... matching W3C vector`
- `Unit: bbs-2023 deriveProof discloses subset → verifies; undisclosed claims absent`
- `Unit: ecdsa-sd-2023 base proof + derived proof verify per W3C vc-di-ecdsa test vectors`
- `Unit: multibase round-trip jwk → multibase → jwk is identity`

---

## Phase 3: DID Issuance & Resolution

### Purpose
Deliver the platform's first identity capability: creating, publishing, and resolving DIDs. This is the trust anchor for everything else — issuers, holders, verifiers, and devices all need DIDs. Implements `did:web` and `did:key` natively (MVP) and proxies ledger methods to the Universal Resolver. After this phase a tenant can create a DID and any party can resolve it.

### Tasks

#### 3.1 — DID method drivers (create + resolve)

**What**: Driver implementations for `did:web` and `did:key`, plus a proxy driver for `did:ion`/`did:ethr`.

**Design**:
```ts
interface DidCreateResult { did: string; document: DidDocument; keys: ManagedKey[]; }
interface DidMethodDriver {
  readonly method: string;            // 'web' | 'key' | 'ion' | 'ethr'
  create(input: CreateDidInput): Promise<DidCreateResult>;
  resolve(did: string): Promise<DidResolutionResult>;  // W3C DID Resolution result envelope
  deactivate?(did: string): Promise<void>;
}
```
- `did:key`: derive DID deterministically from a generated key (no publication).
- `did:web`: DID = `did:web:<host>:<path>`; document published at `https://<host>/<path>/did.json`. The platform serves it from `GET /.well-known/did.json` and `GET /:tenant/dids/:id/did.json`.
- `did:ion` / `did:ethr`: `resolve` delegates to the configured Universal Resolver URL; `create` for these methods returns `NotSupportedError` in MVP (resolve-only).
- `DidDocument` typed in `core-types` per DID Core v1.1 (verificationMethod, authentication, assertionMethod, service).

**Testing**:
- `Unit: create did:key → DID matches multibase of the generated Ed25519 key`
- `Integration: create did:web → row in did_documents, document served at well-known URL, resolves to same doc`
- `Integration (mocked resolver): resolve did:ion:... → proxies to Universal Resolver, returns its document`
- `Unit: resolve malformed DID → notFound metadata in resolution result`
- `Unit: create did:ethr → NotSupportedError`

#### 3.2 — Resolver gateway & cache

**What**: A unified resolver that dispatches to the right driver and caches results in Redis.

**Design**:
- `DidResolver.resolve(did)` parses the method, dispatches to the driver, caches the resolution result in Redis with a TTL (default 300s; `did:key` cached indefinitely as it is deterministic).
- Exposes `GET /1.0/identifiers/:did` matching the Universal Resolver HTTP API contract so existing tooling works unchanged.
- Negative caching (60s) for `notFound` to avoid hammering external resolvers.

**Testing**:
- `Integration: GET /1.0/identifiers/<did:key> → 200 W3C resolution result`
- `Integration: second resolve of same DID → served from Redis (driver not called; assert via spy)`
- `Integration: resolve unknown method → 501 representationNotSupported`
- `Unit: cache key namespaced; TTL honoured`

#### 3.3 — DID management API

**What**: Tenant-facing endpoints to create, list, and deactivate DIDs.

**Design**:
- `POST /v1/dids` body `{ method, entityType, alsoKnownAs?, services? }` → creates subject + DID + keys (via KMS), persists document JSONB, returns `{ did, document }`.
- `GET /v1/dids` paginated list scoped to tenant.
- `POST /v1/dids/:did/deactivate` sets `is_active=false`, updates document.
- Every operation writes an `audit_log` row (`did.create`, `did.deactivate`).

**Testing**:
- `Integration: POST /v1/dids {method:"web"} → 201, did_documents row, kms_keys row, audit_log row`
- `Integration: POST /v1/dids cross-tenant key isolation (RLS) holds`
- `Integration: deactivate then resolve → document marked deactivated`
- `Integration: list returns only caller tenant's DIDs`

---

## Phase 4: Credential Schemas & Issuance

### Purpose
The heart of the product: issuing W3C Verifiable Credentials. Tenants register credential schemas (JSON-LD context + JSON Schema), then issue signed VCs to holder DIDs. This phase delivers the core value proposition — standards-compliant issuance — and is the precondition for verification (Phase 6).

### Tasks

#### 4.1 — Credential schema registry

**What**: CRUD for credential schemas with JSON Schema validation and publication.

**Design**:
- `POST /v1/schemas` body:
  ```ts
  interface CreateSchema {
    name: string; version?: string;
    credentialType: string[];        // ['VerifiableCredential','UniversityDegree']
    contextUrls: string[];           // ['https://www.w3.org/ns/credentials/v2', ...]
    jsonSchema: object;              // JSON Schema 2020-12 for credentialSubject
    metadata?: { complianceTargets?: ('eidas2'|'mdl'|'nist-800-63')[]; piiFields?: string[]; displayHints?: object };
  }
  ```
- Validates `jsonSchema` is a well-formed JSON Schema (compile with Ajv 2020). Stored in `credential_schemas.json_schema`.
- `POST /v1/schemas/:id/publish` flips `is_published=true` and (optionally) anchors the context to IPFS, returning the CID.

**Testing**:
- `Unit: invalid JSON Schema body → 422 ProblemDetails with schema-compile error`
- `Integration: create schema → row with credentialType[] and metadata JSONB`
- `Integration: duplicate (tenant,name,version) → 409`
- `Integration: publish → is_published true; context retrievable`

#### 4.2 — Credential issuance engine

**What**: Issue a signed VC conforming to W3C VC Data Model 2.0, in JSON-LD (Data Integrity) or SD-JWT format.

**Design**:
```ts
interface IssueCredentialInput {
  schemaId: string; issuerDid: string; holderDid: string;
  claims: Record<string, unknown>;
  proofFormat: 'json-ld' | 'sd-jwt' | 'jwt';
  expirationDate?: string;          // ISO 8601
  statusListEntry?: boolean;        // default true → allocate revocation index
}
```
Flow (in `packages/vc/src/issue.ts`):
1. Load issuer DID; assert active and has an `assertionMethod` verification method.
2. Validate `claims` against `schema.json_schema` (Ajv).
3. Build the VC object (`@context`, `type`, `issuer`, `validFrom`, `validUntil`, `credentialSubject` = claims + `id: holderDid`).
4. If `statusListEntry`, allocate `(registry_id, status_index)` from the issuer's revocation registry (Phase 5).
5. Sign per `proofFormat`: JSON-LD → Data Integrity proof (`ecdsa-rdfc-2019`/`bbs-2023`); SD-JWT → `dc+sd-jwt` per SD-JWT VC draft; JWT → `vc-jose-cose`.
6. Persist to `verifiable_credentials`: relational envelope + `claims` JSONB + `signed_credential` text + `credential_hash`.
7. Emit `audit_log` `vc.issue`.
- `POST /v1/credentials` wraps this; returns `{ credentialId, credential }`.

**Testing** (fixtures + Testcontainers):
- `Unit: claims violating schema → ValidationError naming the field`
- `Unit: issuer without assertionMethod → IssuerNotAuthorizedError`
- `Integration: issue json-ld VC → row persisted, signed_credential verifies with issuer pubkey`
- `Integration: issue sd-jwt VC → media type dc+sd-jwt, disclosures present`
- `Integration: issue with statusListEntry → revocation_entries row allocated, status_index unique`
- `E2E: create DID → register schema → issue VC → returned VC passes external W3C verifier`

#### 4.3 — Credential retrieval & holder binding

**What**: Endpoints to fetch issued credentials and bind them to a holder.

**Design**:
- `GET /v1/credentials/:id` → returns envelope + signed credential (tenant-scoped; holder may fetch own via wallet token).
- Holder binding uses `cnf`/`holder` claim so the credential is bound to the holder DID's key (prevents transfer — supports later biometric binding).

**Testing**:
- `Integration: GET own credential → 200; GET other tenant's → 404`
- `Unit: issued VC contains holder binding (cnf) referencing holder DID key`

---

## Phase 5: Revocation (Bitstring Status List)

### Purpose
Issuers must be able to revoke credentials without revealing which holder is affected, and verifiers must check status without contacting the issuer. Implements W3C Bitstring Status List v1.0 (the EUDI-mandated mechanism). After this phase the full issue→revoke→status-check lifecycle works.

### Tasks

#### 5.1 — Status list management & publication

**What**: Maintain per-issuer bitstrings and publish them as signed Status List credentials.

**Design**:
- `RevocationRegistry` allocates sequential `status_index` values (0..131071) per list; rolls to a new list when full.
- Revoke: set the bit at `status_index`, mark `revocation_entries.revoked_at`, regenerate the GZIP-compressed base64url bitstring, re-sign the Status List Credential, publish at `GET /v1/status-lists/:registryId` (and optionally IPFS).
- Publication runs as a BullMQ job (`statuslist.publish`) debounced per registry to batch rapid revocations.
- Status List Credential is itself a VC with `type: ["VerifiableCredential","BitstringStatusListCredential"]` and `credentialSubject.encodedList`.
- Issued credentials carry `credentialStatus`:
  ```json
  { "id":"<base>/v1/status-lists/<rid>#42","type":"BitstringStatusListEntry","statusPurpose":"revocation","statusListIndex":"42","statusListCredential":"<base>/v1/status-lists/<rid>" }
  ```

**Testing**:
- `Unit: allocate indices → strictly increasing, unique per registry`
- `Unit: set bit at index 42 → only that bit set in decoded bitstring`
- `Integration: revoke → published Status List Credential verifies and bit 42 = 1`
- `Integration: registry fills at 131072 → new registry created, allocation continues`
- `Integration: rapid 100 revokes → single debounced publish job, final list correct`

#### 5.2 — Revocation & suspension API

**What**: Endpoints to revoke, suspend, and reinstate credentials.

**Design**:
- `POST /v1/credentials/:id/revoke` `{ reason }` → sets `status='revoked'`, updates entry + bitstring, audit `vc.revoke`.
- `POST /v1/credentials/:id/suspend` and `/reinstate` (suspension uses a separate `statusPurpose:"suspension"` list).
- Idempotent: revoking an already-revoked credential returns 200 no-op.

**Testing**:
- `Integration: revoke active credential → status revoked, bit set, audit row`
- `Integration: revoke already-revoked → 200 idempotent, no duplicate audit`
- `Integration: suspend then reinstate → bit toggles on suspension list, status active`
- `Integration: revoke other tenant's credential → 404`

---

## Phase 6: Verification API & Relying-Party SDK

### Purpose
The read-heavy public-facing capability: relying parties verify credential authenticity, integrity, expiry, schema conformance, and revocation status without contacting the issuer. Ships a typed SDK so integrators add verification in a few lines. This is the universal-verifier differentiator from features.md.

### Tasks

#### 6.1 — Universal verification engine

**What**: Verify a VC or VP across DID methods and proof formats in one call.

**Design**:
```ts
interface VerifyResult {
  verified: boolean;
  checks: { signatureValid: boolean; notExpired: boolean; notRevoked: boolean; schemaMatch: boolean; holderBound?: boolean; predicatesSatisfied?: boolean };
  errors: string[];
  credential?: ParsedCredential;
}
function verifyCredential(input: string | object, opts?: { expectedSchema?: string }): Promise<VerifyResult>;
```
Flow:
1. Detect format (JSON-LD proof / JWT / SD-JWT) by structure.
2. Resolve issuer DID (Phase 3 resolver) → get assertionMethod public key.
3. Verify proof/signature (Phase 2 suites).
4. Check `validFrom`/`validUntil`.
5. Resolve `credentialStatus` → fetch Status List Credential, decode bit at index.
6. If `expectedSchema`, validate claims against it.
- No issuer contact at verification time — status comes from the published list (cacheable).

**Testing** (fixtures):
- `Unit: valid VC → verified true, all checks true`
- `Unit: tampered claim → signatureValid false, verified false`
- `Unit: expired VC → notExpired false`
- `Integration: revoked VC (bit set) → notRevoked false`
- `Unit: SD-JWT with selective disclosure → only disclosed claims present, verifies`
- `Unit: unknown DID method → error "unsupported method", verified false`

#### 6.2 — Verification REST API & webhooks

**What**: Public verification endpoints plus async webhook delivery of results.

**Design**:
- `POST /v1/verify/credential` `{ credential }` → `VerifyResult`.
- `POST /v1/verify/presentation` `{ presentation, presentationDefinition? }` → verifies VP + each VC + DIF Presentation Exchange constraints.
- Optional `callbackUrl`: result delivered via signed webhook (BullMQ `webhook.deliver`, HMAC `X-DIDP-Signature`, retry with backoff).
- All verifications recorded in `verifiable_presentations` / `audit_log` for the audit trail and fraud signals (Phase 9).

**Testing**:
- `Integration: POST /v1/verify/credential valid → 200 VerifyResult`
- `Integration: with callbackUrl → 202, webhook fired with HMAC, signature validates`
- `Integration (mocked endpoint 500): webhook retried with backoff, marked failed after N`
- `Integration: verification recorded in audit_log`

#### 6.3 — Relying-party SDK

**What**: A published `@didp/verifier` npm package wrapping the verification API and offering local verification.

**Design**:
```ts
const client = new DidpVerifier({ baseUrl, apiKey });
await client.verifyCredential(vc);                       // remote
await client.verifyLocally(vc, { resolver });            // offline, no API call
const req = await client.createPresentationRequest({ inputDescriptors }); // OID4VP-ready
```
- Generated from the OpenAPI 3.1 spec where possible; hand-written local-verify path reuses `packages/vc`.

**Testing**:
- `Unit: verifyCredential → calls POST /v1/verify/credential, parses VerifyResult`
- `Unit: verifyLocally → no HTTP, uses bundled verifier`
- `Integration: SDK against running api (Testcontainers) round-trips`

---

## Phase 7: OID4VC Protocol Layer

### Purpose
Interoperability with EUDI Wallets, Microsoft Authenticator, and any standards wallet. Implements OID4VCI (issuance), OID4VP (presentation), and SIOPv2, aligned to the HAIP profile. This unlocks government and cross-wallet use cases and is required for the browser/mobile wallets to receive and present credentials over open protocols.

### Tasks

#### 7.1 — OID4VCI issuance flow

**What**: Issue credentials to external wallets via the OpenID4VCI credential-offer + token + credential-endpoint flow.

**Design**:
- `POST /v1/oid4vci/offers` (issuer) → creates `oid4vc_sessions` row, returns a credential offer URI / QR (`openid-credential-offer://...`).
- `.well-known/openid-credential-issuer` metadata endpoint advertises supported credential configurations (formats: `dc+sd-jwt`, `jwt_vc_json`, `ldp_vc`).
- Token endpoint (pre-authorized_code and authorization_code+PKCE grants) issues an access token bound to the session.
- Credential endpoint validates the wallet's key proof (`jwt` proof type), then issues the VC (reuses Phase 4 engine) bound to the wallet key.

**Testing**:
- `Integration: create offer → session row, offer URI parseable`
- `Integration: well-known metadata → valid OID4VCI issuer metadata JSON`
- `Integration: pre-auth code → token → credential endpoint with valid key proof → SD-JWT VC bound to wallet key`
- `Unit: credential request with invalid proof JWT → invalid_proof error`
- `Conformance: flow passes OID4VCI HAIP test vectors`

#### 7.2 — OID4VP & Presentation Exchange

**What**: Verifiers request presentations from wallets; the platform validates submissions against a Presentation Definition.

**Design**:
- `POST /v1/oid4vp/requests` → builds an Authorization Request (with `presentation_definition` per DIF PE v2) and `request_uri`.
- Callback endpoint receives `vp_token` + `presentation_submission`, verifies (Phase 6) and checks the submission satisfies the definition's input descriptors / constraints (incl. predicate filters for ZK).
- Supports `direct_post` response mode (HAIP).

**Testing**:
- `Integration: create request → Authorization Request with presentation_definition`
- `Integration: valid vp_token satisfying definition → verified, submission matched`
- `Unit: submission missing required input descriptor → rejected with descriptor id`
- `Conformance: passes OID4VP HAIP direct_post vectors`

#### 7.3 — SIOPv2 & OIDC bridge

**What**: Let a VC presentation stand in for an OIDC login, and let holders self-issue identity (SIOPv2).

**Design**:
- OIDC bridge: `oidc_clients` register redirect URIs; `/authorize` triggers an OID4VP request; on success the platform mints an OIDC `id_token` whose claims derive from the disclosed VC attributes. Standard `/token` and `/userinfo` endpoints make it a drop-in OIDC provider.
- SIOPv2: accept self-issued `id_token` signed by the holder's DID key.

**Testing**:
- `Integration: OIDC authorize → VP request → id_token minted with mapped claims; /userinfo returns them`
- `Unit: id_token signature + nonce + aud validate`
- `Integration: SIOPv2 self-issued id_token from holder DID → accepted`

---

## Phase 8: Holder Wallet (Browser Extension)

### Purpose
The low-friction holder entry point (research UX-risk mitigation). A browser-extension wallet that receives credentials via OID4VCI, stores them with WebAuthn-bound keys, and presents them via OID4VP with selective disclosure. Establishes the holder SDK reused by the mobile wallet (Phase 11).

### Tasks

#### 8.1 — Holder SDK

**What**: Framework-agnostic holder logic: key generation/storage, credential storage, OID4VCI accept, OID4VP respond.

**Design**:
```ts
class HolderAgent {
  createDid(method: 'key'|'web'): Promise<string>;
  acceptOffer(offerUri: string): Promise<StoredCredential>;     // OID4VCI
  listCredentials(): Promise<StoredCredential[]>;
  respondToRequest(authRequest: string, selection: DisclosureSelection): Promise<void>; // OID4VP
}
```
- Keys held in a pluggable `KeyStore` (extension: WebAuthn-gated IndexedDB; mobile: Secure Enclave/StrongBox).
- Selective disclosure: for SD-JWT, choose which disclosures to release; for BBS, derive a proof over chosen claims.

**Testing**:
- `Unit (mocked api): acceptOffer → credential stored, key proof generated`
- `Unit: respondToRequest discloses only selected claims (SD-JWT)`
- `Unit: BBS derive presents subset, verifies`

#### 8.2 — Extension UI & WebAuthn binding

**What**: WXT/React extension to onboard, scan/paste offers, view credentials, and approve presentations behind biometric/WebAuthn unlock.

**Design**:
- Screens: Onboarding (create DID), Credential list (cards), Offer accept (review claims), Presentation approval (show verifier + requested claims + disclosure toggles).
- WebAuthn passkey gates key use (per W3C WebAuthn L2); private keys never leave the extension.
- Deep-link handlers for `openid-credential-offer://` and `openid4vp://`.

**Testing** (Playwright on a built extension):
- `E2E: onboard → DID created, shown`
- `E2E: paste offer URI → review → accept → credential appears`
- `E2E: presentation request → toggle off one claim → approve → verifier receives only selected claims`
- `Unit: WebAuthn assertion required before sign`

---

## Phase 9: Admin Console & AI-Native Features

### Purpose
Give non-developer issuers a portal and deliver the AI-native differentiators from features.md: schema-from-description, compliance mapping, natural-language presentation queries, and ML fraud detection. This is what positions the platform above open-source frameworks and below enterprise pricing.

### Tasks

#### 9.1 — Admin console

**What**: Next.js issuer portal: DIDs, schemas, issuance monitoring, audit log, revocation, fraud review.

**Design**:
- Auth via tenant API key / session. Pages: Dashboard (issuance/verification stats from `audit_log` aggregates), Schemas (CRUD + publish), Credentials (issue, search by `claims @>`, revoke), Audit Log (filter by action/actor/date), Fraud Signals (review/resolve `anomaly_signals`).
- Uses the same REST API; no direct DB access.

**Testing**:
- `E2E (Playwright): log in → create schema via form → issue credential → see it in dashboard count`
- `E2E: revoke from UI → status reflects revoked`
- `E2E: audit log filters by action`

#### 9.2 — AI schema design & compliance mapping

**What**: Generate a credential schema from a plain-language description and check it against eIDAS 2.0 / mDL / NIST requirements.

**Design**:
- `POST /v1/ai/schemas/draft` `{ description }` → LLM (Vercel AI SDK, structured output via Zod) returns `{ name, credentialType, contextUrls, jsonSchema, suggestedPiiFields }`. System prompt instructs strict W3C VC DM 2.0 / JSON-LD context conventions; output validated by the same Ajv check as 4.1.
- `POST /v1/ai/schemas/:id/compliance` `{ targets }` → LLM maps schema fields against a curated rules pack (eIDAS ARF mandatory PID attributes, ISO 18013-5 mDL data elements, NIST 800-63 IAL) and returns `{ target, satisfied, missing[], warnings[] }`. Rules pack is data, not model-trusted, so results are deterministic-checked where possible.

**Testing**:
- `Unit (mocked LLM): draft "nurse licence with name, licence number, expiry" → valid JSON Schema passing Ajv`
- `Unit: drafted schema with no @context → post-validation repairs/injects VC v2 context`
- `Unit (mocked LLM + rules pack): mDL compliance for schema missing portrait → satisfied=false, missing includes "portrait"`

#### 9.3 — Natural-language presentation queries

**What**: Verifiers express requests in natural language; the system builds the OID4VP Presentation Definition (incl. ZK predicates).

**Design**:
- `POST /v1/ai/presentation-requests/draft` `{ query }` e.g. "verify this person is a licensed nurse in France without revealing their full name" → LLM emits a DIF Presentation Definition with input descriptors and constraints (`limit_disclosure: required`, predicate filters), validated against the PE schema before use.

**Testing**:
- `Unit (mocked LLM): "over 18 without DOB" → input descriptor with predicate age>=18, limit_disclosure required, name excluded`
- `Unit: emitted definition validates against DIF PE v2 JSON Schema`

#### 9.4 — ML fraud detection sidecar

**What**: Score verification patterns for anomalies and write `anomaly_signals`.

**Design**:
- Python FastAPI sidecar: `POST /score` `{ subjectDid, recentPresentations[] }` → `{ signalType, severity, confidence, evidence }` using rolling z-score on presentation frequency + geo-velocity (impossible-travel) + device-fingerprint reuse.
- A BullMQ worker (`fraud.score`) triggers after each verification; high-severity signals surface in the admin Fraud page and can auto-suspend per tenant policy.
- Writes to `anomaly_signals` (the JSONB `signal_data` mirrors Suggestion 3's example shape).

**Testing**:
- `Unit (pytest): 47 presentations in 15min vs baseline 2.3/hr → unusual_frequency, high severity`
- `Unit: Paris→Tokyo in 10min → geo_anomaly`
- `Integration: verification triggers fraud.score job → anomaly_signals row; admin lists it`
- `Integration: tenant auto-suspend policy on high severity → credential suspended`

---

## Phase 10: Non-Human Identity (IoT & AI Agents)

### Purpose
Capture the underserved machine-identity market (144:1 ratio per research). First-class DID issuance/verification for IoT devices and AI agents, with attestation binding and MCP-I integration for agent identity.

### Tasks

#### 10.1 — Device & agent registration

**What**: Register devices/agents, issue them DIDs, and bind hardware/agent attestation.

**Design**:
- `POST /v1/devices` `{ deviceType, deviceMetadata }` → creates subject (`entity_type` device/ai_agent), DID (`did:key` or `did:web`), persists `device_registrations` with `device_metadata` JSONB (TPM endorsement key, attestation_type, MCP-I profile).
- Attestation verification adapters: `tpm`, `secure_enclave`, `strongbox`, `fido2`.
- Heartbeat endpoint updates `last_heartbeat`.

**Testing**:
- `Integration: register IoT sensor with tpm metadata → device row + DID`
- `Unit (mocked TPM): valid endorsement key → attestation accepted; tampered → rejected`
- `Integration: heartbeat updates last_heartbeat`

#### 10.2 — MCP-I agent identity

**What**: Issue and verify agent identity credentials under the emerging MCP-I standard.

**Design**:
- Agent VC carries `mcp_i_profile` (capabilities, trust_domain, framework) as `credentialSubject`.
- `POST /v1/agents/:did/verify-capability` `{ capability }` → checks the agent holds a valid, non-revoked credential granting that capability.
- MCP-I is unratified (standards.md note): isolate behind a versioned `mcp-i` adapter so the wire format can change without touching callers.

**Testing**:
- `Integration: issue agent credential with capabilities ["invoice.read"] → verify-capability "invoice.read" true, "payment.initiate" false`
- `Unit: revoked agent credential → capability check false`
- `Unit: adapter version pinned; unknown MCP-I version → 501`

---

## Phase 11: Mobile Wallet & Hardware-Backed Keys

### Purpose
Deliver the consumer-facing iOS/Android wallet with biometric unlock, hardware-backed keys (Secure Enclave / StrongBox), and backup/recovery — the table-stakes feature for mainstream and mDL use cases. Reuses the Phase 8 holder SDK.

### Tasks

#### 11.1 — React Native wallet on holder SDK

**What**: Expo app implementing the same flows as the extension with native key storage.

**Design**:
- Native `KeyStore` modules: iOS Secure Enclave (`SecKeyCreateRandomKey`, biometric ACL), Android StrongBox (`KeyStore` with `setIsStrongBoxBacked`). Falls back to TEE if StrongBox absent.
- Biometric unlock (Face ID / fingerprint) gates every signing operation.
- Screens mirror the extension; adds offline ISO 18013-5 mDL presentation (QR/NFC) for the mDL use case.

**Testing**:
- `Unit (mocked native module): sign requires biometric assertion`
- `E2E (Detox/iOS sim): onboard → accept offer → present credential`
- `Integration: key generated in Secure Enclave is non-exportable (assert export throws)`

#### 11.2 — Backup, recovery & social recovery

**What**: Encrypted backup and recovery without exposing private keys, plus optional social recovery (research mitigation for key loss).

**Design**:
- Backup: credentials + wrapped key shares encrypted under a recovery key derived from a user passphrase (Argon2id) and stored in the platform's backup blob store; private keys never leave the device unwrapped.
- Social recovery: Shamir secret sharing (`t`-of-`n`) of the recovery key across guardian contacts; reconstruct on a new device.
- Optional custodial recovery as a compliance toggle (research mitigation).

**Testing**:
- `Unit: backup blob decrypts only with correct passphrase-derived key`
- `Unit: 3-of-5 shares reconstruct recovery key; 2 shares fail`
- `Integration: restore on new device → credentials present, keys re-bound to new device`

---

## Phase 12: Compliance, Hardening & Deployment

### Purpose
Make the platform production- and audit-ready: GDPR retention/erasure, data residency, ISO 27001/27701-aligned controls, rate limiting and abuse protection, and reproducible deployment. Required before any regulated (eIDAS) deployment.

### Tasks

#### 12.1 — Retention, erasure & data residency

**What**: Enforce `retention_policies`, support GDPR right-to-erasure, and pin data residency.

**Design**:
- BullMQ `retention.sweep` cron deletes/anonymises resources past `retention_days` per policy.
- Right-to-erasure: PII claims are encrypted per-subject (crypto-shredding); erasure deletes the per-subject key, rendering claims unrecoverable while preserving non-PII audit integrity.
- Data residency: `tenants.data_residency` selects the storage region/bucket; enforced at the storage adapter.

**Testing**:
- `Integration: credential past retention → swept (deleted/anonymised), audit retains non-PII record`
- `Unit: erasure deletes subject key → claims undecryptable`
- `Integration: eu-residency tenant writes only to eu bucket`

#### 12.2 — Security hardening & abuse protection

**What**: Rate limiting, input hardening, secrets handling, and security headers.

**Design**:
- Per-tenant + per-IP rate limits (Redis); stricter on verification/issuance.
- All inputs validated by JSON Schema at the route; reject unknown fields.
- Secrets only via env/KMS; no secrets in logs (pino redaction).
- Webhook signatures (HMAC), OID4VC nonce/replay protection (one-time nonces in Redis).

**Testing**:
- `Integration: replayed OID4VP nonce → rejected`
- `Integration: oversized/extra-field payload → 422`
- `Unit: logs redact Authorization and private key fields`
- `Security: dependency audit (pnpm audit) clean in CI`

#### 12.3 — Deployment & conformance gate

**What**: docker-compose dev stack, production Dockerfiles, and a conformance CI job.

**Design**:
- `docker-compose.yml`: api, postgres, redis, ml-sidecar, admin.
- Multi-stage Dockerfiles per app; healthchecks; non-root users.
- CI conformance job runs the W3C VC test suite + OID4VC HAIP vectors against a booted stack; failures block release.
- `.env.example` documents every config var (DoD item 7).

**Testing**:
- `CI: docker compose up → /ready 200 within 60s`
- `CI: W3C VC test suite passes`
- `CI: OID4VCI + OID4VP HAIP vectors pass`
- `CI: lint + typecheck + unit + integration all green`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Layer        ─── required by everything
    │
Phase 2: Cryptography & KMS             ─── requires P1
    │
Phase 3: DID Issuance & Resolution      ─── requires P1, P2
    │
Phase 4: Credential Schemas & Issuance  ─── requires P3
    │
Phase 5: Revocation (Status List)       ─── requires P4
    │
Phase 6: Verification API & SDK         ─── requires P4, P5
    │
    ├── Phase 7: OID4VC Protocol Layer       ─── requires P6
    │       │
    │       └── Phase 8: Browser Wallet      ─── requires P7   ┐ can parallel
    │       └── Phase 11: Mobile Wallet       ─── requires P8   ┘ (11 after 8's SDK)
    │
    ├── Phase 9: Admin Console & AI features  ─── requires P6 (can parallel with P7)
    │
    └── Phase 10: Non-Human Identity          ─── requires P4, P6 (can parallel with P7/P9)

Phase 12: Compliance & Deployment       ─── requires all functional phases
```

**Parallelism opportunities:**
- After Phase 6: Phase 7 (OID4VC), Phase 9 (Admin + AI), and Phase 10 (Non-Human Identity) can be developed concurrently by separate streams.
- Phase 8 (browser wallet) depends on Phase 7; Phase 11 (mobile) reuses Phase 8's holder SDK and follows it.
- Phase 9.4 (ML sidecar, Python) is independent of the TS work once the `anomaly_signals` table and `fraud.score` queue contract exist (end of Phase 1 + Phase 6 verification events).

**Estimated scope: large** (12 phases, 31 tasks, multiple deployables, regulated-domain conformance).

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (Vitest / pytest).
3. ESLint + Prettier pass with no errors; Python ruff passes for sidecar changes.
4. `tsc --noEmit` (and mypy for the sidecar) passes — no type errors.
5. Testcontainers integration tests pass against real Postgres + Redis.
6. `docker compose up` boots the stack and `/ready` returns 200.
7. New configuration options are added to `.env.example` and documented.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec at `/openapi.json`.
9. New Drizzle migrations are created, are reversible where feasible, and apply cleanly on a fresh database.
10. The feature works end-to-end via its primary interface (REST, SDK, admin UI, or wallet).
11. Standards touched in the phase are cited and, where a test suite exists (W3C VC, OID4VC HAIP, DIF PE), the relevant conformance vectors pass.
12. An `audit_log` entry is emitted for every state-changing operation introduced in the phase.
```
