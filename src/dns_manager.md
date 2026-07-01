# DNS Certificate Manager

A Go-based ACME certificate manager that automates TLS certificate issuance using DNS-01 challenges via an authoritative DNS server.

## Overview

This service provides fully automated ACME DNS-01 certificate issuance by:

1. Running an authoritative DNS server that serves ACME challenge TXT records
2. Exposing an HTTP API for certificate management operations
3. Leveraging CNAME delegation so domains only need one-time DNS configuration

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DNS Certificate Manager                   │
├─────────────────────────────────────────────────────────────┤
│  HTTP API (Gin)              │  Authoritative DNS Server    │
│  ─────────────               │  ────────────────────────    │
│  POST /acme/register         │  Serves TXT records for      │
│  POST /acme/setup            │  _acme-challenge delegation  │
│  POST /acme/issue            │  Answers SOA/NS/A queries    │
│  GET  /acme/account          │  for the delegation zone     │
├─────────────────────────────────────────────────────────────┤
│              ACME Client (lego) + AutoDNSProvider           │
└─────────────────────────────────────────────────────────────┘
```

### How It Works

1. **One-time CNAME Setup**: Domain owner creates a CNAME record:
   ```
   _acme-challenge.example.com.  CNAME  example.com.auth.example.com.
   ```

2. **Certificate Issuance**: When issuance is requested:
   - ACME client requests a DNS-01 challenge
   - `AutoDNSProvider` writes the TXT value to the authoritative DNS server's `TXTStore`
   - CA queries the domain, follows the CNAME to your authoritative server
   - Authoritative server responds with the TXT challenge value
   - CA validates and issues the certificate

3. **No Further Changes**: Renewals use the same CNAME—no manual DNS updates needed.

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `ACME_DIRECTORY_URL` | ACME directory URL (Let's Encrypt, ZeroSSL, etc.) | Let's Encrypt Production |
| `ACME_ACCOUNT_EMAIL` | Email for ACME account registration | - |
| `ACME_DNS_ZONE` | Delegation zone (e.g., `auth.example.com`) | - |
| `ACME_DNS_LISTEN` | DNS server listen address | `:53` |
| `ACME_DNS_NS` | Nameserver hostname (SOA/NS) | `ns1.<zone>` |
| `ACME_PUBLIC_IP` | Public IP for A record (NS/zone apex) | - |
| `LISTEN_ADDR` | HTTP server listen address | `:8080` |

## API Endpoints

### Health Check

```http
GET /health
```

```json
{"message": "ok"}
```

### Register ACME Account

```http
POST /acme/register
Content-Type: application/json

{
  "key_type": "EC256"  // Optional: EC256 (default) or RSA4096
}
```

```json
{
  "email": "user@example.com",
  "uri": "https://acme-v02.api.letsencrypt.org/acme/acct/...",
  "ca": "https://acme-v02.api.letsencrypt.org/directory"
}
```

### Get CNAME Setup Instructions

```http
POST /acme/setup
Content-Type: application/json

{
  "domain": "example.com"
}
```

```json
{
  "domain": "example.com",
  "cname": {
    "name": "_acme-challenge.example.com",
    "type": "CNAME",
    "value": "example.com.auth.example.com"
  },
  "instructions": "Create this CNAME record once..."
}
```

### Issue Certificate

```http
POST /acme/issue
Content-Type: application/json

{
  "domain": "example.com"
}
```

```json
{
  "domain": "example.com",
  "status": "valid",
  "certificate": "...",
  "private_key": "...",
  "issuer_certificate": "...",
  "cert_url": "..."
}
```

### Get Account Info

```http
GET /acme/account
```

```json
{
  "email": "user@example.com",
  "uri": "https://acme-v02.api.letsencrypt.org/acme/acct/...",
  "ca": "https://acme-v02.api.letsencrypt.org/directory"
}
```

## Quick Start

1. **Configure environment**:
   ```bash
   export ACME_ACCOUNT_EMAIL="your@email.com"
   export ACME_DNS_ZONE="auth.example.com"
   export ACME_PUBLIC_IP="203.0.113.10"
   ```

2. **Delegate DNS**: In your domain's DNS, add:
   ```
   _acme-challenge.yourdomain.com.  CNAME  yourdomain.com.auth.example.com.
   ```

3. **Run the service**:
   ```bash
   go run .
   ```

4. **Obtain certificate**:
   ```bash
   # Register account
   curl -X POST http://localhost:8080/acme/register

   # Get CNAME setup info
   curl -X POST http://localhost:8080/acme/setup \
     -d '{"domain": "yourdomain.com"}'

   # Issue certificate (after CNAME is configured)
   curl -X POST http://localhost:8080/acme/issue \
     -d '{"domain": "yourdomain.com"}'
   ```

## Project Structure

```
.
├── main.go                    # Entry point, HTTP server setup
├── acme/
│   ├── acme.go               # Core ACME logic (client, verifier, DNS provider)
│   ├── dnsserver.go          # Authoritative DNS server + TXT store
│   └── protocol.go           # HTTP handlers and route setup
├── go.mod
└── go.sum
```

## Key Components

### TXTStore (`acme/dnsserver.go`)

Thread-safe in-memory store mapping FQDNs to TXT values. Populated by `AutoDNSProvider` during certificate issuance.

### DNSServer (`acme/dnsserver.go`)

Minimal authoritative DNS server handling:
- **TXT**: Returns ACME challenge values for delegated names
- **SOA**: Authority records for the delegation zone
- **NS**: Nameserver records at zone apex
- **A**: Public IP for NS/zone apex resolution

### AutoDNSProvider (`acme/acme.go`)

Implements lego's DNS challenge provider interface:
- `Present()`: Writes challenge TXT value to the authoritative server
- `CleanUp()`: Removes the TXT value after validation
- `Timeout()`: 3-minute propagation window, 5-second polling

### Verifier (`acme/acme.go`)

Orchestrates the ACME flow:
- Account registration with the CA
- DNS-01 challenge solving via `AutoDNSProvider`
- Certificate issuance and retrieval

## Dependencies

- [gin](https://github.com/gin-gonic/gin) - HTTP framework
- [lego](https://github.com/go-acme/lego) - ACME client library
- [miekg/dns](https://github.com/miekg/dns) - DNS library

## License

See LICENSE file for details.
