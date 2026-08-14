# Postgres on channa-homelab

- **Status:** Design — **not deployed.** No Postgres container, backup script, or
  Ansible task described here exists on the server yet.
- **Date:** 2026-08-13
- **Scope:** `channa-homelab` (Ubuntu 26.04 LTS, single host)
- **Related:** [ADR 0001 — Bind-backed named volumes](adr/0001-bind-backed-named-volumes.md)

This document describes a planned shared Postgres instance backing Prefect and
SQLMesh, how it is backed up, and how the containers that use it reach it.
Claims are marked **verified** where they were measured on this host and
**unverified** where they were not. Treat anything unmarked as design intent
rather than observed behaviour.

## Contents

1. [What runs, and why one instance](#what-runs-and-why-one-instance)
2. [Instance, database, schema](#instance-database-schema)
3. [Storage](#storage)
4. [Compose definition](#compose-definition)
5. [First-boot initialization](#first-boot-initialization)
6. [Network configuration](#network-configuration)
7. [Backups](#backups)
8. [Restore](#restore)
9. [Ansible integration](#ansible-integration)
10. [Migration paths](#migration-paths)
11. [Open items and unverified claims](#open-items-and-unverified-claims)

---

## What runs, and why one instance

Two consumers need durable relational state:

| Consumer | What it stores | Loss tolerance |
| --- | --- | --- |
| **Prefect** | Flow runs, task states, schedules, deployments | Low value — losing a day costs observability |
| **SQLMesh** | Snapshots, environments, plans, materialized intervals | Higher value — encodes which physical tables exist in BigQuery |

Both run as **one Postgres instance with two databases**, not two instances and
not one shared database.

**One instance** because a second instance would double the operational surface
— two data directories, two configs, two upgrade cycles, two backup jobs — to
isolate two workloads that together will not saturate a single instance on a
14 GiB host.

**Two databases** rather than one, for three reasons in ascending order of
weight:

1. **Migration coupling.** Both systems run schema migrations on upgrade —
   Prefect via Alembic on server start, SQLMesh via `sqlmesh migrate`. Both take
   locks. Two independent upgrade cadences against one database is an avoidable
   class of failure.
2. **Isolation that the query planner enforces.** With separate databases,
   `REVOKE CONNECT ON DATABASE prefect FROM PUBLIC` means the `sqlmesh` role
   cannot open a connection at all — not "cannot read the tables." Within a
   shared database the strongest available control is schema grants, which is
   more surface and more ways to misconfigure. This matters because **Prefect
   workers execute arbitrary flow code**, making the worker's credentials the
   least-trusted component in the stack.
3. **Independent restore.** SQLMesh state must stay consistent with the physical
   tables in BigQuery; restoring it to a stale point makes SQLMesh's view of the
   warehouse diverge from the warehouse's actual contents. Prefect's run history
   is largely disposable. Sharing a database would force both to the same
   recovery point, so you would be rolling SQLMesh state back to recover Prefect
   history, or preserving broken Prefect state to protect SQLMesh.

The cost of separation is two extra `CREATE` statements in an init script.

## Instance, database, schema

Postgres's own terminology is a recurring source of confusion, so it is worth
stating the hierarchy explicitly:

```
instance      one postmaster process, one PGDATA, one port
│             Postgres docs call this a "cluster" — unrelated to HA
│
├── database: prefect          a connection binds here for its entire lifetime
│   └── schema: public              Prefect's tables
│
├── database: sqlmesh_state
│   └── schema: sqlmesh             SQLMesh's state tables (state_schema)
│
└── database: postgres         default connection target, kept empty
```

**A connection cannot cross a database boundary.** `\c otherdb` in `psql` looks
like navigation but tears down the connection and opens a new one. Crossing
databases requires `postgres_fdw` or `dblink`, which are wrappers around a
second connection. Schemas, by contrast, are cheap namespaces within one
database and are freely joinable.

That asymmetry is the entire basis for choosing separate databases over separate
schemas above.

Two properties of the split surprise people and are worth recording:

- **Roles are instance-wide.** `CREATE ROLE sqlmesh` creates it everywhere at
  once; there is no per-database role. Isolation comes entirely from grants,
  which is why the `REVOKE CONNECT` lines in the init script do real work. By
  default `PUBLIC` holds `CONNECT` on every database.
- **Extensions are per-database.** `CREATE EXTENSION` in one database does
  nothing for another. If an extension is ever needed, it must be created in the
  database that uses it.

## Storage

Follows [ADR 0001](adr/0001-bind-backed-named-volumes.md): a bind-backed named
volume with the bytes on the 75 GB `srv` LV.

| Path | Owner | Mode | Purpose |
| --- | --- | --- | --- |
| `/srv/postgres` | `999:999` | `0750` | Application root |
| `/srv/postgres/data` | `999:999` | `0700` | Volume backing directory |
| `/srv/postgres/data/pgdata` | `999:999` | `0700` | `PGDATA` (created by `initdb`) |
| `/srv/backups/postgres` | `root:root` | `0700` | Dump output |

Three constraints shape this, all of which cause a hard startup failure if
violated:

**Postgres refuses to start if `PGDATA` is more permissive than `0750`.** It
must be `0700` or `0750`, owned by the running user.

**`PGDATA` points one level below the mount point.** Setting
`PGDATA=/var/lib/postgresql/data/pgdata` rather than `.../data` lets `initdb`
work in a directory it created itself. Bind mount roots frequently carry
ownership or `lost+found` entries that `initdb` rejects as "not empty."

**The container runs as uid 999, not 1000.** This differs from every other entry
in `srv_apps`, which use 1000 (Forgejo) or 1001 (the runner). Confirm before
creating the directories rather than trusting this document — the Alpine variant
of the image uses 70:

```bash
docker run --rm postgres:18 id postgres
```

The corresponding `srv_apps` entries in `ansible/site.yml`:

```yaml
- { path: postgres, owner: "999", group: "999", mode: "0750" }
- { path: postgres/data, owner: "999", group: "999", mode: "0700" }
```

Note that `/srv/backups` sits on the **same LV, and the same physical disk**, as
the data it protects. See [Backups](#backups) for what that does and does not
cover.

## Compose definition

`compose/postgres/compose.yml`:

```yaml
services:
  postgres:
    image: postgres:18
    container_name: postgres
    restart: unless-stopped

    # Docker's default 10s grace period can be tight for a database flushing
    # buffers; a SIGKILL mid-checkpoint means crash recovery on next start.
    # The official image already sets STOPSIGNAL SIGINT (fast shutdown rather
    # than smart shutdown, which would wait for clients and blow past any
    # grace period) — this is cheap insurance on top of that.
    stop_grace_period: 1m

    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_SUPERUSER_PASSWORD}

      # One level below the mount point. See Storage.
      PGDATA: /var/lib/postgresql/data/pgdata

      # Consumed by initdb.d/10-databases.sh on first boot only.
      PREFECT_DB_PASSWORD: ${PREFECT_DB_PASSWORD}
      SQLMESH_DB_PASSWORD: ${SQLMESH_DB_PASSWORD}

    volumes:
      - postgres-data:/var/lib/postgresql/data
      # Plain bind mount, read-only: host-authored config, per ADR 0001.
      - ./initdb.d:/docker-entrypoint-initdb.d:ro

    ports:
      # Mesh only. An unqualified "5432:5432" would bind loopback, because
      # daemon.json sets "ip": "127.0.0.1" as the default publish address.
      # Required only for `sqlmesh plan` from the desktop — see Network
      # configuration for why container-to-container traffic does not need it.
      - "${BIND_IP}:5432:5432"

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

    networks:
      - data

volumes:
  postgres-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /srv/postgres/data

networks:
  data:
    name: data
```

`.env.example` records `BIND_IP=100.108.127.155` with the same warning carried
in `compose/forgejo/.env.example`: leaving it unset silently produces a
loopback-only service.

**Pin the major version.** The on-disk format is major-version-specific. If this
tag moves from `postgres:18` to `postgres:19`, the container will refuse to start
with *"database files are incompatible with server"* — there is no automatic
migration. Minor bumps (18.1 → 18.2) are drop-in safe; major bumps are a
deliberate maintenance operation. `unattended-upgrades` does not touch container
images, so the pin in this file is the only thing governing it.

## First-boot initialization

The official image executes anything in `/docker-entrypoint-initdb.d/` when it
initializes an empty data directory. `POSTGRES_DB` creates only a single
database, so the second one needs a script. Use `.sh` rather than `.sql` —
shell scripts can read environment variables, `.sql` files cannot.

`compose/postgres/initdb.d/10-databases.sh`:

```bash
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
    CREATE ROLE prefect LOGIN PASSWORD '${PREFECT_DB_PASSWORD}';
    CREATE DATABASE prefect OWNER prefect;
    REVOKE CONNECT ON DATABASE prefect FROM PUBLIC;
    GRANT  CONNECT ON DATABASE prefect TO prefect;

    CREATE ROLE sqlmesh LOGIN PASSWORD '${SQLMESH_DB_PASSWORD}';
    CREATE DATABASE sqlmesh_state OWNER sqlmesh;
    REVOKE CONNECT ON DATABASE sqlmesh_state FROM PUBLIC;
    GRANT  CONNECT ON DATABASE sqlmesh_state TO sqlmesh;
EOSQL
```

> **This runs exactly once, on an empty data directory.** If
> `/srv/postgres/data/pgdata` is already initialized, the entrypoint skips
> `initdb.d` **silently** — no warning, no error. Getting this wrong means
> either wiping the volume or applying the statements by hand with `psql`.
> Verify after first start that both databases exist before deploying anything
> against them.

## Network configuration

Three distinct paths reach this database, and they are controlled by three
different mechanisms.

```
    desktop                    ┌─────────── channa-homelab ────────────┐
  sqlmesh plan ───NetBird───▶  │ wt0 100.108.127.155:5432              │
                               │   controlled by: NetBird ACL policy   │
                               │                       │               │
                               │   ┌── docker network "data" ──────┐   │
                               │   │           ▼                   │   │
                               │   │  ┌──────────────┐             │   │
                               │   │  │   postgres   │◀── prefect-server
                               │   │  │    :5432     │◀── prefect-worker
                               │   │  └──────┬───────┘             │   │
                               │   │         │      controlled by: │   │
                               │   │         │      network member │   │
                               │   └─────────┼─────────────────────┘   │
                               │             ▼                         │
                               │    /srv/postgres/data                 │
                               │                                       │
                               │   forgejo ── network "forgejo" ──╳    │
                               │   (cannot reach postgres at all)      │
                               └───────────────────────────────────────┘
                                             │
       BigQuery ◀────── HTTPS 443 outbound ──┘
                        controlled by: UFW default outgoing (allow)
```

### Container to container — no published port required

Containers on the same user-defined bridge network resolve each other by service
name and connect directly. **This traffic never touches a host interface**, so
it involves no published port, no UFW rule, and no NetBird policy.

```
host: postgres      # the compose service name
port: 5432
```

The consequence worth designing around: **network membership is itself the
access control.** Forgejo sits on the `forgejo` network and is not a member of
`data`, so it cannot reach Postgres regardless of what credentials it holds —
there is no route. Only services that need the database should join `data`.

Consumers should also gate startup on the healthcheck rather than racing it:

```yaml
    depends_on:
      postgres:
        condition: service_healthy
```

### Desktop over the mesh — NetBird ACL, not UFW

SQLMesh's development workflow (`sqlmesh plan`) reads and writes state, so the
desktop needs to reach the state database directly. That is the only reason the
`${BIND_IP}:5432:5432` publish exists.

**This is governed by NetBird policy, not by UFW.** Verified on this host
2026-08-08: a port with no UFW allow rule was reachable over `wt0` until the
NetBird ACL was corrected. The division of labour is `wlo1` → UFW, `wt0` →
NetBird ACL. Adding a UFW rule for 5432 would therefore accomplish nothing;
the change required is a NetBird policy permitting tcp/5432 from the desktop
peer to this host.

NetBird policies are additive allow-lists evaluated as a union with default-deny,
so **a restrictive policy has no effect while the permissive `Default` policy
still exists.** Confirm the policy is actually live from the desktop rather than
assuming it from the console:

```bash
# should succeed once the policy is in place
bash -c 'cat </dev/null >/dev/tcp/100.108.127.155/5432'
```

If the desktop path is not wanted, drop the `ports:` block entirely. Everything
except local `sqlmesh plan` continues to work, and the attack surface goes to
zero.

### Egress to BigQuery — nothing to configure

SQLMesh is a **client, not a server**. Whatever process runs it opens two
outbound connections: one to the Postgres state database, one to BigQuery over
HTTPS. Nothing connects inward.

```
   ┌──────────────────────┐
   │ process running      │
   │ SQLMesh              │──── state ────▶ postgres:5432
   │ (prefect-worker,     │
   │  or your desktop)    │──── models ───▶ bigquery.googleapis.com:443
   └──────────────────────┘
```

**BigQuery never connects to the state database and requires no access to it.**
It is a passive endpoint that receives SQL; it has no knowledge that SQLMesh
exists. Google's infrastructure does not reach into a host behind T-Mobile
CGNAT, and nothing in SQLMesh's design asks it to.

(The one construct that would invert this is BigQuery's `EXTERNAL_QUERY`
federation, which does dial out to a Postgres — but it requires Cloud SQL and
concerns querying data, not storing state. Not applicable.)

Outbound HTTPS is already permitted: `DEFAULT_OUTPUT_POLICY="ACCEPT"`.

### Which component actually needs the state database

Worth separating, because it is easy to over-provision access:

| Component | Needs `prefect` | Needs `sqlmesh_state` |
| --- | --- | --- |
| Prefect **server/API** | Yes | **No** |
| Prefect **worker** | No¹ | Yes — it executes the SQLMesh code |
| Desktop (`sqlmesh plan`) | No | Yes |
| BigQuery | No | No |

¹ The worker communicates with the Prefect API over HTTP, not by connecting to
Prefect's database directly.

The orchestrator does not need visibility into SQLMesh's state to orchestrate
it — it launches a process and reads an exit code.

## Backups

### Choice of tool

The two families differ on portability and on achievable RPO:

| | **Logical** (`pg_dump`) | **Physical** (`pg_basebackup` + WAL) |
| --- | --- | --- |
| Captures | SQL to recreate objects | Byte-level copy of the data directory |
| Portability | Across major versions and platforms | Locked to same major version and platform |
| Granularity | One database, schema, or table | Whole instance, all or nothing |
| Restore | Replays SQL, **rebuilds every index** | File copy plus WAL replay |
| Best RPO | Time since last dump | **Seconds** (point-in-time recovery) |

**Decision: `pg_dump`, nightly.** The RPO gap is the real difference, and for
this content it is acceptable — losing a day of Prefect run history costs
observability, and a day of SQLMesh state is recoverable by re-planning. In
exchange, logical dumps are version-portable, which is worth more here than
PITR: this is a single-node homelab where major version upgrades and machine
migration are both more likely than a recovery demanding second-level precision.

`pg_dump` is not a scale concession. It is the correct tool when data volume is
small, RPO tolerance is loose, and portability has value — all three hold.

**Revisit when** something lands in this instance where losing a day hurts:
Forgejo migrating off SQLite, or anything holding human-authored content. At
that point the answer is **pgBackRest** (or Barman, or WAL-G) configured against
this instance directly. Note that it does not require Kubernetes — CloudNativePG
is a declarative wrapper around barman-cloud, not a capability Kubernetes itself
provides. When that happens, keep the logical dump as well: physical backups are
version-locked, and a portable escape hatch is exactly what you want when
standing up a different major version.

### Execution model

**Dump from inside the container.** `pg_dump` refuses to run against a server
newer than itself, so a host-installed client becomes a maintenance burden that
breaks at the next major version bump. Executing inside the container guarantees
the versions match permanently. It also connects over the container's local
socket as the `postgres` OS user, so **no password appears anywhere in the
backup path** — the image's generated `pg_hba.conf` trusts local socket
connections.

**Schedule with a systemd timer, not cron.** On a headless box the difference is
operational: `journalctl -u pg-backup` gives logs for free, `systemctl
list-timers` shows last and next run, `Persistent=true` catches up a run missed
to downtime, and `OnFailure=` is the hook for alerting once alerting exists. A
failed cron job is invisible until someone notices the files stopped appearing.

`ansible/files/pg-backup.sh`, installed to `/usr/local/bin/`:

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR=/srv/backups/postgres
CONTAINER=postgres
STAMP=$(date -u +%Y%m%dT%H%M%SZ)
RETAIN_DAYS=30

mkdir -p "$BACKUP_DIR"

# NOTE: never pass -t to docker exec here. A TTY translates newlines and
# silently corrupts a binary -Fc archive. The corruption is not detected
# until restore.
dump_db() {
  local db=$1 out="$BACKUP_DIR/${db}-${STAMP}.dump"
  docker exec -u postgres "$CONTAINER" pg_dump -Fc -d "$db" > "$out"
  # A valid archive can list its table of contents. Turns "a file was
  # written" into "a restorable archive was written".
  docker exec -i -u postgres "$CONTAINER" pg_restore --list < "$out" > /dev/null
}

# Roles and passwords are NOT included in pg_dump. Without this, a restore
# yields intact databases that no user can log into.
docker exec -u postgres "$CONTAINER" pg_dumpall --globals-only \
  > "$BACKUP_DIR/globals-${STAMP}.sql"

dump_db prefect
dump_db sqlmesh_state

find "$BACKUP_DIR" -type f -mtime "+${RETAIN_DAYS}" -delete
```

With `set -euo pipefail`, a failed dump or a failed verification fails the
systemd unit, which is the intended behaviour.

`pg_dump` is an **online** operation — it takes a consistent snapshot inside a
transaction and does not block writes. The container does not need to be
stopped. Each dump is internally consistent per database; the two dumps are not
consistent with *each other*, which is fine because `prefect` and
`sqlmesh_state` share no referential relationship.

`ansible/files/pg-backup.timer`:

```ini
[Unit]
Description=Nightly Postgres logical backup

[Timer]
OnCalendar=daily
RandomizedDelaySec=15m
Persistent=true

[Install]
WantedBy=timers.target
```

### What this covers, and what it does not

A dump written to `/srv/backups` on this host protects against `DROP TABLE`, a
bad migration, and application-level corruption — the logical mistakes, which
are the most frequent cause of restores.

It does **not** protect against drive failure, filesystem corruption, hardware
death, theft, fire, or a compromise where an attacker with root deletes the
backups alongside the data.

**This host has one physical disk.** The single 476.9 GB NVMe carries the root,
`docker`, and `srv` LVs as logical volumes on one PV, so they share its fate
entirely. Placing backups on a separate LV would buy nothing against drive
failure. RAID would not help either — it mirrors a `DROP TABLE` in microseconds;
RAID addresses uptime, backups address mistakes.

Current status against 3-2-1 (three copies, two media, one offsite):

| Copy | Location | Status |
| --- | --- | --- |
| 1 | Live database, `/srv/postgres` | Planned |
| 2 | Dumps, `/srv/backups` — same disk | Planned |
| 3 | Offsite | **Missing** |

The offsite leg is a pre-existing open item on this project, now covering
Postgres as well as `/srv`. Two candidates that fit the constraint of no
recurring cost:

- **Pull to the desktop over NetBird**, using hardware already owned. Pull
  rather than push: if the server is compromised, a push credential lets the
  attacker delete the backups, whereas a pull means the server never holds write
  access to the backup store.
- **Object storage** — Cloudflare R2's free tier (10 GB, no egress fees) or
  Backblaze B2 (~$6/TB/month). At this data volume that is plausibly $0/month,
  a different category from a VPS rather than a smaller one.

`restic` suits both legs — encrypted, deduplicated, incremental, targeting a
local path, SFTP, or S3-compatible storage with the same commands. One tuning
note: dump with `-Z0` (uncompressed) when feeding restic. `pg_dump -Fc`
compresses by default, and compressed input destroys deduplication, so thirty
daily dumps cost thirty full copies instead of one plus deltas.

**An untested backup is a hypothesis.** Restore into a throwaway container
before relying on any of this.

## Restore

Order matters: roles must exist before objects can be owned by them.

```bash
# 1. Roles and passwords
docker exec -i -u postgres postgres psql < globals-<stamp>.sql

# 2. Each database. -C creates the database, which requires connecting to a
#    different one (the maintenance database `postgres`).
docker exec -i -u postgres postgres \
  pg_restore -C -d postgres < prefect-<stamp>.dump

docker exec -i -u postgres postgres \
  pg_restore -C -d postgres < sqlmesh_state-<stamp>.dump
```

The container is named `postgres`, the OS user is `postgres`, and the
maintenance database is `postgres`. The repetition above is correct, if
unfortunate: `docker exec -u <os-user> <container> pg_restore -d <database>`.

Restore into the **same or a newer** major version, never an older one. If any
extensions are in use, install them in the target image before restoring — the
dump contains `CREATE EXTENSION`, but the shared libraries must already exist.

Use `--no-owner --no-acl` when the target instance's roles do not match the
source's.

## Ansible integration

Four files, following the existing `tasks/` + tag structure in `site.yml`:

```
ansible/
  files/
    pg-backup.sh              installed to /usr/local/bin/, 0755
    pg-backup.service         Type=oneshot, After=docker.service
    pg-backup.timer           OnCalendar=daily
  tasks/
    backup.yml                imported by site.yml, tags: [backup]
```

`tasks/backup.yml`:

```yaml
- name: Backup output directory exists
  ansible.builtin.file:
    path: /srv/backups/postgres
    state: directory
    owner: root
    group: root
    mode: "0700"

- name: Backup script installed
  ansible.builtin.copy:
    src: pg-backup.sh
    dest: /usr/local/bin/pg-backup.sh
    owner: root
    group: root
    mode: "0755"

- name: Backup units installed
  ansible.builtin.copy:
    src: "{{ item }}"
    dest: "/etc/systemd/system/{{ item }}"
    owner: root
    group: root
    mode: "0644"
  loop:
    - pg-backup.service
    - pg-backup.timer

- name: Backup timer enabled
  ansible.builtin.systemd_service:
    name: pg-backup.timer
    enabled: true
    state: started
    daemon_reload: true
```

Plus the two `srv_apps` entries from [Storage](#storage) added to `site.yml`.

This preserves the property that `ansible-playbook site.yml --check --diff -K`
is a drift detector: once applied, every task above reports `ok`, and any
`changed` means the box was hand-edited or the playbook is wrong.

## Migration paths

### To another machine

The dump is the small part; the repository is the migration plan.

```
1. ansible-playbook bootstrap.yml --tags bootstrap    # LVM, filesystems, fstab
2. ansible-playbook site.yml -K                       # docker, /srv, ufw, backups
3. git clone <repo> && docker compose up -d postgres
4. restore globals, then each database                # see Restore
5. docker compose up -d                               # everything else
```

Downtime is proportional to restore time, dominated by index rebuilds. At this
data volume — Prefect metadata and SQLMesh state, likely well under a gigabyte —
that is minutes.

**`pg_dump` covers Postgres and nothing else.** Forgejo's SQLite database and
git repositories live in `/srv/forgejo/data` and need their own mechanism;
`forgejo dump` produces a single archive with the database, repositories, LFS,
and attachments together. "Backups are handled" must not come to mean "Postgres
is handled."

### To Kubernetes

Logical dumps are the neutral interchange format that makes this work, and are
the *only* option that does — a physical backup cannot be restored into a
different major version, and managed platforms generally do not expose the
filesystem needed to use one.

On Kubernetes this would not be a bare container. **CloudNativePG** is the
current consensus operator; it supports bootstrapping from an existing Postgres
via `bootstrap.initdb.import`, with a `monolith` mode that brings multiple
databases plus roles across in one operation — matching this layout directly.
Verify against current CNPG documentation before relying on the specifics.

What transfers: services → Deployments/StatefulSets, published ports →
Services, environment → ConfigMaps and Secrets, and the discipline of keeping
state externalized and configuration declarative. What does not: the
bind-backed named volume convention, which has no Kubernetes equivalent and
becomes PVCs backed by a storage class. The *principle* survives; the mechanism
is replaced, as anticipated in ADR 0001.

## Open items and unverified claims

| Item | Status |
| --- | --- |
| Container uid — `999` assumed for the Debian variant | **Unverified.** Check with `docker run --rm postgres:18 id postgres` before creating `/srv/postgres`. |
| Postgres 18 as the current major | **Verified** 2026-08-13 against Docker Hub — latest tags are `18`/`18.6`; no 19 series exists yet. |
| Prefect's connection-URL setting name | **Unverified.** Newer Prefect 3 releases renamed `PREFECT_API_DATABASE_CONNECTION_URL` to `PREFECT_SERVER_DATABASE_CONNECTION_URL`. Check the pinned version. |
| Mesh-published ports are NetBird-controlled, not UFW-filtered | **Verified** 2026-08-08, on port 8080 rather than 5432. |
| Docker does not bypass UFW on this host | **Verified** 2026-08-08. Re-test after major Docker upgrades — `unattended-upgrades` delivers them silently. |
| Offsite backup copy | **Missing.** Pre-existing open item. |
| Restore has never been exercised | **Outstanding.** Required before this is a backup rather than a file-writing job. |
| `docker volume prune` behaviour against bind-backed volumes | **Unverified** — carried over from ADR 0001. |

## References

- [ADR 0001 — Bind-backed named volumes](adr/0001-bind-backed-named-volumes.md)
- PostgreSQL documentation — *Backup and Restore*, *Managing Databases*
- Docker documentation — *Networking overview* (user-defined bridge networks)
- `docker-library/postgres` — entrypoint behaviour, `initdb.d`, `PGDATA`
- CloudNativePG documentation — *Bootstrap: import*
