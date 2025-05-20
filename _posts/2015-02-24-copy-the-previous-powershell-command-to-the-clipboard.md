---
layout: post
title: "Copy the previous PowerShell command to the clipboard"
date: 2015-02-24
categories: powershell tips-and-tricks
tags: powershell systems-administration
author: "Daniel Streefkerk"
excerpt: "A quick PowerShell one-liner to copy the previous command to the clipboard for reuse in other windows"
---

> **Note:** This article was originally written in February 2015 and is now over 10 years old. That said, it still works.

I often have multiple PowerShell windows open; at least one for testing out commands, and then the ISE for writing scripts.

Here's a quick one-liner to copy the previous PowerShell command to clipboard:

```powershell
h -c 1 | select -exp commandline | clip
```

To elaborate on that, here's the version that doesn't use aliases:

```powershell
Get-History -Count 1 | Select-Object -ExpandProperty CommandLine | clip
```

[Microsoft Documentation: Get-History](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-history)

## Technical Updates

**Modern Alternative (PowerShell 5.0+):** Since PowerShell 5.0 (released in 2016), you can use the native `Set-Clipboard` cmdlet instead of `clip.exe`:

```powershell
Get-History -Count 1 | Select-Object -ExpandProperty CommandLine | Set-Clipboard
```

**Security Considerations:**
- On shared systems, copied credentials or tokens could be exposed to other users through clipboard history features
- In PowerShell 7+, consider using the `-AsOSC52` parameter with `Set-Clipboard` for more secure cross-terminal clipboard operations