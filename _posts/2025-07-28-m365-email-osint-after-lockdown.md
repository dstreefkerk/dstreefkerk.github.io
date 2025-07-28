---
layout: post
title: "M365 Email OSINT After the Lockdown: What Still Works in 2025"
date: 2025-07-28
categories: [azure, security, powershell]
tags: [azure, entra-id, osint, powershell, tenant-enumeration, moera, security-assessment]
author: "Daniel Streefkerk"
excerpt: "Pondering Microsoft's recent Autodiscover service changes, and the information that remains publicly accessible for M365 email security reconnaissance."
---

If like me you've been doing M365 domain recon since [before it was cool](https://thecloudtechnologist.com/2019/04/07/analysis-of-dns-recon-of-the-fortune-500-part-1-of-3/), you may also have noticed that Microsoft has been busy closing doors on OSINT techniques that we've used for years. If you've been relying on AADInternals or similar tools for tenant reconnaissance, you'll know exactly what I'm talking about. The good old days of easy M365 domain discovery are largely behind us.

## The Autodiscover/Get-FederationInformation Lockdown

The secret sauce behind these tools was Microsoft's Autodiscover endpoint, which would cheerfully return complete lists of all domains associated with a tenant without requiring any authentication. 

Here's how it worked: The `Get-FederationInformation` cmdlet internally queried the Autodiscover endpoint to retrieve domain information. Tools like AADInternals leveraged this by calling the Autodiscover service directly, which would return the entire tenant's domain portfolio - custom domains, onmicrosoft.com domains, the lot.

This was incredibly useful for security assessments. Point the tool at a target organisation, get a comprehensive view of their footprint, and proceed accordingly.

### Why was this even possible in the first place?

To understand why this vulnerability existed, we need to look at Autodiscover's origins. 

Introduced with Exchange 2007, Autodiscover was designed to solve a genuine problem: the complexity of configuring email clients in on-premises Exchange environments. Back then, organisations might have multiple Exchange servers across different sites, complex load-balanced configurations, and varying server roles. Mobile devices running Windows Mobile 6.1 or early ActiveSync implementations needed a way to automatically find the right mailbox server without users having to know server names or URLs. Autodiscover made this possible with just an email address and password, dramatically reducing helpdesk tickets and configuration errors.

In the cloud era, however, Autodiscover's original purpose has largely evaporated. Microsoft 365 uses predictable endpoints, modern authentication flows, and Microsoft manages all the infrastructure complexity that Autodiscover was designed to navigate. Yet the service persisted, complete with its overly generous information disclosure, primarily for backward compatibility and hybrid scenarios. This created an awkward situation where a service designed for legitimate client configuration became a reconnaissance goldmine.

### What changed?

In June 2024, Microsoft announced they were changing how the Autodiscover service responds to these queries. The change, which took effect in mid-2025, meant that both the `Get-FederationInformation` cmdlet and direct Autodiscover queries would only return the specific domain you queried, rather than all domains in the tenant. This single change to the Autodiscover service broke multiple enumeration techniques at once.

Previously, you could run something like:
```powershell
Get-FederationInformation -DomainName contoso.com
```

And it would return all federated domain names for the target tenant:
```
{ contoso.com, contoso.net, mail.contoso.com, subsidiary.com, ... }
```

After the change, the same command only returns the specific domain you passed as a parameter:
```
{ contoso.com }
```

As Microsoft mentioned in their [Tech Community announcement](https://techcommunity.microsoft.com/blog/exchange/important-update-to-the-get-federationinformation-cmdlet-in-exchange-online/4410095), this was to "enhance the security and privacy of tenant information" by limiting domain exposure to potential attackers.

## The Security Cat-and-Mouse Game

From Microsoft's perspective, these changes make perfect sense. Why hand over reconnaissance data to anyone who asks? The original behaviour enabled trivial tenant enumeration and domain mapping, which could facilitate everything from social engineering to targeted phishing campaigns.

From a security research and security assessment perspective, it's more complicated. These techniques weren't just used by bad actors, they were fundamental tools for legitimate penetration testing, red team exercises, and security assessments. When you're trying to understand an organisation's attack surface, mapping their Azure/M365 presence is often step one.

Personally, I found this functionality to be extremely useful even as a finger-in-the-air type spot-check of a client's (or potential client's) security posture. If their email security posture was a shambles, then it's quite likely that the rest of their security posture mirrored that.

