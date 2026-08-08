# ADR 0001: Bind-backed named volumes for container persistent state

- **Status:** Accepted
- **Date:** 2026-08-08
- **Scope:** `channa-homelab` (Ubuntu 26.04 LTS, single host)

## Context

`channa-homelab` is a single-host Docker environment serving applications to a
desktop and a MacBook Air. Storage is one 476.9 GB NVMe drive (WDC PC SN520),
with no redundancy. The drive is partitioned as EFI (1 GB) → `/boot` (2 GB) →
a single LVM physical volume (473.9 GB).

The volume group `ubuntu-vg` is carved as:

| LV          | Size    | Filesystem       | Mountpoint        |
| ----------- | ------- | ---------------- | ----------------- |
| `ubuntu-lv` | 100 GB  | ext4             | `/`               |
| `docker`    | 50 GB   | ext4, label `docker` | `/var/lib/docker` |
| `srv`       | 75 GB   | ext4, label `srv`    | `/srv`            |
| *(unallocated)* | ~249 GB | —            | —                 |

Two properties of this layout drive the decision:

1. **`/var/lib/docker` and `/srv` are separate volumes** so that a runaway
   container cannot fill `/` — a full root filesystem takes down a headless
   machine entirely (journald cannot write, sshd may fail to fork).
2. **`/srv` is intended as the backup boundary.** "Back up `/srv`" should
   capture everything irreplaceable and exclude the ~50 GB of regenerable
   Docker image layers.

Property 2 only holds if persistent application state actually lands on `/srv`.
Docker offers two ways to give a container persistent storage, and the choice
determines whether it does:

- **Bind mount** — an explicit host path is named in the service definition.
- **Named volume** — Docker manages the storage under a name. With the default
  `local` driver, the data lives at `/var/lib/docker/volumes/<name>/_data`.

Plain named volumes would put all application state on the 50 GB `docker` LV,
mixed in with image layers and outside the `/srv` backup boundary, leaving the
75 GB `srv` LV unused. So this decision had to be made before the first
container was deployed.

Widely repeated advice says to prefer named volumes "in production," which
appeared to conflict with the layout above. Investigating that advice is what
produced this ADR.

## Decision

**Use bind-backed named volumes for application persistent state**, with the
backing directory under `/srv`:

```yaml
services:
  gitea:
    volumes:
      - gitea-data:/data

volumes:
  gitea-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /srv/gitea/data
```

**Use plain bind mounts for configuration files** authored and edited on the
host (read-only where possible):

```yaml
    volumes:
      - /srv/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
```

Supporting conventions:

- All persistent state lives under `/srv`, one subdirectory per application.
- `/srv` stays `root:root 0755` at the top level; per-application
  subdirectories are owned by the UID the container runs as.
- Compose files are version-controlled in this repository, not hand-edited on
  the server.
- Databases are backed up with database-native tooling (`pg_dump`,
  `pgBackRest`), not by copying files.

## Rationale

### Named vs. bind is not a storage-mechanism difference on Linux

A `local`-driver named volume *is* a directory that Docker bind-mounts into the
container. Same kernel call, same filesystem, identical performance. There is
no overlay and no I/O penalty in either direction. The choice is about **who
owns the lifecycle and who knows the path**, not about the storage path itself.

### Most of the "prefer volumes" advice does not apply here

Checking Docker's stated reasons against this environment:

| Reason given | Applies? |
| --- | --- |
| Higher performance than bind mounts | **No** — that gap is the Docker Desktop VM boundary on macOS/Windows. Native Linux has none. |
| Portable across Linux and Windows containers | No |
| Managed by volume drivers (NFS, cloud, encrypted) | Not today — but this is the one that matters, see below |
| Safely shared among multiple containers | Marginal |
| Don't depend on host directory structure | **Yes** — the real argument |
| Pre-populated from image content on first mount | **Yes**, and it is a genuine functional difference |

Much of the online advice is written from inside orchestrated, multi-host, or
Docker Desktop contexts without stating so.

### What actually transfers to production is the *seam*, not the mechanism

The property worth preserving is that **service definitions do not contain
host paths**:

```yaml
- /srv/gitea/data:/data   # does not transfer — assumes this host's layout
- gitea-data:/data        # transfers — storage becomes a deployment concern
```

A separate declaration binds that name to concrete storage, and changing the
backend means changing only that second declaration. This seam exists
identically in a Compose file and a Kubernetes manifest — the `driver_opts`
block sits exactly where a `StorageClass` would. Bind-backed named volumes
preserve the seam while pinning the backend to storage we control.

### Backup ergonomics are the strongest practical argument

Backing up an opaque named volume conventionally means starting a throwaway
container to `tar` the contents out. With a bind-backed volume the bytes are
ordinary files on `/srv`, walkable by `restic` or `borg` and visible to LVM
snapshots. Backup discipline is the thing homelabs most often fail at, and any
friction there compounds.

### `/srv` is the standards-appropriate location

