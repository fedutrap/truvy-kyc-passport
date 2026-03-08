# TruVy — Verify Once. Use Anywhere.

> **Portable, privacy-first identity credentials for global finance.**  
> Built at the USF Hackathon · March 2026

[![Live Demo](https://img.shields.io/badge/Live%20Demo-truvy.lovable.app-00ff88?style=for-the-badge)](https://truvy.lovable.app)
[![API](https://img.shields.io/badge/API-Railway-blueviolet?style=for-the-badge)](https://truvy-kyc-passport-production.up.railway.app/health)
[![Tests](https://img.shields.io/badge/Tests-6%2F6%20Passing-00ff88?style=for-the-badge)](#testing)

---

## The Problem

Every time a person opens a bank account, they re-upload their passport from scratch. Banks spend **$72.9M/firm/year** on KYC (Fenergo 2025). **70% of banks lose clients** to slow onboarding. **1.4 billion people** remain unbanked globally (World Bank Findex 2022).

### The Real Story: Maria

Maria verifies her identity with **Nubank in Brazil** — full KYC, passport scan, sanctions check.  
Six months later she moves to New York and needs a US bank account.  
With TruVy, she opens a **Bank of America account in 2 minutes**. No paperwork. No re-verification.  
Documents received by BofA: **NONE**.

---

## The Solution

TruVy is the **Visa network for identity**. Jumio/Nubank/any issuer is the bank that issues the card.

```
Verify Once → Credential Issued → Share Proof → Account Opened
```

1. **Verify** — User verifies identity with any TruVy-connected issuer (e.g. Nubank)
2. **Issue** — TruVy issues a cryptographically signed JWT credential (RSA-2048)
3. **Share** — User selectively shares zero-knowledge proofs to any institution
4. **Accept** — Bank verifies signature in milliseconds. Zero raw documents transmitted.

---

## Hackathon Pillars

| Pillar | How TruVy Addresses It |
|--------|----------------------|
| 🌍 **Financial Inclusion** | 1.4B unbanked gain portable credentials usable at any bank worldwide |
| ⚡ **Cross-Border Efficiency** | Eliminate redundant KYC across borders — verify once, use anywhere |
| 🔒 **Security** | RSA-2048 signed JWTs, zero raw document transmission, sanctions screening |

---

## Architecture

```
┌─────────────────┐     ┌──────────────────────────┐     ┌─────────────────┐
│  React Frontend │────▶│   TruVy Backend (Node.js) │────▶│  Claude Vision  │
│  (Lovable/Vite) │     │   Railway Production      │     │  (Doc Extraction│
└─────────────────┘     └──────────────────────────┘     └─────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │   RSA-2048 Signing  │
                         │   JWT Credentials   │
                         │   Sanctions Check   │
                         │   Age Verification  │
                         └─────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + Vite + TailwindCSS (via Lovable) |
| Backend | Node.js + Express |
| Cryptography | RSA-2048 JWT signing (`jsonwebtoken`) |
| Document AI | Anthropic Claude Vision (claude-sonnet) |
| Deployment | Railway (backend) + Lovable (frontend) |
| Testing | Node.js built-in test runner |

---

## API Endpoints

**Base URL:** `https://truvy-kyc-passport-production.up.railway.app`

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Service health check |
| `POST` | `/issue` | Issue credential from form fields |
| `POST` | `/issue-from-document` | Upload doc → Claude Vision extracts → issues credential |
| `POST` | `/verify` | Verify JWT credential signature |
| `GET` | `/status/:id` | Poll session status |
| `POST` | `/ai-score` | Claude Vision risk scoring |
| `GET` | `/public-key` | Returns RSA public key (PEM) |

### Example: Issue a Credential

```bash
curl -X POST https://truvy-kyc-passport-production.up.railway.app/issue \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Maria Silva",
    "country": "Brazil",
    "documentType": "passport",
    "dateOfBirth": "2000-01-01"
  }'
```

**Response:**
```json
{
  "token": "eyJhbGciOiJSUzI1NiJ9...",
  "sharedClaims": {
    "name": "MARIA SILVA",
    "country": "BRAZIL",
    "documentType": "PASSPORT",
    "sanctionsCheck": "PASSED",
    "ageVerified": "21+",
    "issuer": "did:kycp:legitimuz"
  }
}
```

### Example: Verify a Credential

```bash
curl -X POST https://truvy-kyc-passport-production.up.railway.app/verify \
  -H "Content-Type: application/json" \
  -d '{"token": "<JWT_TOKEN>"}'
```

### Age Verification Logic

| Date of Birth | `ageVerified` Returned |
|--------------|----------------------|
| 21+ years old | `"21+"` |
| 18–20 years old | `"18+"` |
| Under 18 | `"under18"` |

---

## Running Locally

### Backend

```bash
# Clone the repo
git clone https://github.com/josebleal/truvy-kyc-passport.git
cd truvy-kyc-passport

# Install dependencies
npm install

# Set environment variables
echo "ANTHROPIC_API_KEY=your_key_here" > .env

# Start server
node server.js
# Server runs on http://localhost:3000
```

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | ✅ | Anthropic API key for Claude Vision |
| `PORT` | Optional | Server port (default: 3000) |

---

## Testing

6 automated tests — all passing ✅

```bash
node test.js
```

| Test | Status |
|------|--------|
| Health check endpoint | ✅ |
| Issue credential (manual fields) | ✅ |
| Issue from document (Claude Vision) | ✅ |
| Verify valid credential | ✅ |
| Reject tampered/forged credential | ✅ |
| Age calculation accuracy | ✅ |

---

## Security

- **RSA-2048** asymmetric signing — same standard as online banking
- **Zero raw documents** transmitted to verifying institutions
- **Selective disclosure** — banks receive only what they need (name, country, sanctions status, age)
- **Forgery-proof** — tampered tokens are rejected instantly via signature validation
- **FATF compliant** design principles
- **NIST IAL2** aligned

---

## Project Structure

```
truvy-kyc-passport/
├── server.js          # Main Express server + all endpoints
├── test.js            # Automated test suite (6 tests)
├── package.json
├── .env               # ANTHROPIC_API_KEY (not committed)
└── README.md
```

**Frontend** (separate repo via Lovable):
- Live at: https://truvy.lovable.app
- Pages: Home · Scan ID · Issue · User Wallet · Any Bank — Verify

---

## Business Model

| Revenue Stream | Model |
|---------------|-------|
| Banks (verifiers) | Per-verification SaaS fee ($0.10–$0.50/verification) |
| Identity Issuers | Revenue share for network participation |
| Enterprise | Annual license for white-label credential issuance |

**TAM:** $72.9M/firm/year KYC spend × thousands of financial institutions worldwide.

---

## Team

| Name | Role |
|------|------|
| **Jose Bleal** | Product & Pitch |
| **Andreas** | Brand & Design |
| **Felipe Dutra Paludetto** | Tech Lead — Backend, API, Deployment |
| **Pedro** | Engineering — GitHub, Testing, Devpost |

---

## Live Links

| Resource | URL |
|----------|-----|
| 🌐 Frontend Demo | https://truvy.lovable.app |
| ⚙️ Backend API | https://truvy-kyc-passport-production.up.railway.app |
| 📋 Health Check | https://truvy-kyc-passport-production.up.railway.app/health |
| 💻 GitHub (Backend) | https://github.com/josebleal/truvy-kyc-passport |

---

## Stats & Sources

| Stat | Source |
|------|--------|
| $72.9M/firm/year on KYC | Fenergo 2025 Financial Crime Industry Trends Report |
| 1.4B unbanked globally | World Bank Global Findex 2022 |
| 70% of banks lost clients to slow onboarding | Fenergo 2025 Financial Crime Report |
| 95 days average KYC review time | Fenergo KYC Trends 2023 |

---

*Built with ❤️ at USF Hackathon · March 2026 · "Verify Once. Use Anywhere."*
