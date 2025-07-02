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

The command we need to enter is as follows:

```bash
claude mcp add --transport http MicrosoftDocs https://learn.microsoft.com/api/mcp
```

## Verifying the Connection

After adding it, check that it's connected:

```bash
claude mcp list
```

You should see something like:
- Status: ✔ connected
- URL: https://learn.microsoft.com/api/mcp
- Capabilities: tools
- Tools: 1 tools

You can also inspect what tools are available:

```bash
claude mcp get MicrosoftDocs
```

## Using It in Practice

Once connected, you can:

- Just ask Claude Code questions about Microsoft tech - it should automatically use the tool when relevant. You can always nudge it to use the Microsoft Docs MCP server.
- Explicitly request searches: "Search Microsoft docs for Azure Container Apps"

The search is semantic, so natural language queries work perfectly. Results are optimised for AI consumption - you get clean, relevant chunks rather than entire documents.

## Connecting Claude Desktop to the Microsoft Docs MCP

You can also add the Microsoft Docs MCP server in Claude Desktop by clicking on the `Connect Apps` button on that chat screen, and then choosing `Add integration`. Fill out a name of your choice, and add the URL `https://learn.microsoft.com/api/mcp`, it's that simple.