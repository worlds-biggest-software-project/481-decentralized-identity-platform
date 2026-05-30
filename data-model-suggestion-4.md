# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A property graph database (Neo4j) where every entity is a node and every relationship is a first-class, queryable, typed edge with properties. Decentralized identity is fundamentally a trust graph: issuers trust holders, verifiers trust issuers, DIDs control other DIDs, credentials chain through delegation, and revocation propagates along trust paths. A graph model represents these relationships natively rather than encoding them as foreign keys in flat tables.

## Why This Is the Domain-Specific Best Fit

Decentralized identity is a graph problem masquerading as a CRUD problem. Consider the core questions the platform must answer:

- **Trust chain traversal**: "Is this credential trustworthy?" requires walking from the credential's issuer DID, through any delegated issuance chain, up to a root trust anchor. In SQL this is a recursive CTE; in a graph it is a simple variable-length path traversal.
- **Credential provenance**: "Show me all credentials that were issued under the authority of this root issuer, transitively" -- a multi-hop relationship query that graphs handle in constant time per hop.
- **DID controller hierarchies**: A DID can be controlled by another DID, which is controlled by yet another. Graph traversal expresses this naturally.
- **Fraud detection**: "Find holders who presented the same credential to more than 5 verifiers within one hour" is a pattern-matching query on the presentation subgraph.
- **Interoperability mapping**: Credentials issued under one schema may be recognised as equivalent to another schema through a trust framework mapping -- a relationship best modelled as an edge.

The main cost is that graph databases are less mature for traditional CRUD operations, lack the ACID guarantees of PostgreSQL for complex multi-entity transactions, and require specialised operational expertise. For a production system, this model works best as the primary store for relationship-heavy queries alongside a relational or document store for transactional writes.

---

## Node Definitions

### Core Identity Nodes

```cypher
// ============================================================
// TENANT
// ============================================================
CREATE CONSTRAINT tenant_id_unique IF NOT EXISTS
FOR (t:Tenant) REQUIRE t.tenantId IS UNIQUE;

// Example node:
// (:Tenant {
//     tenantId: "uuid",
//     name: "Acme University",
//     slug: "acme-university",
//     dataResidency: "eu",
//     createdAt: datetime("2026-05-26T10:00:00Z")
// })

// ============================================================
// DID SUBJECT
// ============================================================
CREATE CONSTRAINT subject_id_unique IF NOT EXISTS
FOR (s:Subject) REQUIRE s.subjectId IS UNIQUE;

// (:Subject {
//     subjectId: "uuid",
//     entityType: "person",          // person | organisation | device | ai_agent | service
//     displayName: "Alice Smith",
//     createdAt: datetime(),
//     deactivatedAt: null
// })

// ============================================================
// DID DOCUMENT
// ============================================================
CREATE CONSTRAINT did_unique IF NOT EXISTS
FOR (d:DID) REQUIRE d.did IS UNIQUE;

// (:DID {
//     did: "did:web:university.edu:users:alice",
//     method: "did:web",
//     isActive: true,
//     documentJson: '{ full DID document as JSON string }',
//     alsoKnownAs: ["https://alice.example.com"],
//     createdAt: datetime(),
//     updatedAt: datetime()
// })

// ============================================================
// VERIFICATION METHOD (KEY)
// ============================================================
CREATE CONSTRAINT vm_key_id_unique IF NOT EXISTS
FOR (k:VerificationMethod) REQUIRE k.keyId IS UNIQUE;

// (:VerificationMethod {
//     vmId: "uuid",
//     keyId: "did:web:university.edu:users:alice#key-1",
//     keyType: "Ed25519VerificationKey2020",
//     publicKeyJwk: '{ JWK as JSON string }',
//     purposes: ["authentication", "assertionMethod"],
//     createdAt: datetime(),
//     revokedAt: null
// })
```

### Credential Nodes

