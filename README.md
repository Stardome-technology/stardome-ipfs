# Stardome IPFS Node

IPFS node deployment with SEAD-based authentication for pin operations.

## Reference Implementation

The Stardome IPFS node deployed at `ipfs.stardome.cloud` serves as the
reference implementation. Third parties are free to build their IPFS nodes
however they see fit, but nodes that match the following characteristics
are guaranteed to be compatible with the SEAD auth stack and the Stardome
ecosystem.

### Key features of the reference node

| Feature | Detail |
|---|---|
| **Kubo version** | `v0.42.0` (Linux amd64) |
| **Init profile** | `server` (`ipfs init --profile server`) |
| **Service manager** | `systemd` with dedicated `ipfs` user, `PrivateTmp=yes`, `NoNewPrivileges=yes` |
| **Data directory** | Dedicated partition at `/mnt/data/ipfs` via `IPFS_PATH` |
| **Storage cap** | 200 GB (`Datastore.StorageMax`) |
| **GC interval** | 1 hour (`Datastore.GCPeriod`), enabled at daemon start (`--enable-gc`) |
| **Relay** | Disabled (`Swarm.Transports.Network.Relay: false`, `Swarm.RelayClient.Enabled: false`) |
| **Connection manager** | LowWater 100 / HighWater 200 |
| **DHT provide interval** | 12 hours (`Provide.DHT.Interval: "12h"`) |
| **API address** | `127.0.0.1:5001` (localhost only) |
| **Gateway** | Disabled (all HTTP served through Nginx reverse proxy) |
| **Rate limiting** | Per-org via Nginx `limit_req_zone` (10 req/s, burst 20) |
| **Auth layer** | SEAD auth stack — Nginx `auth_request` subrequest to auth-service |

These settings are not mandatory, but they represent a tested production
configuration. If you deviate significantly, pay extra attention to
security hardening, connection management, and garbage collection tuning.

See the **[setup guide](SETUP.md)** for step-by-step provisioning
instructions (Nginx, Kubo, Docker, auth stack deployment).

## Architecture

```
Internet ──TLS──> Nginx :443
                    │
                    ├── /auth ──proxy──> auth-service :9000/auth/verify
                    │                       (SEAD auth stack via Docker)
                    ├── /api/v0/(add|pin/add) ──proxy──> IPFS Kubo :5001
                    │                       (SEAD token auth via auth_request)
                    ├── /pins ──proxy──> pin-replicator :32001
                    │                       (bilateral replication)
                    └── everything else ──> 403
```

The authentication layer is provided by the **SEAD auth stack**
(sead-core + auth-service running as Docker containers).

## What gets pinned to IPFS

When an attestation flows through the SEAD pipeline, only the **raw
`stardome_attestation` CBOR bytes** are pinned to IPFS. The Merkle tree
and inclusion proof are **not** pinned — they are the integrator's
responsibility to store separately.

### Pin size breakdown

| Component | Size | Included in IPFS pin? |
|-----------|------|-----------------------|
| Module XMSSMT signature (OID 0x05) | ~18 KB | ✅ Inside attestation CBOR |
| Module XMSSMT public key | ~4 KB | ✅ Inside attestation CBOR |
| Merkle root (32 B) + payload hash (32 B) + metadata | ~100 B | ✅ Inside attestation CBOR |
| **Total attestation CBOR pinned** | **~22-25 KB** | ✅ **This is what IPFS stores** |
| Edge-service commit signature (OID 0x12) | ~2.8 KB | ❌ Goes to sead-core event store |
| Edge-service auth token signature (OID 0x12) | ~2.8 KB | ❌ Ephemeral auth header only |
| Merkle tree file | ~varies | ❌ Integrator's responsibility |
| Inclusion proof | ~varies | ❌ Integrator's responsibility |

### Pin format

