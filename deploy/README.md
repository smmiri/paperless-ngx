# Deploy Paperless at paperless.smmiri.com

This directory is the compile-and-host path for a Docker server. Compose
builds the image from this repository (frontend and backend) and publishes
the app only through a Cloudflare Tunnel.

Public URL: `https://paperless.smmiri.com`

Paperless stores documents in clear text. Treat the host, volumes, and
tunnel token as sensitive.

## What the stack compiles and runs

| Service | Role |
| --- | --- |
| `webserver` | Built from the repo `Dockerfile` (Angular UI + Python backend) |
| `db` | PostgreSQL 18 |
| `broker` | Valkey (Redis-compatible) |
| `gotenberg` + `tika` | Office / email conversion |
| `cloudflared` | Outbound tunnel to `paperless.smmiri.com` |

Port 8000 is not published on the host. The tunnel is the only ingress.

## One-time Cloudflare setup

Do this in the Cloudflare Zero Trust dashboard for the `smmiri.com` zone.

1. Open **Zero Trust > Networks > Tunnels > Create**.
2. Name the tunnel `paperless` (or reuse an existing host tunnel).
3. Copy the connector token into `CLOUDFLARE_TUNNEL_TOKEN` in `.env`.
4. Add a public hostname:
   - **Hostname:** `paperless.smmiri.com`
   - **Service:** `http://webserver:8000`
   - **Type:** HTTP
5. Confirm DNS for `paperless.smmiri.com` is a proxied CNAME to the
   tunnel (`<tunnel-id>.cfargotunnel.com`). `cloudflared tunnel route dns`
   creates that record if you are using a locally managed tunnel instead.

The service hostname must be `webserver`, not `localhost`. `cloudflared`
runs on the Compose network and reaches Paperless by service name.

Optional hardening: put a Cloudflare Access policy on
`paperless.smmiri.com` so only your account can reach the login page.

## First compile on the Docker server

```bash
git clone https://github.com/smmiri/paperless-ngx.git
cd paperless-ngx
git checkout <branch-or-tag>

cd deploy
cp .env.example .env
python3 -c "import secrets; print(secrets.token_urlsafe(64))"
```

Edit `.env`:

- Set `PAPERLESS_SECRET_KEY` to the generated value. Paperless will
  refuse to start if this is empty or still `change-me`.
- Set a strong `PAPERLESS_DBPASS`.
- Paste `CLOUDFLARE_TUNNEL_TOKEN`. Compose will not start without it.
- Optionally set `PAPERLESS_ADMIN_USER` / `PAPERLESS_ADMIN_PASSWORD`.
- Set `USERMAP_UID` / `USERMAP_GID` if your host user is not `1000`.

Then compile and start:

```bash
mkdir -p consume export
docker compose build
docker compose up -d
docker compose ps
docker compose logs -f webserver cloudflared
```

The first image build compiles the UI and installs Python dependencies.
Later rebuilds reuse Docker layers unless those sources change.

If you skipped the admin env vars:

```bash
docker compose exec webserver createsuperuser
```

Open `https://paperless.smmiri.com` and sign in.

## Drop files for consumption

Host path `deploy/consume` is mounted into the container. Files placed
there are ingested. Exports land in `deploy/export`.

## Rebuild after git pulls

```bash
cd /path/to/paperless-ngx
git pull
cd deploy
docker compose build
docker compose up -d
```

## Common checks

```bash
docker compose exec webserver curl -fsS http://127.0.0.1:8000
docker compose logs --tail=100 cloudflared
```

If the hostname returns 1033 or 502, the tunnel token is missing/wrong or
the public hostname does not target `http://webserver:8000`.
