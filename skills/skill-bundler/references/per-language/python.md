# Python Recipe: Embedding and Installing Skills

This recipe covers Python CLIs using `click` or `typer`. Assets are bundled as package
data and accessed at runtime via `importlib.resources`.

---

## 1. Asset Tree Layout

```
acme/
  skills/
    __init__.py
    skills.py
    assets/
      acme-cli/
        SKILL.md
        VERSION
        references/
          bootstrap.md
          diagnose.md
      acme-debug/
        SKILL.md
        VERSION
        references/
```

---

## 2. Declare Assets as Package Data

In `pyproject.toml` (PEP 517 / setuptools):

```toml
[tool.setuptools.package-data]
"acme.skills" = [
    "assets/acme-cli/**",
    "assets/acme-debug/**",
]
```

Or with legacy `setup.cfg`:

```ini
[options.package_data]
acme.skills =
    assets/acme-cli/**
    assets/acme-debug/**
```

**Excluding evals from the package:**

```toml
[tool.setuptools.package-data]
"acme.skills" = [
    "assets/acme-cli/**",
    "assets/acme-debug/**",
    "!assets/acme-cli/evals/**",
    "!assets/acme-debug/evals/**",
]
```

> **Note:** setuptools exclude globs are not always reliable across versions. As a
> belt-and-suspenders measure, add a `MANIFEST.in` exclusion and ensure `evals/` is
> gitignored so it is never included in sdist by accident:

```
# MANIFEST.in
prune acme/skills/assets/acme-cli/evals
prune acme/skills/assets/acme-debug/evals
```

```gitignore
# Skill eval fixtures — local only
acme/skills/assets/**/evals/
```

---

## 3. Access Assets at Runtime

In `acme/skills/skills.py`:

```python
from __future__ import annotations

import os
import shutil
from importlib.resources import files
from pathlib import Path
from dataclasses import dataclass, field
from typing import Iterator


@dataclass
class SkillManifest:
    """Describes one bundled skill."""
    name: str
    package: str  # importlib.resources package path


def bundled() -> list[SkillManifest]:
    """Return all skills compiled into this package."""
    return [
        SkillManifest(name="acme-cli",   package="acme.skills.assets.acme-cli"),
        SkillManifest(name="acme-debug", package="acme.skills.assets.acme-debug"),
    ]


def canonical_dir(tool_name: str, skill_name: str) -> Path:
    """Return ~/.config/<tool>/skills/<skill> (XDG_CONFIG_HOME respected)."""
    base = os.environ.get("XDG_CONFIG_HOME")
    if not base:
        base = Path.home() / ".config"
    return Path(base) / tool_name / "skills" / skill_name


def _iter_package_files(package: str) -> Iterator[tuple[str, bytes]]:
    """Yield (relative_path, content) for every file in a package tree."""
    root = files(package)
    for item in root.iterdir():
        yield from _walk(item, "")


def _walk(item, prefix: str) -> Iterator[tuple[str, bytes]]:
    rel = f"{prefix}{item.name}" if prefix else item.name
    if item.is_dir():
        for child in item.iterdir():
            yield from _walk(child, rel + "/")
    else:
        yield rel, item.read_bytes()


def install_canonical(manifest: SkillManifest, force: bool = False) -> Path:
    """Extract embedded assets to the canonical directory."""
    dest = canonical_dir("acme", manifest.name)
    if dest.exists() and not force:
        return dest
    dest.mkdir(parents=True, exist_ok=True)
    for rel_path, content in _iter_package_files(manifest.package):
        target = dest / rel_path
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_bytes(content)
    return dest
```

---

## 4. Symlink (or Copy Fallback on Windows)

