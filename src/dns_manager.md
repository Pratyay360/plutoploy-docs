# dns-manager

Automated ACME DNS-01 certificate issuance via CNAME delegation. Users configure a single CNAME record per domain; the service handles challenge serving and certificate issuance automatically.

## Quick Start

```bash
# Set required env vars
export ACME_ACCOUNT_EMAIL="you@example.com"
export ACME_DNS_ZONE="auth.example.com"
export ACME_PUBLIC_IP="1.2.3.4"

# Run
go run .
```

## Configuration

| Env Var | Default | Description |
|---------|---------|-------------|
| `ACME_ACCOUNT_EMAIL` | — | Email for ACME account registration |
| `ACME_DIRECTORY_URL` | Let's Encrypt production | ACME directory URL |
| `ACME_DNS_ZONE` | — | Delegation zone (e.g. `auth.example.com`) |
| `ACME_DNS_LISTEN` | `:53` | DNS server listen address |
| `ACME_DNS_NS` | `ns1.<zone>` | Nameserver hostname |
| `ACME_PUBLIC_IP` | — | A record for NS/zone apex |
| `LISTEN_ADDR` | `:8080` | HTTP API listen address |
| `DB_PATH` | `dns-manager.db` | SQLite database path |

## How It Works

1. Your DNS hosts `auth.example.com` with NS pointing to this server
2. User creates one CNAME: `_acme-challenge.example.com` → `example-com.auth.example.com`
3. When the CA validates, it follows the CNAME to your authoritative DNS server
4. The server serves the ACME challenge TXT record from its in-memory store
5. Certificate is issued and persisted to SQLite

## API

All endpoints are under `/acme`.

### Health Check

```
GET /health
```

```json
{"message": "ok"}
```

### Register Account

```
POST /acme/register
```

Registers an ACME account with the CA. Only needed once per deployment (saved to DB).

**Request (optional body):**
```json
{"key_type": "RSA4096"}
```

Key types: `EC256`, `RSA4096` (default).

**Response:**
```json
{
  "email": "you@example.com",
  "uri": "https://acme-v02.api.letsencrypt.org/acme/acct/12345678",
  "ca": "https://acme-v02.api.letsencrypt.org/directory"
}
```

### Link Domain

```
POST /acme/setup
```

Returns the CNAME record the user must configure.

**Request:**
```json
{"domain": "example.com"}
```

**Response:**
```json
{
  "domain": "example.com",
  "cname": {
    "name": "_acme-challenge.example.com",
    "type": "CNAME",
    "value": "example-com.auth.example.com"
  },
  "instructions": "Create this CNAME record once..."
}
```

### Issue Certificate

```
POST /acme/issue
```

Runs the DNS-01 flow. The domain must have its CNAME configured and resolvable.

**Request:**
```json
{"domain": "example.com"}
```

**Response (success):**
```json
{
  "domain": "example.com",
  "status": "valid",
  "certificate": "-----BEGIN CERTIFICATE-----\n...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...",
  "issuer_certificate": "...",
  "cert_url": "https://acme-v02.api.letsencrypt.org/cert/..."
}
```

**Response (failure):**
```json
{
  "domain": "example.com",
  "status": "invalid",
  "error": "acme: ..."
}
```

### List Domains

```
GET /acme/domains
```

**Response:**
```json
[
  {
    "id": 1,
    "domain": "example.com",
    "cname_target": "example-com.auth.example.com",
    "created_at": "2026-07-01T12:00:00Z"
  }
]
```

### List Certificates

```
GET /acme/certs
GET /acme/certs?domain=example.com
```

**Response:**
```json
[
  {
    "id": 1,
    "domain": "example.com",
    "certificate": "-----BEGIN CERTIFICATE-----\n...",
    "private_key": "-----BEGIN PRIVATE KEY-----\n...",
    "issuer_cert": "...",
    "cert_url": "https://...",
    "status": "valid",
    "issued_at": "2026-07-01T12:05:00Z"
  }
]
```

## Setup Guide

### 1. Delegate DNS

Point your domain's `_acme-challenge` record to this service's delegation zone:

```
_acme-challenge.example.com.  CNAME  example-com.auth.example.com.
```

This is a one-time setup per domain. No changes needed for renewals.

### 2. Register and Issue

```bash
# Register account
curl -X POST http://localhost:8080/acme/register \
  -H "Content-Type: application/json" \
  -d '{}'

# Get CNAME instructions
curl -X POST http://localhost:8080/acme/setup \
  -H "Content-Type: application/json" \
  -d '{"domain": "example.com"}'

# Issue certificate
curl -X POST http://localhost:8080/acme/issue \
  -H "Content-Type: application/json" \
  -d '{"domain": "example.com"}'
```

### 3. Verify

```bash
# Check saved account
curl http://localhost:8080/acme/account

# List linked domains
curl http://localhost:8080/acme/domains

# List issued certificates
curl http://localhost:8080/acme/certs
```

## Database

SQLite database stores:

- **accounts** — ACME account credentials (email, URI, private key)
- **domains** — Linked domains and their CNAME targets
- **certificates** — Issued certificates, keys, and metadata

The account private key is persisted so you don't need to re-register after a restart. Saved accounts are automatically loaded on startup.

## Building

```bash
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o dns-manager .
```

Cross-compile:

```bash
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o dns-manager-linux-arm64 .
```
