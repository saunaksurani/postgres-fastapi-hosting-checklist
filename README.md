# Taking a local Postgres + FastAPI app to a hosted environment

A checklist I work through when an application that's only ever run on one machine needs to run somewhere else, with backups and monitoring someone other than me can trust.

Not about picking a cloud provider. That decision matters less than people think and it's usually made for you anyway by whoever owns the billing account.

The order below is roughly the order I do it in. Section 1 is the one people skip.

---

## 1. Your migrations don't describe your database

Start here, before touching any infrastructure. Find out whether the database you're running is actually the database your migration chain produces.

Build a fresh one from migrations alone, dump both schemas, diff them:

```bash
createdb app_from_migrations
DATABASE_URL=postgresql://localhost/app_from_migrations alembic upgrade head

pg_dump --schema-only --no-owner --no-privileges app_live > live.sql
pg_dump --schema-only --no-owner --no-privileges app_from_migrations > migrated.sql
diff <(sort live.sql) <(sort migrated.sql)
```

Sorting first is crude, but it stops pg_dump's ordering from generating noise. You'll still get some. Read it anyway.

On anything that's been alive more than a year or two, that diff won't be empty. What usually shows up:

**Indexes someone added by hand.** Slow query on a Thursday afternoon, index created in psql, query got fast, nobody wrote it down. Most common finding and the worst one, because the app is still correct without it, just unusably slow, so your tests pass and your staging environment looks fine.

**Extensions nobody declared.** `pg_trgm`, `citext`, `uuid-ossp`, `postgis`, `pgcrypto`:

```sql
SELECT extname, extversion FROM pg_extension;
```

Managed Postgres will give you most of these but not all, and the list differs by provider. Check now. If something you depend on isn't available, that's a decision you want to be making this week rather than on migration night.

**Trigger function bodies edited in place.** The trigger itself is in a migration. The function it calls got fixed later with a `CREATE OR REPLACE FUNCTION` typed straight into psql, so your migration chain still produces the old body. This one's genuinely hard to catch in a diff, since both schemas contain a function with the same name. Compare the bodies instead:

```sql
SELECT proname, md5(prosrc) FROM pg_proc
WHERE pronamespace = 'public'::regnamespace ORDER BY proname;
```

**Grants and roles.** `--no-owner --no-privileges` hides these deliberately, which is what you want for a schema diff and not what you want for a migration. Check separately with `\dp` and `\du`. Application roles, read-only reporting roles, whatever the analytics person has been using.

**Sequence values.** Schema-only dumps don't carry `setval`. Rebuild the schema from migrations, copy the data in, and every sequence restarts at 1, so your next insert collides with a primary key from three years ago. Take a full dump or fix the sequences after the load.

Whatever the diff finds becomes a migration, committed, before anything moves.

---

## 2. What the app reads off the filesystem

Anything that ingests or emits files has paths in it, and paths are the first thing to break in a container.

Grep for `/Users`, `/home`, `C:\`, `~/`, `./data`, `tmp`. Also anything writing next to the source tree, which is fine when you run from the repo and fails once the code lives in a read-only image layer.

Object storage is the usual fix and it's more work than it sounds, because the failure modes change. A local write either works or raises straight away. An S3 write can be slow, can be eventually consistent in ways that'll surprise you, and occasionally just needs a retry. If the code assumes a file is there the instant after writing it, go and check that assumption.

If the deadline's tight, a persistent volume mount keeps the paths as they are and gets you hosted. Not where you want to end up, but it buys time.

---

## 3. Config and secrets

Every value the app currently reads off the machine has to come from somewhere else now. Database URL and API keys are obvious. The ones that get missed are defaults.

A settings class with a default baked in works locally, works in staging because staging happens to match it, then does something quietly wrong in production. I'd rather the thing refuse to start:

```python
class Settings(BaseSettings):
    database_url: PostgresDsn          # no default, required
    s3_bucket: str                     # no default, required
    log_level: str = "INFO"            # a real default, safe anywhere
