# Decentralized Identity Platform — Feature & Functionality Survey

> Candidate #481 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Microsoft Entra Verified ID | SaaS | Commercial (Azure) | https://learn.microsoft.com/en-us/entra/verified-id/ |
| Dock.io | SaaS / open-source components | Commercial | https://www.dock.io |
| Trinsic | SaaS | Commercial | https://trinsic.id |
| Extrimian | SaaS | Commercial | https://extrimian.io |
| Hyperledger Aries / AnonCreds | Open source | Apache 2.0 | https://www.hyperledger.org/projects/aries |

## Feature Analysis by Solution

### Microsoft Entra Verified ID

**Core features**
- Verifiable Credential issuance and verification built on the W3C VC Data Model 2.0 and the DIF (Decentralised Identity Foundation) standards stack
- DID:web and DID:ion method support anchored to the Bitcoin sidechain (ION) or DNS for domain-based DIDs
- Credential issuance portal for organisations to configure claim mapping from existing identity sources (Azure AD, HR systems) to VC schemas
- Verification API: REST API callable from any application to verify credential authenticity without contacting the issuer directly
- Microsoft Authenticator as the default credential wallet for holders on iOS and Android

**Differentiating features**
- Deepest enterprise integration: ties directly into Azure AD, Microsoft 365 identity, and existing organisational directories without a separate identity store
- High adoption among enterprise issuers: used by universities for graduate credentials and by banks for customer identity
- AI-powered fraud detection on presentation requests to identify suspicious verification patterns

**UX patterns**
- Issuer portal: configure credential schemas, map claims from Azure AD, and generate issuance invitation links or QR codes
- Verifier integration via REST API with minimal code — issue a verification request and receive a webhook with the verified result
- End user experience via Microsoft Authenticator app: receive credential, view details, and selectively present to verifiers

**Integration points**
- Azure AD B2C for consumer identity use cases
- Microsoft Power Platform for low-code issuance workflow automation
- REST API for integration with any application platform

**Known gaps**
- Microsoft ecosystem dependency: the credential wallet is Microsoft Authenticator; third-party wallet compatibility is improving but not seamless
- DID:ion requires ION network connectivity; less suitable for air-gapped or highly sovereign deployments
- Limited zero-knowledge proof (ZKP) support for selective disclosure without revealing underlying data fields

**Licence / IP notes**
- Proprietary SaaS. W3C VC Data Model and DIF standards are open and patent-unencumbered. Azure pricing per verified credential transaction.

---

### Dock.io

**Core features**
- Full-stack decentralised identity platform: DID issuance and resolution, VC issuance, credential wallet, and verification API
- Multi-method DID support: did:dock (Dock blockchain), did:web, and did:key
- W3C VC Data Model 2.0 and AnonCreds (zero-knowledge capable) credential formats
- Biometric-linked credentials: credentials tied to biometric verification for high-assurance use cases
- Dock Certs: no-code credential issuance tool for organisations without developer resources

**Differentiating features**
- Own L1 blockchain (Dock Chain) for DID anchoring — provides performance and fee predictability not available on general-purpose chains
- AnonCreds support enables true selective disclosure: prove "over 18" without revealing date of birth
- Comprehensive developer documentation and SDK support for quick integration

**UX patterns**
- No-code Certs portal for organisations to design and issue credentials without API integration
- Developer SDK and REST API for programmatic issuance and verification
- Holder wallet: Dock Wallet mobile app for iOS and Android storing and presenting credentials

**Integration points**
- REST API for issuance and verification integration with any platform
- Webhooks for asynchronous verification result delivery
- OIDC bridge for compatibility with existing OAuth 2.0 / OpenID Connect authentication flows

**Known gaps**
- Own blockchain dependency: Dock Chain adoption is less widespread than Ethereum or Hyperledger-based alternatives
- Wallet interoperability with non-Dock wallets (Microsoft Authenticator, other third-party wallets) requires OID4VC compatibility work
- EU eIDAS 2.0 compliance workflow is still maturing as EUDI Wallet standards finalise

**Licence / IP notes**
- Dock SDK and some tools: Apache 2.0. Platform backend: proprietary. Dock Chain: public blockchain with native token.

---

### Hyperledger Aries / AnonCreds

**Core features**
- Open-source framework for building decentralised identity agents, credential issuers, holders, and verifiers
- AnonCreds: W3C-compatible credential format supporting zero-knowledge proofs, predicate proofs (e.g., age over 18 without revealing DOB), and revocation without linkability
- Hyperledger Indy integration for DID anchoring and revocation registry management on a permissioned ledger
- Aries Cloud Agent Python (ACA-Py): production-grade open-source agent framework used by national identity projects in Canada, the EU, and Australia

**Differentiating features**
- Most mature open-source identity agent codebase; deployed in national-scale government identity programmes
- AnonCreds is the most privacy-preserving credential format available, enabling ZK predicate proofs without revealing underlying data
- Active community with strong governance under Linux Foundation stewardship

