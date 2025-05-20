---
layout: post
title: "Determine the Logged On User's AD Group Membership in PowerShell"
date: 2014-05-28
categories: powershell active-directory
tags: powershell active-directory scripting systems-administration security
author: "Daniel Streefkerk"
excerpt: "A simple PowerShell technique to retrieve and display the Active Directory groups a currently logged-on user belongs to, with practical examples and security considerations."
---

> **Note:** This article was originally written in May 2014 and is now over 11 years old. Due to changes in PowerShell and Active Directory modules over time, the described solution may require modification for current environments. Modern authentication methods, including Azure AD and hybrid identities, may require different approaches. Please consider this guide as a conceptual reference rather than a current implementation guide.

When you're writing scripts that need to check user permissions or group memberships, it's helpful to be able to determine what groups the currently logged-on user belongs to. Here's a quick and easy way to determine this using PowerShell.

## Using .NET Classes

The simplest approach uses the System.Security.Principal namespace to get the current user's identity and group memberships:

```powershell
$identity = [System.Security.Principal.WindowsIdentity]::GetCurrent()
$identity.Groups | ForEach-Object {
    $group = $_.Translate([System.Security.Principal.NTAccount])
    $group.Value
}
```

This will output a list of all the groups that the currently logged-on user is a member of, including domain groups and built-in groups.

## Using the ActiveDirectory Module

If you have the ActiveDirectory module installed (part of the Remote Server Administration Tools), you can get more detailed information about group memberships:

```powershell
# Import the module if not already loaded
Import-Module ActiveDirectory

# Get current user's SamAccountName
$currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent()
$samAccountName = $currentUser.Name.Split('\')[1]

# Get the user's group memberships
$groups = Get-ADPrincipalGroupMembership -Identity $samAccountName | Select-Object Name, GroupCategory, GroupScope

# Display the results
$groups | Format-Table -AutoSize
```

This approach provides more information about each group, such as group type (Security or Distribution) and scope (Domain Local, Global, or Universal).

## Recursive Group Membership

Sometimes you need to find all groups a user belongs to, including nested group memberships. Here's how to do that:

```powershell
function Get-ADNestedGroupMembership {
    param(
        [Parameter(Mandatory = $true)]
        [string]$UserName
    )
    
    $groups = @()
    $user = Get-ADUser -Identity $UserName -Properties MemberOf
    
    foreach ($groupDN in $user.MemberOf) {
        $group = Get-ADGroup -Identity $groupDN -Properties Members, MemberOf
        $groups += $group
        
        # Recursively get parent groups
        if ($group.MemberOf) {
            $groups += Get-NestedGroupMemberships -GroupDNs $group.MemberOf
        }
    }
    
    return $groups | Select-Object Name, GroupCategory, GroupScope | Sort-Object Name -Unique
}

function Get-NestedGroupMemberships {
    param(
        [Parameter(Mandatory = $true)]
        [array]$GroupDNs
    )
    
    $groups = @()
    
    foreach ($groupDN in $GroupDNs) {
        $group = Get-ADGroup -Identity $groupDN -Properties MemberOf
        $groups += $group
        
        # Recursively get parent groups
        if ($group.MemberOf) {
            $groups += Get-NestedGroupMemberships -GroupDNs $group.MemberOf
        }
    }
    
    return $groups
}

# Get current user's SamAccountName
$currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent()
$samAccountName = $currentUser.Name.Split('\')[1]

# Get all nested group memberships
Get-ADNestedGroupMembership -UserName $samAccountName
```

This will retrieve all groups the user belongs to, including groups that the user is a member of indirectly through nested group memberships.

## Checking for Specific Group Membership

If you just want to check if the current user is a member of a specific group, you can use this simple function:

```powershell
function Test-GroupMembership {
    param(
        [Parameter(Mandatory = $true)]
        [string]$GroupName
    )
    
    $identity = [System.Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object System.Security.Principal.WindowsPrincipal($identity)
    
    return $principal.IsInRole($GroupName)
}

# Example usage
if (Test-GroupMembership -GroupName "Domain Admins") {
    Write-Host "User is a Domain Admin"
} else {
    Write-Host "User is not a Domain Admin"
}
```

This is very efficient as it doesn't need to retrieve all group memberships when you just want to check for a specific group.

## Security Considerations

When implementing this functionality, keep the following security considerations in mind:

1. **Privilege Escalation**: Code that enumerates group memberships could potentially be used in privilege escalation attacks if not properly secured. Ensure any scripts containing these functions have appropriate access controls.

2. **Performance Impact**: Recursive group membership lookups can be resource-intensive in large environments with complex nesting. Consider caching results when appropriate.

3. **Domain Controller Load**: These functions query Active Directory, which could generate significant load on domain controllers if run frequently or against many users simultaneously.

4. **Least Privilege**: When checking for group membership to authorise actions, always follow the principle of least privilege and only check for the specific groups needed.

5. **Authentication Context**: Remember that the group membership is for the user context running the PowerShell session, which may differ from the logged-on user if using alternate credentials or running in an elevated session.

## Modern Alternatives and Updates

Since this post was written in 2014, several developments have occurred:

1. **PowerShell 7+**: Modern PowerShell versions offer improved performance and security features.

2. **Azure AD/Microsoft Entra ID**: For cloud or hybrid environments, consider using Microsoft Graph API or Azure AD PowerShell modules instead of the on-premises Active Directory cmdlets.

3. **Just-In-Time Administration**: Modern security practices prefer just-in-time privileged access management over standing group memberships.

4. **Privileged Access Management**: For sensitive environments, consider using a PAM solution instead of directly checking AD group memberships.

5. **Security Issues**: The WindowsIdentity.GetCurrent() method only retrieves token groups for the current session, which may not include all groups if the session token is limited or filtered.

6. **Conditional Access Policies**: Modern environments may implement conditional access, where group membership alone doesn't determine access rights.

For current environments, especially those with hybrid identity models, you may need to combine these techniques with Azure AD-based approaches for comprehensive identity management.