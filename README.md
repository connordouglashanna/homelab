# Homelab! 

This repository houses the IaC configuration files and planning for my homelab, which is currently managed using [Docker Compose](https://docs.docker.com/compose/) and [Ansible](https://docs.ansible.com/#get_started). Ansible handles the machine setup, including storage configuration, firewall configuration, and the logical volumes for services managed by Docker Compose. Docker Compose manages the container configuration. This project is not mature, and will require considerable work to approach its final configuration.

## Contents

| Section | Contents | 
|---|---|
| [Overview](#overview) | Architecture, repository layout |
| [Hardware](#hardware) | Current machine, available hardware |
| [Security posture](#security-posture) | Threat model, access paths, and hardening measures |
| [Getting started](#getting-started) | Prerequisites, the bare-metal build, and operating guide |
| [Services](#services) | A status table of available services |
| [Configuration & secrets](#configuration--secrets) | .env conventions and .gitignore |
| [Backup & restore](#backup--restore) | Backup system design overview and the restoration procedure |
| [Design decisions](#design-decisions) | Overview of the ADR indexes and pointers to additional docs |
| [Roadmap](#roadmap) | Planned future additions, architecture changes, and so on |

## Overview

### Architecture

The files in this repository are used to configure and maintain the settings on the single host machine. The relationships between these configuration files and the configuration of the host is shown in the diagram below:

```mermaid
flowchart TB
    subgraph repo["Repository-owned configuration"]
        B["bootstrap.yml"]
        S["site.yml"]
        C["compose/*/compose.yml"]
    end

    subgraph host["channa-homelab - Ubuntu 26.04 on a single host"]
        subgraph storage["LVM on primary 512GB NVMe"]
            ubuntu["ubuntu-lv 100GB - <code>/</code>"]
            docker["docker 50GB - <code>/var/lib/docker</code>"]
            srv["srv 75GB - <code>/srv</code>"]
            empty["~249GB unallocated"] 
        end
        BASE["Base system<br/>hostname, UTC, unattended-upgrades"]
        FW["UFW<br/>SSH on wt0 only"]
        ENG["Docker engine<br/>daemon.json, published to 127.0.0.1 with live-restore"]
        DIRS["/srv per-application directories<br/>owned by container UIDs"]
        SVC["Containers<br/>forgejo · runner · dind · postgres (planned)"]
    end

    B -->|"configures"| storage
    S --> BASE
    S --> FW
    S --> ENG
    S --> DIRS
    C --> SVC
    ENG --> SVC
    SVC -->|"bind-backed named volumes"| DIRS
```

Docker Compose was chosen for the ease of setup. The desired end state (K8s with GitOps configuration) will differ significantly from the relationships shown above.

### Repository Layout

The directory layout for this repository is shown in the diagram below:

```
.
├── ansible
│   ├── files
│   │   └── daemon.json
│   ├── tasks
│   │   ├── base.yml
│   │   ├── docker.yml
│   │   ├── firewall.yml
│   │   └── srv-layout.yml
│   ├── ansible.cfg
│   ├── bootstrap.yml
│   ├── inventory.yml
│   ├── README.md
│   ├── requirements.yml
│   └── site.yml
├── compose
│   ├── forgejo
│   │   ├── .env.example
│   │   └── compose.yml
│   ├── postgres
│   │   └── compose.yml
│   └── README.md
├── docs
│   ├── adr
│   │   └── 0001-bind-backed-named-volumes.md
│   ├── postgres.md
│   └── services.md
├── src
│   └── homelab
│       └── __init__.py
├── .python-version
├── pyproject.toml
├── README.md
└── uv.lock
```

The project is currently split into three primary folders: 

- [ansible](ansible/), which contains the Ansible playbooks and configuration for machine setup
- [compose](compose/), which contains the Docker image configurations for the repository
- [docs](docs/), which contains [ADR files](docs/adr/) and general documentation

The remaining top-level files are the uv Python project that pins this repository's tooling. [`pyproject.toml`](pyproject.toml) declares Ansible as a project dependency, `uv.lock` pins the resolved versions, and `.python-version` fixes the interpreter. `src/homelab/` is an empty uv scaffold.

## Hardware

The homelab currently consists of a single Lenovo Thinkcentre 920Q thin client with 16GB of DDR4 RAM and a 512GB NVME SSD. The processor is an Intel i7-8700T, a low-wattage processor with 6 cores and 12 threads. In the future I'll likely add more Lenovo thin clients for symmetry, low power consumption, and because they're the cheapest hardware on Ebay. This isn't strictly needed for the project to function or add more services, but does enable fun K8s and hardware rack configuration options. 

Other hardware that I may incorporate includes:

### An old Dell XPS 13

A still-operational but dessicated Dell XPS 13 from 2017. The machine has 8GB of DDR4 RAM and a 256GB drive, but lacks an ethernet port, and is currently missing a TPM Device likely due to an expired CMOS battery. The CMOS battery and the primary device battery would both need to be replaced before the machine could be reused, but the backup power supply and convenient screen and keyboard could make it useful as an orchestration host.

### A pair of Odroid C2 units

The Odroid C2 units have 2GB of DDR3 memory and no native storage. They have never been used, but would be ideal for hosting small network services (e.g. PiHole) and experimenting with mixed architecture configurations in a K8s cluster. Because their use may require me to furnish either additional MicroSD cards or an external NVME SSD, these devices may not be cost effective to repurpose.

## Security posture

The full security posture on this setup is still being established. Remote server access is currently restricted to devices enrolled in a [Netbird](https://docs.netbird.io/get-started) mesh. Netbird enables a fully zero-trust security configuration, with no remote access to the machine except through the mesh network. Outbound connections are still permitted. 

Because the mesh removes public network exposure, this configuration is able to accept the following risks: 

- `connor` functionally has passwordless root in order to enable Docker Compose functionality
- The `dind` sidecar backing Forgejo Actions runs with `privileged: true` and an escape is unlikely but possible
- UFW does not filter mesh traffic, and `wt0` traffic to this machine must be filtered by the Netbird mesh configuration
- Netbird's cloud control plane is a third-party dependency and represents the only remote path into the machine
- No secrets managers or backups have been configured

A diagram of the current access procedure for the project is included below:

```mermaid
flowchart LR
    DESK["Desktop / MacBook<br/>enrolled NetBird peers"]
    LAN["LAN 192.168.12.0/24<br/>shared with IoT devices"]
    NET["Internet"]

    subgraph host["channa-homelab"]
        WT["wt0 - NetBird interface"]
        WLO["wlo1 - LAN interface"]
        SSH["sshd :22"]
        PORTS["Container ports<br/>published on BIND_IP"]
    end

    DESK -->|"governed by NetBird ACL"| WT
    WT --> SSH
    WT --> PORTS
    LAN -.->|"blocked, UFW drops 22"| WLO
    NET -.->|"blocked, CGNAT, no inbound"| WLO
    WLO -->|"outbound permitted"| NET
```

### Planned security changes

The project currently resides on a CGNAT network, and will require DDNS over IPv6 and/or a Cloudflare tunnel in order to expose applications to the open internet. Because these requirements are also compliant with security and networking best practices, they do not represent a substantial impediment to the progress of the project. 

Once the services are functional and the container-to-container network is established, I plan on exploring the following changes:

- A [Cloudflare tunnel](https://developers.cloudflare.com/tunnel/) for public domain access, since the network this project is linked to uses CGNAT
- Adding [fail2ban](https://github.com/fail2ban/fail2ban) to prevent brute-force port scanner attacks
- Geoblocking for anyone outside Texas
- [Traefik](https://traefik.io/traefik), [Nginx](https://nginx.org/), or [Caddy](https://caddyserver.com/) for HTTPS hosting
- Direct scope-limited TCP connection for *some* Postgres backends and the homelab itself

## Getting started

### Prerequisites

The workstation applying this configuration needs:

- [uv](https://docs.astral.sh/uv/), then `uv sync` to install Ansible from [`pyproject.toml`](pyproject.toml) into `.venv`. Activate it with `source .venv/bin/activate`, or prefix each command below with `uv run`
- Enrollment in the NetBird mesh, which is the only remote path to the server
- A private SSH key at `~/.ssh/id_ed25519_homelab_ncased` matching the path in [`inventory.yml`](ansible/inventory.yml)

### Building from bare metal

To start a clean setup, perform the following steps: 

1. Boot [Ubuntu Server 26.04](https://ubuntu.com/download/server) or the latest LTS version. Choose the guided LVM partitioning option and create the user `connor`
2. Add your workstation's public key to `~/.ssh/authorized_keys` on the server
3. Install and enroll Netbird on the server
4. Verify connectivity using `cd ansible && ansible all -m ping`
5. Build the storage layer using `ansible-playbook bootstrap.yml --tags bootstrap -K`
6. Apply the host configuration using `ansible-playbook site.yml --diff -K`
7. Log out and log back in to apply Docker group membership
8. Clone the repository onto the server for the Docker Compose files
9. In each `compose/` subdirectory, run `cp .env.example .env` and fill in the values
10. In each `compose/` subdirectory, run `docker compose up -d`

Also note the following details: 

- [`bootstrap.yml`](ansible/bootstrap.yml) creates logical volumes inside the volume group `ubuntu-vg` but never creates the group itself, because it assumes the installer has already done so
- The Docker daemon publishes to `127.0.0.1` by default, so an unset `BIND_IP` produces containers that won't be accessible from the public internet

### Operating guide

See [ansible/README.md](ansible/README.md) and [compose/README.md](compose/README.md) for details.


## Services

| Service | Status | Docs |
|---|---|---|
| Forgejo and Forgejo Actions | Running | [compose/README.md](compose/README.md#forgejo-and-forgejo-actions) |
| Postgres | Designed, not deployed | [docs/postgres.md](docs/postgres.md) |
| Prefect | Planned | [compose/README.md](compose/README.md#prefect) |

## Configuration & secrets

TODO!
intro — two layers, what is committed vs not

| Service | Variable | Secret |
|---|---|---|
| Forgejo | `FORGEJO_DOMAIN` | The domain used in clone URLs. |
| Forgejo | `BIND_IP` | Currently the Netbird mesh address of the host device. |
| Postgres | `BIND_IP` | Currently the Netbird mesh address of the host device. |
| Postgres | `POSTGRES_SUPERUSER_PASSWORD` | Superuser. |
| Postgres | `{service}_DB_PASSWORD` | Per-service, consumed on first-boot init. |

### Environment files

the .env / .env.example convention + per-stack table + BIND_IP callout

### Secrets outside .env

runner token, SSH key

### Current limitations

no secrets manager, missing postgres .env.example, vault/SOPS unused

## Backup & restore

Backups will be via Restic using remote storage hosted on Backblaze or AWS S3. Ideally these backups will be automated by a Github Actions runner associated with this repository, which will (end state) be run by the Github Actions server hosted on the homelab system.

[docs/postgres.md](docs/postgres.md) describes a separate database-specific backup procedure using `pg_dump` driven by a systemd timer, writing to `/srv/backups/postgres`. This format ensures that data would survive a major-version or platform change.

### Disaster recovery

First, perform a fresh installation of Ubuntu Server. 

Then, to apply the Ansible playbooks located in this repository, use the following commands from the [`ansible/`](ansible/) folder: 

```bash
cd ansible

# Can we reach the machine? 
ansible all -m ping

# Storage rebuild. Requires the explicit tag.
ansible-playbook bootstrap.yml --tags bootstrap -K

# Apply the diff in configuration specs 
ansible-playbook site.yml --diff -K
```

This will restore the machine to the network, drive, and other settings specified by this repository. Note that running the bootstrap playbook can rewrite `/etc/fstab`, and could cause next-boot failure for services. For more information, see the [Ansible configuration README.](/ansible/README.md)

Once the Ansible configuration has been established, the Docker daemon will start and the machine will boot the appropriate services. Because storage currently does not contain any critical information, backups haven't been configured. Once backups are configured, the backup restoration procedure will be detailed here.

## Design decisions

ADR files are grouped in [docs/adr/](docs/adr/). Assorted documentation for future services, previous decisions, and design work in progress is stored in [docs/](docs/) directly.

| Document | Status | Summary |
|---|---|---|
| [ADR 0001 — Bind-backed named volumes](docs/adr/0001-bind-backed-named-volumes.md) | Implemented | Why persistent state uses named volumes pinned to `/srv` rather than plain bind mounts or plain named volumes, and what that does and does not transfer to Kubernetes |
| [postgres.md](docs/postgres.md) | Design, not deployed | The planned shared Postgres instance, its two-database layout, backup and restore procedure, and container networking |
| [services.md](docs/services.md) | Planning | Candidate services that have not yet been designed or deployed |

## Roadmap

### Kubernetes

A future migration to [Kubernetes](https://kubernetes.io/docs/home/) is planned but is currently not implemented. Migration details are partially addressed in [ansible/README.md](ansible/README.md#kubernetes-migration) and [compose/README.md](compose/README.md#kubernetes-migration).

### Self-Hosted Spark

[Apache Spark](https://spark.apache.org/docs/latest/running-on-kubernetes.html) is a framework for distributed compute on large-scale data workloads. While the overwhelming majority of companies probably lack the volumes required to make Spark the obvious choice over a single-node tool like [DuckDB](https://duckdb.org/docs/current/) or [Polars](https://docs.pola.rs/), it remains the 'professional standard' for numerous data engineering workloads. Because I think it would be a pretty cool learning project to self-host it on my own infrastructure and run test jobs using publically available data, I could build a 'proof of concept' for fun.
