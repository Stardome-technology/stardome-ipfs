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
# Pull and start
docker compose -f docker-compose.ipfs-auth.yml pull
docker compose -f docker-compose.ipfs-auth.yml up -d

# Health check
curl http://localhost:9000/health
curl http://localhost:8080/health
```

> **Note:** The images are published as public packages on ghcr.io.
> However, some Docker environments may still return `denied` on anonymous pulls.
> If you get an error, log in with a GitHub PAT first:
>
> ```bash
> echo "$GITHUB_PAT" | docker login ghcr.io -u "$GITHUB_USERNAME" --password-stdin
> ```
>
> The PAT needs only `read:packages` scope.

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

The `OrgGenesis` and `EdgeAuthorization` events must already be registered in
`sead-core` from your existing SEAD deployment. Verify they are present:

```bash
curl http://localhost:8080/orgs/<org_id_hex>
# Expected: {"status":"active","org_pk_hex":"<pk>"}

curl http://localhost:8080/edges/<org_id_hex>/<edge_id_hex>
# Expected: {"status":"authorized","edge_pk_hex":"<pk>"}
```

If not, follow the [sead-service bootstrap guide](https://github.com/Stardome-technology/sead-service/blob/main/docs/bootstrap-genesis.md) first.

## Token generation

The auth stack only verifies tokens — it never generates them. You need signed
tokens from an existing SEAD org. The org operator generates them on a secure
laptop using the `gen-token` tool (see [sead-service docs](https://github.com/Stardome-technology/sead-service/blob/main/docs/bootstrap-genesis.md)):

```bash
# On the org's secure laptop (not the IPFS node):
./build/tools/gen-token \
  --org-id <org_id_hex> \
  --org-signing-key <org_secret_key_hex> \
  --org-public-key <org_public_key_hex> \
  --payload-file artifact.cbor
```

The token is a single line of base64url-encoded CBOR. Transfer it to the
IPFS node (e.g., via SSH pipe or mounted secret) and use it in API calls.

### Prerequisites

Before deploying this auth stack, you must have:

1. **A running SEAD org** — sead-core with registered `OrgGenesis` and `EdgeAuthorization` events
2. **The org_id** — the organization identifier (hex) from keygen output
3. **A way to generate tokens** — the `gen-token` tool (see [sead-service](https://github.com/Stardome-technology/sead-service) docs) or an edge-service that holds the org signing key

All of these come from an existing SEAD deployment. If you don't have them
yet, follow the [sead-service bootstrap guide](https://github.com/Stardome-technology/sead-service/blob/main/docs/bootstrap-genesis.md) first.

### Usage example

```bash
curl -X POST \
  -H "Authorization: Bearer <BASE64URL_CBOR_TOKEN>" \
  -F file=@signature.cbor \
  "https://ipfs.yourdomain.com/api/v0/add"
```