```cypher
// ============================================================
// CREDENTIAL SCHEMA
// ============================================================
CREATE CONSTRAINT schema_id_unique IF NOT EXISTS
FOR (cs:CredentialSchema) REQUIRE cs.schemaId IS UNIQUE;

// (:CredentialSchema {
//     schemaId: "uuid",
//     name: "UniversityDegree",
//     version: "1.0",
//     credentialType: ["VerifiableCredential", "UniversityDegree"],
//     contextUrls: ["https://www.w3.org/ns/credentials/v2"],
//     jsonSchema: '{ JSON Schema as string }',
//     isPublished: true,
//     compliance: ["eidas2"],
//     createdAt: datetime()
// })

// ============================================================
// VERIFIABLE CREDENTIAL
// ============================================================
CREATE CONSTRAINT credential_id_unique IF NOT EXISTS
FOR (vc:Credential) REQUIRE vc.credentialId IS UNIQUE;

// (:Credential {
//     credentialId: "uuid",
//     issuanceDate: datetime("2026-05-26T10:00:00Z"),
//     expirationDate: datetime("2031-05-26T10:00:00Z"),
//     status: "active",
//     proofFormat: "json-ld",
//     credentialHash: "sha256:abcdef...",
//     claims: '{ credential subject claims as JSON string }',
//     signedCredential: "eyJ...",
//     statusIndex: 42,
//     createdAt: datetime()
// })

// ============================================================
// VERIFIABLE PRESENTATION
// ============================================================
CREATE CONSTRAINT presentation_id_unique IF NOT EXISTS
FOR (vp:Presentation) REQUIRE vp.presentationId IS UNIQUE;

// (:Presentation {
//     presentationId: "uuid",
//     proofFormat: "sd-jwt",
//     isVerified: true,
//     verifiedAt: datetime(),
//     verificationChecks: '{ checks as JSON string }',
//     createdAt: datetime()
// })

// ============================================================
// PRESENTATION REQUEST
// ============================================================
CREATE CONSTRAINT request_id_unique IF NOT EXISTS
FOR (pr:PresentationRequest) REQUIRE pr.requestId IS UNIQUE;

// (:PresentationRequest {
//     requestId: "uuid",
//     nonce: "random-nonce",
//     requestSpec: '{ requested schemas, attributes, predicates as JSON }',
//     expiresAt: datetime(),
//     createdAt: datetime()
// })
```

### Infrastructure Nodes

```cypher
// ============================================================
// WALLET
// ============================================================
CREATE CONSTRAINT wallet_id_unique IF NOT EXISTS
FOR (w:Wallet) REQUIRE w.walletId IS UNIQUE;

// (:Wallet {
//     walletId: "uuid",
//     walletType: "mobile_ios",
//     biometricBound: true,
//     backupEnabled: true,
//     deviceInfo: '{ device metadata as JSON }',
//     createdAt: datetime(),
//     lastActiveAt: datetime()
// })

// ============================================================
// DEVICE / AI AGENT
// ============================================================
CREATE CONSTRAINT device_id_unique IF NOT EXISTS
FOR (dev:Device) REQUIRE dev.deviceId IS UNIQUE;

// (:Device {
//     deviceId: "uuid",
//     deviceType: "ai_agent",
//     attestationType: "mcp-i",
//     deviceMetadata: '{ manufacturer, model, firmware, MCP-I profile as JSON }',
//     createdAt: datetime(),
//     lastHeartbeat: datetime()
// })

// ============================================================
// REVOCATION REGISTRY
// ============================================================
CREATE CONSTRAINT registry_id_unique IF NOT EXISTS
FOR (rr:RevocationRegistry) REQUIRE rr.registryId IS UNIQUE;

// (:RevocationRegistry {
//     registryId: "uuid",
//     registryType: "StatusList2021",
//     statusListUri: "https://issuer.example.com/status/1",
//     maxCredentials: 131072,
//     createdAt: datetime()
// })

// ============================================================
// OIDC CLIENT
// ============================================================
CREATE CONSTRAINT oidc_client_id_unique IF NOT EXISTS
FOR (oc:OIDCClient) REQUIRE oc.clientId IS UNIQUE;

// (:OIDCClient {
//     clientId: "uuid",
//     clientSecretHash: "bcrypt:...",
//     redirectUris: ["https://app.example.com/callback"],
//     oidcConfig: '{ scopes, token config as JSON }',
//     createdAt: datetime()
// })

// ============================================================
// ANOMALY SIGNAL
// ============================================================
CREATE CONSTRAINT signal_id_unique IF NOT EXISTS
FOR (a:AnomalySignal) REQUIRE a.signalId IS UNIQUE;

// (:AnomalySignal {
//     signalId: "uuid",
//     signalType: "unusual_frequency",
//     severity: "high",
//     signalData: '{ ML model output as JSON }',
//     resolved: false,
//     createdAt: datetime()
// })
```

