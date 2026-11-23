---
id: vol-ttl-locator
type: script-volume
created: 2026-11-16
last-touched: 2026-11-16
source-of: 90-Meta/scripts/ttl_locator.py
fence: "````"
eol: crlf
sha256: 13bb5646e2f77b391d0d23e4c11e229bae11d8af1647690e6abe089dcf675522
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""Shared vault-location resolution for the legacy TTL script entry points."""
from __future__ import annotations

VERSION = "0.1.0"

import os
import sys
from dataclasses import dataclass


class VaultResolutionError(RuntimeError):
    """Raised when no usable vault location can be resolved."""


@dataclass(frozen=True)
class VaultLocation:
    root: str
    source: str


def _is_vault_root(path: str) -> bool:
    return (
        os.path.isfile(os.path.join(path, "00-ROOT.md"))
        and os.path.isdir(os.path.join(path, "01-Specs"))
        and os.path.isfile(os.path.join(path, "90-Meta", "schema.yaml"))
    )


def _config_path() -> str:
    explicit = os.environ.get("TTL_CONFIG")
    if explicit:
        return os.path.abspath(os.path.expanduser(explicit))
    appdata = os.environ.get("APPDATA")
    if appdata:
        return os.path.join(appdata, "ttl", "config.toml")
    xdg = os.environ.get("XDG_CONFIG_HOME")
    if xdg:
        return os.path.join(xdg, "ttl", "config.toml")
    return os.path.join(os.path.expanduser("~"), ".config", "ttl", "config.toml")


def _configured_vault() -> str | None:
    path = _config_path()
    if not os.path.isfile(path):
        return None
    try:
        import tomllib
        with open(path, "rb") as handle:
            raw = handle.read()
        data = tomllib.loads(raw.decode("utf-8-sig"))
    except (OSError, ValueError, ImportError) as exc:
        raise VaultResolutionError("cannot read TTL config %s: %s" % (path, exc))
    value = data.get("vault")
    if value is None and isinstance(data.get("ttl"), dict):
        value = data["ttl"].get("vault")
    if not isinstance(value, str) or not value.strip():
        raise VaultResolutionError(
            "TTL config %s must define vault = \"...\"" % path
        )
    if not os.path.isabs(value):
        value = os.path.join(os.path.dirname(path), value)
    return os.path.abspath(os.path.expanduser(value))


def _discover_from(start: str) -> str | None:
    current = os.path.abspath(start)
    if not os.path.isdir(current):
        current = os.path.dirname(current)
    while True:
        if (
            os.path.exists(os.path.join(current, ".ttl-vault"))
            and _is_vault_root(current)
        ) or _is_vault_root(current):
            return current
        parent = os.path.dirname(current)
        if parent == current:
            return None
        current = parent


def resolve_vault(
    explicit: str | None = None,
    *,
    script_fallback: str | None = None,
    start: str | None = None,
) -> VaultLocation:
    """Resolve a vault without depending on the process cwd or install layout.

    Explicit input remains authoritative. The legacy script-location fallback is
    retained only for the current in-vault entry points and is reported as such.
    """
    if explicit:
        return VaultLocation(os.path.abspath(os.path.expanduser(explicit)), "--vault")

    env = os.environ.get("TTL_VAULT")
    if env:
        return VaultLocation(os.path.abspath(os.path.expanduser(env)), "$TTL_VAULT")

    configured = _configured_vault()
    if configured:
        return VaultLocation(configured, "TTL config %s" % _config_path())

    discovered = _discover_from(start or os.getcwd())
    if discovered:
        return VaultLocation(discovered, "vault marker/discovery")

    if script_fallback:
        legacy = os.path.abspath(script_fallback)
        if _is_vault_root(legacy):
            return VaultLocation(legacy, "legacy script location ../..")

    raise VaultResolutionError(
        "vault not found; use --vault, TTL_VAULT, TTL config, or run inside a vault"
    )


def self_test() -> int:
    import tempfile
    import shutil

    tmp = tempfile.mkdtemp(prefix="ttl_locator_selftest.")
    try:
        vault = os.path.join(tmp, "memory")
        os.makedirs(os.path.join(vault, "01-Specs"))
        os.makedirs(os.path.join(vault, "90-Meta"))
        for rel in ("00-ROOT.md", "90-Meta/schema.yaml"):
            with open(os.path.join(vault, rel), "w", encoding="utf-8") as handle:
                handle.write("")
        nested = os.path.join(vault, "nested", "work")
        os.makedirs(nested)
        old = os.environ.get("TTL_VAULT")
        os.environ["TTL_VAULT"] = vault
        try:
            assert resolve_vault(start=nested).root == vault
            assert resolve_vault(start=nested).source == "$TTL_VAULT"
        finally:
            if old is None:
                os.environ.pop("TTL_VAULT", None)
            else:
                os.environ["TTL_VAULT"] = old
        assert resolve_vault(explicit=vault).root == vault
        try:
            resolve_vault(start=tmp, script_fallback=os.path.join(tmp, "not-a-vault"))
        except VaultResolutionError:
            pass
        else:
            raise AssertionError("invalid legacy fallback must fail explicitly")
        return 0
    except AssertionError:
        return 1
    finally:
        shutil.rmtree(tmp, ignore_errors=True)


if __name__ == "__main__":
    sys.exit(self_test())
````
