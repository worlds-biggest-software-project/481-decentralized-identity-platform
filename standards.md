# Standards & API Reference

> Project: Decentralized Identity Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 18013-5:2021 — Mobile Driving Licence (mDL) Application**
- **URL:** https://www.iso.org/standard/69084.html
- Defines the data model, encoding, and presentation protocols for driver's licences stored on mobile devices. Specifies NFC/QR offline verification and online remote authentication, plus selective disclosure of attributes. A draft revision (DIS 18013-5) is in progress. Mainstream adoption underway in the US, Australia, and EU; the leading near-term use case for consumer-facing decentralised credentials.

**ISO/IEC 27701:2025 — Privacy Information Management System (PIMS)**
- **URL:** https://www.iso.org/standard/27701
- Standalone (from the 2025 edition) privacy management framework building on ISO 27001/27002. Specifies requirements for processing personally identifiable information (PII) as both data controller and data processor. Directly relevant to credential issuance, holder data minimisation, and GDPR alignment. The 2025 edition adds guidance on AI-related processing, biometrics, IoT, and cloud services.

**ISO/IEC 27001:2022 — Information Security Management System (ISMS)**
- **URL:** https://www.iso.org/standard/27001
- The foundational international standard for information security management. Applies to the platform's key management infrastructure, issuer portals, and data residency controls. Commonly required by enterprise issuers as a prerequisite for deploying identity services.

---

### W3C Standards

**W3C Verifiable Credentials Data Model v2.0 (Recommendation, 15 May 2025)**
- **URL:** https://www.w3.org/TR/vc-data-model-2.0/
- The canonical data model for expressing cryptographically verifiable credentials on the web. Defines the issuer–holder–verifier triangle, credential and presentation structures, and extension points for credential status. Published as a full W3C Recommendation in May 2025; v2.1 is planned for 2027. Compliance is table-stakes for any production decentralised identity system.

**W3C Decentralized Identifiers (DIDs) v1.1 (Candidate Recommendation, March 2026)**
- **URL:** https://www.w3.org/TR/did-1.1/
- Specifies DID syntax, the common DID Document data model, core properties, serialised representations, and DID operations. The v1.1 Candidate Recommendation was published March 2026; v1.0 remains the published W3C Recommendation (2022). The platform must implement at least `did:web` and one ledger-anchored method to this specification.

**W3C DID Resolution — DID URL Dereferencing**
- **URL:** https://w3c-ccg.github.io/did-resolution/
- W3C CCG specification defining the resolution algorithm for translating a DID to its DID Document. The DIF Universal Resolver implements this specification. Required reading for anyone building a resolution gateway that supports multiple DID methods.

**W3C JSON-LD 1.1**
- **URL:** https://www.w3.org/TR/json-ld11/
- W3C Recommendation (2020) for serialising Linked Data in JSON. The primary data format for W3C Verifiable Credentials using JSON-LD/Linked Data Proofs. Required for semantic interoperability between different credential ecosystems and schema registries.

**W3C Data Integrity — BBS Cryptosuite v1.0**
- **URL:** https://www.w3.org/TR/vc-di-bbs/
- Defines the `bbs-2023` cryptographic suite for verifiable credentials, enabling BBS+ signature-based selective disclosure. The holder uses the issuer's BBS signature to derive a proof over a selected subset of claims without revealing others. The primary specification for privacy-preserving ZK-style selective disclosure in W3C-format credentials.

**W3C Data Integrity — ECDSA Cryptosuites v1.0**
- **URL:** https://www.w3.org/TR/vc-di-ecdsa/
- Defines `ecdsa-rdfc-2019` and `ecdsa-sd-2023` suites. The `ecdsa-sd-2023` suite supports selective disclosure of individual claims by signing each non-mandatory attribute separately. Relevant for implementations requiring ECDSA-based proofs with selective disclosure without BBS+.

