# Voyage Platform

The whole Voyage Estimation platform in one folder. It is the result of migrations M0–M10: server-authoritative calculation.

How it works (diagrams, flows, edge cases): [ARCHITECTURE.md](ARCHITECTURE.md).

```
voyage-platform/
├── docker-compose.yml     # the full stack
├── .env.example           # copy to .env and fill in
├── ARCHITECTURE.md        # architecture, flows, diagrams, edge cases
├── deploy/edge/nginx.conf # single origin: / → frontend, /api → API replicas
├── frontend/              # React/Vite app (from cozy-crafting-cloud @ migration/M10-cutover)
└── backend/               # Go API + worker (from look-up-service @ feat/sheet-analytics-schema = M10 + migration 0005)
```

## Run

```bash
cp .env.example .env      # then fill in the required secrets
docker compose up -d --build --wait
```

Open http://localhost:8080 and sign in with `ADMIN_EMAIL` / `ADMIN_PASSWORD`.

| URL (localhost only) | What |
|---|---|
| http://localhost:8080 | App: the UI, plus the API under `/api/v1` |
| http://localhost:16686 | Jaeger traces |
| http://localhost:15672 | RabbitMQ management |
| http://localhost:8889/metrics | Prometheus metrics |

Smoke test (needs Go):

```bash
cd backend && SMOKE_EMAIL=... SMOKE_PASSWORD=... go run ./cmd/smoke -base http://127.0.0.1:8080
```

Stop with `docker compose down`. Add `-v` to also delete the data volumes; that cannot be undone.

## Services

| Service | Role |
|---|---|
| `edge` | nginx; the only published app port. Round robin over the API replicas with no sticky sessions. |
| `web` | Static frontend build served by unprivileged nginx. |
| `api-1`, `api-2` | Stateless Go API: REST and the calculation WebSocket. |
| `worker` | Persists calculation events from RabbitMQ into PostgreSQL. Idempotent. |
| `migrate` | One-shot schema migration plus admin seed. The APIs start after it succeeds. |
| `postgres` | The durable source of truth. "Saved" means committed here. |
| `redis` | Sessions, leases, tickets and cache only. |
| `rabbitmq` | Durable persistence events only (not the UI transport). |
| `otel-collector`, `jaeger` | Traces and metrics. |

## Calculation authority

The calculation authority is set by `CALC_AUTHORITY` in `.env`:

- `local`: the browser calculates. This is the default.
- `server_display`: the Go result is shown when it is current for the sheet on screen.
- `server_only`: no browser calculation.

To switch or roll back, change the value and run `docker compose up -d api-1 api-2`. Open pages pick up the change on their next load. Details are in `backend/docs/DEPLOYMENT.md` §8.

## Notes

- **Secrets.** Secrets live only in `.env`, which is gitignored, and are read at runtime. Frontend build args must be public values, because they are compiled into the JS bundle. The frontend build fails if its bundle scan finds a secret.
- **Public URL.** For any URL other than `http://localhost:8080`, set `PUBLIC_ORIGIN` to it. Otherwise the calculation WebSocket is refused. Put TLS termination in front of `edge` for a real deployment.
- **Sea routes.** The sea-route distance service (`SEAROUTE_SERVICE_URL`) is not part of this stack.
- **Development** inside `frontend/` and `backend/` works as before; see each folder's README and `backend/docs/`. The golden/parity scripts expect the original sibling layout (`cozy-crafting-cloud/` next to `Mcs_backend/look-up-service/`).