I used the aforementioned autodiscover functionality in my old [Invoke-EmailRecon.ps1 tool](https://github.com/dstreefkerk/PowerShell/blob/master/Infosec-Related/Invoke-EmailRecon.ps1), as well as an upcoming email security OSINT assessment tool that I've been working on for a while now.

The reality is that we're in an ongoing arms race between security research methodologies and vendor hardening efforts. Microsoft's changes are objectively good for security, but they do make legitimate assessment work more challenging.

## What's Still Discoverable to the Unauthenticated User?

There are still some unauthenticated endpoints we can leverage, though the approach is more involved and less reliable than the old methods.

### GetUserRealm: Organisation Names

The `GetUserRealm` endpoint still works for discovering organisation names. You can query it like this:

```powershell
$url = "https://login.microsoftonline.com/getuserrealm.srf?login=admin@contoso.com&xml=1"
$response = Invoke-RestMethod -Uri $url -Method Get
```

If the domain is associated with an Azure AD tenant, you'll get back XML containing the organisation name in the `FederationBrandName` field. This gives us a starting point for inference.

### OIDC Metadata: Tenant ID Extraction

The OpenID Connect metadata endpoints remain accessible and reveal valuable information:

```powershell
$url = "https://login.microsoftonline.com/contoso.com/v2.0/.well-known/openid-configuration"
$response = Invoke-RestMethod -Uri $url -Method Get
```

If successful, the response includes an issuer URL containing the tenant ID: `https://login.microsoftonline.com/{tenant-id}/v2.0`. This confirms the domain's association with Azure AD and gives you the tenant GUID.

### DNS Validation: MOERA Confirmation

The recent change also meant that it's not possible to easily discover the associated MOERA domain(s) for a given M365 tenant. MOERA (Microsoft Online Email Routing Address) domains are the default `.onmicrosoft.com` domains that every Microsoft 365 tenant receives when it's created. These domains serve several critical purposes:

- **Primary identity namespace**: Acts as the tenant's immutable identifier
- **Default email routing**: Provides a permanent email address that can't be removed
- **Service authentication**: Used for various Microsoft services and integrations
- **Fallback domain**: Ensures email delivery even if custom domains are misconfigured

Every M365 tenant has at least one MOERA domain following the pattern `tenantname.onmicrosoft.com`. While organisations typically add custom domains for their primary email addresses, the MOERA domain remains active and can receive email, making it valuable for reconnaissance.

Once we have an organisation name from GetUserRealm, we can generate candidate MOERA domains and validate them using DNS queries. MOERA domains typically have MX records pointing to Exchange Online protection:

```powershell
$candidate = "microsoft.onmicrosoft.com"
$mxRecords = Resolve-DnsName -Name $candidate -Type MX -ErrorAction SilentlyContinue
if ($mxRecords | Where-Object { $_.NameExchange -like "*mail.protection.outlook.com" }) {
    Write-Output "Found valid MOERA domain: $candidate"
}
```

### Additional Information Leakage

Beyond MOERA domains, Microsoft's publicly-accessible endpoints still reveal useful reconnaissance data:

1. **Federation Status**: GetUserRealm tells you if a domain is federated (ADFS/third-party) or managed (cloud-only)
2. **Tenant Existence**: OIDC endpoints confirm if a domain belongs to an Azure AD tenant
3. **Brand Names**: Organisation names from GetUserRealm can reveal subsidiary relationships
4. **Domain Validation**: DNS queries can confirm email-enabled domains through MX/SPF records

### The Email Discovery Angle

While we can't enumerate all domains anymore, we can still validate specific domains if we have candidates from other sources. This becomes especially powerful when organisations use third-party email gateways but still have Exchange Online configured.

#### EOP Smart Host Discovery

One of the most reliable techniques is checking for Exchange Online Protection (EOP) smart host endpoints. Even when an organisation's MX records point to a third-party secure email gateway (SEG), they often still have Exchange Online configured behind it:

```powershell
# Convert domain to EOP smart host format and validate
$domain = "amazon.com"
$smartHost = $domain.Replace('.', '-') + ".mail.protection.outlook.com"

# Check if the smart host resolves
$eop = Resolve-DnsName $smartHost -ErrorAction SilentlyContinue

if ($eop) {
    Write-Output "$domain uses Microsoft 365 (EOP endpoint found)"
} else {
    Write-Output "$domain does not appear to use Microsoft 365"
}
```

This technique is particularly valuable because:
- It works even when MX records point elsewhere (Proofpoint, Mimecast, etc.)
- The DNS response confirms the domain is configured in M365
- It's a direct validation rather than inference

#### Multiple Validation Points

My old [Invoke-EmailRecon.ps1](https://github.com/dstreefkerk/PowerShell/blob/master/Infosec-Related/Invoke-EmailRecon.ps1) script uses a confidence-based approach, checking multiple indicators, however I've not yet incorporated the latest changes discussed in this blog post:

```powershell
# Check various M365 indicators
$indicators = @{
    'MSOID' = (Resolve-DnsName "msoid.$domain" -EA SilentlyContinue).NameHost -like '*clientconfig.microsoftonline*'
    'TXT_Verification' = (Resolve-DnsName $domain -Type TXT -EA SilentlyContinue).Strings -like 'MS=ms*'
    'Autodiscover' = (Resolve-DnsName "autodiscover.$domain" -EA SilentlyContinue).NameHost -eq 'autodiscover.outlook.com'
    'EnterpriseReg' = (Resolve-DnsName "enterpriseregistration.$domain" -EA SilentlyContinue).NameHost -eq 'enterpriseregistration.windows.net'
    'DKIM_Selectors' = (Resolve-DnsName "selector1._domainkey.$domain" -EA SilentlyContinue) -ne $null
}

# Determine confidence level
$m365Confidence = switch ($true) {
    {$indicators.Autodiscover -or $indicators.DKIM_Selectors} { "Yes" }
    {$indicators.EnterpriseReg} { "Likely" }
    {$indicators.MSOID -or $indicators.TXT_Verification} { "Possibly" }
    default { "No" }
}
```

#### Federation Endpoint Validation

For domains using federated authentication, we can extract additional information:

```powershell
# Get federation details
$uri = "https://login.microsoftonline.com/getuserrealm.srf?login=user@$domain&xml=1"
$federationData = Invoke-RestMethod -Uri $uri -Method Get

# Extract tenant information
$brandName = $federationData.RealmInfo.FederationBrandName
$authUrl = $federationData.RealmInfo.AuthURL
$isFederated = $federationData.RealmInfo.NameSpaceType -eq 'Federated'
```

## The New Inference Approach

While pondering these recent changes, I had the idea that it might still be possible to infer the MOERA domain from other publicly-accessible tenant information. 

I've put together a new PowerShell script that combines some of these techniques into a systematic approach for domain metadata retrieval and MOERA domain discovery. The methodology goes something like this:

1. **Extract organisation name** from GetUserRealm
2. **Generate domain candidates** based on the organisation name
3. **Validate candidates** using DNS MX record lookups
4. **Apply confidence scoring** to distinguish between solid matches and educated guesses

The candidate generation logic handles organisation names like "Microsoft Corporation" → `microsoft.onmicrosoft.com` (high confidence, exact match) or, using some examples from the ASX 100:

| Domain | Organisation Name | MOERA Domain |
|--------|------------------|--------------|
| agl.com.au | AGL Energy | aglenergy.onmicrosoft.com |
| coles.com.au | Coles Supermarkets Pty Ltd | cspl.onmicrosoft.com (Inferred from initials of organisation name) |
| dexus.com | Dexus | dexus.onmicrosoft.com |
| lendlease.com | Lendlease | lendlease.onmicrosoft.com |
| resmed.com.au | Resmed Corp | resmedcorp.onmicrosoft.com |
| transurban.com | Transurban | Not found |
| wesfarmers.com.au | Wesfarmers Limited | Not found |

The approach normalises organisation names by removing special characters, generates acronyms from multi-word organisation names, and creates domain-based candidates from the input domain. Each candidate receives a confidence score that determines whether results are marked as "(Inferred)" or presented as high-confidence matches.

As you can see above, you have roughly a ~50% hit rate as opposed to what we've been used to in the past. The coles.com.au MOERA domain may or may not be correct, it's just a guess based on the organisation name. Also, the MOERA inference approach didn't work for transurban.com.au or wesfarmers.com.au. Their MOERA domains must be significantly different than their domain or their Azure/M365 organisational name.

You can find the complete implementation in my [Resolve-AzureTenant.ps1](https://github.com/dstreefkerk/PowerShell/blob/master/Microsoft%20Entra%20ID/Resolve-AzureTenant.ps1) script.

### Limitations and Reality Check

This approach has significant limitations compared to the old methods. We can't enumerate all domains in a tenant anymore. We can't reliably discover subsidiary domains or alternative naming schemes. Sometimes we get false positives, and sometimes we miss obvious targets.

The technique works best when organisations use predictable naming patterns for their MOERA domains, which thankfully many do. But it's fundamentally an inference game now rather than a data retrieval exercise.

It's worth noting that we need to be careful about false positives. Generic terms like "services" for a company named "ACME Services" might resolve to valid onmicrosoft.com domains, but they're unlikely to belong to the organisation you're actually researching.

Thankfully for paid client engagements, we can still ask them to hand over a full list of domains that are associated with their M365 tenant(s). Or, if you have the appropriate permissions:

```powershell
# Using Microsoft Graph PowerShell SDK (2025 method)
Connect-MgGraph -Scopes "Domain.Read.All"
Get-MgDomain | Select-Object Id, IsVerified, IsDefault | Format-Table

# Or as a one-liner with Exchange Online PowerShell
Connect-ExchangeOnline; Get-AcceptedDomain | Select Name, DomainName, DomainType
```

## Other OSINT Vectors Still Available

While domain enumeration has become more difficult, there are still other reconnaissance techniques that work:

### SPF and TXT Record Intelligence

SPF records remain a goldmine for discovering third-party services and infrastructure:

```powershell
# Extract SPF record and parse includes
$spf = (Resolve-DnsName domain.com -Type TXT | Where-Object {$_.Strings -like "v=spf1*"}).Strings
$includes = [regex]::Matches($spf, 'include:([^\s]+)') | ForEach-Object { $_.Groups[1].Value }

# Common service patterns in SPF includes:
# include:spf.protection.outlook.com - Microsoft 365
# include:_spf.salesforce.com - Salesforce
# include:mail.zendesk.com - Zendesk
# include:servers.mcsv.net - Mailchimp
# include:_spf.google.com - Google Workspace
```
Note that some organisations obfuscate that info behind SPF flattening services or SPF macros, but it's still discoverable if you're determined enough. I've got a proof-of-concept brewing at the back of my mind on this, so keep an eye on the blog here in the next few months.

TXT records often leak verification tokens and service integrations:
```powershell
# Look for service verification patterns
$txtRecords = Resolve-DnsName domain.com -Type TXT
$txtRecords.Strings | Where-Object {
    $_ -match "^(MS=|google-site-verification=|facebook-domain-verification=|adobe-sign-verification=)"
}
```

### Email Intelligence Services

Services like Hunter.io aggregate publicly available email data:
- Discover email naming patterns (first.last@, firstl@, etc.)
- Find additional domains used by the same organisation
- Verify email addresses without sending mail
- Historical email data that might reveal old infrastructure

### DNS History and Infrastructure Analysis

[SecurityTrails](https://securitytrails.com/) and similar services provide historical DNS records:
- Track MX record changes to identify migration patterns
- Find old SPF records revealing previous service providers
- Discover when organisations moved to/from M365
- Identify additional domains through shared infrastructure

Historical analysis often reveals:
```powershell
# Example: Organisation migrated from on-premises to M365
# Historical MX: mail.company.com (2020)
# Current MX: company-com.mail.protection.outlook.com (2025)
# Indicates M365 migration timeline
```

### Certificate Transparency Logs
Certificate transparency logs can reveal mail-related subdomains:
```powershell
# Look for mail-related certificates
# Common patterns: mail.*, smtp.*, webmail.*, owa.*
# These often reveal additional email infrastructure
```

### Domain Correlation Techniques
Passive DNS and reverse lookups can uncover related domains:
- Domains sharing the same mail infrastructure
- Domains with similar SPF records indicating common ownership
- IP address correlations for self-hosted mail servers

### Email Security Posture Analysis
Beyond finding domains, these techniques reveal security configurations:
- DMARC policy analysis across discovered domains
- SPF and DMARC record complexity/completeness indicating email security maturity
- Presence of well-known DKIM selectors showing authentication implementation
- MTA-STS policies indicating transport security adoption

## Conclusion

Microsoft's gradual lockdown of tenant enumeration techniques represents a natural evolution in the security landscape. From their perspective, there's no good reason to make reconnaissance trivially easy for potential attackers.

For those of us doing legitimate security work, it means adapting our methodologies and accepting that some information simply isn't as accessible as it used to be. The techniques I've outlined here work, but they require more effort and produce less comprehensive results than the old approaches.

It's messier and less reliable than the AADInternals heyday, but adaptation is part of the game. At least until Microsoft decides to close these doors too. I have something in the works that will address this, however I may not make it public except for as part of an email security service that I'm looking to launch in my spare time.

The script and techniques discussed here are available for legitimate security research and assessment purposes. Just remember that what works today might not work tomorrow. That's the nature of this particular cat-and-mouse game.

*References: [Microsoft Tech Community - Get-FederationInformation Update](https://techcommunity.microsoft.com/blog/exchange/important-update-to-the-get-federationinformation-cmdlet-in-exchange-online/4410095)*