**W3C Securing Verifiable Credentials using JOSE and COSE**
- **URL:** https://www.w3.org/TR/vc-jose-cose/
- Defines how to secure VCs using JWT (JSON Web Token), SD-JWT, and COSE encodings, complementing the JSON-LD/Data Integrity approach. Required for interoperability with the OID4VC family, which predominantly uses JWT and SD-JWT credential formats.

**W3C Bitstring Status List v1.0 (formerly Status List 2021)**
- **URL:** https://www.w3.org/TR/vc-bitstring-status-list/
- W3C Recommendation for privacy-preserving credential revocation and suspension. Credentials embed a pointer to a bitstring; the issuer flips a bit to revoke without identifying which holder is affected. This is the mandated revocation mechanism for the EUDI Wallet ecosystem and the recommended approach for any production deployment.

**W3C WebAuthn Level 2 (Recommendation, 2021)**
- **URL:** https://www.w3.org/TR/webauthn-2/
- Defines the browser API for accessing public-key credentials (passkeys, FIDO2 security keys). Relevant to the platform's wallet authentication layer, device binding, and hardware-backed key attestation. Level 3 is in First Public Working Draft.

---

### IETF Standards & RFCs

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749
- Foundational authorization framework underlying OpenID Connect and the OID4VC family. Required for the OIDC bridge that allows VC presentation as an OAuth/OIDC authentication flow.

**OpenID Connect Core 1.0**
- **URL:** https://openid.net/specs/openid-connect-core-1_0.html
- The identity layer on top of OAuth 2.0. Defines ID tokens, UserInfo endpoint, and authentication flows. The platform's OIDC bridge maps VC presentations onto OIDC authentication responses for backwards-compatible integration with existing access management systems.

**RFC 9901 — Selective Disclosure for JSON Web Tokens (SD-JWT)**
- **URL:** https://datatracker.ietf.org/doc/rfc9901/
- Published November 2025. Defines the SD-JWT format, enabling selective disclosure of individual JWT claims via disclosure objects separated by tilde `~` characters. The foundation for the SD-JWT VC credential format adopted by OID4VC, EUDI, and over 30 national digital identity programmes.

**IETF Draft: SD-JWT-based Verifiable Credentials (SD-JWT VC)**
- **URL:** https://datatracker.ietf.org/doc/draft-ietf-oauth-sd-jwt-vc/
- Internet-Draft (latest revision at time of writing: draft-16) defining SD-JWT-format Verifiable Credentials with `dc+sd-jwt` media type. Extends RFC 9901 with credential-specific claims, type metadata, status handling, and issuer key resolution. Expected to reach RFC status in 2026.

**IETF Draft: Key Event Receipt Infrastructure (KERI)**
- **URL:** https://datatracker.ietf.org/doc/html/draft-ssmith-keri-00
- Defines Autonomic Identifiers (AIDs) and a self-certifying key event log providing cryptographic root-of-trust without requiring a blockchain or central registry. Relevant as an alternative DID infrastructure for high-assurance or air-gapped deployments, and for IoT/device identity where ledger connectivity is unavailable.

---

### OpenID Foundation Specifications

**OpenID for Verifiable Credential Issuance (OID4VCI) v1.0**
- **URL:** https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html
- Finalised September 2025. Defines an OAuth 2.0-protected API for issuing Verifiable Credentials to wallets. Over 30 national digital identity programmes have deployed OID4VCI. The platform's credential issuance API must support this specification for wallet interoperability.

**OpenID for Verifiable Presentations (OID4VP) v1.0**
- **URL:** https://openid.net/specs/openid-4-verifiable-presentations-1_0.html
- Defines how verifiers request and receive credential presentations from wallets, built on top of OAuth 2.0. Works in concert with DIF Presentation Exchange for specifying the exact credential requirements. Mandatory for EUDI Wallet compatibility.

**Self-Issued OpenID Provider v2 (SIOPv2)**
- **URL:** https://openid.net/specs/openid-connect-self-issued-v2-1_0.html
- Extends OpenID Connect to allow users to act as their own identity provider using a wallet-controlled key pair. Enables holder-initiated authentication without a centralised identity provider. Required for decentralised wallet-to-verifier flows without issuer involvement.

