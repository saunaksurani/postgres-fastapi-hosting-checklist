# Taking a local Postgres + FastAPI app to a hosted environment

A checklist I work through when a working application that has only ever run on one machine needs to run somewhere else, with backups and monitoring that someone other than me can trust.

This is not about choosing a cloud provider. That decision matters less than people think and it is usually made for you by whoever already owns the billing account. This is about the things that are true on the laptop and quietly stop being true in a container.

The order below is roughly the order I do it in. The first section is the one people skip and it is the one that costs the most.

---

## 1. Your migrations do not describe your database

Start here. Before touching infrastructure, find out whether the database you are running is the database your migration chain produces.

Build a fresh database from migrations alone, then dump both schemas and diff them:

```bash
createdb app_from_migrations
DATABASE_URL=postgresql://localhost/app_from_migrations alembic upgrade head

pg_dump --schema-only --no-owner --no-privileges app_live > live.sql
pg_dump --schema-only --no-owner --no-privileges app_from_migrations > migrated.sql
diff <(sort live.sql) <(sort migrated.sql)
```

Sorting before the diff is crude but it stops pg_dump's ordering from generating noise. On a mature schema you will still get a fair amount of it. Read the whole thing anyway, once, slowly.

On any system that has been alive for more than a year or two, that diff is not empty. What usually turns up:

**Indexes added by hand.** Someone had a slow query on a Thursday afternoon, added an index in psql, and the query got fast so nobody wrote it down. These are the most common finding and the most dangerous, because the application will be correct without them and unusably slow.

**Extensions nobody declared.** `pg_trgm`, `citext`, `uuid-ossp`, `postgis`, `pgcrypto`. Check directly:

```sql
SELECT extname, extversion FROM pg_extension;
```

Managed Postgres will let you install most of these but not all of them, and the list differs by provider. Find out now rather than at cutover. If you depend on something your target does not offer, that is a decision to make this week, not on migration night.

**Trigger function bodies edited in place.** The trigger is in the migration. The function it calls was fixed later with a `CREATE OR REPLACE FUNCTION` typed straight into psql. The migration chain still produces the old body. This one is genuinely hard to spot in a diff because both schemas contain a function with the same name, so compare the bodies:

```sql
SELECT proname, md5(prosrc) FROM pg_proc
WHERE pronamespace = 'public'::regnamespace ORDER BY proname;
```

Run it on both and diff the hashes.

**Grants and roles.** `--no-owner --no-privileges` deliberately hides these, which is what you want for a schema diff and not what you want for a migration. Check them separately with `\dp` and `\du` in psql. Application roles, read-only reporting roles and whatever the analytics person is using all need to exist on the other side.

**Sequence values.** Schema-only dumps do not carry `setval`. If you rebuild the schema from migrations and then copy data in, every sequence restarts at 1 and your next insert collides with a primary key from three years ago. Either take a full dump, or fix the sequences explicitly after the data load.

Whatever the diff shows becomes a migration, committed, before anything moves. The goal is that the migration chain is the truth by the time you cut over, because from then on it is the only thing the new environment will ever be built from.

---

## 2. What the application reads from the filesystem

Anything that ingests or emits files has paths in it, and those paths are the first thing to break in a container.

Grep for them. `/Users`, `/home`, `C:\`, `~/`, `./data`, `tmp`. Also look for anything writing next to the source tree, which works fine when you run from the repo and fails the moment the code lives in a read-only image layer.

The fix is usually object storage, which is more work than it sounds because it changes the failure modes. A local file write either works or raises immediately. An S3 write can be slow, can be eventually consistent in ways that surprise you, and will occasionally fail in a way that needs a retry. If the application currently assumes a file is there the instant after it wrote it, that assumption needs checking.

Interim option if the deadline is tight: a persistent volume mount, keeping the paths as they are. It is not the destination but it gets you hosted without a rewrite of the ingestion path, and it buys you the time to do that properly.

---

## 3. Configuration and secrets

Every value the application reads from the machine needs to come from somewhere else now. The obvious ones are the database URL and API keys. The ones that get missed are the defaults.

A settings class with a default baked in is a config value that works locally, works in staging because staging happens to match the default, and then does something quietly wrong in production. I would rather the application refuse to start:

```python
class Settings(BaseSettings):
    database_url: PostgresDsn          # no default, required
    s3_bucket: str                     # no default, required
    log_level: str = "INFO"            # a real default, safe anywhere
