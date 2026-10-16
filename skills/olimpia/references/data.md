# Postgres, Redis, buckets and email

## Contents
- Connecting apps
- Postgres: ORMs, migrations and SQL
- Postgres backups
- Redis
- Buckets
- Email
- Public access from outside Olimpia

## Connecting apps

`connect_resource(app, kind, name)` (or `create_resource(..., connect_to_app=app)`) writes the variables into the app and redeploys:

| Kind | Variables |
| --- | --- |
| postgres | `DATABASE_URL` (force another name with `env_key`; it replaces a variable with that name) |
| redis | `REDIS_URL` |
| bucket | `AWS_ENDPOINT_URL_S3`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `BUCKET_NAME` |
| email | `EMAIL_API_KEY`, `EMAIL_API_URL`, `EMAIL_DOMAIN` |

When the app already has a variable with a default name, the new ones get the resource name as suffix (`DATABASE_URL_MAIN`); `env_keys` in the reply lists the names used, so point the code at them. Rotating a password or bucket keys updates the apps that use it; they need a deploy to pick it up.

These are internal URLs on Olimpia's private network: no TLS parameters needed, and they do not work from the user's machine. One app can use several resources; a resource can be shared by several apps.

## Postgres

- Version 18. User `app` owns the database `app` (not a superuser: no `CREATE EXTENSION` for untrusted extensions, no `ALTER SYSTEM`). Common trusted extensions such as `pgcrypto`, `citext`, `pg_trgm` and `unaccent` work.
- pgvector: `create_resource(kind="postgres", name, vector=true)` creates it from `pgvector/pgvector:pg18`, so `CREATE EXTENSION vector` works (embeddings, similarity search). The default image (`postgres:18-alpine`) does not have it, and the image cannot change later: if the app needs vectors, pass `vector=true` from the start; for an existing database without it, create a new one and move the data. `get_project` shows each database's `image`.
- Prisma: `DATABASE_URL` works as is. Run migrations at start (`"start": "prisma migrate deploy && node server.js"`): the build runs on separate machines and cannot reach the database.
- Drizzle, Knex, TypeORM, SQLAlchemy, Django, Ecto, sqlx: read `DATABASE_URL`; run migrations on start or as a release step in the start command.
- Inspect with `list_tables` and `query_postgres(database, sql)`: read mode accepts one SELECT/WITH/VALUES query (or EXPLAIN) and returns up to 200 rows or 256 KB as JSON (`truncated: true` when cut).
- `query_postgres(..., write=true, confirm="<database name>")` runs any statements separated by `;` in one transaction with a 30 s timeout, as the `app` role. Use it for quick fixes or seeding; prefer the app's migrations for schema changes. Ask the user before every write and pass `confirm` only after they approve it.

## Postgres backups

Olimpia keeps its own daily backups. On top of that, `set_backups` schedules dumps (`.sql.gz`) of a database into a bucket the user controls:

- **Olimpia bucket:** `set_backups(database, bucket="backups")`. Use an existing bucket of the same project, or create one with `create_resource(kind="bucket")` if the user agrees.
- **External S3** (AWS S3, Cloudflare R2, Backblaze B2, MinIO): `set_backups(database, endpoint="https://<account>.r2.cloudflarestorage.com", s3_bucket, access_key_id, secret_access_key, region?)`. Region defaults to `auto`. The key needs list, read and write on that bucket. Ask the user for the credentials; never invent them.
- `schedule` is a 5-field cron in UTC with a fixed minute, at most hourly (default `0 3 * * *`, daily at 03:00 UTC). `keep` is how many backups to keep, 1-90 (default 7); older ones are deleted.
- Olimpia lists the bucket before saving; `backup_unreachable` means wrong endpoint, region, bucket or key permissions.
- Once configured, omitted fields keep their values: `set_backups(database, schedule="0 */6 * * *")` only changes the schedule.
- `run_backup(database)` makes one now: run it after configuring to verify, and before risky migrations. `get_backups` shows the configuration and the latest files.
- `disable_backups(database, confirm=<name>)` stops the schedule; files already in the bucket stay. Ask first.