**OpenID4VC High Assurance Interoperability Profile (HAIP) v1.0**
- **URL:** https://openid.net/specs/openid4vc-high-assurance-interoperability-profile-1_0-final.html
- Defines a constrained, interoperable profile of OID4VCI, OID4VP, and SIOPv2 for high-assurance government and regulated-sector deployments. Mandated by the EU EUDI Wallet ecosystem. Specifies required credential formats (SD-JWT VC), key types, and binding mechanisms.

---

### Decentralized Identity Foundation (DIF) Specifications

**DIF DIDComm Messaging v2 (Approved Specification)**
- **URL:** https://identity.foundation/didcomm-messaging/spec/
- Defines message-based, asynchronous, peer-to-peer communication between DID-authenticated parties. The standard protocol for agent-to-agent credential exchange workflows. Underpins Hyperledger Aries protocols and is used in enterprise and government deployments globally.

**DIF Presentation Exchange v2.0**
- **URL:** https://identity.foundation/presentation-exchange/spec/v2.0.0/
- Defines Presentation Definitions (verifier-side credential requirements) and Presentation Submissions (holder-side responses). Format- and transport-agnostic; works with JWT VCs, SD-JWT VCs, AnonCreds, and JSON-LD credentials. Used alongside OID4VP to specify exactly which credential attributes a verifier requires.

**DIF Universal Resolver**
- **URL:** https://dev.uniresolver.io/ · GitHub: https://github.com/decentralized-identity/universal-resolver
- Open-source HTTP API gateway that resolves DIDs across 45+ DID methods using a driver plugin architecture. The platform should integrate the Universal Resolver (or a subset of its drivers) to provide a ledger-agnostic DID resolution service to verifiers.

---

### EU Regulatory Frameworks

**EU eIDAS 2.0 Regulation — EUDI Wallet Architecture and Reference Framework (ARF) v2.4**
- **URL:** https://eu-digital-identity-wallet.github.io/eudi-doc-architecture-and-reference-framework/2.4.0/
- The European Digital Identity Regulation requires EU member states to provide EUDI Wallets to citizens by end of 2026, with relying parties required to accept them from 2027. The ARF defines the technical architecture, required protocols (OID4VCI, OID4VP, SD-JWT VC, ISO 18013-5), and conformance requirements. Any deployment targeting European markets must align with the ARF.

---

### AnonCreds Specification

**Hyperledger AnonCreds Specification**
- **URL:** https://anoncreds.github.io/anoncreds-spec/ (also: https://hyperledger.github.io/anoncreds-spec/)
- Defines the CL-signature-based verifiable credential format with built-in zero-knowledge proof support, predicate proofs (e.g., `age > 18` without revealing date of birth), and unlinkable presentations. De facto standard for ZKP-based credentials; deployed in national identity programmes in Canada, Australia, and the EU. v2 replaces CL signatures with BBS+. Apache 2.0 licence.

---

### NIST Guidance

**NIST SP 800-63-4 — Digital Identity Guidelines (Revision 4, July 2025)**
- **URL:** https://pages.nist.gov/800-63-4/
- Finalised July 2025. Defines identity assurance levels (IAL), authenticator assurance levels (AAL), and federation assurance levels (FAL). Incorporates support for digital wallets, passkeys, and VC-based identity proofing. Relevant for US-market deployments, enterprise procurement requirements, and defining the assurance levels the platform's credentials can satisfy.

---

### Trust over IP Foundation

**ToIP Technology Architecture Specification**
- **URL:** https://trustoverip.github.io/TechArch/
- Defines a four-layer stack for Internet-scale digital trust, combining cryptographic assurance (technology layers 1–2) with human accountability (governance layers 3–4). Provides the governance framework model that credential ecosystems need alongside technical specifications. Relevant for structuring multi-party trust networks around the platform.

---

## Similar Products — Developer Documentation & APIs

### Microsoft Entra Verified ID

