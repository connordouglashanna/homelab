# Ansible configuration

This project comes packaged with two [Ansible](https://docs.ansible.com/) playbooks, each of which runs a separate set of tasks in the process of configuring the homelab server. The project uses [`bootstrap.yml`](bootstrap.yml) to initialize a bare metal machine with a stock Ubuntu configuration, and [`site.yml`](site.yml) as an idempotent initialization script for all non-destructive system configuration.

## Directory layout

The layout for the files in this directory is shown below. Comments have been added to describe files not explicitly mentioned in one of the detail sections below.

```
ansible/
├── ansible.cfg          # Config entry point for Ansible
├── inventory.yml        # The host(s) information
├── .env.example         # Template for .env
├── .env                 # Not committed. Names the local SSH key
├── requirements.yml     # Ansible collection dependencies for the playbooks
├── site.yml
├── bootstrap.yml
├── files/
│   └── daemon.json      # Docker daemon config used by tasks/docker.yml
└── tasks/
    ├── base.yml
    ├── docker.yml
    ├── firewall.yml
    └── srv-layout.yml
```

## Setup

To install Ansible on your device, use the following Bash command: 

```bash
uv tool install ansible
```

The device applying these playbooks will also need to:

- Set up SSH connection to the server and enroll in the Netbird network
- SSH configuration will require either access to an existing Netbird node or physical access to the server
- Hold a private key under `~/.ssh/`, named in `.env` as described below
- The Ansible playbooks in this repository must be run from the Ansible subdirectory in order to ensure that [`ansible.cfg`](ansible.cfg) is available

### Naming the SSH key

[`inventory.yml`](inventory.yml) does not hardcode a key name, so the same inventory works from any workstation regardless of what that machine calls its key. The name is read from the `HOMELAB_SSH_KEY` environment variable and resolved under `~/.ssh/`:

```bash
cp .env.example .env
nvim .env          # set HOMELAB_SSH_KEY to your key's filename
```

Note that this `.env` is for local use only. Only the `.env` files in [`compose/`](../compose/README.md#environment-files) are populated on the server.

Ansible does not read `.env` on its own. Export it into the environment first, from this directory:

```bash
set -a; source .env; set +a
```

Once Ansible is installed and configured, run:

```bash
ansible all -m ping
```

This will ensure that the host device is able to connect to the server.

## Playbooks 

### Bootstrap

The [bootstrap playbook](bootstrap.yml) owns setup for the logical volumes and filesystem on the machine. By default, Ubuntu server leaves the overwhelming majority of the drive unallocated. In order to configure our services, we therefore need to manually allocate storage to flexible logical volumes underneath ext4 filesystem storage. At runtime, each step runs for each defined logical volume before moving to the next step. Currently, there are two: 

```yaml
  vars:
    vg: ubuntu-vg
    lvs:
      - { name: docker, size: 50g, label: docker, mount: /var/lib/docker }
      - { name: srv, size: 75g, label: srv, mount: /srv }
```

The `docker` LV holds all information related to the Docker images being hosted on the machine, and `srv` holds persistent service state. The design of the storage system is intended to allow flexible expansion of storage volumes; as `docker` fills with images, the filesystem and logical volume underneath it can be freely extended to consume more space. To extend the logical volumes and increase available storage, edit the size specified in `lvs:` and rerun the playbook.

Note that this playbook does **not** create the volume group for the machine. All volumes and filesystems are created in the default Ubuntu volume group `ubuntu-vg`. 

To run this playbook, use the following command: 

```bash
# applying the playbook
ansible-playbook bootstrap.yml --tags bootstrap -K
```

Note that the tags argument is necessary because the `bootstrap.yml` playbook is deliberately tagged `never` in order to avoid accidentally overwriting a drive configuration. The `--tags bootstrap` argument here is an opt-in override. `-K` is provided in order to run the playbook tasks with root permissions.

### Site

The [site playbook](site.yml) executes the configuration details related to the core managed services on the server. The `site.yml` playbook currently owns the service definitions themselves, while implementation is defined within each of the task files in `tasks/` that it imports. Separated into domain areas for each 'group' of operations performed in the setup:

- [`base.yml`](tasks/base.yml) defines the time zone, installs `unattended-upgrades`, sets the hostname, and other core settings
- [`docker.yml`](tasks/docker.yml) installs Docker, initializes the Docker daemon, and adds the `connor` user to the docker group
- [`srv-layout.yml`](tasks/srv-layout.yml) constructs the per-application directories using the app configurations inside `site.yml`
- [`firewall.yml`](tasks/firewall.yml) manages the network configuration, currently opening only port 22 to Netbird traffic

Note that the Docker admin status is effectively passwordless root permissioning. This is acceptable only because of the network configuration of the device, and may become a blocker before the device can be safely exposed to the open internet. 

The playbook can be used to update the homelab configuration by running: 

```bash
# checking the diff against the existing configuration
ansible-playbook site.yml --check --diff -K

# applying the playbook
ansible-playbook site.yml --diff -K
```

Note that the `--diff` argument is suggested here so that the difference between the current configuration and the configuration of the setup defined by `site.yml` can be reviewed. The site playbook is idempotent, and will show an exit message `ok` for each set of tasks if the machine was up-to-date with the configuration specified by the playbook prior to running it. Note that the `docker.yml` task will report `changed` on first run. `-K` is provided in order to run the playbook tasks with root permissions.

## Application state and UIDs

`/srv` holds persistent application state with one directory per application. These subdirectories are managed by [`srv-layout.yml`](tasks/srv-layout.yml) using the `srv_apps` list defined by `site.yml`. These directories are configured as bind-backed named volumes, as described in [ADR 0001](../docs/adr/0001-bind-backed-named-volumes.md).

The UID values represent the users that the container processes run as, and the Mode defaults to `root/root/0750`. Parents should be listed before children, since the process in `site.yml` creates the volumes in order. 

This configuration was chosen because it allows backups and restores to target the `/srv` LV instead of walking a shared LV and managing backups using filesystem organization, file whitelists, and so on. This backup strategy and LV configuration will likely be revisited once K8s is implemented. 

## Troubleshooting

### Timeout (12s) waiting for privilege escalation prompt

Ubuntu 26.04 ships `sudo-rs` as the default `sudo`. Ansible identifies the become prompt by checking whether an output line starts with the prompt it sent. `sudo-rs` wraps it as `[sudo: ... ] Password:`, so the match never fires and the password is never sent. `inventory.yml` works around this with `ansible_become_exe: /usr/bin/sudo.ws`, pointing at stock sudo. Seeing this error means that line is missing or `/usr/bin/sudo.ws` no longer exists.

### HOMELAB_SSH_KEY is unset

`.env` has not been exported into the current shell. Run `set -a; source .env; set +a` from `ansible/`. The variable is not inherited by a new terminal tab, so this is per-shell. If `.env` itself is missing, create it from `.env.example`.

### skipping: no hosts matched, or the inventory isn't found

You're not in `ansible/`. `ansible.cfg` is read only from the current directory, so running from the repo root silently loses the inventory and every setting in it.

### Host unreachable

The NetBird mesh is the only path in. UFW drops port 22 on the LAN interface, and the rule that would allow it is commented out in `firewall.yml`. If NetBird is down, recovery requires physical access to the machine.

### A container starts but its published port is unreachable

`files/daemon.json` sets `"ip": "127.0.0.1"` as Docker's default publish address, so a bare `"3000:3000"` binds loopback only. Compose files must name an interface explicitly using `${BIND_IP}`.

### A container can't write its own data directory

UID mismatch between the srv_apps entry and the image. See the table above.

### docker still needs sudo after a successful run

Group membership takes effect at next login, not immediately.

## Kubernetes migration

In the event of a Kubernetes migration, the Ansible configuration of this setup will change substantially. Ansible would continue to run [`bootstrap.yml`](bootstrap.yml). The responsibilities of [`site.yml`](site.yml) will be partially absorbed by the Kubernetes configuration: 

- [`base.yml`](tasks/base.yml) remains unchanged
- [`docker.yml`](tasks/docker.yml) will be completely replaced by a group of K8s setup tasks
- [`srv-layout.yml`](tasks/srv-layout.yml) will be absorbed by K8s' storage configuration
- [`firewall.yml`](tasks/firewall.yml) will be completely rewritten to allow for K8s to direct API, kubelet, and CNI traffic
