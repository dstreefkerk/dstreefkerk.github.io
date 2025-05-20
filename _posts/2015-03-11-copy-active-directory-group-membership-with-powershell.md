---
layout: post
title: "Copy Active Directory Group Membership with PowerShell"
date: 2015-03-11
categories: [powershell, active-directory]
tags: [active-directory, powershell, systems-administration]
author: "Daniel Streefkerk"
excerpt: "A quick PowerShell snippet to copy group memberships from one user or computer account to another in Active Directory."
---

> **Note:** This article was originally written in March 2015 and is now over 10 years old. Due to changes in Active Directory and PowerShell over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

If you need to copy AD group memberships from one user or computer account to another, you can do so with the following two lines of PowerShell.

This depends on the [ActiveDirectory module](https://learn.microsoft.com/en-us/powershell/module/activedirectory/) being loaded, or auto-loaded in PS 3.0 or newer.

```powershell
# Get the memberships from the source computer account
$memberships = Get-ADComputer source -Properties memberof | Select-Object -ExpandProperty memberof

# Apply the memberships to the destination computer account
Get-ADComputer destination | Add-ADPrincipalGroupMembership -MemberOf $memberships -Verbose -WhatIf
```

Remove the `-WhatIf` parameter when you're satisfied with the previewed changes and ready to apply them.

> **Security Note:** When copying group memberships, ensure you understand the permissions being granted. Some groups may provide elevated privileges that could pose security risks if assigned inappropriately. Always follow the principle of least privilege and verify the memberships before applying them in production environments.

> **Update Note:** Modern Active Directory environments may use more advanced identity management solutions and privileged access management (PAM) systems. Consider reviewing your organisation's current identity management practices before implementing this script.