---

## Relationship Definitions

This is where the graph model truly shines. Every relationship is a typed, directed edge with properties.

```cypher
// ============================================================
// IDENTITY RELATIONSHIPS
// ============================================================

// Tenant -> Subject membership
// (:Subject)-[:BELONGS_TO]->(:Tenant)

// Subject -> DID ownership
// (:DID)-[:IDENTIFIES]->(:Subject)

// DID -> DID controller hierarchy
// (:DID)-[:CONTROLLED_BY { since: datetime() }]->(:DID)

// DID -> Verification Method
// (:VerificationMethod)-[:BELONGS_TO_DID]->(:DID)

// Subject -> Wallet
// (:Wallet)-[:OWNED_BY]->(:Subject)

// Subject -> Device/Agent
// (:Device)-[:REGISTERED_TO]->(:Subject)

// ============================================================
// CREDENTIAL LIFECYCLE RELATIONSHIPS
// ============================================================

// Schema belongs to tenant
// (:CredentialSchema)-[:DEFINED_BY]->(:Tenant)

// Credential conforms to schema
// (:Credential)-[:CONFORMS_TO]->(:CredentialSchema)

// Issuer issued credential
// (:Credential)-[:ISSUED_BY]->(:DID)

// Holder holds credential
// (:Credential)-[:HELD_BY]->(:DID)

// Credential signed with specific key
// (:Credential)-[:SIGNED_WITH]->(:VerificationMethod)

// Credential registered in revocation registry
// (:Credential)-[:REGISTERED_IN { statusIndex: 42 }]->(:RevocationRegistry)

// Registry managed by issuer
// (:RevocationRegistry)-[:MANAGED_BY]->(:DID)

// Wallet stores credential
// (:Wallet)-[:STORES { addedAt: datetime(), isFavourite: true }]->(:Credential)

// ============================================================
// PRESENTATION AND VERIFICATION RELATIONSHIPS
// ============================================================

// Verifier created presentation request
// (:PresentationRequest)-[:REQUESTED_BY]->(:DID)

// Presentation fulfills request
// (:Presentation)-[:FULFILLS]->(:PresentationRequest)

// Holder created presentation
// (:Presentation)-[:PRESENTED_BY]->(:DID)

// Presentation sent to verifier
// (:Presentation)-[:PRESENTED_TO]->(:DID)

// Presentation includes credential (with selective disclosure info)
// (:Presentation)-[:INCLUDES {
//     disclosedAttributes: ["degree", "field"],
//     zkProofs: '{ predicate results }'
// }]->(:Credential)

// ============================================================
// TRUST AND DELEGATION RELATIONSHIPS
// ============================================================

// Trust framework: issuer trusted by trust anchor
// (:DID)-[:TRUSTED_BY {
//     trustLevel: "qualified",
//     framework: "eidas2",
//     since: datetime(),
//     until: datetime()
// }]->(:DID)

// Delegated issuance: issuer delegates to sub-issuer
// (:DID)-[:DELEGATES_ISSUANCE {
//     schemaIds: ["uuid"],
//     maxDepth: 2,
//     since: datetime(),
//     until: datetime()
// }]->(:DID)

// Schema equivalence across trust frameworks
// (:CredentialSchema)-[:EQUIVALENT_TO {
//     mappingRule: "1:1",
//     framework: "eidas2"
// }]->(:CredentialSchema)

// ============================================================
// OIDC BRIDGE RELATIONSHIPS
// ============================================================

// OIDC client belongs to tenant
// (:OIDCClient)-[:REGISTERED_WITH]->(:Tenant)

// OIDC client accepts schemas
// (:OIDCClient)-[:ACCEPTS]->(:CredentialSchema)

// ============================================================
// ANOMALY DETECTION RELATIONSHIPS
// ============================================================

// Anomaly signal raised for a subject's DID
// (:AnomalySignal)-[:FLAGGED_ON]->(:DID)

// Anomaly signal related to a presentation
// (:AnomalySignal)-[:TRIGGERED_BY]->(:Presentation)
```

