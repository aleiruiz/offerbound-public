# Offerbound

A self-hosted job-search tracker. It reads your job-related Gmail, keeps every application up to date on its own, and finds and ranks new leads for you.

- **Gmail triage.** Read-only Gmail access. Confirmations, rejections, interview invites and recruiter messages are sorted and applied to the right application. Non-job email is never sent in full to the AI.
- **AI search and ranking.** A search agent looks for roles that match your brief and CV, and ranks every lead by fit.
- **Dead leads stay out of the way.** Each job link is checked, and leads whose posting has closed are hidden.
- **Spreadsheet import.** Bring your existing tracker as a CSV; rows already found in Gmail are skipped.
- **English and Spanish (Mexico)** interface.

Offerbound is single-user: one instance serves one Google account (`OWNER_EMAIL`).

## Requirements

- Docker with Compose
- For local development: Node 22.19+ and pnpm 9.15.9 (`corepack enable`)
- For your own instance: a Google Cloud OAuth client and an OpenAI API key (below)

## Try it in demo mode

Demo mode needs no Google or OpenAI credentials. It loads synthetic data and fakes the AI responses.

```sh
cp .env.example .env
# In .env: set APP_MODE=demo and PUBLIC_BASE_URL=http://localhost:5175,
# and leave OWNER_EMAIL, GOOGLE_*, TOKEN_ENCRYPTION_KEY, OPENAI_API_KEY and AI_* empty.
docker compose up -d                 # Postgres 17 on localhost:5432
pnpm install
set -a; . ./.env; set +a
pnpm db:generate && pnpm db:migrate

# Three terminals (each with the .env exported as above):
cd apps/api && PORT=3002 pnpm dev                                          # API
cd apps/api && npx tsx --conditions=@offerbound/source src/worker.ts       # worker (seeds the demo)
cd apps/web && pnpm dev                                                    # web on http://localhost:5175
```

Open http://localhost:5175.

## Set up your own instance

### 1. Google OAuth client (Gmail)

In [Google Cloud Console](https://console.cloud.google.com/):

1. Create a project and enable the **Gmail API**.
2. **Google Auth Platform**, Branding: set an app name and support email.
3. Data access: add the scopes `openid`, `email`, `profile` and `https://www.googleapis.com/auth/gmail.readonly`. Nothing else.
4. Audience: **External**, then **Publish app** so refresh tokens don't expire after 7 days. The app stays unverified, so Google shows an "unverified app" screen when you sign in. Choose _Advanced_, then continue.
5. Credentials, **OAuth client ID**, type **Web application**. Add the redirect URI `<PUBLIC_BASE_URL>/api/v1/auth/google/callback`, for example `https://jobs.example.com/api/v1/auth/google/callback` (and `http://localhost:5175/api/v1/auth/google/callback` for local development).
6. Copy the client ID and secret.

### 2. OpenAI API key

Create a dedicated project at [platform.openai.com](https://platform.openai.com/) and an API key in it. Note the per-million-token prices of the models you use and the price of one web search call; Offerbound uses them to track what it spends.

### 3. Environment variables

Copy `.env.example` to `.env` (local) or set these in your hosting panel (production). Keep secrets out of git.

| Variable                                   | Required   | What it is                                                                                                                                                               |
| ------------------------------------------ | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `APP_MODE`                                 | yes        | `private` for your instance, `demo` for the credential-free demo                                                                                                         |
| `PUBLIC_BASE_URL`                          | yes        | The URL you open the app at; used for OAuth redirects and cookies                                                                                                        |
| `DATABASE_URL`                             | local only | Postgres URL; `compose.prod.yaml` builds it from `SERVICE_PASSWORD_POSTGRES`                                                                                             |
| `OWNER_EMAIL`                              | yes        | The only Google account allowed to sign in                                                                                                                               |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | yes        | From step 1                                                                                                                                                              |
| `TOKEN_ENCRYPTION_KEY`                     | yes        | 32 random bytes, base64: `openssl rand -base64 32`. Encrypts the stored Google tokens; keep a copy, a lost key means reconnecting Gmail                                  |
| `OPENAI_API_KEY`                           | yes        | From step 2                                                                                                                                                              |
| `AI_PRICING`                               | yes        | One line of JSON keyed by model: `{"<model>":{"inputPerMTokMicros":n,"cachedInputPerMTokMicros":n,"outputPerMTokMicros":n}}` (millionths of a dollar per million tokens) |
| `AI_WEB_SEARCH_COST_PER_CALL_MICROS`       | yes        | Price of one web search call, e.g. `10000` = $0.01                                                                                                                       |
| `USER_TIMEZONE`, `HOME_COUNTRY`            | no         | Default `America/Monterrey`, `MX`                                                                                                                                        |
| `GMAIL_BACKFILL_SINCE`                     | no         | How far back the first sync reads, e.g. `2026-09-01`                                                                                                                     |

Optional tuning (models per purpose, reasoning effort, sync interval, search limits, backups, logging) is listed in `.env.example` and [docs/deploy.md](docs/deploy.md). The API refuses to start if a required variable is missing and names it in the log.

### 4. Deploy

Production runs from `compose.prod.yaml`: Postgres, the API (runs migrations on start), a background worker, and Caddy serving the web app on port 80. No service publishes a port; put it behind a reverse proxy that terminates TLS and points at the `web` service. Coolify generates `SERVICE_PASSWORD_POSTGRES` for you; elsewhere set it yourself.

Step-by-step guide for Coolify: [docs/deploy.md](docs/deploy.md). Restores and other operations: [docs/operations.md](docs/operations.md).

## Using it

1. Open your instance and **sign in with Google**, granting read-only Gmail access.
2. Press **Start syncing**. Offerbound backfills your email and shows progress while you continue.
3. Upload your **CV** so leads can be ranked against it.
4. Optionally **import your spreadsheet** as CSV (columns _Company_, _Role_, _Status_).
5. Paste a **search brief** describing what you're looking for.

From then on, **Today** shows what needs you, **Applications** tracks each process with its email timeline, **Leads** lists new roles ranked by fit (mark one _No longer available_ to dismiss it), and **Review** holds the emails Offerbound wasn't sure about. Switch between English and Spanish on the sign-in page or in **Settings**.

## Development

```sh
cp .env.example .env
docker compose up -d
pnpm install
pnpm test                 # unit then integration tests
pnpm lint && pnpm typecheck && pnpm format:check
```

Product, design and technical specs live in [`docs/`](docs/).
