# Sublay skills for Claude Code

Official [Sublay](https://sublay.io) skills for Claude Code, distributed as a plugin.

Sublay is infrastructure for user-based products: pre-modeled accounts, content, interactions, files, chat, notifications, moderation, search and more, with SDKs for React, React Native, JavaScript and Node. These skills give a coding agent an accurate map of what the platform can do, so it builds on what already exists instead of reinventing it.

## Install

In Claude Code:

```
/plugin marketplace add sublay-io/claude-skills
/plugin install sublay@sublay-io
```

The first command registers this repo as a plugin marketplace; the second installs the `sublay` plugin from it.

## What's included

- **implementation-expert** — a complete, accurate map of Sublay's models, bundles, and capabilities (entities, spaces, chat, custom tables, storage, semantic search & AI, and more), organized by domain. It's a first point of contact, not a full spec: for exact hook signatures, params, and endpoint shapes it points at the live docs.

Installing the plugin also wires up the public Sublay docs MCP server (`https://docs.sublay.io/mcp`, no auth) via [`.mcp.json`](.mcp.json), so the skills can pull exact field and endpoint detail on demand.

## Adding more skills

This is a single plugin (`sublay`) with a `skills/` directory. To add another skill, drop a new folder under [`skills/`](skills/) containing a `SKILL.md` and bump the plugin `version` in [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json). Everyone who has the plugin installed picks it up on update.

## Docs

Full platform docs: [docs.sublay.io](https://docs.sublay.io).
