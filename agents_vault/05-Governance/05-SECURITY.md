---
id: gov-2026-security
type: constraint
status: ACTIVE
priority: P0
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-16
aliases: [安全红线, Security Boundary]
---

# Security Boundary

## Absolute red lines

These do not weaken under `fast-iterate`:

1. Do not install or execute third-party skills/plugins/packages without explicit human review.
2. Never write secrets, credentials, or private sensitive material into the vault.
3. Do not connect the vault to an automatic cloud-LLM upload path.
4. Network use remains bounded by the existing project rules.
5. Do not use `obsidian eval`.
6. Do not expand the MCP tool permission set for convenience.
7. Do not treat localhost or a development machine as automatic authorization.

## Current MCP read boundary

When using Local REST API MCP for memory retrieval, stay within the currently approved read-only set:

- `search_simple`
- `search_query`
- `vault_read`
- `vault_get_document_map`
- `tag_list`
- `vault_list`

Do not call:
- command execution;
- delete;
- move;
- write;
- or any unapproved dangerous tool.

## Write boundary

Memory writes currently use the official first-party `obsidian` CLI path.

## Mode rule

`fast-iterate` may reduce confirmation for reversible, low-risk operations.

It never lowers an absolute safety boundary.
