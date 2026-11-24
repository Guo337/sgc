---
id: vol-rest-read
type: script-volume
created: 2026-11-16
last-touched: 2026-11-25
source-of: 90-Meta/scripts/rest_read.py
fence: "````"
eol: crlf
depends-on: [vol-ttl-locator]
sha256: 5843b20c6037d5a5bc01127605cb3358514f0c6d68477c1e70c3419f17036144
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""rest_read.py -- read-only Local REST API reader for the TTL vault.

A thin "better eyes" helper for environments WITHOUT the obsidian MCP tool
table (e.g. a fresh VSCode/Copilot sub-agent). It speaks the same plugin's
plain HTTP REST surface so a cold-start agent can read the vault following
SPEC-READ's env-split read channel: MCP present -> MCP tools; MCP absent ->
this REST reader; both absent -> official CLI -> offline grep.

Design boundary (Owner philosophy: strong text constraint, weak script
constraint): this is a READ-ONLY fact feeder for thin guardrails, NOT an
execution engine. No cache, no index, no semantic routing, no write path.

Hard rules enforced here:
  * READ-ONLY. Only GET, plus the two POST search routes (which are reads).
    Never POST/PATCH/PUT/DELETE to /vault/, never command_execute / eval.
  * Identity probe first: fetch 00-ROOT.md and compare its sha256 against the
    on-disk file, so a same-name vault collision cannot silently answer for
    the wrong library (the D: vault collision lesson).
  * API key is read in-memory from the plugin data.json (gitignored) and is
    NEVER printed, logged, or persisted.
  * Graceful downgrade: if the App/plugin is unreachable, report the reason
    and exit non-zero so the caller falls back to CLI/offline; never blame
    the user (App/plugin state is runtime, not user fault).

Endpoint shapes verified live against plugin 5.3.1 (see 02-Cards REST card):
  GET  /                              -> manifest (liveness)
  GET  /vault/{file}                  -> raw markdown
  GET  /vault/{file}  (Accept document-map+json) -> version token + headings
  GET  /vault/{file}/heading/{H1}/{H2}/... -> one section's markdown
       (each segment = the FULL heading text verbatim, URL-encoded; a '/'
        inside a heading is encoded as %2F so segments never split on it)
  GET  /vault/{dir}/                  -> {files:[...]}
  POST /search/simple/?query=&contextLength= -> [{filename,score,matches}]
  POST /search/  (Content-Type jsonlogic+json, body=JsonLogic) -> [{filename,result}]
  GET  /tags/                         -> {tags:[{name,count}]}
Note: REST has NO standalone backlinks route (MCP vault_read provides it);
for in-edges use the official CLI `obsidian backlinks` or an offline scan.

