# API Documentation

A comprehensive reference for all platform APIs including Authentication, Key Management, Cryptographic Operations, Audit Logs, Dashboard, and Contact services.

#### Table of Contents

- [1. Authentication & Security API](#1-authentication--security-api)
  - [General API Information](#general-api-information)
  - [Authentication](#authentication)
  - [Rate Limits](#rate-limits)
  - [Google OAuth Login](#google-oauth-login)
  - [Verify Two-Factor Authentication Login](#verify-two-factor-authentication-login)
  - [Get User Profile](#get-user-profile)
  - [Setup Two-Factor Authentication](#setup-two-factor-authentication)
  - [Verify Two-Factor Authentication Setup](#verify-two-factor-authentication-setup)
  - [Get Two-Factor Authentication Status](#get-two-factor-authentication-status)
  - [Disable Two-Factor Authentication](#disable-two-factor-authentication)
  - [Get IP Whitelist Settings](#get-ip-whitelist-settings)
  - [Update IP Whitelist Settings](#update-ip-whitelist-settings)
  - [Get MTLS Settings](#get-mtls-settings)
  - [Update MTLS Settings](#update-mtls-settings)
  - [Issue Certificate](#issue-certificate)
  - [Get Active Certificates](#get-active-certificates)
  - [Get Revoked Certificates](#get-revoked-certificates)
  - [Revoke Certificate](#revoke-certificate)
  - [Download Certificates](#download-certificates)
  - [Verify Certificate](#verify-certificate)
- [2. Authentication Key Management API](#2-authentication-key-management-api)
  - [General API Information](#general-api-information-1)
  - [Authentication](#authentication-1)
  - [Supported Algorithms](#supported-algorithms)
  - [Get Authentication Keys](#get-authentication-keys)
  - [Create Authentication Key](#create-authentication-key)
  - [Update Authentication Key](#update-authentication-key)
  - [Authentication Key Object](#authentication-key-object)
  - [Security Notes](#security-notes)
- [3. Policy Management API](#3-policy-management-api)
  - [General API Information](#general-api-information-2)
  - [Authentication](#authentication-2)
  - [Policy Components](#policy-components)
  - [Get Policies](#get-policies)
  - [Create Policy](#create-policy)
  - [Update Policy](#update-policy)
  - [Policy Object](#policy-object)
  - [Policy Validation Rules](#policy-validation-rules)
  - [Security Notes](#security-notes-1)
- [4. PQC Key Management API](#4-pqc-key-management-api)
  - [General API Information](#general-api-information-3)
  - [Authentication](#authentication-3)
  - [Supported Algorithms](#supported-algorithms-1)
  - [Supported Operations](#supported-operations)
  - [Get PQC Keys](#get-pqc-keys)
  - [Create PQC Key](#create-pqc-key)
  - [Update PQC Key](#update-pqc-key)
  - [Rotate PQC Key](#rotate-pqc-key)
  - [PQC Key Object](#pqc-key-object)
  - [Versioning](#versioning)
  - [Key Rotation](#key-rotation)
  - [Security Notes](#security-notes-2)
- [5. Cryptographic Operations API](#5-cryptographic-operations-api)
  - [General API Information](#general-api-information-4)
  - [Authentication](#authentication-4)
  - [Security Controls](#security-controls)
  - [Supported Operations](#supported-operations-1)
  - [Supported Algorithms](#supported-algorithms-2)
  - [Sign Data](#sign-data)
  - [Verify Signature](#verify-signature)
  - [Encapsulate Secret](#encapsulate-secret)
  - [Decapsulate Secret](#decapsulate-secret)
  - [Encrypt Data](#encrypt-data)
  - [Decrypt Data](#decrypt-data)
  - [Policy Enforcement](#policy-enforcement)
  - [Security Notes](#security-notes-3)
- [6. Audit Logs API](#6-audit-logs-api)
  - [General API Information](#general-api-information-5)
  - [Authentication](#authentication-5)
  - [Get Audit Logs](#get-audit-logs)
  - [Audit Log Object](#audit-log-object)
  - [Filtering Rules](#filtering-rules)
  - [Sorting Rules](#sorting-rules)
  - [Security Notes](#security-notes-4)
- [7. Dashboard API](#7-dashboard-api)
  - [General API Information](#general-api-information-6)
  - [Authentication](#authentication-6)
  - [Dashboard Metrics](#dashboard-metrics)
  - [Get Dashboard Statistics](#get-dashboard-statistics)
  - [Dashboard Statistics Object](#dashboard-statistics-object)
  - [Security Notes](#security-notes-5)
- [8. Contact API](#8-contact-api)
  - [General API Information](#general-api-information-7)
  - [Authentication](#authentication-7)
  - [Submit Contact Request](#submit-contact-request)
  - [Contact Request Object](#contact-request-object)
  - [Error Responses](#error-responses-contact)
  - [Security Notes](#security-notes-6)

---

## 1. Authentication & Security API

Provides authentication, user profile management, Two-Factor Authentication (2FA), IP whitelist management, and Mutual TLS (mTLS) certificate operations.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- Authentication is handled using JWT Bearer Tokens.
- Google OAuth is used as the primary login provider.
- Two-Factor Authentication (2FA) is supported using TOTP-compatible authenticator applications.
- IP Whitelisting can be configured to restrict account access.
- Mutual TLS (mTLS) certificates can be issued, managed, revoked, downloaded, and verified through the API.

### Authentication

Protected endpoints require a valid JWT access token.

```
Authorization: Bearer <access_token>
```

**Public Endpoints (no token required):**

```
POST /api/auth/google
POST /api/auth/2fa/verify
```

All other endpoints require authentication.

### Rate Limits

Authentication endpoints may be subject to rate limiting to prevent abuse, brute-force attacks, and unauthorized access attempts.

---

### Google OAuth Login

Authenticate a user using a Google OAuth authorization code.

#### Request

```
POST /api/auth/google
```

#### Parameters

| Name | Type   | Mandatory | Description                        |
|------|--------|-----------|------------------------------------|
| code | String | Required  | Google OAuth authorization code    |

#### Example Request

```json
{
  "code": "4/0AX4XfWhExampleGoogleAuthorizationCode"
}
```

#### Success Response

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

#### Response Fields

| Field       | Type    | Description                                          |
|-------------|---------|------------------------------------------------------|
| requires2FA | Boolean | Indicates whether 2FA verification is required       |
| token       | String  | JWT access token                                     |
| user        | Object  | User profile information                             |

#### Error Responses

| Status Code               | Description                        |
|---------------------------|------------------------------------|
| 400 Bad Request           | Authorization code is missing      |
| 401 Unauthorized          | Invalid Google authorization code  |
| 500 Internal Server Error | Unexpected server error            |

---

### Verify Two-Factor Authentication Login

Verify a TOTP code during login when 2FA is enabled.

#### Request

```
POST /api/auth/2fa/verify
```

#### Parameters

| Name      | Type   | Mandatory | Description                          |
|-----------|--------|-----------|--------------------------------------|
| tempToken | String | Required  | Temporary authentication token       |
| code      | String | Required  | Six-digit authenticator code         |

#### Example Request

```json
{
  "tempToken": "temporary_token",
  "code": "123456"
}
```

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

---

### Get User Profile

Retrieve profile information of the currently authenticated user.

#### Request

```
GET /api/auth/me
```

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

---

### Setup Two-Factor Authentication

Generate a TOTP secret and QR code information for enabling 2FA.

#### Request

```
POST /api/auth/2fa/setup
```

#### Success Response

```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "otpauthUri": "otpauth://totp/...",
  "qrData": "otpauth://totp/..."
}
```

---

### Verify Two-Factor Authentication Setup

Verify the generated secret and enable 2FA.

#### Request

```
POST /api/auth/2fa/verify-setup
```

#### Parameters

| Name   | Type   | Mandatory | Description                    |
|--------|--------|-----------|--------------------------------|
| code   | String | Required  | Six-digit authenticator code   |
| secret | String | Required  | Generated secret               |

#### Success Response

```json
{
  "enabled": true
}
```

---

### Get Two-Factor Authentication Status

Retrieve the current 2FA status for the authenticated user.

#### Request

```
GET /api/auth/2fa/status
```

#### Success Response

```json
{
  "enabled": true
}
```

---

### Disable Two-Factor Authentication

Disable 2FA for the authenticated user.

#### Request

```
DELETE /api/auth/2fa
```

#### Success Response

```json
{
  "enabled": false
}
```

---

### Get IP Whitelist Settings

Retrieve current IP whitelist configuration.

#### Request

```
GET /api/auth/settings/ip-whitelist
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

---

### Update IP Whitelist Settings

Update IP whitelist configuration.

#### Request

```
PUT /api/auth/settings/ip-whitelist
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

---

### Get MTLS Settings

Retrieve current Mutual TLS (mTLS) configuration.

#### Request

```
GET /api/auth/mtls/settings
```

#### Success Response

```json
{
  "mtlsMode": "mtls",
  "mtlsEnabled": true,
  "certificateCount": 2,
  "certificates": []
}
```

---

### Update MTLS Settings

Update Mutual TLS (mTLS) configuration.

#### Request

```
PUT /api/auth/mtls/settings
```

#### Success Response

```json
{
  "mtlsMode": "mtls",
  "mtlsEnabled": true
}
```

---

### Issue Certificate

Issue a new mTLS client certificate.

#### Request

```
POST /api/auth/mtls/issue
```

#### Success Response

```json
{
  "success": true,
  "certificate": {
    "id": 1,
    "certificateName": "Production Client",
    "serialNumber": "0xA1B2C3D4",
    "fingerprint": "SHA256_HASH"
  }
}
```

---

### Get Active Certificates

Retrieve all active mTLS certificates.

#### Request

```
GET /api/auth/mtls/certificates
```

#### Success Response

```json
{
  "certificates": [],
  "count": 0
}
```

---

### Get Revoked Certificates

Retrieve all revoked mTLS certificates.

#### Request

```
GET /api/auth/mtls/certificates/revoked
```

#### Success Response

```json
{
  "certificates": [],
  "count": 0
}
```

---

### Revoke Certificate

Revoke an active mTLS certificate by ID.

#### Request

```
DELETE /api/auth/mtls/certificates/{certificateId}
```

#### Success Response

```json
{
  "success": true,
  "message": "Certificate revoked successfully"
}
```

---

### Download Certificates

Download the certificate bundle for a given certificate ID.

#### Request

```
GET /api/auth/mtls/download/{certificateId}
```

#### Success Response

Returns the certificate bundle, including:
- Client Certificate
- CA Certificate
- Server Certificate

---

### Verify Certificate

Verify the validity of a presented mTLS certificate.

#### Request

```
POST /api/auth/mtls/verify
```

#### Success Response

```json
{
  "valid": true,
  "userId": 1,
  "certificateId": 1,
  "certificateName": "Production Client"
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

## 2. Authentication Key Management API

Provides authentication key generation, storage, management, fingerprinting, status management, and hybrid cryptographic key support. Authentication keys are used throughout the platform for identity verification, digital signatures, policy enforcement, and cryptographic operations.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- All endpoints require authentication.
- Authentication keys are generated and managed per user account.
- Each key is assigned a unique fingerprint calculated using SHA-256.
- Hybrid keys combine classical and post-quantum cryptographic algorithms.
- Private keys are encrypted before storage.

### Authentication

All Authentication Key Management endpoints require a valid JWT access token.

```
Authorization: Bearer <access_token>
```

### Supported Algorithms

| Algorithm | Description                                      |
|-----------|--------------------------------------------------|
| RSA       | Rivest–Shamir–Adleman public key cryptography    |
| ECDSA     | Elliptic Curve Digital Signature Algorithm       |
| ML-DSA    | Post-Quantum Digital Signature Algorithm         |
| Hybrid    | Combination of ECDSA and ML-DSA                  |

---

### Get Authentication Keys

Retrieve all authentication keys belonging to the authenticated user.

#### Request

```
GET /api/auth-keys
```

#### Success Response

```json
[
  {
    "id": "f8d9e1d0-7f4c-4f56-a2e4-b3e2d6f9c100",
    "name": "Primary Signing Key",
    "algorithm": "RSA",
    "fingerprint": "d3c9c3f96c8d9d5f4a9f5c5f74e8a7f0",
    "publicKey": "-----BEGIN PUBLIC KEY-----...",
    "publicKeyDsa": null,
    "status": "active",
    "created": "2025-06-05T10:15:30.000Z",
    "createdAt": "2025-06-05T10:15:30.000Z",
    "updatedAt": "2025-06-05T10:15:30.000Z"
  }
]
```

#### Response Fields

| Field        | Type          | Description                              |
|--------------|---------------|------------------------------------------|
| id           | String (UUID) | Unique authentication key identifier     |
| name         | String        | Authentication key name                  |
| algorithm    | String        | Cryptographic algorithm used             |
| fingerprint  | String        | SHA-256 fingerprint of the key           |
| publicKey    | String        | Classical public key                     |
| publicKeyDsa | String / Null | PQC public key component                 |
| status       | String        | Current key status                       |
| created      | String        | Key creation timestamp                   |
| createdAt    | String        | Database creation timestamp              |
| updatedAt    | String        | Last modification timestamp              |

#### HTTP Status Codes

| Status Code               | Description                                  |
|---------------------------|----------------------------------------------|
| 200 OK                    | Authentication keys retrieved successfully   |
| 401 Unauthorized          | Invalid or missing authentication token      |
| 500 Internal Server Error | Failed to retrieve authentication keys       |

---

### Create Authentication Key

Create a new authentication key using one of the supported cryptographic algorithms.

#### Request

```
POST /api/auth-keys
```

#### Parameters

| Name      | Type   | Mandatory | Description                   |
|-----------|--------|-----------|-------------------------------|
| name      | String | Required  | Authentication key name       |
| algorithm | String | Required  | Cryptographic algorithm       |
| status    | String | Optional  | Initial key status            |

#### Example Request

```json
{
  "name": "Primary Signing Key",
  "algorithm": "RSA",
  "status": "active"
}
```

#### Example Request (Hybrid)

```json
{
  "name": "Hybrid Enterprise Key",
  "algorithm": "Hybrid"
}
```

#### Success Response

```json
{
  "id": "f8d9e1d0-7f4c-4f56-a2e4-b3e2d6f9c100",
  "name": "Primary Signing Key",
  "algorithm": "RSA",
  "fingerprint": "d3c9c3f96c8d9d5f4a9f5c5f74e8a7f0",
  "publicKey": "-----BEGIN PUBLIC KEY-----...",
  "privateKey": "-----BEGIN PRIVATE KEY-----...",
  "publicKeyDsa": null,
  "privateKeyDsa": null,
  "status": "active",
  "created": "2025-06-05T10:15:30.000Z",
  "createdAt": "2025-06-05T10:15:30.000Z",
  "updatedAt": "2025-06-05T10:15:30.000Z"
}
```

#### Response Fields

| Field         | Type          | Description                          |
|---------------|---------------|--------------------------------------|
| id            | String (UUID) | Unique key identifier                |
| name          | String        | Authentication key name              |
| algorithm     | String        | Selected cryptographic algorithm     |
| fingerprint   | String        | SHA-256 fingerprint                  |
| publicKey     | String        | Generated classical public key       |
| privateKey    | String        | Generated classical private key      |
| publicKeyDsa  | String / Null | Generated PQC public key             |
| privateKeyDsa | String / Null | Generated PQC private key            |
| status        | String        | Current key status                   |

#### Validation Rules

| Field     | Rule                                      |
|-----------|-------------------------------------------|
| name      | Must not be empty                         |
| algorithm | Must be RSA, ECDSA, ML-DSA, or Hybrid     |

#### HTTP Status Codes

| Status Code               | Description                                |
|---------------------------|--------------------------------------------|
| 201 Created               | Authentication key created successfully    |
| 400 Bad Request           | Missing or invalid fields                  |
| 401 Unauthorized          | Invalid or missing authentication token    |
| 500 Internal Server Error | Key generation failed                      |

---

### Update Authentication Key

Update the name or status of an existing authentication key.

#### Request

```
PUT /api/auth-keys/{id}
```

#### Parameters

| Name   | Type   | Mandatory | Description                    |
|--------|--------|-----------|--------------------------------|
| name   | String | Optional  | New authentication key name    |
| status | String | Optional  | Updated status                 |

#### Example Request

```json
{
  "name": "Updated Signing Key"
}
```

#### Example Request (Status Change)

```json
{
  "status": "disabled"
}
```

#### Success Response

```json
{
  "id": "f8d9e1d0-7f4c-4f56-a2e4-b3e2d6f9c100",
  "name": "Updated Signing Key",
  "algorithm": "RSA",
  "fingerprint": "d3c9c3f96c8d9d5f4a9f5c5f74e8a7f0",
  "publicKey": "-----BEGIN PUBLIC KEY-----...",
  "publicKeyDsa": null,
  "status": "disabled",
  "createdAt": "2025-06-05T10:15:30.000Z",
  "updatedAt": "2025-06-06T11:22:45.000Z"
}
```

#### Validation Rules

| Field  | Rule                          |
|--------|-------------------------------|
| name   | Cannot be empty               |
| status | Must be `active` or `disabled` |

#### HTTP Status Codes

| Status Code               | Description                                |
|---------------------------|--------------------------------------------|
| 200 OK                    | Authentication key updated successfully    |
| 400 Bad Request           | Invalid update parameters                  |
| 401 Unauthorized          | Invalid or missing authentication token    |
| 404 Not Found             | Authentication key not found               |
| 500 Internal Server Error | Failed to update authentication key        |

---

### Authentication Key Object

Authentication keys are represented using the following structure:

```json
{
  "id": "uuid",
  "name": "Primary Signing Key",
  "algorithm": "RSA",
  "fingerprint": "sha256_fingerprint",
  "publicKey": "public_key_data",
  "publicKeyDsa": "pqc_public_key_data",
  "status": "active",
  "createdAt": "2025-06-05T10:15:30.000Z",
  "updatedAt": "2025-06-05T10:15:30.000Z"
}
```

### Security Notes

- Private keys are encrypted before being stored.
- SHA-256 fingerprints are generated automatically for all keys.
- Hybrid keys combine classical and post-quantum cryptographic algorithms.
- Authentication keys are isolated per user account.
- Status changes are recorded in audit logs.
- Key creation, modification, and status updates generate audit trail entries.
- Renaming an authentication key automatically updates linked policy references.
- Private key material should never be shared with unauthorized users.

---

## 3. Policy Management API

Provides creation, management, enforcement, and lifecycle control of security policies. Policies define how Authentication Keys and Post-Quantum Cryptography (PQC) Keys are associated, what operations are permitted, and what rate limits apply to cryptographic operations within the platform.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- All endpoints require authentication.
- Policies are managed per user account.
- Policies can be linked to Authentication Keys and PQC Keys.
- Only active Authentication Keys and PQC Keys can be assigned to a policy.
- A unique policy must exist for each Authentication Key and PQC Key combination.
- All policy operations are recorded in the audit logging system.

### Authentication

All Policy Management endpoints require a valid JWT access token.

```
Authorization: Bearer <access_token>
```

### Policy Components

A policy may contain the following components:

| Component          | Description                                               |
|--------------------|-----------------------------------------------------------|
| Authentication Key | Classical authentication key associated with the policy   |
| PQC Key            | Post-Quantum Cryptographic key associated with the policy  |
| Operations         | Allowed cryptographic operations                          |
| Rate Limit         | Allowed request rate configuration                        |
| Status             | Current policy state                                      |

---

### Get Policies

Retrieve all policies associated with the authenticated user.

#### Request

```
GET /api/policies
```

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

#### Response Fields

| Field     | Type          | Description                        |
|-----------|---------------|------------------------------------|
| id        | String (UUID) | Unique policy identifier           |
| name      | String        | Policy name                        |
| authKey   | String        | Authentication key name            |
| authKeyId | String (UUID) | Authentication key identifier      |
| pqcKey    | String        | PQC key name                       |
| pqcKeyId  | String (UUID) | PQC key identifier                 |
| operations| String        | Allowed operations                 |
| rateLimit | String        | Configured rate limit              |
| status    | String        | Policy status                      |
| created   | String        | Creation timestamp                 |
| createdAt | String        | Database creation timestamp        |
| updatedAt | String        | Last modification timestamp        |

#### HTTP Status Codes

| Status Code               | Description                          |
|---------------------------|--------------------------------------|
| 200 OK                    | Policies retrieved successfully      |
| 401 Unauthorized          | Invalid or missing authentication token |
| 500 Internal Server Error | Failed to retrieve policies          |

---

### Create Policy

Create a new policy and associate it with Authentication Keys and PQC Keys.

#### Request

```
POST /api/policies
```

#### Parameters

| Name      | Type          | Mandatory | Description                       |
|-----------|---------------|-----------|-----------------------------------|
| name      | String        | Required  | Policy name                       |
| authKeyId | String (UUID) | Optional  | Authentication Key identifier     |
| pqcKeyId  | String (UUID) | Optional  | PQC Key identifier                |
| operations| String        | Optional  | Allowed operations                |
| rateLimit | String        | Optional  | Rate limit configuration          |
| status    | String        | Optional  | Policy status                     |

#### Example Request

```json
{
  "name": "Production Security Policy",
  "authKeyId": "a100e8400-e29b-41d4-a716-446655440001",
  "pqcKeyId": "b200e8400-e29b-41d4-a716-446655440002",
  "operations": "Sign · Verify",
  "rateLimit": "1000/hour",
  "status": "active"
}
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
  "operations": "Sign · Verify",
  "rateLimit": "1000/hour",
  "status": "active"
}
```

#### Validation Rules

| Rule                              | Description                                                                 |
|-----------------------------------|-----------------------------------------------------------------------------|
| Policy Name Required              | Policy name cannot be empty                                                 |
| Active PQC Key Required           | Linked PQC key must be active                                               |
| Active Authentication Key Required| Linked Authentication key must be active                                    |
| Unique Key Pair                   | Only one policy may exist for a specific Authentication Key and PQC Key combination |

#### HTTP Status Codes

| Status Code               | Description                                             |
|---------------------------|---------------------------------------------------------|
| 201 Created               | Policy created successfully                             |
| 400 Bad Request           | Invalid PQC Key ID                                      |
| 400 Bad Request           | Invalid Authentication Key ID                           |
| 400 Bad Request           | Linked key is not active                                |
| 400 Bad Request           | Policy already exists for selected key combination      |
| 401 Unauthorized          | Invalid or missing authentication token                 |
| 500 Internal Server Error | Failed to create policy                                 |

---

### Update Policy

Update an existing policy.

#### Request

```
PUT /api/policies/{id}
```

#### Parameters

| Name      | Type          | Mandatory | Description                    |
|-----------|---------------|-----------|--------------------------------|
| name      | String        | Optional  | Updated policy name            |
| authKeyId | String (UUID) | Optional  | Updated Authentication Key     |
| pqcKeyId  | String (UUID) | Optional  | Updated PQC Key                |
| operations| String        | Optional  | Updated allowed operations     |
| rateLimit | String        | Optional  | Updated rate limit             |
| status    | String        | Optional  | Updated status                 |

#### Allowed Status Values

| Status   |
|----------|
| active   |
| disabled |

#### Example Request

```json
{
  "name": "Updated Production Policy",
  "status": "disabled"
}
```

#### Success Response

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Updated Production Policy",
  "authKey": "Primary Signing Key",
  "pqcKey": "Production PQC Key",
  "status": "disabled"
}
```

#### HTTP Status Codes

| Status Code               | Description                                               |
|---------------------------|-----------------------------------------------------------|
| 200 OK                    | Policy updated successfully                               |
| 400 Bad Request           | Invalid Authentication Key ID                             |
| 400 Bad Request           | Invalid PQC Key ID                                        |
| 400 Bad Request           | Linked key is not active                                  |
| 400 Bad Request           | Duplicate Authentication Key and PQC Key combination      |
| 401 Unauthorized          | Invalid or missing authentication token                   |
| 404 Not Found             | Policy not found                                          |
| 500 Internal Server Error | Failed to update policy                                   |

---

### Policy Object

A policy is represented using the following structure:

```json
{
  "id": "uuid",
  "name": "Production Security Policy",
  "authKey": "Primary Signing Key",
  "authKeyId": "uuid",
  "pqcKey": "Production PQC Key",
  "pqcKeyId": "uuid",
  "operations": "Sign · Verify",
  "rateLimit": "1000/hour",
  "status": "active",
  "createdAt": "2025-06-05T10:15:30.000Z",
  "updatedAt": "2025-06-05T10:15:30.000Z"
}
```

### Policy Validation Rules

The following validation checks are enforced before a policy is created or updated:

- Policy name must be provided.
- Authentication Key must belong to the authenticated user.
- PQC Key must belong to the authenticated user.
- Only active Authentication Keys can be linked.
- Only active PQC Keys can be linked.
- A unique policy must exist for every Authentication Key and PQC Key combination.
- Key references are automatically synchronized with linked key names.

### Security Notes

- Policies are isolated per user account.
- Policies cannot reference keys owned by other users.
- Disabled keys cannot be attached to policies.
- Policy updates automatically validate linked key ownership.
- Duplicate key combinations are prevented.
- Policy creation and modification events generate audit log entries.
- Policy references remain synchronized with Authentication Key and PQC Key names.
- Policy status changes are recorded in audit logs.

---

## 4. PQC Key Management API

Provides generation, management, storage, rotation, and lifecycle management of Post-Quantum Cryptographic (PQC) keys. The API supports post-quantum, classical, hybrid, and symmetric cryptographic algorithms for secure signing, verification, encryption, decryption, encapsulation, and decapsulation operations.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- All endpoints require authentication.
- PQC keys are managed per user account.
- Private key material is encrypted before storage.
- Hybrid algorithms combine classical and post-quantum cryptography.
- Key rotation is supported through a dedicated endpoint.
- Every PQC key operation is recorded in the audit log system.

### Authentication

All PQC Key Management endpoints require a valid JWT access token.

```
Authorization: Bearer <access_token>
```

### Supported Algorithms

| Algorithm   | Description                                      |
|-------------|--------------------------------------------------|
| ML-DSA      | Post-Quantum Digital Signature Algorithm         |
| ML-KEM      | Post-Quantum Key Encapsulation Mechanism         |
| ECDSA       | Classical Elliptic Curve Digital Signature Algorithm |
| Hybrid-DSA  | ML-DSA + ECDSA Hybrid Signature Algorithm        |
| Hybrid-KEM  | ML-KEM + X25519 Hybrid Key Exchange              |
| AES-256     | Symmetric Encryption Key                         |

### Supported Operations

Operations are automatically assigned based on the selected algorithm.

| Algorithm   | Supported Operations         |
|-------------|------------------------------|
| ML-DSA      | Sign, Verify                 |
| ECDSA       | Sign, Verify                 |
| Hybrid-DSA  | Sign, Verify                 |
| ML-KEM      | Encapsulate, Decapsulate     |
| Hybrid-KEM  | Encapsulate, Decapsulate     |
| AES-256     | Encrypt, Decrypt             |

---

### Get PQC Keys

Retrieve all PQC keys associated with the authenticated user.

#### Request

```
GET /api/pqc-keys
```

#### Success Response

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Production PQC Key",
    "algorithm": "ML-DSA",
    "parameters": "ML-DSA-65",
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

#### HTTP Status Codes

| Status Code               | Description                              |
|---------------------------|------------------------------------------|
| 200 OK                    | PQC keys retrieved successfully          |
| 401 Unauthorized          | Invalid or missing authentication token  |
| 500 Internal Server Error | Failed to retrieve PQC keys              |

---

### Create PQC Key

Generate and create a new PQC key.

#### Request

```
POST /api/pqc-keys
```

#### Parameters

| Name        | Type   | Mandatory | Description                          |
|-------------|--------|-----------|--------------------------------------|
| name        | String | Required  | PQC key name                         |
| algorithm   | String | Required  | Cryptographic algorithm              |
| environment | String | Optional  | `Development` or `Production`        |

#### Example Request

```json
{
  "name": "Production PQC Key",
  "algorithm": "ML-DSA",
  "environment": "Production"
}
```

#### Example Request (Hybrid)

```json
{
  "name": "Hybrid Signing Key",
  "algorithm": "Hybrid-DSA",
  "environment": "Production"
}
```

#### Example Request (AES-256)

```json
{
  "name": "Data Encryption Key",
  "algorithm": "AES-256"
}
```

#### Success Response

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Production PQC Key",
  "algorithm": "ML-DSA",
  "parameters": "ML-DSA-65",
  "operations": "Sign · Verify",
  "environment": "Production",
  "status": "active",
  "version": 1
}
```

#### Validation Rules

| Field     | Rule                                         |
|-----------|----------------------------------------------|
| name      | Must not be empty                            |
| algorithm | Must be one of the supported algorithms      |

#### HTTP Status Codes

| Status Code               | Description                                   |
|---------------------------|-----------------------------------------------|
| 201 Created               | PQC key created successfully                  |
| 400 Bad Request           | Invalid algorithm or duplicate active key name |
| 401 Unauthorized          | Invalid or missing authentication token       |
| 500 Internal Server Error | Failed to generate PQC key                   |

---

### Update PQC Key

Update an existing PQC key's name or status.

#### Request

```
PUT /api/pqc-keys/{id}
```

#### Parameters

| Name   | Type   | Mandatory | Description          |
|--------|--------|-----------|----------------------|
| name   | String | Optional  | Updated key name     |
| status | String | Optional  | Updated key status   |

#### Allowed Status Values

| Status   |
|----------|
| active   |
| rotated  |
| disabled |

#### Example Request

```json
{
  "name": "Updated Production Key"
}
```

#### Example Request (Status Update)

```json
{
  "status": "disabled"
}
```

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

#### HTTP Status Codes

| Status Code               | Description                              |
|---------------------------|------------------------------------------|
| 200 OK                    | PQC key updated successfully             |
| 400 Bad Request           | Invalid update parameters                |
| 401 Unauthorized          | Invalid or missing authentication token  |
| 404 Not Found             | PQC key not found                        |
| 500 Internal Server Error | Failed to update PQC key                 |

---

### Rotate PQC Key

Rotate an existing PQC key by creating a new version while preserving the same configuration.

#### Request

```
POST /api/pqc-keys/{id}/rotate
```

#### Description

Key rotation performs the following actions:

- Marks the existing key as `rotated`.
- Generates a new cryptographic key pair.
- Creates a new active key version.
- Updates associated policies.
- Records an audit log entry.

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

#### HTTP Status Codes

| Status Code               | Description                              |
|---------------------------|------------------------------------------|
| 200 OK                    | PQC key rotated successfully             |
| 401 Unauthorized          | Invalid or missing authentication token  |
| 404 Not Found             | PQC key not found                        |
| 500 Internal Server Error | Failed to rotate PQC key                 |

---

### PQC Key Object

A PQC key is represented using the following structure:

```json
{
  "id": "uuid",
  "name": "Production PQC Key",
  "algorithm": "ML-DSA",
  "parameters": "ML-DSA-65",
  "operations": "Sign · Verify",
  "environment": "Production",
  "status": "active",
  "version": 1,
  "createdAt": "2025-06-05T10:15:30.000Z",
  "updatedAt": "2025-06-05T10:15:30.000Z"
}
```

### Versioning

Each PQC key contains a version number.

| Version | Description          |
|---------|----------------------|
| 1       | Original key         |
| 2+      | Rotated key versions |

Every successful rotation increments the version number automatically.

### Key Rotation

Key rotation enables cryptographic agility and long-term security. During rotation:

- Existing keys are marked as `rotated`.
- New key material is generated.
- Associated policy references are updated automatically.
- Audit log records are created.
- Previous versions remain available for historical reference.

### Security Notes

- Private keys are encrypted before storage.
- Operations are automatically assigned based on algorithm selection.
- Users cannot manually modify algorithm-specific operations.
- Duplicate active key names are prevented.
- Hybrid algorithms provide classical and post-quantum protection simultaneously.
- AES-256 keys are generated for symmetric encryption operations.
- Key rotation supports cryptographic lifecycle management.
- All key creation, update, and rotation events are logged.
- Policy references are automatically synchronized during key updates and rotations.

---

## 5. Cryptographic Operations API

Provides secure cryptographic operations using Authentication Keys, Post-Quantum Cryptography (PQC) Keys, Policies, IP Whitelisting, Mutual TLS (mTLS), and Rate Limiting controls. All cryptographic operations are validated against active policies before execution.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- All cryptographic operations require authentication.
- All operations are validated against active policies.
- Authentication Keys and PQC Keys must both be active.
- Operations are audited and recorded in Audit Logs.
- IP Whitelisting restrictions are enforced before operation execution.
- Mutual TLS (mTLS) validation is enforced when enabled.
- Crypto endpoints are protected by rate limiting.

### Authentication

All Crypto endpoints require the following headers:

```
Authorization: Bearer <access_token>
x-pqc-key-id: <pqc_key_id>
x-auth-key-id: <auth_key_id>
```

### Security Controls

Before any cryptographic operation is executed, the platform validates:

- User authentication.
- Active policy exists.
- PQC key is active.
- Authentication key is active.
- Requested operation is permitted.
- IP address is whitelisted (if enabled).
- mTLS certificate requirements are satisfied.
- Rate limits have not been exceeded.

### Supported Operations

| Operation    | Description                           |
|--------------|---------------------------------------|
| Sign         | Generate digital signatures           |
| Verify       | Verify digital signatures             |
| Encapsulate  | Generate shared secrets using KEM     |
| Decapsulate  | Recover shared secrets                |
| Encrypt      | Symmetric encryption                  |
| Decrypt      | Symmetric decryption                  |

### Supported Algorithms

| Algorithm   | Supported Operations         |
|-------------|------------------------------|
| ML-DSA      | Sign, Verify                 |
| ECDSA       | Sign, Verify                 |
| Hybrid-DSA  | Sign, Verify                 |
| ML-KEM      | Encapsulate, Decapsulate     |
| Hybrid-KEM  | Encapsulate, Decapsulate     |
| AES-256     | Encrypt, Decrypt             |

---

### Sign Data

Generate a digital signature using the configured PQC or Hybrid signing key.

#### Request

```
POST /api/crypto/sign
```

#### Headers

| Header        | Required |
|---------------|----------|
| Authorization | Yes      |
| x-pqc-key-id  | Yes      |
| x-auth-key-id | Yes      |

#### Request Body

```json
{
  "data": "Hello World"
}
```

#### Success Response

```json
{
  "signature": "base64_signature_data"
}
```

#### HTTP Status Codes

| Status Code               | Description                        |
|---------------------------|------------------------------------|
| 200 OK                    | Signature generated successfully   |
| 400 Bad Request           | Missing parameters                 |
| 403 Forbidden             | Policy restriction                 |
| 403 Forbidden             | IP whitelist violation             |
| 403 Forbidden             | MTLS requirement failed            |
| 500 Internal Server Error | Signature generation failed        |

---

### Verify Signature

Verify a digital signature.

#### Request

```
POST /api/crypto/verify
```

#### Request Body

```json
{
  "data": "Hello World",
  "signature": "base64_signature_data"
}
```

#### Success Response

```json
{
  "isValid": true
}
```

#### HTTP Status Codes

| Status Code | Description                            |
|-------------|----------------------------------------|
| 200 OK      | Signature verification completed       |

---

### Encapsulate Secret

Generate a shared secret using a Key Encapsulation Mechanism (KEM).

#### Request

```
POST /api/crypto/encapsulate
```

#### Success Response (ML-KEM)

```json
{
  "ciphertext": "encapsulated_ciphertext",
  "sharedSecret": "shared_secret"
}
```

#### Success Response (Hybrid-KEM)

```json
{
  "ciphertext": {
    "pqc": "pqc_ciphertext",
    "classical": "classical_ciphertext"
  },
  "sharedSecret": {
    "pqc": "pqc_secret",
    "classical": "classical_secret"
  }
}
```

#### HTTP Status Codes

| Status Code | Description                              |
|-------------|------------------------------------------|
| 200 OK      | Shared secret generated successfully     |

---

### Decapsulate Secret

Recover a shared secret from encapsulated ciphertext.

#### Request

```
POST /api/crypto/decapsulate
```

#### Request Body

```json
{
  "ciphertext": "encapsulated_ciphertext"
}
```

#### Success Response

```json
{
  "sharedSecret": "shared_secret"
}
```

#### HTTP Status Codes

| Status Code | Description                              |
|-------------|------------------------------------------|
| 200 OK      | Shared secret recovered successfully     |

---

### Encrypt Data

Encrypt data using an AES-256 PQC key.

#### Request

```
POST /api/crypto/encrypt
```

#### Request Body

```json
{
  "data": "Sensitive Information"
}
```

#### Success Response

```json
{
  "ciphertext": "encrypted_data",
  "iv": "initialization_vector",
  "authTag": "authentication_tag"
}
```

#### HTTP Status Codes

| Status Code | Description                    |
|-------------|--------------------------------|
| 200 OK      | Data encrypted successfully    |

---

### Decrypt Data

Decrypt previously encrypted data.

#### Request

```
POST /api/crypto/decrypt
```

#### Request Body

```json
{
  "ciphertext": "encrypted_data",
  "iv": "initialization_vector",
  "authTag": "authentication_tag"
}
```

#### Success Response

```json
{
  "data": "Sensitive Information"
}
```

#### HTTP Status Codes

| Status Code | Description                    |
|-------------|--------------------------------|
| 200 OK      | Data decrypted successfully    |

---

### Policy Enforcement

Every cryptographic operation is validated against an active policy. A policy must satisfy:

- Authentication Key is active.
- PQC Key is active.
- Requested operation is listed in policy operations.
- Key algorithm supports the requested operation.

| Operation    | Valid Algorithms              |
|--------------|-------------------------------|
| Sign         | ML-DSA, ECDSA, Hybrid-DSA     |
| Verify       | ML-DSA, ECDSA, Hybrid-DSA     |
| Encapsulate  | ML-KEM, Hybrid-KEM            |
| Decapsulate  | ML-KEM, Hybrid-KEM            |
| Encrypt      | AES-256                       |
| Decrypt      | AES-256                       |

Operations attempted against unsupported algorithms are rejected.

### Security Notes

- All cryptographic operations generate audit log entries.
- IP whitelist restrictions are enforced before execution.
- MTLS certificate validation is enforced when enabled.
- Policy validation occurs before cryptographic processing.
- Disabled keys cannot be used.
- Rotated keys cannot be used unless referenced by an active policy.
- Hybrid algorithms provide both classical and post-quantum security.
- Shared secrets generated through KEM operations should be treated as highly sensitive.
- AES-256 encryption uses authenticated encryption mechanisms.
- Rate limiting protects cryptographic endpoints from abuse and denial-of-service attacks.

---

## 6. Audit Logs API

Provides audit log records associated with the authenticated user. Audit logs are used to track security-related and operational activities such as key management, policy management, cryptographic operations, authentication events, and platform activity.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- All endpoints require authentication.
- Audit logs are generated automatically by platform services.
- Audit logs are returned in descending order of creation time (newest first).
- Audit log records cannot be modified through the API.

### Authentication

All Audit Log endpoints require a valid JWT access token.

```
Authorization: Bearer <access_token>
```

---

### Get Audit Logs

Retrieve all audit log entries associated with the authenticated user.

#### Request

```
GET /api/audit-logs
```

#### Headers

| Name          | Mandatory | Description         |
|---------------|-----------|---------------------|
| Authorization | Required  | JWT access token    |

> This endpoint does not require path parameters, query parameters, or request body data.

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

#### Response Fields

| Field     | Type          | Description                                    |
|-----------|---------------|------------------------------------------------|
| id        | Integer       | Unique audit log identifier                    |
| operation | String        | Operation or event name                        |
| authKey   | String        | Associated Authentication Key name             |
| authKeyId | String / Null | Authentication Key identifier                  |
| pqcKey    | String        | Associated PQC Key name                        |
| pqcKeyId  | String / Null | PQC Key identifier                             |
| policyId  | String / Null | Associated Policy identifier                   |
| sourceIP  | String        | Source IP address                              |
| result    | String        | Operation result (`success` or `failure`)      |
| createdAt | Integer       | Unix timestamp in milliseconds                 |
| timestamp | String        | ISO-8601 formatted timestamp                   |

#### HTTP Status Codes

| Status Code               | Description                              |
|---------------------------|------------------------------------------|
| 200 OK                    | Audit logs retrieved successfully        |
| 401 Unauthorized          | Invalid or missing authentication token  |
| 500 Internal Server Error | Failed to retrieve audit logs            |

---

### Audit Log Object

An audit log entry is represented using the following structure:

```json
{
  "id": 101,
  "operation": "pqc_sign",
  "authKey": "Primary Signing Key",
  "authKeyId": "uuid",
  "pqcKey": "Production PQC Key",
  "pqcKeyId": "uuid",
  "policyId": "uuid",
  "sourceIP": "192.168.1.100",
  "result": "success",
  "createdAt": 1749112345678,
  "timestamp": "2025-06-05T10:32:25.678Z"
}
```

### Filtering Rules

**Records always returned:**

- Successful audit log entries.
- Manual audit log entries.
- Standard platform activity logs.
- Key management events.
- Policy management events.
- Cryptographic operation events.

**Failure Log Filtering:**

Failure logs prefixed with `FAILED:` are only returned when related to the following platform modules:

- `/api/pqc-keys`
- `/api/auth-keys`
- `/api/policies`
- `/api/crypto`

Failure logs outside these modules are excluded from the response.

### Sorting Rules

Audit logs are sorted by creation date in **descending order** (newest → oldest).

**Example order:**
```
2025-06-05T10:32:25.678Z
2025-06-05T10:14:05.678Z
2025-06-05T09:58:17.000Z
```

### Security Notes

- Audit logs are isolated per user account.
- Users can only access their own audit records.
- Audit records are immutable through the API.
- Authentication Key operations generate audit records.
- PQC Key operations generate audit records.
- Policy management events generate audit records.
- Cryptographic operations generate audit records.
- IP whitelist enforcement events may generate audit records.
- MTLS validation events may generate audit records.
- Audit logs support security monitoring, compliance reporting, and forensic investigations.

---

## 7. Dashboard API

Provides dashboard statistics and summary metrics associated with the authenticated user. The Dashboard API aggregates information from Authentication Keys, Post-Quantum Cryptography (PQC) Keys, Policies, and Audit Logs to provide a high-level overview of platform activity and resource utilization.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- All endpoints require authentication.
- Dashboard statistics are calculated dynamically.
- Statistics are generated using user-specific resources and audit records.
- Dashboard data provides an overview of cryptographic assets and platform activity.

### Authentication

All Dashboard API endpoints require a valid JWT access token.

```
Authorization: Bearer <access_token>
```

### Dashboard Metrics

The Dashboard API aggregates the following information:

| Metric              | Description                                                  |
|---------------------|--------------------------------------------------------------|
| PQC Keys            | Total number of PQC Keys owned by the user                   |
| Authentication Keys | Total number of Authentication Keys owned by the user        |
| Policies            | Total number of Policies owned by the user                   |
| Total Operations    | Total number of audit log entries recorded                   |
| Successful Events   | Total number of successful audit log events                  |

---

### Get Dashboard Statistics

Retrieve aggregated dashboard statistics for the authenticated user.

#### Request

```
GET /api/dashboard/stats
```

#### Headers

| Name          | Mandatory | Description         |
|---------------|-----------|---------------------|
| Authorization | Required  | JWT access token    |

> This endpoint does not require path parameters, query parameters, or request body data.

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

#### Response Fields

| Field            | Type    | Description                                    |
|------------------|---------|------------------------------------------------|
| pqcKeys          | Integer | Total number of PQC Keys                       |
| authKeys         | Integer | Total number of Authentication Keys            |
| policies         | Integer | Total number of Policies                       |
| totalOperations  | Integer | Total number of recorded audit log events      |
| successfulEvents | Integer | Total number of successful audit log events    |

#### HTTP Status Codes

| Status Code               | Description                                        |
|---------------------------|----------------------------------------------------|
| 200 OK                    | Dashboard statistics retrieved successfully        |
| 401 Unauthorized          | Invalid or missing authentication token            |
| 500 Internal Server Error | Failed to retrieve dashboard statistics            |

---

### Dashboard Statistics Object

A dashboard statistics record is represented using the following structure:

```json
{
  "pqcKeys": 8,
  "authKeys": 4,
  "policies": 6,
  "totalOperations": 245,
  "successfulEvents": 231
}
```

#### Field Definitions

| Field            | Description                                                             |
|------------------|-------------------------------------------------------------------------|
| pqcKeys          | Count of PQC Keys associated with the authenticated user                |
| authKeys         | Count of Authentication Keys associated with the authenticated user     |
| policies         | Count of Policies associated with the authenticated user                |
| totalOperations  | Count of audit log records associated with the authenticated user       |
| successfulEvents | Count of audit log records with a `success` result                      |

### Security Notes

- Dashboard statistics are isolated per user account.
- Users can only access their own dashboard data.
- Statistics are generated from live platform records.
- Dashboard data includes counts only and does not expose private cryptographic material.
- Authentication is required for all dashboard operations.
- Dashboard statistics may be used for monitoring platform usage and activity trends.
- Audit log records contribute directly to operational statistics.
- All calculations are performed server-side before being returned to the client.

---

## 8. Contact API

Provides a public interface for submitting contact and support requests. The API validates user information and securely forwards the request to an external support service for processing and response management.

### General API Information

- The base endpoint depends on the application deployment environment.
- All endpoints return JSON responses.
- Contact requests are forwarded to an external support service.
- Contact request submissions generate audit log records.
- Contact requests are processed immediately after validation.
- The API acts as a secure gateway between users and the external support platform.

### Authentication

The Contact API is **publicly accessible**. Authentication is not required.

---

### Submit Contact Request

Submit a contact or support request.

#### Request

```
POST /api/contact
```

#### Parameters

| Name     | Type   | Mandatory | Description                         |
|----------|--------|-----------|-------------------------------------|
| fullName | String | Required  | Full name of the requester          |
| email    | String | Required  | Email address of the requester      |
| message  | String | Optional  | Contact request message             |
| subject  | String | Optional  | Request subject                     |
| phone    | String | Optional  | Contact phone number                |

#### Example Request

```json
{
  "fullName": "John Doe",
  "email": "john.doe@example.com",
  "subject": "Technical Support Request",
  "message": "I need assistance with my account.",
  "phone": "+1-555-123-4567"
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

#### Response Fields

| Field   | Type   | Description                                               |
|---------|--------|-----------------------------------------------------------|
| message | String | Status message                                            |
| data    | Object | Response returned from the external support service       |

#### HTTP Status Codes

| Status Code | Description                               |
|-------------|-------------------------------------------|
| 200 OK      | Contact request submitted successfully    |

---

### Contact Request Object

A contact request may contain the following information:

```json
{
  "fullName": "John Doe",
  "email": "john.doe@example.com",
  "subject": "Technical Support Request",
  "message": "I need assistance with my account.",
  "phone": "+1-555-123-4567"
}
```

**Required Fields:**

| Field    | Required |
|----------|----------|
| fullName | Yes      |
| email    | Yes      |

**Optional Fields:**

| Field   |
|---------|
| subject |
| message |
| phone   |

> Additional fields may be accepted and forwarded to the external support service without modification.

---

### Error Responses (Contact)

**Missing Required Fields**

```json
{
  "error": "Full Name and Email are required"
}
```

| Status Code     | Description                      |
|-----------------|----------------------------------|
| 400 Bad Request | Required fields are missing      |

**External Service Failure**

```json
{
  "error": "Failed to contact external service"
}
```

| Status Code     | Description                                  |
|-----------------|----------------------------------------------|
| 502 Bad Gateway | External support service is unavailable      |

**External API Error**

```json
{
  "error": "External API Error",
  "details": {
    "message": "Invalid request"
  }
}
```

| Status Code  | Description                                    |
|--------------|------------------------------------------------|
| 4xx / 5xx   | Error returned by external support service     |

**Server Configuration Error**

```json
{
  "error": "Server configuration error (Missing API URL)"
}
```

| Status Code               | Description                                         |
|---------------------------|-----------------------------------------------------|
| 500 Internal Server Error | Contact service configuration is incomplete         |

---

### Security Notes

- Contact requests are forwarded using secure API authentication.
- External API communication uses API Key and Bearer Token authentication.
- Request payloads are forwarded without modification.
- Successful submissions generate audit log entries.
- Sensitive configuration values are stored in environment variables.
- Contact requests are processed through a centralized external support service.
- Authentication is not required for submitting contact requests.
