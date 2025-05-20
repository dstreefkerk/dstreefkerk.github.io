---
layout: post
title: "PowerShell: Provide a listing and count of computers in an OU structure"
date: 2015-01-21
categories: [powershell, active-directory]
tags: [powershell, active-directory, systems-administration]
author: "Daniel Streefkerk"
excerpt: "How to use PowerShell to quickly determine the number of computer accounts in Active Directory by location in your OU structure."
---

> **Note:** This article was originally written in January 2015 and is now over 10 years old. Due to changes in Active Directory and PowerShell over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

Today I had to quickly figure out how many server computer accounts we had in AD. We have an OU structure like domain/servers/location1, domain/servers/location2, etc.

This is how I did it:

```powershell
# Assumes a CanonicalName of server accounts like example.org/servers//
# This groups the results of Get-ADComputer on the element of the CName
Get-ADComputer -SearchBase "OU=servers,DC=example,DC=org" -Filter 'Enabled -eq $true' -Properties CanonicalName | Group-Object {($_.CanonicalName -Split "/")[2]}
```

That results in the following output:

```
Count Name                      Group
----- ----                      -----
   43 Sydney                    {Computer object data}
    8 Adelaide                  {Computer object data}
    8 Brisbane                  {Computer object data}
    5 Hobart                    {Computer object data}
   17 Perth                     {Computer object data}
   39 Melbourne                 {Computer object data}
   10 Canberra                  {Computer object data}
```

## May 2017 update

I noticed that some people are hitting this site when googling "*how to get count of computers in AD in an OU*".

Counting the number of computer objects in a single OU is simple:

```powershell
Get-ADComputer -SearchBase "OU=Computers,DC=contoso,DC=com" -Filter * -Properties Name | Measure-Object
```

Here is how I'd count the number of computer objects in multiple OUs that aren't under a common parent OU:

```powershell
$orgUnits = 'OU=Workstations,DC=contoso,DC=com','OU=Servers,DC=contoso,DC=com'
$orgUnits | ForEach-Object {Get-ADComputer -SearchBase $_ -Filter * -Properties Name} | Measure-Object
```

## 2025 update

Since this post was originally written, several PowerShell and Active Directory best practices have evolved:

1. For modern PowerShell versions (5.1+), use string syntax for filters instead of script blocks:
   
   ```powershell
   # Modern approach
   Get-ADComputer -SearchBase "OU=servers,DC=example,DC=org" -Filter 'Enabled -eq $true' -Properties CanonicalName
   ```

2. For more robust OU path handling, consider using a more flexible approach to handle varying OU depths:
   
   ```powershell
   Get-ADComputer -SearchBase "OU=servers,DC=example,DC=org" -Filter 'Enabled -eq $true' -Properties CanonicalName | 
   Group-Object {
       $locationPart = $_.CanonicalName -Split "/"
       # Get the location segment, which might be at different position depending on your OU structure
       if ($locationPart.Count -gt 2) { $locationPart[2] } else { "Unknown" }
   }
   ```

3. Always include error handling for production scripts:
   
   ```powershell
   try {
       Get-ADComputer -SearchBase "OU=servers,DC=example,DC=org" -Filter 'Enabled -eq $true' -Properties CanonicalName -ErrorAction Stop |
       Group-Object {($_.CanonicalName -Split "/")[2]}
   } catch {
       Write-Error "Failed to query AD: $_"
   }
   ```

4. For security best practices, consider:
   
   - Running these commands against a Read-Only Domain Controller when possible
   - Using the principle of least privilege (dedicated service accounts with minimal permissions)
   - Validating OU paths before querying
   - Implementing proper logging for auditing purposes