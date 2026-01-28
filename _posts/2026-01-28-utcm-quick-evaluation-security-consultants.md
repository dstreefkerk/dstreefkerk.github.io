---
layout: post
title: "UTCM: Quick Evaluation for Security Consultants"
date: 2026-01-28
categories: [microsoft, security]
tags: [utcm, microsoft-365, graph-api, security-assessment, scubagear, microsoft365dsc]
author: "Daniel Streefkerk"
excerpt: "Microsoft's new Unified Tenant Configuration Management (UTCM) looks promising for drift monitoring, but doesn't fit the bill for point-in-time security assessments."
---

> **Note:** Check [current Microsoft documentation](https://learn.microsoft.com/en-us/graph/unified-tenant-configuration-management-concept-overview) before relying on any of this.

Microsoft's Unified Tenant Configuration Management (UTCM) is a new Graph API family (`graph.microsoft.com/beta/`) for reading M365 tenant configuration. It does two things:

- Snapshot APIs capture current configuration state on demand
- Monitoring APIs compare current state against a baseline every 6 hours and flag differences

It covers 300+ resource types across Intune, Exchange Online, Teams, Entra ID, Purview, and Defender. No SharePoint or OneDrive support yet.

## Skip it for point-in-time assessments

I've briefly looked at whether UTCM could be used to facilitate point-in time (external) security assessments. Short answer: no. It was built for continuous drift monitoring by internal teams, not configuration pulls for external review.

### The problems

- Your app needs Graph permissions, and then the UTCM service principal needs separate workload permissions. That's a lot of setup for a short engagement.
- Snapshots auto-delete after 7 days whether you've exported them or not. Awkward when you're documenting findings for a client report.
- No compliance mapping. Raw configuration only, no CIS or NIST alignment. You're mapping everything yourself, or building a new tool to do that for you.
- It's a beta API. No SLA, no production support. Putting unsupported tooling in client deliverables is asking for trouble when something breaks.
- The 20,000/month API quota means large tenants can blow through their limit in a single comprehensive pull.
- No published guidance for GCC/GCC-High/DoD. If you're working with government clients, this is a non-starter.

## Where UTCM might actually fit

For internal security teams doing ongoing configuration management, drift monitoring could be useful. Knowing when someone changes a Conditional Access policy or tweaks Intune configuration without going through change management would be valuable, but the jury is out on whether you want to alert on that via a different method than your existing Sentinel workspace (or other SIEM solution).
