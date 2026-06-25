# build-mcp-server

A portable Agent Skill for designing and building Model Context Protocol servers that work with Claude. The skill guides agents through use-case discovery, deployment-model selection, tool design, auth choices, scaffolding, testing, and publication.

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
└── references/
```
