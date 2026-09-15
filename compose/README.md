# Docker Compose services

This directory holds the container configuration for every service on the homelab. Each service carries its own subdirectory with its own `compose.yml` and `.env`. 

## Directory layout

```
compose/
├── forgejo
│   ├── .env.example
│   └── compose.yml
├── postgres
│   └── compose.yml
└── README.md
```

An application directory contains:

| File | Committed | Purpose |
|---|---|---|
| `compose.yml` | Yes | The service definitions |
| `.env.example` | Yes | Template structured with safe placeholders |
| `.env` | No | Values populated only on the server |

## Prerequisites

Before starting any application on the server:

- [`site.yml`](../ansible/site.yml) has been applied, so Docker is installed and the `/srv` directories exist with the correct owner UIDs
- The repository is cloned onto the server
- The `docker` group membership has been activated after the first run of `site.yml`
- Each stack has a populated `.env` following the structure of `.env.example`

Applications can also be driven from a workstation using a remote context:

```bash
docker context create homelab --docker host=ssh://homelab
docker context use homelab
```

## Conventions

### Bind-backed named volumes

Persistent state uses a named volume whose backing directory is pinned to `/srv`, rather than a plain bind mount or a plain named volume:

```yaml
volumes:
  forgejo-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /srv/forgejo/data
```

The service definition references an abstract name (`forgejo-data:/data`) and never a host path, so storage stays a deployment concern rather than something baked into the service. The bytes still land on the 75 GB `srv` logical volume, where they are inside the backup boundary and walkable by ordinary tools.

The full reasoning, including what does and does not carry over to Kubernetes, is in [ADR 0001](../docs/adr/0001-bind-backed-named-volumes.md).

### Publishing ports and BIND_IP

Currently, [`daemon.json`](../ansible/files/daemon.json) sets `"ip": "127.0.0.1"` as Docker's default publish address in order to protect accidental service exposure on this device. As a result, externally exposed ports *must* bind to the Netbird address (`BIND_IP`) or to `0.0.0.0`, as shown below:

```yaml
ports:
  - "3000:3000"              # localhost only
  - "${BIND_IP}:3000:3000"   # port open to Netbird members
```

### Environment files

Compose reads `.env` from the same directory as `compose.yml`. The file must be manually configured from the 'example' template before for each service before the services can run:

```bash
cd compose/forgejo
cp .env.example .env
nvim .env
```

## Operating a stack

The following commands must all be run from inside a stack directory:

```bash
cd compose/forgejo

docker compose up -d                  # apply compose.yml
docker compose ps                     # container status
docker compose logs -f forgejo        # stream one service's logs
docker compose pull && docker compose up -d   # update images
docker compose down                   # stop containers, retain data
docker compose config                 # render the file with .env values
```

Of particular note, use `docker compose config` to quickly confirm `BIND_IP` resolved. 

**NOTE:** Avoid `docker compose down -v`. The `-v` flag removes named volumes. [ADR 0001](../docs/adr/0001-bind-backed-named-volumes.md) records the behaviour of volume removal against a bind-backed volume as unverified. It is expected to remove Docker's metadata and leave the host directory intact, but needs testing.

## Networks

Each application declares its own named network. Containers reach each other by service name within a network.

| Network | Members | Notes |
|---|---|---|
| `forgejo` | `forgejo`, `runner`, `dind` | `dind` is isolated to this network only |
| `data` | `postgres` | Future database consumers join here |

Note that database consumers do not need a published port. A future Prefect container joins the `data` network and reaches Postgres at `postgres:5432` by DNS. The published `${BIND_IP}:5432` exists only for tooling run from a workstation.

## Services

### Forgejo and Forgejo Actions

