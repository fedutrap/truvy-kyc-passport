# TruVy KYC API — Reference Guide

**Base URL (Production):** `https://truvy-kyc-passport-production.up.railway.app`  
**Base URL (Local):** `http://localhost:3000`

All requests/responses use JSON unless otherwise noted. No authentication required for public endpoints.

---

## Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Service health check |
| `GET` | `/public-key` | Retrieve RSA-2048 public key (PEM) |
| `POST` | `/issue` | Issue a credential from manual form fields |
| `POST` | `/issue-from-document` | Upload a document → Claude Vision extracts fields → issues credential |
| `POST` | `/verify` | Verify a JWT credential signature |
| `GET` | `/verify-qr/:sessionId/:token` | Verify a credential via QR code scan |
| `GET` | `/status/:sessionId` | Poll the status of a verification session |
| `POST` | `/ai-score` | Run Claude AI risk scoring on an ID document image |

---

## GET `/health`

Returns service status and lists all available endpoints.

**Request:**
```bash
curl https://truvy-kyc-passport-production.up.railway.app/health
```

**Response:**
```json
{
  "status": "ok",
  "service": "TruVy KYC API",
  "version": "2.1.0-agefix",
  "endpoints": [
    "POST /issue",
    "POST /issue-from-document",
    "POST /verify",
    "GET /status/:id",
    "POST /ai-score",
    "GET /public-key"
  ],
  "timestamp": "2026-04-01T12:00:00.000Z"
}
```

---

## GET `/public-key`

Returns the RSA-2048 public key used to verify credential JWT signatures. Banks and verifiers call this to validate credentials without contacting TruVy on every verification.

**Request:**
```bash
curl https://truvy-kyc-passport-production.up.railway.app/public-key
```

**Response:**
```json
{
  "publicKey": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkq...\n-----END PUBLIC KEY-----\n"
}
```

> **Note:** The public key is generated fresh on every server restart. In production, this would be persisted and versioned.

---

## POST `/issue`

Issues a signed JWT credential from manually submitted identity fields (e.g. from a web form or Persona verification callback).

**Request Body:**
```json
{
  "name": "Maria Silva",
  "country": "Brazil",
  "documentType": "driver's license",
  "dateOfBirth": "2000-05-15"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | ✅ | — | Full legal name |
| `country` | string | ✅ | — | Issuing country |
| `documentType` | string | ❌ | `"driver's license"` | Document type used for KYC |
| `dateOfBirth` | string | ❌ | — | Used only to compute `ageVerified`; never stored or transmitted |

**Response:**
```json
{
  "sessionId": "sess_1711234567890_abc123",
  "token": "eyJhbGciOiJSUzI1NiJ9...",
  "qrBase64": "data:image/png;base64,...",
  "sharedClaims": {
    "name": "Maria Silva",
    "country": "Brazil",
    "documentType": "driver's license",
    "sanctionsCheck": "PASSED",
    "ageVerified": "21+",
    "issuer": "did:kycp:legitimuz",
    "issuedAt": "2026-04-01T12:00:00.000Z"
  },
  "message": "Credential issued. Zero raw documents transmitted."
}
```

---

## POST `/issue-from-document`

Accepts a real ID document (photo or PDF). Claude Vision extracts identity fields and issues a signed credential. **Zero raw document data is forwarded to any bank.**

**Request:** `multipart/form-data`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `document` | file | ✅ | JPG, PNG, or PDF — max 10MB |
```bash
curl -X POST https://truvy-kyc-passport-production.up.railway.app/issue-from-document \
  -F "document=@/path/to/drivers_license.jpg"
