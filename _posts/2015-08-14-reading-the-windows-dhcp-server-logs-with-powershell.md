---
layout: post
title: "Reading the Windows DHCP Server logs with PowerShell"
date: 2015-08-14
categories: windows powershell
tags: dhcp powershell server-2012-r2 windows
author: "Daniel Streefkerk"
excerpt: "A PowerShell function to read and parse Windows DHCP server logs for troubleshooting IP assignment issues across various VLANs."
---

> **Note:** This article was originally written in August 2015 and is now over 9 years old. Due to changes in Windows Server and PowerShell over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

> 

> **Modern Alternatives:** Consider using the `Get-DhcpServerv4Log` cmdlet from the DhcpServer module in newer Windows Server versions, which provides more robust logging capabilities and additional filtering options.

I've been doing a bit of work with DHCP over the last week or so - specifically with troubleshooting IP assignment from various VLANs. I threw together a quick function to read the last (x) lines out of the current day's DHCP server log. For now, there's no support for reading the logs remotely. This needs to be run on the server itself.

Once you've [dot sourced](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_scripts?view=powershell-7.3#script-scope-and-dot-sourcing) the script, just call the function. By default, it will grab the last 20 lines out of the current day's log file.

```powershell
Get-DHCPServerLog
```

You can specify a day, and/or the number of lines to grab from the end of the log file:

```powershell
Get-DHCPServerLog -Lines 5 -Day mon
```

You can also pipe it to Select-Object:

```powershell
Get-DHCPServerLog | Select-Object Date,Time,Description,MAC*,IP* | Format-Table -AutoSize
```

Here's the source:

```powershell
function Get-DHCPServerLog {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$false)]
        [string]$Day,

        [Parameter(Mandatory=$false)]
        [int]$Lines=20
    )

    # Set default to the current day if not specified
    if (!$Day) {
        $Day = (Get-Date).DayOfWeek.ToString().Substring(0,3).ToLower()
    }

    # Get the log path from registry
    $DHCPServer = Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\DHCPServer\Parameters
    $LogPath = Join-Path -Path $DHCPServer.DatabasePath -ChildPath "DhcpSrvLog-$Day.log"

    # Check if log exists
    if (!(Test-Path $LogPath)) {
        Write-Error "DHCP Server log for $Day not found at $LogPath"
        return
    }

    # Get last x lines and parse them
    $LogEntries = Get-Content $LogPath -Tail $Lines | ForEach-Object {
        if ($_ -match "^(\d\d|\s\d)\.(\d\d|\s\d)\.(\d\d|\s\d)\s(\d\d|\s\d):(\d\d|\s\d):(\d\d|\s\d)\s([^\s]+)\s(.+)") {
            $Description = $Matches[8]

            # Extract IP and MAC where possible
            $IP = if ($Description -match "\s(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})[\s,]") { $Matches[1] } else { $null }
            $MAC = if ($Description -match "\s([0-9a-f]{2}[:-][0-9a-f]{2}[:-][0-9a-f]{2}[:-][0-9a-f]{2}[:-][0-9a-f]{2}[:-][0-9a-f]{2})[\s,]") { $Matches[1] } else { $null }

            # Create custom object with parsed data
            [PSCustomObject]@{
                Date = "$($Matches[1]).$($Matches[2]).$($Matches[3])"
                Time = "$($Matches[4]):$($Matches[5]):$($Matches[6])"
                ID = $Matches[7]
                Description = $Description
                IPAddress = $IP
                MACAddress = $MAC
            }
        }
    }

    return $LogEntries
}
```

> 