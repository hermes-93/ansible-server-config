# ansible-server-config

[![CI](https://github.com/hermes-93/ansible-server-config/actions/workflows/ci.yml/badge.svg)](https://github.com/hermes-93/ansible-server-config/actions/workflows/ci.yml)
[![Deploy Check](https://github.com/hermes-93/ansible-server-config/actions/workflows/deploy-check.yml/badge.svg)](https://github.com/hermes-93/ansible-server-config/actions/workflows/deploy-check.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Production-ready Ansible roles for Ubuntu 22.04+ server configuration.
Covers baseline hardening, Docker CE, Nginx, and security controls — tested with Molecule on every commit.

## Architecture

```
ansible-server-config/
├── ansible.cfg                   # Project-scoped Ansible configuration
├── inventory/
│   ├── hosts.ini                 # Host groups: webservers, dbservers, monitoring
│   └── group_vars/
│       ├── all.yml               # Variables applied to every host
│       └── webservers.yml        # Nginx + Docker overrides for web tier
├── playbooks/
│   ├── site.yml                  # Full stack: common → docker → nginx → security
│   ├── webservers.yml            # Web-tier only
│   └── security.yml             # Security hardening in isolation
├── roles/
│   ├── common/                   # Base OS setup
│   ├── docker/                   # Container runtime
│   ├── nginx/                    # Web server
│   └── security/                 # Hardening controls
├── molecule/
│   ├── default/                  # All 4 roles — full-stack test
│   └── security/                 # Security role in isolation
└── .github/workflows/
    ├── ci.yml                    # Lint → syntax-check → molecule matrix
    └── deploy-check.yml          # --check dry-run against live inventory
```

## Role Reference

### `common` — Base OS

| Feature | Details |
|---------|---------|
| Package management | Installs curl, git, htop, vim, jq, tzdata, etc. |
| Timezone | `community.general.timezone` via `timezone` variable |
| Swap | Creates and mounts swap file (size configurable) |
| Sysctl | Kernel tuning: `vm.swappiness`, `net.core.somaxconn`, file-max |
| NTP | Enables `systemd-timesyncd` |

### `docker` — Container Runtime

| Feature | Details |
|---------|---------|
| Packages | `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin` |
| Repository | Official Docker APT repo, GPG-verified |
| Daemon config | `/etc/docker/daemon.json` — json-file logging, overlay2, live-restore |
| Service | Enabled at boot, configurable via `docker_service_state` |
| Users | Adds specified users to `docker` group |

### `nginx` — Web Server

| Feature | Details |
|---------|---------|
| Install | `nginx` package from Ubuntu APT |
| Config | `nginx.conf.j2` — worker tuning, gzip, security headers |
| Virtual hosts | `vhost.conf.j2` — per-host HTTP/HTTPS vhost template |
| Service | Enabled at boot, reloaded on config change |

### `security` — Hardening Controls

| Feature | Details |
|---------|---------|
| SSH | Custom `sshd_config` — port, auth methods, timeouts, ciphers |
| fail2ban | Installed and configured via `jail.local.j2` |
| UFW | Default deny-in / allow-out; rules per `ufw_rules` list |
| auditd | Kernel audit rules from `audit.rules` |
| Sysctl | `net.ipv4.ip_forward=0`, SYN-cookie, ICMP redirect blocks, ASLR |
| Packages | Removes `telnet`, `rsh-*`, `nis`, `yp-tools` |
| Passwords | Sets `PASS_MAX_DAYS=90` / `PASS_MIN_DAYS=1` in `/etc/login.defs` |

## CI Pipeline

```
push / pull_request
        │
        ▼
  ┌───────────┐
  │   Lint    │  yamllint + ansible-lint --profile production
  └─────┬─────┘
        │
        ▼
  ┌──────────────┐
  │ Syntax Check │  ansible-playbook --syntax-check (site.yml + security.yml)
  └──────┬───────┘
         │
         ▼
  ┌──────────────────────────────────┐
  │   Molecule Tests (matrix)         │
  │  default scenario  │  security    │
  │  (all 4 roles)     │  (isolated)  │
  └──────────────────────────────────┘
```

## Quick Start

### Prerequisites

```bash
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
```

### Run the full stack

```bash
# Syntax check
ansible-playbook playbooks/site.yml --syntax-check

# Dry-run with diff
ansible-playbook playbooks/site.yml --check --diff

# Apply
ansible-playbook playbooks/site.yml
```

### Run a single role

```bash
ansible-playbook playbooks/site.yml --tags docker
ansible-playbook playbooks/site.yml --tags security
ansible-playbook playbooks/site.yml --tags "nginx,common"
```

### Target a single host

```bash
ansible-playbook playbooks/site.yml --limit web01
```

### Security hardening only

```bash
ansible-playbook playbooks/security.yml
```

## Key Variables

Full defaults in each `roles/<name>/defaults/main.yml`.

### `common`

| Variable | Default | Description |
|----------|---------|-------------|
| `timezone` | `UTC` | Server timezone |
| `swap_enabled` | `true` | Create a swap file |
| `swap_size_mb` | `2048` | Swap file size (MB) |
| `swap_swappiness` | `10` | `vm.swappiness` kernel param |

### `docker`

| Variable | Default | Description |
|----------|---------|-------------|
| `docker_edition` | `ce` | Docker edition |
| `docker_service_state` | `started` | Service state on apply |
| `docker_service_enabled` | `true` | Start at boot |
| `docker_users` | `[]` | Users added to `docker` group |

### `nginx`

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_http_port` | `80` | HTTP listen port |
| `nginx_https_port` | `443` | HTTPS listen port |
| `nginx_server_name` | `_` | Server name (catch-all) |

### `security`

| Variable | Default | Description |
|----------|---------|-------------|
| `ssh_port` | `22` | SSH listen port |
| `ssh_permit_root_login` | `no` | Allow root SSH login |
| `ssh_password_authentication` | `no` | Password auth |
| `fail2ban_enabled` | `true` | Enable fail2ban |
| `ufw_enabled` | `true` | Enable UFW firewall |
| `auditd_enabled` | `true` | Enable auditd |

## Testing with Molecule

```bash
# Run all scenarios
molecule test --all

# Run a specific scenario
molecule test --scenario-name default
molecule test --scenario-name security

# Step-by-step (useful for debugging)
molecule create
molecule converge
molecule verify
molecule destroy
```

The `default` scenario spins up an Ubuntu 22.04 container with systemd and applies all 4 roles.
The `security` scenario tests the security role in isolation.

## Inventory Example

```ini
[webservers]
web01 ansible_host=10.0.1.10
web02 ansible_host=10.0.1.11

[dbservers]
db01 ansible_host=10.0.2.10

[all:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
```

## License

MIT
