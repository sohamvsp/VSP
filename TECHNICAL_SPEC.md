# VSP Technical Specification
## Voloweb Secure Protocol - Detailed Architecture & Design

**Version:** 1.0  
**Date:** December 2025  
**Author:** Soham Kumawat  
**Organization:** Voloweb (Resellton)  
**Status:** Conceptual Research Proposal

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Problem Statement](#2-problem-statement)
3. [Design Goals](#3-design-goals)
4. [System Architecture](#4-system-architecture)
5. [Cryptographic Primitives](#5-cryptographic-primitives)
6. [Protocol Specification](#6-protocol-specification)
7. [DIM - Decentralized Identity Mapping](#7-dim---decentralized-identity-mapping)
8. [MNV - Multi-Node Verification](#8-mnv---multi-node-verification)
9. [Security Analysis](#9-security-analysis)
10. [Performance Considerations](#10-performance-considerations)
11. [Open Questions](#11-open-questions)
12. [References](#12-references)
13. [References](#13-references)

---

## 1. Abstract

The Voloweb Secure Protocol (VSP) is a conceptual post-quantum secure communication protocol designed to replace the Certificate Authority (CA) trust model with a decentralized, multi-witness verification system. VSP leverages NIST-standardized post-quantum cryptographic algorithms (Kyber, Dilithium) and introduces two novel subsystems: Decentralized Identity Mapping (DIM) for certificate-less authentication, and Multi-Node Verification (MNV) for distributed trust consensus.

This document describes the theoretical design, cryptographic foundation, protocol flow, and security properties of VSP. **Note: VSP is not implemented and exists purely as a research concept.**

---

## 2. Problem Statement

### 2.1 Current Web Security Limitations

The modern internet relies on TLS/SSL with X.509 certificates issued by Certificate Authorities. This model has several fundamental weaknesses:

#### 2.1.1 Centralized Trust
- Over 100 root CAs are trusted by default in modern browsers
- Compromise of any single CA can undermine global security
- Government coercion of CAs has been documented (DigiNotar, 2011)

#### 2.1.2 Quantum Vulnerability
- RSA-2048 and ECC P-256 will be broken by sufficiently powerful quantum computers
- Current TLS 1.3 implementations lack post-quantum key exchange
- "Store now, decrypt later" attacks threaten long-term confidentiality

#### 2.1.3 Privacy Concerns
- Certificate Transparency logs expose browsing metadata
- OCSP requests leak site visits to third parties in real-time
- DNS queries reveal user behavior

#### 2.1.4 Single Point of Failure
- CT log operators can censor or manipulate records
- OCSP responders create availability dependencies
- Certificate pinning is fragile and rarely used

### 2.2 Why Existing Solutions Are Insufficient

| Solution | Limitation |
|----------|------------|
| Certificate Transparency | Still requires CAs; logs are centralized |
| DANE/DNSSEC | Relies on DNS infrastructure; not quantum-safe |
| Web of Trust (PGP) | Poor usability; no revocation mechanism |
| Blockchain PKI | High latency; scalability issues; energy costs |

---

## 3. Design Goals

VSP aims to satisfy the following requirements:

### 3.1 Security Goals
- **Post-Quantum Security**: Resist attacks from quantum computers
- **Decentralized Trust**: No single entity controls authentication
- **Forward Secrecy**: Compromise of long-term keys doesn't expose past sessions
- **Replay Protection**: Each handshake is unique and non-replayable

### 3.2 Privacy Goals
- **Metadata Minimization**: Reduce observable connection patterns
- **Anonymity Preservation**: Don't require user identification
- **No Third-Party Leakage**: Eliminate OCSP/CT-style data exposure

### 3.3 Operational Goals
- **Incremental Deployability**: Can coexist with HTTPS
- **Low Latency**: Handshake completes in reasonable time (<500ms)
- **Browser Compatibility**: Works via extension initially
- **Revocation Support**: Handle compromised keys promptly

---

## 4. System Architecture

VSP consists of four primary components:

### 4.1 Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         VSP Ecosystem                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐         ┌──────────────┐                │
│  │   Browser    │◄────────┤ VSP Daemon   │                │
│  │  Extension   │         │   (Client)   │                │
│  └──────────────┘         └───────┬──────┘                │
│                                   │                         │
│                                   ▼                         │
│                         ┌─────────────────┐                │
│                         │  MNV Network    │                │
│                         │  (3+ Nodes)     │                │
│                         └────────┬────────┘                │
│                                  │                          │
│                                  ▼                          │
│                         ┌─────────────────┐                │
│                         │   DIM Ledger    │                │
│                         │ (Domain→PubKey) │                │
│                         └─────────────────┘                │
│                                  ▲                          │
│                                  │                          │
│                         ┌────────┴────────┐                │
│                         │  VSP Server     │                │
│                         │   (Website)     │                │
│                         └─────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 VSP Daemon (Client)

**Responsibilities:**
- Initiates VSP handshakes
- Manages post-quantum key exchange
- Queries MNV nodes for verification
- Maintains local trust cache
- Handles connection upgrades (HTTPS → VSP)

**Communication:**
- Browser Extension ↔ Daemon: WebSocket/IPC
- Daemon ↔ MNV Nodes: JSON-RPC over TLS
- Daemon ↔ VSP Server: Custom binary protocol

### 4.3 VSP Server

**Responsibilities:**
- Responds to VSP handshake requests
- Signs nonces with Dilithium private key
- Participates in Kyber key exchange
- Establishes encrypted tunnel

**Key Material:**
- Dilithium2 keypair (identity)
- Ephemeral Kyber512 keypair (per-session)

### 4.4 MNV Network (Multi-Node Verification)

**Purpose:** Distributed consensus on domain→pubkey mappings

**Node Types:**
- **Validator Nodes**: Maintain full DIM replicas
- **Query Nodes**: Handle client verification requests
- **Archive Nodes**: Store historical DIM states

**Consensus Mechanism:**
- Minimum 3 nodes queried per verification
- 2-of-3 agreement required (configurable threshold)
- Nodes selected via deterministic randomness

### 4.5 DIM Ledger (Decentralized Identity Mapping)

**Purpose:** Replace X.509 certificates with cryptographic bindings

**Structure:**
```json
{
  "domain": "example.com",
  "pubkey": "dilithium2_public_key_bytes",
  "timestamp": 1735948800,
  "signature": "domain_owner_signature",
  "metadata": {
    "contact": "security@example.com",
    "policy_url": "https://example.com/.well-known/vsp-policy"
  }
}
```

**Properties:**
- Append-only (entries cannot be deleted, only superseded)
- Timestamped for auditability
- Self-signed by domain owner's Dilithium key

---

## 5. Cryptographic Primitives

### 5.1 Algorithm Selection Rationale

| Algorithm | Type | Purpose | Justification |
|-----------|------|---------|---------------|
| **Kyber512** | KEM | Key Exchange | NIST PQC winner; fast; 128-bit security |
| **Dilithium2** | Signature | Authentication | NIST PQC winner; small signatures; efficient |
| **ChaCha20-Poly1305** | AEAD | Symmetric Encryption | Fast in software; constant-time |
| **BLAKE3** | Hash | Integrity/KDF | Faster than SHA-3; tree mode parallelism |

### 5.2 Kyber512 Key Encapsulation

**Operation Flow:**
1. Client generates ephemeral Kyber keypair: `(pk_c, sk_c)`
2. Client encapsulates shared secret: `(ct, ss) ← Kyber.Encaps(pk_s)`
3. Server decapsulates: `ss ← Kyber.Decaps(ct, sk_s)`
4. Both derive session keys: `K_enc, K_mac ← BLAKE3-KDF(ss || nonce)`

**Parameters:**
- Public key size: 800 bytes
- Ciphertext size: 768 bytes
- Shared secret: 32 bytes
- Security level: NIST Level 1 (128-bit quantum)

### 5.3 Dilithium2 Digital Signatures

**Operation Flow:**
1. Server signs challenge: `sig ← Dilithium.Sign(sk, nonce || domain)`
2. Client verifies: `valid ← Dilithium.Verify(pk, sig, nonce || domain)`

**Parameters:**
- Public key size: 1,312 bytes
- Signature size: 2,420 bytes
- Security level: NIST Level 2 (128-bit quantum)

### 5.4 ChaCha20-Poly1305 AEAD

**Encryption:**
```
ciphertext, tag ← ChaCha20-Poly1305.Encrypt(
    key=K_enc,
    nonce=counter || session_id,
    plaintext=data,
    associated_data=header
)
```

**Properties:**
- 256-bit key
- 96-bit nonce (64-bit counter + 32-bit session ID)
- 128-bit authentication tag

### 5.5 BLAKE3 Key Derivation

**KDF Construction:**
```
session_keys ← BLAKE3-KDF(
    ikm=kyber_shared_secret,
    info="VSP-v1" || client_random || server_random,
    output_length=96  // 32-byte enc + 32-byte mac + 32-byte IV
)
```

---

## 6. Protocol Specification

### 6.1 Handshake Overview

```
Client                    MNV Nodes              Server
  │                          │                      │
  ├─────── ClientHello ──────┼─────────────────────►│
  │        (nonce_c)         │                      │
  │                          │                      │
  │◄────── ServerHello ──────┼──────────────────────┤
  │     (pk_dil, sig, pk_kyber)                    │
  │                          │                      │
  ├─ VerifyRequest(domain, pk_dil) ────────────────►│
  │                          │                      │
  │◄─ Consensus(valid/invalid) ─────────────────────┤
  │                          │                      │
  ├─── KeyExchange ──────────┼─────────────────────►│
  │   (kyber_ciphertext)     │                      │
  │                          │                      │
  │◄────── Finished ─────────┼──────────────────────┤
  │      (HMAC proof)        │                      │
  │                          │                      │
  ├═══ Encrypted Data ═══════┼═════════════════════►│
  │                          │                      │
```

### 6.2 Phase 1: Identity Assertion

#### 6.2.1 ClientHello

**Structure:**
```json
{
  "version": "VSP/1.0",
  "nonce_client": "<32-byte random>",
  "supported_kems": ["kyber512", "kyber768"],
  "supported_sigs": ["dilithium2", "dilithium3"],
  "extensions": {
    "sni": "example.com",
    "alpn": ["h2", "http/1.1"]
  }
}
```

**Transmission:** Sent in plaintext over TCP (or inside HTTPS tunnel)

#### 6.2.2 ServerHello

**Structure:**
```json
{
  "version": "VSP/1.0",
  "nonce_server": "<32-byte random>",
  "selected_kem": "kyber512",
  "selected_sig": "dilithium2",
  "pubkey_dilithium": "<1312-byte Dilithium2 public key>",
  "signature": "<Dilithium signature of (nonce_c || nonce_s || domain)>",
  "pubkey_kyber": "<800-byte Kyber512 public key>",
  "timestamp": 1735948800
}
```

**Signature Verification:**
```
message = nonce_client || nonce_server || "example.com"
assert Dilithium2.Verify(pubkey_dilithium, signature, message)
```

### 6.3 Phase 2: Multi-Node Verification

#### 6.3.1 Node Selection Algorithm

```python
def select_mnv_nodes(domain: str, num_nodes: int = 3) -> List[Node]:
    seed = BLAKE3(domain + current_day())
    rng = PRNG(seed)
    available_nodes = get_mnv_registry()
    return rng.sample(available_nodes, num_nodes)
```

**Properties:**
- Deterministic for same domain + day
- Resistant to targeted node compromise
- Rotates daily to prevent collusion

#### 6.3.2 Verification Request

**Client → MNV Node:**
```json
{
  "operation": "verify",
  "domain": "example.com",
  "pubkey": "<dilithium2_public_key>",
  "challenge": "<32-byte random>",
  "timestamp": 1735948800
}
```

**MNV Node → Client:**
```json
{
  "result": "valid",
  "confidence": 0.95,
  "dim_entry": {
    "domain": "example.com",
    "pubkey": "<matches_query>",
    "registered": 1700000000,
    "last_updated": 1735000000
  },
  "node_signature": "<node's attestation signature>"
}
```

#### 6.3.3 Consensus Logic

```python
def verify_consensus(responses: List[Response]) -> bool:
    valid_count = sum(1 for r in responses if r.result == "valid")
    threshold = len(responses) * 2 // 3  # 66% threshold
    
    # Require supermajority agreement
    if valid_count >= threshold:
        return True
    
    # If significant disagreement, abort
    return False
```

### 6.4 Phase 3: Key Exchange & Tunnel Establishment

#### 6.4.1 Client Key Exchange

```json
{
  "kyber_ciphertext": "<768-byte Kyber512 ciphertext>",
  "key_confirmation": "<HMAC of session transcript>"
}
```

**Shared Secret Derivation:**
```python
# Both client and server now have:
shared_secret = kyber_shared_secret  # 32 bytes

# Derive session keys
transcript = nonce_c || nonce_s || pk_kyber || ct_kyber
session_keys = BLAKE3_KDF(
    ikm=shared_secret,
    salt=transcript,
    info="VSP-v1-session-keys",
    output_length=96
)

K_c2s_enc = session_keys[0:32]   # Client → Server encryption
K_s2c_enc = session_keys[32:64]  # Server → Client encryption
K_mac = session_keys[64:96]      # HMAC key for key confirmation
```

#### 6.4.2 Server Finished Message

```json
{
  "status": "ready",
  "key_confirmation": "<HMAC of session transcript>",
  "session_id": "<unique session identifier>"
}
```

### 6.5 Data Transmission

**Packet Format:**
```
┌────────────────────────────────────────────────────┐
│ Header (12 bytes)                                  │
├────────────────────────────────────────────────────┤
│ • Version (1 byte): 0x01                          │
│ • Flags (1 byte): [encrypted|compressed|...]      │
│ • Sequence Number (8 bytes): monotonic counter    │
│ • Length (2 bytes): payload size                  │
├────────────────────────────────────────────────────┤
│ Encrypted Payload (variable)                       │
├────────────────────────────────────────────────────┤
│ Poly1305 Tag (16 bytes)                           │
└────────────────────────────────────────────────────┘
```

**Encryption:**
```
nonce = sequence_number || session_id_truncated
ciphertext = ChaCha20(plaintext, key=K_enc, nonce=nonce)
tag = Poly1305(ciphertext || header, key=K_mac)
```

---

## 7. DIM - Decentralized Identity Mapping

### 7.1 Architecture

DIM is a distributed, append-only ledger storing domain→pubkey bindings.

**Design Considerations:**
- **Not a Blockchain**: Avoids proof-of-work/stake overhead
- **Eventually Consistent**: Tolerates temporary inconsistencies
- **Merkle Tree Structure**: Enables efficient proofs of inclusion

### 7.2 Entry Format

```json
{
  "version": 1,
  "domain": "example.com",
  "pubkey_dilithium2": "<base64-encoded-public-key>",
  "timestamp": 1735948800,
  "signature": "<self-signature by domain owner>",
  "metadata": {
    "contact": "security@example.com",
    "policy": "https://example.com/.well-known/vsp-policy.json"
  },
  "previous_hash": "<hash of previous entry for this domain>"
}
```

### 7.3 Registration Process

**Step 1: Domain Owner generates keypair**
```bash
$ vsp-keygen --domain example.com
Generated Dilithium2 keypair:
  Public Key: dilithium2_pk_abc123...
  Private Key: [REDACTED]
```

**Step 2: Create DIM Entry**
```bash
$ vsp-register --domain example.com --pubkey dilithium2_pk_abc123...
Entry created:
  Domain: example.com
  PubKey: dilithium2_pk_abc123...
  Signature: [self-signed]
```

**Step 3: Submit to DIM Network**
```bash
$ vsp-submit-entry entry.json
Submitted to 5 DIM nodes
Confirmed by 4/5 nodes
Propagation ETA: ~10 minutes
```

### 7.4 Update Mechanism

**Key Rotation:**
```json
{
  "domain": "example.com",
  "new_pubkey": "<new_dilithium2_key>",
  "signature_old": "<signed with OLD private key>",
  "signature_new": "<signed with NEW private key>",
  "reason": "routine_rotation",
  "effective_date": 1736000000
}
```

**Grace Period:** 30 days where both old and new keys are accepted

### 7.5 Revocation

**Emergency Revocation:**
```json
{
  "domain": "example.com",
  "action": "revoke",
  "pubkey": "<compromised_key>",
  "signature": "<signed by domain owner OR 3-of-5 emergency contacts>",
  "reason": "key_compromise",
  "timestamp": 1735948800
}
```

**Immediate Effect:** MNV nodes reject the key within propagation delay (~1 minute)

---

## 8. MNV - Multi-Node Verification

### 8.1 Network Topology

```
        ┌─────────────────────────────────┐
        │       MNV Coordinator           │
        │  (Node Discovery & Routing)     │
        └──────────────┬──────────────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    ┌────▼───┐    ┌───▼────┐   ┌───▼────┐
    │ Node A │    │ Node B │   │ Node C │
    │ (Geo:  │    │ (Geo:  │   │ (Geo:  │
    │  US)   │    │  EU)   │   │  ASIA) │
    └────┬───┘    └───┬────┘   └───┬────┘
         │            │            │
         └────────────┼────────────┘
                      ▼
              ┌───────────────┐
              │  DIM Ledger   │
              │   (Replicas)  │
              └───────────────┘
```

### 8.2 Node Requirements

**Hardware:**
- 4+ CPU cores
- 16GB RAM
- 500GB SSD storage
- 1Gbps network

**Software:**
- Full DIM replica (100GB+ estimated)
- VSP daemon
- Dilithium2 + Kyber512 libraries

**Incentives (Future Work):**
- Micropayments per query (Lightning Network)
- Reputation system (slashing for misbehavior)

### 8.3 Query Protocol

**Request Format:**
```
POST /mnv/verify
Content-Type: application/json

{
  "domain": "example.com",
  "claimed_pubkey": "<dilithium2_pk>",
  "challenge": "<32-byte nonce>",
  "client_signature": "<optional client attestation>"
}
```

**Response Format:**
```json
{
  "status": "valid",
  "dim_entry": { },
  "merkle_proof": ["hash1", "hash2"],
  "node_id": "node-a-us-east",
  "signature": "<node's attestation of this response>"
}
```

### 8.4 Byzantine Fault Tolerance

**Assumptions:**
- At most 1/3 of nodes are malicious
- Network partitions are temporary
- Clients query diverse geographic nodes

**Mitigation Strategies:**
1. **Redundant Queries**: Always query 3+ nodes
2. **Stake-Based Selection**: Higher-reputation nodes preferred
3. **Fraud Proofs**: Clients can report malicious nodes
4. **Economic Penalties**: Staked tokens slashed for provable misbehavior

---

## 9. Security Analysis

### 9.1 Threat Model

**Adversary Capabilities:**
- Compromises up to 1/3 of MNV nodes
- Controls network (active MITM)
- Has quantum computer (post-2030 scenario)
- Can coerce CAs (legacy HTTPS context)

**Out of Scope:**
- Endpoint compromise (malware on client/server)
- Side-channel attacks (timing, power analysis)
- Social engineering attacks

### 9.2 Security Properties

#### 9.2.1 Post-Quantum Security

**Theorem:** VSP resists quantum attacks if:
1. Kyber512 is IND-CCA2 secure against quantum adversaries
2. Dilithium2 is EUF-CMA secure against quantum adversaries

**Proof Sketch:** All authentication and key exchange rely exclusively on PQC algorithms. Breaking the protocol requires breaking NIST Level 1+ security.

#### 9.2.2 Forward Secrecy

VSP achieves forward secrecy through:
- Ephemeral Kyber keypairs (regenerated per-session)
- Session keys derived from ephemeral shared secrets
- No reuse of key material across sessions

**Property:** Compromise of long-term Dilithium keys does not compromise past session keys.

#### 9.2.3 Replay Protection

Each handshake includes:
- Client nonce (nonce_c)
- Server nonce (nonce_s)
- Timestamp validation (±5 minutes)

**Property:** Captured handshakes cannot be replayed.

#### 9.2.4 Identity Binding

**Guarantee:** A domain's identity is cryptographically bound to its Dilithium public key in DIM.

**Attack Resistance:**
- **Impersonation**: Attacker cannot forge Dilithium signatures
- **Substitution**: MNV consensus prevents unauthorized key changes
- **Domain Hijacking**: Requires compromising both domain control AND private key

### 9.3 Attack Scenarios

#### 9.3.1 Sybil Attack on MNV

**Attack:** Adversary operates 50% of MNV nodes

**Mitigation:**
- Client selects nodes via deterministic randomness (hard to predict)
- Geographic diversity requirements
- Stake-weighted selection (costly to acquire majority stake)

**Residual Risk:** If adversary controls >66% of nodes, consensus breaks.

#### 9.3.2 DIM Poisoning

**Attack:** Inject malicious domain→pubkey mappings

**Mitigation:**
- All entries self-signed by domain owner
- Requires proof of DNS control (TXT record validation)
- MNV nodes verify signatures before accepting entries

#### 9.3.3 Downgrade Attack

**Attack:** Force client to use VSP-over-HTTPS with compromised HTTPS

**Mitigation:**
- VSP daemon performs independent verification (doesn't trust HTTPS)
- MNV consensus prevents MITM even if HTTPS is broken
- Optional: HSTS-like "VSP-only" policy

---

## 10. Performance Considerations

### 10.1 Handshake Latency

**Component Breakdown:**
| Phase | Operations | Est. Time |
|-------|-----------|-----------|
| ClientHello/ServerHello | 1 RTT | 20-100ms |
| MNV Verification (3 nodes) | Parallel queries | 50-200ms |
| Key Exchange | 1 RTT | 20-100ms |
| **Total** | | **90-400ms** |

**Comparison to TLS 1.3:**
- TLS 1.3: ~50-150ms (1-RTT)
- VSP: ~90-400ms (2-RTT + MNV)

**Overhead:** ~50-250ms additional latency

### 10.2 Computational Cost

**Client Operations:**
- Dilithium2 verification: ~1ms
- Kyber512 encapsulation: ~0.5ms
- ChaCha20-Poly1305: ~1GB/s throughput

**Server Operations:**
- Dilithium2 signing: ~2ms
- Kyber512 decapsulation: ~0.5ms

**Bottleneck:** MNV network queries (not cryptography)

### 10.3 Bandwidth Overhead

**Handshake Size:**
- ClientHello: ~200 bytes
- ServerHello: ~4KB (Dilithium2 pubkey + sig + Kyber pubkey)
- MNV queries: ~500 bytes × 3 = 1.5KB
- Key Exchange: ~800 bytes (Kyber ciphertext)

**Total:** ~6.5KB (vs. ~4KB for TLS 1.3)

**Data Transmission:**
- ChaCha20-Poly1305 overhead: 16 bytes per packet
- Negligible compared to TLS

### 10.4 Scalability

**DIM Storage:**
- Estimate: 100M domains × 5KB per entry = 500GB
- Growth: ~10GB/year (assuming 2M new domains/year)

**MNV Network:**
- Query rate: ~10,000 qps per node (estimated)
- 1000 nodes → 10M qps global capacity

---

## 11. Open Questions

### 11.1 Technical Challenges

**Q1:** How to bootstrap initial trust in MNV nodes?
- **Options:** Hard-coded seed nodes, DNSSEC-based discovery, social consensus
- **Status:** Unsolved

**Q2:** How to handle DIM storage growth?
- **Options:** Pruning old entries, sharding by TLD, pay-per-entry model
- **Status:** Needs economic analysis

**Q3:** What happens if all 3 MNV nodes disagree?
- **Options:** Query additional nodes, fall back to HTTPS, user warning
- **Status:** Needs UX research

### 11.2 Governance & Economics

**Q4:** Who pays for MNV node operation?
- **Options:** Micropayments, subscription model, altruism, ads
- **Status:** Economic model undefined

**Q5:** How to prevent DIM spam?
- **Options:** Proof-of-work, registration fees, reputation systems
- **Status:** Requires game-theoretic analysis

### 11.3 Adoption Challenges

**Q6:** How to incentivize websites to adopt VSP?
- **Value Props:** Marketing ("quantum-safe"), privacy, decentralization
- **Barriers:** Development cost, uncertainty, chicken-egg problem

**Q7:** Browser vendor buy-in?
- **Status:** Would require Mozilla/Google collaboration
- **Alternative:** Extension-only deployment

---

## 12. References

1. NIST Post-Quantum Cryptography Standardization (2024)
2. "ML-KEM (Kyber) Specification" - NIST FIPS 203
3. "ML-DSA (Dilithium) Specification" - NIST FIPS 204
4. "SoK: Certificate Transparency" - IEEE S&P 2021
5. "The BLAKE3 Cryptographic Hash Function" - 2020
6. "ChaCha20 and Poly1305 for IETF Protocols" - RFC 8439
7. "Analysis of Certificate Authority Trust Models" - USENIX Security 2012
8. "Post-Quantum TLS Without Handshake Signatures" - ACM CCS 2020

---

**Note:** This is a conceptual research proposal. For questions or collaboration, please open an issue in the repository or contact: soham@voloweb.io
