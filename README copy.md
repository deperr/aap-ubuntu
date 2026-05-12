# aap-ubuntu

Create local apt repositories and configure Ubuntu hosts to pull packages from an internal mirror server. Designed for Ansible Automation Platform (AAP) 2.6.

## Repository Structure

```
playbooks/
  setup_mirror_server.yml              # Stand up apt-mirror + nginx on a mirror host
  configure_client_mirrors.yml         # Configure clients (Docker, Ubuntu, or both) + optional source lockdown
  configure_docker_client_mirror.yml   # Shortcut — Docker CE clients only
  configure_ubuntu_client_mirror.yml   # Shortcut — Ubuntu OS clients only

roles/
  config_server_ubuntu_mirror/         # Installs apt-mirror, nginx, cron sync
  config_client_docker_mirror/         # Adds Docker CE source on clients
  config_client_ubuntu_mirror/         # Rewrites Ubuntu OS sources on clients
  config_client_external_sources/      # Removes any apt source not referencing the internal mirror
```

## Roles

### config_server_ubuntu_mirror

Installs `apt-mirror` and `nginx` on a mirror host, deploys `/etc/apt/mirror.list`, enables an nginx vhost to serve mirrored content, and schedules a cron job for periodic syncs.

Supports two profiles via `apt_mirror_server_profile`:

| Profile | Upstream | Disk footprint |
|---------|----------|----------------|
| `ubuntu` | `http://archive.ubuntu.com/ubuntu` | TB-scale (full archive) |
| `docker_ce` | `https://download.docker.com/linux/ubuntu` | Small (recommended for POC) |

### config_client_docker_mirror

Downloads Docker's GPG signing key and drops a `sources.list.d` entry pointing at the internal mirror.

### config_client_ubuntu_mirror

Rewrites the system apt sources (`ubuntu.sources` deb822 on 24.04+ or classic `sources.list` on older releases) to point at the internal mirror.

### config_client_external_sources

Reads every file in `/etc/apt/sources.list.d/` and removes any that do **not** contain the allowed origin hostname. Optionally blanks `/etc/apt/sources.list` if it also doesn't reference the mirror. Useful for locking VMs to internal sources only.

## Shared Variables

All client roles derive their mirror URL from a single variable:

| Variable | Default | Description |
|----------|---------|-------------|
| `mirror_server_hostname` | `ubuntu-svr-01.dperrone.dev` | FQDN or IP of the mirror host |

If not set, all client roles fall back to `ubuntu-svr-01.dperrone.dev`.

---

## AAP Job Templates

### 1. Setup Mirror Server

| Field | Value |
|-------|-------|
| **Name** | Setup Mirror Server |
| **Playbook** | `playbooks/setup_mirror_server.yml` |
| **Inventory** | Mirror server host(s) |
| **Credential** | Machine credential (SSH + become) |

#### Survey (standalone)

| Prompt | Variable | Type | Default | Required |
|--------|----------|------|---------|----------|
| Mirror profile | `apt_mirror_server_profile` | Multiple choice (`ubuntu`, `docker_ce`) | `docker_ce` | Yes |
| Upstream URL | `apt_mirror_server_upstream` | Text | `https://download.docker.com/linux/ubuntu` | Yes |
| Codenames (JSON list) | `apt_mirror_server_codenames` | Text | `["noble"]` | Yes |
| Architectures (JSON list) | `apt_mirror_server_architectures` | Text | `["amd64"]` | No |
| nginx listen directive | `apt_mirror_server_nginx_listen` | Text | `80 default_server` | No |
| Cron hour | `apt_mirror_server_cron_hour` | Text | `2` | No |
| Cron minute | `apt_mirror_server_cron_minute` | Text | `30` | No |
| Run initial sync now? | `apt_mirror_server_run_initial_sync` | Multiple choice (`true`, `false`) | `false` | Yes |

### 2. Configure Client Mirrors

| Field | Value |
|-------|-------|
| **Name** | Configure Client Mirrors |
| **Playbook** | `playbooks/configure_client_mirrors.yml` |
| **Inventory** | Ubuntu client VMs |
| **Credential** | Machine credential (SSH + become) |

#### Survey (standalone)

| Prompt | Variable | Type | Default | Required |
|--------|----------|------|---------|----------|
| Mirror server hostname | `mirror_server_hostname` | Text | `ubuntu-svr-01.dperrone.dev` | Yes |
| Client profiles (JSON list) | `client_mirror_profiles` | Text | `["docker_ce"]` | Yes |
| Docker codename | `config_client_docker_mirror_codename` | Text | `noble` | No |
| Docker channel | `config_client_docker_mirror_channel` | Text | `stable` | No |
| Remove external sources? | `config_client_external_sources_enabled` | Multiple choice (`true`, `false`) | `false` | No |
| Disable sources.list? | `config_client_external_sources_disable_sources_list` | Multiple choice (`true`, `false`) | `true` | No |

