---
name: pyinfra-automation
description: >
    Practical guidance for infrastructure automation with pyinfra: project scaffolding,
    inventory/deploy structure, Linux package/service/file operations, Ansible-to-pyinfra
    conversion, custom facts/operations, and Windows hosts via WinRM. Use this when the
    user asks for pyinfra help or Python-based, agentless host provisioning.
---

# Pyinfra Automation Skill

## What This Skill Is

This skill is a playbook for writing and debugging pyinfra projects in a repeatable way.

It helps with:
- Building a clean `inventory.py` + `deploy.py` + `group_data/` layout
- Converting Ansible or shell workflows into idiomatic pyinfra operations
- Adding custom pyinfra SDK facts and operations
- Managing Windows hosts with `pyinfra-windows` over WinRM
- Troubleshooting common idempotency and orchestration mistakes

It is not a generic infrastructure framework guide. It is specifically for pyinfra-based workflows.

pyinfra is a Python-based, agentless infrastructure automation tool. It SSHs into hosts
(or uses Docker/local connectors) and runs operations idempotently. Think Ansible but in
pure Python — no YAML, full language power, fast parallel execution.

## Step 1: Identify the workflow

Determine which use case applies before writing any code:

| User wants to… | Go to |
|---|---|
| Set up servers for the first time | [New Deploy](#new-deploy) |
| Convert Ansible playbook or shell script | [Convert from Ansible/shell](#convert-from-ansibleshell) |
| Write a custom fact or operation | [Custom SDK](#custom-sdk) |
| Manage Windows hosts | [Windows/WinRM](#windowswinrm) |
| Fix or improve existing pyinfra code | [Debug/Improve](#debugimprove) |

---

## Standard Project Scaffold

**Always produce this structure** for any new deploy or project:

```
infra/
├── inventory.py          ← defines host groups
├── deploy.py             ← top-level orchestration
├── group_data/
│   ├── all.py            ← data available to every host
│   └── <group>.py        ← group-specific overrides
└── host_data/
    └── <hostname>.py     ← per-host overrides (optional)
```

**inventory.py** — group hosts:
```python
web = ["web1.example.com", "web2.example.com"]
db = ["db1.example.com"]
```

**group_data/all.py** — shared defaults:
```python
app_user = "deploy"
app_dir = "/opt/app"
```

**deploy.py** — import and call operations:
```python
from pyinfra import host
from pyinfra.operations import apt, files, systemd

# Operations run on all hosts in the inventory
apt.packages(name="Install nginx", packages=["nginx"], present=True)
```

Run with: `pyinfra inventory.py deploy.py`

---

## New Deploy

1. **Gather from user**: host addresses, SSH user, sudo needed, desired end state (what
   should be installed/configured/running)
2. **Scaffold** the project structure above
3. **Fill operations** — use the right namespace for the OS:
   - Debian/Ubuntu → `apt.packages`
   - RHEL/CentOS/Rocky → `yum.packages` or `dnf.packages`
   - Use `files.template` or `files.put` for config files
   - Use `systemd.service` to enable/start services
4. **Add idempotency** — operations are idempotent by default; avoid `server.shell`
   unless there is no dedicated operation
5. **Verify locally** with `pyinfra inventory.py deploy.py --dry-run` before running live

**Common new-deploy template**:
```python
from pyinfra import host
from pyinfra.operations import apt, files, server, systemd

apt.packages(
    name="Install nginx",
    packages=["nginx"],
    update=True,
    present=True,
    _sudo=True,
)

files.directory(
    name="Create web root",
    path="/var/www/mysite",
    user="www-data",
    group="www-data",
    _sudo=True,
)

files.put(
    name="Upload site files",
    src="./site/",
    dest="/var/www/mysite/",
    _sudo=True,
)

systemd.service(
    name="Enable and start nginx",
    service="nginx",
    running=True,
    enabled=True,
    _sudo=True,
)
```

---

## Convert from Ansible/shell

### Ansible module → pyinfra operation mapping

| Ansible | pyinfra |
|---|---|
| `apt` / `yum` | `apt.packages` / `yum.packages` |
| `copy` / `template` | `files.put` / `files.template` |
| `file` (state=directory) | `files.directory` |
| `service` | `systemd.service` |
| `git` | `git.repo` |
| `pip` | `pip.packages` |
| `command` / `shell` | `server.shell` (use sparingly) |
| `user` | `server.user` |
| `lineinfile` | `files.line` |
| `handlers` (notify) | `_if=op.did_change` pattern (see below) |

### Shell script → pyinfra

- `apt-get install foo` → `apt.packages(packages=["foo"])`
- `mkdir -p /opt/app` → `files.directory(path="/opt/app")`
- `systemctl enable --now nginx` → `systemd.service(service="nginx", enabled=True, running=True)`
- `cp ./config /etc/app.conf` → `files.put(src="./config", dest="/etc/app.conf")`

### Warn about these non-idiomatic patterns

- Long `server.shell` commands that could be replaced with dedicated ops
- `when:` conditionals based on `ansible_facts` → use `host.get_fact()` and `_if`
- Ansible `vars_files` / `group_vars` → use `group_data/` hierarchy
- Ansible roles → plain Python modules imported in `deploy.py`

---

## Custom SDK

For custom facts and operations, read **references/sdk-custom.md** — it has the full
patterns with working code examples.

**Quick summary:**

**Custom fact**:
```python
from pyinfra.api import FactBase

class PortListening(FactBase):
    """Check if a TCP port is listening."""
    def command(self, port):
        return f"ss -tlnp | grep ':{port} ' | wc -l"

    def process(self, output):
        return int(output[0].strip()) > 0

    default = False
```

Usage: `is_running = host.get_fact(PortListening, port=5432)`

**Custom operation**:
```python
from pyinfra.api import operation
from pyinfra.api.command import StringCommand

@operation()
def ensure_postgres_running(state, host, port=5432):
    """Only run migrations if postgres is already up."""
    is_up = host.get_fact(PortListening, port=port)
    if is_up:
        yield StringCommand("psql -f migrations.sql")
```

See **references/sdk-custom.md** for the full idempotency checklist and command types.

---

## Windows/WinRM

For Windows host management, read **references/windows-winrm.md** — it has install
instructions, connector syntax, available operations, and caveats.

**Quick summary:**
- Install: `uv tool install pyinfra --with https://github.com/pyinfra-dev/pyinfra-windows.git`
- Target: `pyinfra @winrm/192.168.1.50 deploy.py`
- Use `pyinfra_windows.operations` and `pyinfra_windows.facts` namespaces

---

## Debug/Improve

When reviewing existing pyinfra code, check for:

1. **Non-idempotent `server.shell` calls** — replace with dedicated operations where possible
2. **Missing `_sudo`** — if the operation requires root, add `_sudo=True`
3. **Hardcoded paths** — move to `group_data/all.py` as variables, access via `host.data.path`
4. **Missing service restart on config change** — use `_if=nginx_config.did_change`
5. **No dry-run testing** — remind user to run `--dry-run` before applying
6. **Brittle `server.shell` checks** — replace with custom facts for reliable state detection

---

## Key Idioms

### React to changes (`_if` chaining)
```python
from pyinfra.operations import files, systemd

nginx_config = files.template(
    name="Deploy nginx config",
    src="templates/nginx.conf.j2",
    dest="/etc/nginx/nginx.conf",
    _sudo=True,
)

systemd.service(
    name="Reload nginx if config changed",
    service="nginx",
    reloaded=True,
    _sudo=True,
    _if=nginx_config.did_change,
)
```

### Access host data
```python
from pyinfra import host

# From group_data/all.py or group_data/<group>.py
app_port = host.data.app_port
app_user = host.data.app_user
```

### Conditional logic with facts
```python
from pyinfra import host
from pyinfra.facts.server import LinuxName

distro = host.get_fact(LinuxName)
if distro == "Ubuntu":
    apt.packages(packages=["nginx"])
else:
    yum.packages(packages=["nginx"])
```

### Target specific hosts or groups
```python
# CLI flags
pyinfra inventory.py deploy.py --limit web      # only 'web' group
pyinfra inventory.py deploy.py --dry-run        # simulate only
```

---

## Reference Files

| File | Read when |
|---|---|
| `references/core-concepts.md` | Need full operations cheatsheet, built-in facts, inventory patterns, or global arguments |
| `references/sdk-custom.md` | Writing custom facts, custom operations, or complex idempotency patterns |
| `references/windows-winrm.md` | Any Windows host management, WinRM connector setup, or pyinfra-windows ops |
