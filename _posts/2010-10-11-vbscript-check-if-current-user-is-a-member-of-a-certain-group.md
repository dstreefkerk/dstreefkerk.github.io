---
layout: post
title: "Updated: VBScript - Check if current user is a member of a certain group"
date: 2010-10-11
categories: technology scripting
tags: scripting systems-administration windows vbscript powershell
author: "Daniel Streefkerk"
excerpt: "A VBScript solution to check if the current user is a member of a specified Active Directory group."
updated: 2025-05-20
---

> **Note:** This article was originally written in October 2010 and is now over 14 years old. Due to changes in Windows scripting technologies over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

I found some code to do this out on the net the other day. I've modified it a little, and added the part that checks environment variables.

**Note 3:** 30th May 2014 - This post is consistently one of the most popular posts on my blog. By now, you should be looking at PowerShell to replace VBScript. The 40-odd lines of VBScript below can be replaced by a single line of PowerShell:

```powershell
# Check if current user is a member of a specific group
Get-ADPrincipalGroupMembership $env:USERNAME | Where-Object {$_.Name -eq "GroupNameToCheckGoesHere"}

# Alternative method (if AD module is not available)
([ADSISearcher]"(&(objectCategory=User)(samAccountName=$env:USERNAME))").FindOne().Properties.memberof -like "*GroupNameToCheckGoesHere*"
```

**Note 2**: 21st February 2013 – I've updated the script so that it will work with Option Explicit. People who used this would see every check returning "true". Thanks to "Zounder1" for making me aware of this.

**Note**: 11th October 2010 – Since this post is so popular, I've cleaned up the code a bit and re-posted it below.

```vbscript
Option Explicit
Dim objShell,grouplistD,ADSPath,userPath,listGroup
On Error Resume Next

set objShell = WScript.CreateObject( "WScript.Shell" )

'Calls the isMember function with the specified group to see if the current user
' is a member of that group.
If isMember("GroupNameToCheckGoesHere") Then
       'MsgBox("Is member") ' Do something here if they are a member of the group
    Else
       'MsgBox("Is not member") ' Do something here if they are not a member of the group
End If

' *****************************************************
'This function checks to see if the passed group name contains the current
' user as a member. Returns True or False
Function IsMember(groupName)
    If IsEmpty(groupListD) then
        Set groupListD = CreateObject("Scripting.Dictionary")
        groupListD.CompareMode = 1
        ADSPath = EnvString("userdomain") & "/" & EnvString("username")
        Set userPath = GetObject("WinNT://" & ADSPath & ",user")
        For Each listGroup in userPath.Groups
            groupListD.Add listGroup.Name, "-"
        Next
    End if
    IsMember = CBool(groupListD.Exists(groupName))
End Function
' *****************************************************

' *****************************************************
'This function returns a particular environment variable's value.
' for example, if you use EnvString("username"), it would return
' the value of %username%.
Function EnvString(variable)
    variable = "%" & variable & "%"
    EnvString = objShell.ExpandEnvironmentStrings(variable)
End Function
' *****************************************************

' Clean up
Set objShell = Nothing
```

**Security Considerations:**

- This script uses "On Error Resume Next" which can hide important errors. In production environments, implement proper error handling.
- The VBScript approach shown here doesn't support modern authentication methods or multi-forest environments.

**Outdated Elements:**

- VBScript is considered a legacy technology by Microsoft. PowerShell is the recommended scripting language for Windows administration tasks.
- The WinNT provider used in this script has been superseded by the ActiveDirectory module in PowerShell.