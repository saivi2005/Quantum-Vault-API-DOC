# QuantumVault — Client-Side API Documentation

A complete reference for all API calls made by the QuantumVault frontend. Covers request construction, payload shapes, response handling, error behavior, and client-side data patterns used across the React application.

#### Table of Contents

- [Setup & Configuration](#setup--configuration)
  - [Environment Variables](#environment-variables)
  - [Base URL](#base-url)
  - [Token Storage](#token-storage)
  - [Request Headers](#request-headers)
  - [Response Handler](#response-handler)
- [1. Authentication](#1-authentication)
  - [Google OAuth Login](#google-oauth-login)
  - [Verify Two-Factor Authentication Login](#verify-two-factor-authentication-login)
  - [Get User Profile](#get-user-profile)
  - [Setup Two-Factor Authentication](#setup-two-factor-authentication)
  - [Verify Two-Factor Authentication Setup](#verify-two-factor-authentication-setup)
  - [Get Two-Factor Authentication Status](#get-two-factor-authentication-status)
  - [Disable Two-Factor Authentication](#disable-two-factor-authentication)
- [2. PQC Key Management](#2-pqc-key-management)
  - [Get PQC Keys](#get-pqc-keys)
  - [Create PQC Key](#create-pqc-key)
  - [Update PQC Key](#update-pqc-key)
  - [Rotate PQC Key](#rotate-pqc-key)
- [3. Authentication Key Management](#3-authentication-key-management)
  - [Get Authentication Keys](#get-authentication-keys)
  - [Create Authentication Key](#create-authentication-key)
  - [Update Authentication Key](#update-authentication-key)
- [4. Policy Management](#4-policy-management)
  - [Get Policies](#get-policies)
  - [Create Policy](#create-policy)
  - [Update Policy](#update-policy)
- [5. Audit Logs](#5-audit-logs)
  - [Get Audit Logs](#get-audit-logs)
  - [Client-Side Filtering](#client-side-filtering)
- [6. Dashboard](#6-dashboard)
  - [Get Dashboard Statistics](#get-dashboard-statistics)
- [7. IP Whitelist Settings](#7-ip-whitelist-settings)
  - [Get IP Whitelist](#get-ip-whitelist)
  - [Update IP Whitelist](#update-ip-whitelist)
- [8. MTLS Certificate Management](#8-mtls-certificate-management)
  - [Get MTLS Settings](#get-mtls-settings)
  - [Update MTLS Settings](#update-mtls-settings)
  - [Issue MTLS Certificate](#issue-mtls-certificate)
  - [Get Active MTLS Certificates](#get-active-mtls-certificates)
  - [Get Revoked MTLS Certificates](#get-revoked-mtls-certificates)
  - [Revoke MTLS Certificate](#revoke-mtls-certificate)
  - [Download MTLS Certificates](#download-mtls-certificates)
  - [Verify MTLS Certificate](#verify-mtls-certificate)
- [9. Contact / Support](#9-contact--support)
  - [Submit Contact Request](#submit-contact-request)
- [Client-Side State Management](#client-side-state-management)
  - [Local Storage Keys](#local-storage-keys)
  - [Session Timeout](#session-timeout)
  - [Data Caching Strategy](#data-caching-strategy)
- [Client-Side Validation Rules](#client-side-validation-rules)
- [Utility Helpers](#utility-helpers)
  - [Date Formatting](#date-formatting)
  - [Fingerprint Formatting](#fingerprint-formatting)
  - [Name Validation & Truncation](#name-validation--truncation)

---

## Setup & Configuration

### Environment Variables

The frontend is configured via a `.env` file. Copy `.env.example` and populate the values before running the app.

```
VITE_GOOGLE_CLIENT_ID=<your_google_oauth_client_id>
VITE_API_URL=http://localhost:5000/api
```

| Variable              | Description                                                  |
|-----------------------|--------------------------------------------------------------|
| `VITE_GOOGLE_CLIENT_ID` | Google OAuth 2.0 client ID for the authorization code flow |
| `VITE_API_URL`          | Full base URL of the backend API, including `/api`         |

> All Vite environment variables must be prefixed with `VITE_` to be accessible in the browser bundle.

---

### Base URL

```javascript
const API_URL = import.meta.env.VITE_API_URL;
// Example: "http://localhost:5000/api"
```

All API calls are constructed as `${API_URL}/<endpoint>` — for example:

```
http://localhost:5000/api/pqc-keys
http://localhost:5000/api/auth/google
```

The Contact API is the only exception — it strips `/api` from the base and calls a sibling endpoint:

```javascript
const url = API_URL.replace('/api', '') + '/requests';
// Example: "http://localhost:5000/requests"
```

---

### Token Storage

JWT tokens are stored and retrieved from `localStorage`:

```javascript
// Store on login
localStorage.setItem('jwt_token', token);

// Retrieve for authenticated requests
const token = localStorage.getItem('jwt_token');

// Clear on logout
localStorage.removeItem('jwt_token');
```

---

### Request Headers

Two header configurations are used throughout the application:

**Authenticated request (default):**

```javascript
{
  'Content-Type': 'application/json',
  'Authorization': 'Bearer <jwt_token>'
}
```

**Unauthenticated request (public endpoints only):**

```javascript
{
  'Content-Type': 'application/json'
}
```

Public endpoints that skip the Authorization header:

```
POST /api/auth/google
POST /api/auth/2fa/verify
```

---

### Response Handler

All API calls pass through a centralized `handleResponse` function that:

- Parses JSON from the response body.
- Returns an empty object `{}` for `204 No Content` responses.
- Throws an `Error` with the status code attached for non-`2xx` responses.
- Extracts error messages from two formats:

```javascript
// Simple string format (preferred)
{ "error": "Invalid token" }

// Legacy nested format
{ "error": { "message": "Invalid token" } }
```

**Error object shape thrown to the caller:**

```javascript
{
  message: "Human-readable error string",
  status: 401,       // HTTP status code
  data: { ... }      // Full response body (if available)
}
```

**Example usage in a component:**

```javascript
try {
  const result = await api.getPqcKeys();
} catch (err) {
  if (err.status === 401) {
    // Token expired — trigger logout
  }
  toast.error(err.message || 'Something went wrong');
}
```

---

## 1. Authentication

### Google OAuth Login

Exchanges a Google authorization code for a JWT token. Uses the `auth-code` OAuth flow — the Google SDK returns an authorization code, not an access token directly.

#### Function

```javascript
api.googleLogin(code)
```

#### Request

```
POST /api/auth/google
```

No `Authorization` header required.

#### Payload

```json
{
  "code": "4/0AX4XfWhExampleGoogleAuthorizationCode"
}
```

| Field | Type   | Description                             |
|-------|--------|-----------------------------------------|
| code  | String | Authorization code from Google OAuth SDK |

#### Success Response — No 2FA

```json
{
  "requires2FA": false,
  "token": "jwt_token",
  "user": {
    "name": "John Doe",
    "email": "john@example.com",
    "avatar": "https://example.com/avatar.jpg",
    "id": "109876543210"
  }
}
```

#### Success Response — 2FA Required

```json
{
  "requires2FA": true,
  "tempToken": "temporary_token",
  "user": {
    "name": "John Doe",
    "email": "john@example.com",
    "avatar": "https://example.com/avatar.jpg",
    "id": "109876543210"
  }
}
```

#### Client Handling

```javascript
const result = await api.googleLogin(codeResponse.code);

if (result.requires2FA) {
  // Gate the session — show 2FA verification screen
  initiateTwoFactorLogin(result.user, result.tempToken);
} else {
  // Finalize login
  login(result.user, result.token);
  navigate('/dashboard');
}
```

#### Error Responses

| Status Code               | Description                        |
|---------------------------|------------------------------------|
| 400 Bad Request           | Authorization code is missing      |
| 401 Unauthorized          | Invalid Google authorization code  |
| 500 Internal Server Error | Unexpected server error            |

---

### Verify Two-Factor Authentication Login

Submits the 6-digit TOTP code along with the temporary token issued during Google login when 2FA is enabled.

#### Function

```javascript
api.verify2FALogin(tempToken, code)
```

#### Request

```
POST /api/auth/2fa/verify
```

No `Authorization` header required.

#### Payload

```json
{
  "tempToken": "temporary_token",
  "code": "123456"
}
```

| Field     | Type   | Description                            |
|-----------|--------|----------------------------------------|
| tempToken | String | Temporary token from `googleLogin`     |
| code      | String | 6-digit TOTP code from authenticator  |

#### Success Response

```json
{
  "token": "jwt_token",
  "user": {
    "name": "John Doe",
    "email": "john@example.com",
    "avatar": "https://example.com/avatar.jpg",
    "id": "109876543210"
  }
}
```

#### Client Handling

```javascript
const result = await api.verify2FALogin(tempToken, code);
login(result.user, result.token);
```

#### Error Responses

| Status Code      | Description                   |
|------------------|-------------------------------|
| 401 Unauthorized | Invalid or expired TOTP code  |

---

### Get User Profile

Retrieves the authenticated user's profile. Called on app load to hydrate the user state.

#### Function

```javascript
api.getProfile()
```

#### Request

```
GET /api/auth/me
```

Requires `Authorization` header.

#### Success Response

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "avatar": "https://example.com/avatar.jpg",
  "id": "109876543210",
  "twoFactorEnabled": true
}
```

#### Error Responses

| Status Code      | Description                             |
|------------------|-----------------------------------------|
| 401 Unauthorized | Token is missing, expired, or invalid   |

---

### Setup Two-Factor Authentication

Generates a TOTP secret and QR code URI. The client uses the returned `qrData` to render a QR code image using the `qrcode` npm library.

#### Function

```javascript
api.setup2FA()
```

#### Request

```
POST /api/auth/2fa/setup
```

Requires `Authorization` header.

#### Success Response

```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "otpauthUri": "otpauth://totp/QuantumVault:john@example.com?secret=JBSWY3DPEHPK3PXP&issuer=QuantumVault",
  "qrData": "otpauth://totp/QuantumVault:john@example.com?secret=JBSWY3DPEHPK3PXP&issuer=QuantumVault"
}
```

#### Client Handling

```javascript
const result = await api.setup2FA();

// Render QR code from the otpauth URI
const qrUrl = await QRCode.toDataURL(result.qrData, {
  width: 200,
  margin: 2,
  color: { dark: '#000000', light: '#ffffff' }
});
setQrCodeUrl(qrUrl);
setSetupSecret(result.secret);
```

---

### Verify Two-Factor Authentication Setup

Confirms the TOTP secret by verifying a live code from the authenticator app, then enables 2FA.

#### Function

```javascript
api.verifySetup2FA(code, secret)
```

#### Request

```
POST /api/auth/2fa/verify-setup
```

Requires `Authorization` header.

#### Payload

```json
{
  "code": "123456",
  "secret": "JBSWY3DPEHPK3PXP"
}
```

| Field  | Type   | Description                               |
|--------|--------|-------------------------------------------|
| code   | String | 6-digit TOTP code from authenticator app  |
| secret | String | TOTP secret from `setup2FA` response      |

#### Client-Side Validation

```
- code must be exactly 6 digits
```

#### Success Response

```json
{
  "enabled": true
}
```

#### Error Responses

| Status Code      | Description                                               |
|------------------|-----------------------------------------------------------|
| 400 Bad Request  | Code is invalid or secret mismatch                        |
| 401 Unauthorized | Token expired                                             |

---

### Get Two-Factor Authentication Status

Checks whether 2FA is currently enabled for the authenticated user. Called on Settings page load.

#### Function

```javascript
api.get2FAStatus()
```

#### Request

```
GET /api/auth/2fa/status
```

Requires `Authorization` header.

#### Success Response

```json
{
  "enabled": true
}
```

---

### Disable Two-Factor Authentication

Disables 2FA for the authenticated user. The client shows a browser confirm dialog before calling this endpoint.

#### Function

```javascript
api.disable2FA()
```

#### Request

```
DELETE /api/auth/2fa
```

Requires `Authorization` header.

#### Success Response

```json
{
  "enabled": false
}
```

---

## 2. PQC Key Management

### Get PQC Keys

Fetches all PQC keys for the authenticated user. Called during the global data sync on login.

#### Function

```javascript
api.getPqcKeys()
```

#### Request

```
GET /api/pqc-keys
```

Requires `Authorization` header.

#### Success Response

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Production PQC Key",
    "algorithm": "ML-DSA",
    "parameters": "ML-DSA-65 (FIPS 204)",
    "operations": "Sign · Verify",
    "operationsObj": {
      "sign": true,
      "verify": true,
      "encrypt": false,
      "decrypt": false,
      "encapsulate": false,
      "decapsulate": false
    },
    "environment": "Production",
    "status": "active",
    "version": 1,
    "publicKeyClassical": null,
    "symmetricKey": null,
    "created": "2025-06-05T10:15:30.000Z",
    "createdAt": "2025-06-05T10:15:30.000Z",
    "updatedAt": "2025-06-05T10:15:30.000Z"
  }
]
```

#### Response Fields

| Field              | Type          | Description                                              |
|--------------------|---------------|----------------------------------------------------------|
| id                 | String (UUID) | Unique key identifier                                    |
| name               | String        | Key name                                                 |
| algorithm          | String        | Selected cryptographic algorithm                         |
| parameters         | String        | Algorithm parameter set label                            |
| operations         | String        | Human-readable operations string (e.g. `"Sign · Verify"`) |
| operationsObj      | Object        | Boolean map of each supported operation                  |
| environment        | String        | `Development` or `Production`                            |
| status             | String        | `active`, `rotated`, or `disabled`                       |
| version            | Integer       | Key version (increments on rotation)                     |
| publicKeyClassical | String / Null | Classical public key for Hybrid algorithms               |
| symmetricKey       | String / Null | Symmetric key value for AES-256                          |
| createdAt          | String        | ISO-8601 creation timestamp                              |
| updatedAt          | String        | ISO-8601 last updated timestamp                          |

---

### Create PQC Key

Creates a new PQC key. The client enforces a multi-step form (Name → Algorithm → Environment → Review) before calling this endpoint.

#### Function

```javascript
api.createPqcKey(keyData)
```

#### Request

```
POST /api/pqc-keys
```

Requires `Authorization` header.

#### Payload

```json
{
  "name": "Production PQC Key",
  "algorithm": "ML-DSA",
  "environment": "Production"
}
```

| Field       | Type   | Mandatory | Description                           |
|-------------|--------|-----------|---------------------------------------|
| name        | String | Required  | Key name (max 30 chars, alphanumeric with `_` or `-`) |
| algorithm   | String | Required  | One of the supported algorithms       |
| environment | String | Optional  | `Development` or `Production`         |

#### Supported Algorithms (from UI)

| Algorithm   | Operation Group             | Parameters                        |
|-------------|-----------------------------|-----------------------------------|
| ML-DSA      | Sign & Verify               | ML-DSA-65 (FIPS 204)             |
| ML-KEM      | Encapsulate & Decapsulate   | ML-KEM-768 (FIPS 203)            |
| ECDSA       | Sign & Verify               | ECDSA P-256 (NIST Curve)         |
| Hybrid-DSA  | Sign & Verify               | Hybrid (ML-DSA-65 + ECDSA P-256) |
| Hybrid-KEM  | Encapsulate & Decapsulate   | Hybrid (ML-KEM-768 + X25519)     |
| AES-256     | Encrypt & Decrypt           | AES-256-GCM Symmetric Key        |

#### Success Response

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Production PQC Key",
  "algorithm": "ML-DSA",
  "parameters": "ML-DSA-65 (FIPS 204)",
  "operations": "Sign · Verify",
  "environment": "Production",
  "status": "active",
  "version": 1
}
```

#### Client Handling

```javascript
const newKey = await api.createPqcKey(keyData);
setPqcKeys(prev => [newKey, ...prev]);
await refreshAuditLogs();
toast.success('PQC key created successfully');
```

#### Client-Side Validation (enforced in UI before API call)

| Rule             | Description                                              |
|------------------|----------------------------------------------------------|
| Name required    | Key name cannot be empty                                 |
| Name format      | Alphanumeric, `_`, `-` only; max 30 characters           |
| No duplicate name| A key with the same name (case-insensitive) must not already exist |

#### Error Responses

| Status Code               | Description                                    |
|---------------------------|------------------------------------------------|
| 400 Bad Request           | Invalid algorithm or duplicate active key name |
| 401 Unauthorized          | Invalid or missing authentication token        |
| 500 Internal Server Error | Failed to generate PQC key                    |

---

### Update PQC Key

Updates the name or status of an existing PQC key. After a successful update, policies are also refreshed to reflect any cascading name changes.

#### Function

```javascript
api.updatePqcKey(id, updateData)
```

#### Request

```
PUT /api/pqc-keys/{id}
```

Requires `Authorization` header.

#### Payload

```json
{
  "name": "Updated Production Key",
  "status": "disabled"
}
```

| Field  | Type   | Mandatory | Description                          |
|--------|--------|-----------|--------------------------------------|
| name   | String | Optional  | Updated key name                     |
| status | String | Optional  | `active`, `rotated`, or `disabled`   |

#### Success Response

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Updated Production Key",
  "algorithm": "ML-DSA",
  "status": "disabled",
  "version": 1
}
```

#### Client Handling

```javascript
const updated = await api.updatePqcKey(updatedKey.id, updatedKey);
setPqcKeys(prev => prev.map(k => k.id === updated.id ? updated : k));

// Refresh policies as key rename cascades
const freshPolicies = await api.getPolicies();
setPolicies(freshPolicies);
```

#### Error Responses

| Status Code               | Description                              |
|---------------------------|------------------------------------------|
| 400 Bad Request           | Invalid update parameters                |
| 401 Unauthorized          | Invalid or missing authentication token  |
| 404 Not Found             | PQC key not found                        |
| 500 Internal Server Error | Failed to update PQC key                 |

---

### Rotate PQC Key

Rotates an existing PQC key by generating a new key version. The old key is marked `rotated`. After rotation, both PQC keys and policies are fully refreshed from the backend.

#### Function

```javascript
api.rotatePqcKey(id)
```

#### Request

```
POST /api/pqc-keys/{id}/rotate
```

Requires `Authorization` header. No request body.

#### Success Response

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440001",
  "name": "Production PQC Key",
  "algorithm": "ML-DSA",
  "status": "active",
  "version": 2
}
```

#### Client Handling

```javascript
await api.rotatePqcKey(keyId);

// Full reload — old key is now 'rotated', new key is 'active'
const freshKeys = await api.getPqcKeys();
setPqcKeys(freshKeys);

// Policies are migrated to new key by the backend
const freshPolicies = await api.getPolicies();
setPolicies(freshPolicies);
```

#### Error Responses

| Status Code               | Description                              |
|---------------------------|------------------------------------------|
| 401 Unauthorized          | Invalid or missing authentication token  |
| 404 Not Found             | PQC key not found                        |
| 500 Internal Server Error | Failed to rotate PQC key                 |

---

## 3. Authentication Key Management

### Get Authentication Keys

Fetches all authentication keys for the authenticated user. Called during the global data sync on login.

#### Function

```javascript
api.getAuthKeys()
```

#### Request

```
GET /api/auth-keys
```

Requires `Authorization` header.

#### Success Response

```json
[
  {
    "id": "f8d9e1d0-7f4c-4f56-a2e4-b3e2d6f9c100",
    "name": "Primary Signing Key",
    "algorithm": "RSA",
    "fingerprint": "d3:c9:c3:f9:6c:8d:9d:5f ... 4e:8a:7f:00",
    "publicKey": "-----BEGIN PUBLIC KEY-----...",
    "publicKeyDsa": null,
    "status": "active",
    "created": "2025-06-05T10:15:30.000Z",
    "createdAt": "2025-06-05T10:15:30.000Z",
    "updatedAt": "2025-06-05T10:15:30.000Z"
  }
]
```

---

### Create Authentication Key

Creates a new authentication key. The client supports two key source modes — uploading an existing public key, or server-side key generation.

#### Function

```javascript
api.createAuthKey(keyData)
```

#### Request

```
POST /api/auth-keys
```

Requires `Authorization` header.

#### Payload — Upload Public Key (RSA / ECDSA)

```json
{
  "name": "Primary Signing Key",
  "algorithm": "RSA",
  "publicKey": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A...\n-----END PUBLIC KEY-----"
}
```

#### Payload — Upload Public Key (ML-DSA)

```json
{
  "name": "PQC Auth Key",
  "algorithm": "ML-DSA",
  "publicKey": "",
  "publicKeyDsa": "04a1b2c3d4e5f6... (Hexadecimal ML-DSA Public Key)"
}
```

#### Payload — Upload Public Key (Hybrid)

```json
{
  "name": "Hybrid Auth Key",
  "algorithm": "Hybrid",
  "publicKey": "-----BEGIN PUBLIC KEY-----\n(ECDSA classical component)\n-----END PUBLIC KEY-----",
  "publicKeyDsa": "04a1b2c3d4e5f6... (ML-DSA post-quantum component)"
}
```

#### Payload — Generate Key Pair (Server-Side)

```json
{
  "name": "Generated Key",
  "algorithm": "RSA"
}
```

> When `source` is `generate`, no public key fields are sent. The backend generates the keypair and returns the private key **once** in the response.

#### Payload Fields

| Field        | Type   | Mandatory       | Description                                              |
|--------------|--------|-----------------|----------------------------------------------------------|
| name         | String | Required        | Key name (max 30 chars, alphanumeric with `_` or `-`)    |
| algorithm    | String | Required        | `RSA`, `ECDSA`, `ML-DSA`, or `Hybrid`                    |
| publicKey    | String | Conditional     | Classical public key (PEM for RSA/ECDSA; empty for ML-DSA upload) |
| publicKeyDsa | String | Conditional     | PQC public key in hexadecimal (ML-DSA or Hybrid only)    |

#### Success Response — Upload

```json
{
  "id": "f8d9e1d0-7f4c-4f56-a2e4-b3e2d6f9c100",
  "name": "Primary Signing Key",
  "algorithm": "RSA",
  "fingerprint": "d3:c9:c3:f9:6c:8d:9d:5f ... 4e:8a:7f:00",
  "publicKey": "-----BEGIN PUBLIC KEY-----...",
  "publicKeyDsa": null,
  "status": "active",
  "createdAt": "2025-06-05T10:15:30.000Z"
}
```

#### Success Response — Generate (private key shown once)

```json
{
  "id": "f8d9e1d0-7f4c-4f56-a2e4-b3e2d6f9c100",
  "name": "Generated Key",
  "algorithm": "RSA",
  "fingerprint": "d3:c9:c3:f9:6c:8d:9d:5f ... 4e:8a:7f:00",
  "publicKey": "-----BEGIN PUBLIC KEY-----...",
  "privateKey": "-----BEGIN PRIVATE KEY-----...",
  "publicKeyDsa": null,
  "privateKeyDsa": null,
  "status": "active"
}
```

> **Important:** `privateKey` (and `privateKeyDsa` for Hybrid) are returned only once at creation. The client offers an immediate download and warns the user the key will never be shown again.

#### Client — Private Key Download

```javascript
// Triggered from the success step in CreateAuthenticationKeyModal
const handleDownload = () => {
  let keyToDownload = createdKey.privateKey;
  if (createdKey.privateKeyDsa) {
    keyToDownload += '\n\n' + createdKey.privateKeyDsa;
  }
  // Triggers browser download as <keyname>_private_key.pem
};
```

#### Error Responses

| Status Code               | Description                                |
|---------------------------|--------------------------------------------|
| 400 Bad Request           | Missing or invalid fields                  |
| 401 Unauthorized          | Invalid or missing authentication token    |
| 500 Internal Server Error | Key generation failed                      |

---

### Update Authentication Key

Updates the name or status of an existing authentication key. Policies are re-fetched after a successful update to capture cascading name changes.

#### Function

```javascript
api.updateAuthKey(id, updateData)
```

#### Request

```
PUT /api/auth-keys/{id}
```

Requires `Authorization` header.

#### Payload

```json
{
  "name": "Updated Signing Key",
  "status": "disabled"
}
```

| Field  | Type   | Mandatory | Description                     |
|--------|--------|-----------|---------------------------------|
| name   | String | Optional  | Updated key name                |
| status | String | Optional  | `active` or `disabled`          |

#### Success Response

```json
{
  "id": "f8d9e1d0-7f4c-4f56-a2e4-b3e2d6f9c100",
  "name": "Updated Signing Key",
  "algorithm": "RSA",
  "status": "disabled",
  "updatedAt": "2025-06-06T11:22:45.000Z"
}
```

#### Error Responses

| Status Code               | Description                                |
|---------------------------|--------------------------------------------|
| 400 Bad Request           | Invalid update parameters                  |
| 401 Unauthorized          | Invalid or missing authentication token    |
| 404 Not Found             | Authentication key not found               |
| 500 Internal Server Error | Failed to update authentication key        |

---

## 4. Policy Management

### Get Policies

Fetches all policies for the authenticated user. Called during the global data sync on login, and again after any PQC key or auth key update.

#### Function

```javascript
api.getPolicies()
```

#### Request

```
GET /api/policies
```

Requires `Authorization` header.

#### Success Response

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Production Security Policy",
    "authKey": "Primary Signing Key",
    "authKeyId": "a100e8400-e29b-41d4-a716-446655440001",
    "pqcKey": "Production PQC Key",
    "pqcKeyId": "b200e8400-e29b-41d4-a716-446655440002",
    "operations": "Sign · Verify",
    "rateLimit": "1000/hour",
    "status": "active",
    "created": "2025-06-05T10:15:30.000Z",
    "createdAt": "2025-06-05T10:15:30.000Z",
    "updatedAt": "2025-06-05T10:15:30.000Z"
  }
]
```

---

### Create Policy

Creates a new access policy. The client enforces a 4-step wizard (Keys → Operations → Rate Limit → Review). Operations are filtered in the UI to only show those supported by the selected PQC key's algorithm.

#### Function

```javascript
api.createPolicy(policyData)
```

#### Request

```
POST /api/policies
```

Requires `Authorization` header.

#### Payload

```json
{
  "name": "Production Security Policy",
  "authKey": "Primary Signing Key",
  "authKeyId": "a100e8400-e29b-41d4-a716-446655440001",
  "pqcKey": "Production PQC Key",
  "pqcKeyId": "b200e8400-e29b-41d4-a716-446655440002",
  "operations": "Sign, Verify",
  "rateLimit": "1000",
  "status": "active"
}
```

| Field     | Type          | Mandatory | Description                                            |
|-----------|---------------|-----------|--------------------------------------------------------|
| name      | String        | Required  | Policy name (max 30 chars, alphanumeric with `_` or `-`) |
| authKey   | String        | Required  | Authentication key name                                |
| authKeyId | String (UUID) | Required  | Authentication key identifier                          |
| pqcKey    | String        | Required  | PQC key name                                           |
| pqcKeyId  | String (UUID) | Required  | PQC key identifier                                     |
| operations| String        | Required  | Comma-separated list of enabled operations (capitalized) |
| rateLimit | String        | Optional  | Rate limit value (e.g. `"1000"`); empty means unlimited |
| status    | String        | Required  | `active` (always on creation)                          |

#### Operations by PQC Algorithm

The UI filters available operations based on the selected PQC key:

| PQC Algorithm         | Shown Operations              |
|-----------------------|-------------------------------|
| ML-DSA, ECDSA, Hybrid-DSA | Sign, Verify               |
| ML-KEM, Hybrid-KEM    | Encapsulate, Decapsulate      |
| AES-256               | Encrypt, Decrypt              |

#### Client — Payload Construction

```javascript
const payload = {
  name: formData.name,
  authKey: formData.authKey,
  authKeyId: selectedAuthKeyObj.id,
  pqcKey: formData.pqcKey,
  pqcKeyId: selectedPQCKeyObj.id,
  operations: Object.entries(formData.operations)
    .filter(([op, enabled]) => enabled)
    .map(([op]) => op.charAt(0).toUpperCase() + op.slice(1))
    .join(', '),
  rateLimit: formData.rateLimit,
  status: 'active'
};
```

#### Success Response

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Production Security Policy",
  "authKey": "Primary Signing Key",
  "authKeyId": "a100e8400-e29b-41d4-a716-446655440001",
  "pqcKey": "Production PQC Key",
  "pqcKeyId": "b200e8400-e29b-41d4-a716-446655440002",
  "operations": "Sign, Verify",
  "rateLimit": "1000",
  "status": "active"
}
```

#### Error Responses

| Status Code               | Description                                              |
|---------------------------|----------------------------------------------------------|
| 400 Bad Request           | Invalid key IDs or duplicate key combination             |
| 401 Unauthorized          | Invalid or missing authentication token                  |
| 500 Internal Server Error | Failed to create policy                                  |

---

### Update Policy

Updates an existing policy's name, keys, operations, rate limit, or status.

#### Function

```javascript
api.updatePolicy(id, updateData)
```

#### Request

```
PUT /api/policies/{id}
```

Requires `Authorization` header.

#### Payload

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Updated Production Policy",
  "authKey": "Primary Signing Key",
  "authKeyId": "a100e8400-e29b-41d4-a716-446655440001",
  "pqcKey": "Production PQC Key",
  "pqcKeyId": "b200e8400-e29b-41d4-a716-446655440002",
  "operations": "Sign, Verify",
  "rateLimit": "500",
  "status": "disabled"
}
```

#### Allowed Status Values

| Status   |
|----------|
| active   |
| disabled |

#### Error Responses

| Status Code               | Description                                              |
|---------------------------|----------------------------------------------------------|
| 400 Bad Request           | Invalid key IDs or duplicate key combination             |
| 401 Unauthorized          | Invalid or missing authentication token                  |
| 404 Not Found             | Policy not found                                         |
| 500 Internal Server Error | Failed to update policy                                  |

---

## 5. Audit Logs

### Get Audit Logs

Fetches all audit log entries for the authenticated user. Called during global data sync on login and refreshed after every create/update operation.

#### Function

```javascript
api.getAuditLogs()
```

#### Request

```
GET /api/audit-logs
```

Requires `Authorization` header.

#### Success Response

```json
[
  {
    "id": 101,
    "operation": "pqc_sign",
    "authKey": "Primary Signing Key",
    "authKeyId": "550e8400-e29b-41d4-a716-446655440001",
    "pqcKey": "Production PQC Key",
    "pqcKeyId": "660e8400-e29b-41d4-a716-446655440002",
    "policyId": "770e8400-e29b-41d4-a716-446655440003",
    "sourceIP": "192.168.1.100",
    "result": "success",
    "createdAt": 1749112345678,
    "timestamp": "2025-06-05T10:32:25.678Z"
  }
]
```

#### Client Auto-Refresh

Audit logs are refreshed automatically after every mutating action:

```javascript
const refreshAuditLogs = async () => {
  const logs = await api.getAuditLogs();
  setAuditLogs(logs);
};

// Called after: addPQCKey, updatePQCKey, rotatePQCKey,
//               addAuthKey, updateAuthKey, addPolicy, updatePolicy
await refreshAuditLogs();
```

---

### Client-Side Filtering

Audit logs are filtered entirely on the client after fetching. No query parameters are sent to the API.

#### Available Filters

| Filter          | Options                                   |
|-----------------|-------------------------------------------|
| Authentication Key | `All` or any specific key name         |
| PQC Key         | `All` or any specific key name            |
| Result          | `All`, `Success`, or `Denied`             |

#### Filter Logic

```javascript
const filteredLogs = auditLogs.filter(log => {
  const matchesAuthKey = filterAuthKey === 'All' || log.authKey === filterAuthKey;
  const matchesPQCKey  = filterPQCKey  === 'All' || log.pqcKey  === filterPQCKey;
  const matchesResult  =
    filterResult === 'All' ||
    (filterResult === 'Success' && log.result === 'success') ||
    (filterResult === 'Denied'  && log.result !== 'success');

  return matchesAuthKey && matchesPQCKey && matchesResult;
});
```

---

## 6. Dashboard

### Get Dashboard Statistics

Fetches aggregated platform statistics for the authenticated user. Called during global data sync on login. Falls back to local counts derived from cached arrays if the API call fails.

#### Function

```javascript
api.getDashboardStats()
```

#### Request

```
GET /api/dashboard/stats
```

Requires `Authorization` header.

#### Success Response

```json
{
  "pqcKeys": 8,
  "authKeys": 4,
  "policies": 6,
  "totalOperations": 245,
  "successfulEvents": 231
}
```

#### Client Fallback Logic

```javascript
const stats = dashboardStats || {};
const pqcCount  = stats.pqcKeys  ?? pqcKeys.length;
const authCount = stats.authKeys ?? authKeys.length;
// totalOperations falls back to auditLogs.length if stats unavailable
```

#### Error Responses

| Status Code               | Description                                        |
|---------------------------|----------------------------------------------------|
| 401 Unauthorized          | Invalid or missing authentication token            |
| 500 Internal Server Error | Failed to retrieve dashboard statistics            |

---

## 7. IP Whitelist Settings

### Get IP Whitelist

Fetches current IP restriction settings. Called on Settings page load.

#### Function

```javascript
api.getIpWhitelist()
```

#### Request

```
GET /api/auth/settings/ip-whitelist
```

Requires `Authorization` header.

#### Success Response

```json
{
  "mode": "selected",
  "whitelist": [
    "192.168.1.10",
    "10.0.0.0/24"
  ]
}
```

| Field     | Type            | Description                                                   |
|-----------|-----------------|---------------------------------------------------------------|
| mode      | String          | `any` (allow all IPs) or `selected` (only whitelisted IPs)   |
| whitelist | Array\<String\> | List of allowed IPv4, IPv6, or CIDR addresses                 |

---

### Update IP Whitelist

Updates the IP restriction mode and/or whitelist. The mode can be updated independently (e.g. switching from `any` to `selected`) or with a new list of IPs from the management modal.

#### Function

```javascript
api.updateIpWhitelist({ mode, whitelist })
```

#### Request

```
PUT /api/auth/settings/ip-whitelist
```

Requires `Authorization` header.

#### Payload

```json
{
  "mode": "selected",
  "whitelist": [
    "192.168.1.10",
    "10.0.0.0/24"
  ]
}
```

| Field     | Type            | Mandatory | Description                              |
|-----------|-----------------|-----------|------------------------------------------|
| mode      | String          | Required  | `any` or `selected`                      |
| whitelist | Array\<String\> | Required  | List of IP addresses or CIDR blocks      |

#### Client-Side IP Validation (enforced in UI before save)

```
- IPv4: four octets, each 0–255 (e.g. 192.168.1.10)
- IPv4 CIDR: valid IP + prefix 0–32 (e.g. 10.0.0.0/24)
- IPv6: simplified colon-hex format
- Duplicates within the list are rejected
```

#### Success Response

```json
{
  "mode": "selected",
  "whitelist": [
    "192.168.1.10",
    "10.0.0.0/24"
  ]
}
```

> **Warning:** If mode is `selected` and the whitelist is empty, all cryptographic operations (`/api/crypto/*`) will be blocked.

---

## 8. MTLS Certificate Management

### Get MTLS Settings

Fetches current Mutual TLS mode. Called on Settings page load.

#### Function

```javascript
api.getMTLSSettings()
```

#### Request

```
GET /api/auth/mtls/settings
```

Requires `Authorization` header.

#### Success Response

```json
{
  "mtlsMode": "standard",
  "mtlsEnabled": false,
  "certificateCount": 0,
  "certificates": []
}
```

| Field            | Type    | Description                                           |
|------------------|---------|-------------------------------------------------------|
| mtlsMode         | String  | `standard` (JWT only) or `mtls` (JWT + certificate)   |
| mtlsEnabled      | Boolean | Whether mTLS enforcement is currently active          |
| certificateCount | Integer | Number of active certificates                         |
| certificates     | Array   | Active certificate list (may be empty)                |

---

### Update MTLS Settings

Switches the MTLS mode. The client reverts to the previous mode automatically if the API call fails.

#### Function

```javascript
api.updateMTLSSettings(mtlsMode)
```

#### Request

```
PUT /api/auth/mtls/settings
```

Requires `Authorization` header.

#### Payload

```json
{
  "mtlsMode": "mtls"
}
```

| Field    | Type   | Mandatory | Description                           |
|----------|--------|-----------|---------------------------------------|
| mtlsMode | String | Required  | `standard` or `mtls`                  |

#### Success Response

```json
{
  "mtlsMode": "mtls",
  "mtlsEnabled": true
}
```

---

### Issue MTLS Certificate

Issues a new mTLS client certificate by submitting a CSR file. The client accepts `.csr` or `.pem` files via file input. The certificate bundle is automatically downloaded immediately after issuance.

#### Function

```javascript
api.issueMTLSCertificate(csr, certificateName)
```

#### Request

```
POST /api/auth/mtls/issue
```

Requires `Authorization` header.

#### Payload

```json
{
  "csr": "-----BEGIN CERTIFICATE REQUEST-----\nMIICvDCCAaQCAQAwdzEL...\n-----END CERTIFICATE REQUEST-----",
  "certificateName": "Production Server"
}
```

| Field           | Type   | Mandatory | Description                                              |
|-----------------|--------|-----------|----------------------------------------------------------|
| csr             | String | Required  | Full PEM-encoded Certificate Signing Request             |
| certificateName | String | Required  | Unique display name for this certificate (min 3 chars)   |

#### Client-Side Validation

```
- CSR file must be uploaded (non-empty)
- Certificate name must not be empty and must be at least 3 characters
- A 300ms debounce is applied to prevent accidental double-submission
```

#### Success Response

```json
{
  "success": true,
  "certificate": {
    "id": 1,
    "certificateName": "Production Server",
    "serialNumber": "0xA1B2C3D4",
    "fingerprint": "SHA256_HASH",
    "clientCert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "caCert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "serverCert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }
}
```

#### Client — Auto-Download After Issue

```javascript
// Triggered immediately after successful issuance
const certsToDownload = [
  { filename: 'client.crt', content: certificate.clientCert || certificate.clientCrt },
  { filename: 'ca.crt',     content: certificate.caCert     || certificate.caCrt     },
  { filename: 'server.crt', content: certificate.serverCert || certificate.serverCrt }
];

// Each file is created as a Blob and downloaded via a temporary <a> element
// A 500ms delay is inserted between downloads to prevent browser blocking
```

---

### Get Active MTLS Certificates

Retrieves all currently active (non-revoked) certificates. Called each time the MTLS management modal opens.

#### Function

```javascript
api.getActiveMTLSCertificates()
```

#### Request

```
GET /api/auth/mtls/certificates
```

Requires `Authorization` header.

#### Success Response

```json
{
  "certificates": [
    {
      "id": 1,
      "certificateName": "Production Server",
      "serialNumber": "0xA1B2C3D4",
      "issuedAt": "2025-06-05T10:15:30.000Z",
      "validUntil": "2026-06-05T10:15:30.000Z"
    }
  ],
  "count": 1
}
```

#### Response Fields (per certificate)

| Field           | Type    | Description                          |
|-----------------|---------|--------------------------------------|
| id              | Integer | Certificate identifier               |
| certificateName | String  | Display name                         |
| serialNumber    | String  | Certificate serial number            |
| issuedAt        | String  | ISO-8601 issue timestamp             |
| validUntil      | String  | ISO-8601 expiry timestamp            |

---

### Get Revoked MTLS Certificates

Retrieves all revoked certificates.

#### Function

```javascript
api.getRevokedMTLSCertificates()
```

#### Request

```
GET /api/auth/mtls/certificates/revoked
```

Requires `Authorization` header.

#### Success Response

```json
{
  "certificates": [],
  "count": 0
}
```

---

### Revoke MTLS Certificate

Revokes a certificate. The client shows a browser confirm dialog before calling this endpoint. Revocation reason and notes are passed as query parameters.

#### Function

```javascript
api.revokeMTLSCertificate(certificateId, reason, notes)
```

#### Request

```
DELETE /api/auth/mtls/certificates/{certificateId}?reason={reason}&notes={notes}
```

Requires `Authorization` header.

#### Query Parameters

| Parameter | Type   | Default          | Description                                    |
|-----------|--------|------------------|------------------------------------------------|
| reason    | String | `user_requested` | Revocation reason                              |
| notes     | String | `''`             | Optional notes (URL-encoded via `encodeURIComponent`) |

#### Example Constructed URL

```
DELETE /api/auth/mtls/certificates/1?reason=user_requested&notes=
```

#### Success Response

```json
{
  "success": true,
  "message": "Certificate revoked successfully"
}
```

---

### Download MTLS Certificates

Downloads the certificate bundle for an existing certificate. Returns raw certificate data; the client constructs and triggers three separate file downloads (`client.crt`, `ca.crt`, `server.crt`).

#### Function

```javascript
api.downloadMTLSCertificates(certificateId)
```

#### Request

```
GET /api/auth/mtls/download/{certificateId}
```

Requires `Authorization` header.

#### Success Response

```json
{
  "certificates": {
    "clientCrt": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "caCrt": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "serverCrt": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }
}
```

#### Client — Re-mapping for Download

```javascript
// The download endpoint uses different field names than the issue endpoint
// The client normalises them:
const downloadPayload = {
  clientCert: certData.certificates?.clientCrt,
  caCert:     certData.certificates?.caCrt,
  serverCert: certData.certificates?.serverCrt
};
```

---

### Verify MTLS Certificate

Verifies a PEM-encoded certificate against the server's CA and revocation list.

#### Function

```javascript
api.verifyMTLSCertificate(certificate)
```

#### Request

```
POST /api/auth/mtls/verify
```

Requires `Authorization` header.

#### Payload

```json
{
  "certificate": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
}
```

#### Success Response

```json
{
  "valid": true,
  "userId": 1,
  "certificateId": 1,
  "certificateName": "Production Server"
}
```

#### Failed Verification Response

```json
{
  "valid": false,
  "error": "Certificate has been revoked"
}
```

---

## 9. Contact / Support

### Submit Contact Request

Submits a contact or sales enquiry. This is the only endpoint that bypasses the `/api` prefix — it posts to a sibling `/requests` endpoint on the backend. The payload is transformed by the client before sending.

#### Function

```javascript
api.submitContactRequest(data)
```

#### Request

```
POST /requests
```

No `Authorization` header. No `/api` prefix.

```javascript
// URL construction
const url = API_URL.replace('/api', '') + '/requests';
// e.g. "http://localhost:5000/requests"
```

#### Form Fields (collected by ContactUsModal)

| Field   | Type   | Mandatory | Description                                   |
|---------|--------|-----------|-----------------------------------------------|
| name    | String | Required  | Full name of the requester                    |
| email   | String | Required  | Work email address                            |
| company | String | Required  | Company name                                  |
| tier    | String | Required  | Selected plan (`Hardware-Anchored PQC` or `PQC Cloud HSM`) |

#### Client Payload Transformation

The form fields are remapped before sending:

```javascript
const payload = {
  fullName: `${data.name} - ${data.company}`,  // Combined name + company
  mobile: "9999999999",                          // Placeholder value
  email: data.email,
  serviceOffering: data.tier,
  message: "cms contact us",                     // Static placeholder
  agreePrivacy: true,
  subscribeUpdates: true
};
```

#### Transformed Payload (sent to server)

```json
{
  "fullName": "John Doe - Acme Inc.",
  "mobile": "9999999999",
  "email": "john@acme.com",
  "serviceOffering": "Hardware-Anchored PQC",
  "message": "cms contact us",
  "agreePrivacy": true,
  "subscribeUpdates": true
}
```

#### Success Response

```json
{
  "message": "Contact request submitted successfully!",
  "data": {
    "success": true,
    "referenceId": "REQ-2025-001"
  }
}
```

#### Error Responses

| Status Code     | Description                                  |
|-----------------|----------------------------------------------|
| 400 Bad Request | Required fields are missing                  |
| 502 Bad Gateway | External support service is unavailable      |
| 500 Internal Server Error | Server configuration error          |

---

## Client-Side State Management

### Local Storage Keys

The application persists authentication and data in `localStorage` across sessions:

| Key                    | Value                         | Description                                  |
|------------------------|-------------------------------|----------------------------------------------|
| `jwt_token`            | String                        | JWT access token                             |
| `user_profile`         | JSON String                   | Authenticated user profile object            |
| `lastActive`           | Unix timestamp (ms)           | Last user activity timestamp for session timeout |
| `cache_pqc_keys`       | JSON String                   | Cached PQC keys array                        |
| `cache_auth_keys`      | JSON String                   | Cached authentication keys array             |
| `cache_policies`       | JSON String                   | Cached policies array                        |
| `cache_audit_logs`     | JSON String                   | Cached audit logs array                      |
| `cache_dashboard_stats`| JSON String                   | Cached dashboard statistics object           |

All cache keys are cleared on logout:

```javascript
localStorage.removeItem('jwt_token');
localStorage.removeItem('user_profile');
localStorage.removeItem('lastActive');
localStorage.removeItem('cache_pqc_keys');
localStorage.removeItem('cache_auth_keys');
localStorage.removeItem('cache_policies');
localStorage.removeItem('cache_audit_logs');
localStorage.removeItem('cache_dashboard_stats');
```

---

### Session Timeout

The application automatically logs the user out after 30 minutes of inactivity.

Activity is tracked on `mousemove`, `keydown`, and `click` events. An interval checks inactivity every 60 seconds.

```javascript
// Timeout threshold
const TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes

// Check
const inactiveTime = Date.now() - parseInt(localStorage.getItem('lastActive'));
if (inactiveTime > TIMEOUT_MS) {
  performLogout();
}
```

---

### Data Caching Strategy

On every successful login, all data is loaded in a single parallel request:

```javascript
const [keys, aKeys, pols, logs, stats] = await Promise.all([
  api.getPqcKeys(),
  api.getAuthKeys(),
  api.getPolicies(),
  api.getAuditLogs(),
  api.getDashboardStats()
]);
```

Data is served from the in-memory React state (initialized from `localStorage` on mount) for instant UI rendering. The cache is kept in sync after every mutation — either via optimistic local update or a full re-fetch depending on the operation:

| Action          | Strategy                                              |
|-----------------|-------------------------------------------------------|
| Create key      | Prepend new item to existing array                    |
| Update key      | Replace matching item by `id` in existing array       |
| Rotate PQC key  | Full re-fetch of `pqc-keys` and `policies`            |
| Update auth key | Replace + full re-fetch of `policies` (cascade names) |
| Update PQC key  | Replace + full re-fetch of `policies` (cascade names) |
| Any mutation    | Always refresh audit logs after                       |

If the API returns `401` during the data load, the user is automatically logged out:

```javascript
if (err.status === 401) {
  performLogout();
}
```

---

## Client-Side Validation Rules

These validations are enforced in the UI before any API call is made:

### Name Validation (Keys & Policies)

```javascript
// Regex: alphanumeric, underscore, and hyphen only
const regex = /^[a-zA-Z0-9_-]*$/;

// Rules:
// - Max 30 characters
// - Only a-z, A-Z, 0-9, _ and - are allowed
// - Empty string passes (required check is separate)
```

**Error message shown in UI:**
> Name must be alphanumeric with only `_` or `-` and under 30 characters.

### Duplicate Name Check (Keys)

```javascript
const isDuplicateName = existingKeys.some(key =>
  key.name.toLowerCase() === formData.name.trim().toLowerCase()
);
```

**Error message shown in UI:**
> A key with the name `"<name>"` already exists.

### IP Address Validation

```
- IPv4:      1–3 digits per octet, each 0–255
- IPv4 CIDR: valid IPv4 + /0 to /32
- IPv6:      simplified colon-hex groups
- Duplicates are rejected within the same list
```

### MTLS Certificate Name

```
- Must not be empty
- Minimum 3 characters
```

### TOTP Code

```
- Must be exactly 6 digits
- Non-numeric characters are stripped on input
```

---

## Utility Helpers

### Date Formatting

```javascript
import { formatDate, formatDateOnly } from '../utils/dateFormatter';

// Full date + time
formatDate('2025-06-05T10:15:30.000Z');
// → "Jun 5, 2025, 10:15 AM"

// Date only
formatDateOnly('2025-06-05T10:15:30.000Z');
// → "Jun 5, 2025"
```

Returns the original ISO string as a fallback if the date is invalid.

---

### Fingerprint Formatting

Truncates long colon-separated fingerprint strings to a condensed format for display:

```javascript
import { formatFingerprint } from '../utils/fingerprintFormatter';

formatFingerprint('AA:BB:CC:DD:EE:FF:GG:HH');
// → "AA:BB:CC ... FF:GG:HH"
// (first 3 parts ... last 3 parts)
// Strings with 6 or fewer parts are returned unchanged
```

---

### Name Validation & Truncation

```javascript
import { validateName, truncateName } from '../utils/validation';

// Validate (returns boolean)
validateName('my-key_01'); // true
validateName('my key!');   // false (space and ! not allowed)
validateName('');          // true (empty allowed during typing)

// Truncate for display
truncateName('A Very Long Key Name That Exceeds The Limit');
// Desktop (>768px): first 30 chars + "..."
// Mobile (≤768px):  first 20 chars + "..."
```
