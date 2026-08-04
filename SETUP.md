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

All other service ports (Kubo API `5001`, auth-service `9000`, sead-core
`30080`, pin-replicator `32001`) are bound to localhost / the Docker network
and should **not** be exposed publicly. For bilateral replication, restrict
inbound `4001` to your partner nodes' IPs.

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
    location = /auth {
        internal;
        proxy_pass http://127.0.0.1:9000/auth/verify$is_args$args;
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

---

## 2. Kubo IPFS

### Install Kubo

```bash
wget https://dist.ipfs.tech/kubo/v0.42.0/kubo_v0.42.0_linux-amd64.tar.gz
tar -xvzf kubo_v0.42.0_linux-amd64.tar.gz
sudo mv kubo/ipfs /usr/local/bin
ipfs --version
ipfs init --profile server
cp ~/.ipfs/config ~/.ipfs/config.bak
```

### Configure

```bash
ipfs config Addresses.API /ip4/127.0.0.1/tcp/5001
ipfs config Addresses.Gateway ""
ipfs config Datastore.StorageMax "200GB"
ipfs config Datastore.GCPeriod "1h"
ipfs config --json Swarm.Transports.Network '{"Relay": false}'
ipfs config --json Swarm.RelayClient '{"Enabled": false}'
ipfs config --json Swarm.ConnMgr '{"LowWater": 100, "HighWater": 200}'
ipfs config --json Provide.DHT '{"Interval": "12h"}'
```

### Dedicated data partition (optional)

If you have a dedicated data partition (e.g. `/mnt/data`):

```bash
sudo mkdir -p /mnt/data/ipfs
sudo useradd --system --home /mnt/data/ipfs --no-create-home --shell /usr/sbin/nologin ipfs
sudo chown ipfs:ipfs /mnt/data/ipfs
sudo chmod 700 /mnt/data/ipfs
sudo rsync -a ~/.ipfs/ /mnt/data/ipfs/
sudo chown -R ipfs:ipfs /mnt/data/ipfs
```

Test the daemon:

```bash
ipfs daemon --enable-gc
```

If it starts cleanly, remove the old `~/.ipfs/` directory.

### systemd service

Create `/etc/default/ipfs`:

```
IPFS_PATH=/mnt/data/ipfs
```

```bash
sudo chown root:root /etc/default/ipfs && sudo chmod 644 /etc/default/ipfs
```

Create a systemd service file at `/etc/systemd/system/ipfs.service`:

```ini
[Unit]
Description=IPFS daemon
After=network.target

[Service]
Type=simple
User=ipfs
EnvironmentFile=/etc/default/ipfs
ExecStart=/usr/local/bin/ipfs daemon --enable-gc
Restart=on-failure
RestartSec=10
PrivateTmp=yes
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ipfs.service
```

### Verify

```bash
sudo systemctl status ipfs.service
sudo journalctl -u ipfs -f
sudo -u ipfs env IPFS_PATH=/mnt/data/ipfs /usr/local/bin/ipfs repo stat
sudo -u ipfs env IPFS_PATH=/mnt/data/ipfs /usr/local/bin/ipfs id
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
curl http://localhost:30080/health
curl http://localhost:9000/health
```

> **Note:** The images are published as public packages on ghcr.io.
> If your Docker environment returns `denied` on anonymous pulls, log in
> with a GitHub PAT (needs only `read:packages` scope):
>
> ```bash
> echo "$GITHUB_PAT" | docker login ghcr.io -u "$GITHUB_USERNAME" --password-stdin
> ```

### Register an organization

The auth-service needs to know your org's public key to verify tokens.
Generate the genesis envelope on a **secure machine** (not the IPFS node),
then POST it to the local sead-core.

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
curl -X POST http://localhost:30080/events \
  -H "Content-Type: application/json" \
  -d "{\"envelope_hex\": \"$(cat envelope.hex)\"}"
```

#### Verify

```bash
curl http://localhost:30080/orgs/<org_id_hex>
# Expected: {"status":"active","org_pk_hex":"<pk>"}
```

> **Tip:** Register multiple orgs by generating one envelope per org
> (each with its own keypair) and POSTing each one.

---

## 5. Next steps

- **Generate tokens** — see the [Token generation](README.md#token-generation) section in the main README
- **Set up bilateral replication** — see the [Bilateral Pin Replication](README.md#bilateral-pin-replication) section
- **Monitor the node** — use `journalctl -u ipfs -f` and `docker compose logs -f`