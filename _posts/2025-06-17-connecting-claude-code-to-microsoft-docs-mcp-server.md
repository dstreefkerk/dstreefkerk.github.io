---
layout: post
title: "Connecting Microsoft Docs to Claude Code via MCP"
date: 2025-06-17
categories: [tools, ai, microsoft]
tags: [claude-code, mcp, microsoft-docs, ai-tools, documentation]
author: "Daniel Streefkerk"
excerpt: "How to connect the Microsoft Docs MCP server to Claude Code for real-time access to official Microsoft documentation, eliminating outdated info and guesswork."
---

Microsoft [recently released](https://github.com/MicrosoftDocs/mcp) a MCP (Model Context Protocol) server that gives AI assistants direct access to their official documentation. This means Claude can search through Microsoft Learn, Azure docs, and other official sources in real-time. Here's how to set it up in [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview).

## What You're Getting

The Microsoft Docs MCP server provides a single tool called `microsoft_docs_search` that:

- Searches across all official Microsoft/Azure documentation
- Returns up to 10 relevant content chunks (max 500 tokens each)
- Includes article titles, URLs, and self-contained content excerpts
- Uses semantic search to find contextually relevant docs

## The Setup

First, check what MCP commands Claude Code supports:

```bash
claude mcp
```

The command we need is `add-json`:

```bash
claude mcp add-json microsoft-docs-mcp '{"type":"http","url":"https://learn.microsoft.com/api/mcp"}'
```

Note: Use hyphens or underscores in the server name - it doesn't seem to like it if you use dots.

## Verifying the Connection

After adding it, check that it's connected:

```bash
claude mcp list
```

You should see:
- Status: ✔ connected
- URL: https://learn.microsoft.com/api/mcp
- Capabilities: tools
- Tools: 1 tools

You can also inspect what tools are available:

```bash
claude mcp get microsoft-docs-mcp
```

## Using It in Practice

Once connected, you can:

- Just ask Claude Code questions about Microsoft tech - it'll automatically use the tool when relevant
- Explicitly request searches: "Search Microsoft docs for Azure Container Apps"
- Get authoritative, up-to-date info straight from the source

The search is semantic, so natural language queries work perfectly. Results are optimised for AI consumption - you get clean, relevant chunks rather than entire documents.

## Quick Reference

```bash
# Add the server
claude mcp add-json microsoft-docs-mcp '{"type":"http","url":"https://learn.microsoft.com/api/mcp"}'

# Verify it's connected
claude mcp list

# Check the available tools
claude mcp get microsoft-docs-mcp
```

The endpoint (`https://learn.microsoft.com/api/mcp`) is for programmatic access only - don't try to hit it in your browser.