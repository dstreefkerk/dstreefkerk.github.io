---
layout: post
title: "Running Claude Code from Windows CLI: A Practical Guide"
date: 2026-01-29
categories: [tools, automation, ai]
tags: [claude-code, anthropic, windows, cli, python, automation, llm]
author: "Daniel Streefkerk"
excerpt: "Practical lessons learned from programmatically invoking Claude Code on Windows, including the gotchas around tool permissions and system prompts that took some time to figure out."
---

I've been building automation workflows that invoke Claude Code programmatically from Python scripts on Windows. What started as a straightforward subprocess call turned into a chunk of time spent on troubleshooting non-obvious behaviours. This post is based on a braindump written by Claude Code after going through that process.

I'm documenting this because executing Claude Code from an orchestrator script is a very useful pattern, and it's not the first time I've run into this particular issue. If you've seen the Ralph projects doing the rounds of social media recently (e.g. https://github.com/mikeyobrien/ralph-orchestrator, https://github.com/michaelshimeles/ralphy), this is the same sort of idea.

## Basic invocation

The simplest way to call Claude Code non-interactively:

```bash
> claude -p "Say hello in exactly 15 words"
Hello there! I hope you are having a wonderful and productive day today, friend!
```

The `-p` (or `--print`) flag runs in non-interactive mode, prints the response, and exits. Combine it with `--output-format json` to get structured output you can parse, including tool calls and token usage.

```bash
> claude -p --output-format json --model sonnet "Say hello in exactly 15 words"
```
```json
{
  "type": "result",
  "subtype": "success",
  "is_error": false,
  "duration_ms": 3302,
  "duration_api_ms": 2918,
  "num_turns": 1,
  "result": "Hello! I'm Claude, your AI assistant here to help with your software engineering tasks.",
  "session_id": "3892c079-721c-4e63-b06b-38a0be87809b",
  "total_cost_usd": 0.03988395,
  "usage": {
    "input_tokens": 3,
    "cache_creation_input_tokens": 9443,
    "cache_read_input_tokens": 13829,
    "output_tokens": 21,
    "server_tool_use": {
      "web_search_requests": 0,
      "web_fetch_requests": 0
    },
    "service_tier": "standard",
    "cache_creation": {
      "ephemeral_1h_input_tokens": 9443,
      "ephemeral_5m_input_tokens": 0
    }
  },
  "modelUsage": {
    "claude-sonnet-4-5-20250929": {
      "inputTokens": 3,
      "outputTokens": 21,
      "cacheReadInputTokens": 13829,
      "cacheCreationInputTokens": 9443,
      "webSearchRequests": 0,
      "costUSD": 0.03988395,
      "contextWindow": 200000,
      "maxOutputTokens": 64000
    }
  },
  "permission_denials": [],
  "uuid": "6cf4cfcc-0bb6-463b-8864-2f9bba2dfca2"
}
```

## The tool permission gotcha

To use tools like `WebSearch`, `WebFetch`, `Bash`, or `Edit`, you need *both* flags:

```bash
claude -p \
  --tools "WebSearch,WebFetch" \
  --allowedTools "WebSearch,WebFetch" \
  "Search for Python tutorials"
```

| Flag | Purpose |
|------|---------|
| `--tools` | Makes tools *available* to the model |
| `--allowedTools` | Grants *permission* to use them without prompting |

If you only use `--tools`, the model will attempt to use them, but you'll get a cryptic error: "Claude requested permissions to use X, but you haven't granted it yet." The fix is simple once you know it, but the error message doesn't make the solution obvious.

To disable all tools entirely, pass an empty string:

```bash
claude -p --tools "" "Your prompt"
```

## System prompts on Windows: the hidden problem

Long or complex system prompts passed via `--system-prompt` can cause tool permissions to silently fail on Windows, even when `--allowedTools` is set correctly.

The workaround is to keep your system prompt minimal and put detailed instructions in the user prompt instead:

```python
# This causes issues with long instructions
cmd = [claude_path, "-p", "--system-prompt", very_long_instructions, ...]

# This works reliably
system_prompt = "You are a research assistant. Use WebSearch immediately."
user_prompt = f"{detailed_instructions}\n\n---\n\nNow do this: {task}"

proc = subprocess.Popen(cmd, stdin=subprocess.PIPE, ...)
stdout, stderr = proc.communicate(input=user_prompt)
```

This may be related to Windows command-line argument length limits or escaping issues, but my prompts were under the Windows arg length limit. I haven't dug into the root cause, this workaround is enough for me.

## Subprocess pattern

This pattern worked for reliable invocation:

```python
import subprocess
import sys

cmd = [
    claude_path, "-p",
    "--output-format", "json",
    "--model", "sonnet",
    "--system-prompt", short_system_prompt,
    "--tools", "WebSearch,WebFetch",
    "--allowedTools", "WebSearch,WebFetch",
]

creation_flags = 0
if sys.platform == "win32":
    creation_flags = subprocess.CREATE_NEW_PROCESS_GROUP

proc = subprocess.Popen(
    cmd,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    encoding="utf-8",
    creationflags=creation_flags,
)

stdout, stderr = proc.communicate(input=user_prompt, timeout=300)
```

Notes:

- `CREATE_NEW_PROCESS_GROUP` on Windows helps with process tree management and avoids some hanging issues
- Piping the user prompt via stdin sidesteps command-line length limits
- `encoding="utf-8"` is advisable on Windows - the default `cp1252` encoding can cause instability when Claude returns a non-ASCII character or emoji

## Parsing the JSON output

The CLI returns a JSON array with different entry types:

```python
import json

data = json.loads(stdout)
for entry in data:
    if entry.get("type") == "result":
        # Final result text
        print(entry.get("result"))
        # Token usage
        usage = entry.get("usage", {})
        print(f"Tokens: {usage.get('input_tokens')} in / {usage.get('output_tokens')} out")

    elif entry.get("type") == "assistant":
        # Check for tool calls
        for block in entry.get("message", {}).get("content", []):
            if block.get("type") == "tool_use":
                print(f"Tool called: {block.get('name')}")

    elif entry.get("type") == "user":
        # Tool results come back as user messages
        for block in entry.get("message", {}).get("content", []):
            if block.get("type") == "tool_result":
                print(f"Tool result: {block.get('content')}")
```

## Agentic mode

For tasks requiring multiple tool calls, use `--max-turns`:

```bash
claude -p --max-turns 5 --tools "WebSearch,WebFetch" --allowedTools "WebSearch,WebFetch" "Research this topic thoroughly"
```

The model will loop through tool calls up to the specified limit before returning its final response.

## Security note

If you're using `--allowedTools "Bash"` or `--allowedTools "Edit"`, you're giving Claude auto-approved access to run arbitrary commands or modify files. This is fine when you control the prompt, but if any part of the user prompt comes from an untrusted source (a web form, external API, etc.), you've created a remote code execution vector. An attacker could inject instructions to wipe the filesystem. Treat `--allowedTools` the same way you'd treat `sudo` - don't grant it to untrusted input.

## Quick reference

| Flag | Example | Purpose |
|------|---------|---------|
| `-p` | `-p` | Non-interactive print mode |
| `--model` | `--model opus` | Select model (sonnet, opus, haiku, or full IDs) |
| `--output-format` | `--output-format json` | Output format (text, json, stream-json) |
| `--system-prompt` | `--system-prompt "Be concise"` | Custom system prompt (keep short on Windows!) |
| `--tools` | `--tools "Bash,Edit"` | Available tools |
| `--allowedTools` | `--allowedTools "Bash,Edit"` | Permitted tools |
| `--max-turns` | `--max-turns 10` | Agentic turn limit |
| `--timeout` | `--timeout 300000` | Timeout in milliseconds |

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| "Claude requested permissions but you haven't granted it" | Missing `--allowedTools` | Add `--allowedTools` matching your `--tools` |
| Tools silently fail with long system prompt | Windows CLI argument handling | Move instructions to user prompt via stdin |
| Process hangs on timeout | Pipe handle inheritance issues | Use `CREATE_NEW_PROCESS_GROUP` on Windows |
| Unicode errors in output | Missing encoding specification | Add `encoding="utf-8"` to Popen |

