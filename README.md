# proxy-stack

Traefik + cloudflared + authentik edge for `*.ghoulhub.uk`.

    Internet → Cloudflare → cloudflared (tunnel) → traefik (websecure, LE certs) → app

- `traefik.yml` + `docker-compose.yaml`: traefik with DNS-01 LE certs (`CF_DNS_API_TOKEN` injected from `.env`), the shared middlewares (`rate-limit`, `security-headers`, `authentik-forwardauth`) and the `authentik-outpost` service.
- `/etc/cloudflared/config.yml` (systemd unit `cloudflared`): server-side ingress, one block per hostname, all pointing at `https://localhost:443`.

## Adding a new app to the public chain (runbook)

Verified end-to-end on shopping-list (2026-10-11) — it was added with exactly these steps.

### 1. Traefik — labels in the app's docker-compose.yaml

The app's frontend service joins the external `proxy` network:

```yaml
networks:
  proxy:
    external: true
```

Then add the openwebui-shape labels (`<name>` = short router name, `<sub>` = subdomain). Reference block — shopping-list, exactly as deployed:

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=proxy
  - traefik.http.routers.shoppinglist.rule=Host(`shopping-list.ghoulhub.uk`)
  - traefik.http.routers.shoppinglist.entrypoints=websecure
  - traefik.http.routers.shoppinglist.tls.certresolver=le
  - traefik.http.routers.shoppinglist.middlewares=rate-limit,security-headers,authentik-forwardauth
  - traefik.http.routers.shoppinglist-outpost.rule=Host(`shopping-list.ghoulhub.uk`) && PathPrefix(`/outpost.goauthentik.io/`)
  - traefik.http.routers.shoppinglist-outpost.entrypoints=websecure
  - traefik.http.routers.shoppinglist-outpost.tls.certresolver=le
  - traefik.http.routers.shoppinglist-outpost.priority=15
  - traefik.http.routers.shoppinglist-outpost.service=authentik-outpost
```

Notes:

- Drop `,authentik-forwardauth` from the middlewares if the app does its own auth (see 4 / Option B).
- The companion `-outpost` router is REQUIRED whenever `authentik-forwardauth` is used — it serves the `/outpost.goauthentik.io/` redirect flow (priority 15, `service=authentik-outpost`).
- The shared pieces are defined once in proxy-stack's `docker-compose.yaml` and apps only reference them by name: `security-headers` (STS 31536000, nosniff, frameDeny, …), `rate-limit` (`average=60`, `burst=100`, `period=1m`), `authentik-forwardauth` (`forwardauth.address=http://authentik-server:9000/outpost.goauthentik.io/auth/traefik`, `trustForwardHeader=true`, authResponseHeaders `X-authentik-*`), and `authentik-outpost` (`loadbalancer.server.url=http://authentik-server:9000`).

### 2. Cloudflared — server-side ingress

Back up first, then add the block to `/etc/cloudflared/config.yml` BEFORE the catch-all (`- service: http_status:404`):

```bash
sudo cp /etc/cloudflared/config.yml /etc/cloudflared/config.yml.bak-$(date +%F)
sudoedit /etc/cloudflared/config.yml
```

```yaml
  - hostname: <sub>.ghoulhub.uk
    service: https://localhost:443
    originRequest:
      originServerName: <sub>.ghoulhub.uk
```

Validate, restart, check:

```bash
sudo python3 -c "import yaml; yaml.safe_load(open('/etc/cloudflared/config.yml'))"
sudo systemctl restart cloudflared
systemctl is-active cloudflared   # → active
```

### 3. DNS — the CF_DNS_API_TOKEN way

The DNS-01 token lives in `proxy-stack/.env` as `CF_DNS_API_TOKEN` (referenced in `docker-compose.yaml` as `CF_DNS_API_TOKEN: ${CF_DNS_API_TOKEN}`). NEVER print it. Run the two calls inside the traefik container (where the env var exists), or `set -a; . ~/Documents/proxy-stack/.env; set +a` in a shell without echoing it:

```bash
# 1) zone id
curl -s -H "Authorization: Bearer $CF_DNS_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=ghoulhub.uk"

# 2) CNAME → tunnel (tunnel id comes from /etc/cloudflared/config.yml)
TUNNEL_ID=$(sudo grep -oP 'tunnel: \K[a-f0-9-]+' /etc/cloudflared/config.yml)
curl -s -X POST -H "Authorization: Bearer $CF_DNS_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"type":"CNAME","name":"<sub>","content":"'"$TUNNEL_ID"'.cfargotunnel.com","proxied":true}' \
  "https://api.cloudflare.com/client/v4/zones/<zone-id>/dns_records"

dig @1.1.1.1 +short <sub>.ghoulhub.uk
```

Alternative (only if a credentials cert.pem ever exists): `cloudflared tunnel route dns <tunnel-id> <sub>.ghoulhub.uk`.

### 4. Authentik — two options

**Option A — no authentik changes (what shopping-list uses).** Put `authentik-forwardauth` in the app's middlewares list (step 1): authentik's embedded outpost protects the whole app at the proxy, so every visitor needs an authentik login. The outpost router labels from step 1 are REQUIRED for the redirect flow.

**Option B — app-specific SSO (browser UI only — cannot be done via shell).** In the authentik admin UI (https://sso.ghoulhub.uk): Applications → Create → choose the provider type (Proxy Provider for the forward-auth shape, or OAuth2/OpenID Provider if the app speaks OAuth) → set the redirect / post-logout URLs to the app's domain → copy the client id/secret into the app's `.env` (gitignored) → if a new outpost is needed: Admin → Outposts → integrate.

### 5. Secrets

- `.env` is gitignored (`.env`, `.env.*` with `!.env.example`); commit `.env.example` with placeholders instead.
- Strong FAMILY_KEY / API keys: `openssl rand -hex 24`.

### 6. Test checklist

```bash
curl -s http://127.0.0.1:<loopback>/          # app serves on its loopback port
curl -sk --resolve <sub>.ghoulhub.uk:443:127.0.0.1 \
  https://<sub>.ghoulhub.uk/ -o /dev/null -w "%{http_code}\n"
#   302 → forwardauth in front; 200 → app-auth
curl https://<sub>.ghoulhub.uk/               # after DNS record exists
docker compose up -d --build
```

- Add the data volume + kopia source (see backup-stack).