---

## Key Graph Queries

```cypher
// 1. Trust chain verification: Is this credential's issuer trusted
//    (directly or transitively) by a known trust anchor?
MATCH path = (vc:Credential {credentialId: $credId})
  -[:ISSUED_BY]->(issuer:DID)
  -[:TRUSTED_BY*1..5]->(anchor:DID {did: $trustAnchorDid})
WHERE ALL(r IN relationships(path) WHERE
    r.since <= datetime() AND (r.until IS NULL OR r.until > datetime())
)
RETURN path, length(path) AS chainLength;

// 2. Find all credentials transitively issued under a root issuer's authority
MATCH (root:DID {did: $rootDid})
  <-[:DELEGATES_ISSUANCE*0..3]-(delegatee:DID)
  <-[:ISSUED_BY]-(vc:Credential)
WHERE vc.status = 'active'
RETURN delegatee.did AS issuer, collect(vc.credentialId) AS credentials;

// 3. Fraud detection: holders who presented the same credential
//    to more than N distinct verifiers in the last hour
MATCH (vc:Credential)<-[:INCLUDES]-(vp:Presentation)-[:PRESENTED_TO]->(verifier:DID)
WHERE vp.createdAt > datetime() - duration('PT1H')
WITH vc, count(DISTINCT verifier) AS verifierCount
WHERE verifierCount > 5
RETURN vc.credentialId, verifierCount
ORDER BY verifierCount DESC;

// 4. Credential provenance: full lifecycle of a single credential
MATCH (vc:Credential {credentialId: $credId})
OPTIONAL MATCH (vc)-[:ISSUED_BY]->(issuer:DID)
OPTIONAL MATCH (vc)-[:HELD_BY]->(holder:DID)
OPTIONAL MATCH (vc)-[:CONFORMS_TO]->(schema:CredentialSchema)
OPTIONAL MATCH (vc)-[:SIGNED_WITH]->(key:VerificationMethod)
OPTIONAL MATCH (vc)-[:REGISTERED_IN]->(reg:RevocationRegistry)
OPTIONAL MATCH (vc)<-[inc:INCLUDES]-(vp:Presentation)-[:PRESENTED_TO]->(verifier:DID)
RETURN vc, issuer, holder, schema, key, reg,
       collect({
           presentationId: vp.presentationId,
           verifier: verifier.did,
           disclosed: inc.disclosedAttributes,
           verifiedAt: vp.verifiedAt
       }) AS presentations;

// 5. DID controller hierarchy
MATCH path = (leaf:DID {did: $did})-[:CONTROLLED_BY*1..10]->(root:DID)
WHERE NOT (root)-[:CONTROLLED_BY]->()
RETURN [n IN nodes(path) | n.did] AS controllerChain;

// 6. Schema equivalence: find all schemas that are equivalent to a given one
MATCH (s:CredentialSchema {schemaId: $schemaId})
  -[:EQUIVALENT_TO*1..3]-(equiv:CredentialSchema)
RETURN equiv.schemaId, equiv.name, equiv.version;

// 7. Non-human identity: find all AI agents with active credentials
MATCH (dev:Device {deviceType: "ai_agent"})
  -[:REGISTERED_TO]->(subj:Subject)
  <-[:IDENTIFIES]-(did:DID)
  <-[:HELD_BY]-(vc:Credential {status: "active"})
RETURN dev.deviceId, did.did, collect(vc.credentialId) AS activeCredentials;
```

