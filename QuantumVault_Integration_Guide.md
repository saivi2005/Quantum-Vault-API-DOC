# QuantumVault - Integration Guide

> Post-Quantum Cryptography as a Service - for developers and architects building quantum-safe systems today.

#### Table of Contents

- [What is QuantumVault?](#what-is-quantumvault)
- [Why Post-Quantum Cryptography Now?](#why-post-quantum-cryptography-now)
- [Platform Architecture](#platform-architecture)
  - [System Overview](#system-overview)
  - [Identity Model](#identity-model)
  - [Request Security Model](#request-security-model)
- [Quick Start - 5 Steps to First API Call](#quick-start--5-steps-to-first-api-call)
- [Core Concepts](#core-concepts)
  - [PQC Keys](#pqc-keys)
  - [Authentication Keys](#authentication-keys)
  - [Access Policies](#access-policies)
  - [Supported Algorithms](#supported-algorithms)
- [Making API Calls](#making-api-calls)
  - [Base URL](#base-url)
  - [Required Headers](#required-headers)
  - [Request Body Structure](#request-body-structure)
  - [Request Signing - How It Works](#request-signing--how-it-works)
  - [Signing with ML-DSA (Recommended)](#signing-with-ml-dsa-recommended)
  - [Signing with ECDSA](#signing-with-ecdsa)
  - [Signing with RSA](#signing-with-rsa)
  - [Signing with Hybrid (ML-DSA + ECDSA)](#signing-with-hybrid-ml-dsa--ecdsa)
- [Cryptographic Operations](#cryptographic-operations)
  - [Sign Data](#sign-data)
  - [Verify Signature](#verify-signature)
  - [Encapsulate - KEM](#encapsulate--kem)
  - [Decapsulate - KEM](#decapsulate--kem)
  - [Encrypt - AES-GCM](#encrypt--aes-gcm)
  - [Decrypt - AES-GCM](#decrypt--aes-gcm)
- [Operation Chaining](#operation-chaining)
  - [Sign → Verify](#sign--verify)
  - [Encapsulate → Decapsulate](#encapsulate--decapsulate)
  - [Encrypt → Decrypt](#encrypt--decrypt)
- [Real-World Use Cases](#real-world-use-cases)
  - [Quantum-Safe Document Signing](#quantum-safe-document-signing)
  - [Secure API Request Authentication](#secure-api-request-authentication)
  - [Post-Quantum JWT Replacement](#post-quantum-jwt-replacement)
  - [Encrypted Data Storage](#encrypted-data-storage)
  - [Secure Key Exchange Between Services](#secure-key-exchange-between-services)
  - [File Integrity Verification](#file-integrity-verification)
- [Security Controls](#security-controls)
  - [IP Whitelisting](#ip-whitelisting)
  - [Mutual TLS (mTLS)](#mutual-tls-mtls)
  - [Rate Limiting](#rate-limiting)
  - [Audit Logging](#audit-logging)
- [Migration from Classical Cryptography](#migration-from-classical-cryptography)
  - [From RSA Signing to ML-DSA](#from-rsa-signing-to-ml-dsa)
  - [From ECDH Key Exchange to ML-KEM](#from-ecdh-key-exchange-to-ml-kem)
  - [From AES with Local Key Management to QuantumVault AES-256](#from-aes-with-local-key-management-to-quantumvault-aes-256)
  - [Hybrid Migration Strategy](#hybrid-migration-strategy)
- [Testing Your Integration](#testing-your-integration)
  - [PQC Tester Tool](#pqc-tester-tool)
  - [Tester Workflow](#tester-workflow)
- [Error Reference](#error-reference)
- [Security Best Practices](#security-best-practices)

---

## What is QuantumVault?

QuantumVault is a **Post-Quantum Cryptography as a Service (PQCaaS)** platform. It gives your application access to NIST-standardised post-quantum cryptographic operations — signing, verification, key encapsulation, and symmetric encryption — through a simple REST API, without requiring your team to manage cryptographic key material directly.

**In plain terms:** instead of bundling cryptographic libraries into your application and managing keys yourself, you call QuantumVault's API to perform operations. Your private keys never leave the vault. Your application only ever sees outputs — signatures, ciphertexts, shared secrets.

**What you can do with QuantumVault:**

- Sign data and verify signatures using ML-DSA, ECDSA, RSA, or Hybrid algorithms.
- Establish shared secrets between services using post-quantum Key Encapsulation Mechanisms (KEM).
- Encrypt and decrypt data using AES-256-GCM backed by vault-managed symmetric keys.
- Enforce access control through policies that bind authenticated identities to specific key operations.
- Protect cryptographic endpoints with IP whitelisting, mTLS, and rate limits.

---

## Why Post-Quantum Cryptography Now?

Classical cryptographic algorithms — RSA, ECDSA, ECDH — derive their security from mathematical problems (integer factorisation, discrete logarithm) that are computationally hard for classical computers. A sufficiently powerful quantum computer running **Shor's algorithm** can solve these problems efficiently, breaking the security of these schemes entirely.

The threat is not theoretical. Adversaries can collect encrypted data **today** and decrypt it once quantum computers become capable — a strategy known as **"Harvest Now, Decrypt Later" (HNDL)**. Data with a long confidentiality requirement (medical records, financial contracts, government communications, IP) is already at risk.

NIST completed its post-quantum cryptography standardisation process in 2024, publishing:

| Standard    | QuantumVault Algorithm | Purpose                     |
|-------------|------------------------|-----------------------------|
| FIPS 204    | ML-DSA (Dilithium)     | Digital Signatures          |
| FIPS 203    | ML-KEM (Kyber)         | Key Encapsulation / Exchange |

QuantumVault implements these standards and makes them API-accessible. You do not need to compile cryptographic libraries or manage FIPS-compliant key storage — QuantumVault handles it.

---

## Platform Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Your Application                       │
│                                                             │
│  1. Sign request body locally (using Auth Private Key)      │
│  2. Send signed request with Key IDs in headers             │
└───────────────────────┬─────────────────────────────────────┘
                        │  HTTPS
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    QuantumVault API                         │
│                                                             │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────┐   │
│  │   Policy     │   │  Signature     │   │  IP / mTLS   │   │
│  │  Enforcement │──▶│  Verification  │─▶│   Guards     │   │
│  └──────────────┘   └────────────────┘   └──────┬───────┘   │
│                                                 │           │
│  ┌──────────────────────────────────────────────▼─────── ┐  │
│  │                  Crypto Engine                        │  │
│  │   ML-DSA · ML-KEM · ECDSA · RSA · AES-256-GCM         │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌────────────────┐   ┌──────────────────┐                  │
│  │   Key Store    │   │   Audit Logger   │                  │
│  │ (Encrypted at  │   │ (Immutable Trail)│                  │
│  │    rest)       │   └──────────────────┘                  │
│  └────────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

### Identity Model

Every API call to a cryptographic endpoint identifies itself using **two IDs passed in request headers** — never in the request body:

| Header          | What it identifies                                                      |
|-----------------|-------------------------------------------------------------------------|
| `x-pqc-key-id`  | The PQC Key to use for the operation (Sign, Verify, Encrypt, KEM, etc.) |
| `x-auth-key-id` | The Authentication Key proving the caller's identity                    |

The pair of IDs must match an active **Access Policy** in your QuantumVault account. The policy controls which operations are permitted and enforces any rate limits.

### Request Security Model

Every request body is **signed by your application** using the private key corresponding to your Authentication Key. QuantumVault verifies this signature server-side before executing any cryptographic operation.

```
Your App                          QuantumVault
────────                          ────────────
 1. Build body { data, timestamp }
 2. Canonicalize body (sorted keys)
 3. Sign canonical body with your
    Auth Private Key
 4. Send:
    Headers:
      x-pqc-key-id  →             Looks up PQC Key
      x-auth-key-id →             Looks up Auth Key (public key)
      x-signature   →             Verifies signature
    Body: { data, timestamp }
                                  5. If valid → execute operation
                                  6. Return result
```

**Key security properties:**

- Private keys for authentication are held **only by your application** — QuantumVault only stores the corresponding public key.
- PQC private keys (used for signing your data, KEM, encryption) are held **only by QuantumVault** — your application never sees them.
- Every request is replay-protected via the `timestamp` field in the body.
- Identity (Key IDs) is enforced exclusively through headers, never embedded in body data.

---

## Quick Start — 5 Steps to First API Call

### Step 1 — Create a PQC Key

Log into QuantumVault, go to **PQC Keys → Create PQC Key**. Choose an algorithm (start with `ML-DSA` for signing), give it a name, and set the environment.

Note the key's **ID** from the key list — this becomes your `x-pqc-key-id`.

### Step 2 — Create an Authentication Key

Go to **Authentication Keys → Create Authentication Key**. Choose your algorithm (`RSA` for broad compatibility, or `ML-DSA` for full post-quantum authentication). Use **Generate Key Pair** to let QuantumVault create the keypair — download the private key immediately; it will not be shown again.

Note the key's **ID** — this becomes your `x-auth-key-id`.

### Step 3 — Create an Access Policy

Go to **Policies → Create Access Policy**. Link your Authentication Key and PQC Key together. Select the operations you need (`Sign`, `Verify`, etc.). Set an optional rate limit.

Your application can only use key combinations that have an active policy.

### Step 4 — Configure Security Controls (Recommended)

In **Settings**, configure:

- **IP Restriction** — restrict calls to your server's IP addresses.
- **mTLS** — require client certificates for additional identity assurance.

### Step 5 — Make Your First API Call

```bash
# Sign "Hello World" using your key pair
# (The tester at https://katoki-dev.github.io/PQC-Tester/ does this interactively)

curl -X POST https://your-api-base/api/crypto/sign \
  -H "Content-Type: application/json" \
  -H "x-pqc-key-id: YOUR_PQC_KEY_ID" \
  -H "x-auth-key-id: YOUR_AUTH_KEY_ID" \
  -H "x-signature: YOUR_COMPUTED_SIGNATURE" \
  -d '{"data":"Hello World","timestamp":"1749112345678"}'
```

---

## Core Concepts

### PQC Keys

A **PQC Key** is the cryptographic key stored securely inside QuantumVault that performs the actual operation. Your application never has access to PQC private key material.

| Algorithm  | Operations               | Standard      |
|------------|--------------------------|---------------|
| ML-DSA     | Sign, Verify             | FIPS 204      |
| ML-KEM     | Encapsulate, Decapsulate | FIPS 203      |
| ECDSA      | Sign, Verify             | NIST P-256    |
| Hybrid-DSA | Sign, Verify             | ML-DSA + ECDSA|
| Hybrid-KEM | Encapsulate, Decapsulate | ML-KEM + X25519|
| AES-256    | Encrypt, Decrypt         | AES-256-GCM   |

Keys support versioning through **rotation**. When you rotate a key, the old version is marked `rotated`, a new version is generated, and policies are automatically migrated.

### Authentication Keys

An **Authentication Key** is a keypair where:

- The **public key** is stored in QuantumVault to verify your request signatures.
- The **private key** is held **only by your application** and used to sign each request body.

| Algorithm | Description                          | Private Key Format          |
|-----------|--------------------------------------|-----------------------------|
| RSA       | 2048-bit RSA                         | PKCS#8 PEM                  |
| ECDSA     | P-256 Elliptic Curve                 | PKCS#8 PEM                  |
| ML-DSA    | Post-Quantum (FIPS 204)              | 4032-byte hex               |
| Hybrid    | ML-DSA + ECDSA combined              | ECDSA PEM + ML-DSA hex      |

### Access Policies

A **Policy** is a binding rule that says: *"Authentication Key A is allowed to perform operations [Sign, Verify] on PQC Key B, up to N requests per hour."*

No policy → no access. Both keys in the policy must be `active`.

### Supported Algorithms

```
Signing / Verification:
  ML-DSA-65   (FIPS 204) — Post-Quantum
  ECDSA P-256             — Classical
  RSA-2048                — Classical
  Hybrid                  — ML-DSA-65 + ECDSA P-256

Key Encapsulation:
  ML-KEM-768  (FIPS 203) — Post-Quantum
  Hybrid-KEM              — ML-KEM-768 + X25519

Symmetric Encryption:
  AES-256-GCM             — Authenticated Encryption
```

---

## Making API Calls

### Base URL

```
https://<your-deployment>/api/crypto
```

All cryptographic operation endpoints are under `/api/crypto/`.

### Required Headers

Every request to a crypto endpoint must include:

| Header           | Value                                      | Notes                               |
|------------------|--------------------------------------------|-------------------------------------|
| `Content-Type`   | `application/json`                         | Always required                     |
| `x-pqc-key-id`   | UUID of the PQC Key                        | Identity — never put this in body   |
| `x-auth-key-id`  | UUID of the Authentication Key             | Identity — never put this in body   |
| `x-signature`    | Hex-encoded signature of the request body  | Computed by your app (see below)    |

> **Design rule:** Key IDs identify *who is making the request* and *which key to use*. They belong in headers as part of the transport identity layer — not in the request body, which is treated as application data.

### Request Body Structure

Every request body must contain:

```json
{
  "timestamp": "1749112345678",
  "data": "your payload here"
}
```

| Field     | Type   | Required | Description                                           |
|-----------|--------|----------|-------------------------------------------------------|
| timestamp | String | Always   | Unix timestamp in milliseconds as a string — prevents replay attacks |
| data      | String | Depends on operation | The data to operate on                  |
| signature | String / Object | Verify only | Signature to verify              |
| ciphertext| String / Object | Decapsulate, Decrypt | Ciphertext to process     |
| iv        | String | Decrypt only | Initialization vector from encrypt response         |
| authTag   | String | Decrypt only | Auth tag from encrypt response                      |

### Request Signing — How It Works

Before sending any request, your application must:

1. Build the request body as a JSON object with `timestamp` and any operation-specific fields.
2. **Canonicalize** the body — sort all keys alphabetically and serialize to a compact JSON string with no extra whitespace.
3. Sign the canonical JSON bytes using your Auth Private Key.
4. Hex-encode the resulting signature.
5. Include the hex signature in the `x-signature` header.

**Canonicalization is critical.** The server canonicalizes the received body using the same algorithm before verifying the signature. Any difference in key ordering or whitespace will cause verification failure.

```javascript
// Canonical JSON — deterministic, sorted key order, no whitespace
function canonicalizeJSON(obj) {
  if (obj === null || typeof obj !== 'object') return JSON.stringify(obj);
  if (Array.isArray(obj)) {
    return '[' + obj.map(item => canonicalizeJSON(item)).join(',') + ']';
  }
  const sortedKeys = Object.keys(obj).sort();
  return '{' + sortedKeys.map(k => `"${k}":${canonicalizeJSON(obj[k])}`).join(',') + '}';
}
```

---

### Signing with ML-DSA (Recommended)

ML-DSA is the NIST FIPS 204 post-quantum digital signature standard. It is the recommended authentication algorithm for full quantum resistance.

**Private key format:** 4032-byte hex string (or PEM-wrapped).

**JavaScript (Browser / Node with `@noble/post-quantum`):**

```javascript
import { ml_dsa65 } from '@noble/post-quantum/ml-dsa';

function toHex(buf) {
  return Array.from(new Uint8Array(buf), b => b.toString(16).padStart(2, '0')).join('');
}

function parseMlDsaKey(hexOrPem) {
  const clean = hexOrPem.trim();
  let bytes;
  if (clean.startsWith('-----')) {
    // PEM-wrapped
    const b64 = clean.replace(/-----[^-]+-----/g, '').replace(/\s+/g, '');
    bytes = Uint8Array.from(atob(b64), c => c.charCodeAt(0));
  } else {
    const hex = clean.replace(/0x|[:\s]/g, '');
    bytes = new Uint8Array(hex.match(/.{1,2}/g).map(b => parseInt(b, 16)));
  }
  // ML-DSA-65 secret key is exactly 4032 bytes
  if (bytes.length > 4032) return bytes.slice(bytes.length - 4032);
  return bytes;
}

async function signRequest(body, mlDsaPrivateKeyHex) {
  const canonical = canonicalizeJSON(body);
  const bodyBytes = new TextEncoder().encode(canonical);
  const secretKey = parseMlDsaKey(mlDsaPrivateKeyHex);
  const sig = ml_dsa65.sign(bodyBytes, secretKey);
  return toHex(sig);
}

// Usage
const body = { data: 'Hello World', timestamp: Date.now().toString() };
const signature = await signRequest(body, process.env.ML_DSA_PRIVATE_KEY);

const response = await fetch(`${API_BASE}/crypto/sign`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-pqc-key-id':  PQC_KEY_ID,
    'x-auth-key-id': AUTH_KEY_ID,
    'x-signature':   signature
  },
  body: JSON.stringify(body)
});
```

**Python (with `dilithium-py` or `pyml-dsa`):**

```python
import json
import os
import binascii
from pyml_dsa import ml_dsa_65  # pip install pyml-dsa

def canonicalize(obj):
    if not isinstance(obj, dict):
        return json.dumps(obj, separators=(',', ':'))
    return '{' + ','.join(
        f'"{k}":{canonicalize(v)}'
        for k, v in sorted(obj.items())
    ) + '}'

def sign_request(body: dict, private_key_hex: str) -> str:
    canonical = canonicalize(body)
    body_bytes = canonical.encode('utf-8')
    secret_key = bytes.fromhex(private_key_hex.strip())
    # Trim to 4032 bytes if needed
    if len(secret_key) > 4032:
        secret_key = secret_key[-4032:]
    signature = ml_dsa_65.sign(body_bytes, secret_key)
    return signature.hex()

import time
import requests

body = {'data': 'Hello World', 'timestamp': str(int(time.time() * 1000))}
signature = sign_request(body, os.environ['ML_DSA_PRIVATE_KEY'])

response = requests.post(
    f"{API_BASE}/crypto/sign",
    headers={
        'Content-Type':  'application/json',
        'x-pqc-key-id':  PQC_KEY_ID,
        'x-auth-key-id': AUTH_KEY_ID,
        'x-signature':   signature
    },
    json=body
)
print(response.json())
```

**cURL (with pre-computed signature):**

```bash
TIMESTAMP=$(date +%s%3N)
BODY="{\"data\":\"Hello World\",\"timestamp\":\"$TIMESTAMP\"}"
# (Compute SIGNATURE externally — see signing scripts)

curl -X POST https://your-api-base/api/crypto/sign \
  -H "Content-Type: application/json" \
  -H "x-pqc-key-id: $PQC_KEY_ID" \
  -H "x-auth-key-id: $AUTH_KEY_ID" \
  -H "x-signature: $SIGNATURE" \
  -d "$BODY"
```

---

### Signing with ECDSA

**Private key format:** PKCS#8 PEM-encoded ECDSA P-256 private key.

**JavaScript:**

```javascript
async function signWithECDSA(body, pemPrivateKey) {
  const clean = pemPrivateKey.replace(/-----[^-]+-----/g, '').replace(/\s+/g, '');
  const keyData = Uint8Array.from(atob(clean), c => c.charCodeAt(0)).buffer;

  const key = await crypto.subtle.importKey(
    'pkcs8', keyData,
    { name: 'ECDSA', namedCurve: 'P-256' },
    false, ['sign']
  );

  const canonical = canonicalizeJSON(body);
  const bodyBytes = new TextEncoder().encode(canonical);
  const sigBuf = await crypto.subtle.sign({ name: 'ECDSA', hash: 'SHA-256' }, key, bodyBytes);
  return toHex(sigBuf);
}
```

**Python:**

```python
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import ec

def sign_with_ecdsa(body: dict, pem_private_key: str) -> str:
    canonical = canonicalize(body)
    body_bytes = canonical.encode('utf-8')

    private_key = serialization.load_pem_private_key(
        pem_private_key.encode(), password=None
    )
    signature = private_key.sign(body_bytes, ec.ECDSA(hashes.SHA256()))
    return signature.hex()
```

---

### Signing with RSA

**Private key format:** PKCS#8 PEM-encoded RSA-2048 private key.

**JavaScript:**

```javascript
async function signWithRSA(body, pemPrivateKey) {
  const clean = pemPrivateKey.replace(/-----[^-]+-----/g, '').replace(/\s+/g, '');
  const keyData = Uint8Array.from(atob(clean), c => c.charCodeAt(0)).buffer;

  const key = await crypto.subtle.importKey(
    'pkcs8', keyData,
    { name: 'RSASSA-PKCS1-v1_5', hash: 'SHA-256' },
    false, ['sign']
  );

  const canonical = canonicalizeJSON(body);
  const bodyBytes = new TextEncoder().encode(canonical);
  const sigBuf = await crypto.subtle.sign({ name: 'RSASSA-PKCS1-v1_5' }, key, bodyBytes);
  return toHex(sigBuf);
}
```

**Python:**

```python
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

def sign_with_rsa(body: dict, pem_private_key: str) -> str:
    canonical = canonicalize(body)
    body_bytes = canonical.encode('utf-8')

    private_key = serialization.load_pem_private_key(
        pem_private_key.encode(), password=None
    )
    signature = private_key.sign(body_bytes, padding.PKCS1v15(), hashes.SHA256())
    return signature.hex()
```

---

### Signing with Hybrid (ML-DSA + ECDSA)

Hybrid mode produces **two signatures** — one classical (ECDSA) and one post-quantum (ML-DSA). The server verifies both. The `x-signature` header carries a JSON object.

**Private key input:** ECDSA PEM key and ML-DSA hex key, both available from the key creation step.

**JavaScript:**

```javascript
async function signWithHybrid(body, ecdsaPem, mlDsaHex) {
  const canonical = canonicalizeJSON(body);
  const bodyBytes = new TextEncoder().encode(canonical);

  // Classical signature (ECDSA)
  const ecKey = await importClassicalKey(ecdsaPem, 'ECDSA');
  const classicalSigBuf = await crypto.subtle.sign(
    { name: 'ECDSA', hash: 'SHA-256' }, ecKey, bodyBytes
  );
  const classicalSig = toHex(classicalSigBuf);

  // PQC signature (ML-DSA)
  const secretKey = parseMlDsaKey(mlDsaHex);
  const pqcSig = toHex(ml_dsa65.sign(bodyBytes, secretKey));

  // Return combined signature as JSON string for header
  return JSON.stringify({ classical: classicalSig, pqc: pqcSig });
}

// x-signature header value will be:
// {"classical":"3045...","pqc":"c2de..."}
```

**Python:**

```python
import json

def sign_with_hybrid(body: dict, ecdsa_pem: str, ml_dsa_hex: str) -> str:
    canonical = canonicalize(body)
    body_bytes = canonical.encode('utf-8')

    # Classical (ECDSA)
    ecdsa_key = serialization.load_pem_private_key(ecdsa_pem.encode(), password=None)
    classical_sig = ecdsa_key.sign(body_bytes, ec.ECDSA(hashes.SHA256())).hex()

    # Post-Quantum (ML-DSA)
    secret_key = bytes.fromhex(ml_dsa_hex.strip())
    if len(secret_key) > 4032:
        secret_key = secret_key[-4032:]
    pqc_sig = ml_dsa_65.sign(body_bytes, secret_key).hex()

    return json.dumps({'classical': classical_sig, 'pqc': pqc_sig})
```

---

## Cryptographic Operations

### Sign Data

Generate a digital signature over arbitrary data using a vault-managed PQC or classical signing key.

#### Endpoint

```
POST /api/crypto/sign
```

#### Request

```json
{
  "data": "The document content or hash to sign",
  "timestamp": "1749112345678"
}
```

#### Headers

```
Content-Type:  application/json
x-pqc-key-id:  <your-pqc-key-id>
x-auth-key-id: <your-auth-key-id>
x-signature:   <hex-signature-of-canonicalized-body>
```

#### Response

```json
{
  "signature": "c2de4f8a1b3d..."
}
```

For **Hybrid-DSA** keys the response is an object:

```json
{
  "signature": {
    "pqc":       "c2de4f8a...",
    "classical": "3045022100..."
  }
}
```

#### Full Example — JavaScript

```javascript
const body = {
  data: 'Invoice #1042 — Amount: $4,200 — Due: 2025-07-01',
  timestamp: Date.now().toString()
};

const signature = await signRequest(body, process.env.AUTH_PRIVATE_KEY);

const res = await fetch(`${API_BASE}/crypto/sign`, {
  method: 'POST',
  headers: {
    'Content-Type':  'application/json',
    'x-pqc-key-id':  PQC_KEY_ID,
    'x-auth-key-id': AUTH_KEY_ID,
    'x-signature':   signature
  },
  body: JSON.stringify(body)
});

const { signature: pqcSignature } = await res.json();
// Store pqcSignature alongside the document for later verification
```

#### Full Example — Python

```python
import time, requests

body = {
    'data': 'Invoice #1042 — Amount: $4,200 — Due: 2025-07-01',
    'timestamp': str(int(time.time() * 1000))
}
x_sig = sign_request(body, os.environ['AUTH_PRIVATE_KEY'])

response = requests.post(
    f"{API_BASE}/crypto/sign",
    headers={
        'Content-Type':  'application/json',
        'x-pqc-key-id':  PQC_KEY_ID,
        'x-auth-key-id': AUTH_KEY_ID,
        'x-signature':   x_sig
    },
    json=body
)
pqc_signature = response.json()['signature']
```

---

### Verify Signature

Verify a signature that was previously produced by the Sign endpoint using the same PQC key.

#### Endpoint

```
POST /api/crypto/verify
```

#### Request

```json
{
  "data": "The document content or hash to sign",
  "signature": "c2de4f8a1b3d...",
  "timestamp": "1749112345678"
}
```

For Hybrid signatures, pass the object:

```json
{
  "data": "The document content or hash to sign",
  "signature": { "pqc": "c2de...", "classical": "3045..." },
  "timestamp": "1749112345678"
}
```

#### Response

```json
{
  "isValid": true
}
```

#### Full Example — JavaScript

```javascript
const body = {
  data: 'Invoice #1042 — Amount: $4,200 — Due: 2025-07-01',
  signature: pqcSignatureFromStorage,
  timestamp: Date.now().toString()
};

const signature = await signRequest(body, process.env.AUTH_PRIVATE_KEY);

const res = await fetch(`${API_BASE}/crypto/verify`, {
  method: 'POST',
  headers: {
    'Content-Type':  'application/json',
    'x-pqc-key-id':  PQC_KEY_ID,
    'x-auth-key-id': AUTH_KEY_ID,
    'x-signature':   signature
  },
  body: JSON.stringify(body)
});

const { isValid } = await res.json();
console.log(isValid ? 'Document is authentic' : 'Signature invalid — document may be tampered');
```

---

### Encapsulate — KEM

Generate a shared secret and its corresponding ciphertext using a vault-managed ML-KEM or Hybrid-KEM key. The shared secret is returned to your application; the ciphertext is sent to the receiving party who will decapsulate it.

#### Endpoint

```
POST /api/crypto/encapsulate
```

#### Request

```json
{
  "timestamp": "1749112345678"
}
```

No `data` field is required — the operation is self-contained.

#### Response — ML-KEM

```json
{
  "ciphertext":   "4a7f23bc...",
  "sharedSecret": "e3d1cc9a..."
}
```

#### Response — Hybrid-KEM

```json
{
  "ciphertext": {
    "pqc":       "4a7f23bc...",
    "classical": "9f24ab..."
  },
  "sharedSecret": {
    "pqc":       "e3d1cc9a...",
    "classical": "a12f..."
  }
}
```

#### Full Example — Python

```python
body = {'timestamp': str(int(time.time() * 1000))}
x_sig = sign_request(body, os.environ['AUTH_PRIVATE_KEY'])

response = requests.post(
    f"{API_BASE}/crypto/encapsulate",
    headers={
        'Content-Type':  'application/json',
        'x-pqc-key-id':  PQC_KEY_ID,
        'x-auth-key-id': AUTH_KEY_ID,
        'x-signature':   x_sig
    },
    json=body
)
data = response.json()
shared_secret  = data['sharedSecret']   # Use to derive session key
ciphertext     = data['ciphertext']     # Send to the other party
```

---

### Decapsulate — KEM

Recover a shared secret from a ciphertext produced by the Encapsulate endpoint. Called by the receiving party.

#### Endpoint

```
POST /api/crypto/decapsulate
```

#### Request

```json
{
  "ciphertext": "4a7f23bc...",
  "timestamp":  "1749112345678"
}
```

For Hybrid-KEM:

```json
{
  "ciphertext": { "pqc": "4a7f23bc...", "classical": "9f24ab..." },
  "timestamp":  "1749112345678"
}
```

#### Response

```json
{
  "sharedSecret": "e3d1cc9a..."
}
```

---

### Encrypt — AES-GCM

Encrypt plaintext data using a vault-managed AES-256-GCM symmetric key. Returns the ciphertext, initialization vector (IV), and authentication tag — all required for decryption.

#### Endpoint

```
POST /api/crypto/encrypt
```

#### Request

```json
{
  "data":      "Sensitive customer record: John Doe, DOB 1985-04-12",
  "timestamp": "1749112345678"
}
```

#### Response

```json
{
  "ciphertext": "8f3a1d...",
  "iv":         "a1b2c3d4e5f6",
  "authTag":    "9e8d7c6b..."
}
```

> Store `ciphertext`, `iv`, and `authTag` together. All three are required for decryption.

#### Full Example — JavaScript

```javascript
const body = {
  data:      'Sensitive customer record: John Doe, DOB 1985-04-12',
  timestamp: Date.now().toString()
};
const signature = await signRequest(body, process.env.AUTH_PRIVATE_KEY);

const res = await fetch(`${API_BASE}/crypto/encrypt`, {
  method: 'POST',
  headers: {
    'Content-Type':  'application/json',
    'x-pqc-key-id':  PQC_KEY_ID,
    'x-auth-key-id': AUTH_KEY_ID,
    'x-signature':   signature
  },
  body: JSON.stringify(body)
});

const { ciphertext, iv, authTag } = await res.json();
// Persist { ciphertext, iv, authTag } — all needed to decrypt later
```

---

### Decrypt — AES-GCM

Decrypt ciphertext produced by the Encrypt endpoint using the same vault-managed AES-256 key.

#### Endpoint

```
POST /api/crypto/decrypt
```

#### Request

```json
{
  "ciphertext": "8f3a1d...",
  "iv":         "a1b2c3d4e5f6",
  "authTag":    "9e8d7c6b...",
  "timestamp":  "1749112345678"
}
```

#### Response

```json
{
  "data": "Sensitive customer record: John Doe, DOB 1985-04-12"
}
```

#### Full Example — Python

```python
body = {
    'ciphertext': stored['ciphertext'],
    'iv':         stored['iv'],
    'authTag':    stored['authTag'],
    'timestamp':  str(int(time.time() * 1000))
}
x_sig = sign_request(body, os.environ['AUTH_PRIVATE_KEY'])

response = requests.post(
    f"{API_BASE}/crypto/decrypt",
    headers={
        'Content-Type':  'application/json',
        'x-pqc-key-id':  PQC_KEY_ID,
        'x-auth-key-id': AUTH_KEY_ID,
        'x-signature':   x_sig
    },
    json=body
)
plaintext = response.json()['data']
```

---

## Operation Chaining

Many integration patterns require chaining two operations together — the output of one becomes the input of the next. The PQC Tester tool supports this interactively via the **"Use →"** chain button.

### Sign → Verify

```
1. Call /crypto/sign  with { data, timestamp }
   → Receive: { signature }

2. Call /crypto/verify with { data, signature, timestamp }
   → Receive: { isValid: true }
```

The `data` value in step 2 must match step 1 exactly.

### Encapsulate → Decapsulate

```
1. Call /crypto/encapsulate with { timestamp }
   → Receive: { ciphertext, sharedSecret }
   → Send ciphertext to receiving party
   → Use sharedSecret locally (e.g. derive an AES session key)

2. Receiving party calls /crypto/decapsulate with { ciphertext, timestamp }
   → Receive: { sharedSecret }
   → Both parties now share the same secret without transmitting it
```

### Encrypt → Decrypt

```
1. Call /crypto/encrypt with { data, timestamp }
   → Receive: { ciphertext, iv, authTag }
   → Store or transmit { ciphertext, iv, authTag }

2. Call /crypto/decrypt with { ciphertext, iv, authTag, timestamp }
   → Receive: { data }
```

---

## Real-World Use Cases

### Quantum-Safe Document Signing

**Problem:** Legal documents, contracts, and audit records need long-term verifiable signatures. RSA and ECDSA signatures will be breakable by quantum computers within the lifetime of these documents.

**Solution:** Use QuantumVault's ML-DSA signing key to produce quantum-resistant signatures over document hashes.

```
Architecture:
┌──────────────┐    hash(doc)     ┌─────────────────┐    signature    ┌──────────────┐
│  Your App    │ ───────────────▶ │  QuantumVault   │ ─────────────▶ │  Document DB │
│              │                  │  /crypto/sign   │                 │  + signature │
└──────────────┘                  └─────────────────┘                 └──────────────┘

Verification:
┌──────────────┐  hash + sig     ┌─────────────────┐   { isValid }  ┌──────────────┐
│  Your App    │ ───────────────▶ │  QuantumVault   │ ─────────────▶ │  Return to   │
│              │                  │  /crypto/verify │                 │  caller      │
└──────────────┘                  └─────────────────┘                 └──────────────┘
```

**Implementation sketch — JavaScript:**

```javascript
const crypto = require('crypto');

async function signDocument(documentBuffer) {
  // Hash the document first — sign the hash, not the whole file
  const hash = crypto.createHash('sha256').update(documentBuffer).digest('hex');

  const body = { data: hash, timestamp: Date.now().toString() };
  const requestSig = await signRequest(body, AUTH_PRIVATE_KEY);

  const res = await fetch(`${API_BASE}/crypto/sign`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-pqc-key-id': PQC_KEY_ID,
      'x-auth-key-id': AUTH_KEY_ID,
      'x-signature': requestSig
    },
    body: JSON.stringify(body)
  });

  const { signature } = await res.json();
  return { documentHash: hash, pqcSignature: signature };
}

async function verifyDocument(documentBuffer, storedHash, storedSignature) {
  const hash = crypto.createHash('sha256').update(documentBuffer).digest('hex');
  if (hash !== storedHash) return { isValid: false, reason: 'Document hash mismatch' };

  const body = { data: hash, signature: storedSignature, timestamp: Date.now().toString() };
  const requestSig = await signRequest(body, AUTH_PRIVATE_KEY);

  const res = await fetch(`${API_BASE}/crypto/verify`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-pqc-key-id': PQC_KEY_ID,
      'x-auth-key-id': AUTH_KEY_ID,
      'x-signature': requestSig
    },
    body: JSON.stringify(body)
  });

  return await res.json(); // { isValid: true/false }
}
```

---

### Secure API Request Authentication

**Problem:** Service-to-service API calls need to be authenticated in a way that is tamper-proof and quantum-safe.

**Solution:** Each service holds an Authentication Key private key. Every outbound API call is signed and verified by QuantumVault.

```
Service A                QuantumVault              Service B
─────────                ────────────              ─────────
Build request body
Sign body (ML-DSA)
  ─── POST /crypto/sign ──▶
                        Verify Auth Key
                        Sign with PQC Key
  ◀── { signature } ────
Attach signature to
outbound call ──────────────────────────────────▶ Verify signature
                                                  (via /crypto/verify)
```

---

### Post-Quantum JWT Replacement

**Problem:** JWTs signed with RS256 or ES256 will be forgeable by quantum adversaries.

**Solution:** Issue tokens with ML-DSA signatures using QuantumVault. The token structure remains JSON-based but the signature is quantum-resistant.

```javascript
async function issueToken(payload) {
  const header  = { alg: 'ML-DSA-65', typ: 'QVT' };
  const content = Buffer.from(JSON.stringify(header)).toString('base64url') + '.' +
                  Buffer.from(JSON.stringify(payload)).toString('base64url');

  const body = { data: content, timestamp: Date.now().toString() };
  const requestSig = await signRequest(body, AUTH_PRIVATE_KEY);

  const res = await fetch(`${API_BASE}/crypto/sign`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-pqc-key-id':  PQC_KEY_ID,
      'x-auth-key-id': AUTH_KEY_ID,
      'x-signature':   requestSig
    },
    body: JSON.stringify(body)
  });

  const { signature } = await res.json();
  return content + '.' + signature;   // header.payload.pqcsignature
}
```

---

### Encrypted Data Storage

**Problem:** Sensitive records (PII, health data, financial data) stored in a database need encryption with a key that is centrally managed, audited, and never embedded in application code.

**Solution:** Use QuantumVault's AES-256 key to encrypt records on write and decrypt on read. The key never leaves the vault.

```python
import psycopg2

def store_patient_record(conn, patient_id, record_data):
    # Encrypt via QuantumVault
    body = {'data': record_data, 'timestamp': str(int(time.time() * 1000))}
    x_sig = sign_request(body, AUTH_PRIVATE_KEY)

    response = requests.post(f"{API_BASE}/crypto/encrypt",
        headers={
            'Content-Type': 'application/json',
            'x-pqc-key-id':  AES_PQC_KEY_ID,
            'x-auth-key-id': AUTH_KEY_ID,
            'x-signature':   x_sig
        },
        json=body)

    enc = response.json()

    # Store ciphertext + iv + authTag
    with conn.cursor() as cur:
        cur.execute(
            "INSERT INTO patients (id, ciphertext, iv, auth_tag) VALUES (%s, %s, %s, %s)",
            (patient_id, enc['ciphertext'], enc['iv'], enc['authTag'])
        )
    conn.commit()


def retrieve_patient_record(conn, patient_id):
    with conn.cursor() as cur:
        cur.execute("SELECT ciphertext, iv, auth_tag FROM patients WHERE id = %s", (patient_id,))
        row = cur.fetchone()

    body = {
        'ciphertext': row[0], 'iv': row[1], 'authTag': row[2],
        'timestamp': str(int(time.time() * 1000))
    }
    x_sig = sign_request(body, AUTH_PRIVATE_KEY)

    response = requests.post(f"{API_BASE}/crypto/decrypt",
        headers={
            'Content-Type': 'application/json',
            'x-pqc-key-id':  AES_PQC_KEY_ID,
            'x-auth-key-id': AUTH_KEY_ID,
            'x-signature':   x_sig
        },
        json=body)

    return response.json()['data']
```

---

### Secure Key Exchange Between Services

**Problem:** Two microservices need to establish a shared session key to encrypt their communications without transmitting the key itself.

**Solution:** Use ML-KEM encapsulation. Service A generates the shared secret and ciphertext; Service B decapsulates to recover the same secret. Neither service transmits the actual key.

```
Service A                                         Service B
─────────                                         ─────────
POST /crypto/encapsulate
→ { ciphertext, sharedSecret }
                          ── send ciphertext ──▶
use sharedSecret                                  POST /crypto/decapsulate
as session key                                    with { ciphertext }
                                                  → { sharedSecret }
                                                  use sharedSecret
                                                  as session key

Both sides now share the same secret.
The key was never transmitted over the network.
```

---

### File Integrity Verification

**Problem:** Files uploaded to a storage system need integrity guarantees — you need to detect if a file has been modified after upload.

**Solution:** Sign the file hash on upload; verify on download.

```python
import hashlib

def upload_file(file_path):
    with open(file_path, 'rb') as f:
        file_hash = hashlib.sha256(f.read()).hexdigest()

    body = {'data': file_hash, 'timestamp': str(int(time.time() * 1000))}
    x_sig = sign_request(body, AUTH_PRIVATE_KEY)

    res = requests.post(f"{API_BASE}/crypto/sign",
        headers={
            'Content-Type':  'application/json',
            'x-pqc-key-id':  PQC_KEY_ID,
            'x-auth-key-id': AUTH_KEY_ID,
            'x-signature':   x_sig
        }, json=body)

    pqc_signature = res.json()['signature']
    # Store file_hash + pqc_signature in metadata DB
    return {'file_hash': file_hash, 'pqc_signature': pqc_signature}


def verify_file(file_path, stored_hash, stored_signature):
    with open(file_path, 'rb') as f:
        current_hash = hashlib.sha256(f.read()).hexdigest()

    if current_hash != stored_hash:
        return {'isValid': False, 'reason': 'File has been modified'}

    body = {'data': current_hash, 'signature': stored_signature,
            'timestamp': str(int(time.time() * 1000))}
    x_sig = sign_request(body, AUTH_PRIVATE_KEY)

    res = requests.post(f"{API_BASE}/crypto/verify",
        headers={
            'Content-Type':  'application/json',
            'x-pqc-key-id':  PQC_KEY_ID,
            'x-auth-key-id': AUTH_KEY_ID,
            'x-signature':   x_sig
        }, json=body)

    return res.json()
```

---

## Security Controls

### IP Whitelisting

Restrict which IP addresses can make calls to the crypto endpoints. Configured per account in **Settings → IP Restriction**.

| Mode       | Behaviour                                             |
|------------|-------------------------------------------------------|
| `any`      | All IPs are allowed (default)                         |
| `selected` | Only IPs in the whitelist can execute crypto operations |

Supports individual IPv4, CIDR ranges, and IPv6 addresses.

> If `selected` mode is active and the whitelist is empty, all cryptographic operations will be blocked.

**Recommendation:** Set mode to `selected` and add only your application server's IP addresses or CIDR block. This eliminates the entire class of credential-theft attacks where a leaked key ID still cannot be used from an unauthorised network.

### Mutual TLS (mTLS)

Add certificate-based client authentication on top of JWT authentication.

| Mode       | Behaviour                                                    |
|------------|--------------------------------------------------------------|
| `standard` | JWT + request signature only                                 |
| `mtls`     | JWT + request signature + valid client certificate required  |

When `mtls` mode is enabled, each request must also present a valid client certificate issued by QuantumVault's CA. Certificates can be issued, downloaded, and revoked from **Settings → Mutual TLS**.

**Certificate workflow:**

```
1. Generate a CSR (Certificate Signing Request) on your server.
2. Upload CSR to QuantumVault → receive client.crt, ca.crt, server.crt.
3. Configure your HTTP client to present client.crt on every request.
4. QuantumVault validates the certificate before processing the request.
```

### Rate Limiting

Per-policy rate limits prevent accidental or malicious overuse of cryptographic operations.

Configured in each Access Policy as operations per hour (e.g. `1000`). When the limit is exceeded, the API returns `429 Too Many Requests`.

**Recommendation:** Set conservative rate limits in production and monitor via audit logs for unexpected spikes.

### Audit Logging

Every cryptographic operation generates an immutable audit log entry containing:

| Field      | Description                                    |
|------------|------------------------------------------------|
| operation  | Operation type (`pqc_sign`, `pqc_verify`, etc.) |
| authKey    | Authentication Key name                        |
| pqcKey     | PQC Key name                                   |
| sourceIP   | Originating IP address                         |
| result     | `success` or `failure`                         |
| timestamp  | ISO-8601 timestamp                             |

Logs are accessible via the Dashboard and Audit Logs section of the QuantumVault interface, and via the API at `GET /api/audit-logs`.

---

## Migration from Classical Cryptography

### From RSA Signing to ML-DSA

```
Before:
  Application holds RSA private key locally
  Calls crypto.sign(data, rsaPrivateKey) inline

After:
  Application holds Auth private key (RSA, ECDSA, or ML-DSA) for request signing only
  Calls QuantumVault POST /crypto/sign with data
  QuantumVault signs with ML-DSA vault key
  Application receives quantum-safe signature
```

**Code change — before:**

```python
# Before: local RSA signing
from cryptography.hazmat.primitives.asymmetric import padding
signature = rsa_key.sign(data, padding.PSS(...), hashes.SHA256())
```

**Code change — after:**

```python
# After: QuantumVault ML-DSA signing
body = {'data': data.decode(), 'timestamp': str(int(time.time() * 1000))}
x_sig = sign_request(body, AUTH_PRIVATE_KEY)
res = requests.post(f"{API_BASE}/crypto/sign", headers={...}, json=body)
signature = res.json()['signature']
```

**What changed for your verifiers:** Verifiers must also use QuantumVault's `/crypto/verify` instead of calling a local RSA verify. If external parties need to verify your signatures, they need access to the QuantumVault verify endpoint (or you expose a proxy endpoint that calls it).

---

### From ECDH Key Exchange to ML-KEM

```
Before:
  Service A generates ECDH keypair, sends public key to Service B
  Service B computes shared secret using A's public key + B's private key
  Service A computes shared secret using B's public key + A's private key

After (simpler):
  Service A calls QuantumVault /crypto/encapsulate
  QuantumVault returns { ciphertext, sharedSecret }
  Service A uses sharedSecret locally; sends ciphertext to Service B
  Service B calls QuantumVault /crypto/decapsulate with ciphertext
  Service B receives sharedSecret
```

The QuantumVault KEM model eliminates the need for each service to manage their own KEM keypairs. Both parties use the same vault-managed ML-KEM key and the ciphertext acts as the transport mechanism.

---

### From AES with Local Key Management to QuantumVault AES-256

```
Before:
  AES key hardcoded or in environment variable
  Application calls aes.encrypt(key, data) locally

After:
  AES key managed in QuantumVault — never exposed
  Application calls POST /crypto/encrypt — receives { ciphertext, iv, authTag }
  Application calls POST /crypto/decrypt — sends { ciphertext, iv, authTag }
```

**Benefits of vault-managed AES:**

- Key rotation without re-encrypting data (old ciphertext still decryptable with rotated key via policy migration).
- Centralised access control — different applications can be granted encrypt-only or decrypt-only operations via policies.
- Full audit trail of every encryption and decryption operation.
- Key material never touches application memory.

---

### Hybrid Migration Strategy

If you cannot switch everything to post-quantum algorithms immediately, use **Hybrid mode** as a bridge:

```
Phase 1 (Now):
  Deploy Hybrid-DSA authentication keys and Hybrid-DSA PQC keys.
  All signatures contain both ECDSA and ML-DSA components.
  Classical verifiers still work (ignoring the ML-DSA component).
  PQC verifiers benefit from full quantum resistance.

Phase 2 (Transition):
  Migrate verifiers to use the PQC component.
  Keep classical as fallback for compatibility.

Phase 3 (Full PQC):
  Switch to ML-DSA-only keys.
  Retire classical components.
```

Hybrid mode is specifically designed for this transition period — it provides quantum safety without breaking existing integrations that expect classical signatures.

---

## Testing Your Integration

### PQC Tester Tool

QuantumVault provides a browser-based testing utility at:

```
https://katoki-dev.github.io/PQC-Tester/
```

The tester performs the full request signing workflow in-browser — canonicalization, ML-DSA/ECDSA/RSA signing using WebCrypto and `@noble/post-quantum`, and the signed API call — without sending your private key to any server.

**What you can test:**

- All six operations: Sign, Verify, Encapsulate, Decapsulate, Encrypt, Decrypt.
- All four authentication protocols: ML-DSA, ECDSA, RSA, Hybrid.
- Operation chaining: Sign → Verify, Encapsulate → Decapsulate, Encrypt → Decrypt.
- Live API calls against your own deployment (configurable base URL).

### Tester Workflow

**1. Configure the base URL**

Enter your API base URL in the top bar:

```
http://localhost:5000/api/crypto    (local development)
https://your-deployment/api/crypto  (production)
```

**2. Select the operation**

Choose from the Crypto Operation dropdown:

| Option              | Endpoint called            |
|---------------------|----------------------------|
| Sign Data           | `POST /crypto/sign`        |
| Verify Signature    | `POST /crypto/verify`      |
| Encapsulate (KEM)   | `POST /crypto/encapsulate` |
| Decapsulate (KEM)   | `POST /crypto/decapsulate` |
| Encrypt (AES-GCM)   | `POST /crypto/encrypt`     |
| Decrypt (AES-GCM)   | `POST /crypto/decrypt`     |

**3. Enter your Key IDs**

Paste your PQC Key ID into **PQC Key ID** and your Auth Key ID into **Auth Key ID**. These populate the `x-pqc-key-id` and `x-auth-key-id` headers.

**4. Select the signing protocol**

Click the appropriate pill: `ML-DSA (PQC)`, `ECDSA`, `RSA`, or `Hybrid (PQC+Classical)`. Each protocol expects a different private key format:

| Protocol | Private Key Input                                           |
|----------|-------------------------------------------------------------|
| ML-DSA   | 4032-byte hex string or PEM-wrapped secret key              |
| ECDSA    | PKCS#8 PEM-encoded P-256 private key                        |
| RSA      | PKCS#8 PEM-encoded RSA-2048 private key                     |
| Hybrid   | ECDSA PEM key + ML-DSA hex key (on separate lines or space-separated) |

**5. Paste your Auth Private Key**

Paste the private key you downloaded when creating your Authentication Key. This is used locally in-browser to sign the request — it is never transmitted.

**6. Fill operation-specific fields**

| Operation   | Fields required                              |
|-------------|----------------------------------------------|
| Sign        | Data to Sign                                 |
| Verify      | Original Data + Signature                    |
| Encapsulate | (none)                                       |
| Decapsulate | Ciphertext                                   |
| Encrypt     | Plaintext Data                               |
| Decrypt     | Ciphertext + IV + Auth Tag                   |

**7. Execute and inspect the output**

Click **🚀 Execute** (or press `Ctrl+Enter`). The right panel shows:

- **API Response** — full JSON response from the server.
- **Request Headers** — the headers sent, including the truncated signature.
- **Signed Request Body** — the canonicalized JSON that was signed.
- **HTTP status** and **latency** in milliseconds.

**8. Chain operations**

After a successful Sign, Encapsulate, or Encrypt — a **"Use →"** banner appears. Click it to automatically populate the next chained operation's input fields with the result. This lets you test a full Sign → Verify or Encrypt → Decrypt round-trip without copying values manually.

---

## Error Reference

| HTTP Status | Error                          | Meaning                                                      |
|-------------|--------------------------------|--------------------------------------------------------------|
| 400         | Missing parameters             | Required field (`data`, `signature`, `ciphertext`, etc.) missing from request body |
| 401         | Unauthorized                   | JWT token missing or expired                                 |
| 403         | Policy restriction             | No active policy for the given Auth Key + PQC Key combination |
| 403         | IP whitelist violation         | Request originated from an IP not on the whitelist           |
| 403         | MTLS requirement failed        | mTLS mode is enabled but no valid client certificate was presented |
| 403         | Operation not permitted        | The policy does not grant the requested operation            |
| 403         | Key is not active              | PQC Key or Auth Key is disabled or rotated                   |
| 429         | Rate limit exceeded            | Policy rate limit has been reached                           |
| 500         | Signature generation failed    | Internal error during cryptographic operation                |

**Signature verification failure** (the `x-signature` header is invalid or the body was tampered with) returns `403` with an error message indicating the signature did not verify.

---

## Security Best Practices

**Private key handling:**

- Store Auth Private Keys in a secret manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) — never in source code or environment files committed to version control.
- Rotate Authentication Keys periodically and immediately if a key is suspected to be compromised.
- For production deployments, prefer ML-DSA auth keys — they protect against future quantum compromise of the key itself.

**Request integrity:**

- Always include a fresh `timestamp` (current time in milliseconds) in every request body. This prevents replay attacks.
- Do not cache or reuse signed request bodies across different API calls.
- Canonicalize the exact body you send — any difference between the signed payload and the transmitted payload will cause verification failure.

**Network controls:**

- Enable IP whitelisting to restrict crypto operations to known server IPs.
- Enable mTLS for highest-security deployments (financial, healthcare, government).
- Use HTTPS — never HTTP — for all API communication.

**Policy design:**

- Apply the principle of least privilege: each application should have a policy granting only the operations it actually needs.
- Use separate PQC keys for different environments (Development / Production) — do not share keys across environments.
- Set rate limits on all policies; review audit logs for anomalies.

**Key lifecycle:**

- Rotate PQC keys periodically. QuantumVault's rotation feature handles policy migration automatically.
- Use the Audit Logs to monitor for unexpected operations, off-hours activity, or IP addresses outside your expected ranges.
- Disable — do not delete — keys that may still have active ciphertexts that need decryption (the key must remain accessible to decrypt old data).