```

Pydantic raises when `Settings()` is constructed, so as long as that happens at startup the process dies immediately rather than running on a wrong value.

Use whatever secret store your provider gives you. The only property that really matters is that rotating a secret doesn't need a code deploy, because the first time you have to rotate one you'll be in a hurry.

---

## 4. Backups aren't done until you've restored one

This is the one that's bitten me most.

A dump job goes green every night for eight months. Then you need it and the restore fails, because an extension isn't present on the target, or it was taken with `--no-owner` and the roles don't exist, or it came off a replica mid-transaction. The job wasn't lying to you. It just wasn't testing the part you cared about.

So restore into a scratch database and run the test suite against it:

```bash
createdb restore_check
pg_restore --no-owner --dbname=restore_check latest.dump
DATABASE_URL=postgresql://localhost/restore_check pytest
```

If the suite passes on a restored copy, the backup is real. Do it on a schedule, not once. A restore that worked in March tells you nothing about last night's dump. Monthly is plenty.

Two other things I'd want:

- **Point-in-time recovery**, not just nightly dumps. Managed Postgres gives you this and honestly it's most of the reason to pay for managed Postgres. The case a nightly dump doesn't cover is a bad `UPDATE` with no `WHERE` clause at 11am, when what you need is the database as it stood at 10:59.
- **Your actual restore time.** Not the RPO written in a document somewhere. Wall-clock minutes to bring a copy of the real data up. Measure it during the restore test, because it's always longer than people assume, and it's the number someone will ask you for while the site is down.

---

## 5. Timezone

Laptop's on a local clock, container's on UTC. Anything involving a window, an age, a cutoff or an eligibility period shifts.

```sql
SHOW timezone;
```

Run it on both.

Then the related question, which is whether your columns are `timestamp` or `timestamptz`. A naive `timestamp` written by an app on a local clock carries no record of which clock that was, and once it's sitting in a UTC environment you can't recover the intent. Anything storing a moment in time should be `timestamptz`. If some of yours aren't, converting them is a data migration with a judgement call inside it, and that call is much easier to make while you still have the machine that wrote the rows.

Set the timezone explicitly in the new environment. Don't inherit whatever the base image happens to use.

---

## 6. Connections

Local Postgres with one process and a handful of connections behaves nothing like managed Postgres with a hard connection limit and several app containers in front of it.

Work out the ceiling: containers × pool size × workers, against `max_connections`. Easy to blow through by accident, and the failure looks strange, because the app stops serving while the database sits there nearly idle.

If you put a pooler in front, know which mode you're in. Transaction pooling is where the throughput is, and it breaks anything assuming session state: prepared statements, session-level `SET`, advisory locks held across statements, `LISTEN`/`NOTIFY`. With asyncpg, which is where most FastAPI apps end up, transaction pooling and the driver's prepared-statement cache don't get along. Turn it off:

```python
create_async_engine(
    url,
    connect_args={"statement_cache_size": 0},   # asyncpg's own cache
    prepared_statement_cache_size=0,            # SQLAlchemy's asyncpg dialect
)
```

Two different caches, confusingly similar names, and you need both off. First is a driver argument so it goes in `connect_args`. Second is a dialect argument and goes on the engine. I've watched people set one and assume they were done.

Miss it and you don't get a clean error. You get something intermittent that looks like a race condition and only shows up under concurrency, which means staging won't find it for you.

---

## 7. Running migrations against a live database

On your laptop a migration runs against a database nobody else is touching. In production it's competing for locks with live traffic.

Two settings at the top of the migration session:

```sql
SET lock_timeout = '5s';
SET statement_timeout = '15min';
```

Without `lock_timeout`, a migration needing `ACCESS EXCLUSIVE` on a busy table will sit and wait for it, and everything behind it queues up too. Five seconds of failure you retry off-peak beats a deploy that takes the site down while it waits politely for a lock.

Index creation wants `CONCURRENTLY` on any table big enough to matter, and `CREATE INDEX CONCURRENTLY` can't run inside a transaction, so in Alembic it needs its own migration with autocommit:

```python
def upgrade():
    with op.get_context().autocommit_block():
        op.create_index("ix_records_source_id", "records", ["source_id"],
                        postgresql_concurrently=True)
```

A `CONCURRENTLY` build can also fail partway and leave an invalid index behind, which won't be used and won't raise. Worth checking after:

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

---

## 8. Monitoring, and the alert everyone forgets

Baseline stuff, none of it controversial: error rate, request latency, pool saturation, replication lag if you have replicas, disk headroom, long-running queries.

```sql
SELECT pid, now() - query_start AS runtime, state, left(query, 120)
FROM pg_stat_activity
WHERE state <> 'idle' AND now() - query_start > interval '1 minute'
ORDER BY runtime DESC;
```

The one that gets left out is **absence alerting**. Everything above fires when something happens. Nothing fires when something fails to happen, and that's a different class of problem: a scheduled job that didn't run produces no error and no log line, a nightly import that stops arriving looks exactly like a quiet night, and a dead queue consumer leaves a queue growing slowly enough that nobody gets paged for a week.

So for anything on a schedule, record when it last finished and alert on the gap:

```sql
CREATE TABLE job_runs (
    job_name    text        NOT NULL,
    started_at  timestamptz NOT NULL DEFAULT now(),
    finished_at timestamptz,
    status      text        NOT NULL DEFAULT 'running',
    detail      jsonb
);
```

Then one check per job asking whether the last successful finish is older than that job's expected interval. Small table, small query. In my experience it catches more real incidents than any dashboard I've built.

---

## Order I actually work in

1. Schema diff against a fresh build from migrations. Commit whatever it turns up.
2. Restore a backup into a scratch database, run the suite against it.
3. Stand the hosted environment up, restore into it, point a staging copy of the app at it.
4. Run the suite against staging. Filesystem and timezone surprises show up here.
5. Backups, retention and PITR configured, restore tested on the real instance, restore time measured.
6. Monitoring and absence alerting in.
7. Cut over.

1 and 2 are the ones people want to skip when the deadline's close, and they're the two that decide how the rest goes.

---

## What this leaves out on purpose

Zero-downtime cutover, multi-region, read replicas, autoscaling. Most systems coming off a single machine don't need any of it yet, and a short maintenance window announced in advance is a perfectly respectable answer. Adding that layer before the basics are solid mostly creates more places for the basics to be wrong.

---

Written by Saunak Surani. Corrections and disagreements welcome.
