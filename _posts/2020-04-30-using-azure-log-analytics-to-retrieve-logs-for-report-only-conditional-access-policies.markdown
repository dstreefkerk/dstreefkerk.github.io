---
layout: post
title: "Using Azure Log Analytics to retrieve logs for Report-Only Conditional Access Policies"
date: 2020-04-30
categories: [security, azure]
tags: [azure, azuread, azure-monitor, conditional-access, kql, log-analytics]
author: "Daniel Streefkerk"
excerpt: "How to use Azure Monitor and KQL queries to analyze sign-ins affected by report-only conditional access policies in Azure AD."
---

I've recently been working on reviewing conditional access policies in Azure AD. Thankfully this process has become much easier than the early days with the introduction of [Azure Monitor](https://azure.microsoft.com/en-us/services/monitor/) and [Report-Only mode](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-conditional-access-report-only) conditional access policies which allow you to properly pilot a configuration before going live.

I needed to grab an export of all sign-ins that were failing a particular report-only policy that was set up to [block legacy authentication](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/block-legacy-authentication). This led me down the path of Azure Monitor and writing my first [KQL](https://docs.microsoft.com/en-us/sharepoint/dev/general-development/keyword-query-language-kql-syntax-reference) query.

Note that this process depends on having set up streaming of [Azure AD logs into Azure Monitor](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/howto-integrate-activity-logs-with-log-analytics).

This KQL query grabs all sign-ins that have failed a report-only conditional access policy, and outputs the sign-in data alongside information about the policy in question.

Here's the KQL query code:

```kusto
SigninLogs
| mvexpand ConditionalAccessPolicies
| where ConditionalAccessPolicies.result == "reportOnlyFailure"
| project TimeGenerated, UserPrincipalName, AppDisplayName, ClientAppUsed, ConditionalAccessStatus, 
    PolicyName = ConditionalAccessPolicies.displayName,
    PolicyResult = ConditionalAccessPolicies.result,
    PolicyCondition = ConditionalAccessPolicies.conditionsSatisfied,
    PolicyControlResult = ConditionalAccessPolicies.enforcedGrantControls
```

To explain what the query does:

1. Retrieves all sign-in logs
2. Uses [mvexpand](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/mvexpandoperator) to expand the ConditionalAccessPolicies collection that's included along with each sign-in's data. The collection contains one object per conditional access policy in the Azure AD environment
3. Narrows down the list to only sign-ins where the result of a policy was a "reportOnlyFailure"
4. Uses the ['project' operator](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/projectoperator) to retrieve only the data we're interested in

From here, you can export the data to CSV and work your magic with it.