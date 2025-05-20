---
layout: post
title: "A quick one-liner to find out a Windows system's uptime"
date: 2013-05-20
categories: technology systems-administration
tags: windows command-line
author: "Daniel Streefkerk"
excerpt: "A simple one-liner command to check Windows system uptime from the command prompt, with an improved method suggested by a reader."
---

> **Note:** This article was originally written in May 2013 and is now over 11 years old. Due to changes in Windows operating systems over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

Based on the info I found [here](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/), this one-liner is a quick way of displaying a Windows system's uptime from the command prompt...

```
net stats srv | find "since"
```

Results in the following:

```
Statistics since 16/05/2013 2:32:48 PM
```

[Oleg](https://about.me/chw) left me a comment with a more accurate one:

```
systeminfo | find "Boot Time"
```

Just remember that [find.exe](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/find) is case-sensitive unless you use the /i parameter

The "net stats srv" method I originally posted is a little less accurate (by mere seconds) as it's based on when the Server service started:

*[Image removed during migration: Screenshot showing comparison between the two uptime methods with the systeminfo command displaying a more accurate boot time]*

Note: Edited to include a better, more accurate method

## Modern Alternatives (2025)

For those using newer Windows versions, here are some modern alternatives to check system uptime:

1. **Task Manager**: Open Task Manager (Ctrl+Shift+Esc), go to the Performance tab, and look for "Up time" at the bottom.

2. **PowerShell**: Use this command for detailed uptime information:
   
   ```
   (get-date) - (gcim Win32_OperatingSystem).LastBootUpTime
   ```

3. **Get-Uptime cmdlet**: In PowerShell 6.0 and later, which appears to just wrap the output of the above example:
   
   ```
   Get-Uptime
   ```

These methods are generally more reliable and provide more detailed information than the older command-line approaches mentioned above.