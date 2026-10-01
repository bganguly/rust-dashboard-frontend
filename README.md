# rust-dashboard-frontend — React 19 + TypeScript + AWS App Runner

Production-grade **React 19 / TypeScript** SPA for the orders dashboard, delivering sub-second
search and chart responses across 4 M+ orders. Served via multi-stage Docker build (Vite → Nginx),
deployed as an AWS App Runner service. Nginx acts as a BFF proxy — routing `/api/*` to the Rust backend.

---

## Live Service

| Endpoint | URL |
|---|---|
| **App** | available on demand via `deploy.sh` |
| **Portfolio demo** | https://bganguly.github.io/#rust_dashboard |

> App Runner scales to zero when idle; run `deploy.sh` to provision AWS infrastructure and start the service.

---

## Using the App

1. **Search** — type in the search bar to query orders across all columns (name, notes, total, order ID, status, region, date) via the backend's `search_text` GIN trigram index; sub-second on 4 M+ rows.
2. **Filter** — use the sidebar to narrow by status, region, date range, or total amount; filters compose with search.
3. **Aggregates chart** — stacked bar chart of daily orders by product category; drag the brush to zoom into any date window.
4. **Dark mode** — system-preference detection via `useIsDark` hook; persisted to `localStorage`.
5. **API Explorer** — open `/api-explorer` to run live requests against every backend endpoint from the browser; no curl required.

---

## Architecture

### Topology

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              AWS Account                                │
│                                                                         │
│   ECR (Elastic Container Registry)                                      │
│   ┌──────────────────┐    ◄── AWS CodeBuild (deploy.sh, S3 source)     │
│   │  frontend image  │         Vite → Nginx multi-stage                │
│   │  backend image   │                                                  │
│   └──────────────────┘                                                  │
│           │ image pull                                                  │
│           ▼                                                             │
│   App Runner: rust-dash-frontend                                        │
│   ┌─────────────────────────┐                                           │
│   │ Nginx (port 8080)       │       App Runner: rust-dash-backend       │
│   │ • serves Vite dist      │       ┌──────────────────────┐           │
│   │ • proxies /api/* ───────┼──────►│ Rust / Actix-web 4   │           │
│   │                         │ HTTPS │ • REST /api/*        │           │
│   │ • 1–2 instances         │       │ • sqlx migrations    │           │
│   └─────────────────────────┘       │ • 1–2 instances      │           │
│           ▲                         └──────────┬───────────┘           │
│           │ HTTPS                              │                        │
│       Browser                    ┌─────────────▼──────────┐            │
│                                  │  Neon serverless PG     │            │
│                                  │  4 M+ orders            │            │
│                                  │  GIN trigram index      │            │
│                                  │  pre-agg summary tables │            │
│                                  └────────────────────────┘            │
└─────────────────────────────────────────────────────────────────────────┘

Deploy flow
───────────
local machine
  └─ deploy.sh
       ├─ [1] local  → Vite dev server on :5173
       └─ [2] AWS    → zip source → S3 → CodeBuild → ECR
                       → aws apprunner update-service rust-dash-frontend
                         with BACKEND_URL env var → Nginx template substitution
```

### Key design decisions

| Concern | Approach |
|---|---|
| **BFF proxy** | Nginx forwards `/api/*` to Rust backend via `${BACKEND_URL}` env var substituted at container start via `nginx.conf.template`; browser sees a single origin, no CORS. |
| **Image build** | AWS CodeBuild via S3 source upload — no local Docker required. Content-hash tag skips rebuilds when source is unchanged. |
| **Search** | GIN trigram index on denormalized `search_text` column; sub-second on 4 M+ rows. |
| **Aggregates** | Pre-aggregated summary tables — chart queries never hit the raw `orders` table. |
| **Pagination** | Keyset cursor `(placedAt, orderId)` — O(1) deep-page navigation, no OFFSET scans. |
| **IaC** | `aws apprunner update-service` direct from `deploy.sh` — no Pulumi or Terraform required. |

---

## Stack

| Component | Implementation |
|---|---|
| **React / TypeScript front-end** | React 19, TypeScript 5.7, Vite 6, Recharts 3.8 |
| **BFF layer** | Nginx reverse proxy — `/api/*` → Rust backend via `${BACKEND_URL}` env var |
| **Serverless / cloud-native** | AWS App Runner — 1–2 instances, ECR image source, auto-deploy disabled |
| **Image build** | AWS CodeBuild via S3 source upload — remote build, no local Docker |
| **Performance** | Sub-second chart from pre-aggregated tables; sub-second search via GIN trigram index on `search_text` |

---

## Deployment / Running

```bash
./scripts/deploy.sh      # [1] local dev · [2] AWS App Runner
./scripts/infra-down.sh  # teardown AWS stack
```

| Action | Script | Prompt |
|---|---|---|
| Start local dev server (port 5173) | `./scripts/deploy.sh` | `[1]` |
| Deploy to AWS App Runner | `./scripts/deploy.sh` | `[2]` |
| Teardown AWS stack | `./scripts/infra-down.sh` | — |

Deploy backend first (`rust-dashboard-backend`) before deploying this service — `deploy.sh` reads the backend's `.env.aws` file for `BACKEND_URL`.

Override the backend target for local dev:

```bash
BACKEND_URL=http://localhost:8080 ./scripts/deploy.sh
```

### Cost

| Resource | Cost |
|---|---|
| **App Runner** | Min 1 instance — ~$5–7/mo at idle |
| **Neon Postgres** | Free tier — auto-suspends when idle |
| **ECR** | Negligible at demo image count |
| **CodeBuild** | Free tier covers demo-frequency builds |

---

## Scale & Performance

> **4 M+ orders** served with sub-second search and chart responses. Full-text search hits a single GIN trigram index on `search_text`; chart aggregates hit pre-aggregated summary tables — neither touches the raw `orders` table on the hot path.

```
Browser ──HTTPS──► Nginx / App Runner ──proxy /api/*──► Rust / Actix-web 4 (App Runner) ──► Neon PG
                   rust-dash-frontend                    rust-dash-backend                   4 M+ rows
                   1–2 instances                         1–2 instances                       GIN trigram index
```

---

## Features

- **Orders table** — paginated (keyset cursor), sortable (ID / customer / total / date), filter sidebar (status, region, date range, total range)
- **Full-text search** — multi-token AND search across all visible columns via backend `search_text` GIN trigram index; sub-second on 4 M+ rows
- **Aggregates chart** — stacked bar chart of daily orders by product category; sub-second from pre-aggregated tables, never queries raw orders
- **Date brush** — Recharts brush on the aggregates chart; drag to zoom into any date window
- **Dark mode** — system-preference detection via `useIsDark` hook; light / dark / system toggle
- **API Explorer** — embedded request runner at `/api-explorer`; proxied through Nginx BFF, no CORS
- **BFF proxy** — Nginx forwards `/api/*` to Rust backend via `${BACKEND_URL}`; browser sees a single origin, no CORS
