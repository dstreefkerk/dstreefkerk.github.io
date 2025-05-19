---
layout: post
title: "Calling Claude Code from PowerShell via WSL — Without Breaking NVM or Losing Context"
date: 2025-05-20
categories: [powershell, wsl, node, tools]
tags: [claude-code, anthropic, nvm, bash, cli, windows]
author: "Daniel Streefkerk"
excerpt: "How to reliably call Claude Code CLI from Windows PowerShell when it's installed in WSL under Node.js via NVM, preserving context and working directory."
---
# Using Claude Code's SDK from Windows PowerShell

Anthropic just announced the [Claude Code SDK](https://docs.anthropic.com/en/docs/claude-code/sdk) overnight, although it's already been available for a while in preview.

I was using Claude Code CLI "SDK" in Windows Subsystem for Linux (WSL) yesterday for some automation work, and I got curious as to whether I could access Claude via Windows PowerShell within Windows itself. That would save me the step of having to fire up a WSL console, and would also allow me to write PowerShell scripts that leverage Claude Code.

Note: Claude Code doesn't yet support being installed on a Windows box, so it's necessary to use WSL to get it running in the first place. Using the method described below will still run WSL in the background as soon as wsl.exe is executed for the first time.

## The challenge

Straightforward in theory:

```powershell
wsl claude --help
```

In reality:

```
/bin/bash: line 1: claude: command not found
```

Trying the full path instead:

```powershell
wsl /home/daniel/.nvm/versions/node/v18.20.8/bin/claude
```

That failed too:

```
/usr/bin/env: 'node': No such file or directory
```

The shebang was intact, but `node` wasn't resolving — which pointed to the environment not being properly initialised.

> **What's a shebang?**
>
> Hint: It has nothing to do with Ricky Martin
>
> A *shebang* (from `#!`, pronounced "sha-bang") is the first line of a script that tells the operating system which interpreter to use. For example:
>
> ```bash
> #!/usr/bin/env node
> ```
>
> This tells the shell to use the version of `node` found in your `$PATH`. If your environment isn't initialised properly (like when NVM hasn't been sourced), this fails.

## The culprit: NVM and non-login shells

If you've used NVM in WSL before, this won't surprise you. WSL won't source your `~/.bash_profile` or `nvm.sh` unless you explicitly invoke a login shell.

In other words, none of the following will load NVM:

```powershell
wsl node -v
wsl claude --help
```

But this will:

```powershell
wsl bash -lc "node -v"
```

So will this:

```powershell
wsl bash -lc "claude --help"
```

As long as your login shell actually sources NVM properly.

## Fixing the environment

I added the following to my `~/.bash_profile`:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

Once that was in place, calling `claude` from PowerShell like this worked reliably:

```powershell
wsl bash -lc "claude --help"
```

## What about working directory?

You'd expect `bash -lc` to drop you into your WSL home directory (`/home/daniel`) regardless of where PowerShell was launched from.

Turns out, WSL preserves the calling path from Windows, even when starting a login shell, which I thought was pretty cool:

```powershell
PS C:\> wsl bash -lc "pwd"
/mnt/c
```

So if you're sitting in `C:\`, the WSL side runs from `/mnt/c`.

## Making it feel native

To make proper use of this, I added a simple function to my PowerShell profile file:

```powershell
function claude {
    # Join all arguments with proper escaping
    $joinedArgs = ($args | ForEach-Object { 
        # Double quote each argument and escape inner quotes
        """" + ($_ -replace '"', '\"') + """"
    }) -join " "

    # Build the command with proper escaping for WSL
    $commandToRun = "claude $joinedArgs"

    # Run the command in WSL with proper quoting
    wsl bash -lc "$commandToRun"
}
```

With that in place, I can just run:

```powershell
claude -p "Which directory are you running out of?"
```

…and get exactly what I'd expect:

```
/mnt/c/Users/Daniel
```

This might not be the perfect solution, but it's a handy workaround until we get native Windows support in Claude Code.