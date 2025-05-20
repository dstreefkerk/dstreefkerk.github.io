---
layout: post
title: "Braindump: ADDS NTP Configuration via GPO"
date: 2014-01-14
categories: [technology, windows]
tags: [active-directory, group-policy, systems-administration, windows]
author: "Daniel Streefkerk"
excerpt: "A quick reference for configuring Active Directory Domain Services time synchronization using Group Policy Objects with proper NTP settings."
---

> **Note:** This article was originally written in January 2014 and is now over 11 years old. Due to changes in Active Directory and Group Policy over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

There's a great article on the Microsoft AD team blog about configuring the authoritative time server automatically via group policy and WMI filters. This may save you from domain time sync issues if your PDC emulator role eventually ends up moving to a different server.

[Configuring an Authoritative Time Server with Group Policy Using WMI Filtering](https://techcommunity.microsoft.com/blog/askds/configuring-an-authoritative-time-server-with-group-policy-using-wmi-filtering/395806)

Their article covers how to set up the WMI filter, but doesn't address the settings for NTP. Those are listed in detail under [this support article](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/configure-authoritative-time-server).

These are the settings I've implemented in my GPO using Admin Templates->System->Windows Time Service:

```
Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Config]
"MaxNegPhaseCorrection"=dword:00000708
"MaxPosPhaseCorrection"=dword:00000708
"AnnounceFlags"=dword:00000005
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Parameters]
"Type"="NTP"
"ntpserver"="au.pool.ntp.org,0x1"
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpClient]
"Enabled"=dword:00000001
"SpecialPollInterval"=dword:00000384
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer]
"Enabled"=dword:00000001
```

> **2025 Update Notes:**
> 
> **Improved Practices:** For better reliability, consider using multiple NTP servers (e.g., 0.au.pool.ntp.org through 3.au.pool.ntp.org) instead of just au.pool.ntp.org.
> 
> **Security Considerations:** The configuration above lacks authentication for external time sources. In modern environments, consider using AnnounceFlag 0xA instead of 0x5 for more stable time synchronisation on unreliable connections.
> 
> **Virtualisation:** If your domain controllers are virtualised, ensure that Hyper-V Time Synchronisation services are disabled to prevent conflicts with domain time synchronisation.
> 
> **Windows Server 2016+:** Newer Windows Server versions support more accurate time synchronisation. Research "Windows Server Accurate Time" for modern implementations.