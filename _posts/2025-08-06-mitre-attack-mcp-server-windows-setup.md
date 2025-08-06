---
layout: post
title: "Setting Up MITRE ATT&CK MCP Server on Windows for Claude"
date: 2025-08-06
categories: [security, ai, tools]
tags: [mitre-attack, mcp, claude, threat-intelligence, security-analysis, model-context-protocol, windows]
author: "Daniel Streefkerk"
excerpt: "How to set up the mitre-mcp server on Windows to give Claude direct access to MITRE ATT&CK framework data for threat intelligence and security analysis."
---

Yesterday during team discussions, I pondered building a MITRE ATT&CK framework MCP server as it'd be really handy to be able to discuss MITRE ATT&CK tactics, techniques, and threat actors with Claude while having the proper context. I speculated that it should be fairly easy, given that MITRE publishes this data in JSON format across various GitHub repos. This morning I discovered that someone beat me to it (which is perfectly fine by me!). The team at [Montimage](https://www.montimage.eu) have created an MCP server that gives MCP clients like Claude direct access to MITRE ATT&CK data.

## Enter the mitre-mcp Server

This [MCP](https://modelcontextprotocol.io) server bridges the gap between the MITRE ATT&CK knowledge base and AI assistants like Claude by providing a Model Context Protocol (MCP) interface.

What this means in practice is that Claude can directly query and utilise MITRE ATT&CK data during our conversations about security. No more hallucinations or out-of-date information, Claude can look it up directly from MITRE.

## Setting It Up on Windows

Getting the server running on Windows was reasonably straightforward, though there were a couple of gotchas worth mentioning.

First, I installed the package system-wide instead of within a venv. This isn't ideal, or the cleanest way to do it, but I just wanted to test the MCP Server out:

```bash
pip install mitre-mcp
```
For MCP Servers that I've authored using Python, I usually provide a wrapper script that activates the venv and then runs the MCP server. This project doesn't include such a wrapper.

The key trick on Windows is getting the module name right in Claude's configuration. The config file lives at `%APPDATA%\Claude\claude_desktop_config.json`. 

Here's what worked for me. Note that the module name differs from what's in the project documentation:

```json
{
  "mcpServers": {
    "mitreattack": {
      "command": "python",
      "args": ["-m", "mitre_mcp.mitre_mcp_server"]
    }
  }
}
```

The documentation says to use `mitre_mcp_server` as the module name, but on my Windows installation, `mitre_mcp.mitre_mcp_server` was what actually worked. 

After saving the config, you'll need to completely restart Claude Desktop for it to pick up the new MCP server. That is, File -> Exit, rather than the 'close' button.

## Caching

One useful feature in this implementation is its intelligent caching. The MCP server downloads the MITRE ATT&CK data on first run and stores it locally. It then checks if the cached data is more than a day old on subsequent runs - if it is, it fetches fresh data automatically. For those times when you need the absolute latest data, you can force a refresh with the `--force-download` flag.

## Real-World Use Cases

Having MITRE ATT&CK data directly accessible in Claude opens up some useful possibilities for security professionals:

### Incident Response and Analysis

When analysing a security incident, you might describe the observed behaviours to Claude and immediately get relevant MITRE techniques, associated threat groups, and recommended mitigations. For example, if you're seeing PowerShell being used to download and execute payloads, Claude can instantly identify this as technique T1059.001 (Command and Scripting Interpreter: PowerShell) and tell you which APT groups commonly use this technique.

### Threat Hunting

Planning a threat hunt? Claude can help you identify techniques commonly used by specific threat actors. Ask about APT29's favourite techniques, and [you'll get a comprehensive list](https://claude.ai/share/0d13433c-f5b6-4b71-a08f-16f9c82e1927) you can use to focus your hunting efforts.

### Security Architecture Reviews

When reviewing security controls, you can map them against MITRE ATT&CK mitigations to identify gaps. Claude can tell you which techniques a particular mitigation addresses and, more importantly, which techniques might still pose a risk.

### Training and Education

For those learning about cybersecurity, having an AI assistant that can explain MITRE ATT&CK concepts with real examples is invaluable. You can ask Claude to explain the difference between tactics and techniques, or to provide examples of specific attack patterns.

## Available Tools

At the time of writing, the MCPServer provides 9 different tools that Claude can use:

- `get_techniques` - Retrieve all techniques with filtering options
- `get_tactics` - Get tactical categories
- `get_groups` - List known threat actors
- `get_software` - Find malware and tools
- `get_techniques_by_tactic` - Techniques for specific tactics
- `get_techniques_used_by_group` - Techniques used by specific threat groups
- `get_mitigations` - Security measures to counter techniques
- `get_techniques_mitigated_by_mitigation` - Techniques addressed by specific mitigations
- `get_technique_by_id` - Look up techniques by their ID

## Performance Considerations

The server is designed to be lightweight and efficient. By default, it limits technique queries to 20 results at a time and uses token-optimised responses to work well within LLM context limits. The caching mechanism ensures you're not constantly downloading the full MITRE ATT&CK dataset, which can be quite substantial.

### Adjusting the Technique Result Limit

If you need more than 20 techniques returned in a single query, you can adjust this through natural conversation with Claude. Simply specify how many results you want - for example, "Show me 50 enterprise techniques" will cause Claude to retrieve 50 techniques instead of the default 20.

The server also supports pagination through an offset parameter, so if you need to work through large result sets, you can ask Claude to get the next batch of results. For instance, after seeing the first 20 techniques, you could ask for "the next 20 techniques" and Claude will handle the pagination automatically.

## Final Thoughts

The mitre-mcp server is a great example of how MCP can bridge specialised knowledge bases with AI assistants. For security professionals, having MITRE ATT&CK data at Claude's fingertips transforms it from a general-purpose AI into a security-aware assistant that speaks the language of threat intelligence.

If you're working in cyber security and are using Claude, this is definitely worth setting up. The installation is straightforward, the caching is intelligent, and the value for security analysis work is immediate. Kudos to the Montimage team for creating this useful bridge between two powerful tools in the security professional's toolbag.

You can find the project on GitHub at [montimage/mitre-mcp](https://github.com/montimage/mitre-mcp).