```

**Response (success):**
```json
{
  "sessionId": "sess_1711234567890_xyz789",
  "token": "eyJhbGciOiJSUzI1NiJ9...",
  "qrBase64": "data:image/png;base64,...",
  "sharedClaims": {
    "name": "John Doe",
    "country": "United States",
    "documentType": "driver's license",
    "sanctionsCheck": "PASSED",
    "ageVerified": "21+",
    "issuer": "did:kycp:legitimuz",
    "issuedAt": "2026-04-01T12:00:00.000Z",
    "extractionConfidence": 94
  },
  "documentFields": {
    "name": "John Doe",
    "country": "United States",
    "documentType": "driver's license",
    "documentNumber": "AB*******",
    "dateOfBirth": "**/**/**** (age verified)",
    "expiryDate": "2030-01-15",
    "extractionConfidence": 94
  },
  "withheldFromBanks": [
    "documentNumber",
    "dateOfBirth",
    "homeAddress",
    "taxId",
    "rawDocumentImage"
  ],
  "message": "Document scanned. Credential issued. Zero raw documents will be transmitted to any bank."
}
```

**Response (unreadable document — 422):**
```json
{
  "error": "Document could not be read",
  "readable": false,
  "detail": "The document was unclear or did not contain recognizable identity fields.",
  "suggestion": "Make sure the ID is flat, well-lit, and fully visible in the frame."
}
```

---

## POST `/verify`

Verifies a TruVy JWT credential. Called by banks and institutions to validate a user's credential. Returns only safe, privacy-preserving claims. **No raw document data is ever returned.**

**Request Body:**
```json
{
  "token": "eyJhbGciOiJSUzI1NiJ9...",
  "sessionId": "sess_1711234567890_abc123"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | ✅ | JWT credential to verify |
| `sessionId` | string | ❌ | If provided, marks the session as verified |

**Response (valid):**
```json
{
  "valid": true,
  "claims": {
    "name": "Maria Silva",
    "country": "Brazil",
    "documentType": "driver's license",
    "sanctionsCheck": "PASSED",
    "ageVerified": "21+",
    "issuer": "did:kycp:legitimuz",
    "issuedAt": "2026-04-01T12:00:00.000Z"
  },
  "withheld": [
    "documentNumber",
    "dateOfBirth",
    "homeAddress",
    "taxId",
    "rawDocumentImage"
  ],
  "message": "Credential verified. Documents received: NONE."
}
```

**Response (invalid/tampered — 401):**
```json
{
  "valid": false,
  "error": "invalid signature",
  "message": "Credential rejected. Cannot forge a TruVy Passport."
}
```

---

## GET `/verify-qr/:sessionId/:token`

Used when a QR code is scanned by a bank's verification terminal. Validates the credential and marks the session as verified.

**Example:**
```
GET /verify-qr/sess_1711234567890_abc123/eyJhbGciOiJSUzI1NiJ9...
```

**Response:**
```json
{
  "valid": true,
  "message": "QR scan verified. Session marked complete.",
  "sessionId": "sess_1711234567890_abc123"
}
```

---

## GET `/status/:sessionId`

Polls the status of a verification session. The frontend uses this to know when a QR scan has been completed.

**Example:**
```bash
curl https://truvy-kyc-passport-production.up.railway.app/status/sess_1711234567890_abc123
```

**Response (pending):**
```json
{
  "sessionId": "sess_1711234567890_abc123",
  "status": "pending",
  "claims": null
}
```

**Response (verified):**
```json
{
  "sessionId": "sess_1711234567890_abc123",
  "status": "verified",
  "claims": {
    "name": "Maria Silva",
    "country": "Brazil",
    "documentType": "driver's license",
    "sanctionsCheck": "PASSED",
    "ageVerified": "21+",
    "issuer": "did:kycp:legitimuz",
    "issuedAt": "2026-04-01T12:00:00.000Z"
  }
}
```

---

## POST `/ai-score`

Sends an ID document image to Claude Vision for AI-powered authenticity and risk scoring. Returns a risk assessment without storing or transmitting the image.

**Request Body:**
```json
{
  "imageBase64": "<base64-encoded image string>",
  "mimeType": "image/jpeg"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `imageBase64` | string | ❌ | — | Base64-encoded JPG or PNG. If omitted, returns a demo score. |
| `mimeType` | string | ❌ | `"image/jpeg"` | MIME type of the image |

**Response:**
```json
{
  "riskScore": 12,
  "riskLevel": "LOW",
  "documentAuthenticity": "LIKELY_GENUINE",
  "flags": [],
  "recommendation": "APPROVE",
  "confidence": 94
}
```

| Field | Values | Description |
|-------|--------|-------------|
| `riskScore` | 0–100 | Lower = safer |
| `riskLevel` | `LOW`, `MEDIUM`, `HIGH` | Risk classification |
| `documentAuthenticity` | `LIKELY_GENUINE`, `SUSPICIOUS`, `UNREADABLE` | Claude's assessment |
| `recommendation` | `APPROVE`, `REVIEW`, `REJECT` | Suggested action |
| `confidence` | 0–100 | Claude's confidence in the assessment |

---

## Age Verification Logic

The `ageVerified` field is computed from `dateOfBirth` at issuance time. The actual date of birth is **never stored or transmitted**.

| Age at Issuance | `ageVerified` Value |
|-----------------|---------------------|
| 21 or older | `"21+"` |
| 18–20 | `"18+"` |
| Under 18 | `"under18"` |
| Unknown / masked | `"18+"` (safe default) |

---

## Privacy & Security

- **RSA-2048** asymmetric JWT signing — no symmetric secrets shared with verifiers
- **Zero raw documents** are ever transmitted to banks or verifying institutions
- `documentNumber` and `dateOfBirth` are always masked by Claude before being stored in the credential
- Credentials expire after **7 days** (`expiresIn: "7d"`)
- Tampered or forged tokens are rejected instantly via signature validation

---

## Error Reference

| HTTP Status | Meaning |
|-------------|---------|
| `400` | Bad request — missing required fields |
| `401` | Invalid or tampered JWT signature |
| `404` | Session not found |
| `422` | Document unreadable or no identity fields detected |
| `500` | Internal server error |

---

## Running Locally
```bash
git clone https://github.com/fedutrap/truvy-kyc-passport.git
cd truvy-kyc-passport
npm install
echo "ANTHROPIC_API_KEY=your_key_here" > .env
node server.js
# API available at http://localhost:3000
```

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | ✅ | Anthropic API key for Claude Vision document extraction and AI scoring |
| `PORT` | ❌ | Server port (default: `3000`) |
| `BASE_URL` | ❌ | Public URL for QR code links (default: `http://localhost:3000`) |

---

*TruVy KYC API — Built at USF Hackathon · March 2026*
