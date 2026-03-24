# pyinfra Windows/WinRM Reference

> **Stability notice**: pyinfra-windows is a community module under active development.
> As of early 2026 it is suitable for development, testing, and light automation — but
> not yet recommended for critical production Windows infrastructure. Verify the current
> state of the project before committing to large-scale use.

## Table of Contents
1. [Installation](#installation)
2. [Targeting Windows Hosts](#targeting-windows-hosts)
3. [Authentication](#authentication)
4. [Available Operations](#available-operations)
5. [Available Facts](#available-facts)
6. [Mixed Linux + Windows Inventory](#mixed-linux--windows-inventory)
7. [Full Example](#full-example)
8. [Troubleshooting](#troubleshooting)

---

## Installation

Install pyinfra with the pyinfra-windows extra from the GitHub source:

```bash
# Using uv (recommended)
uv tool install pyinfra --with "pyinfra-windows @ git+https://github.com/pyinfra-dev/pyinfra-windows.git"

# Using pip
pip install pyinfra
pip install "pyinfra-windows @ git+https://github.com/pyinfra-dev/pyinfra-windows.git"
```

**WinRM prerequisites on the Windows host:**

```powershell
# Run on the Windows host as Administrator
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Service\Auth\Basic -Value $true
Set-Item WSMan:\localhost\Service\AllowUnencrypted -Value $true   # dev only; use HTTPS in prod
winrm set winrm/config/service '@{MaxConcurrentOperationsPerUser="100"}'
```

For production, enable WinRM over HTTPS and use certificate or Kerberos auth instead of
Basic/unencrypted.

---

## Targeting Windows Hosts

Use the `@winrm/<address>` connector syntax:

```bash
# Ad-hoc single host
pyinfra @winrm/192.168.1.50 deploy.py

# With explicit credentials
pyinfra "@winrm/192.168.1.50" deploy.py \
  --winrm-username Administrator \
  --winrm-password 'MyP@ssword'

# Multiple hosts
pyinfra "@winrm/192.168.1.50,@winrm/192.168.1.51" deploy.py
```

In an inventory file:

```python
# inventory.py
windows_hosts = [
    ("@winrm/192.168.1.50", {
        "winrm_username": "Administrator",
        "winrm_password": "MyP@ssword",
        "winrm_port": 5985,          # 5985 = HTTP (default), 5986 = HTTPS
        "winrm_transport": "ntlm",   # "basic", "ntlm", "kerberos", "certificate"
    }),
]
```

---

## Authentication

| Transport | When to use | Notes |
|---|---|---|
| `basic` | Dev/lab only | Requires unencrypted WinRM; not for production |
| `ntlm` | Domain or workgroup | No domain join required; encrypted by default |
| `kerberos` | Active Directory | Requires `kerberos` Python extras; most secure |
| `certificate` | Cert-based auth | Requires cert setup on both sides |

```python
# group_data/windows.py — common WinRM settings
winrm_username = "Administrator"
winrm_password = "{{ vault.windows_admin_password }}"
winrm_transport = "ntlm"
winrm_port = 5985
```

---

## Available Operations

Import from `pyinfra_windows.operations`:

```python
from pyinfra_windows import operations as win_ops
```

### Files

```python
# Upload a file to the Windows host
win_ops.files.put(
    name="Upload config",
    src="./configs/app.config",
    dest="C:\\inetpub\\wwwroot\\app.config",
)

# Create a directory
win_ops.files.directory(
    name="Create app directory",
    path="C:\\Program Files\\MyApp",
)

# Remove a file
win_ops.files.file(
    name="Remove old config",
    path="C:\\old\\config.xml",
    present=False,
)
```

### PowerShell commands

```python
# Run a PowerShell command
win_ops.powershell.command(
    name="Set execution policy",
    command="Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force",
)

# Run a PowerShell script file
win_ops.powershell.script(
    name="Run setup script",
    src="./scripts/setup.ps1",
)
```

### Windows Features / Roles

```python
# Enable a Windows feature (IIS, etc.)
win_ops.windows.feature(
    name="Install IIS",
    feature="Web-Server",
    include_sub_features=True,
    include_management_tools=True,
)
```

### Services

```python
# Manage a Windows service
win_ops.services.service(
    name="Start and enable W3SVC",
    service="W3SVC",
    running=True,
    enabled=True,
)
```

---

## Available Facts

Import from `pyinfra_windows.facts`:

```python
from pyinfra_windows import facts as win_facts
from pyinfra import host

# Get system info (ComputerInfo, OS version, CPU, RAM, etc.)
sys_info = host.get_fact(win_facts.server.ComputerInfo)
print(sys_info["OsName"])          # "Microsoft Windows Server 2022 Standard"
print(sys_info["CsNumberOfProcessors"])
print(sys_info["OsTotalVisibleMemorySize"])

# List installed programs
installed = host.get_fact(win_facts.server.InstalledPrograms)
for prog in installed:
    print(prog["DisplayName"], prog.get("DisplayVersion"))

# Check if a Windows feature is installed
features = host.get_fact(win_facts.server.WindowsFeatures)
iis_installed = any(f["Name"] == "Web-Server" and f["Installed"] for f in features)

# Running services
services = host.get_fact(win_facts.server.WindowsServices)
w3svc = next((s for s in services if s["Name"] == "W3SVC"), None)
```

---

## Mixed Linux + Windows Inventory

You can manage Linux and Windows hosts in the same project. Use group-level data and
conditional imports:

```python
# inventory.py
linux_web = ["192.168.1.10", "192.168.1.11"]
windows_app = [
    ("@winrm/192.168.1.50", {"winrm_username": "Administrator", "winrm_password": "pass"}),
]
```

```python
# group_data/windows_app.py
winrm_transport = "ntlm"
app_dir = "C:\\Program Files\\MyApp"
```

```python
# deploy.py
from pyinfra import host
from pyinfra.facts.server import LinuxName

# Only run linux ops on linux hosts
if host.get_fact(LinuxName):
    from pyinfra.operations import apt
    apt.packages(name="Install nginx", packages=["nginx"], _sudo=True)

# Windows hosts won't have LinuxName — guard windows ops
if not host.get_fact(LinuxName):
    from pyinfra_windows import operations as win_ops
    win_ops.windows.feature(name="Install IIS", feature="Web-Server")
```

Or use `host.groups` to check:
```python
if "windows_app" in host.groups:
    # windows operations
elif "linux_web" in host.groups:
    # linux operations
```

---

## Full Example

Get system info from a Windows Server 2022 host:

```python
# deploy.py
from pyinfra import host
from pyinfra_windows import facts as win_facts
from pyinfra_windows import operations as win_ops

# Gather facts first
sys_info = host.get_fact(win_facts.server.ComputerInfo)
print(f"OS: {sys_info.get('OsName', 'unknown')}")
print(f"RAM: {sys_info.get('OsTotalVisibleMemorySize', 0) // 1024} MB")

# Ensure required directory exists
win_ops.files.directory(
    name="Create app directory",
    path="C:\\Program Files\\MyApp",
)

# Upload configuration
win_ops.files.put(
    name="Upload app config",
    src="./configs/app.config",
    dest="C:\\Program Files\\MyApp\\app.config",
)

# Run PowerShell setup
win_ops.powershell.script(
    name="Run setup",
    src="./scripts/setup.ps1",
)
```

```bash
# Run against a Windows Server 2022 host
pyinfra "@winrm/192.168.1.50" deploy.py \
  --winrm-username Administrator \
  --winrm-password "MyP@ssword" \
  --winrm-transport ntlm
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `WinRMTransportError: 401` | Wrong credentials or transport; try `ntlm` instead of `basic` |
| `ConnectionError: timed out` | Check firewall — ports 5985 (HTTP) or 5986 (HTTPS) must be open |
| `psrp errors` | Install `pywinrm[kerberos]` or `pywinrm[credssp]` as needed |
| Commands don't run as expected | WinRM runs in a non-interactive session; avoid console-interactive commands |
| File upload fails | Check destination path uses `\\` not `/`, and destination directory exists |
| `Enable-PSRemoting` fails | Must run as Administrator in an elevated PowerShell prompt |

### Quick connectivity check

```bash
# Test WinRM connection with curl
curl -v --ntlm --user "Administrator:password" \
  "http://192.168.1.50:5985/wsman"
```

If you get a 200 response, WinRM is reachable and auth is working.
