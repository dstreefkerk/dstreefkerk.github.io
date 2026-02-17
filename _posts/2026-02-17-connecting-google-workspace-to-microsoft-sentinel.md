---
layout: post
title: "Connecting Google Workspace to Microsoft Sentinel"
date: 2026-02-17
categories: [sentinel, google]
tags: [microsoft-sentinel, google-workspace, data-connector, oauth, codeless-connector-framework, log-ingestion]
author: "Daniel Streefkerk"
excerpt: "A practical walkthrough for connecting Google Workspace activity logs to Microsoft Sentinel, including the undocumented gotchas that'll save you from a frustrating afternoon."
---

The Google Workspace Activities data connector for Microsoft Sentinel uses the Codeless Connector Framework (still in Preview) to pull user activity logs from Google Workspace into your Sentinel workspace. The official documentation in the connector onboarding UI leaves a lot to be desired.

Here's the full process, with the gaps filled in.

> **⚠️ Known issue as of January 2026:** This connector *must* be configured from the **Azure portal** (`portal.azure.com`), not the unified Defender portal (`security.microsoft.com`). The OAuth redirect URI is hardcoded to `portal.azure.com`, so attempting the connection from the Defender portal will complete the Google authorisation successfully, then fail with a "Failed to retrieve auth token from provider" error when the token exchange bounces off the wrong URL.

## What You'll Need

- A Google Workspace account with **Super Admin** privileges
- Access to the [Google Cloud Console](https://console.cloud.google.com)
- A Microsoft Sentinel workspace with the **Google Workspace Activities** solution installed from Content Hub
- The **Microsoft Sentinel Contributor** role on the relevant workspace

## Part 1: Google Cloud Configuration

### Enable the Admin SDK API

Log in to the [Google Cloud Console](https://console.cloud.google.com), select or create a project for the integration, then navigate to **APIs & Services → Enabled APIs & Services**. Click **+ ENABLE APIS AND SERVICES**, search for **Admin SDK API**, and enable it.

### Configure the OAuth Consent Screen

Navigate to **APIs & Services → OAuth consent screen** (this may appear under **Google Auth Platform** depending on your console version).

Under **User type**, select **External**. This is the non-obvious bit — "Internal" seems like the right choice since you're connecting your own organisation's data, but Internal restricts OAuth to users within your Google Workspace tenant. The Sentinel connector's token exchange is actually performed by Microsoft's backend infrastructure, which is external to your tenant. Selecting Internal will cause the token exchange to fail silently.

Configure the consent screen with a descriptive app name (e.g. "Microsoft Sentinel Integration"), your admin email for support and developer contact, and add `azure.com` as an authorised domain.

### Configure OAuth Scopes

In the consent screen configuration, navigate to the **Scopes** section and add the following scope:

```
https://www.googleapis.com/auth/admin.reports.audit.readonly
```

This is the *only* scope the connector needs. When you search for "Admin SDK API" in the Google scopes list, you'll see multiple options — the one you want is specifically `admin.reports.audit.readonly`. Don't add `admin.reports.usage.readonly`; that covers the Usage Reports API, which this connector doesn't call.

Under the hood, the connector creates 21 separate RestApiPoller connections — one per Google Workspace application (admin, login, drive, calendar, chat, meet, and so on) — and they all use this same scope to call the [Activities: list](https://developers.google.com/admin-sdk/reports/reference/rest/v1/activities/list) API endpoint.

### Add Test Users

In the **Test users** section, add the email address of the Google Workspace admin account you'll use to authorise the connector. While the consent screen is in **Testing** status, only listed test users can complete the OAuth flow. Testing mode with your admin account listed is sufficient for Sentinel connector purposes.

### Create OAuth 2.0 Client Credentials

Navigate to **APIs & Services → Credentials**, click **+ CREATE CREDENTIALS → OAuth client ID**, and set the application type to **Web application**.

Under **Authorised redirect URIs**, add this URI exactly:

```
https://portal.azure.com/TokenAuthorize/ExtensionName/Microsoft_Azure_Security_Insights
```

**This must match exactly.** No Defender portal URL. No trailing slash.

Copy the **Client ID** and **Client Secret** — you'll need both for the Sentinel side.

## Part 2: Microsoft Sentinel Configuration

**Use the Azure portal for this step.** Navigate to [portal.azure.com](https://portal.azure.com), not `security.microsoft.com`.

In the Azure portal, go to **Microsoft Sentinel → [your workspace] → Configuration → Data connectors** and search for **Google Workspace Activities** (it may appear with the full "via Codeless Connector Framework (Preview)" suffix). If it's not listed, install the **Google Workspace** solution from Content Hub first.

Open the connector page, enter the Client ID and Client Secret, and click **Connect**. A Google sign-in window will open — authenticate with the admin account you added as a test user, grant the requested permissions, and the window should redirect back to `portal.azure.com` with a "Redirecting... Your account has been successfully authorized" message before closing automatically.

The connector status should change to **Connected**.

## Troubleshooting

### "Failed to retrieve auth token from provider"

This is by far the most common error. Work through these checks in order:

1. **Were you using the Defender portal?** Try again using `portal.azure.com`. You're not the first person to get tripped up by this.
2. **Is the OAuth consent screen set to Internal?** Change it to External.
3. **Does the redirect URI match exactly?** Check for trailing slashes or typos.
4. **Is the `admin.reports.audit.readonly` scope added?** It's the only one you need.
5. **Is the Admin SDK API enabled?** Verify under APIs & Services.
6. **Trailing whitespace in the Client ID or Secret?** Re-copy carefully.
7. **Admin account not listed as a test user?** Add it while the app is in Testing status.

### "Redirecting..." Page Hangs

The Google OAuth authorisation succeeded (you can verify by checking for a `code=` parameter in the URL bar), but the Azure portal failed to exchange the authorisation code for a token. This is almost always the Defender portal issue from item 1 above.

### Connected but No Data

Allow up to 30 minutes for initial ingestion. Query the `GWorkspace_ReportsAPI_audit_CL` table in Log Analytics to verify data flow. Check the `SentinelHealth` table for connector health issues. Also confirm that audit logging is enabled in the Google Admin console under **Reporting → Audit and investigation**.

## Quick Reference

| Item | Value |
|---|---|
| Google API | Admin SDK API |
| OAuth user type | External |
| OAuth scope | `https://www.googleapis.com/auth/admin.reports.audit.readonly` |
| Application type | Web application |
| Redirect URI | `https://portal.azure.com/TokenAuthorize/ExtensionName/Microsoft_Azure_Security_Insights` |
| Configuration portal | Azure portal (`portal.azure.com`) |
| Sentinel data table | `GWorkspace_ReportsAPI_audit_CL` |
