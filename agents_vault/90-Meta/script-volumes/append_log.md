---
id: vol-append-log
type: script-volume
created: 2026-11-16
last-touched: 2026-11-16
source-of: 90-Meta/scripts/append_log.py
fence: "````"
eol: crlf
depends-on: [vol-ttl-locator]
sha256: 86f0956f807361a13549f15a4f1bf8410a2edd01f2e08e616c7c585da7fd1822
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""append_log.py -- guarded L1 append / frontmatter backfill for the TTL vault.

Batch 4 (write closure). Offline-first, pure standard library, no network by
default. This is the "append-only guard" for session-end L1 recording, so that
interrupt / duplicate / drift / App-not-running never silently corrupt history
(SPEC-WRITE 记入协议 + 安全 B; community report P2 acceptance).

Modes
-----
  new-log   Safely append a NEW L1 log file. Guards: the log id must not already
            exist in 04-Logs, the target path must not exist (append-only: never
            overwrite history), and the content's frontmatter id must match --id.
  backfill  Append one derived-pointer value to an EXISTING log's frontmatter
            list field (reflect-to / corrections). The body is left
            byte-identical; the operation is idempotent (no duplicate value).

Guards
------
  dedup           new id must be unique across 04-Logs; target file must be new.
  append-only     backfill rewrites ONLY the frontmatter field; the body hash is
                  verified unchanged before/after.
  drift (ifMatch) offline content-hash token mirrors the Local REST API
                  `ifMatch`/`version` optimistic-concurrency model. Pass
                  --expect-hash <sha256>; if the file changed since it was read,
                  the write aborts with exit 3 and the file is untouched.
  dry-run default nothing is written unless --apply; a unified diff is printed.

Channels (--channel)
--------------------
  offline           atomic file write (temp + os.replace). Fully self-tested;
                    the byte-exact target the cli channel downgrades to.
  cli (default)     official `obsidian` CLI -- the first-party write path (D23).
                    Invoked via the RESOLVED FULL-PATH launcher (Obsidian.com) +
                    argv list (never the bare name: Windows CreateProcess would
                    pick the 225MB GUI Obsidian.exe, which returns rc=-1). The
                    App re-indexes on write. new-log -> `create`; backfill ->
                    `create overwrite` (byte-exact whole-file rewrite). Content
                    holding a literal backslash-n/-t the CLI would mangle, or an
                    unreachable App / wrong active vault, DOWNGRADES explicitly to
                    offline. Needs the App running; NOT exercised by --self-test.
  rest              OPTIONAL Local REST API adapter. The plugin is third-party
                    and is installed/enabled by the USER; TTL never auto-installs
                    or auto-executes it. Reached only via --channel rest plus
                    OLR_API_URL / OLR_API_KEY. Native ifMatch via the
                    document-map `version` token. NOT exercised by --self-test.
                    Treat as experimental: always --dry-run against the running
                    plugin before --apply.

