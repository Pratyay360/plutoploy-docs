# DNS Manager

Containerized ACME (Let's Encrypt) domain verification and certificate issuance service, driven by a backend over HTTP.

## Documentation

- [Overview & Quick Start](docs/README.md)
- [Architecture](docs/architecture.md) — components, data flow, persistence model
- [API Reference](docs/api.md) — endpoints, request/response schemas, status codes
- [Configuration](docs/configuration.md) — environment variables and provider credentials
- [Challenge Types](docs/challenges.md) — DNS-01 vs HTTP-01 vs TLS-ALPN-01
- [Deployment](docs/deployment.md) — Docker, Compose, Kubernetes, volumes, ports
- [Backend Integration](docs/integration.md) — calling the service from your backend
- [Operations & Troubleshooting](docs/operations.md) — rate limits, renewals, debugging