**UX patterns**
- No built-in end-user UI; a developer framework requiring custom UI development on top
- REST API (Aries Agent Admin API) for programmatic control of credential issuance and verification
- Swagger/OpenAPI documentation for all agent endpoints

**Integration points**
- Hyperledger Indy or Besu ledgers for DID and schema anchoring
- Standard ARIES protocols (DIDComm) for agent-to-agent communication
- OID4VC bridge libraries available for interoperability with W3C ecosystem wallets

**Known gaps**
- Not a product; a developer framework requiring significant engineering investment to deploy and maintain
- Hyperledger Indy network governance complexity for self-hosted ledger deployments
- No built-in credential wallet application — must use an Aries-compatible wallet (BCWallet, Lissi, etc.)

**Licence / IP notes**
- Apache 2.0: fully permissive for commercial use. Linux Foundation stewardship ensures governance stability and no contributor lock-in.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- DID issuance and resolution supporting at least did:web and one public ledger-based method
- W3C VC Data Model 2.0-compliant credential issuance with issuer signature and schema validation
- Mobile credential wallet for iOS and Android with biometric unlock
- Verification API: programmatic credential authenticity and revocation status checking without contacting the issuer
- Privacy-preserving revocation mechanism that does not reveal which holder is affected
- Admin console for managing credential schemas, monitoring issuance volume, and auditing verification events

### Differentiating Features
- Zero-knowledge predicate proofs enabling "over 18" without revealing date of birth (AnonCreds, Dock.io)
- Non-human identity (IoT device, AI agent) DID issuance and verification for machine-to-machine trust
- Biometric-linked credential binding preventing credential transfer between users
- EU eIDAS 2.0 EUDI Wallet protocol compatibility for European government identity use cases
- MCP-I (Model Context Protocol Identity) integration for AI agent identity (emerging 2026 standard)

### Underserved Areas / Opportunities
- Mid-market SaaS issuance platform combining no-code credential design, W3C VC issuance, and verification API at accessible pricing for organisations below enterprise scale
- Developer-friendly sandbox environment with synthetic credential issuance for testing verifier integrations without a live identity infrastructure
- Universal verifier API abstracting across multiple DID methods and credential formats, so relying parties do not need to implement each separately
- AI agent identity service: issuing and verifying DIDs for autonomous AI agents under emerging MCP-I standards

### AI-Augmentation Candidates
- Automated schema design: AI generating credential schema from a plain-language description of the attributes needed
- Fraud detection on verification patterns: ML identifying abnormal presentation frequency suggesting credential theft or sharing
- AI-assisted regulatory compliance mapping: checking whether a proposed credential schema satisfies eIDAS 2.0, mDL (ISO 18013-5), or sector-specific requirements
- Natural-language VC query: "verify that this person is a licensed nurse in France without revealing their full name"

## Legal & IP Summary

W3C Verifiable Credentials Data Model 2.0 and DID Core are W3C Recommendations available under W3C's royalty-free patent licence — any compliant implementation is covered. OpenID for Verifiable Credentials (OID4VC) is a work item of the IETF and OpenID Foundation under open terms. AnonCreds is Apache 2.0. Hyperledger Aries is Apache 2.0. The EU eIDAS 2.0 regulation and EUDI Wallet standards are government mandates creating open conformance requirements. ISO/IEC 18013-5 (mDL) requires a licence fee from ISO for the standard document but implementation in software is not separately restricted. The Dock Chain and Microsoft Entra Verified ID are proprietary services; their specific implementation details would require independent re-engineering. Biometric data handling in credential binding requires GDPR Article 9 compliance (explicit consent, purpose limitation). No known patent barriers exist for implementing W3C VC, DID Core, or OID4VC standards.

## Recommended Feature Scope

**Must-have (MVP)**:
- DID issuance and resolution: did:web (immediate, infrastructure-light) and did:ion or did:key (for off-internet use)
- W3C VC Data Model 2.0 credential issuance with JSON-LD schema validation and issuer signature
- Credential wallet: iOS and Android mobile wallet with biometric unlock and backup/recovery
- Verification API: REST endpoint for relying parties to verify credential authenticity and revocation status
- Privacy-preserving revocation registry (Status List 2021 or equivalent) that does not reveal the holder identity when revoked
- Admin console: credential schema management, issuance volume monitoring, and verification audit log

**Should-have (v1.1)**:
- AnonCreds or SD-JWT credential format support for selective disclosure and ZK predicate proofs
- OIDC bridge: allow VC presentation as an OpenID Connect authentication flow for compatibility with existing access management systems
- OID4VC protocol support for interoperability with EUDI Wallet and third-party credential wallets
- Non-human identity: DID issuance and verification for IoT devices and AI agents
- No-code credential design portal for non-developer organisations

**Nice-to-have (backlog)**:
- EU eIDAS 2.0 EUDI Wallet protocol full conformance for European deployment
- ISO 18013-5 mDL (mobile driver's licence) credential format support
- Biometric-linked credential binding preventing credential transfer between users
- MCP-I integration for AI agent identity under the emerging standard
- Universal verifier API abstracting across multiple DID methods and credential formats