To restore, the user downloads a file from the dashboard (Postgres → Backups) or from the bucket. Then, with the target database public (`set_public_access`, ask first, turn it off afterwards): `gunzip -c backup.sql.gz | psql "<public_url>"` with `sslmode=require`.

## Redis

- Version 8, persisted to disk (AOF), no eviction: keys are never dropped, which suits queues (BullMQ, Sidekiq, Celery).
- User `default`; admin commands (`CONFIG`, `ACL`, `SHUTDOWN`) are disabled.
- Use `REDIS_URL` as is (`redis://default:...@host:6379`).

## Buckets

S3-compatible storage (Tigris). Each bucket has its own access key limited to it. Any S3 SDK works:

```js
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
const s3 = new S3Client({});
await s3.send(new PutObjectCommand({ Bucket: process.env.BUCKET_NAME, Key: "a.txt", Body: "hola" }));
```

The AWS SDKs read `AWS_ENDPOINT_URL_S3` and `AWS_REGION` (`auto`) automatically. For browser uploads, generate presigned URLs in the app. `set_public_access(kind=bucket, public=true)` makes objects readable by URL (`public_url` in `get_connection`).

## Email

Transactional email from a domain the user owns. One domain per `create_resource(kind="email", name="mail.example.com")`; prefer a subdomain so its reputation stays apart from the user's mailbox. A domain exists once in all of Olimpia (`domain_taken` otherwise) and cannot be under olimpia.dev or olimpia.cc.

1. `create_resource` returns `dns_records` (DKIM and SPF TXT, an MX, and a recommended `_dmarc` TXT). Names are relative to the domain's zone. Ask the user to create them; you cannot.
2. `get_connection(kind="email", name)` checks them again on every call. `status`: `NOT_STARTED` or `FAILED` (records missing or wrong), `PENDING` (checking), `SUCCESS` (verified). Sending is rejected until `SUCCESS`.
3. `connect_resource(app, kind="email", name)` injects the variables and redeploys.

The app sends with any HTTP client:

```js
const res = await fetch(`${process.env.EMAIL_API_URL}/v1/emails`, {
  method: "POST",
  headers: { Authorization: `Bearer ${process.env.EMAIL_API_KEY}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    from: `App <no-reply@${process.env.EMAIL_DOMAIN}>`,
    to: "user@example.com",
    subject: "Welcome",
    html: "<p>Hi</p>",
  }),
});
const { emailId } = await res.json();
```

- Body: `from` (an address at the key's domain), `to`, `subject`, `html` and/or `text`, optional `cc`, `bcc`, `replyTo`, `headers`, `attachments` (`[{ filename, content: base64 }]`). Unknown fields are rejected.
- 1 to 50 recipients per email (to + cc + bcc). `POST /v1/emails/batch` takes an array of up to 100 emails. `GET /v1/emails/<emailId>` returns its delivery events.
- 3,000 recipients per account per month (`429` beyond that; failed sends do not count).
- Errors are `{ "error": { "code", "message" } }`: `401` bad key, `403` wrong sender domain, unverified or suspended domain, `400` invalid body.
- A domain is suspended automatically when, with 50+ emails in the last 7 days, bounces exceed 10 % or complaints 0.3 %; only Olimpia support reactivates it. Send only to addresses that signed up, and do not retry addresses that bounced.
- `get_usage` reports the recipients sent this month. Never print `EMAIL_API_KEY`; it is a secret.

## Public access from outside Olimpia

By default Postgres and Redis only accept connections from apps on Olimpia. To connect from a laptop, a BI tool or another cloud, ask the user, then `set_public_access(kind, name, public=true)` and use `public_url` from `get_connection`:

- Host `pg-<id>.db.olimpia.cc:5432` / `rd-<id>.db.olimpia.cc:6379`, TLS required with SNI.
- Postgres: `sslmode=require`, libpq 14 or newer (`psql`, recent drivers).
- Redis: `rediss://` scheme.

Turn it off again when it is no longer needed.