Pure standard library. English comments/docstring per SPEC-GOVERN; user-facing
report strings may be Chinese (vault content language).
"""
from __future__ import annotations

import argparse
import hashlib
import io
import json
import os
import ssl
import sys
import urllib.error
import urllib.parse
import urllib.request

from ttl_locator import resolve_vault, VaultResolutionError   # depends-on: vol-ttl-locator

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "0.3.0"

# Read-only allowlist of routes this helper will ever touch. Anything outside
# this set is refused by construction (the helper has no write code path).
DOC_MAP_ACCEPT = "application/vnd.olrapi.document-map+json"
JSONLOGIC_CT = "application/vnd.olrapi.jsonlogic+json"


class RestReadError(RuntimeError):
    """Raised on unreachable plugin, identity mismatch, or HTTP error."""


def read_plugin_config(vault: str):
    """Resolve (base_url, api_key) from the plugin data.json, in-memory only.

    Mirrors append_log.read_plugin_rest_config: the key never leaves memory.
    data.json is gitignored (agents_vault/.obsidian/plugins/).
    """
    cfg = os.path.join(vault, ".obsidian", "plugins",
                       "obsidian-local-rest-api", "data.json")
    if not os.path.exists(cfg):
        raise RestReadError(
            "plugin config not found: %s -- enable 'Local REST API with MCP' "
            "in Obsidian first (user-installed; TTL never auto-installs)"
            % cfg.replace("\\", "/"))
    try:
        with io.open(cfg, "r", encoding="utf-8") as f:
            data = json.load(f)
    except (OSError, ValueError) as e:
        raise RestReadError("could not read plugin config: %s" % e)
    key = data.get("apiKey")
    if not key:
        raise RestReadError("plugin data.json has no apiKey -- open the plugin "
                            "settings once so it generates a key")
    if data.get("enableInsecureServer"):
        base = "http://127.0.0.1:%d" % int(data.get("insecurePort", 27123))
    else:
        base = "https://127.0.0.1:%d" % int(data.get("port", 27124))
    return base, key


def _ssl_context():
    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE  # plugin serves a local self-signed cert
    return ctx


class RestClient:
    """Minimal read-only REST client. Only GET and the two search POSTs."""

    def __init__(self, base: str, key: str):
        self.base = base.rstrip("/")
        self._key = key
        self._ctx = _ssl_context()

    def _request(self, method: str, path: str, accept=None, content_type=None,
                 body=None):
        headers = {"Authorization": "Bearer " + self._key}
        if accept:
            headers["Accept"] = accept
        payload = None
        if body is not None:
            payload = json.dumps(body).encode("utf-8")
            headers["Content-Type"] = content_type or "application/json"
        url = self.base + path
        req = urllib.request.Request(url, data=payload, headers=headers,
                                     method=method)
        try:
            with urllib.request.urlopen(req, context=self._ctx, timeout=10) as r:
                return r.status, r.read().decode("utf-8", "replace")
        except urllib.error.HTTPError as e:
            detail = e.read().decode("utf-8", "replace")
            raise RestReadError("REST HTTP %s on %s: %s" % (e.code, path, detail))
        except urllib.error.URLError as e:
            raise RestReadError(
                "REST unreachable (%s) -- App/plugin not running? downgrade to "
                "CLI/offline (runtime state, not user fault)" % e.reason)
        except Exception as e:  # noqa: BLE001 - surface, never crash silently
            raise RestReadError("REST call failed on %s: %s" % (path, e))

    def get(self, path, accept=None):
        return self._request("GET", path, accept=accept)

    def manifest(self):
        st, body = self.get("/")
        return st, json.loads(body)

    def read(self, rel_path):
        st, body = self.get("/vault/" + urllib.parse.quote(rel_path))
        return st, body

    def document_map(self, rel_path):
        st, body = self.get("/vault/" + urllib.parse.quote(rel_path),
                            accept=DOC_MAP_ACCEPT)
        return st, json.loads(body)

    def read_heading(self, rel_path, headings):
        """GET one section by its full heading path (each segment verbatim).

        Segments are quoted with safe="" so a '/' inside a heading text is
        encoded as %2F and cannot be mistaken for a path separator.
        """
        path = "/vault/" + urllib.parse.quote(rel_path) + "/heading/" + \
            "/".join(urllib.parse.quote(h, safe="") for h in headings)
        return self.get(path)

    def list_dir(self, rel_dir=""):
        path = "/vault/" + urllib.parse.quote(rel_dir)
        if not path.endswith("/"):
            path += "/"
        st, body = self.get(path)
        return st, json.loads(body)

    def search_simple(self, query, context_length=50):
        path = "/search/simple/?query=%s&contextLength=%d" % (
            urllib.parse.quote(query), int(context_length))
        st, body = self._request("POST", path)
        return st, json.loads(body)

    def search_jsonlogic(self, logic):
        st, body = self._request("POST", "/search/", content_type=JSONLOGIC_CT,
                                 body=logic)
        return st, json.loads(body)

    def tags(self):
        st, body = self.get("/tags/")
        return st, json.loads(body)


def sha256_text(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()


def identity_probe(client: RestClient, vault: str) -> str:
    """Fetch 00-ROOT.md via REST and compare sha256 with the on-disk file.

    Returns the matching hash. Raises RestReadError on mismatch so a same-name
    vault collision is caught before any answer is trusted.
    """
    disk_path = os.path.join(vault, "00-ROOT.md")
    # newline="" keeps raw CRLF bytes: universal-newline translation would
    # fake a MISMATCH against REST on Windows autocrlf checkouts (sgc#1).
    with io.open(disk_path, "r", encoding="utf-8", newline="") as f:
        disk_hash = sha256_text(f.read())
    _, remote = client.read("00-ROOT.md")
    remote_hash = sha256_text(remote)
    if disk_hash != remote_hash:
        raise RestReadError(
            "identity probe MISMATCH: REST 00-ROOT.md sha256=%s != disk %s -- "
            "the plugin is answering for a DIFFERENT vault; do not trust it"
            % (remote_hash[:12], disk_hash[:12]))
    return disk_hash


def _client(vault: str) -> RestClient:
    base, key = read_plugin_config(vault)
    return RestClient(base, key)


def format_heading_tree(headings: dict, indent: int = 0):
    """Render a document-map heading tree as indented lines. Pure facts:
    every verbatim title is shown so the caller fixes the path itself."""
    lines = []
    for title, sub in headings.items():
        lines.append("  " * indent + "- " + title)
        lines.extend(format_heading_tree(sub, indent + 1))
    return lines


def read_heading_or_facts(client: RestClient, rel_path: str, headings):
    """Read one section by full heading path. On failure, fetch the document
    map and attach the available heading tree to the error so the caller can
    correct the path (feed facts; never guess or fuzzy-match a heading)."""
    try:
        _, body = client.read_heading(rel_path, headings)
        return body
    except RestReadError as e:
        msg = str(e)
        try:
            _, dm = client.document_map(rel_path)
            tree = format_heading_tree(dm.get("headings", {}))
            if tree:
                msg += ("\n可用标题树（--heading 每段须逐字精确，含括号/版本号；"
                        "每层一个 --heading）：\n" + "\n".join(tree))
            else:
                msg += "\n（document-map 显示该文件没有任何标题）"
        except RestReadError as e2:
            msg += "\n（document-map 也不可用，无法列出标题树：%s）" % e2
        raise RestReadError(msg)


def cmd_probe(vault: str) -> int:
    client = _client(vault)
    st, manifest = client.manifest()
    info = manifest.get("manifest", {})
    h = identity_probe(client, vault)
    sys.stdout.write("REST OK: plugin=%s version=%s status=%s\n"
                     % (info.get("id", "?"), info.get("version", "?"),
                        manifest.get("status", "?")))
    sys.stdout.write("identity probe PASS: 00-ROOT.md sha256=%s (REST==disk)\n" % h[:16])
    # SH pointer at the physical choke point every REST consumer passes
    # (corr-127 philosophy: thin guardrail, reachable without reading specs).
    sys.stdout.write("身份与义务包：03-Atlas/SH-index.md（先认身份再读知识，SPEC-READ §入口 6）\n")
    return 0


# Directories scanned by `resolve`. Fixed allowlist keeps the GET count small
# and the surface read-only; extend only with an explicit ruling.
RESOLVE_DIRS = ("02-Cards", "01-Specs", "03-Atlas")
RESOLVE_MAX_DEPTH = 4


def resolve_name(client, name: str):
    """Exact basename match over directory listings. Returns (hits, scanned).

    Facts only: no substring, no fuzzy, no alias lookup (aliases would need
    per-file frontmatter GETs -- out of scope for this thin fact feeder).
    Multiple hits are ALL returned (a name collision is a fact for the caller
    to adjudicate, never silently resolved to the first match).
    """
    target = name if name.endswith(".md") else name + ".md"
    hits, scanned = [], []

    def walk(rel_dir, depth):
        scanned.append(rel_dir or "/")
        _, data = client.list_dir(rel_dir)
        for entry in data.get("files", []):
            if entry.endswith("/"):
                if depth < RESOLVE_MAX_DEPTH:
                    sub = (rel_dir + "/" + entry.rstrip("/")) if rel_dir \
                        else entry.rstrip("/")
                    walk(sub, depth + 1)
            elif entry == target:
                hits.append((rel_dir + "/" + entry) if rel_dir else entry)

    for d in RESOLVE_DIRS:
        walk(d, 1)
    return hits, scanned


def cmd_resolve(vault: str, name: str) -> int:
    client = _client(vault)
    identity_probe(client, vault)
    hits, scanned = resolve_name(client, name)
    if not hits:
        sys.stdout.write("resolve GAP: 「%s」精确名未命中（非别名/非语义匹配）。"
                         "已扫目录：%s\n" % (name, ", ".join(scanned)))
        return 1
    for h in hits:
        sys.stdout.write(h + "\n")
    if len(hits) > 1:
        sys.stdout.write("（同名多命中 %d 件，全部列出供裁，不静默取首）\n" % len(hits))
    return 0


def cmd_read(vault: str, rel_path: str, as_map: bool, headings=None) -> int:
    client = _client(vault)
    identity_probe(client, vault)
    if headings:
        if any(not h.strip() for h in headings):
            sys.stderr.write("--heading 段不能为空：每段须是完整标题原文\n")
            return 2
        try:
            body = read_heading_or_facts(client, rel_path, headings)
        except RestReadError as e:
            sys.stderr.write("按节读取失败：%s\n" % e)
            return 1
    elif as_map:
        _, data = client.document_map(rel_path)
        sys.stdout.write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
        return 0
    else:
        _, body = client.read(rel_path)
    sys.stdout.write(body)
    if not body.endswith("\n"):
        sys.stdout.write("\n")
    return 0


def cmd_list(vault: str, rel_dir: str) -> int:
    client = _client(vault)
    identity_probe(client, vault)
    _, data = client.list_dir(rel_dir)
    for name in data.get("files", []):
        sys.stdout.write(name + "\n")
    return 0


def cmd_search(vault: str, query: str, context_length: int) -> int:
    client = _client(vault)
    identity_probe(client, vault)
    _, results = client.search_simple(query, context_length)
    sys.stdout.write(json.dumps(results, ensure_ascii=False, indent=2) + "\n")
    return 0


def cmd_tags(vault: str) -> int:
    client = _client(vault)
    identity_probe(client, vault)
    _, data = client.tags()
    sys.stdout.write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
    return 0


def self_test() -> int:
    """Offline self-test: no network, no App. Verifies the read-only contract,
    identity-probe mismatch detection, and config parsing using fixtures."""
    import tempfile
    import shutil

    failures = []

    def check(name, cond):
        if cond:
            sys.stdout.write("  [OK] %s\n" % name)
        else:
            failures.append(name)
            sys.stdout.write("  [FAIL] %s\n" % name)

    tmp = tempfile.mkdtemp(prefix="rest_read_selftest.")
    try:
        vault = os.path.join(tmp, "vault")
        plug = os.path.join(vault, ".obsidian", "plugins", "obsidian-local-rest-api")
        os.makedirs(plug)
        with io.open(os.path.join(vault, "00-ROOT.md"), "w", encoding="utf-8") as f:
            f.write("# root\nidentity-anchor\n")

        # 1. config parse: insecure http port selection, key never returned wrong
        with io.open(os.path.join(plug, "data.json"), "w", encoding="utf-8") as f:
            json.dump({"apiKey": "x" * 64, "enableInsecureServer": True,
                       "insecurePort": 27123}, f)
        base, key = read_plugin_config(vault)
        check("CONFIG_HTTP", base == "http://127.0.0.1:27123" and len(key) == 64)

        # 2. config parse: https fallback when insecure disabled
        with io.open(os.path.join(plug, "data.json"), "w", encoding="utf-8") as f:
            json.dump({"apiKey": "y" * 64, "enableInsecureServer": False,
                       "port": 27124}, f)
        base2, _ = read_plugin_config(vault)
        check("CONFIG_HTTPS", base2 == "https://127.0.0.1:27124")

        # 3. missing apiKey -> fail loud
        with io.open(os.path.join(plug, "data.json"), "w", encoding="utf-8") as f:
            json.dump({"enableInsecureServer": True}, f)
        try:
            read_plugin_config(vault)
            check("CONFIG_NOKEY", False)
        except RestReadError:
            check("CONFIG_NOKEY", True)

        # 4. identity probe MATCH (fake client returning disk content)
        with io.open(os.path.join(vault, "00-ROOT.md"), encoding="utf-8",
                     newline="") as f:
            disk_text = f.read()

        class FakeClient:
            def __init__(self, text):
                self.text = text

            def read(self, rel):
                return 200, self.text

        h = identity_probe(FakeClient(disk_text), vault)
        check("IDENTITY_MATCH", h == sha256_text(disk_text))

        # 5. identity probe MISMATCH -> raise (wrong-vault collision guard)
        try:
            identity_probe(FakeClient("# different vault\n"), vault)
            check("IDENTITY_MISMATCH", False)
        except RestReadError:
            check("IDENTITY_MISMATCH", True)

        # 5b. CRLF regression (sgc#1): disk file with CRLF, REST returns the
        # same raw bytes -> must MATCH (universal-newline read faked MISMATCH)
        crlf_text = "# root\r\nidentity-anchor\r\n"
        with io.open(os.path.join(vault, "00-ROOT.md"), "w",
                     encoding="utf-8", newline="") as f:
            f.write(crlf_text)
        h = identity_probe(FakeClient(crlf_text), vault)
        check("IDENTITY_CRLF", h == sha256_text(crlf_text))

        # 6. read-only contract: client exposes no write method
        write_methods = [m for m in dir(RestClient)
                         if m in ("write", "patch", "delete", "put", "append",
                                  "create", "command_execute", "eval")]
        check("READONLY_SURFACE", not write_methods)

        # 7. search_simple builds POST + ?query= (not GET, not JSON body)
        captured = {}

        class CaptureClient(RestClient):
            def __init__(self):
                pass

            def _request(self, method, path, accept=None, content_type=None,
                         body=None):
                captured["method"] = method
                captured["path"] = path
                captured["body"] = body
                return 200, "[]"

        cc = CaptureClient()
        cc.search_simple("SPEC-GOVERN", 30)
        check("SEARCH_POST_QUERY",
              captured["method"] == "POST"
              and "/search/simple/" in captured["path"]
              and "query=SPEC-GOVERN" in captured["path"]
              and captured["body"] is None)

        # 8. jsonlogic search uses vendor Content-Type
        captured.clear()
        cc.search_jsonlogic({"==": [{"var": "frontmatter.project"}, "memory-lib"]})
        check("SEARCH_JSONLOGIC_CT",
              captured["method"] == "POST"
              and captured["path"] == "/search/"
              and captured["body"] is not None)

        # 9. heading URL: Chinese / inner-slash / fullwidth-paren segments
        #    are percent-encoded; round-trip must recover each segment verbatim
        captured.clear()
        segs = ["规格书 v0.9.1（草案）", "通道（MCP / REST / CLI）", "角色"]
        cc.read_heading("01-Specs/SPEC-GOVERN.md", segs)
        prefix = "/vault/01-Specs/SPEC-GOVERN.md/heading/"
        tail = captured["path"][len(prefix):]
        check("HEADING_URL",
              captured["method"] == "GET"
              and captured["path"].startswith(prefix)
              and tail.count("/") == 2
              and "%2F" in tail
              and "%EF%BC%88" in tail
              and all(ord(c) < 128 for c in captured["path"])
              and [urllib.parse.unquote(t) for t in tail.split("/")] == segs)

        # 10. --heading is repeatable (append); inner '/' stays inside a segment
        args = build_parser().parse_args(
            ["read", "a.md", "--heading", "H1（含 / 斜杠）", "--heading", "H2"])
        check("HEADING_APPEND",
              args.heading == ["H1（含 / 斜杠）", "H2"] and args.map is False)

        # 11. --map and --heading are mutually exclusive -> parser exits 2
        import contextlib
        try:
            with contextlib.redirect_stderr(io.StringIO()):
                build_parser().parse_args(
                    ["read", "a.md", "--map", "--heading", "H"])
            check("HEADING_MAP_EXCLUSIVE", False)
        except SystemExit as e:
            check("HEADING_MAP_EXCLUSIVE", e.code == 2)

        # 12. heading tree rendering preserves nesting and verbatim titles
        tree = {"H1": {"H2（v1.0）": {"H3": {}}, "H2b": {}}, "H1b": {}}
        check("HEADING_TREE", format_heading_tree(tree) == [
            "- H1", "  - H2（v1.0）", "    - H3", "  - H2b", "- H1b"])

        # 13. heading read success passes the section body through verbatim
        class HeadingFake:
            def read_heading(self, rel, headings):
                return 200, "| 角色 | 职责 |\n"

            def document_map(self, rel):
                raise AssertionError("map must not be fetched on success")

        body = read_heading_or_facts(HeadingFake(), "x.md", ["H1"])
        check("HEADING_OK", body == "| 角色 | 职责 |\n")

        # 14. heading 404 -> error carries the available heading tree (facts)
        class Heading404:
            def read_heading(self, rel, headings):
                raise RestReadError("REST HTTP 404 on /vault/x.md/heading/WRONG")

            def document_map(self, rel):
                return 200, {"headings": {"真实H1": {"真实H2（v2.0）": {}}}}

        try:
            read_heading_or_facts(Heading404(), "x.md", ["WRONG"])
            check("HEADING_404_FACTS", False)
        except RestReadError as e:
            msg = str(e)
            check("HEADING_404_FACTS",
                  "404" in msg and "真实H1" in msg and "真实H2（v2.0）" in msg)

        # 15. resolve: unique exact hit returns the correct relative path
        class DirFake:
            def __init__(self, tree):
                self.tree = tree
                self.calls = []

            def list_dir(self, rel):
                self.calls.append(rel)
                return 200, {"files": self.tree.get(rel, [])}

        tree = {
            "02-Cards": ["甲卡.md", "incubator/"],
            "02-Cards/incubator": ["乙卡.md"],
            "01-Specs": ["SPEC-READ.md"],
            "03-Atlas": ["SH-index.md", "knowledge-engineering/"],
            "03-Atlas/knowledge-engineering": ["roles/"],
            "03-Atlas/knowledge-engineering/roles": ["role-ke-consumer.md"],
        }
        df = DirFake(tree)
        hits, scanned = resolve_name(df, "role-ke-consumer")
        check("RESOLVE_UNIQUE",
              hits == ["03-Atlas/knowledge-engineering/roles/role-ke-consumer.md"])

        # 16. resolve: same basename in two dirs -> BOTH listed, never first-only
        tree2 = dict(tree)
        tree2["01-Specs"] = ["SPEC-READ.md", "甲卡.md"]
        hits2, _ = resolve_name(DirFake(tree2), "甲卡")
        check("RESOLVE_MULTI",
              hits2 == ["02-Cards/甲卡.md", "01-Specs/甲卡.md"])

        # 17. resolve: zero hit -> empty hits + scanned facts (GAP, no guessing)
        hits3, scanned3 = resolve_name(DirFake(tree), "不存在的卡")
        check("RESOLVE_GAP", hits3 == [] and "02-Cards" in scanned3
              and "03-Atlas/knowledge-engineering/roles" in scanned3)

        # 18. resolve: substring / case variants do NOT match (exact, not fuzzy)
        h_sub, _ = resolve_name(DirFake(tree), "甲")          # substring
        h_case, _ = resolve_name(DirFake(tree), "spec-read")  # case variant
        check("RESOLVE_EXACT", h_sub == [] and h_case == [])

        # 19. resolve: read-only surface -- only directory GETs (trailing slash)
        captured_calls = []

        class ResolveCapture(RestClient):
            def __init__(self):
                pass

            def _request(self, method, path, accept=None, content_type=None,
                         body=None):
                captured_calls.append((method, path))
                return 200, json.dumps({"files": []})

        resolve_name(ResolveCapture(), "x")
        check("RESOLVE_READONLY",
              all(m == "GET" and p.startswith("/vault/") and p.endswith("/")
                  for m, p in captured_calls) and len(captured_calls) >= 3)
    finally:
        shutil.rmtree(tmp, ignore_errors=True)

    if failures:
        sys.stdout.write("SELF-TEST FAIL: %s\n" % ", ".join(failures))
        return 1
    sys.stdout.write("SELF-TEST PASS (19 rules: config http/https/nokey, "
                     "identity match/mismatch/crlf, readonly surface, "
                     "search post+query, jsonlogic CT, heading "
                     "url/append/exclusive/tree/ok/404-facts, "
                     "resolve unique/multi/gap/exact/readonly)\n")
    return 0


def build_parser():
    p = argparse.ArgumentParser(
        description="Read-only Local REST API reader for the TTL vault "
                    "(MCP-absent environments). Never writes.")
    p.add_argument("--vault", default=None,
                   help="vault root (default: $TTL_VAULT / config / discovery)")
    sub = p.add_subparsers(dest="cmd")
    sub.add_parser("probe", help="liveness + identity probe (00-ROOT sha256)")
    pr = sub.add_parser("read",
                        help="read a note (raw markdown, --map, or --heading)")
    pr.add_argument("path", help="vault-relative path, e.g. 01-Specs/SPEC-READ.md")
    mode = pr.add_mutually_exclusive_group()
    mode.add_argument("--map", action="store_true",
                      help="return document-map (version token + headings)")
    mode.add_argument("--heading", action="append", metavar="TEXT",
                      help="read one section by its full heading path; repeat "
                           "once per level, each TEXT the verbatim heading "
                           "(inner '/' is safe), e.g. --heading \"H1\" "
                           "--heading \"H2（v1.0）\"")
    pl = sub.add_parser("list", help="list a directory")
    pl.add_argument("dir", nargs="?", default="", help="vault-relative dir")
    ps = sub.add_parser("search", help="full-text search (POST /search/simple/)")
    ps.add_argument("query")
    ps.add_argument("--context-length", type=int, default=50)
    sub.add_parser("tags", help="list tags")
    prs = sub.add_parser("resolve",
                         help="exact basename -> vault path(s); facts only "
                              "(no alias/semantic routing -- that stays with "
                              "read_orchestrator)")
    prs.add_argument("name", help="note name, with or without .md")
    return p


def main(argv=None) -> int:
    if argv is None:
        argv = sys.argv[1:]
    if "--self-test" in argv:
        return self_test()
    args = build_parser().parse_args(argv)
    if not args.cmd:
        build_parser().print_help()
        return 2
    try:
        loc = resolve_vault(args.vault,
                            script_fallback=os.path.join(
                                os.path.dirname(__file__), "..", ".."))
    except VaultResolutionError as e:
        sys.stdout.write("vault not found: %s\n" % e)
        return 2
    vault = loc.root
    try:
        if args.cmd == "probe":
            return cmd_probe(vault)
        if args.cmd == "read":
            return cmd_read(vault, args.path, args.map, args.heading)
        if args.cmd == "list":
            return cmd_list(vault, args.dir)
        if args.cmd == "search":
            return cmd_search(vault, args.query, args.context_length)
        if args.cmd == "tags":
            return cmd_tags(vault)
        if args.cmd == "resolve":
            return cmd_resolve(vault, args.name)
    except RestReadError as e:
        sys.stdout.write("REST read failed: %s\n" % e)
        return 1
    return 2


if __name__ == "__main__":
    sys.exit(main())
````
