---
layout: post
title: "Making Claude Code Work for You: An Introduction to Hooks"
date: 2025-07-02
categories: [tools, automation, ai]
tags: [claude-code, anthropic, python, automation, hooks, development-workflow]
author: "Daniel Streefkerk"
excerpt: "How to use Claude Code hooks to automate the tedious bits of development workflow, ensuring consistent formatting, linting, and quality checks without manual intervention."
---

You know the drill. Claude writes some Python code, it works eventually, but the formatting is all over the place. Or it creates a file but doesn't run your linter. Or it makes changes without updating your pre-commit hooks. These aren't deal-breakers, but they're the kind of friction that adds up over a day of development work.

Anthropic [recently released](https://docs.anthropic.com/en/docs/claude-code/hooks) the ability to define hooks. Claude Code hooks are essentially shell commands that run automatically at specific points in Claude's workflow. Think of them as your personal automation layer that ensures certain things always happen, rather than hoping Claude remembers to do them.

How many times have you committed instructions to Claude's memory, but it wholesale ignores them?!

## What Are Hooks, Really?

In practical terms, hooks are shell commands that execute when specific events occur in Claude Code:

- **PreToolUse**: Before Claude runs a tool (perfect for validation or permission checks)
- **PostToolUse**: After Claude completes a tool (ideal for formatting, linting, or cleanup)
- **Notification**: When Claude sends you a notification
- **Stop**: When Claude finishes responding

The best part is that these aren't mere suggestions for Claude to consider, they're guaranteed to run. If you want every Python file formatted with [Black](https://black.readthedocs.io/en/stable/index.html), it will happen. Every time.

## A Real-World Example: Automatic Python Formatting

Let me walk you through setting up a hook that automatically formats Python code with Black. This is the kind of thing that seems trivial until you're working on a project for hours and realising half your files are inconsistently formatted.

### The Setup

First, you'll need a formatting script. In this case, I'm keeping mine in `.claude/scripts/` within the project, so it's version-controlled and shared across the various machines on which I maintain this particular project:

```bash
#!/bin/bash
# .claude/scripts/format-python.sh

# Read JSON input from stdin
input_json=$(cat)

# Extract file path from the JSON input
file_path=$(echo "$input_json" | jq -r '.tool_input.file_path // empty')

# Check if we got a valid file path
if [[ -z "$file_path" ]]; then
    exit 0
fi

# Check if the file is a Python file
if [[ "$file_path" =~ \.py$ ]]; then
    # Check if black is installed
    if ! command -v black &> /dev/null; then
        echo "Warning: black is not installed. Install with: pip install black" >&2
        exit 1
    fi
    
    # Check if file exists
    if [[ ! -f "$file_path" ]]; then
        echo "Warning: File $file_path does not exist" >&2
        exit 1
    fi
    
    # Run black on the Python file
    if black "$file_path" --quiet; then
        echo "Formatted Python file: $file_path"
        exit 0
    else
        echo "Error: Failed to format $file_path with black" >&2
        exit 1
    fi
else
    exit 0
fi
```

Then you configure the hook in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "./.claude/scripts/format-python.sh"
          }
        ]
      }
    ]
  }
}
```

### Why This Approach Works

The hook receives JSON data about what Claude just did, including the exact file path. Using `jq` to parse this gives you surgical precision - you're only formatting the specific file that was just modified, not scanning the entire filesystem or running formatters unnecessarily.

When I tested this setup, I asked Claude to create a deliberately poorly formatted Python file:

```python
def test(x,y):
    if x>y:return x
    else:return y
```

The hook then automatically transformed it into proper Black-formatted code:

```python
def test(x, y):
    if x > y:
        return x
    else:
        return y
```

No manual intervention required. No remembering to run Black afterwards. It just works.

## The Bigger Picture

This example scratches the surface of what's possible. You could set up hooks for:

- **Code quality**: Running linters, type checkers, or security scanners
- **Notifications**: Custom alerts when Claude modifies sensitive files
- **Logging**: Tracking all commands for compliance or debugging
- **Permissions**: Blocking modifications to production code or critical directories
- **Integration**: Triggering CI/CD pipelines or updating documentation

The key insight is that hooks turn suggestions into guarantees. Instead of prompting Claude to "remember to format the code," you encode that requirement at the application level.

## Security Considerations

Hooks execute with your full user permissions, so they come with the usual security considerations. Review any hook commands carefully - you're essentially giving them automatic execution rights.

Also, hooks use a 60-second timeout and run in parallel, so keep them focused and efficient. The formatting hook above typically completes in milliseconds, which is exactly what you want.

## Getting Started

If you're already using Claude Code, try the `/hooks` slash command to see the configuration interface. Start simple - maybe just logging the commands Claude runs - and build up from there.

In my experience, the most valuable hooks are the ones that eliminate small, repetitive tasks that you'd otherwise forget about until code review time. Automatic formatting is a perfect example, but there's probably something specific to your workflow that would benefit from this approach.

The [documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) is thorough, and the JSON input structure is well-designed for extracting exactly the information you need. It's worth spending some time with hooks - they're one of those features that becomes indispensable once you start using them properly.