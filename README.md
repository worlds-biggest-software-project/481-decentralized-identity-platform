# Decentralized Identity Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> Self-sovereign identity infrastructure that lets individuals and machines issue, hold, and verify cryptographic credentials without relying on a central authority.

Digital identity today depends on centralised gatekeepers -- governments, social platforms, and enterprises -- that warehouse personal credentials and create honeypot targets for data breaches. The Decentralized Identity Platform gives users and organisations a standards-based toolkit for issuing, storing, and verifying W3C Verifiable Credentials anchored to decentralised identifiers (DIDs), eliminating the need to contact a central issuer at verification time. It targets identity teams at enterprises, government agencies adopting eIDAS 2.0 / mDL mandates, and developers building trust layers for IoT devices and AI agents.

---

## Why Decentralized Identity Platform?

- **Centralised honeypots are the norm.** Every major incumbent (Microsoft Entra Verified ID, Dock.io, Trinsic) stores credential metadata or wallet state in proprietary cloud infrastructure. A breach of the central service compromises the trust anchor for every credential it ever issued.
- **Vendor lock-in through wallets.** Microsoft ties holders to Microsoft Authenticator; Dock.io ties them to the Dock Wallet. Interoperability between wallets remains incomplete, fragmenting the ecosystem rather than unifying it.
- **Enterprise-only pricing excludes mid-market issuers.** Azure charges per verified-credential transaction and requires an Azure AD tenant. Dock.io and Trinsic price for enterprise contracts. Mid-market organisations -- professional licensing bodies, small universities, trade associations -- lack an affordable issuance path.
- **Open-source options are frameworks, not products.** Hyperledger Aries / ACA-Py is production-grade and deployed by governments, but it ships no UI, no wallet, and demands significant engineering investment to operationalise.
- **Non-human identity is an afterthought.** Machine-to-human identity ratios have reached 144:1 in some enterprise environments, yet no incumbent offers a first-class DID issuance path for IoT devices and AI agents integrated with emerging MCP-I standards.

---

## Key Features

### DID Issuance and Resolution

- Create and publish Decentralised Identifiers using multiple DID methods (did:web, did:ion, did:key)
- Ledger-agnostic DID resolver gateway handling resolution across methods without coupling to a single chain
- Support for at least one production-grade public ledger (Hyperledger Indy, Hedera, or Ethereum L2)

### Verifiable Credential Issuance and Management

- W3C VC Data Model 2.0-compliant credential issuance with JSON-LD schema validation and issuer signature
- Admin console for credential schema management, issuance volume monitoring, and verification audit logging
- No-code credential design portal for non-developer organisations (v1.1)

### Credential Wallet

- Mobile wallet for iOS and Android with biometric unlock and backup/recovery
- Selective disclosure: present only the claims a verifier requires
- Browser extension wallet as a low-friction entry point before mobile onboarding

### Verification and Revocation

- REST API and SDK for relying parties to verify credential authenticity and revocation status without contacting the issuer
- Privacy-preserving revocation registry (Status List 2021 or accumulator-based) that does not reveal which holder is affected
- Universal verifier API abstracting across multiple DID methods and credential formats

### Zero-Knowledge Proofs and Selective Disclosure

- AnonCreds or SD-JWT credential format support for ZK predicate proofs (e.g., "over 18" without revealing date of birth)
- Biometric-linked credential binding preventing credential transfer between users

### Non-Human Identity

- DID issuance and verification for IoT devices, AI agents, and automated services
- MCP-I (Model Context Protocol Identity) integration for AI agent identity under the emerging 2026 standard

### Interoperability and Compliance

- OID4VC protocol support for interoperability with EUDI Wallets and third-party credential wallets
- OIDC bridge allowing VC presentation as an OpenID Connect authentication flow
- EU eIDAS 2.0 EUDI Wallet protocol conformance pathway
- ISO 18013-5 mDL (mobile driver's licence) credential format support
- GDPR-aligned data minimisation, data residency options, and configurable retention policies

---

## AI-Native Advantage

AI transforms decentralised identity from a protocol-plumbing exercise into an accessible platform. Automated schema design generates credential schemas from plain-language descriptions of the attributes needed, removing the need for deep standards expertise. ML-powered fraud detection identifies abnormal presentation patterns -- unusual verification frequency or geographic anomalies -- that suggest credential theft or sharing. AI-assisted regulatory compliance mapping checks whether a proposed credential schema satisfies eIDAS 2.0, mDL (ISO 18013-5), or sector-specific requirements before issuance begins. Natural-language VC queries let verifiers express requests like "verify that this person is a licensed nurse in France without revealing their full name" and have the system construct the correct ZK proof request automatically.

---

## Tech Stack & Deployment

The platform is designed for self-hosted, cloud, and hybrid deployment. The verification layer is ledger-agnostic: a DID resolver gateway (building on the DIF Universal Resolver) handles multiple DID methods so the product is not coupled to a single chain. Credential formats follow W3C VC Data Model 2.0 and the OpenID for Verifiable Credentials (OID4VC) protocol family. AnonCreds or JSON-LD proofs provide ZK-capable credentials. Hardware-backed key storage targets Secure Enclave (iOS) and Android StrongBox. FIDO2/WebAuthn attestation is supported where relying parties require it. Schema registries may use IPFS or KERI for decentralised anchoring.

---

## Market Context

The decentralised identity market was valued at approximately $4.89 billion in 2025 and is projected to exceed $7.4 billion in 2026 at a CAGR above 50% ([Security Boulevard, 2026](https://securityboulevard.com/2026/03/decentralized-identity-and-verifiable-credentials-the-enterprise-playbook-2026/)). The EU's eIDAS 2.0 regulation requires every member state to provide a EUDI Wallet by end of 2026, with mandatory acceptance by public and private services from 2027. Primary buyers are enterprise identity teams, government digital-identity programmes, professional licensing bodies, and developers building machine-to-machine trust layers. Incumbent pricing is transaction-based (Microsoft Entra) or enterprise-contract-only (Dock.io, Trinsic), leaving mid-market issuers underserved.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

The core standards underpinning this project -- W3C VC Data Model 2.0, DID Core, and OID4VC -- are available under royalty-free patent licences. Hyperledger Aries and AnonCreds are Apache 2.0. No known patent barriers exist for implementing W3C VC, DID Core, or OID4VC standards.
