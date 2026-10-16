# Troubleshooting deployments

Start with `get_deployment(app)` (latest) and read `status`, `error` and `build_log_tail`.

## By error

| `error` | Meaning | What to do |
| --- | --- | --- |
| `source_failed` | The code could not be fetched. | Upload: the `source_key` expired or was never uploaded; get a new URL, upload, deploy. GitHub: the repo or branch is no longer accessible to the Olimpia GitHub App, or the branch does not exist. |
| `build_failed` | Railpack or the Dockerfile build failed. | Read `build_log_tail`. See "Build failures". |
| `import_failed` | The built image could not be stored. | Platform side: deploy again; if it repeats, tell the user to write to hola@olimpia.dev. |
| `release_failed` | The container could not be started or replaced. | Usually the start command exits immediately. Check `get_logs(app, since="5m")`. |
| `internal_error` | Platform error. | Retry once with `deploy(app)`. |

## Build failures

- **No start command / could not determine how to build**: add a `start` script, a `Procfile` or a `railpack.json`, or set `root_dir` for monorepos. See [deploy.md](deploy.md).
- **Lockfile mismatch** (`npm ci`, `pnpm install --frozen-lockfile`, `bun install --frozen-lockfile`): regenerate the lockfile locally, then upload again.
- **TypeScript or lint errors during `build`**: fix them in the code; the build runs in production mode.
- **Missing env var at build time** (for example `NEXT_PUBLIC_*`, or code that reads env vars while building): `set_env`, then deploy.
- **Out of memory / killed**: reduce build parallelism or use a multi-stage Dockerfile.
- **Native modules** (`sharp`, `bcrypt`, `canvas`): usually fine with Railpack; if not, use a Dockerfile based on a full Debian image.

## Runtime failures (status `active` but the app does not work)

`active` only means the container started. Check:

1. `curl -sS -i <url>`.
2. `get_logs(app, since="1h")`, or with `query="level:error"` (also `http.status_code:>=500`, `-healthcheck`, `pattern:<hash>` from `top_patterns`; `around=<timestamp>` for the lines around one). A stack trace comes as one entry. If `covered_from` is present, that app wrote more than the 2000 lines read and older lines of the window were not searched.

Common causes:

- **Bad gateway / no response**: the app listens on `localhost` or a fixed port instead of `0.0.0.0:$PORT`. Fix the code or set `port` to the real one with `update_app` (or `set_env PORT=...`), then deploy.
- **Crash loop**: missing env var or database. `get_env`, then `set_env` / `connect_resource`.
- **Database connection refused**: using a public URL or `localhost` instead of the internal `DATABASE_URL`; or migrations not run.
- **404 on deep links in an SPA**: serve `index.html` as the fallback.

## The Olimpia page instead of the app

A page from Olimpia on `*.olimpia.cc` means there is no live deployment for that subdomain yet, or the subdomain changed. `get_app` shows the current `url`.

## Domains

`check_domain` reports `errors` and the expected `dns_records`. DNS changes can take minutes. A CNAME at the root of a domain needs CNAME flattening or ALIAS support at the DNS provider.