---

## AAP Workflow Template

### Deploy Mirror and Configure Clients

Chain the two job templates so the mirror is ready before clients are configured.

```
[Setup Mirror Server] ──success──> [Configure Client Mirrors]
```

When running as a workflow, each job template should have its **survey disabled** on the node (surveys are answered at the workflow level instead). Define the survey on the **Workflow Job Template** itself.

#### Workflow Survey

| Prompt | Variable | Type | Default | Required |
|--------|----------|------|---------|----------|
| Mirror server hostname | `mirror_server_hostname` | Text | `ubuntu-svr-01.dperrone.dev` | Yes |
| Mirror profile | `apt_mirror_server_profile` | Multiple choice (`ubuntu`, `docker_ce`) | `docker_ce` | Yes |
| Upstream URL | `apt_mirror_server_upstream` | Text | `https://download.docker.com/linux/ubuntu` | Yes |
| Codenames (JSON list) | `apt_mirror_server_codenames` | Text | `["noble"]` | Yes |
| Run initial sync now? | `apt_mirror_server_run_initial_sync` | Multiple choice (`true`, `false`) | `false` | Yes |
| Client profiles (JSON list) | `client_mirror_profiles` | Text | `["docker_ce"]` | Yes |
| Remove external sources? | `config_client_external_sources_enabled` | Multiple choice (`true`, `false`) | `false` | No |

> **Tip:** When using the workflow, variables from the workflow survey are passed to all nodes. Node-specific overrides (e.g. different inventories) are configured on each workflow node, not in the survey.

#### Standalone vs Workflow survey differences

| Variable | Standalone: Setup Mirror Server | Standalone: Configure Client Mirrors | Workflow |
|----------|:---:|:---:|:---:|
| `mirror_server_hostname` | - | Yes | Yes |
| `apt_mirror_server_profile` | Yes | - | Yes |
| `apt_mirror_server_upstream` | Yes | - | Yes |
| `apt_mirror_server_codenames` | Yes | - | Yes |
| `apt_mirror_server_architectures` | Yes | - | - |
| `apt_mirror_server_nginx_listen` | Yes | - | - |
| `apt_mirror_server_cron_hour` | Yes | - | - |
| `apt_mirror_server_cron_minute` | Yes | - | - |
| `apt_mirror_server_run_initial_sync` | Yes | - | Yes |
| `client_mirror_profiles` | - | Yes | Yes |
| `config_client_docker_mirror_codename` | - | Yes | - |
| `config_client_docker_mirror_channel` | - | Yes | - |
| `config_client_external_sources_enabled` | - | Yes | Yes |
| `config_client_external_sources_disable_sources_list` | - | Yes | - |

Variables marked `-` in a column are either not relevant to that template or left at role defaults (can be added to the survey if needed).

---

## Quick Start (CLI)

```bash
# 1. Set up the mirror server (Docker CE POC)
ansible-playbook playbooks/setup_mirror_server.yml -l mirror_server \
  -e apt_mirror_server_profile=docker_ce \
  -e apt_mirror_server_upstream=https://download.docker.com/linux/ubuntu \
  -e 'apt_mirror_server_codenames=["noble"]'

# 2. Run the first sync on the mirror server
ssh mirror-server 'sudo apt-mirror'

# 3. Point client VMs at the mirror (Docker CE only)
ansible-playbook playbooks/configure_client_mirrors.yml -l ubuntu_clients \
  -e mirror_server_hostname=ubuntu-svr-01.dperrone.dev \
  -e 'client_mirror_profiles=["docker_ce"]'

# 4. Optionally lock down external sources
ansible-playbook playbooks/configure_client_mirrors.yml -l ubuntu_clients \
  -e mirror_server_hostname=ubuntu-svr-01.dperrone.dev \
  -e 'client_mirror_profiles=["docker_ce"]' \
  -e config_client_external_sources_enabled=true
```

## Verification

On any configured client:

```bash
# Confirm the Docker source points at the mirror
cat /etc/apt/sources.list.d/docker-ce-mirror.list

# Verify apt uses the mirror
apt-cache policy docker-ce

# Watch download URLs during install
sudo apt-get install -y docker-ce 2>&1 | grep '^Get:'
```

On the mirror server:

```bash
# Watch requests arrive
tail -f /var/log/nginx/access.log
```

## Notes

- The mirror server listens on **port 80** by default. Override `apt_mirror_server_nginx_listen` to change it.
- First Ubuntu archive sync can take **hours to days** and consumes TB of disk. Docker CE sync is much smaller and completes in minutes.
- Client OS packages (kernel, libc, etc.) still come from Ubuntu's public repos unless you also enable the `ubuntu` client profile **and** have a running Ubuntu archive mirror.
- The `config_client_external_sources` role is **opt-in** (`config_client_external_sources_enabled: false` by default). When enabled, any apt source file not referencing your mirror hostname is removed.