```

Pydantic raises when `Settings()` is constructed, so as long as that happens at startup the process dies immediately instead of running with a wrong value. That is the right time to find out.

For the secret store itself, use whatever your provider gives you. The one property that matters is that rotating a secret does not require a code deploy, because the first time you need to rotate one you will be in a hurry.

---

## 4. Backups are not done until you have restored one

This is the one I have been bitten by most.

A backup job goes green every night for eight months. Then you need it, and the restore fails because an extension is not present on the target, or the dump was taken with `--no-owner` and the roles do not exist, or it was taken from a replica mid-transaction. The job was never lying. It just was not testing the thing you cared about.

So: restore into a scratch instance and run the test suite against the restored copy. If the suite passes on a restored database, the backup is real.

```bash
createdb restore_check
pg_restore --no-owner --dbname=restore_check latest.dump
DATABASE_URL=postgresql://localhost/restore_check pytest
```

Do this on a schedule, not once. A restore that worked in March is not evidence about the dump you took last night. Monthly is enough for most systems.

Two further things worth having:

- **Point-in-time recovery**, not just nightly dumps. Managed Postgres gives you this and it is most of the reason to pay for managed Postgres. The scenario a nightly dump does not cover is a bad `UPDATE` without a `WHERE` clause at 11am, where what you need is the database as it was at 10:59.
- **Know your actual restore time.** Not the RPO you wrote in a document, the wall-clock minutes it takes to bring a copy of your real data up. Measure it during the restore test. It is usually longer than people assume, and it is the number that matters when someone asks how long you will be down.

---

## 5. Timezone

The laptop is on a local clock. The container is on UTC. Any calculation involving a window, an age, a cutoff or an eligibility period shifts.

```sql
SHOW timezone;
```

Run it on both. If they differ, work out what that does to your data before you move.

The related question is whether columns are `timestamp` or `timestamptz`. A naive `timestamp` written by an application running on a local clock carries no record of which clock that was, and once it is in a UTC environment there is no way to recover the intent. Columns storing a moment in time should be `timestamptz`. If some of yours are not, changing them is a data migration with a judgement call in it, and it is much easier to make that call while you still have the machine that wrote them.

Set the timezone explicitly in the new environment rather than accepting whatever the base image gives you. Inherited defaults are how this comes back in six months.

---

## 6. Connections

Local Postgres with one process and a handful of connections behaves nothing like managed Postgres with a connection limit and several application containers.

Work out your ceiling: containers × pool size × workers, against the instance's `max_connections`. It is easy to exceed by accident, and the failure is the application refusing to serve while the database sits nearly idle.

If you put a pooler in front, know which mode you are running. Transaction pooling is where the throughput is, and it breaks anything that assumes session state: prepared statements, `SET` at session level, advisory locks held across statements, `LISTEN`/`NOTIFY`. With asyncpg, which is what FastAPI applications usually end up on, transaction pooling and the driver's prepared-statement cache do not get along, and you want the cache off:

```python
create_async_engine(
    url,
    connect_args={"statement_cache_size": 0},   # asyncpg's own cache
    prepared_statement_cache_size=0,            # SQLAlchemy's asyncpg dialect
)
```

Those are two different caches with confusingly similar names, and you need both off. The first is a driver argument and goes in `connect_args`. The second is a dialect argument and goes on the engine.

The symptom if you miss this is not a clean error. It is intermittent, it looks like a race condition, and it only appears under concurrency, which means staging will not show it to you.

---

## 7. Running migrations against a live database

On the laptop, a migration runs against a database nobody else is using. In production it competes for locks with live traffic.

Two settings to put at the top of the migration session:

```sql
SET lock_timeout = '5s';
SET statement_timeout = '15min';
```

Without `lock_timeout`, a migration that needs `ACCESS EXCLUSIVE` on a busy table will sit and wait, and every query behind it queues too. A five-second failure that you retry off-peak is a much better outcome than a deploy that takes the site down while it waits politely.

Index creation needs `CONCURRENTLY` on any table large enough to matter, and `CREATE INDEX CONCURRENTLY` cannot run inside a transaction, so in Alembic it needs its own migration with autocommit:

```python
def upgrade():
    with op.get_context().autocommit_block():
        op.create_index("ix_records_source_id", "records", ["source_id"],
                        postgresql_concurrently=True)
```

It is also worth knowing that a `CONCURRENTLY` build can fail and leave an invalid index behind, which will not be used and will not error. Check for them after:

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

---

## 8. Monitoring, and the alert everyone forgets

Baseline, none of which is controversial: error rate, request latency, connection pool saturation, replication lag if you have replicas, disk headroom, and long-running queries.

```sql
SELECT pid, now() - query_start AS runtime, state, left(query, 120)
FROM pg_stat_activity
WHERE state <> 'idle' AND now() - query_start > interval '1 minute'
ORDER BY runtime DESC;
```

The one that gets left out: **absence alerting**. Every alert listed above fires when something happens. None of them fire when something fails to happen.

A scheduled job that does not run produces no error and no log line. A nightly import that stops arriving looks exactly like a quiet night. A queue consumer that died leaves a queue that grows slowly enough not to page anyone for a week.

So for anything that is supposed to happen on a schedule, record when it last completed and alert on the gap:

```sql
CREATE TABLE job_runs (
    job_name    text        NOT NULL,
    started_at  timestamptz NOT NULL DEFAULT now(),
    finished_at timestamptz,
    status      text        NOT NULL DEFAULT 'running',
    detail      jsonb
);
```

Then one check that asks, for each job, whether the most recent successful finish is older than that job's expected interval. It is a small table and a small query, and in my experience it catches more real incidents than any dashboard.

---

## Order I actually work in

1. Schema diff against a fresh build from migrations. Commit whatever it finds.
2. Restore a backup into a scratch database and run the test suite against it.
3. Stand up the hosted environment, restore into it, point a staging copy of the app at it.
4. Run the suite against staging. Fix the filesystem and timezone surprises that appear here.
5. Backups, retention and PITR configured. Restore tested on the real instance, restore time measured.
6. Monitoring and absence alerting in place.
7. Cut over.

Steps 1 and 2 are the ones under time pressure people want to skip, and they are the two that decide whether the rest goes well.

---

## What this deliberately leaves out

Zero-downtime cutover, multi-region, read replicas and autoscaling. Most systems moving off a single machine do not need any of it yet, and a short maintenance window announced in advance is a perfectly respectable answer. Adding those before the basics are solid mostly adds places for the basics to be wrong.

---

Written by Saunak Surani. Corrections and disagreements welcome.
