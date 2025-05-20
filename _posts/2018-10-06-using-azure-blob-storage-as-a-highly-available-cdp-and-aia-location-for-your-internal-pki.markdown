---
layout: post
title: "Using Azure Blob Storage as a highly-available CDP and AIA location for your internal PKI"
date: 2018-10-06
categories: [azure, security]
tags: [pki, windows, azure-storage, certificates, blob-storage]
author: "Daniel Streefkerk"
excerpt: "A practical guide to using Azure Blob Storage as a reliable, highly-available location for hosting your internal PKI's CDP and AIA components."
---

> **Note:** This article was originally written in October 2018 and is now over 7 years old. Due to changes in Azure services and AzCopy functionality over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.


I inherited a Windows PKI setup that had the Root CA installed on a Windows Server 2008 R2 Domain Controller, with the root certificate [signed with a SHA1 hash](https://blog.qualys.com/ssllabs/2014/09/09/sha1-deprecation-what-you-need-to-know). That DC was in the process of being decommissioned, and I also wanted to move to a better PKI design.

I'd previously set up 2-tier Windows PKI infrastructures with offline Root CAs, so I knew that this was the route I was going to take again (note that this is for an SMB environment).

There are plenty of good guides on configuring a 2-tier Windows PKI. In my opinion the best of the crop at the time of writing is probably Timothy Gruber's [7-part guide to deploying a PKI on Windows Server 2016](https://timothygruber.com/pki/deploy-a-pki-on-windows-server-2016-part-1/).

I would, however, highly recommend reading up on the topic before blindly following a guide. PKI is a complex topic, and you want to make the correct decisions up-front to avoid issues later on. Some additional recommended reading:

- [MS Directory Services Team Blog: Moving Your Organization from a Single Microsoft CA to a Microsoft Recommended PKI](https://blogs.technet.microsoft.com/askds/2010/08/23/moving-your-organization-from-a-single-microsoft-ca-to-a-microsoft-recommended-pki/)
- [Best Practices for Implementing a Microsoft Windows Server 2003 Public Key Infrastructure (Still contains some useful info)](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc772670(v=ws.10))
- [Aaron Parker: Deploying an Enterprise Root Certificate Authority](https://stealthpuppy.com/deploy-enterprise-root-certificate-authority/)
- [Andrzej Kaźmierczak: The DOs and DON'Ts of PKI – Microsoft ADCS](https://777notes.wordpress.com/2016/07/11/certificates-the-dos-and-donts-of-pki/)
- [Microsoft Docs: Server 2012/2012 R2 Certification Authority Guidance](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/hh831574(v=ws.11))

There are many recommendations around where to publish/advertise the [AIA](http://www.pkiglobe.org/auth_info_access.html) and [CDP](http://www.pkiglobe.org/crl_dist_points.html). Some of these include:

- In the default location - LDAP and locally via HTTP on the CA server
- To an internally-hosted web server, and then reverse-proxy connections from the Internet
- To an externally-hosted web server

I'd already used Azure Blob Storage to store some other small files, so I thought I'd have a go at seeing if it's able to be used for AIA and CDP storage. As it turns out, it's quite easy to do, and you don't even need to [mess around with double-escaping](https://blogs.iis.net/thomad/iis7-rejecting-urls-containing) like you would need to if you hosted on IIS [or an Azure Web App](https://pertorben.wordpress.com/2018/01/05/host-crl-and-aia-for-your-internal-pki-in-microsoft-azure-web-app/):

<div class="embed-twitter">
<blockquote class="twitter-tweet" data-width="550" data-dnt="true">
<p lang="en" dir="ltr">Playing with AzCopy and Blob Storage as a PoC for PKI CRL storage. It's pretty handy! <a href="https://t.co/yrBzDyBEJJ">https://t.co/yrBzDyBEJJ</a></p>&mdash; Daniel Streefkerk (@dstreefkerk) <a href="https://twitter.com/dstreefkerk/status/1042296844305350658?ref_src=twsrc%5Etfw">September 19, 2018</a>
</blockquote>
</div>

> TLDR; The CA saves the CRL files to the default location of C:\Windows\System32\CertSrv\CertEnroll, and AzCopy then copies them up to an Azure Blob Storage account that's configured with a custom domain of pki.yourdomain.com

Here are the requirements to get this all set up:

1. CDP and AIA on Enterprise/issuing CA configured to save to the default C: location, and also advertise availability at http://pki.yourdomain.com
2. [AzCopy](http://aka.ms/azcopy) installed on the Enterprise CA
3. Allow outbound HTTPS/443 from the Enterprise CA to Azure Blob Storage
4. An [Azure Storage Account](https://docs.microsoft.com/en-us/azure/storage/common/storage-quickstart-create-account?tabs=portal) with blob storage configured for HTTP access. I'd recommend at least [Zone Redundant Storage](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview#replication) for availability.
5. A [custom domain name](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-custom-domain-name) for the above storage account
6. A folder in the blob storage named 'pki' (not necessary, but you'll need to adjust the script if you don't use this folder)
7. A [SAS key](https://docs.microsoft.com/en-us/azure/storage/common/storage-dotnet-shared-access-signature-part-1) with read/write/change access to blob storage only (don't assign more access than necessary)
8. A [scheduled task](https://blogs.technet.microsoft.com/heyscriptingguy/2012/08/11/weekend-scripter-use-the-windows-task-scheduler-to-run-a-windows-powershell-script/) running hourly as NETWORK SERVICE to call the below PowerShell script
9. Ensure that NETWORK SERVICE has modify permissions to the log location (default is %ProgramData%\ScriptLogs\Invoke-UpdateAzureBlobPKIStorage.log)

You'll need to manually copy your offline root CA certificate and CRL to the blob storage location. This script is designed to copy the much more frequent CRLs and Delta CRLs from your Enterprise CA to blob storage.

As it turns out, AzCopy is perfect for this because it supports the [/XO parameter](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy#azcopy-parameters) to only copy new files. That allows us to schedule the script to run hourly without incurring additional data transfer costs for files that already exist in the storage account.

I wrote a PowerShell script that does the following:

1. Checks that AzCopy is installed
2. Determines if the C:\Windows\System32\CertSrv\CertEnroll folder exists
3. Copies only changed files with extension .CRL to to the blob storage account
4. Logs successful and failed transfers to %ProgramData%\ScriptLogs\Invoke-UpdateAzureBlobPKIStorage.log

You can find the script on my Github repo here: [https://github.com/dstreefkerk/PowerShell/blob/master/PKI/Invoke-UpdateAzureBlobPKIStorage.ps1](https://github.com/dstreefkerk/PowerShell/blob/master/PKI/Invoke-UpdateAzureBlobPKIStorage.ps1)

*[Image removed: Screenshot of pkiview.msc on a domain-joined machine showing the status of CDP and AIA]*

*[Image removed: Screenshot showing how to generate a SAS with least-privilege for AzCopy to use, demonstrating that "Allowed Protocols" should be set to "HTTPS and HTTP", not HTTPS only]*

*[Image removed: Screenshot of the script's archive log, showing the successful transfer of the CRL and Delta CRL]*



As always, use this at your own risk and your mileage may vary. Please drop me a comment below if you have any questions, feedback, or run into issues with the script.

---

## Updates and Community Notes

**Update from Jeff M (January 2020):**  
If you're migrating an existing PKI to Blob Storage and your CRL had the URL "CertEnroll" (which won't work due to lowercase requirements on Blob Container names), you can use Azure CDN to re-write /CertEnroll to /certenroll for pennies a month. This allows you to mothball your old Windows webserver when combined with the script above.

**Note on AzCopy Version:**  
If you're using AzCopy v10+, be aware that it has been fundamentally redesigned, and the /XO switch no longer works as in v8. You may need to use AzCopy v8 for this script to function properly.
