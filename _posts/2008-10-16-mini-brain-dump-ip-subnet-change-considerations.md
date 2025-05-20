---
layout: post
title: "Mini Brain Dump: IP Subnet Change Considerations"
date: 2008-10-16
categories: technology systems-administration
tags: windows networking ip-subnetting systems-administration
author: "Daniel Streefkerk"
excerpt: "A checklist of important considerations when changing IP subnet ranges in a Windows network environment."
---

> **Note:** This article was originally written in October 2008 and is now over 16 years old. Due to changes in Windows networking technologies over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

I've been sitting on this post for a long time, and intended to write a more detailed description. 

Here are some things you may need to consider (outside of the obvious like DHCP scopes, DNS server settings, Firewall settings & rules, etc) when changing the IP range your Windows network operates on:

- TCP/IP Printer ports on print server
- Printer/Copier IP/DNS/SMTP settings
- Exchange allowed relay ranges
- Any copy/print accounting devices attached to copiers
- Monitoring host settings. Eg. Big Brother/Hobbit - Both client and server side, if not configured to use DNS in config files
- Server iLO IP addresses

Some steps for changing domain controller IP addresses. Do these first before any other important servers: 

1. Change IP
2. ipconfig /flushdns
3. ipconfig /registerdns
4. Either restart the Netlogon service, or run 'nltest /dsregdns'
5. Reboot

Disclaimer: This is by no means a complete list. Use these directions at your own risk.

## Security Considerations

- Updating firewall rules should be done carefully to avoid inadvertently exposing services
- Modern networks should implement network segmentation using VLANs in addition to IP subnetting
- Consider implementing 802.1X authentication when restructuring network access
- Domain controllers and other critical infrastructure should be on protected subnets with restricted access

## Outdated Tools and Techniques

- The tool 'nltest' is still available but PowerShell's 'Test-ComputerSecureChannel' offers more capabilities in modern environments
- Big Brother/Hobbit monitoring systems have largely been replaced by more modern monitoring solutions
- Modern Windows Server environments have better tools for managing IP configuration including PowerShell cmdlets and Windows Admin Center
- iLO interfaces now support DHCP and DNS configuration which can simplify management during subnet migrations