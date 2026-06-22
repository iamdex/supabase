# Supabase on Dokploy

Self-hosted Supabase tuned for a **VPS running [Dokploy](https://dokploy.com)**.
One domain, automatic HTTPS via Dokploy's Traefik, Auth + Storage + REST +
Realtime + Edge Functions + Studio all working out of the box. Data persists in
named Docker volumes across redeploys.

## How it works

```
Internet ──HTTPS──> Traefik (Dokploy) ──> Kong :8000 ──┬─ /            Studio dashboard (basic-auth)
  ${SUPABASE_DOMAIN}                                    ├─ /auth/v1     GoTrue (Auth)
                                                        ├─ /rest/v1     PostgREST
                                                        ├─ /storage/v1  Storage
                                                        ├─ /realtime/v1 Realtime (websockets)
                                                        └─ /functions/v1 Edge Functions
```

Kong is the only service exposed to Traefik. Everything else talks over the
internal compose network. Postgres is reachable directly on the VPS through
Supavisor (`:5432` session / `:6543` transaction) — optional, can be disabled.

## Prerequisites

- A VPS with Dokploy installed (`curl -sSL https://dokploy.com/install.sh | sh`).
  The installer creates the external `dokploy-network` and runs Traefik with a
  `letsencrypt` cert resolver — this stack relies on both.
- A domain with a DNS **A record** pointing to the VPS IP, e.g.
  `supabase.espertini.com -> 203.0.113.10`.
- Open ports `80` and `443` on the VPS firewall (Traefik). Open `5432`/`6543`
  only if you want external Postgres access.

## Setup

### 1. Generate secrets

On any machine with `openssl`, from the repo `docker/` directory:

```bash
sh utils/generate-keys.sh
```

Copy the printed values into a copy of
[`.env.dokploy.example`](.env.dokploy.example). At minimum replace:
`POSTGRES_PASSWORD`, `JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY`,
`SECRET_KEY_BASE`, `VAULT_ENC_KEY`, `PG_META_CRYPTO_KEY`,
`S3_PROTOCOL_ACCESS_KEY_ID`, `S3_PROTOCOL_ACCESS_KEY_SECRET`, and a strong
`DASHBOARD_PASSWORD`.

### 2. Set the domain

Supabase API lives on `supabase.espertini.com`; the app/redirect target is
`store.espertini.com`:

```env
SUPABASE_DOMAIN=supabase.espertini.com
SUPABASE_PUBLIC_URL=https://supabase.espertini.com
API_EXTERNAL_URL=https://supabase.espertini.com
SITE_URL=https://store.espertini.com
ADDITIONAL_REDIRECT_URLS=https://store.espertini.com,https://store.espertini.com/**
```

`SUPABASE_DOMAIN` is the only host Traefik routes — point its DNS A record at
the VPS. `store.espertini.com` is just an allowed auth redirect target; it does
**not** need to resolve to this VPS.

### 3. Configure SMTP (for real Auth emails)

Auth email confirmations / password resets need a real SMTP provider. Fill the
`SMTP_*` block (Resend, Postmark, SES, Mailgun…). For a quick test without
email, set `ENABLE_EMAIL_AUTOCONFIRM=true` instead.

### 4. Create the Dokploy service

In the Dokploy dashboard:

1. **Create → Compose** (Docker Compose, not Application).
2. **Provider:** point it at this Git repo + branch.
3. **Compose Path:** `docker/dokploy/docker-compose.yml`
4. **Environment** tab: paste your finished `.env` contents.
5. **Deploy.**

Traefik picks up the labels on the `kong` service automatically and issues the
Let's Encrypt certificate for `${SUPABASE_DOMAIN}`. No Domain entry needs to be
added in the Dokploy UI — routing is defined by the compose labels.

> First boot takes a minute or two while Postgres runs the init scripts and the
> other services become healthy.

## Verify

- Dashboard: `https://supabase.espertini.com` → prompts for
  `DASHBOARD_USERNAME` / `DASHBOARD_PASSWORD`.
- Auth health: `https://supabase.espertini.com/auth/v1/health` → `{"...":...}`.
- REST: `https://supabase.espertini.com/rest/v1/` with header
  `apikey: <ANON_KEY>`.
- Storage: create a bucket in Studio → upload a file → it persists in the
  `storage-data` volume.

## Connecting an app

```ts
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  'https://supabase.espertini.com',
  '<ANON_KEY>'
)
```

Direct Postgres (if Supavisor ports are exposed):

```
postgresql://postgres.<POOLER_TENANT_ID>:<POSTGRES_PASSWORD>@<VPS_IP>:6543/postgres
```

## Persistence & backups

Data lives in named volumes: `db-data` (Postgres), `storage-data` (uploaded
files), `db-config`, `studio-snippets`, `deno-cache`. These survive redeploys
and `compose down`. They are **deleted** by `docker compose down -v` — don't run
that unless you mean it.

Back up the database regularly:

```bash
docker exec supabase-db pg_dumpall -U postgres > backup.sql
```

## Customizing

- **Edge Functions:** add folders under `docker/volumes/functions/<name>/` and
  redeploy. `main` is the router; `hello` is an example.
- **OAuth providers:** uncomment the `GOTRUE_EXTERNAL_*` lines in
  [`docker-compose.yml`](docker-compose.yml) (auth service) and set the matching
  `*_CLIENT_ID` / `*_SECRET` in your env. Redirect URI is
  `https://${SUPABASE_DOMAIN}/auth/v1/callback`.
- **Lock down the database:** comment out the `ports:` block on the `supavisor`
  service to keep Postgres private to the stack.
- **S3-backed storage:** the default is local `file` storage on the
  `storage-data` volume. To use an external S3 bucket, set `STORAGE_BACKEND=s3`
  and the `GLOBAL_S3_*` / `AWS_*` vars on the `storage` service.

## Security checklist

- [ ] Every secret replaced with generated values (no defaults left).
- [ ] Strong `DASHBOARD_PASSWORD`.
- [ ] HTTPS working (Traefik cert issued).
- [ ] Supavisor ports closed if external DB access isn't needed.
- [ ] SMTP configured so password resets actually send.
- [ ] `DISABLE_SIGNUP=true` if you don't want open public registration.