Exit codes: 0 ok / 2 tool error / 3 guard rejection (dedup / drift / append-only).
"""
from __future__ import annotations

import argparse
import difflib
import hashlib
import io
import os
import re
import sys
import tempfile
from ttl_locator import resolve_vault   # depends-on: vol-ttl-locator

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "0.3.0"
SCHEMA_VER = "0.7"
FM_FENCE = "---"
# Frontmatter list fields that are pure derived pointers (append-only backfill
# targets). Body prose and descriptive fields are NOT backfillable here.
DERIVED_LIST_FIELDS = {"reflect-to", "corrections", "superseded-by"}


# --------------------------------------------------------------------------- #
# vault location + small text helpers
# --------------------------------------------------------------------------- #
def find_vault(explicit: str | None) -> str:
    """Locate the vault through the shared resolver."""
    here = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", ".."))
    return resolve_vault(explicit, script_fallback=here).root


def read_plugin_rest_config(vault: str):
    """Resolve (base_url, api_key) from the Local REST API plugin's own data.json.

    The key is read in-memory only -- never printed, logged, or persisted, and
    data.json is gitignored (see .gitignore: agents_vault/.obsidian/plugins/). Field
    names mirror the plugin's LocalRestApiSettings: apiKey / port (https, default
    27124) / insecurePort (http, default 27123) / enableInsecureServer.
    """
    import json
    cfg = os.path.join(vault, ".obsidian", "plugins",
                       "obsidian-local-rest-api", "data.json")
    if not os.path.exists(cfg):
        raise GuardError("plugin config not found at %s -- enable 'Local REST API "
                         "with MCP' in Obsidian first (it then generates the key)"
                         % cfg.replace("\\", "/"))
    try:
        with io.open(cfg, "r", encoding="utf-8") as f:
            data = json.load(f)
    except (OSError, ValueError) as e:
        raise GuardError("could not read plugin config: %s" % e)
    key = data.get("apiKey")
    if not key:
        raise GuardError("plugin data.json has no apiKey -- open the plugin "
                         "settings once so it generates a key")
    if data.get("enableInsecureServer"):
        base = "http://127.0.0.1:%d" % int(data.get("insecurePort", 27123))
    else:
        base = "https://127.0.0.1:%d" % int(data.get("port", 27124))
    return base, key


def sha256_text(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()


def read_file(path: str) -> str:
    with io.open(path, "r", encoding="utf-8") as f:
        return f.read()


def split_frontmatter(text: str):
    """Split a note into (frontmatter_lines, body_text, has_fm).

    frontmatter_lines excludes the two '---' fences. body_text is everything
    after the closing fence (kept byte-for-byte so append-only can be verified).
    """
    lines = text.split("\n")
    if not lines or lines[0].strip() != FM_FENCE:
        return [], text, False
    for j in range(1, len(lines)):
        if lines[j].strip() == FM_FENCE:
            return lines[1:j], "\n".join(lines[j + 1:]), True
    return [], text, False


def fm_get(fm_lines: list, key: str):
    """Return the raw value string of a top-level frontmatter key, or None."""
    pat = re.compile(r"^%s:\s*(.*)$" % re.escape(key))
    for ln in fm_lines:
        m = pat.match(ln)
        if m:
            return m.group(1).strip()
    return None


def parse_inline_list(raw: str) -> list:
    """Parse an inline list value like '["a", "b"]' or '[]' into ['a','b'].

    Only the inline bracketed form is supported (what the schema/logs use);
    a bare scalar is returned as a single-item list, empty string as [].
    """
    raw = (raw or "").strip()
    if not raw:
        return []
    if raw.startswith("[") and raw.endswith("]"):
        inner = raw[1:-1].strip()
        if not inner:
            return []
        items = []
        for part in inner.split(","):
            part = part.strip().strip('"').strip("'")
            if part:
                items.append(part)
        return items
    return [raw.strip('"').strip("'")]


def serialize_inline_list(items: list) -> str:
    return "[" + ", ".join('"%s"' % it for it in items) + "]"


# --------------------------------------------------------------------------- #
# offline atomic write
# --------------------------------------------------------------------------- #
def atomic_write(path: str, text: str) -> None:
    """Write text to path atomically (temp file in same dir + os.replace)."""
    d = os.path.dirname(os.path.abspath(path)) or "."
    os.makedirs(d, exist_ok=True)
    fd, tmp = tempfile.mkstemp(prefix=".append_log.", suffix=".tmp", dir=d)
    try:
        with os.fdopen(fd, "w", encoding="utf-8", newline="\n") as f:
            f.write(text)
        os.replace(tmp, path)
    except BaseException:
        if os.path.exists(tmp):
            os.unlink(tmp)
        raise


def show_diff(old: str, new: str, path: str) -> str:
    old_lines = old.splitlines(keepends=True)
    new_lines = new.splitlines(keepends=True)
    return "".join(difflib.unified_diff(
        old_lines, new_lines, fromfile="a/" + path, tofile="b/" + path))


# --------------------------------------------------------------------------- #
# dedup index (offline)
# --------------------------------------------------------------------------- #
def existing_log_ids(vault: str) -> set:
    """Collect all L1 ids under 04-Logs (logs/ + corrections/ + legacy flat root).

    Nested split: ids must stay unique across BOTH subdirs, so dedup walks the
    whole 04-Logs tree recursively (also covers the pre-split flat layout).
    """
    ids = set()
    base = os.path.join(vault, "04-Logs")
    if not os.path.isdir(base):
        return ids
    for dp, dn, fn in os.walk(base):
        for name in fn:
            if not name.endswith(".md"):
                continue
            try:
                text = read_file(os.path.join(dp, name))
            except OSError:
                continue
            fm, _, has = split_frontmatter(text)
            if has:
                v = fm_get(fm, "id")
                if v:
                    ids.add(v.strip('"').strip("'"))
    return ids


# --------------------------------------------------------------------------- #
# core guarded operations (channel-agnostic decision, then a channel write)
# --------------------------------------------------------------------------- #
def plan_new_log(vault: str, rel_path: str, log_id: str, content: str):
    """Validate a new-log append; return (new_text, report) or raise GuardError."""
    if not rel_path.startswith("04-Logs/"):
        raise GuardError("new-log target must live under 04-Logs/ (got %s)" % rel_path)
    abs_path = os.path.join(vault, rel_path.replace("/", os.sep))
    if os.path.exists(abs_path):
        raise GuardError("target file already exists -- refusing to overwrite "
                         "history (append-only): %s" % rel_path)
    if log_id in existing_log_ids(vault):
        raise GuardError("log id %r already exists in 04-Logs (duplicate)" % log_id)
    fm, _, has = split_frontmatter(content)
    if not has:
        raise GuardError("content has no frontmatter block")
    cid = (fm_get(fm, "id") or "").strip('"').strip("'")
    if cid != log_id:
        raise GuardError("content frontmatter id %r != --id %r" % (cid, log_id))
    report = "new-log %s -> %s (%d bytes)" % (log_id, rel_path, len(content.encode("utf-8")))
    return content, report


def plan_backfill(vault: str, rel_path: str, field: str, value: str, expect_hash: str | None):
    """Validate a frontmatter backfill; return (old, new, report) or raise."""
    if field not in DERIVED_LIST_FIELDS:
        raise GuardError("field %r is not an append-only derived pointer "
                         "(allowed: %s)" % (field, ", ".join(sorted(DERIVED_LIST_FIELDS))))
    abs_path = os.path.join(vault, rel_path.replace("/", os.sep))
    if not os.path.exists(abs_path):
        raise GuardError("target file does not exist: %s" % rel_path)
    old = read_file(abs_path)
    cur_hash = sha256_text(old)
    if expect_hash and expect_hash != cur_hash:
        raise GuardError("drift detected (ifMatch): expected %s but file is %s -- "
                         "refetch and retry" % (expect_hash[:12], cur_hash[:12]))
    fm, body, has = split_frontmatter(old)
    if not has:
        raise GuardError("target has no frontmatter block: %s" % rel_path)
    raw = fm_get(fm, field)
    # Block-style YAML guard: our rebuild writes inline lists and replaces only
    # the "field:" line. If the field is block-style ("field:" then indented
    # "- item" lines), replacing just that line orphans the items = silent data
    # loss, so refuse loudly. Obsidian's UI editor and the Local REST API plugin
    # both reformat frontmatter to block style; TTL's convention is inline.
    if raw is not None and raw.strip() == "":
        fld_pat = re.compile(r"^%s:\s*$" % re.escape(field))
        for i, ln in enumerate(fm):
            if fld_pat.match(ln):
                nxt = fm[i + 1] if i + 1 < len(fm) else ""
                if re.match(r"^\s+-\s", nxt):
                    raise GuardError(
                        "block-style YAML for %r in %s: its values sit on following "
                        "'- ' lines, which this inline-writing tool would orphan "
                        "(silent data loss). Reformat that field to inline style "
                        "first, e.g.  %s: [\"a\", \"b\"]" % (field, rel_path, field))
                break
    items = parse_inline_list(raw) if raw is not None else []
    if value in items:
        return old, old, "backfill no-op: %s already in %s of %s" % (value, field, rel_path)
    items.append(value)
    new_line = "%s: %s" % (field, serialize_inline_list(items))
    pat = re.compile(r"^%s:\s*.*$" % re.escape(field))
    replaced = False
    for i, ln in enumerate(fm):
        if pat.match(ln):
            fm[i] = new_line
            replaced = True
            break
    if not replaced:
        fm.append(new_line)  # field absent -> add it (still frontmatter-only)
    new_text = FM_FENCE + "\n" + "\n".join(fm) + "\n" + FM_FENCE + "\n" + body
    # append-only invariant: body must be byte-identical
    _, new_body, _ = split_frontmatter(new_text)
    if new_body != body:
        raise GuardError("internal: backfill altered body -- aborting (append-only)")
    report = "backfill %s += %s in %s" % (field, value, rel_path)
    return old, new_text, report


class GuardError(Exception):
    """A guard rejected the write (dedup / drift / append-only / bad target)."""


# --------------------------------------------------------------------------- #
# channels
# --------------------------------------------------------------------------- #
def write_offline(vault: str, rel_path: str, old: str, new: str, apply: bool) -> None:
    abs_path = os.path.join(vault, rel_path.replace("/", os.sep))
    if not apply:
        sys.stdout.write(show_diff(old, new, rel_path) or "(no textual change)\n")
        return
    atomic_write(abs_path, new)


def _resolve_obsidian():
    """Resolve the obsidian CLI launcher to a FULL PATH (or None if absent).

    Must not pass the bare name `obsidian` to subprocess: on Windows CreateProcess
    appends `.exe` first and launches the 225MB GUI Obsidian.exe, which returns
    rc=-1 for colon+param commands (property:set, search:context). The real CLI is
    the small console launcher Obsidian.com, which shutil.which picks via PATHEXT
    (.COM before .EXE). If PATH resolution somehow landed on .exe, prefer a .com
    sibling. See PIT-obsidian裸名误调GUI版.
    """
    import shutil
    p = shutil.which("obsidian")
    if not p:
        return None
    if p.lower().endswith(".exe"):
        com = p[:-4] + ".com"
        if os.path.exists(com):
            return com
    return p


def _active_vault_matches(obs: str, vault: str):
    """Return (ok, info): ok True when the App's ACTIVE vault is our target vault.

    The CLI writes to whatever vault is active in the running App; a mismatch (or
    an unreachable App) means the CLI write path is unavailable, so the caller
    downgrades explicitly to offline. info carries the active path or the reason.
    """
    import subprocess
    try:
        r = subprocess.run([obs, "vault", "info=path"], capture_output=True,
                           text=True, encoding="utf-8", timeout=20)
    except Exception as e:  # noqa: BLE001 - any failure means "CLI unavailable"
        return False, "vault query failed: %s" % e
    active = (r.stdout or "").strip()
    if r.returncode != 0 or not active:
        return False, "App not reachable (vault info rc=%s)" % r.returncode
    if os.path.normcase(os.path.normpath(active)) != os.path.normcase(os.path.normpath(vault)):
        return False, "active vault %r != target %r" % (active, vault)
    return True, active


# Hard cap on payload size for the CLI channel. The CLI->App IPC serialises the
# whole `content=` argument into one JSON message over a socket; a large and/or
# backslash-laden payload was observed to corrupt that JSON and raise an
# Uncaught Exception in the App main process (log-2026-09-29-02). The true safe
# limit is UNKNOWN and must NOT be probed against a live App, so anything above
# this conservative cap downgrades to the offline channel.
CLI_MAX_BYTES = 2048


def _cli_content_is_safe(text: str) -> bool:
    """True if `text` is safe to hand to the CLI `content=` argument.

    Rejects ANY literal backslash. The CLI unescapes `\\n`/`\\t` naively (there
    is no way to represent a genuine backslash-n/-t), and a lone backslash (e.g.
    a Windows path `..\\x.md`) was observed to corrupt the CLI->App IPC JSON and
    crash the App. Rejecting all backslashes (not just backslash-n/-t) is the
    conservative gate. Real newlines/tabs are fine: _escape_cli_content
    re-encodes them. Payload size is gated separately against CLI_MAX_BYTES.
    """
    return "\\" not in text


def _escape_cli_content(text: str) -> str:
    """Encode real newlines/tabs as the literal \\n / \\t the CLI expects.

    Verified byte-exact on a realistic multi-line frontmatter+body (round 8).
    Callers MUST gate this behind _cli_content_is_safe (a pre-existing literal
    backslash-n/-t cannot be represented and would corrupt).
    """
    return text.replace("\n", "\\n").replace("\t", "\\t")


def write_cli(vault: str, rel_path: str, old: str, new: str, apply: bool) -> None:
    """Official `obsidian` CLI channel (first-party write path, D23).

    Writes flow through the running App so its metadataCache / link graph re-index
    immediately. new-log (old == "") -> `create` (no overwrite: never clobber
    history); backfill (old != "") -> `create overwrite` with the guarded new_text
    (the body is byte-identical, so overwrite does not break the append-only
    invariant -- it is the App-mediated analogue of the offline atomic write).

    Byte-exactness: the write DOWNGRADES explicitly to the offline channel (never
    silently) when the CLI is unavailable -- launcher missing, App unreachable, a
    different active vault, or content holding any backslash / exceeding
    CLI_MAX_BYTES (the CLI->App IPC JSON corrupts and can crash the App). After
    a CLI write the file is read back and compared to `new`;
    any mismatch raises (never trust an unverified write). Not exercised by
    --self-test (offline only).
    """
    is_new = (old == "")
    if not apply:
        verb = "create" + ("" if is_new else " overwrite")
        sys.stdout.write("[cli dry-run] would run: obsidian %s path=%s content=<%d bytes>\n"
                         % (verb, rel_path, len(new.encode("utf-8"))))
        sys.stdout.write(show_diff(old, new, rel_path) or "(no textual change)\n")
        return
    # pre-flight: is the CLI path usable? if not, EXPLICIT downgrade to offline
    reason = None
    obs = _resolve_obsidian()
    nbytes = len(new.encode("utf-8"))
    if not _cli_content_is_safe(new):
        reason = "content holds a backslash the CLI/IPC would mangle"
    elif nbytes > CLI_MAX_BYTES:
        reason = ("content %d bytes > CLI_MAX_BYTES %d (large payloads corrupt "
                  "the CLI->App IPC JSON)" % (nbytes, CLI_MAX_BYTES))
    elif obs is None:
        reason = "obsidian CLI not found on PATH"
    else:
        ok, info = _active_vault_matches(obs, vault)
        if not ok:
            reason = info
    if reason is not None:
        sys.stdout.write("[cli] %s -- downgrading to the offline atomic write\n" % reason)
        write_offline(vault, rel_path, old, new, apply=True)
        return
    import subprocess
    args = [obs, "create", "path=" + rel_path,
            "content=" + _escape_cli_content(new)]
    if not is_new:
        args.append("overwrite")
    r = subprocess.run(args, capture_output=True, text=True,
                       encoding="utf-8", cwd=vault)
    if r.returncode != 0:
        raise GuardError("obsidian create failed (rc=%s): %s"
                         % (r.returncode, (r.stderr or r.stdout or "").strip()[:200]))
    abs_path = os.path.join(vault, rel_path.replace("/", os.sep))
    got = read_file(abs_path) if os.path.exists(abs_path) else None
    if got != new:
        raise GuardError("cli write verification failed: on-disk bytes != intended "
                         "(use --channel offline for a byte-exact write)")


def write_rest(vault: str, rel_path: str, old: str, new: str, apply: bool,
               rest_op: dict | None = None) -> None:
    """OPTIONAL Local REST API adapter -- READ-ONLY (third-party plugin).

    Dry-run only: GET the document map to verify connectivity + surface its
    content-hash `version` token (the ifMatch drift reference). --apply is
    DISABLED. Verified finding (2026-09-28): the plugin's frontmatter PATCH
    re-serializes the WHOLE YAML block -- inline lists ["a","b"] become block
    lists (- a / - b) and quoted scalars lose their quotes. That breaks our
    inline-list convention and our offline parse_inline_list (which would then
    read e.g. reflect-to as EMPTY = silent data loss), and churns git history.
    So all writes go through --channel offline (byte-exact, self-tested). TTL
    never auto-installs or auto-executes the plugin, and issues only GET here --
    never POST/PATCH/delete/move/command_execute/eval.
    """
    import json
    import ssl
    import urllib.error
    import urllib.parse
    import urllib.request
    base = os.environ.get("OLR_API_URL")
    key = os.environ.get("OLR_API_KEY")
    if not base or not key:
        raise GuardError("rest channel requires OLR_API_URL and OLR_API_KEY "
                         "(user-provided; TTL does not auto-install the plugin)")
    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE  # plugin serves a local self-signed cert
    # keep '/' as real path separators; percent-encode any non-ASCII segment
    url = base.rstrip("/") + "/vault/" + urllib.parse.quote(rel_path)

    def req(method, headers=None, data=None):
        h = {"Authorization": "Bearer " + key}
        if headers:
            h.update(headers)
        r = urllib.request.Request(url, data=data, headers=h, method=method)
        with urllib.request.urlopen(r, context=ctx, timeout=10) as resp:
            return resp.status, resp.read().decode("utf-8")

    def get_version():
        st, body = req("GET", {"Accept": "application/vnd.olrapi.document-map+json"})
        if st != 200:
            raise GuardError("rest GET document-map returned status %s" % st)
        try:
            return json.loads(body).get("version", "")
        except ValueError:
            raise GuardError("rest document-map response is not valid JSON")

    try:
        if not apply:
            # read-only: verify connectivity + surface the drift token; no write
            ver = get_version()
            sys.stdout.write("[rest dry-run] GET document-map status=200 version=%s\n" % ver)
            sys.stdout.write(show_diff(old, new, rel_path) or "(no textual change)\n")
            return
        # --apply is intentionally refused on the rest channel (see docstring):
        # the plugin's frontmatter PATCH re-serializes YAML (inline->block, strips
        # quotes), which silently breaks our inline-list convention + offline
        # parser. Use --channel offline for byte-exact writes.
        raise GuardError(
            "rest --apply is disabled: the plugin's frontmatter PATCH re-serializes "
            "YAML (inline->block, strips quotes), breaking our inline-list convention "
            "and offline parser (silent data loss). Use --channel offline for writes; "
            "rest is read-only (dry-run GET connectivity + version token).")
    except urllib.error.HTTPError as e:
        if e.code == 412:
            raise GuardError("rest drift (If-Match 412): file changed -- refetch and retry")
        detail = e.read().decode("utf-8", "replace")
        raise GuardError("rest HTTP %s: %s" % (e.code, detail))
    except GuardError:
        raise
    except Exception as e:  # noqa: BLE001 - surface, never crash silently
        raise GuardError("rest call failed: %s" % e)


def dispatch_write(channel, vault, rel_path, old, new, apply, rest_op=None):
    if channel == "offline":
        write_offline(vault, rel_path, old, new, apply)
    elif channel == "cli":
        write_cli(vault, rel_path, old, new, apply)
    elif channel == "rest":
        write_rest(vault, rel_path, old, new, apply, rest_op)
    else:
        raise GuardError("unknown channel %r" % channel)


# --------------------------------------------------------------------------- #
# CLI
# --------------------------------------------------------------------------- #
def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(
        description="Guarded L1 append / frontmatter backfill for the TTL vault.")
    p.add_argument("--vault", default=None, help="vault root (default: $TTL_VAULT or ../..)")
    p.add_argument("--channel", default="cli", choices=["offline", "cli", "rest"])
    p.add_argument("--apply", action="store_true", help="actually write (default is dry-run)")
    p.add_argument("--key-from-plugin", action="store_true",
                   help="rest channel: read apiKey in-memory from the Local REST API "
                        "plugin's data.json instead of env (never printed/persisted)")
    sub = p.add_subparsers(dest="mode", required=True)

    nl = sub.add_parser("new-log", help="safely append a NEW L1 log file")
    nl.add_argument("--id", required=True, dest="log_id")
    nl.add_argument("--path", required=True, help="vault-relative path under 04-Logs/")
    nl.add_argument("--content-file", default=None,
                    help="file holding the full prepared log markdown (default: stdin)")

    bf = sub.add_parser("backfill", help="append a derived-pointer to a log's frontmatter list")
    bf.add_argument("--path", required=True, help="vault-relative path of the existing log")
    bf.add_argument("--field", required=True, help="reflect-to | corrections | superseded-by")
    bf.add_argument("--value", required=True, help="the pointer value to append")
    bf.add_argument("--expect-hash", default=None,
                    help="sha256 of the file as read; aborts on drift (ifMatch)")
    return p


def main(argv=None) -> int:
    args = build_parser().parse_args(argv)
    vault = find_vault(args.vault)
    if not os.path.isdir(vault):
        sys.stdout.write("vault not found: %s\n" % vault)
        return 2
    try:
        if getattr(args, "key_from_plugin", False):
            # resolve REST base+key from the plugin config into the process env
            # (in-memory only; write_rest reads OLR_API_URL/OLR_API_KEY, never prints)
            _base, _key = read_plugin_rest_config(vault)
            os.environ["OLR_API_URL"] = _base
            os.environ["OLR_API_KEY"] = _key
        if args.mode == "new-log":
            if args.content_file:
                content = read_file(args.content_file)
            else:
                content = sys.stdin.read()
            if not content.endswith("\n"):
                content += "\n"
            new_text, report = plan_new_log(vault, args.path, args.log_id, content)
            rest_op = {"kind": "new-log", "content": new_text}
            dispatch_write(args.channel, vault, args.path, "", new_text, args.apply, rest_op)
        else:  # backfill
            old, new_text, report = plan_backfill(
                vault, args.path, args.field, args.value, args.expect_hash)
            rest_op = {"kind": "backfill", "field": args.field, "value": args.value}
            dispatch_write(args.channel, vault, args.path, old, new_text, args.apply, rest_op)
    except GuardError as e:
        sys.stdout.write("GUARD REJECTED: %s\n" % e)
        return 3
    except Exception as e:  # noqa: BLE001 - tool error
        sys.stdout.write("ERROR: %s\n" % e)
        return 2
    verb = "APPLIED" if args.apply else "DRY-RUN"
    sys.stdout.write("%s [%s] %s\n" % (verb, args.channel, report))
    return 0


# --------------------------------------------------------------------------- #
# self-test (offline only; pure stdlib, no network, no App)
# --------------------------------------------------------------------------- #
def _self_test() -> int:
    import shutil
    tmp = tempfile.mkdtemp(prefix="append_log_selftest.")
    failures = []
    try:
        logs = os.path.join(tmp, "04-Logs")
        os.makedirs(logs)
        seed = ('---\nid: "log-2026-00001"\nproject: memory-lib\n'
                'reflect-to: ["c-2026-00006"]\n---\n\n# Body\n\noriginal prose\n')
        with io.open(os.path.join(logs, "log-2026-00001.md"), "w", encoding="utf-8") as f:
            f.write(seed)

        # 1. new-log success (dry-run writes nothing)
        newc = '---\nid: "log-2026-00002"\nproject: memory-lib\n---\n\n# L2\n\nbody\n'
        _, rep = plan_new_log(tmp, "04-Logs/log-2026-00002.md", "log-2026-00002", newc)
        write_offline(tmp, "04-Logs/log-2026-00002.md", "", newc, apply=False)
        if os.path.exists(os.path.join(logs, "log-2026-00002.md")):
            failures.append("dry-run must not write")
        write_offline(tmp, "04-Logs/log-2026-00002.md", "", newc, apply=True)
        if not os.path.exists(os.path.join(logs, "log-2026-00002.md")):
            failures.append("apply new-log did not create file")

        # 2. dedup: same id again must be rejected
        try:
            plan_new_log(tmp, "04-Logs/log-2026-00003.md", "log-2026-00002", newc)
            failures.append("duplicate id not rejected")
        except GuardError:
            pass

        # 3. overwrite: existing path must be rejected
        try:
            plan_new_log(tmp, "04-Logs/log-2026-00001.md", "log-2026-00009", seed)
            failures.append("existing-path overwrite not rejected")
        except GuardError:
            pass

        # 4. backfill appends to list, body unchanged, idempotent second time
        old, new, rep = plan_backfill(tmp, "04-Logs/log-2026-00001.md",
                                      "reflect-to", "c-2026-00012", None)
        if "c-2026-00012" not in new or "# Body" not in new or "original prose" not in new:
            failures.append("backfill lost value or body")
        write_offline(tmp, "04-Logs/log-2026-00001.md", old, new, apply=True)
        old2, new2, rep2 = plan_backfill(tmp, "04-Logs/log-2026-00001.md",
                                         "reflect-to", "c-2026-00012", None)
        if "no-op" not in rep2 or new2 != read_file(os.path.join(logs, "log-2026-00001.md")):
            failures.append("backfill not idempotent")

        # 5. drift: wrong expect-hash must be rejected (ifMatch)
        good = sha256_text(read_file(os.path.join(logs, "log-2026-00001.md")))
        try:
            plan_backfill(tmp, "04-Logs/log-2026-00001.md", "reflect-to",
                          "c-2026-00099", "0" * 64)
            failures.append("drift not rejected")
        except GuardError:
            pass
        # correct hash passes
        plan_backfill(tmp, "04-Logs/log-2026-00001.md", "reflect-to", "c-2026-00099", good)

        # 6. non-derived field must be rejected
        try:
            plan_backfill(tmp, "04-Logs/log-2026-00001.md", "session-task", "x", None)
            failures.append("non-derived field not rejected")
        except GuardError:
            pass

        # 7. round-trip: split/serialize inline list
        if parse_inline_list('["a", "b"]') != ["a", "b"]:
            failures.append("parse_inline_list broken")
        if serialize_inline_list(["a"]) != '["a"]':
            failures.append("serialize_inline_list broken")

        # 8. block-style YAML must be refused (replacing only the "field:" line
        #    would orphan the "- " items = silent data loss)
        blockseed = ('---\nid: "log-2026-00004"\nproject: memory-lib\n'
                     'reflect-to:\n  - c-2026-00006\n  - c-2026-00012\n'
                     '---\n\n# Body\n\nprose\n')
        with io.open(os.path.join(logs, "log-2026-00004.md"), "w", encoding="utf-8") as f:
            f.write(blockseed)
        try:
            plan_backfill(tmp, "04-Logs/log-2026-00004.md",
                          "reflect-to", "c-2026-00099", None)
            failures.append("block-style frontmatter not rejected")
        except GuardError:
            pass

        # 9. CLI content escaping/guard (pure functions; the live CLI call needs
        #    the App, so only the encoding + safety gate are self-tested here)
        if _escape_cli_content("a\nb\tc") != "a\\nb\\tc":
            failures.append("_escape_cli_content broken")
        if not _cli_content_is_safe("plain body\nreal newline"):
            failures.append("safe content wrongly flagged unsafe")
        if _cli_content_is_safe("win path C:\\temp\\x"):
            failures.append("literal backslash-t not flagged unsafe")
        if _cli_content_is_safe("lone backslash ..\\x.md"):
            failures.append("lone backslash not flagged unsafe")
        if CLI_MAX_BYTES >= 5365:
            failures.append("CLI_MAX_BYTES too high (a 5.3KB payload crashed the App)")

        # 10. nested split: id dedup must span logs/ + corrections/ subdirs
        nested_logs = os.path.join(tmp, "04-Logs", "logs")
        nested_corr = os.path.join(tmp, "04-Logs", "corrections")
        os.makedirs(nested_logs)
        os.makedirs(nested_corr)
        with io.open(os.path.join(nested_logs, "log-2026-00010.md"), "w", encoding="utf-8") as f:
            f.write('---\nid: "log-2026-00010"\nproject: memory-lib\n---\n\n# N\n\nbody\n')
        with io.open(os.path.join(nested_corr, "corr-2026-00011.md"), "w", encoding="utf-8") as f:
            f.write('---\nid: "corr-2026-00011"\nproject: memory-lib\n---\n\n# C\n\nbody\n')
        ids = existing_log_ids(tmp)
        if "log-2026-00010" not in ids or "corr-2026-00011" not in ids:
            failures.append("nested-split id dedup missed logs/ or corrections/")
        # a new id colliding with a correction must be rejected
        try:
            plan_new_log(tmp, "04-Logs/logs/log-2026-00012.md", "corr-2026-00011", newc)
            failures.append("cross-subdir duplicate id not rejected")
        except GuardError:
            pass
        # a genuinely new id under logs/ passes the guard
        newc12 = '---\nid: "log-2026-00012"\nproject: memory-lib\n---\n\n# L12\n\nbody\n'
        plan_new_log(tmp, "04-Logs/logs/log-2026-00012.md", "log-2026-00012", newc12)
    finally:
        shutil.rmtree(tmp, ignore_errors=True)

    if failures:
        sys.stdout.write("SELF-TEST FAIL (%d):\n" % len(failures))
        for x in failures:
            sys.stdout.write("  - %s\n" % x)
        return 1
    sys.stdout.write("SELF-TEST PASS (10 rules: new-log/dedup/overwrite/"
                     "backfill/idempotent/drift-ifMatch/field-guard/block-style/"
                     "cli-escape/nested-split-dedup)\n")
    return 0


if __name__ == "__main__":
    if "--self-test" in sys.argv:
        sys.exit(_self_test())
    sys.exit(main())
````