The bytes pinned to IPFS are **not** wrapped in any container format.
They are the raw CBOR-encoded `stardome_attestation` as defined in
[sead_v1.1.2.cddl](https://github.com/Stardome-technology/stardome-cbor-schemes/blob/main/sead_v1.1.2.cddl):

```cddl
stardome_attestation = {
  ext_id:         bstr,    ; module identifier
  xmss_pk:        bstr,    ; module XMSSMT public key
  merkle_root:    bstr,    ; Merkle tree root (32 bytes)
  xmss_sig:       bstr,    ; module XMSSMT signature
  previous_att:   bstr,    ; previous attestation hash
  counter:        uint,    ; attestation sequence number
  schema_version: uint,    ; CDDL schema version
}
```

### What is NOT pinned and why

- **Edge-service commit signature** — posted to sead-core as a SEAD DAG
  event (`edge_commit`). The verifier resolves it from the event store,
  not from IPFS.
- **Edge-service auth token signature** — generated per-pin, used as an
  HTTP `Authorization` header, then discarded. It is ephemeral by design.
- **Merkle tree** — the tree is produced by `stardome-client` during
  attestation but is intentionally **not** sent to the edge-service or
  pinned. The integrator is responsible for storing the tree file if
  they need full Merkle proof verification. The verifier currently
  checks `merkle_root == commitment_root` without requiring the full tree.
- **Inclusion proof** — derived from the tree at verification time.
  Not pinned because the tree itself is not pinned.

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
curl http://localhost:30080/health
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

The compose file (`docker-compose.ipfs-auth.yml`) runs three services:
- **sead-core** — event store and org/edge key resolution
- **auth-service** — token verification endpoint for Nginx `auth_request`
- **pin-replicator** — bilateral pin replication between node operators

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

The `OrgGenesis` events must already be registered in
`sead-core` from your existing SEAD deployment. Verify they are present:

```bash
curl http://localhost:30080/orgs/<org_id_hex>
# Expected: {"status":"active","org_pk_hex":"<pk>"}
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
  --payload-file payload.file
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

---

## Bilateral Pin Replication

The **pin-replicator** service enables node operators to establish
bilateral replication agreements. Each operator pins content from their
own orgs; the replicator pushes CIDs to partner nodes under per-partner
storage quotas. No shared cluster secret, no cross-org token flow —
orgs remain fully isolated on their home node.

This is useful when two organizations want to ensure each other's
pinned artifacts are available even if one node goes offline.

### Architecture

```
Node A (Operator A)              Node B (Operator B)
┌─────────────┐                  ┌─────────────┐
│ Org X pins  │                  │ Org Y pins  │
│     ↓        │                  │     ↓        │
│ Nginx :443  │                  │ Nginx :443  │
│     ↓        │                  │     ↓        │
│ pin-replicator ────POST /pins───┤ pin-replicator│
│   :32001     │  (secret_AB)    │   :32001     │
│     ↓        │                  │     ↓        │
│ IPFS Kubo   │                  │ IPFS Kubo   │
│   :5001     │                  │   :5001     │
└─────────────┘                  └─────────────┘
```

### How it works

1. **Org pins content** to its home node via SEAD-authenticated API
   (`/api/v0/add` with Bearer SEAD token).
2. **pin-replicator polls** the local IPFS API (`/api/v0/pin/ls`) at
   a configurable interval (default: 30s).
3. **New CIDs are pushed** to all active partners via `POST /pins`
   (authenticated with a per-partner shared secret).
4. **Quota is enforced** — if a partner's storage quota is exceeded,
   the pin is rejected with HTTP 413.
5. **Periodic verification** (default: 5min) checks that CIDs are still
   pinned on both nodes and re-pins if missing (e.g., after GC).

> **Node-to-node block transfer.** When a partner receives `POST /pins`, it
> pins the CID locally — Kubo fetches the actual blocks from the sender via
> bitswap. This requires the two nodes to be directly reachable on the
> swarm port and statically peered. See
> [Establishing a partnership](#establishing-a-partnership).

### Deploying the pin-replicator

The pin-replicator is included in `docker-compose.ipfs-auth.yml` and
starts automatically alongside sead-core and auth-service:

```bash
docker compose -f docker-compose.ipfs-auth.yml up -d

# Verify it is healthy
curl http://localhost:32001/health
# Expected: {"status":"ok","service":"pin-replicator","partners":0,...}
```

The replicator connects to the local IPFS Kubo instance via
`127.0.0.1:5001`. In `docker-compose.ipfs-auth.yml` it runs with
`network_mode: host`, so it shares the host's network namespace and reaches
the host Kubo API on its loopback address directly.

#### Replicator ↔ IPFS contract (deliberate Kubo coupling)

The replicator talks to the local node over **Kubo RPC** (`/api/v0/*`),
which in practice is the de-facto universal IPFS HTTP wire API. The calls
are a small, stable set:

| Call | Purpose |
|------|---------|
| `POST /api/v0/pin/ls?type=recursive` | list recursive pins |
| `POST /api/v0/pin/add?arg=<cid>` | re-pin for GC resilience |
| `POST /api/v0/pin/rm?arg=<cid>` | unpin |
| `POST /api/v0/dag/stat?arg=<cid>` | CID size for quota |

> **POST-only:** Kubo's HTTP API rejects `GET /api/v0/...` with `405`
> (`Allow: OPTIONS, POST`) since Kubo v0.5.0 as browser CSRF protection.
> All four calls use `POST`, and each passes its argument as a query
> parameter (`?arg=...`) — Kubo's RPC API does **not** read arguments from
> a JSON body.

This coupling is **chosen, not accidental**: the reference nodes are Kubo,
and the RPC surface is tiny and stable. The operator-facing flexibility
("host your node however you like") concerns *how* the node is hosted, not
the replicator's wire protocol.

If a future requirement hardens against non-Kubo nodes, the unwind path is
to swap this seam for the implementation-agnostic
[IPFS Pinning Service API (IPIP-0373)](https://ipfs.github.io/specs/pinning/):
add a pinning-service adapter and point the replicator at it. The four
calls above are centralized in `tools/pin-replicator/main.go`, so the swap
is contained to that file.

> **Kubo RPC is POST-only.** The Kubo HTTP API rejects `GET /api/v0/...` with
> `405 Method Not Allowed` (`Allow: OPTIONS, POST`) since Kubo v0.5.0, as a
> browser CSRF protection. The replicator therefore calls Kubo with `POST`.

### Establishing a partnership

To set up replication between Node A and Node B, each node must register
the other as a partner using the **same shared secret**.

> **Prerequisite — the two nodes must be able to exchange blocks.**
> The replicator only pushes the *CID* over HTTPS (`POST /pins`); the
> receiver fetches the actual blocks from the sender via **bitswap** on
> the libp2p swarm. That requires:
>
> 1. **Swarm port reachable between the two nodes** (default TCP/4001, plus
>    UDP/4001 for QUIC if enabled). Open it in the cloud provider firewall
>    *and* any host firewall, restricted to the partner's IP:
>    ```bash
>    # On each node, allow the partner's public IP on the swarm port
>    sudo ufw allow from <partner_public_ip> to any port 4001 proto tcp
>    sudo ufw allow from <partner_public_ip> to any port 4001 proto udp
>    ```
>    Verify with `nc -vz -w 5 <partner_host> 4001` — must report `succeeded`
>    quickly.
>
> 2. **Static peering between the two nodes.** The replicator does not
>    establish libp2p connections — it only orchestrates pins. With the
>    reference config (`Provide.DHT.Interval: 12h`), a newly pinned CID has
>    no fresh provider record, and the nodes won't naturally dial each other
>    without shared bootstraps or mDNS. Add each node to the other's
>    `Peering.Peers` so they maintain a permanent direct connection (bitswap
>    then always has the sender reachable):
>
>    ```bash
>    # Get the other node's peer ID (on each node):
>    ipfs id
>
>    # On Node A — peer with Node B:
>    ipfs config --json Peering.Peers '[
>      {"ID": "<node_b_peer_id>", "Addrs": ["/ip4/<node_b_public_ip>/tcp/4001"]}
>    ]'
>
>    # On Node B — peer with Node A:
>    ipfs config --json Peering.Peers '[
>      {"ID": "<node_a_peer_id>", "Addrs": ["/ip4/<node_a_public_ip>/tcp/4001"]}
>    ]'
>
>    # Restart the daemon on both nodes, then confirm each sees the other:
>    ipfs swarm peers | grep <partner_peer_id>
>    ```
>
>    > `Peering.Peers` is additive — nothing in the reference config needs
>    > to be removed. It complements `Provide.DHT` (which keeps content
>    > discoverable on the public DHT) and does not use relays, so it's
>    > compatible with `Swarm.RelayClient.Enabled: false`. Kubo exempts
>    > peered nodes from connection-manager pruning.

Generate a strong random secret:

```bash
openssl rand -hex 32
```

#### On Node A — register Node B as partner

```bash
curl -X POST http://localhost:32001/partners \
  -H "Content-Type: application/json" \
  -d '{
    "partner_id": "node-b",
    "api_url": "https://node-b.example.com/pins",
    "shared_secret": "<random_secret_AB>",
    "quota": "50GB"
  }'
# Expected: 201 Created
```

#### On Node B — register Node A as partner

```bash
curl -X POST http://localhost:32001/partners \
  -H "Content-Type: application/json" \
  -d '{
    "partner_id": "node-a",
    "api_url": "https://node-a.example.com/pins",
    "shared_secret": "<random_secret_AB>",
    "quota": "50GB"
  }'
```

> **Important:** The `shared_secret` must be identical on both nodes
> for the bilateral pair. Each partnership uses its own unique secret.

### Partner API reference

All partner API endpoints are served on port `32001` (localhost only).
In production, the Nginx reverse proxy exposes them at `/pins`.

#### Add partner

```bash
POST /partners
Content-Type: application/json

{
  "partner_id": "node-b",
  "api_url": "https://node-b.example.com/pins",
  "shared_secret": "secret",
  "quota": "50GB"
}

# Response: 201 Created
{
  "id": "node-b",
  "api_url": "https://node-b.example.com/pins",
  "quota_bytes": 53687091200,
  "quota_used": 0,
  "active": true
}
```

#### List partners

```bash
GET /partners

# Response: 200 OK
[
  {
    "id": "node-b",
    "api_url": "https://node-b.example.com/pins",
    "quota_bytes": 53687091200,
    "quota_used": 1073741824,
    "active": true,
    "created_at": "2025-01-15T10:30:00Z"
  }
]
```

#### Remove partner

```bash
DELETE /partners/node-b

# Response: 200 OK
{"message": "partner removed"}
```

#### Receive pin from partner

```bash
POST /pins
X-Partner-Token: <shared_secret>
Content-Type: application/json

{"cid": "QmXoAprrLZCZ3m6kEFrP3j2MkkFFPz8PmJtUn1eKWBnq"}

# Response: 200 OK
{
  "cid": "QmXoAprrLZCZ3m6kEFrP3j2MkkFFPz8PmJtUn1eKWBnq",
  "size_bytes": 1024,
  "quota_used": 1074766848,
  "quota_limit": 53687091200
}

# Response: 413 Request Entity Too Large (quota exceeded)
{
  "error": "quota exceeded",
  "quota_used": 53687091200,
  "quota_limit": 53687091200,
  "size": 1024
}

# Response: 403 Forbidden (invalid token)
{"error": "invalid partner token"}
```

#### Check if a CID is pinned locally

```bash
GET /pins/<cid>
X-Partner-Token: <shared_secret>

# Response: 200 OK
{"cid": "Qm...", "pinned": true}

# Response: 404 Not Found
{"error": "CID not found locally: Qm..."}
```

#### List CIDs pinned by a partner

```bash
GET /pins
X-Partner-Token: <shared_secret>

# Response: 200 OK
{
  "partner_id": "node-b",
  "pins": [
    {"cid": "Qm...", "size_bytes": 1024, "pinned_at": "2025-01-15T10:30:00Z"}
  ],
  "total_pins": 1,
  "quota_used": 1024,
  "quota_limit": 53687091200
}
```

#### Unpin a CID (partner revocation)

```bash
DELETE /pins/<cid>
X-Partner-Token: <shared_secret>

# Response: 200 OK
{"message": "unpinned", "cid": "Qm..."}
```

### Nginx config for the `/pins` endpoint

If you use Nginx as a reverse proxy (recommended), add this location
block to expose the partner API:

```nginx
location /pins {
    proxy_pass http://127.0.0.1:32001;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;
    proxy_request_buffering off;
    proxy_read_timeout 600s;
}

# Health endpoint (no auth needed)
location /pins/health {
    proxy_pass http://127.0.0.1:32001/health;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
}
```

### Configuration reference

| Flag | Default | Description |
|------|---------|-------------|
| `--port` | `32001` | HTTP listen port |
| `--db` | `/data/pin-replicator.db` | SQLite database path |
| `--ipfs-api-url` | `http://127.0.0.1:5001` | IPFS Kubo API URL |
| `--poll-interval` | `30s` | Interval to poll local IPFS for new pins |
| `--verify-interval` | `5m` | Interval to verify partner replication |

| Environment variable | Default | Description |
|----------------------|---------|-------------|
| `LOG_LEVEL` | `info` | Log level (debug, info, warn, error) |

### Testing replication end-to-end

```bash
# 1. Pin a file on Node A (via SEAD token)
curl -X POST https://node-a.example.com/api/v0/add \
  -H "Authorization: Bearer $(cat token_org_a.b64)" \
  -F "file=@payload.file"
# Expected: 200 + CID

# 2. Wait for poll interval (default 30s), then verify on Node B:
curl http://localhost:32001/pins \
  -H "X-Partner-Token: <secret_AB>"
# Expected: CID appears in pins list

# 3. Test quota enforcement — fill quota on Node B, then pin on Node A
# Node B should return 413 and Node A should log quota warning
```