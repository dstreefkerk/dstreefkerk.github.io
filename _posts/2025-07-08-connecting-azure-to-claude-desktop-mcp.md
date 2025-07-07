---
layout: post
title: "Connecting Azure to Claude Desktop via MCP"
date: 2025-07-08
categories: [azure, ai, tools]
tags: [claude, mcp, azure, ai-tools, model-context-protocol]
author: "Daniel Streefkerk"
excerpt: "How to connect the Azure MCP server to Claude Desktop for direct access to Azure resources, enabling Claude to help with Azure development and operations."
---

Microsoft [recently released](https://devblogs.microsoft.com/foundry/integrating-azure-ai-agents-mcp/) a MCP (Model Context Protocol) server that gives AI assistants direct access to Azure resources. While Microsoft's documentation expectedly focuses on VS Code and GitHub Copilot integration, you can also use it with other MCP Clients like Claude Desktop too. Here's how to set it up.

## What You're Getting

The Azure MCP Server provides tools that let Claude:

- List and inspect Azure resources across subscriptions
- Query Azure Monitor logs and metrics
- Examine resource configurations and properties
- Access Azure documentation in real-time
- Work with Log Analytics workspaces
- Investigate Application Insights data

At the time of writing, there are some features that I wish were included, such as the ability to inspect Logic App configurations, but I'm sure that'll come soon enough.

By default, the server has full read/write access to your Azure resources. For safety, I recommend using the `--read-only` flag as shown in this guide.

## Prerequisites

Before setting up the Azure MCP server, ensure you have:

- [Claude Desktop](https://claude.ai/download) installed
- [Node.js](https://nodejs.org/) (Latest LTS version)
- Azure CLI installed and authenticated (`az login`)
- Appropriate read permissions on the Azure resources you want to access

## The Setup

Setting up the Azure MCP server in Claude Desktop is straightforward. You need to edit Claude's configuration file to add the server.

### Step 1: Locate the Configuration File

Find your Claude Desktop configuration file:

- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Linux**: `~/.config/Claude/claude_desktop_config.json`

### Step 2: Add the Azure MCP Server

Open the configuration file in a text editor and add the Azure MCP server to the `mcpServers` section:

```json
{
  "mcpServers": {
    "Azure Read-Only": {
      "command": "npx",
      "args": [
        "-y",
        "@azure/mcp@latest",
        "server",
        "start",
        "--read-only"
      ]
    }
  }
}
```

If you already have other MCP servers configured, just add the Azure server as another entry in the `mcpServers` object.

### Step 3: Restart Claude Desktop

After saving the configuration file, completely quit and restart Claude Desktop for the changes to take effect. In Windows, at least, you need to use the waffle menu at the top-left, and choose File, Exit. Don't just click on the 'x' at the top-right of the window.

## Verifying the Connection

Once Claude Desktop restarts, you can verify the Azure MCP server is connected by:

1. Starting a new conversation
2. Asking Claude to list your Azure subscriptions
3. Claude should automatically use the Azure MCP tools to retrieve this information

You'll see Claude mention when it's using the Azure tools, something like "I'll check your Azure subscriptions for you.". It'll ask for permission the first time it uses each tool.

## Authentication

The Azure MCP server uses your existing Azure CLI authentication. Make sure you're logged in:

```bash
az login
```

If you work with multiple Azure accounts or subscriptions, ensure you've set the correct context:

```bash
# List available subscriptions
az account list --output table

# Set the active subscription
az account set --subscription "Your Subscription Name"
```

## Usage Examples

Once connected, you can ask Claude to:

- **Explore resources**: "What virtual machines do I have in my Azure subscription?"
- **Check configurations**: "Show me the configuration of my storage accounts"
- **Query logs**: "Find errors in my Application Insights logs from the last 24 hours"
- **Analyse costs**: "Which resource groups are consuming the most resources?"
- **Troubleshoot issues**: "Help me investigate why my web app is running slowly"

## Security Considerations

The `--read-only` flag in our configuration ensures Claude can only view your Azure resources, not modify them. Without this flag, the Azure MCP server has full read/write access to your Azure resources, which is why I strongly recommend using read-only mode. 

**Production Data Warning**: Connecting production Azure environments to public LLMs like Claude poses significant privacy and security risks. Your Azure resources may contain:
- Sensitive configuration details
- API keys and connection strings
- User data and access patterns
- Business-critical information

**Best Practices**:
- Use test or development subscriptions when possible
- Be aware that Claude can see all resources your Azure CLI credentials can access
- Consider using a service principal with minimal read permissions instead of your full user credentials
- For production scenarios, consider using a private LLM deployment instead of public services

## Alternative: Full Access Mode

If you need Claude to help with resource creation or modification, you can remove the `--read-only` flag:

```json
{
  "mcpServers": {
    "Azure Full Access": {
      "command": "npx",
      "args": [
        "-y",
        "@azure/mcp@latest",
        "server",
        "start"
      ]
    }
  }
}
```

**Warning**: Only use full access mode if you understand the implications and trust the operations you're asking Claude to perform. I wouldn't use it anywhere other than a lab tenant.

## Troubleshooting

If the Azure MCP server isn't working:

1. **Check Node.js**: Ensure Node.js is installed and in your PATH
2. **Verify Azure CLI**: Run `az account show` to confirm you're authenticated
3. **Review logs**: Check Claude Desktop's logs for any error messages
4. **Test manually**: Try running the npx command directly in a terminal to see if there are any issues

## Combining with Other MCP Servers

The Azure MCP server works great alongside other MCP servers. For example, you might use it with the Microsoft Docs MCP server for comprehensive Azure assistance:

```json
{
  "mcpServers": {
    "Azure Read-Only": {
      "command": "npx",
      "args": ["-y", "@azure/mcp@latest", "server", "start", "--read-only"]
    },
    "Microsoft Docs": {
      "url": "https://learn.microsoft.com/api/mcp"
    }
  }
}
```

This combination gives Claude both real-time access to your Azure resources and the latest Azure documentation.

Just remember to start with read-only mode and be mindful of the security implications of giving an AI assistant access to your cloud resources.