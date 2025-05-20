---
layout: post
title: "Apply GPO based on installed Server Feature (WMI Filtering)"
date: 2014-10-08
categories: [technology, windows]
tags: [active-directory, group-policy, systems-administration, wmi]
author: "Daniel Streefkerk"
excerpt: "How to use WMI filtering to apply Group Policy Objects to servers based on installed features, improving GPO organization and targeting."
---

> **Note:** This article was originally written in October 2014 and is now over 10 years old. WMI filtering remains a powerful technique for targeting GPOs based on system characteristics, however due to this article's age, please consider this guide as a conceptual reference rather than a current implementation guide.

Today I came across a server that had been placed in a sub-OU by a colleague simply for the purposes of applying a GPO to it. The GPO in question was configured to make some changes to the BranchCache feature.

If the policy needs to apply to a subset of all servers in an OU based on installed features, it would be cleaner to apply a [WMI filter](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj717288(v=ws.11)) to the GPO itself rather than limiting the scope of the GPO by explicit security filtering.

Here's what I did to clean it up:

1. Created a WMI filter in GPMC:
   
   ```
   SELECT * FROM Win32_ServerFeature WHERE Name like 'branchcache%'
   ```
2. Applied the filter to the GPO in question
3. Applied the GPO to the OU where the server originally lived
4. Moved the server back to the original OU

This same strategy could be used to apply a policy to all IIS servers, all file servers, etc. The possibilities are practically limitless.

## Security Considerations

When implementing WMI filters:

1. Be cautious with complex WMI queries as they can slow down Group Policy processing
2. WMI filters run in the security context of the computer account, so ensure appropriate permissions
3. Test all WMI filters thoroughly before deployment to production environments