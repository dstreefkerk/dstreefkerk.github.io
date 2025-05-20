---
layout: post
title: "PowerShell: Finding folders that don't match a pattern"
date: 2015-08-13
categories: powershell system-administration
tags: powershell file-management regex pattern-matching
author: "Daniel Streefkerk"
excerpt: "A PowerShell technique to identify folders that don't follow a specific naming convention, including how to track down who created them."
---

> **Note:** This article was originally written in August 2015 and is now over 10 years old. Due to changes in PowerShell and Windows management over time, the described solution may need adaptation for current environments. Please consider this guide as a conceptual reference rather than a current implementation guide.

This week I came across an issue where some folders didn't match a naming convention that was required by another third-party system. Because of this, data wasn't being extracted from all of the incorrectly-named folders.

The naming convention in question goes like this: "{whatever} {4-digit-number}". Lots of folders were missing the relevant number at the end. 

To quickly identify which folders were wrongly-named, I used a regex that searches the end of a string for a space followed by four numbers. I then used that in conjunction with Where-Object:

```powershell
Get-ChildItem -Path | Where-Object {$_.Name -notmatch " \d{4}$"}
```

Then, so that I could identify who was naming new folders incorrectly, I then did this:

```powershell
Get-ChildItem -Path | Where-Object {$_.Name -notmatch " \d{4}$"} | Select-Object -Property Name,CreationTime,@{Name="Owner";Expression={(Get-Acl $_.FullName).Owner}}
```

I then piped the above command to Sort-Object, and exported the lot to a CSV file for the department in question to review:

```powershell
Get-ChildItem -Path | Where-Object {$_.Name -notmatch " \d{4}$"} | Select-Object -Property Name,CreationTime,@{Name="Owner";Expression={(Get-Acl $_.FullName).Owner}}| Sort-Object CreationTime -Descending | Export-Csv c:\temp\insolfolder.csv -NoTypeInformation
```

In all, a quick, easy, and repeatable way of getting a report out to the people who need to maintain the folder structure. It's possible to extend it further to just email the owners of each of the non-compliant folders on a set schedule.

## Modern PowerShell Considerations

Since writing this post originally, PowerShell has evolved significantly:

- The regex pattern used is still valid, but PowerShell 7+ offers more powerful pattern matching capabilities
- For better performance with large directory structures, consider using `[System.IO.Directory]::GetDirectories()` method
- More efficient filtering can be achieved using the `-Filter` parameter where applicable:
  
  ```powershell
  Get-ChildItem -Path $path -Directory | Where-Object {$_.Name -notmatch " \d{4}$"}
  ```