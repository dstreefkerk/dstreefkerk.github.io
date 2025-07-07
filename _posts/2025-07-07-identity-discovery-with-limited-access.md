---
layout: post
title: "Useful Identity Discovery KQL Queries"
date: 2025-07-07
categories: [azure, sentinel, security]
tags: [kql, microsoft-sentinel, identity, entra-id, log-analytics, consulting]
author: "Daniel Streefkerk"
excerpt: "KQL queries to extract identity, group membership, and device information from Microsoft Sentinel when you don't have direct access to Entra ID or Intune portals."
---

When conducting security assessments or incident investigations, you may have access to Microsoft Sentinel but not have access to the broader set of identity and device context in Entra and Intune.

Here are some KQL queries that can be used to analyse identity, group, and device information from Log Analytics/Sentinel, as long as the prerequisite tables exist.

## 1. Geographic Distribution of Users

Discovers user locations based on group membership patterns.

**Required table:** `IdentityInfo` (Entra ID Identity Protection connector)  
*Check table availability using the [prerequisite check query](#prerequisite-check)*

```kql
// Discover user distribution across geographic locations based on group membership. Adjust the names as needed.
IdentityInfo
| where TimeGenerated > ago(30d)
| where isnotempty(GroupMembership)
| extend Groups = parse_json(GroupMembership)
| mv-expand Group = Groups
| extend GroupName = tostring(Group)
// Look for location-based groups
| where GroupName has_any (
    "Office", "Location", "Site", "Building", "Campus",
    // Common city names
    "London", "New York", "Sydney", "Singapore", "Tokyo", "Paris", "Dubai",
    // Country codes or names
    "USA", "UK", "AU", "SG", "JP", "FR", "AE"
)
| summarize 
    UserCount = dcount(AccountObjectId),
    SampleUsers = make_set(AccountUPN, 5),
    Departments = make_set(Department, 10)
    by GroupName
| where UserCount > 5  // Filter out small groups
| order by UserCount desc
```

## 2. Organisational Structure Discovery

Maps departments, job titles, and reporting relationships.

**Required table:** `IdentityInfo` (Entra ID Identity Protection connector)  
*Check table availability using the [prerequisite check query](#prerequisite-check)*

```kql
// Map organisational hierarchy through manager relationships and departments
IdentityInfo
| where TimeGenerated > ago(7d)
| where isnotempty(Department) or isnotempty(Manager)
| summarize arg_max(TimeGenerated, *) by AccountObjectId
| project 
    UserPrincipalName = AccountUPN,
    DisplayName = AccountDisplayName,
    Department,
    JobTitle,
    Manager,
    AssignedRoles,
    GroupMembership
| extend ParsedGroups = parse_json(GroupMembership)
| extend GroupCount = array_length(ParsedGroups)
// Create department summary
| summarize 
    UserCount = count(),
    Titles = make_set(JobTitle, 20),
    ManagerCount = dcount(Manager),
    AvgGroupMembership = avg(GroupCount)
    by Department
| where isnotempty(Department)
| order by UserCount desc
```

## 3. Privileged Account Discovery

Identifies administrative and high-privilege accounts.

**Required table:** `IdentityInfo` (Entra ID Identity Protection connector)  
*Check table availability using the [prerequisite check query](#prerequisite-check)*

```kql
// Find accounts with administrative roles or privileged group memberships
IdentityInfo
| where TimeGenerated > ago(30d)
| where isnotempty(AssignedRoles) or GroupMembership has_any (
    "admin", "Admin", "Administrator",
    "Global", "Privileged", "Security",
    "Exchange", "SharePoint", "Teams",
    "Compliance", "Billing", "Password"
)
| summarize arg_max(TimeGenerated, *) by AccountObjectId
| extend 
    Roles = parse_json(AssignedRoles),
    Groups = parse_json(GroupMembership)
| mv-expand Role = Roles
| extend RoleName = tostring(Role)
| where RoleName !has "Guest" and RoleName !has "Member"
| summarize 
    Users = make_set(AccountUPN),
    UserCount = dcount(AccountObjectId),
    SampleRoles = make_set(RoleName, 10)
    by RoleType = case(
        RoleName has "Global", "Global Administrator",
        RoleName has_any ("Security", "Compliance"), "Security & Compliance",
        RoleName has_any ("Exchange", "Teams", "SharePoint"), "Workload Administrator",
        RoleName has_any ("User", "Password", "Authentication"), "Identity Administrator",
        RoleName has_any ("Application", "Cloud"), "Application Administrator",
        "Other Privileged"
    )
| order by UserCount desc
```

## 4. Active Directory Group Analysis

Analyses AD group patterns to understand organisational structure.

**Required table:** `IdentityInfo` (Entra ID Identity Protection connector)  
*Check table availability using the [prerequisite check query](#prerequisite-check)*

```kql
// Analyze AD group patterns to understand organizational structure
IdentityInfo
| where TimeGenerated > ago(30d)
| where isnotempty(GroupMembership)
| summarize arg_max(TimeGenerated, *) by AccountObjectId
| extend Groups = parse_json(GroupMembership)
| mv-expand Group = Groups
| extend GroupName = tostring(Group)
// Categorize groups by naming patterns
| extend GroupCategory = case(
    GroupName has_any ("VPN", "Remote", "WiFi", "Network"), "Network Access",
    GroupName has_any ("Drive", "Share", "Folder", "FS_"), "File Share Access",
    GroupName has_any ("App_", "Application", "Software"), "Application Access",
    GroupName has_any ("DL_", "Distribution", "Mail"), "Distribution List",
    GroupName has_any ("Sec_", "Security", "Role"), "Security Group",
    GroupName has_any ("Dept_", "Department", "Team"), "Organizational Unit",
    GroupName has_any ("Project", "Proj_"), "Project Group",
    "Other"
)
| summarize 
    MemberCount = dcount(AccountObjectId),
    SampleMembers = make_set(AccountUPN, 3),
    Departments = make_set(Department, 5)
    by GroupName, GroupCategory
| where MemberCount > 2
| order by GroupCategory asc, MemberCount desc
```

## 5. Service Account Discovery

Identifies potential service accounts based on naming patterns and behaviour.

**Required tables:** 
- `SigninLogs` (Azure AD Sign-in logs) - **Required**
- `AADNonInteractiveUserSignInLogs` (Azure AD Non-Interactive Sign-in logs) - **Optional, enhances results**

*Check table availability using the [prerequisite check query](#prerequisite-check)*

**Note:** This query uses `union` which gracefully handles missing tables. If `AADNonInteractiveUserSignInLogs` isn't available, the query will still run using only `SigninLogs` data.

```kql
// Identify potential service accounts based on naming patterns and behaviour
union withsource=TableName SigninLogs, AADNonInteractiveUserSignInLogs
| where TimeGenerated > ago(30d)
| where UserPrincipalName has_any (
    "svc", "service", "app", "api", "system", "batch", "job", "task",
    "automation", "integration", "sync", "backup", "monitor"
)
| extend Hour = hourofday(TimeGenerated)
| extend 
    LocationCity = column_ifexists("LocationDetails", dynamic({})).city,
    LocationCountry = column_ifexists("LocationDetails", dynamic({})).countryOrRegion
| summarize 
    SignInCount = count(),
    UniqueIPs = dcount(IPAddress),
    UniqueApps = dcount(AppDisplayName),
    HourlyDistribution = make_list(Hour),
    Apps = make_set(AppDisplayName, 10),
    Locations = make_set(strcat(tostring(LocationCity), ", ", tostring(LocationCountry)), 5)
    by UserPrincipalName, UserType
| extend 
    IsLikelyServiceAccount = iff(
        UniqueIPs <= 3 and SignInCount > 100, 
        true, 
        false
    )
| where IsLikelyServiceAccount == true or UserPrincipalName has_any ("svc_", "sa_")
| order by SignInCount desc
```

## 6. Device Inventory

Builds device inventory from available logs.

**Required table:** `DeviceInfo` (Microsoft 365 Defender connector)  
*Check table availability using the [prerequisite check query](#prerequisite-check)*

```kql
// Build device inventory from multiple sources
DeviceInfo
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by DeviceId
| extend 
    // Check which fields exist and provide defaults
    DeviceNameField = column_ifexists("DeviceName", ""),
    DeviceTypeField = column_ifexists("DeviceType", "Unknown"),
    OSPlatformField = column_ifexists("OSPlatform", ""),
    OSVersionField = column_ifexists("OSVersion", column_ifexists("OSVersionInfo", "")),
    ComplianceField = column_ifexists("IsCompliant", false),
    ManagedField = column_ifexists("IsManaged", false),
    JoinTypeField = column_ifexists("JoinType", ""),
    LoggedOnUsersField = tostring(column_ifexists("LoggedOnUsers", "")),
    LastSeen = TimeGenerated
| extend 
    ManagementStatus = case(
        ManagedField == true and ComplianceField == true, "Managed & Compliant",
        ManagedField == true and ComplianceField == false, "Managed & Non-Compliant",
        ManagedField == false, "Unmanaged",
        "Unknown"
    ),
    // Only extract users if the field is not empty
    Users = iff(isnotempty(LoggedOnUsersField), 
                extract_all(@"([^\\]+)\\([^,]+)", LoggedOnUsersField), 
                dynamic([]))
| extend PrimaryUser = iff(array_length(Users) > 0, tostring(Users[0][1]), "Unknown")
| summarize 
    DeviceCount = count(),
    SampleDevices = make_set(DeviceNameField, 10),
    OSVersions = make_set(OSVersionField, 5),
    PrimaryUsers = make_set(PrimaryUser, 10)
    by DeviceTypeField, OSPlatformField, ManagementStatus
| project-rename 
    DeviceType = DeviceTypeField,
    OSPlatform = OSPlatformField
| order by DeviceCount desc
```

## 7. License and Application Usage

Tracks application usage patterns for license optimisation.

**Required table:** `SigninLogs` (Azure AD Sign-in logs)  
*Check table availability using the [prerequisite check query](#prerequisite-check)*

```kql
// Discover application usage patterns and potential license optimisation
SigninLogs
| where TimeGenerated > ago(30d)
| where ResultType == 0
| where AppDisplayName !in ("Windows Sign In", "Microsoft Authentication Broker")
| summarize 
    UserCount = dcount(UserPrincipalName),
    SignInCount = count(),
    UniqueUsers = make_set(UserPrincipalName, 100),
    Departments = make_set(extract(@"([^@]+)@", 1, UserPrincipalName), 20)
    by AppDisplayName, AppId
| extend AvgSignInsPerUser = round(toreal(SignInCount) / toreal(UserCount), 2)
| order by UserCount desc
| project 
    AppDisplayName,
    UserCount,
    SignInCount,
    AvgSignInsPerUser,
    SampleUsers = array_slice(UniqueUsers, 0, 5),
    DepartmentCount = array_length(Departments)
```

## Usage Notes

- Adjust time ranges based on data retention policies
- Add domain filters with `| where AccountUPN endswith "@yourdomain.com"`
- Use `column_ifexists()` to handle schema variations across environments
- Export results using `| project` to select specific columns
- Consider performance impact on large datasets - add filters early in queries

## Common Filters

Filter by department:
```kql
| where Department has "IT" or Department has "Engineering"
```

Exclude system accounts:
```kql
| where AccountUPN !startswith "SystemAccount" 
| where AccountUPN !has "$"
```

Focus on recent changes:
```kql
| where TimeGenerated > ago(7d)
| where AccountCreationTime > ago(30d)  // New accounts only
```

## Prerequisite Check

Run this query to check which tables are available in your workspace:

```kql
let RequiredTables = dynamic([
    "IdentityInfo", 
    "SigninLogs", 
    "AADNonInteractiveUserSignInLogs", 
    "DeviceInfo"
]);
let AvailableTables = 
    search * 
    | where TimeGenerated > ago(1h)
    | distinct $table
    | summarize AvailableTables = make_set($table);
AvailableTables
| extend Table = RequiredTables
| mv-expand Table to typeof(string)
| extend Status = iff(AvailableTables has Table, "Available", "Not Available")
| extend RowCount = iff(Status == "Available", 
    toscalar(
        union withsource=T *
        | where T == Table and TimeGenerated > ago(7d)
        | count
    ), 0)
| project Table, Status, RowCount
| order by Status desc, Table asc
```