---
layout: post
title: "Delete old log files with PowerShell"
date: 2015-08-12
categories: powershell automation
tags: powershell log-files file-management cleanup system-administration
author: "Daniel Streefkerk"
excerpt: "A PowerShell script to automatically find and delete log files older than a specified age, helping maintain disk space and system performance."
---

> **Note:** This article was originally written in August 2015 and is now over 9 years old. Due to changes in PowerShell and Windows management over time, the described solution may need adaptation for current environments. Please consider this guide as a conceptual reference rather than a current implementation guide.

Today I came across a folder full of log files created by an import/integration application on one of our servers. The folder contained over 50,000 files. Rather than manually delete the old logs, here's how I removed all items older than 1 month:

```powershell
Get-ChildItem "C:\Path_To_logs\*.txt" | Where-Object {$_.CreationTime -lt (Get-Date).AddMonths(-1)} | Remove-Item -Force -Verbose -WhatIf
```

Since PowerShell is based on the .NET Framework, you can use standard [System.DateTime methods](https://learn.microsoft.com/en-us/dotnet/api/system.datetime?view=net-9.0) when working with dates and times.

```powershell
PS C:\> Get-Date | Get-Member Add*


   TypeName: System.DateTime

Name            MemberType Definition
----            ---------- ----------
Add             Method     datetime Add(timespan value)
AddDays         Method     datetime AddDays(double value)
AddHours        Method     datetime AddHours(double value)
AddMilliseconds Method     datetime AddMilliseconds(double value)
AddMinutes      Method     datetime AddMinutes(double value)
AddMonths       Method     datetime AddMonths(int months)
AddSeconds      Method     datetime AddSeconds(double value)
AddTicks        Method     datetime AddTicks(long value)
AddYears        Method     datetime AddYears(int value)
```

Passing a negative value to one of the Add* methods will result in subtracting that amount of time:

```powershell
PS C:\> Get-Date
Wednesday, 12 August 2015 2:37:23 PM

PS C:\> (Get-Date).AddMonths(1)
Saturday, 12 September 2015 2:37:37 PM

PS C:\> (Get-Date).AddMonths(-1)
Sunday, 12 July 2015 2:37:40 PM
```

Obviously you need to be very careful with specifying a path to a command like Remove-Item. I've left the -WhatIf switch on the example code above.

It's easy to add something like this to a scheduled task to keep a log folder tidy.

> **2025 Update:**
> 
> **Modern Considerations:**
> 
> In newer PowerShell versions, you might prefer using LastWriteTime instead of CreationTime as it's more reliable for determining when a file was last modified:
> 
> ```powershell
> Get-ChildItem "C:\Path_To_logs\*.txt" | Where-Object {$_.LastWriteTime -lt (Get-Date).AddMonths(-1)} | Remove-Item -Force -Verbose -WhatIf
> ```
> 
> **Security Considerations:**
> 
> 1. Always review the results with -WhatIf before running the actual deletion
> 2. Consider archiving important logs before deletion
> 3. Ensure proper permissions when automating via scheduled tasks
> 4. For sensitive systems, implement logging of the deletion process itself
> 5. Test scripts thoroughly in non-production environments first