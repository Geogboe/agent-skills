# pyinfra SDK: Custom Facts and Operations

## Table of Contents
1. [Custom Facts](#custom-facts)
2. [Dynamic (Parameterised) Facts](#dynamic-parameterised-facts)
3. [Custom Operations](#custom-operations)
4. [Command Types](#command-types)
5. [Calling Other Operations from Within an Operation](#calling-other-operations)
6. [Idempotency Checklist](#idempotency-checklist)

---

## Custom Facts

A **fact** is a read-only query that returns information about host state. Custom facts
extend `FactBase` and define a shell command to run plus a function to parse its output.

```python
from pyinfra.api import FactBase

class PortListening(FactBase):
    """Returns True if a TCP port is listening on the host."""

    def command(self, port):
        # Shell command to run on the remote host
        return f"ss -tlnp | grep -c ':{port} ' || true"

    def process(self, output):
        # output is a list of lines from stdout
        count = int(output[0].strip()) if output else 0
        return count > 0

    # Value returned when command fails or produces no output
    default = False

    # Optional: assert a binary is available before running
    requires_command = "ss"
```

**Using the fact:**
```python
from pyinfra import host
from myproject.facts import PortListening

is_postgres_up = host.get_fact(PortListening, port=5432)

if is_postgres_up:
    server.shell(name="Run migrations", commands=["psql -f migrations.sql"])
```

### More fact examples

```python
class ServiceRunning(FactBase):
    """Returns True if a systemd service is active."""

    def command(self, service):
        return f"systemctl is-active {service} || true"

    def process(self, output):
        return output[0].strip() == "active" if output else False

    default = False


class FileChecksum(FactBase):
    """Returns the SHA256 checksum of a file, or None if not found."""

    def command(self, path):
        return f"sha256sum {path} 2>/dev/null | awk '{{print $1}}'"

    def process(self, output):
        return output[0].strip() if output and output[0].strip() else None

    default = None


class DatabaseExists(FactBase):
    """Returns True if a PostgreSQL database exists."""

    def command(self, dbname):
        return f"psql -lqt | cut -d '|' -f 1 | grep -qw '{dbname}' && echo yes || echo no"

    def process(self, output):
        return output[0].strip() == "yes" if output else False

    default = False
    requires_command = "psql"
```

### Fact without parameters

```python
class SwapEnabled(FactBase):
    """Returns True if any swap space is active."""

    command = "swapon --show --noheadings | wc -l"

    def process(self, output):
        return int(output[0].strip()) > 0 if output else False

    default = False
```

---

## Dynamic (Parameterised) Facts

When `command` is a method (not a string), the fact is **parameterised** — you pass
arguments when calling `host.get_fact(MyFact, arg=value)`.

The `command` method signature must match the keyword arguments you pass:

```python
class AppVersion(FactBase):
    """Returns the installed version of an application."""

    def command(self, app_name, version_flag="--version"):
        return f"{app_name} {version_flag} 2>&1 | head -1"

    def process(self, output):
        return output[0].strip() if output else None

    default = None
```

```python
node_ver = host.get_fact(AppVersion, app_name="node")
python_ver = host.get_fact(AppVersion, app_name="python3")
```

---

## Custom Operations

A **custom operation** is a Python function decorated with `@operation()` that yields
shell commands. Operations are the right abstraction when you have multi-step logic that
should appear as a single named step in pyinfra's output.

### Basic structure

```python
from pyinfra.api import operation
from pyinfra.api.command import StringCommand

@operation()
def deploy_app(state, host, app_dir, git_repo, branch="main"):
    """Clone or update the app and install dependencies."""
    # The function body runs locally (planning phase)
    # yield commands that will run on the remote host

    yield StringCommand(f"git -C {app_dir} pull origin {branch} || "
                        f"git clone -b {branch} {git_repo} {app_dir}")
    yield StringCommand(f"cd {app_dir} && pip install -r requirements.txt")
```

**Using the operation:**
```python
from myproject.operations import deploy_app

deploy_app(
    name="Deploy myapp",
    app_dir="/opt/myapp",
    git_repo="https://github.com/myorg/myapp.git",
    branch="production",
    _sudo=True,
    _sudo_user="deploy",
)
```

### Idempotent operation with state check

```python
from pyinfra.api import operation
from pyinfra.api.command import StringCommand
from myproject.facts import PortListening, DatabaseExists

@operation()
def run_migrations(state, host, db_name, migrations_dir="/opt/app/migrations"):
    """Run database migrations only if postgres is running and db exists."""

    postgres_up = host.get_fact(PortListening, port=5432)
    db_exists = host.get_fact(DatabaseExists, dbname=db_name)

    if not postgres_up:
        # Don't yield anything — operation becomes a no-op
        host.noop(f"Skipping migrations: postgres not running on port 5432")
        return

    if not db_exists:
        yield StringCommand(f"createdb {db_name}")

    yield StringCommand(f"psql -d {db_name} -f {migrations_dir}/latest.sql")
```

---

## Command Types

Operations yield different command types depending on what they need to do:

### StringCommand — run a shell command

```python
from pyinfra.api.command import StringCommand

yield StringCommand("systemctl restart nginx")
yield StringCommand("echo 'Hello' > /tmp/test")
```

### FileUploadCommand — upload a file

```python
from pyinfra.api.command import FileUploadCommand

yield FileUploadCommand(
    src_filename="./configs/myapp.conf",   # local path
    dest_filename="/etc/myapp/myapp.conf",  # remote path
)
```

### FileDownloadCommand — download a file from host

```python
from pyinfra.api.command import FileDownloadCommand

yield FileDownloadCommand(
    src_filename="/var/log/app.log",        # remote path
    dest_filename="./logs/app.log",          # local path
)
```

### FunctionCommand — run a local Python function mid-deploy

```python
from pyinfra.api.command import FunctionCommand

def process_output(state, host):
    # This runs locally between remote commands
    print(f"Processing results on {host.name}")
    return True

yield FunctionCommand(process_output)
```

### Combining command types in one operation

```python
@operation()
def deploy_config(state, host, config_src, config_dest):
    # Upload the file
    yield FileUploadCommand(src_filename=config_src, dest_filename=config_dest)

    # Run a local function to log what happened
    def log_deploy(state, host):
        print(f"Deployed config to {host.name}:{config_dest}")

    yield FunctionCommand(log_deploy)

    # Restart the service
    yield StringCommand("systemctl reload nginx")
```

---

## Calling Other Operations

To call a pyinfra built-in (or another custom operation) from within your operation,
use `._inner()`. This inlines the sub-operation's commands into your operation's output:

```python
from pyinfra.api import operation
from pyinfra.operations import files, systemd

@operation()
def deploy_nginx_site(state, host, site_name, config_src):
    """Deploy an nginx site config and reload."""

    # Call files.put as an inner operation — yields its commands inline
    yield from files.put._inner(
        src=config_src,
        dest=f"/etc/nginx/sites-available/{site_name}",
    )

    yield from files.link._inner(
        path=f"/etc/nginx/sites-enabled/{site_name}",
        target=f"/etc/nginx/sites-available/{site_name}",
    )

    yield from systemd.service._inner(
        service="nginx",
        reloaded=True,
    )
```

---

## Idempotency Checklist

Before finalising a custom operation, verify each of these:

- [ ] **Check state first** — use `host.get_fact()` to read current state before yielding
  commands that change state. Don't issue commands whose effect you haven't verified.

- [ ] **No-op when already in desired state** — if the host is already in the desired
  state, yield nothing. `host.noop("reason")` explicitly marks the op as "nothing to do".

- [ ] **Idempotent sub-commands** — each shell command you yield should be safe to run
  twice. Prefer `||` fallbacks:
  ```python
  yield StringCommand("id myuser || useradd myuser")
  ```

- [ ] **Conditional create** — check before creating:
  ```python
  if not host.get_fact(DatabaseExists, dbname="mydb"):
      yield StringCommand("createdb mydb")
  ```

- [ ] **Atomic writes** — write to a temp file, then move:
  ```python
  yield StringCommand("cat > /tmp/app.conf.tmp << 'EOF'\n...\nEOF")
  yield StringCommand("mv /tmp/app.conf.tmp /etc/app.conf")
  ```

- [ ] **Return value** — the `@operation()` decorator handles wrapping; your function
  should only `yield` commands or call `host.noop()`. Don't `return` commands.

- [ ] **Test with `--dry-run`** — dry-run mode won't execute commands but will print
  them. Make sure your fact checks handle the case where the host state is unknown.