- **Description:** Enterprise SaaS decentralised identity platform built on Azure, supporting W3C VC Data Model 2.0, DID:web and DID:ion. Deepest enterprise integration with Azure AD and Microsoft 365.
- **API Documentation:** https://learn.microsoft.com/en-us/entra/verified-id/
- **Issuance Request API:** https://learn.microsoft.com/en-us/entra/verified-id/issuance-request-api
- **Verification Request API:** https://learn.microsoft.com/en-us/entra/verified-id/presentation-request-api
- **Admin API:** https://learn.microsoft.com/en-us/entra/verified-id/admin-api
- **SDKs/Libraries:** Node.js, .NET, Python, Java samples — https://github.com/Azure-Samples/active-directory-verifiable-credentials
- **Standards:** W3C VC DM 2.0, DID:web, DID:ion, OID4VC, Bitstring Status List
- **Authentication:** OAuth 2.0 bearer tokens via Azure AD (scope: `6a8b4b39-c021-437c-b060-5a14a3fd65f3/full_access`)

---

### Dock.io / Truvera

- **Description:** Full-stack decentralised identity platform with own L1 blockchain (Dock Chain), REST API and mobile SDK. Supports W3C VC, AnonCreds, and did:dock / did:web / did:key. Includes a no-code credential design portal (Dock Certs).
- **API Documentation:** https://docs.api.dock.io/ · https://docs.truvera.io/
- **Getting Started:** https://docs.truvera.io/developer-documentation/dock-api/getting-started
- **SDKs/Libraries:** JavaScript/TypeScript SDK — https://github.com/docknetwork/sdk; REST API wrapper available
- **Standards:** W3C VC DM 2.0, AnonCreds, OID4VC (in progress), Bitstring Status List
- **Authentication:** API keys (obtained via Dock Certs console)

---

### Trinsic

- **Description:** Developer-focused SaaS identity infrastructure positioning itself as "Stripe for identity." Provides REST/RPC API, whitelabel wallet SDK, and verification policy engine. Supports JSON-LD credentials with BBS+ signatures.
- **API Documentation:** https://docs.trinsic.id · https://v2-docs.trinsic.id/
- **SDKs/Libraries:** TypeScript, Python, Go, Java, Ruby — multi-language SDK suite
- **Standards:** W3C VC DM, JSON-LD, BBS+ signatures, DIDComm v2
- **Authentication:** API key (dashboard-issued)

---

### Hyperledger Aries Cloud Agent Python (ACA-Py)

- **Description:** Open-source, production-grade framework for building decentralised identity agents (issuer, holder, verifier) in non-mobile environments. Used by national identity programmes globally. Controller pattern: your application communicates with ACA-Py via REST API and webhook callbacks.
- **API Documentation:** https://aca-py.org · https://aries-cloud-agent-python.readthedocs.io/
- **OpenAPI Demo:** https://aca-py.org/0.12.2/demo/AriesOpenAPIDemo/
- **GitHub:** https://github.com/hyperledger/aries-cloudagent-python (now under OpenWallet Foundation)
- **SDKs/Libraries:** Python (native); controller SDKs exist in JavaScript and others
- **Standards:** W3C VC DM 2.0, AnonCreds, DIDComm v2, Aries RFCs, OpenAPI 3.0 admin interface
- **Authentication:** Admin API secured via API key or mTLS; webhook callbacks to controller

---

### walt.id

- **Description:** Open-source, all-in-one identity and wallet infrastructure toolkit adopted by 38K+ developers and organisations. Provides issuer API, verifier API, and wallet API as composable services. Aligned with W3C, ISO, OIDF, and IETF standards.
- **API Documentation:** https://docs.walt.id/
- **Issuer API Getting Started:** https://docs.walt.id/community-stack/issuer/api/getting-started
- **GitHub:** https://github.com/walt-id/waltid-identity
- **SDKs/Libraries:** Kotlin/JVM core; REST API accessible from any language; TypeScript bindings available
- **Standards:** W3C VC DM 2.0, SD-JWT VC, ISO 18013-5 (mDoc), OID4VCI, OID4VP, SIOPv2, HAIP, GDPR/eIDAS2
- **Authentication:** API key or configurable auth middleware
- **Licence:** Apache 2.0

