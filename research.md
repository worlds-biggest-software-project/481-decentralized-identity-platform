# 481 - Decentralized Identity Platform

**Date:** 2026-05-02

## 1. Problem Statement

Digital identity today relies on centralised authorities — governments, social platforms, and enterprises — that store and control personal credentials on behalf of users. This creates honeypot targets for data breaches, forces repeated re-verification across services, and leaves individuals with little sovereignty over their own identity information. Organisations managing non-human identities (bots, devices, AI agents) face the same structural problem at even greater scale. A decentralised identity platform addresses these issues by enabling self-sovereign, cryptographically verifiable credentials that any relying party can confirm without contacting a central issuer.

## 2. Market Landscape

The decentralised identity market was valued at approximately $4.89 billion in 2025 and is projected to exceed $7.4 billion in 2026 at a CAGR above 50%. Regulatory momentum is a primary driver: the EU's eIDAS 2.0 regulation requires every member state to provide at least one European Digital Identity (EUDI) Wallet to citizens by end of 2026, and public and private services must accept it for authentication from 2027. Gartner predicts 30% of enterprises will adopt decentralised identity tools to mitigate risk by 2026. Demand is compounded by explosive growth in non-human identities — machine-to-human ratios in some enterprise environments have reached 144:1, growing 44% year over year.

Key players include Microsoft Entra Verified ID, Dock.io, Trinsic, Extrimian, and Hedera's decentralised identity stack. Mobile driver's licences (mDL, ISO/IEC 18013-5) are gaining traction in the US and Australia as a near-term mainstream use case.

## 3. Core Features / Functional Requirements

- **DID issuance and resolution:** Create, publish, and resolve Decentralised Identifiers (DIDs) anchored to a public ledger or peer-to-peer network, with support for multiple DID methods (did:web, did:ion, did:hedera).
- **Verifiable credential issuance:** Issue W3C-compliant Verifiable Credentials (VCs) signed with the issuer's private key, covering professional licences, educational certificates, employment records, and government IDs.
- **Credential wallet:** Mobile and browser wallet for storing, managing, and selectively disclosing VCs, with biometric unlock and backup/recovery options.
- **Verification API:** Relying-party SDK and REST API to verify credential authenticity, revocation status, and schema conformance without contacting the issuer directly.
- **Revocation registry:** Privacy-preserving revocation mechanism (e.g., status list 2021 or accumulator-based) so issuers can revoke credentials without revealing which holder is affected.
- **Selective disclosure and ZK proofs:** Allow holders to prove claims (e.g., "I am over 18") without revealing underlying data, using zero-knowledge proof techniques.
- **Non-human identity support:** Issue and verify DIDs for IoT devices, AI agents, and automated services, enabling machine-to-machine trust without centralised PKI.
- **Admin console:** Issuer portal for managing credential schemas, monitoring issuance volume, and auditing verification events.
- **Compliance and privacy controls:** Data residency options, GDPR-aligned data minimisation, and configurable retention policies.

## 4. Technical Considerations

The platform must support at least one production-grade public ledger (Hyperledger Indy, Hedera, or an Ethereum L2) without coupling the entire product to a single chain. A DID resolver gateway that handles multiple DID methods keeps the verification layer ledger-agnostic. Credential formats should follow W3C VC Data Model 2.0 and the OpenID for Verifiable Credentials (OID4VC) family of protocols to maximise interoperability with existing identity wallets and government EUDI implementations.

Key technical challenges include: (a) key rotation and recovery without breaking previously issued credentials; (b) performance of on-chain revocation checks at scale; (c) hardware-backed key storage on consumer devices (Secure Enclave / Android StrongBox); (d) cross-border legal recognition of digitally issued credentials. The wallet application must pass FIDO2/WebAuthn attestation checks where relying parties require it. For AI agent identity, the platform should integrate with emerging MCP-I (Model Context Protocol Identity) standards.

Tooling choices: DIF Universal Resolver for multi-method DID resolution, AnonCreds or JSON-LD proofs for ZK-capable credentials, IPFS or KERI for decentralised schema registries.

## 5. Key Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Regulatory fragmentation (EU vs US vs APAC credential formats) | High | High | Build to W3C/OID4VC standards; use pluggable format adapters; monitor eIDAS 2.0 conformance requirements |
| Low issuer adoption without network effects | High | High | Launch with a vertically integrated use case (e.g., professional licensing body) to establish a live credential ecosystem before opening to general issuers |
| Key loss or theft resulting in permanent identity compromise | Medium | High | Mandatory social recovery and hardware-backed key generation; offer custodial recovery as a compliance option |
| Privacy regulation conflicts with on-chain data | Medium | High | Store no personal data on-chain; anchor only cryptographic commitments; use off-chain VC storage with on-chain revocation bits |
| User adoption friction from wallet UX | High | Medium | Invest in progressive disclosure UX; offer browser extension wallet as a low-friction entry point before mobile onboarding |

## Citations

- [Decentralized Identity and Verifiable Credentials: The Enterprise Playbook 2026 - Security Boulevard](https://securityboulevard.com/2026/03/decentralized-identity-and-verifiable-credentials-the-enterprise-playbook-2026/)
- [Decentralized Identity: The Ultimate Guide 2026 - Dock.io](https://www.dock.io/post/decentralized-identity)
- [Decentralized Identifiers (DIDs): The Ultimate Beginner's Guide 2026 - Dock.io](https://www.dock.io/post/decentralized-identifiers)
- [Introduction to Microsoft Entra Verified ID - Microsoft Learn](https://learn.microsoft.com/en-us/entra/verified-id/decentralized-identifier-overview)
- [AI Agents with Decentralized Identifiers and Verifiable Credentials - arXiv](https://arxiv.org/html/2511.02841v1)
- [Decentralized Identity & MCP-I: Know Your Agent - Vouched.id](https://www.vouched.id/learn/blog/decentralized-identity-did-and-blockchain-a-future-vision-for-user-controlled-identity)
- [Verifiable Credentials, DIDs, and Cybersecurity - Extrimian](https://extrimian.io/verifiable-credentials-issuing/)
- [Decentralized Identity for AI Using Blockchain - Blockchain Council](https://www.blockchain-council.org/blockchain/decentralized-identity-for-ai-blockchain-authenticate-agents-devices-users/)
