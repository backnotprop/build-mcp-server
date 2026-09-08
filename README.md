# build-mcp-server

A portable Agent Skill for designing, building, auditing, and migrating Model
Context Protocol servers. The skill guides agents through use-case discovery,
deployment-model selection, tool/resource/prompt design, auth choices,
scaffolding, testing, and deployment.

The current guidance targets the released MCP `2026-07-28` stateless protocol
and keeps compatibility with the initialization-based 2025 era explicit. Its
default TypeScript scaffold uses the official v2 split packages, and its
migration playbook covers modern-only and dual-era upgrades.

Authorization is covered on both sides: the protected-resource half every MCP
server needs, and the authorization-server half for teams that issue their own
tokens, including CORS for browser-hosted clients, registration hardening,
consent, revocation, and the assertions that prove the surface.

This repo contains one skill: `build-mcp-server`.

## Install

```bash
npx skills add backnotprop/build-mcp-server
```

List available skills first:

```bash
npx skills add backnotprop/build-mcp-server --list
```

For local development from this checkout:

```bash
npx skills add /Users/ramos/oss/build-mcp-server
```

## Layout

```text
skills/build-mcp-server/
├── SKILL.md
├── agents/openai.yaml
├── assets/typescript-http/       # pinned, compile-tested modern HTTP example
└── references/
    ├── protocol-eras.md
    ├── migrate-2026-07-28.md
    ├── remote-http-scaffold.md
    ├── auth.md
    ├── authorization-server.md
    └── ...
```
