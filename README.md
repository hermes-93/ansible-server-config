# ansible-server-config

Ansible playbooks and roles for automated server configuration.
Covers base hardening, Docker installation, Nginx setup, and security hardening for Ubuntu 22.04+ servers.

## Architecture

```
ansible-server-config/
├── ansible.cfg               # Ansible configuration
├── inventory/
│   ├── hosts.ini             # Server inventory
│   └── group_vars/
│       ├── all.yml           # Variables for all hosts
│       └── webservers.yml    # Variables for webservers group
├── playbooks/
│   ├── site.yml              # Master playbook (all roles)
│   ├── webservers.yml        # Web server setup
│   └── security.yml          # Security hardening only
├── roles/
│   ├── common/               # Base packages, swap, sysctl tuning
│   ├── docker/               # Docker CE + Docker Compose v2
│   ├── nginx/                # Nginx install + vhost config
│   └── security/             # SSH hardening, fail2ban, ufw firewall
└── .github/workflows/
    ├── ci.yml                # Lint + syntax check + molecule dry-run
    └── deploy-check.yml      # Ansible --check mode against inventory
```

## Roles

| Role     | Description                                                   |
|----------|---------------------------------------------------------------|
| common   | System packages, swap, sysctl tuning, NTP, timezone          |
| docker   | Docker CE, Docker Compose v2, non-root docker group          |
| nginx    | Nginx install, vhost templates, logrotate                    |
| security | SSH hardening, fail2ban, ufw firewall, auditd, sysctl knobs  |

## Quick Start

### Prerequisites

```bash
pip install ansible ansible-lint
```

### Run against all servers

```bash
# Syntax check first
ansible-playbook playbooks/site.yml --syntax-check

# Dry-run (no changes)
ansible-playbook playbooks/site.yml --check --diff

# Apply
ansible-playbook playbooks/site.yml
```

### Run a single role

```bash
ansible-playbook playbooks/site.yml --tags docker
ansible-playbook playbooks/site.yml --tags security
```

### Run against a single host

```bash
ansible-playbook playbooks/site.yml --limit web01
```

## Variables

Key variables (see `inventory/group_vars/all.yml`):

| Variable              | Default   | Description                     |
|-----------------------|-----------|---------------------------------|
| `timezone`            | `UTC`     | Server timezone                 |
| `swap_enabled`        | `true`    | Enable swap                     |
| `swap_size_mb`        | `2048`    | Swap size in MB                 |
| `ssh_port`            | `22`      | SSH port (security role)        |
| `nginx_http_port`     | `80`      | Nginx HTTP port                 |
| `nginx_https_port`    | `443`     | Nginx HTTPS port                |

## Testing

```bash
pip install molecule molecule-docker pytest testinfra
molecule test --all
```

## CI/CD

Every push triggers:
1. **yamllint** — YAML formatting
2. **ansible-lint** — Ansible best practices
3. **syntax-check** — `ansible-playbook --syntax-check`
4. **molecule converge** — Docker-based dry-run

## License

MIT
