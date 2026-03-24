---
name: github-release-installers
description: Create production-ready install.sh and install.ps1 installers for CLI tools distributed via GitHub Releases. Use when asked to add one-command installation via curl/irm, including platform detection, latest-release lookup, checksum verification, install path selection, and PATH updates.
---

# GitHub Release Installers Skill

## What This Skill Is

This skill standardizes one-command installers for CLI binaries published on GitHub Releases.

It focuses on:
- `install.sh` for Linux/macOS (`curl | sh` flow)
- `install.ps1` for Windows (`irm | iex` flow)
- Safe download and checksum verification
- Cross-platform OS/arch mapping
- Practical install-dir and PATH handling

## Required Env Overrides

Every generated installer must support environment variable overrides for runtime behavior.

Required capabilities:
- Enable debug logging
- Override install directory
- Pin an explicit version (instead of latest)
- Force reinstall even if the binary already exists

Recommended variable pattern:
- Tool-specific: `<BINARY>_DEBUG`, `<BINARY>_INSTALL_DIR`, `<BINARY>_VERSION`, `<BINARY>_FORCE`
- Optional generic aliases: `INSTALLER_DEBUG`, `INSTALLER_INSTALL_DIR`, `INSTALLER_VERSION`, `INSTALLER_FORCE`

Example for binary `promptc`:
- `PROMPTC_DEBUG=1`
- `PROMPTC_INSTALL_DIR=/opt/bin`
- `PROMPTC_VERSION=v1.2.3`
- `PROMPTC_FORCE=1`

Behavior rules:
- Debug mode must increase script verbosity without changing install semantics
- Version override must bypass latest-release lookup
- Force mode must reinstall even when target binary already exists
- Install-dir override must take precedence over default directory selection
- After install, verify the binary path and confirm the install directory is in PATH
- If PATH is missing, print a clear, copy/paste-ready remediation prompt with the exact command for the user's shell profile

Create a `curl | sh` style installer (`install.sh`) and a PowerShell equivalent (`install.ps1`) so users can install a CLI tool in one command without manually downloading binaries.

---

## When to use this skill

- User wants `curl -fsSL .../install.sh | sh` style installation
- User asks for easy/one-line installation of a CLI tool
- Repo already uses GoReleaser (checksums.txt is already generated)
- Need to support Linux, macOS, and Windows

---

## Shell Script (`install.sh`)

### Pattern

```sh
#!/bin/sh
set -eu

REPO="owner/repo"
BINARY="mybinary"

die() { echo "error: $*" >&2; exit 1; }
```

**Key sections in order:**

1. **OS detection** — `uname -s`, map to `linux` / `darwin`
2. **Arch detection** — `uname -m`, map `x86_64` → `amd64`, `aarch64|arm64` → `arm64`
3. **Version selection** — use env override first, then fallback to GitHub latest release API:
   ```sh
    VERSION_VAR="${BINARY}_VERSION"
    TAG="${!VERSION_VAR:-${INSTALLER_VERSION:-}}"
    if [ -z "${TAG}" ]; then
       TAG="$(curl -fsSL "https://api.github.com/repos/${REPO}/releases/latest" \
          | grep '"tag_name"' \
          | sed 's/.*"tag_name": *"\([^"]*\)".*/\1/')"
    fi
   ```
4. **Archive URL** — match GoReleaser naming: `${BINARY}_${TAG}_${OS}_${ARCH}.tar.gz`
5. **Temp dir** — `mktemp -d` with `trap 'rm -rf "${TMP}"' EXIT`
6. **Download** — `curl -fsSL -o ...`
7. **Verify checksum** — detect `sha256sum` (Linux) vs `shasum -a 256` (macOS):
   ```sh
   (cd "${TMP}" && grep "${ARCHIVE}" checksums.txt | sha256sum --check --status)
   ```
8. **Extract** — `tar -xzf ... "${BINARY}"` (extract only the binary, not the whole archive)
9. **Install dir priority**:
   - `$<BINARY>_INSTALL_DIR` (or `$INSTALLER_INSTALL_DIR`) if set
   - `/usr/local/bin` if writable
   - `~/.local/bin` as fallback; warn if not in PATH
10. **Force reinstall behavior** — if `$<BINARY>_FORCE=1` (or `$INSTALLER_FORCE=1`), reinstall even when binary exists
11. **No `sudo`** — never bake in sudo; let the user handle permissions
12. **Post-install verification (required)**:
    - Confirm the binary exists at the destination path
    - Confirm the command resolves from PATH (`command -v "$BINARY"`)
    - If PATH is missing, show a user-friendly, shell-specific fix command
13. **Confirm** — run `"${BINARY}" --version`

### PATH remediation prompt (shell-specific)

When PATH does not include the install directory, output a clear prompt with the exact command the user should run for their shell.

Example prompt format:
```text
The binary was installed to /home/user/.local/bin, but that directory is not in your PATH yet.
Run the command below, then open a new shell (or source your profile) and retry:

<exact command>
```

Profile command examples:
- bash:
   `echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc`
- zsh:
   `echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc`
- fish:
   `set -U fish_user_paths $HOME/.local/bin $fish_user_paths`

### Checksum verification detail