[Forgejo](https://forgejo.org/docs/latest/) and [Forgejo Actions](https://forgejo.org/docs/v15.0/user/actions/reference/) are the current "latest and greatest" in self-hosted cloud Git providers. I don't want to migrate from Github until I know my backups function and the service is reliable outside the Netbird mesh, but setup is painless so it earns a place as an early addition.

| Container | Image | Role |
|---|---|---|
| `forgejo` | `codeberg.org/forgejo/forgejo:16` | The 'forge', UID 1000 |
| `forgejo-runner` | `data.forgejo.org/forgejo/runner:13` | Actions runner, UID 1001 |
| `forgejo-runner-dind` | `docker:dind` | Docker daemon for CI jobs with `privileged: true` |

Published on the mesh at `${BIND_IP}:3000` for HTTP and `${BIND_IP}:2222` for git over SSH. Port 2222 was used because the host's `sshd` owns 22. The service uses the recommended SQLite database so that `forgejo dump` captures the full application. The `dind` sidecar was selected to create a layer of insulation between a potentially hostile Actions run and root daemon access. 

### Postgres

[Postgres](https://www.postgresql.org/) is the production database du jour at the time of writing, and is capable of scaling far beyond the requirements of this project. Because this is a 'dress rehearsal' for a production-ready system, it made sense to use Postgres.
The full design for the server is in [docs/postgres.md](../docs/postgres.md).

Four things block a first start:

1. `compose.yml` mounts `./init-scripts` as the init directory, but the directory present is named `initdb.d`
2. `initdb.d` is empty, and git does not track empty directories — it will not survive a clone
3. There is no `.env.example`. The stack needs `BIND_IP`, `POSTGRES_SUPERUSER_PASSWORD`, `PREFECT_DB_PASSWORD`, and `SQLMESH_DB_PASSWORD`
4. [`site.yml`](../ansible/site.yml) has no `srv_apps` entries for `/srv/postgres` and `/srv/postgres/data`, so the backing directory will not exist at the UID the container runs as

### Prefect

[Prefect](https://docs.prefect.io/v3/get-started) is a modern data engineering orchestration platform. I'll be using the Prefect instance configuration in this repository to build a proof-of-concept for work. Because the setup and configuration is fully portable, it should quickly and easily swap from a homelab context to a work machine. While [Dagster](https://docs.dagster.io/) is another competitive modern option for data engineering with an arguably stronger design, the ease-of-use provided by Prefect and the lower infrastructure burden of self-hosting the service are both attractive. 

Once the move to Kubernetes is complete, I may choose to reevaluate Dagster. Owing to the [recent acquisition by Prefect](https://www.prefect.io/prefect-acquires-dagster), I suspect that that may not be necessary. 

**Status: planned.** Dependent on the completion of the Postgres server.

### Github Actions

Because connection to this homelab is currently only possible via Netbird-enabled TCP, connecting a [Github Actions](https://docs.github.com/en/actions/concepts/runners/self-hosted-runners) runner to the state database will require additional fuss. In order to maintain the 'zero trust' security posture, the Actions runners for repositories which rely on this repo will be hosted on the machine itself. Github will remain the repository host until the Forgejo service can be made sufficiently reliable.

**Status: planned.** No compose file yet.

## Adding a new service

1. Verify the image's UID:

   ```bash
   docker run --rm <image> id <user>
   ```

2. Add `srv_apps` entries to [`ansible/site.yml`](../ansible/site.yml), listing parents before children.

3. Apply the `site.yml` playbook to create the directories:

   ```bash
   cd ansible && ansible-playbook site.yml --diff -K --tags srv
   ```

4. Write `compose/<service>/compose.yml` with a bind-backed named volume and `${BIND_IP}` on every published port.

5. Write `compose/<service>/.env.example`, then on the server run `cp .env.example .env` and fill it in.

6. Start and verify the service functions as intended:

   ```bash
   docker compose up -d && docker compose ps && docker compose logs -f
   ```

7. Update the Services table in the [root README](../README.md#services).

## Troubleshooting

### A container starts but its published port is unreachable

`BIND_IP` is empty or unset, so the port bound to `127.0.0.1`. Confirm with `docker compose config` that the rendered port mapping contains a real address, and check that `.env` sits in the same directory as `compose.yml`.

### A container can't write its own data directory

UID mismatch between the `srv_apps` entry in [`site.yml`](../ansible/site.yml) and the UID the image actually runs as. Verify with `docker run --rm <image> id <user>`, correct the entry, and re-run `site.yml --tags srv`.

### A volume mounts but is empty

Expected. Bind-backed named volumes receive no image pre-population — see [ADR 0001](../docs/adr/0001-bind-backed-named-volumes.md). The backing directory has to be created and populated deliberately.

### permission denied on /var/run/docker.sock

`docker` group membership takes effect at next login, not immediately. Log out and back in.

### Changes to .env aren't taking effect

Compose substitutes variables at container creation, not at start. Run `docker compose up -d` to recreate, rather than `docker compose restart`.

## Kubernetes migration

The conventions here were chosen so that the seam survives even though the mechanism does not. From [ADR 0001](../docs/adr/0001-bind-backed-named-volumes.md): the `driver_opts` block is designed to be *discarded* on scale-out, not extended.

| Compose | Kubernetes |
|---|---|
| service, stateless | Deployment |
| service, stateful | StatefulSet, or an operator |
| `ports:` with `${BIND_IP}` | Service plus Ingress |
| `environment:` | ConfigMap and Secret |
| bind-backed named volume | PersistentVolumeClaim plus StorageClass |
| named network | flat cluster network, restricted by NetworkPolicy |
| `depends_on:` | no equivalent — use init containers and probes |

Two conventions disappear entirely. The `${BIND_IP}` pattern and the `"ip": "127.0.0.1"` daemon default are both replaced by not publishing host ports at all. Postgres would move under an operator such as CloudNativePG rather than running as a bare container; see [docs/postgres.md](../docs/postgres.md) for the data migration path.

The corresponding changes to the host layer are described in [ansible/README.md](../ansible/README.md#kubernetes-migration).
