# Price-Pal Database Schema (Supabase/Postgres)

This container hosts the PostgreSQL schema for Price-Pal (jobs, scrape results, price history, caching).  
The schema is created under a dedicated namespace:

- Schema: `pricepal`

## Connection

The authoritative connection string for local development is stored in:

- `supabase_database/db_connection.txt`

Example:

```bash
psql "$(cat supabase_database/db_connection.txt | sed 's/^psql //')"
```

## Extensions

- `pgcrypto` (for `gen_random_uuid()`)
- `citext` (case-insensitive text for retailer identifiers)

## Types (Enums)

- `pricepal.job_status`: `queued | running | succeeded | failed | cancelled`
- `pricepal.scrape_status`: `success | partial | no_results | blocked | error`
- `pricepal.cache_kind`: `compare | scrape | price_history | http`

## Tables

### 1) `pricepal.jobs`

Background job queue and status tracking.

Columns (high-level):
- `id uuid PK` (default `gen_random_uuid()`)
- `created_at`, `updated_at`
- `requested_by uuid` (nullable; in Supabase typically references `auth.users.id`)
- `product_query`, `normalized_query`
- `status pricepal.job_status`
- `priority`, `max_attempts`, `attempt_count`
- `locked_at`, `locked_by` (worker coordination)
- `run_after` (schedule/backoff)
- `completed_at`
- `error_code`, `error_message`
- `meta jsonb`

Indexes:
- `(status, run_after)` for worker polling
- `(requested_by, created_at desc)` for per-user history
- `(normalized_query, created_at desc)` for recent job lookup/dedupe

Trigger:
- `updated_at` maintained via `pricepal.trigger_set_timestamp()` on UPDATE

### 2) `pricepal.scrape_results`

Stores per-retailer scrape outputs tied to a job.

Columns (high-level):
- `id uuid PK`
- `job_id uuid FK -> pricepal.jobs(id)` ON DELETE CASCADE
- `product_query`, `normalized_query`
- `retailer citext`
- `status pricepal.scrape_status`
- `product_title`, `product_url`, `image_url`
- `currency_code` (default `INR`)
- `price_amount`, `list_price_amount`
- `in_stock`
- `scraped_at`
- `raw jsonb` (unstructured extraction payload)
- `error_message`

Indexes:
- `(job_id)`
- `(normalized_query, retailer, scraped_at desc)`
- `(retailer, scraped_at desc)`

### 3) `pricepal.price_history`

Time-series price tracking (append-only).

Columns (high-level):
- `id bigserial PK`
- `created_at`
- `job_id uuid FK -> pricepal.jobs(id)` ON DELETE SET NULL
- `normalized_query`
- `retailer citext`
- `product_title`, `product_url`
- `currency_code` (default `INR`)
- `price_amount numeric(12,2)` (NOT NULL)
- `scraped_at`

Constraint:
- `UNIQUE(normalized_query, retailer, product_url, scraped_at)` to reduce accidental duplicates.

Indexes:
- `(normalized_query, retailer, scraped_at desc)` (primary API query pattern)
- `(job_id)`

### 4) `pricepal.cache_entries`

Generic JSON cache with TTL.

Columns (high-level):
- `id uuid PK`
- `created_at`, `updated_at`
- `kind pricepal.cache_kind`
- `cache_key text` (unique with `kind`)
- `normalized_query text` (nullable)
- `retailer citext` (nullable)
- `ttl_seconds int`
- `expires_at timestamptz`
- `payload jsonb`
- `etag text` (optional HTTP cache support)
- `hit_count bigint`
- `last_accessed_at timestamptz`

Constraints / Indexes:
- `UNIQUE(kind, cache_key)`
- `expires_at` (eviction/cleanup)
- `(kind, cache_key)` (lookup)
- partial index `(normalized_query, retailer, expires_at desc) WHERE normalized_query IS NOT NULL`

Trigger:
- `updated_at` maintained via `pricepal.trigger_set_timestamp()` on UPDATE

## Supabase-ready RLS Posture (Assumptions)

RLS is **enabled** on all tables:
- `pricepal.jobs`
- `pricepal.scrape_results`
- `pricepal.price_history`
- `pricepal.cache_entries`

Policies are intentionally **not** created here, because they depend on how the app authenticates and whether you expose these tables directly to the client.

Recommended policy assumptions:
- **Edge Functions / server** use the Supabase **service role** key to bypass RLS for inserts/updates.
- If exposing reads to authenticated users:
  - `jobs`: allow `select` where `requested_by = auth.uid()`
  - `scrape_results`: allow `select` via joining job ownership
  - `price_history`: optionally public-readable (no `requested_by`), or restrict by ownership if you add ownership columns.
  - `cache_entries`: typically server-only; do not allow client access.

## Retention Notes

Suggested cleanup strategy (to implement later via cron/pg_cron or Edge scheduled function):
- `cache_entries`: delete rows where `expires_at < now()`
- `jobs`: optionally delete/compact jobs older than N days where status in (`succeeded`,`failed`,`cancelled`)
- `scrape_results`: optionally delete older than N days (results are persisted in `price_history`)
- `price_history`: usually kept long-term; consider downsampling/partitioning if volume grows.

## Minimal Fixtures (Inserted for verification)

The following fixture IDs were inserted for quick visual verification in the DB viewer:
- job: `11111111-1111-1111-1111-111111111111`
- scrape_result: `22222222-2222-2222-2222-222222222222`
- cache_entry: `33333333-3333-3333-3333-333333333333`

You can verify counts with:

```sql
select 'jobs' as table, count(*) from pricepal.jobs
union all select 'scrape_results', count(*) from pricepal.scrape_results
union all select 'price_history', count(*) from pricepal.price_history
union all select 'cache_entries', count(*) from pricepal.cache_entries;
```
