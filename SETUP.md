# Stardome IPFS Node — Setup Guide

This guide walks through provisioning a production IPFS node with SEAD
authentication from scratch. It covers Nginx, Kubo IPFS, Docker, and the
SEAD auth stack.

> **Prerequisites:** A Linux server (tested on Ubuntu 26.04) with
> root access, a public IP, and a DNS A record pointing `ipfs.<yourdomain>`
> to that IP. Firewall must allow TCP/80 and TCP/443.

## Public ports to open

Before provisioning, open these ports on the host firewall (and any cloud
security group):

- **`80/tcp`** — HTTP (for Let's Encrypt / certbot HTTP-01 challenge)
- **`443/tcp`** — HTTPS (Nginx reverse proxy — all client API and pin traffic)
- **`4001/tcp`** — IPFS swarm (libp2p TCP — block exchange between nodes)
- **`4001/udp`** — IPFS swarm (QUIC, if enabled)

All other service ports (Kubo API `5001`, gateway `30080`, pin-replicator
`32001`) are bound to localhost / the Docker network and should **not** be
exposed publicly. For bilateral replication, restrict inbound `4001` to your
partner nodes' IPs.

---

## 1. Nginx reverse proxy

```bash
sudo apt update
sudo apt install nginx

sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d ipfs.<yourdomain>
```

### Nginx global config

Edit `/etc/nginx/nginx.conf` and add inside the `http` block:

```nginx
log_format main '$remote_addr - $org_id - $request';
access_log /var/log/nginx/access.log main;

map $org_id $org_id_safe {
    "" "anonymous";
    default $org_id;
}

limit_req_zone $org_id zone=org_limit:20m rate=10r/s;
```

### Nginx site config

Create `/etc/nginx/sites-available/ipfs.<yourdomain>`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name ipfs.<yourdomain>;

    ssl_certificate /etc/letsencrypt/live/ipfs.<yourdomain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ipfs.<yourdomain>/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    client_max_body_size 10M;

    # --- AUTH SUBREQUEST ---
    # The gateway (collapsed from auth-service) serves /auth/verify on
    # 127.0.0.1:30080. It resolves org keys from sead-core over gRPC.
    location = /auth {
        internal;
        proxy_pass http://127.0.0.1:30080/auth/verify$is_args$args;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header Authorization $http_authorization;
        proxy_set_header X-Original-URI $request_uri;
    }

    # --- IPFS API PROXY (SEAD-authenticated) ---
    location ~ ^/api/v0/(add|pin/add) {
        auth_request /auth;
        auth_request_set $org_id $upstream_http_x_org_id;
        proxy_set_header X-Org-ID $org_id;
        limit_req zone=org_limit burst=20;
        proxy_pass http://127.0.0.1:5001;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_read_timeout 600s;
    }

    location /api/v0/ {
        return 403;
    }

    # --- PARTNER PINNING API (operator-to-operator) ---
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

    location /pins/health {
        proxy_pass http://127.0.0.1:32001/health;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }

    location / {
        return 403;
    }
}
```

Enable the site:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/ipfs.<yourdomain> /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

> **The `location /api/v0/ { return 403; }` catch-all is intentional.**
> The node is a pinning backend, not a public content gateway: only
> `add` and `pin/add` are proxied. `cat`, `pin/ls`, `block/stat`, `id`
> and `version` are deliberately **not** exposed — see the
> [Retrieval model](README.md#retrieval-model-verification) in the README.

---

## 2. Kubo IPFS (official Docker image)

Run Kubo from the **official Docker image**
[`ipfs/kubo`](https://hub.docker.com/r/ipfs/kubo) — the supported
production deployment path. Do **not** install from tarballs or build
from source: the image is how integrators are expected to run the node,
it tracks upstream releases cleanly, and Docker restart policies replace
host-level service management.

> `ipfs/kubo:latest` always points at the latest stable release. To hold
> a specific version for reproducibility, use `ipfs/kubo:v0.43.1` (or
> any `vN.N.N` tag).
>
> **Prerequisite:** Docker Engine — see [section 3](#3-docker) — must be
> installed before starting the node.

### Prepare the data directory

The repo lives on the dedicated data partition and is mounted into the
container at `/data/ipfs` (the image's default repo path). The container's
`ipfs` user is **UID 1000**, and a bind mount enforces ownership by
numeric UID — so the host directory must be owned by numeric `1000`:

```bash
sudo mkdir -p /mnt/data/ipfs
sudo chown 1000:1000 /mnt/data/ipfs
sudo chmod 700 /mnt/data/ipfs
```

> **Do NOT create a host `ipfs` system user** for the Docker path.
> `useradd --system ipfs` allocates a UID from the 100–999 system range
> (e.g. 999), which the container cannot read. The image brings its own
> `ipfs` user (UID 1000); the host only needs the numeric ownership
> above. On a standard fresh Ubuntu server install (Docker
> `userns-remap` off, the default), this works as-is. If `userns-remap`
> is enabled, container UID 1000 maps to a subordinate host UID and the
> `chown` target must be that mapped range instead.

### Run the container

```bash
docker run -d \
  --name ipfs \
  --restart unless-stopped \
  --network host \
  -e IPFS_PROFILE=server \
  -v /mnt/data/ipfs:/data/ipfs \
  ipfs/kubo:latest \
  daemon --enable-gc
```

- **`--network host`** keeps the exact SEAD topology: the RPC API binds
  `127.0.0.1:5001` on the host (Nginx and the pin-replicator both
  target it there) and the swarm listens on `4001`. No published ports
  are needed.
- **`IPFS_PROFILE=server`** applies the `server` profile on first init
  (equivalent to `ipfs init --profile server`).
- **`daemon --enable-gc`** enables garbage collection on the configured
  `GCPeriod`.
- If you must run on a bridge network instead, publish the API on the
  host loopback only (`-p 127.0.0.1:5001:5001`) and set the
  in-container API address to `/ip4/0.0.0.0/tcp/5001` so the host can
  reach it. Never expose the RPC API on a public interface.

### Configure

Apply the reference configuration after the first start:

```bash
docker exec ipfs ipfs config Addresses.API /ip4/127.0.0.1/tcp/5001
docker exec ipfs ipfs config Addresses.Gateway ""
docker exec ipfs ipfs config Datastore.StorageMax "200GB"
docker exec ipfs ipfs config Datastore.GCPeriod "1h"
docker exec ipfs ipfs config --json Swarm.Transports.Network '{"Relay": false}'
docker exec ipfs ipfs config --json Swarm.RelayClient '{"Enabled": false}'
docker exec ipfs ipfs config --json Swarm.ConnMgr '{"LowWater": 100, "HighWater": 200}'
docker exec ipfs ipfs config --json Provide.DHT '{"Interval": "12h"}'
docker restart ipfs
```

### Verify

```bash
docker logs -f ipfs          # wait for "Daemon is ready"
docker exec ipfs ipfs repo stat
docker exec ipfs ipfs id
curl -sX POST http://127.0.0.1:5001/api/v0/version   # Kubo RPC is POST-only
```

### Upgrades

```bash
docker pull ipfs/kubo:latest
docker rm -f ipfs
# re-run the `docker run` command above — the repo persists in the volume
docker logs -f ipfs
```

---

## 3. Docker

Install Docker Engine:

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
```

Log out and back in for the group change to take effect.

---

## 4. SEAD Auth Stack

### Deploy

```bash
wget -O docker-compose.ipfs-auth.yml \
  https://raw.githubusercontent.com/Stardome-technology/stardome-ipfs/main/docker-compose.ipfs-auth.yml

docker compose -f docker-compose.ipfs-auth.yml pull
docker compose -f docker-compose.ipfs-auth.yml up -d

# Verify
curl http://localhost:30080/health    # gateway (auth/verify + proxy)
curl http://localhost:32001/health    # pin-replicator
```

> **Note:** The images are published as public packages on ghcr.io.
> If your Docker environment returns `denied` on anonymous pulls, log in
> with a GitHub PAT (needs only `read:packages` scope):
>
> ```bash
> echo "$GITHUB_PAT" | docker login ghcr.io -u "$GITHUB_USERNAME" --password-stdin
> ```

### Register an organization

The gateway needs to know your org's public key to verify tokens.
Generate the genesis envelope on a **secure machine** (not the IPFS node),
then POST it to the local sead-core via the gateway.

#### Generate the envelope (on a secure machine)

```bash
# Pull the gen-bootstrap tool
docker pull ghcr.io/stardome-technology/stardome-sead/gen-bootstrap:latest

# Generate the envelope
docker run --rm -v "$(pwd):/data" \
  ghcr.io/stardome-technology/stardome-sead/gen-bootstrap org-genesis \
  --org-id <org_id_hex> \
  --org-signing-key <org_secret_key_hex> \
  --org-public-key <org_public_key_hex> \
  --attestation-file /data/endorse_att.bin \
  --not-before <unix_epoch_sec> \
  --not-after <unix_epoch_sec> \
  --out-file /data/envelope.hex
```

> **Security:** The org signing key is the root of trust. Never copy it
> to the IPFS node. Generate the envelope offline and transfer only the
> resulting `envelope.hex` file.

#### POST the envelope to sead-core

Copy `envelope.hex` to the IPFS node, then:

```bash
# This IPFS auth stack does NOT set SEAD_AUTH_SECRET (gateway is localhost-only;
# Nginx is the public auth enforcement point), so no Authorization header is needed.
curl -X POST http://localhost:30080/events \
  -H "Content-Type: application/json" \
  -d "{\"envelope_hex\": \"$(cat envelope.hex)\"}"
```

#### Verify

```bash
# No Authorization header needed here (see note above).
curl http://localhost:30080/orgs/<org_id_hex>
# Expected: {"status":"active","org_pk_hex":"<pk>"}
```

> **Tip:** Register multiple orgs by generating one envelope per org
> (each with its own keypair) and POSTing each one.

---

## 4.1 Clients pinning to this node (pin-service TLS)

A SEAD **pin-service** (Go, running on an organization infrastructure) pins artifacts to this
IPFS node by calling `POST https://ipfs.<yourdomain>/api/v0/add` with a Bearer
auth token. Because this endpoint is served by Nginx with a **Let's Encrypt**
certificate (or any other public CA one), the pin-service's HTTP client must trust the **public CA
bundle** — it does **not** need your private CA.

- The pin-service uses Go's `net/http` client with a 60-second timeout. It sends
  the auth token in the `Authorization: Bearer <token>` header. Go's default CA
  bundle is used automatically — no explicit CA path configuration is needed.
- If you deploy this IPFS node behind a **private CA** instead of public one,
  configure the pin-service to trust that CA bundle (e.g. via `SSL_CERT_FILE`
  env var or system CA installation).
- The POC's local CA is **not** needed here — this node presents a public cert.

---

## 5. Next steps

- **Generate tokens** — see the [Token generation](README.md#token-generation) section in the main README
- **Set up bilateral replication** — see the [Bilateral Pin Replication](README.md#bilateral-pin-replication) section
- **Monitor the node** — use `journalctl -u ipfs -f` and `docker compose logs -f`