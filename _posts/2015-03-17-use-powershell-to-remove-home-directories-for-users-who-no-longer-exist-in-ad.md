---
layout: post
title: "Use PowerShell to remove home directories for users who no longer exist in AD"
date: 2015-03-17
categories: 
  - powershell
  - active-directory
tags:
  - powershell
  - active-directory
  - system-administration
  - file-management
author: "Daniel Streefkerk"
excerpt: "A PowerShell script to identify and move home directories for users who no longer exist in Active Directory, helping to clean up file shares during server migrations."
---

> **Note:** This article was originally written in March 2015 and is now over 10 years old. Due to changes in PowerShell and Active Directory management tools over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

Today I had to migrate a home directory share to another server. I didn't want to migrate folders for users who no longer existed in AD or were disabled, so I wrote a script to move those users' folders into another location.

You could tweak this to be run as a scheduled task, thereby keeping your user home directory share clear of old users' folders.

Note that this requires the ActiveDirectory PowerShell module in order to enumerate the list of users from AD. I started looking at a fallback method using [adsisearcher], but it wasn't worth my time.

> Note: 2 years later and at a different employer, and I built one using [adsisearcher] to remove user profile folders (not home directories) because I wanted it to run locally on a file server without the requirement to have the AD PowerShell module installed. See the bottom of the page for that update.

```powershell
$homeDriveRoot = "\\server1\userfolders\"
$leaversRoot = "\\server1\userfolders\oldusers\"

# Get the list of folders in the home drive share
$folders = Get-ChildItem $homeDriveRoot | Select-Object -ExpandProperty Name

# Get the list of active users from AD
$activeUsers = Get-ADUser -Filter {Enabled -eq $true} | Select-Object -ExpandProperty SamAccountName

# Compare the list of users to the list of folders
$differences = Compare-Object -ReferenceObject $activeUsers -DifferenceObject $folders | Where-Object {$_.SideIndicator -eq "=>"} | Select-Object -ExpandProperty InputObject

# For each folder that shouldn't exist, move it
$differences | ForEach-Object {Move-Item -Path "$homeDriveRoot$_" -Destination "$leaversRoot$_" -Force}
```

## 2017 update

Here's a version using ADSI instead of Active Directory PowerShell that I wrote to remove old roaming user profiles. Note that you'll need to remove the -WhatIf from the last line to have it actually delete folders.

```powershell
$profilesFolder = 'D:\path\to\profiles'
$profiles = Get-ChildItem $profilesFolder

foreach ($roamingProfile in $profiles) {
 # Split on the dot, because of .V2 and .V4 folders
 $username = $roamingProfile.Name.Split('.')[0]

 # Find a matching user using ADSI
 $matchingUser = ([ADSISEARCHER]"samaccountname=$($username)").Findone()

 # Skip this folder if we DO find a matching user
 if ($matchingUser -ne $null) { continue }

 # Remove the folder
 $roamingProfile | Remove-Item -Recurse -Force -Verbose -WhatIf
}
```

## 2025 update: Modern considerations

Since this post was written, several PowerShell and Active Directory best practices have evolved:

1. For modern PowerShell versions (5.1+), use string syntax for filters instead of script blocks:
   
   ```powershell
   # Modern approach - string filter instead of script block
   $activeUsers = Get-ADUser -Filter 'Enabled -eq $true' | Select-Object -ExpandProperty SamAccountName
   ```

2. Add proper error handling for production scripts:
   
   ```powershell
   try {
       $folders = Get-ChildItem $homeDriveRoot -ErrorAction Stop | Select-Object -ExpandProperty Name
       $activeUsers = Get-ADUser -Filter 'Enabled -eq $true' -ErrorAction Stop | 
                      Select-Object -ExpandProperty SamAccountName
   } catch {
       Write-Error "Failed to retrieve folders or AD users: $_"
       exit
   }
   ```

3. Consider additional safeguards like:
   
   ```powershell
   # Preview changes before execution
   $differences | ForEach-Object {
       Write-Host "Would move: $homeDriveRoot$_ to $leaversRoot$_"
   }
   
   # Prompt for confirmation before executing moves
   $confirmation = Read-Host "Move these folders? (y/n)"
   if ($confirmation -ne 'y') {
       Write-Host "Operation cancelled."
       exit
   }
   ```

4. Implement logging for tracking changes:
   
   ```powershell
   # Add to your script to log operations
   $logFile = "C:\Logs\HomeDirCleanup_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
   Start-Transcript -Path $logFile
   # [Your script here]
   Stop-Transcript
   ```

# 