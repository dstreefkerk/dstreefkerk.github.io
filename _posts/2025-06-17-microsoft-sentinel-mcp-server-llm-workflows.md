---
layout: post
title: "Connecting Microsoft Sentinel to a LLM via Model Context Protocol (MCP)"
date: 2025-06-17
categories: [azure, security, ai, automation]
tags: [microsoft-sentinel, mcp, claude, llm, automation, kql, security-operations, azure-monitor, security-copilot]
author: "Daniel Streefkerk"
excerpt: "How I built an MCP server to bridge Microsoft Sentinel with Large Language Models."
---

I've been working on some projects this year that involve querying Microsoft Sentinel in my lab environment and building up Analytics Rules and KQL queries. While Claude is decent at churning out KQL code, once you're into the more advanced scenarios it almost always delivers buggy queries that don't work properly out of the box.

What follows is a back-and-forth dance of "me: hey, I got this error" and "Claude: oops, yes, I should have done {xyz}" until we come to a workable solution.

When I discovered Model Context Protocol (MCP), I thought it'd be a perfect test-case to connect Claude to Sentinel directly and cut out the middle man.

At the same time, I was observing a customer attempt to implement [Security Copilot](https://learn.microsoft.com/en-us/copilot/security/microsoft-security-copilot) on a shoestring budget. As it turns out, Security Copilot doesn't work very well unless you throw a lot of [Security Compute Units](https://learn.microsoft.com/en-us/copilot/security/manage-usage) (SCUs) at it. In Microsoft's own demos they're often assigning 500 SCUs or more, so it's no wonder that the product performs nicely during the demos but falls flat when you don't throw a similar number of SCUs at it.

> Side note: a generalised LLM like Claude is not a proper alternative for what Microsoft have built with Security Copilot. Copilot does a lot more than just link a LLM to your security data - for example the "grounding" process that Security Copilot uses to avoid hallucinations. There's also the fact that it's not a good idea to hook up Claude or any other public LLM to a production Sentinel instance.

## What's Model Context Protocol?

The Model Context Protocol is Anthropic's open standard for connecting AI assistants to external data sources and tools. Think of it as a bridge that lets Claude (or other LLMs) interact with your systems in a structured manner, without developers needing to write a specific Sentinel interface for each different LLM.

Instead of copying and pasting data between systems or writing one-off scripts, MCP servers expose capabilities through a standardised interface. The LLM can then call these tools naturally as part of a conversation, making complex multi-step workflows feel seamless.

The conversation about Anthropic re-inventing protocols when we already have battle-tested protocols for transferring text across networks is a matter for another day. Authentication and Authorisation has also been an afterthought, however that's now changing. Microsoft [recently came out and announced](https://techcrunch.com/2025/05/19/github-microsoft-embrace-anthropics-spec-for-connecting-ai-models-to-data-sources/) that they're jumping on board with MCP, and at the time of writing, VS Code is the only fully-featured MCP client that supports the full protocol suite.

## Enter the Microsoft Sentinel MCP Server

I built this MCP server purely to scratch my own itch and to expore what's possible if you give a LLM like Claude access to Sentinel. 

My MCP server provides **read-only** access to a Sentinel instance with over 40 different tools covering the spectrum of security operations. I purposefully chose not to implement any write/delete operations, and I've also made it quite clear in the project documentation that it's not intended to be connected to production Sentinel environments.

Here's what I've included:

### Core Security Operations
- **KQL Query Execution**: Run queries against Log Analytics, including testing with mock data
- **Local KQL Validation**: Save on LLM context and calls by first validating KQL locally using Microsoft's own Kusto DLL
- **Incident Management**: List and analyse security incidents with full details and related alerts
- **Analytics Rules**: Browse detection rules, analyse by MITRE tactics/techniques, and explore rule templates
- **Hunting Queries**: Access the full library of hunting queries with tactical analysis

### Infrastructure and Configuration
- **Log Analytics Management**: Workspace details, table schemas, and data retention info
- **Data Connectors**: Inventory and configure your data ingestion pipelines. Keep in mind that the Graph API doesn't surface all DCs nowadays, so this tool's usefulness might be limited.
- **Watchlists**: Manage threat intelligence and asset inventories

### Threat Intelligence and Enrichment
- **Domain WHOIS Lookups**: Get registration details for suspicious domains
- **IP Geolocation**: Enrich network indicators with location data
- **Metadata and Source Control**: Track deployment history and rule provenance

### Identity and Access
- **Entra ID Integration**: Query users and groups directly from Azure AD, if the identity running the MCP Server has the correct privileges in Entra.

## The Technical Bits

The server is built in Python using Anthropic's MCP SDK and integrates with Azure's extensive SDK ecosystem. Authentication works through Azure CLI or service principal credentials, supporting any method that the Azure SDK's `DefaultAzureCredential` recognises.

I've structured it as a modular platform. Tools are auto-discovered from the `tools/` directory, so extending functionality is straightforward. Each tool follows a consistent pattern for error handling, parameter validation, and response formatting.

One feature that is kind of cool is the KQL testing functionality. The LLM can provide mock data in XML or CSV format, and the server will automatically create a `datatable` construct to test your queries against realistic data without touching production Log Analytics tables. This has been invaluable for developing detection rules in environments where I can't yet access real security logs, but where I have sample logs from the vendor.

## Security Considerations (Because They Matter)

To reiterate: **this is for test environments only**. I've deliberately kept the server read-only, but even read access to a production Sentinel instance contains incredibly sensitive data. User activities, security alerts, network traffic patterns aren't something that should just be handed over to the tech bros.

> **I cannot stress this enough: do not connect this to production Sentinel instances unless you're using a private LLM deployment that you fully control.** The potential for data exposure to public LLM providers is simply too high for production security operations.

For test environments, though, it's been awesome. I can rapidly prototype detection logic, validate rule coverage, and generate comprehensive security reports without manual work or the usual API juggling.

## Installation and Getting Started

Head over to the project on GitHub, the [readme.md](https://github.com/dstreefkerk/ms-sentinel-mcp-server/blob/main/README.md) is pretty comprehensive.

I've included a PowerShell installation script that handles the tedious bits—creating the Python virtual environment, installing dependencies, and generating the Claude Desktop configuration. Just run `.\install.ps1` from the repository root, and you'll have everything ready to paste into your MCP client's MCP server config file.

I've not catered for Linux or Mac users (sorry), but the MCP server _should_ work ok, you'll just not be able to run the installer script.

The MCP server supports both Azure CLI authentication (simplest for local development) and service principal authentication (better for automated workflows). For service principals, you'll need `Log Analytics Reader` and `Microsoft Sentinel Reader` roles at minimum.

Just remember the security implications and stick to test environments until private MCP-capable LLM deployments become more accessible.

The code's available on GitHub at [dstreefkerk/ms-sentinel-mcp-server](https://github.com/dstreefkerk/ms-sentinel-mcp-server). As always, use at your own risk.

## What's Next

To be honest, not much. I'm not using the project right now, but I'm sure that there'll be updates and tweaks to be made next time I need to use it again. The MCP spec has already evolved significantly since I built the server in the first place.