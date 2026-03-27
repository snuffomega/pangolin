# Pangolin w. Traefik + Cloudflare Tunnel

[![Pangolin](https://img.shields.io/badge/Pangolin-1.16.2-blue?logo=docker&logoColor=white)](https://hub.docker.com/r/fosrl/pangolin)
[![Traefik](https://img.shields.io/badge/Traefik-v3.6.10-blue?logo=traefik&logoColor=white)](https://hub.docker.com/_/traefik)
[![Cloudflared](https://img.shields.io/badge/Cloudflared-2026.3.0-orange?logo=cloudflare&logoColor=white)](https://hub.docker.com/r/cloudflare/cloudflared)
[![CF-Bridge](https://img.shields.io/badge/CF--Bridge-v2.0.0-lightgrey?logo=github&logoColor=white)](https://github.com/hhftechnology/pangolin-cloudflare-tunnel)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-Support-FF5E5B?logo=kofi&logoColor=white)](https://ko-fi.com/snuffomega)


This Pangolin stack is focused on utlizing Pangolins central management hub for secure local implementations, utilizing an automated metadata bridge between Traefik’s reverse proxy and Cloudflare’s Tunneling protocol. By monitoring local container labels and dynamically pushing updates to the Cloudflare dashboard, it eliminates the need for manual DNS records and open firewall ports, ensuring your self-hosted services remain both accessible and secure.

- **Reverse proxy:** Traefik (internal only, no published ports)
- **External access:** Cloudflare Tunnel
- **TLS:** Let's Encrypt via Cloudflare DNS challenge
- **Auth gateway:** Pangolin (manages tunnel routes + user sessions)

## Services

```
docker-compose.yml
         |───────  core containers  ─────────────────────────
         ├─ pangolin → fosrl/pangolin:1.16.2
         |   └─ Tunnel manager / dashboard UI
         ├─ pangolin-traefik → traefik:v3.6.10
         |   └─ Reverse proxy + TLS termination
         ├─ pangolin-cloudflared → cloudflare/cloudflared:2026.3.0
         |   └─ Cloudflare tunnel connector
         └─ pangolin-cf-bridge → hhftechnology/pangolin-cloudflare-tunnel:v2.0.0 
             └─  Syncs Traefik routes → Cloudflare
```

## Prerequisites

- **Docker Compose** Docker Compose v2+
- **Cloudflare account** with a domain managed in Cloudflare DNS
- **Cloudflare Zero Trust tunnel** created at Zero Trust → Networks → Tunnels
  - Note the **Tunnel Token** and **Tunnel ID** (UUID)
- **Cloudflare API token** with permissions:
  - `Zone → DNS → Edit` (Let's Encrypt DNS challenge + bridge DNS records)
  - `Account → Cloudflare Tunnel → Edit` (bridge updates tunnel ingress)
- A **free LAN IP** reserved for Traefik (e.g. `192.168.1.X`)
  - No router port forwards needed

---

## Quick Start

### 1. Create directory structure

```bash

mkdir -p /mnt/user/appdata/pangolin
cd /mnt/user/appdata/pangolin
git clone https://github.com/<your-org>/pangolin_cloudflared.git.
mkdir -p config/traefik config/letsencrypt config/logs
```

### 2. Verify repo files

`git clone` from step 1 automatically copies these contents, confirm for successful installation: `/mnt/user/appdata/pangolin/` — and the `config/traefik/` configs.

### 3. Edit Pangolin `config.yml` with your custom domain

```bash
cp config/example.config.yml config/config.yml
nano config/config.yml
```

Update the three domain fields to match your domain, and replace `server.secret` with a random string:

```bash
openssl rand -hex 32   # generates a suitable secret
```

### 4. Configure environment variables

```bash
cp example.env .env
nano .env
```

Update `.env`:

```env
#### Domains Must match those from step #3 ####
PUBLIC_URL=https://pangolin.yourdomain.com
PANGOLIN_DOMAIN=pangolin.yourdomain.com
BASE_DOMAIN=yourdomain.com
LETSENCRYPT_EMAIL=your-email@example.com

#### Cloudflare account and tokens ####
CF_TUNNEL_TOKEN=<from Cloudflare Zero Trust → Tunnels>
CF_API_TOKEN=<API token with DNS+Tunnel perms>
CF_ACCOUNT_ID=<Cloudflare account ID>
CF_TUNNEL_ID=<UUID of your tunnel>
CF_ZONE_ID=<Zone ID for your domain>
```

### 5. Set Traefik LAN IP (recommended)

In `docker-compose.yml`, update to create Traefik static IP to an open ip on your LAN `192.168.1.X`:

```yaml
lan:
  ipv4_address: 192.168.1.X   # pick a free IP from your LAN/Router
```

### 6. Start the stack

```bash
docker compose up -d
docker compose ps
```

---

## First Run — Initial Setup

Once containers are running, get the one-time setup token from the logs:

```bash
docker compose logs pangolin | grep -i token
```

If you miss it, restart to regenerate:

```bash
docker compose restart pangolin && docker compose logs pangolin | grep -i token
```

Open the Pangolin setup wizard and enter the token:

```
http://<unraid-ip>:3002
```

1. Enter the setup token
2. Create your first admin user
3. Confirm your organisation and base domain match what you set in `config/config.yml`
4. After setup, the dashboard will be available at your `PUBLIC_URL` through the Cloudflare tunnel

---

## Verification

### Check local Pangolin health

```bash
curl -I http://127.0.0.1:3002
curl -I http://127.0.0.1:3000/api/v1
```

### Check Traefik routing internally

```bash
docker exec -it pangolin-traefik sh -lc \
  'wget -S -O- --header="Host: pangolin.yourdomain.com" https://localhost/ --no-check-certificate'
```

Expected: HTTP 200 + Pangolin HTML

### Check Cloudflare tunnel is live

```bash
dig +short pangolin.yourdomain.com
curl -I https://pangolin.yourdomain.com
```

Expected: Cloudflare IPs in dig output, `server: cloudflare` header, HTTP 200

---

## Validation Checklist

- [ ] `https://pangolin.yourdomain.com` loads
- [ ] Login works and JWT session is issued
- [ ] Protected app redirects to Pangolin login correctly
- [ ] No router ports forwarded (confirm in router)
- [ ] Survives `docker compose down && docker compose up -d`

---

## Adding a Protected Application

In the Pangolin dashboard, create a new site/resource pointed at an internal service. The CF bridge automatically syncs the new Traefik route to a Cloudflare DNS record + tunnel ingress rule.

Example: `https://pang-navidrome.yourdomain.com` → internal Navidrome container

---

## Common Commands

```bash
# Start
docker compose up -d

# View logs
docker compose logs -f

# Stop
docker compose down

# Update images and restart
docker compose pull && docker compose up -d
```

---

## Backup Scope

Back up these paths from `/mnt/user/appdata/pangolin/`:

| Path | Include |
|---|---|
| `docker-compose.yml` | Yes |
| `.env` | Yes (store securely) |
| `config/config.yml` | Yes — Pangolin app config (contains secret key) |
| `config/traefik/` | Yes — Traefik static + dynamic configs |
| `config/db/` | Yes — SQLite database |
| `config/letsencrypt/acme.json` | Optional (certs regenerate automatically) |
| `config/logs/` | No |

---

## Updating Image Versions

Image versions are pinned in `docker-compose.yml`. Check releases and update tags, then repull:

- Pangolin: https://github.com/fosrl/pangolin/releases
- Traefik: https://github.com/traefik/traefik/releases
- Cloudflared: https://github.com/cloudflare/cloudflared/releases
- CF Bridge: https://github.com/hhftechnology/pangolin-cloudflare-tunnel/releases

---

## Troubleshooting

**Traefik dashboard** — uncomment `ports: ["8081:8080"]` in `docker-compose.yml` under the `traefik` service, then visit `http://<traefik-lan-ip>:8080/dashboard/`

**Healthcheck failing** — check `docker compose logs pangolin`. Pangolin must be able to read `config/config.yml` to pass the healthcheck.

**Let's Encrypt issues** — certs persist in `config/letsencrypt/acme.json` across restarts. If you hit rate limits, existing certs will continue to be used until expiry.

**CF Bridge not syncing routes** — check `docker compose logs pangolin-cf-bridge`. It polls `http://pangolin-traefik:8080` — requires `api.insecure: true` in `traefik_config.yml`.

**Tunnel not connecting** — verify the tunnel shows `HEALTHY` in the Cloudflare Zero Trust dashboard. The `cloudflared` container must be running before traffic can route.

---

If this helped you out, consider [buying me a coffee](https://ko-fi.com/snuffomega).

[![Support on Ko-fi](https://img.shields.io/badge/Ko--fi-F16061?style=flat-square&logo=ko-fi&logoColor=white)](https://ko-fi.com/snuffomega)