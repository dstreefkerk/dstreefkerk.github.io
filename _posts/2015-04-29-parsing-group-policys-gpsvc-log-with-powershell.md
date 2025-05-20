---
layout: post
title: "Parsing Group Policy's gpsvc.log with PowerShell"
date: 2015-04-29
categories: powershell windows
tags: group-policy powershell regex select-string
author: "Daniel Streefkerk"
excerpt: "A PowerShell function to parse Group Policy's gpsvc.log file, making it easier to troubleshoot Group Policy issues by extracting process ID, thread ID, timestamp, and message details."
---

> **Note:** This article was originally written in April 2015 and is now over 10 years old. Due to changes in Windows, PowerShell, and Group Policy over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

Troubleshooting a Group Policy issue this week has had me [enabling the Group Policy debug log](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/applying-group-policy-troubleshooting-guidance) and trying to make head or tail of it.

The log looks like this:

```powershell
GPSVC(23ec.341c) 23:25:15:981 Waiting for machine group policy thread to terminate.
GPSVC(23ec.341c) 23:25:16:008 CGPService::KeepServiceAlive: Beginning WaitForSingleObject.
GPSVC(23ec.341c) 23:25:16:031 CGPService::KeepServiceAlive: Completed WaitForSingleObject.
GPSVC(23ec.341c) 23:25:16:061 machine group policy thread has terminated.
GPSVC(23ec.341c) 23:25:16:091 CGroupPolicySession::CleanupEnvironment:--
GPSVC(23ec.341c) 23:25:16:573 Deleting machine
GPSVC(1478.1d60) 23:50:10:565 CGPNotify::OnNotificationTriggered: SetEvent = 0x30c
GPSVC(1478.1d60) 23:50:10:595 CGPNotify::OnNotificationTriggered: SetEvent = 0x288
GPSVC(1478.1d60) 23:50:10:631 CGPNotify::OnNotificationTriggered: SetEvent = 0x38c
GPSVC(1478.1d60) 23:50:10:674 CGPNotify::OnNotificationTriggered: SetEvent = 0x3cc
```

Since the log includes timestamps, I thought it would be a good idea to parse the log with PowerShell so that I could do some comparisons between log entries. I started down this path, and [then discovered](https://techcommunity.microsoft.com/t5/ask-the-directory-services-team/a-treatise-on-group-policy-troubleshooting-8211-now-with-gpsvc/ba-p/400304) later on that there are multiple processes, and multiple threads, all logging to the same log file. That leads to entries being a bit all over the place.

Regardless of that, I soldiered on - more for the sake of experimentation, and ended up with a function that will parse each line from the log into a custom PSObject containing process ID, thread ID, timestamp, and the message:

```powershell
function Get-GPLog {
    if (!(Test-Path "$env:windir\debug\usermode\gpsvc.log")) { Write-Error "Can't access gpsvc.log"; return }

    # Our regular expression to capture the parts of each log line that are of interest
    $regex = "^GPSVC\((?<pid>[0-9A-Fa-f]+)\.(?<tid>[0-9A-Fa-f]+)\) (?<time>\d{2}:\d{2}:\d{2}:\d{3}) (?<message>.+)$"

    # Get the content of gpsvc.log, ensuring that blank lines are excluded
    $log = Get-Content $env:windir\debug\usermode\gpsvc.log -Encoding Unicode | Where-Object {$_ -ne "" } 

    # Loop through each line in the log, and convert it to a custom object
    foreach ($line in $log) {
        # Split the line, using our regular expression
        $matchResult = $line | Select-String -Pattern $regex

        # Split up the timestamp string, so that we can convert it to a DateTime object
        $splitTime = $matchResult.Matches.Groups[3].Value -split ":"

        # Create a custom object to store our parsed data, and put it onto the pipeline.
        # Note that we're also converting the hex PID and TID values to decimal
        [pscustomobject]@{
            process_id = [System.Convert]::ToInt32($matchResult.Matches.Groups[1].Value,16);
            thread_id = [System.Convert]::ToInt32($matchResult.Matches.Groups[2].Value,16);
            time = (Get-Date -Hour $splitTime[0] -Minute $splitTime[1] -Second $splitTime[2] -Millisecond $splitTime[3])
            message = $matchResult.Matches.Groups[4].Value
        }
    }
}
```

From there, it's easy to pipe the output to Group-Object or Where-Object to narrow down your search.

```powershell
# Get all log entries, and group by process ID
Get-GpLog | Group-Object process_id | Sort-Object -Property Count -Descending

# or

# Get all log entries for a specific process ID
Get-GpLog | Where-Object {$_.process_id -eq 11280}
```

> There's one caveat with this whole process, and that's the fact that the gpsvc.log entries don't contain a full date – only the time component. This means that all log entries parsed by this function appear to be "today".

I'm particularly happy that I got the regular expression working correctly. It ended up looking like this:

```
^GPSVC\((?<pid>[0-9A-Fa-f]+)\.(?<tid>[0-9A-Fa-f]+)\) (?<time>\d{2}:\d{2}:\d{2}:\d{3}) (?<message>.+)$
```

This regex, which I'm sure could be done much more efficiently by a regex guru, uses capturing groups to extract the various segments of each line. When combined with Select-String –Pattern, you get the following output:

We're not interested in the first group as it's returning the entire log line, but the subsequent groups are exactly what we're after:

1. PID
2. TID
3. Timestamp
4. Message

There's a tool that already does this and more. It's called SysProSoft Policy Reporter, and it's free. Download it [here](https://www.sysprosoft.com/policyreporter.shtml).

Additionally, [gpresult](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc733160(v=ws.11)) outputs some great stuff when you use /H, just ensure that it's run with elevated privileges.

> **Note on security:** When enabling verbose Group Policy debug logging as described in this post, remember to turn it off when you're done troubleshooting by setting the GPSvcDebugLevel registry value back to 0. Leaving debug logging enabled indefinitely can fill your disk space and potentially expose sensitive configuration details in the logs.
> 
> **Note on obsolescence:** As of 2025, modern Windows systems may offer enhanced Group Policy troubleshooting tools. Check Microsoft's documentation for the latest recommended approaches, such as using the built-in Event Viewer's Group Policy operational logs. The PowerShell script in this post still demonstrates useful regex techniques but may need adjustments for current Windows versions.