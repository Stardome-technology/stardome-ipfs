# Stardome IPFS Node

IPFS node deployment with SEAD-based authentication for pin operations.

## Architecture

```
Internet ──TLS──> Nginx :443
                    │
                    ├── /auth ──proxy──> auth-service :9000/auth/verify
                    │                       (SEAD auth stack via Docker)
                    ├── /api/v0/(add|pin/add) ──proxy──> IPFS Kubo :5001
                    └── everything else ──> 403
```

The authentication layer is provided by the **SEAD auth stack**
(sead-core + auth-service running as Docker containers).

### Token format

See [sead_auth_token_v1.0.0.cddl](https://github.com/Stardome-technology/stardome-cbor-schemes/blob/main/sead_auth_token_v1.0.0.cddl)

CBOR map with:
- `org_id` — organization identifier
- `scope` — `"ipfs_pin"`
- `expiry` — UNIX timestamp
- `nonce` — random bytes
- `signature` — XMSS signature over canonical CBOR of fields 1-4
- `payload_hash` (optional) — binds token to a specific artifact

Token is **base64url-encoded** CBOR (`RFC 4648 §5`, no padding) and passed
as `Authorization: Bearer <token>`.

## SEAD Auth Stack

Deploy the minimal auth stack alongside your IPFS node to validate
tokens against org public keys registered via SEAD DAG events.

```bash
# Pull and start (no auth needed — images are public)
docker compose -f docker-compose.ipfs-auth.yml pull
docker compose -f docker-compose.ipfs-auth.yml up -d

# Health check
curl http://localhost:9000/health
curl http://localhost:8080/health
```

The compose file (`docker-compose.ipfs-auth.yml`) runs two services:
- **sead-core** — event store and org/edge key resolution
- **auth-service** — token verification endpoint for Nginx `auth_request`

### Nginx config

Add this location block to your IPFS site config:

```
location = /auth {
    internal;
    proxy_pass http://127.0.0.1:9000/auth/verify$is_args$args;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header Authorization $http_authorization;
    proxy_set_header X-Original-URI $request_uri;
}
```

### Bootstrap genesis events

Before auth works, register `OrgGenesis` and `EdgeAuthorization` events:

```bash
curl -X POST http://localhost:8080/events \
  -H "Content-Type: application/json" \
  -d '{"envelope_hex": "<org_genesis_cbor_hex>"}'

curl -X POST http://localhost:8080/events \
  -H "Content-Type: application/json" \
  -d '{"envelope_hex": "<edge_auth_cbor_hex>"}'
```

### Usage example

```bash
curl -X POST \
  -H "Authorization: Bearer <BASE64URL_CBOR_TOKEN>" \
  -F file=@signature.cbor \
  "https://ipfs.yourdomain.com/api/v0/add"
```