GoReleaser's `checksums.txt` format is `<hash>  <filename>`. The `grep` + pipe to `sha256sum --check` pattern works because `sha256sum -c` reads that format:

```sh
grep "${ARCHIVE}" checksums.txt | sha256sum --check --status
```

macOS uses `shasum` instead:
```sh
grep "${ARCHIVE}" checksums.txt | shasum -a 256 --check --status
```

---

## PowerShell Script (`install.ps1`)

### Header

```powershell
#Requires -Version 5.1
$ErrorActionPreference = 'Stop'
```

**Key sections in order:**

1. **Arch detection** — `[System.Environment]::Is64BitOperatingSystem` → `amd64`; throw for 32-bit
2. **Version selection** — use env override first, then fallback to latest tag via `Invoke-RestMethod`:
   ```powershell
   $Prefix = $Binary.ToUpper().Replace('-', '_')
   $Tag = (Get-Item -Path "Env:${Prefix}_VERSION" -ErrorAction SilentlyContinue).Value
   if (-not $Tag) { $Tag = $env:INSTALLER_VERSION }
   if (-not $Tag) {
       $Release = Invoke-RestMethod "https://api.github.com/repos/$Repo/releases/latest"
       $Tag = $Release.tag_name
   }
   ```
3. **Archive URL** — Windows uses `.zip` not `.tar.gz`: `promptc_${Tag}_windows_amd64.zip`
4. **Temp dir** — `[System.IO.Path]::GetTempPath()` + `GetRandomFileName()`; wrap in `try/finally` for cleanup
5. **Download** — `Invoke-WebRequest -Uri ... -OutFile ... -UseBasicParsing`
6. **Verify SHA256**:
   ```powershell
   $Expected = (Get-Content $ChecksumsPath | Where-Object { $_ -match [regex]::Escape($Archive) }) -split '\s+' | Select-Object -First 1
   $Actual = (Get-FileHash -Path $ArchivePath -Algorithm SHA256).Hash.ToLower()
   if ($Actual -ne $Expected.ToLower()) { throw "Checksum mismatch!" }
   ```
7. **Extract** — `Expand-Archive -Path ... -DestinationPath ... -Force`
8. **Install dir** — `$env:<BINARY>_INSTALL_DIR` or `$env:INSTALLER_INSTALL_DIR` or fallback to `$env:USERPROFILE\.local\bin`
9. **Force reinstall behavior** — if `$env:<BINARY>_FORCE=1` (or `$env:INSTALLER_FORCE=1`), reinstall even when binary exists
10. **PATH management** — add to user PATH via `[Environment]::SetEnvironmentVariable("Path", ..., "User")`:
   ```powershell
   $UserPath = [Environment]::GetEnvironmentVariable('Path', 'User')
   if ($UserPath -notlike "*$InstallDir*") {
       [Environment]::SetEnvironmentVariable('Path', "$UserPath;$InstallDir", 'User')
       $env:PATH = "$env:PATH;$InstallDir"  # also update current session
   }
   ```
   - `"User"` scope writes to `HKCU\Environment` — no admin required
   - `"Machine"` scope requires admin — avoid unless explicitly needed
11. **Post-install verification (required)**:
   - Confirm the binary exists at destination
   - Confirm command resolution from PATH (`Get-Command <binary> -ErrorAction SilentlyContinue`)
   - If PATH is missing, print a user-friendly remediation message with exact profile update commands
12. **Confirm** — `& $Destination --version`

### PATH remediation prompt (PowerShell)

When PATH does not include the install directory, provide an explicit prompt with a copy/paste-ready command.

Example prompt format:
```text
The binary was installed to C:\Users\<user>\.local\bin, but that directory is not in your PATH yet.
Run this command, then restart your terminal:

[Environment]::SetEnvironmentVariable('Path', "$([Environment]::GetEnvironmentVariable('Path','User'));C:\Users\<user>\.local\bin", 'User')
```

---

## README update pattern

Replace manual download instructions with:

```markdown
## Installation

### Shell script (Linux / macOS) — Recommended
curl -fsSL https://raw.githubusercontent.com/OWNER/REPO/main/install.sh | sh

### PowerShell (Windows)
irm https://raw.githubusercontent.com/OWNER/REPO/main/install.ps1 | iex

### go install (requires Go 1.21+) (if using Go)
go install github.com/OWNER/REPO/cmd/BINARY@latest

### Manual download
https://github.com/OWNER/REPO/releases/latest
```

---

## GoReleaser compatibility

These scripts assume the standard GoReleaser archive naming convention:
- `{binary}_{tag}_{os}_{arch}.tar.gz` for Linux/macOS
- `{binary}_{tag}_{os}_{arch}.zip` for Windows

Verify your `goreleaser.yaml` uses these patterns. If GoReleaser uses `name_template`, check it matches. The `checksums.txt` file is generated automatically by GoReleaser.

---

## Common mistakes to avoid

- **Never hardcode a version** — always fetch latest from GitHub API so the script works for all future releases
- **Don't omit env overrides** — debug, install dir, version, and force reinstall must always be configurable
- **Never use `sudo`** — use install dir fallback instead
- **Don't skip checksum verification** — it's one extra curl + grep, and prevents supply chain attacks
- **Test the fallback PATH case** — run in a shell where `/usr/local/bin` is read-only to verify `~/.local/bin` fallback works