---

## Full Schema Setup Script

```cypher
// ============================================================
// INDEXES (beyond uniqueness constraints above)
// ============================================================

CREATE INDEX subject_entity_type IF NOT EXISTS FOR (s:Subject) ON (s.entityType);
CREATE INDEX did_method IF NOT EXISTS FOR (d:DID) ON (d.method);
CREATE INDEX did_active IF NOT EXISTS FOR (d:DID) ON (d.isActive);
CREATE INDEX credential_status IF NOT EXISTS FOR (vc:Credential) ON (vc.status);
CREATE INDEX credential_issuance IF NOT EXISTS FOR (vc:Credential) ON (vc.issuanceDate);
CREATE INDEX presentation_created IF NOT EXISTS FOR (vp:Presentation) ON (vp.createdAt);
CREATE INDEX device_type IF NOT EXISTS FOR (dev:Device) ON (dev.deviceType);
CREATE INDEX anomaly_resolved IF NOT EXISTS FOR (a:AnomalySignal) ON (a.resolved);
CREATE INDEX anomaly_type IF NOT EXISTS FOR (a:AnomalySignal) ON (a.signalType);

// Full-text indexes for search
CREATE FULLTEXT INDEX subject_name IF NOT EXISTS
FOR (s:Subject) ON EACH [s.displayName];

CREATE FULLTEXT INDEX schema_name IF NOT EXISTS
FOR (cs:CredentialSchema) ON EACH [cs.name];
```

---

## Trade-offs

**Strengths:**
- Relationship traversal is O(1) per hop regardless of dataset size, making trust chain verification, delegation queries, and fraud pattern detection extremely fast.
- The schema is self-documenting: reading the relationship types tells you exactly how the system works.
- Adding new relationship types (e.g., a new trust framework mapping) requires no schema migration -- just create edges with a new type.
- Pattern matching (Cypher `MATCH`) naturally expresses identity domain queries that would require multiple JOINs or recursive CTEs in SQL.
- Graph visualisation tools (Neo4j Browser, Bloom) provide immediate insight into trust topologies for compliance teams.

**Weaknesses:**
- Neo4j's ACID transactions are single-database; cross-database transactions require application-level coordination.
- Aggregate queries (COUNT, SUM, GROUP BY) are slower than in relational databases. Dashboard statistics should be materialised separately.
- JSONB-equivalent flexible property storage exists but lacks PostgreSQL's GIN index sophistication.
- Operational maturity is lower than PostgreSQL: fewer hosting options, smaller talent pool, less mature backup/restore tooling.
- Write throughput for bulk credential issuance is lower than PostgreSQL. Batch imports should use the Neo4j Admin Import tool.

## Scalability Considerations

- Neo4j Aura (managed cloud) or Neo4j Cluster for horizontal read scaling.
- Shard by tenant using Neo4j Fabric for multi-database federation.
- Use the APOC library for bulk operations and periodic data cleanup.
- For the highest write throughput, consider a polyglot architecture: PostgreSQL for transactional writes, Neo4j for relationship queries, synchronised via CDC (Change Data Capture).

## Migration Path

- A hybrid deployment is recommended: use PostgreSQL (Suggestion 1 or 3) as the transactional system of record and synchronise to Neo4j for trust graph queries, fraud detection, and compliance analytics.
- CDC from PostgreSQL (using Debezium) can feed a Neo4j Sink Connector, keeping the graph projection near-real-time.
- Start with Neo4j for the trust chain and delegation subgraph only, then expand to the full credential lifecycle as the team gains graph database expertise.
- If the team later decides to consolidate on a single database, the graph queries can be translated to PostgreSQL recursive CTEs and `ltree` extensions, though with reduced performance for deep traversals.