```python
import platform
import subprocess

_MARKER_NAME = ".acme-skill-source"


def link_at(canonical: Path, parent_dir: Path, force: bool = False) -> tuple[Path, bool]:
    """
    Create a symlink (or managed copy on Windows without privileges).

    Returns (target_path, is_copy).
    """
    target = parent_dir / canonical.name
    if force and target.exists():
        shutil.rmtree(target)

    parent_dir.mkdir(parents=True, exist_ok=True)

    if platform.system() != "Windows":
        target.symlink_to(canonical)
        return target, False

    # Windows: try symlink, fall back to copy
    try:
        target.symlink_to(canonical, target_is_directory=True)
        return target, False
    except OSError as exc:
        # ERROR_PRIVILEGE_NOT_HELD = 1314
        if getattr(exc, "winerror", None) != 1314:
            raise
    # Copy fallback
    shutil.copytree(canonical, target)
    (target / _MARKER_NAME).write_text(str(canonical), encoding="utf-8")
    return target, True


def remove_link_at(target: Path, canonical: Path, force: bool = False) -> None:
    """Safely remove a symlink or managed copy."""
    if not target.exists():
        return
    if target.is_symlink():
        if target.resolve() != canonical.resolve() and not force:
            raise RuntimeError(f"{target} points elsewhere; use --force")
        target.unlink()
        return
    # Managed copy
    marker = target / _MARKER_NAME
    if not marker.exists() and not force:
        raise RuntimeError(f"{target} has no ownership marker; use --force")
    if marker.exists():
        src = marker.read_text(encoding="utf-8").strip()
        if Path(src).resolve() != canonical.resolve() and not force:
            raise RuntimeError(f"Marker mismatch in {target}; use --force")
    shutil.rmtree(target)
```

---

## 5. Click / Typer Install Command

Using `click`:

```python
import click
from acme.skills import skills as skill_lib


@click.group()
def skills_cmd():
    """Manage bundled agent skills."""


@skills_cmd.command("install")
@click.argument("skill_name", required=False)
@click.option("--user/--no-user", default=True, show_default=True,
              help="Install into ~/.agents/skills/")
@click.option("--project", is_flag=True, help="Install into ./.agents/skills/")
@click.option("--path", "extra_paths", multiple=True,
              help="Additional target directory (repeatable)")
@click.option("--force", is_flag=True, help="Overwrite existing installation")
def install_cmd(skill_name, user, project, extra_paths, force):
    """Install bundled skill(s) into agent directories."""
    manifests = skill_lib.bundled()
    if skill_name:
        manifests = [m for m in manifests if m.name == skill_name]
        if not manifests:
            raise click.ClickException(f"Unknown skill: {skill_name}")

    for m in manifests:
        canonical = skill_lib.install_canonical(m, force=force)
        parents = _resolve_parents(user, project, list(extra_paths))
        for p in parents:
            path, is_copy = skill_lib.link_at(canonical, Path(p), force=force)
            kind = "copied" if is_copy else "linked"
            click.echo(f"{m.name}: {kind} → {path}")


def _resolve_parents(user: bool, project: bool, paths: list[str]) -> list[str]:
    if not user and not project and not paths:
        user = True  # default
    result = []
    if user:
        result.append(str(Path.home() / ".agents" / "skills"))
    if project:
        result.append(str(Path(".") / ".agents" / "skills"))
    result.extend(paths)
    return result
```

---

## 6. Tests

### Verify assets are bundled

```python
# tests/skills/test_skills.py
from importlib.resources import files
import pytest
from acme.skills.skills import bundled


def test_bundled_skills_have_skill_md():
    for manifest in bundled():
        root = files(manifest.package)
        skill_md = root / "SKILL.md"
        assert skill_md.is_file(), f"{manifest.name} is missing SKILL.md"


def test_evals_not_bundled():
    for manifest in bundled():
        root = files(manifest.package)
        evals_dir = root / "evals"
        assert not evals_dir.is_dir(), \
            f"{manifest.name}: evals/ must not be packaged into the distribution"
```

### Drift test → see [drift-and-help-all.md](../drift-and-help-all.md#python)

---

## 7. Package Note: Dotted Package Names with Hyphens

Python package paths use dots, but skill names use hyphens (e.g. `acme-cli`). The
directory `assets/acme-cli/` maps to an importlib traversal, not a dotted import.
Access it via `files("acme.skills").joinpath("assets/acme-cli")` instead of a dotted
package string if the hyphen causes import issues.

```python
from importlib.resources import files

root = files("acme.skills").joinpath("assets/acme-cli")
skill_md = root.joinpath("SKILL.md").read_text()
```

This is the safer cross-version approach (works in Python 3.9+).

---

## 8. See Also

- [Drift test and `help all` for Python](../drift-and-help-all.md#python)
- [Architecture: canonical + symlinks](../architecture.md)