FHS 3.0 §3.16 defines `/srv` as "site-specific data which is served by this
system." The web-hosting slant in most online references is an artifact of the
examples the spec offers (`ftp`, `rsync`, `www`, `cvs` — three of four are not
websites), not of its definition. The spec explicitly states that the
methodology for naming subdirectories of `/srv` is unspecified, so
`/srv/<app>/` is not a deviation from a prescribed scheme.

Practically, no Debian/Ubuntu package manages content under `/srv`, and the FHS
directs distributions not to remove locally placed files there. This contrasts
with `/var/lib`, where packages legitimately create and remove subdirectories —
a real consideration across release upgrades.

## Consequences

### Positive

- Persistent state lands on the 75 GB `srv` LV, so the 50/75 split works as
  designed and `/srv` is a true backup boundary.
- Service definitions are host-path-free and portable.
- Data is walkable by standard tools; LVM snapshots of `srv` capture it.
- Docker image churn cannot consume the space reserved for application data,
  and neither can consume `/`.

### Negative / accepted costs

- **No image pre-population.** A plain named volume, when first mounted empty,
  receives whatever the image ships at that path, with correct ownership. A
  bind-backed volume does not. Backing directories must be created by hand,
  with the right UID, before first start.
- **`driver_opts` is the most host-pinned configuration in Docker's
  vocabulary.** On another node, `device: /srv/gitea/data` is meaningless — or
  worse, exists and is empty, and the container starts cleanly against the
  wrong data. This block is designed to be *discarded*, not extended, on
  scale-out.
- Slightly more verbose than either alternative.

### Unverified

- The behaviour of `docker volume prune` against bind-backed volumes is
  expected to remove only Docker's volume metadata and leave the host directory
  intact. **This has not been tested.** Confirm on a throwaway volume before
  relying on it.

## Alternatives considered

**Plain bind mounts** (`- /srv/gitea/data:/data`). Simplest, and identical in
mechanism. Rejected because host paths in service definitions are a coupling
defect that has to be undone later; the indirection costs nothing to adopt now.
Still used for host-authored config files, where the explicit path is the point.

**Plain named volumes** (`- gitea-data:/data` with no `driver_opts`). Rejected
because state would land in `/var/lib/docker/volumes` on the 50 GB LV, mixed
with image layers and outside the `/srv` backup boundary, leaving the 75 GB LV
unused. Choosing this would require re-sizing: shrink `srv` and `lvextend -r`
the `docker` LV. Note that shrinking ext4 requires unmounting it, which is
trivial while the volumes are empty and disruptive once applications depend on
them.

**`/opt/<stack>`, `/var/lib/<app>`, or `/data` as the root.** All are in wide
use; there is no universal consensus in the container world. `/srv` was chosen
as standards-grounded and unowned by any package. Internal consistency matters
more than the specific choice.

## Notes

### On describing this as a production architecture

The interface matches production; the implementation behind it is the
single-host degenerate case. Stated precisely:

- **What transfers:** the seam — an abstract name in the service definition,
  bound to concrete storage by a separate declaration.
- **What does not transfer:** dynamic provisioning, attach/detach,
  rescheduling, and network-backed durability. On a single host none of these
  have anything to bind to.

Two distinct production strategies should not be conflated. An *externalized
managed service* (RDS, Cloud SQL) is reached over the network via a connection
string and involves **no volume at all** — that is the entire point, and it is
why containers there can be destroyed freely. An *in-cluster stateful workload*
does use volumes: a PVC references a StorageClass, a CSI driver provisions a
network-attached device, and it attaches to whichever node the pod lands on.
These are alternatives, not two halves of one mechanism.

The honest framing is therefore: the volume block is the seam where a CSI
driver or a managed-database connection string would substitute in.

### This decision provides no redundancy

A bind-backed named volume delivers the indirection seam and nothing else: one
copy, one drive, one machine. "High availability" and "backups" are four
separate layers that fail differently:

| Layer | Protects against | Mechanism |
| --- | --- | --- |
| Backups | Deletion, corruption, ransomware, operator error | 3-2-1, retention, tested restores |
| Drive redundancy | One disk dying | RAID1 / ZFS mirror / LVM RAID |
| Host redundancy | One machine dying | Replicated or shared storage + failover |
| Application replication | Same, for databases | Postgres streaming replication |

**HA is not a backup.** Replication propagates a `DROP TABLE` or an encryption
payload to the replica in milliseconds. HA protects uptime; backups protect
data. Neither substitutes for the other, and backups come first.

Real HA for stateful services is a database problem, not a volume problem, and
is not reachable from this decision by incremental change — it requires a
different storage substrate (Ceph via Rook, or Longhorn on k3s) adopted
wholesale. Deferred deliberately.

## References

- FHS 3.0 §3.16 — `/srv : Data for services provided by this system`
- Docker documentation — *Volumes* / *Bind mounts*
- Filesystem UUIDs at time of writing: `docker`
  `6489c2ea-65cf-4bf8-93b6-6ea978b4051e`, `srv`
  `c38cad7a-37fa-443a-8de5-70186bfb868e`
