# pyinfra Core Concepts Reference

## Table of Contents
1. [Operations Cheatsheet](#operations-cheatsheet)
2. [Facts System](#facts-system)
3. [Inventory Patterns](#inventory-patterns)
4. [group_data / host_data Hierarchy](#group_data--host_data-hierarchy)
5. [Global Arguments](#global-arguments)
6. [`_if` Chaining Pattern](#_if-chaining-pattern)
7. [CLI Flags](#cli-flags)

---

## Operations Cheatsheet

### Package Management

```python
from pyinfra.operations import apt, yum, dnf, pacman, pip

# Debian/Ubuntu
apt.packages(name="Install nginx", packages=["nginx", "curl"], update=True, _sudo=True)
apt.packages(name="Remove apache", packages=["apache2"], present=False, _sudo=True)
apt.ppa(name="Add certbot PPA", src="ppa:certbot/certbot", _sudo=True)
apt.key(name="Add Docker GPG key", src="https://download.docker.com/linux/ubuntu/gpg", _sudo=True)
apt.repo(name="Add Docker repo", src="deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable", _sudo=True)

# RHEL/CentOS/Rocky/Fedora
yum.packages(name="Install nginx", packages=["nginx"], _sudo=True)
dnf.packages(name="Install nginx", packages=["nginx"], _sudo=True)

# Arch Linux
pacman.packages(name="Install nginx", packages=["nginx"], _sudo=True)

# Python
pip.packages(name="Install deps", packages=["flask", "gunicorn"], present=True)
pip.packages(name="Install from requirements", requirements="requirements.txt")
pip.venv(name="Create venv", path="/opt/app/venv")
```

### Files and Directories

```python
from pyinfra.operations import files

# Upload a single file
files.put(name="Upload config", src="./configs/app.conf", dest="/etc/app.conf", _sudo=True)

# Upload a Jinja2 template (variables from host.data.*)
files.template(
    name="Render nginx config",
    src="templates/nginx.conf.j2",
    dest="/etc/nginx/sites-available/mysite",
    _sudo=True,
)

# Create/ensure a directory exists
files.directory(
    name="Create app dir",
    path="/opt/app",
    user="deploy",
    group="deploy",
    mode="755",
    _sudo=True,
)

# Create/ensure a file exists (touch)
files.file(name="Create empty file", path="/etc/app/.lock", present=True, _sudo=True)

# Create/remove a symlink
files.link(
    name="Enable nginx site",
    path="/etc/nginx/sites-enabled/mysite",
    target="/etc/nginx/sites-available/mysite",
    _sudo=True,
)

# Replace/ensure a line in a file
files.line(
    name="Set max connections",
    path="/etc/postgresql/14/main/postgresql.conf",
    line="max_connections = 200",
    replace="max_connections = .*",
    _sudo=True,
)

# Sync a local directory to remote (rsync-like)
files.rsync(name="Sync site files", src="./site/", dest="/var/www/html/", _sudo=True)

# Set file permissions
files.chmod(name="Set permissions", path="/opt/app/bin/server", mode="755", _sudo=True)
files.chown(name="Set ownership", path="/opt/app", user="deploy", recursive=True, _sudo=True)
```

### Systemd Services

```python
from pyinfra.operations import systemd

# Enable and start a service
systemd.service(
    name="Start and enable nginx",
    service="nginx",
    running=True,
    enabled=True,
    _sudo=True,
)

# Reload (not restart) a service
systemd.service(name="Reload nginx", service="nginx", reloaded=True, _sudo=True)

# Restart a service
systemd.service(name="Restart postgres", service="postgresql", restarted=True, _sudo=True)

# Disable and stop
systemd.service(
    name="Disable apache",
    service="apache2",
    running=False,
    enabled=False,
    _sudo=True,
)

# Reload systemd daemon (after installing a new unit file)
systemd.daemon_reload(name="Reload systemd", _sudo=True)
```

### Git

```python
from pyinfra.operations import git

# Clone or update a repo
git.repo(
    name="Clone app repo",
    src="https://github.com/myorg/myapp.git",
    dest="/opt/app",
    branch="main",
    update=True,
    _sudo=True,
    _sudo_user="deploy",
)
```

### Server / System

```python
from pyinfra.operations import server

# Create a system user
server.user(
    name="Create deploy user",
    user="deploy",
    shell="/bin/bash",
    home="/home/deploy",
    create_home=True,
    _sudo=True,
)

# Add user to a group
server.group(name="Create app group", group="appgroup", _sudo=True)

# Run an arbitrary shell command (use sparingly — not idempotent by default)
server.shell(name="Run migration", commands=["cd /opt/app && python manage.py migrate"], _sudo=True)

# Reboot the host
server.reboot(name="Reboot after kernel update", delay=10, interval=5, reboot_timeout=300, _sudo=True)

# Set hostname
server.hostname(name="Set hostname", hostname="web1.example.com", _sudo=True)

# Manage cron jobs
server.cron(
    name="Add backup cron",
    name="backup",
    command="/opt/scripts/backup.sh",
    minute="0",
    hour="2",
    user="deploy",
    _sudo=True,
)
```

---

## Facts System

Facts let you inspect current host state before deciding what to do.

### Common Built-in Facts

```python
from pyinfra import host
from pyinfra.facts.server import (
    LinuxName,        # "Ubuntu", "Debian", "CentOS", etc.
    LinuxDistribution,  # {"name": "Ubuntu", "version": "22.04", ...}
    Hostname,
    Users,            # dict of {username: {...}}
    Groups,
    Command,          # run an arbitrary command and get output
    Which,            # path to a binary (None if not found)
    Path,             # info about a path (size, type, mtime, etc.)
)
from pyinfra.facts.files import File, Directory, FindInFile
from pyinfra.facts.apt import AptPackages, AptSources
from pyinfra.facts.systemd import SystemdStatus
from pyinfra.facts.python import PythonVersion

# Usage
distro = host.get_fact(LinuxName)
users = host.get_fact(Users)
nginx_installed = host.get_fact(Command, command="which nginx")
```

### Using Facts for Conditional Logic

```python
from pyinfra import host
from pyinfra.facts.server import LinuxName
from pyinfra.operations import apt, yum

distro = host.get_fact(LinuxName)

if distro in ("Ubuntu", "Debian"):
    apt.packages(packages=["postgresql"], _sudo=True)
elif distro in ("CentOS", "Rocky Linux", "AlmaLinux"):
    yum.packages(packages=["postgresql-server"], _sudo=True)
```

### Fact Caching

Facts are cached per host per deploy run. Calling `host.get_fact(Hostname)` twice returns
the same value. To force re-evaluation within a complex deploy, use a fresh call in a
separate deploy file.

---

## Inventory Patterns

### SSH (most common)

```python
# inventory.py
web = ["192.168.1.10", "192.168.1.11", "web.example.com"]
db = ["db.example.com"]

# With per-host SSH options
web = [
    ("192.168.1.10", {"ssh_user": "ubuntu", "ssh_key": "~/.ssh/id_rsa"}),
    ("192.168.1.11", {"ssh_user": "deploy"}),
]
```

Run: `pyinfra inventory.py deploy.py`

### Ad-hoc targets (no inventory file)

```python
# Single host
pyinfra 192.168.1.10 deploy.py

# Multiple hosts (comma-separated)
pyinfra 192.168.1.10,192.168.1.11 deploy.py

# Docker container
pyinfra @docker/ubuntu:22.04 deploy.py

# Local machine
pyinfra @local deploy.py

# Windows via WinRM
pyinfra @winrm/192.168.1.50 deploy.py
```

### Dynamic inventory with Python

```python
# inventory.py — generate hosts from an API or file
import json

with open("hosts.json") as f:
    data = json.load(f)

web = [h["ip"] for h in data if h["role"] == "web"]
db  = [h["ip"] for h in data if h["role"] == "db"]
```

### SSH connection options

Set globally in `group_data/all.py`:
```python
ssh_user = "deploy"
ssh_key = "~/.ssh/mykey"
ssh_port = 22
```

Or per group in `group_data/web.py`:
```python
ssh_user = "ubuntu"
```

---

## group_data / host_data Hierarchy

Data flows from broad to specific; more specific overrides less specific.

```
group_data/all.py       ← applies to ALL hosts
group_data/web.py       ← applies only to the 'web' group
host_data/web1.py       ← applies only to host 'web1'
```

**Example:**

`group_data/all.py`:
```python
app_user = "deploy"
app_dir = "/opt/app"
app_port = 8000
db_host = "localhost"
```

`group_data/db.py`:
```python
db_host = "127.0.0.1"
db_port = 5432
```

`host_data/db1.example.com.py`:
```python
db_port = 5433  # non-standard port on this specific host
```

**Access in deploy.py:**
```python
from pyinfra import host

print(host.data.app_dir)    # "/opt/app"
print(host.data.app_port)   # 8000
print(host.data.db_port)    # 5432 (or 5433 for db1)
```

---

## Global Arguments

These apply to any operation as keyword arguments:

| Argument | Type | Description |
|---|---|---|
| `_sudo` | bool | Run via `sudo` |
| `_sudo_user` | str | `sudo -u <user>` |
| `_su_user` | str | `su - <user>` |
| `_env` | dict | Set environment variables |
| `_chdir` | str | Change to this directory first |
| `_shell_executable` | str | Override shell (default: `/bin/sh`) |
| `_timeout` | int | Seconds before command times out |
| `_retries` | int | Retry count on failure |
| `_retry_delay` | int | Seconds between retries |
| `_if` | operation | Only run if this operation's `did_change` is True |
| `_ignore_errors` | bool | Continue even if command fails |
| `_serial` | bool | Run on hosts sequentially (not in parallel) |
| `_run_once` | bool | Run only once across all matched hosts |
| `_get_pty` | bool | Allocate a PTY (needed for some interactive commands) |

---

## `_if` Chaining Pattern

The `_if` argument makes an operation conditional on whether a previous operation made
a change. This is how you implement Ansible-style "notify handlers":

```python
from pyinfra.operations import apt, files, systemd

# Step 1: install the package (idempotent — does nothing if already installed)
apt.packages(name="Install nginx", packages=["nginx"], _sudo=True)

# Step 2: deploy the config
nginx_config = files.template(
    name="Deploy nginx config",
    src="templates/nginx.conf.j2",
    dest="/etc/nginx/nginx.conf",
    _sudo=True,
)

# Step 3: only reload if the config actually changed
systemd.service(
    name="Reload nginx if config changed",
    service="nginx",
    reloaded=True,
    _sudo=True,
    _if=nginx_config.did_change,   # ← the magic
)
```

`op.did_change` is a deferred boolean — it evaluates to True only when the operation
actually modified the host during this run.

---

## CLI Flags

```bash
# Dry run — show what would happen without making changes
pyinfra inventory.py deploy.py --dry-run

# Limit to specific hosts or groups
pyinfra inventory.py deploy.py --limit web
pyinfra inventory.py deploy.py --limit web1.example.com

# Show verbose output
pyinfra inventory.py deploy.py --debug

# Run with a specific SSH key
pyinfra inventory.py deploy.py --ssh-key ~/.ssh/deploy_key

# Run with sudo password prompt
pyinfra inventory.py deploy.py --sudo-password

# Run operations in a specific order (no parallelism)
pyinfra inventory.py deploy.py --serial

# Facts only — list facts for hosts without deploying
pyinfra inventory.py fact server.Hostname
pyinfra 192.168.1.10 fact server.LinuxDistribution
```