---

### Sphereon SSI SDK

- **Description:** Open-source TypeScript/JavaScript SDK providing modular components for W3C VCs, DIDs, OID4VC, SIOPv2, OID4VP, and Presentation Exchange. Higher-level than raw VC libraries; offers a full agent solution with plugin architecture.
- **API Documentation:** https://ssisdk.docs.sphereon.com/ · https://docs.sphereon.com/w3c/vc-api/api
- **GitHub:** https://github.com/Sphereon-Opensource/SSI-SDK
- **SDKs/Libraries:** TypeScript/JavaScript (npm packages); SIOP-OID4VP library: https://github.com/Sphereon-Opensource/SIOP-OID4VP
- **Standards:** W3C VC DM 2.0, JSON-LD, JWT, SD-JWT, OID4VCI, OID4VP, SIOPv2, DIF PE v2
- **Authentication:** Configurable; supports OAuth 2.0 and API key
- **Licence:** Apache 2.0

---

### Veramo

- **Description:** Modular TypeScript/JavaScript framework for verifiable data and SSI. Runs on Node, browsers, and React Native with a consistent API. Core + plugin architecture; supports DID management, VC issuance/verification, and credential storage.
- **API Documentation:** https://veramo.io/docs/api/
- **Getting Started:** https://veramo.io/docs/node_tutorials/node_setup_identifiers/
- **GitHub:** https://github.com/decentralized-identity/veramo
- **SDKs/Libraries:** TypeScript npm packages (`@veramo/core`, `@veramo/credential-w3c`, `@veramo/did-manager`, etc.)
- **Standards:** W3C VC DM 2.0, JSON-LD, JWT, multiple DID methods, DIDComm
- **Authentication:** Controlled by plugins; REST server plugin adds HTTP API with configurable auth
- **Licence:** Apache 2.0

---

### Extrimian

- **Description:** Enterprise decentralised identity platform providing REST API and npm SDK for DID management, VC issuance, and secure messaging via WACI-DIDComm. Targets Latin American and global regulated markets with an eIDAS-aligned stack.
- **API Documentation:** https://docs.extrimian.com/ · http://docs.extrimian.com/en/docs/ApiExtrimian/extrimian-ssi-service-api/
- **SDKs/Libraries:** npm packages: `@extrimian/did-registry`, `@extrimian/did-core`, `@extrimian/kms-client`, `@extrimian/kms-core`
- **GitHub:** https://github.com/extrimian
- **Standards:** W3C VC DM, DIDs, DIDComm v2, WACI-DIDComm
- **Authentication:** API key (via idconnect.extrimian.com portal)

---

## Notes

**Evolving areas to monitor:**

- **SD-JWT VC RFC publication:** The IETF draft `draft-ietf-oauth-sd-jwt-vc` is expected to reach RFC status in 2026. Implementations should track the final media type (`dc+sd-jwt`) as it replaced `vc+sd-jwt` in late 2024.

- **DID Core v1.1 finalisation:** Currently a Candidate Recommendation (March 2026). The transition from v1.0 to v1.1 introduces minor but implementer-relevant changes to the DID Document data model; monitor the W3C DID Working Group for the final Recommendation.

- **MCP-I (Model Context Protocol Identity):** Emerging standard for issuing and verifying DIDs for autonomous AI agents. Not yet formalised at the time of writing (May 2026); monitor Anthropic's Model Context Protocol repository and the DIF's AI Agent identity working items.

- **AnonCreds v2:** Replaces CL signatures with BBS+ and introduces a more scalable revocation scheme. The v2 specification is in active development at the Hyperledger AnonCreds project.

- **EUDI Wallet conformance requirements:** CIR 2025/849 formalises the list of certified EUDI Wallets; conformance testing infrastructure and ARF v2.4+ specifications are still evolving ahead of the 2027 mandatory acceptance deadline.
