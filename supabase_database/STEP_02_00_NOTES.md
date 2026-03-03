# Step 02.00 Notes — Price-Pal DB Schema (Postgres/Supabase)

This step creates the database schema needed for:
- background jobs
- scrape results storage
- price history time-series
- caching with TTL

## What was created in Postgres

Schema:
- `pricepal`

Extensions:
- `pgcrypto`
- `citext`

Enums:
- `pricepal.job_status`
- `pricepal.scrape_status`
- `pricepal.cache_kind`

Tables:
- `pricepal.jobs`
- `pricepal.scrape_results`
- `pricepal.price_history`
- `pricepal.cache_entries`

Indexes:
- worker polling / query patterns / TTL eviction (see `SCHEMA.md`)

Triggers:
- `pricepal.trigger_set_timestamp()` function
- `set_timestamp_jobs` trigger
- `set_timestamp_cache_entries` trigger

RLS:
- Enabled on all four tables (policies intentionally deferred to Supabase migration layer / auth decisions).

## Minimal fixtures inserted

For quick verification, one row was inserted into each table:
- jobs: 1 row
- scrape_results: 1 row
- price_history: 1 row
- cache_entries: 1 row

## Verify via DB Visualizer

1. Ensure Postgres is running (this repo uses the startup script).
2. Start the viewer:

```bash
cd supabase_database/db_visualizer
npm install
source postgres.env
npm start
```

3. Open the viewer UI and inspect schema `pricepal` tables (note: viewer lists tables from `public` by default; if it only shows `public`, you can still query via psql, or adjust the viewer to include non-public schemas in a later step).

## Verify via psql

```bash
psql "$(cat supabase_database/db_connection.txt | sed 's/^psql //')" -c \
"select 'jobs' as table, count(*) from pricepal.jobs
 union all select 'scrape_results', count(*) from pricepal.scrape_results
 union all select 'price_history', count(*) from pricepal.price_history
 union all select 'cache_entries', count(*) from pricepal.cache_entries;"
```
