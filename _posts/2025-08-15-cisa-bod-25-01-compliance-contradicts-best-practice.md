---
layout: post
title: "CISA BOD 25-01: When Compliance Contradicts Best Practice"
date: 2025-08-15
categories: [security, email]
tags: [email-security, spf, dmarc, compliance, bod-25-01]
author: "Daniel Streefkerk"
excerpt: "US Federal civilian executive branch agencies using Microsoft 365 must choose between complying with CISA's SCuBA requirements or following industry best practice for email authentication. They can't do both."
---

I noticed [CISA's new Binding Operational Directive 25-01](https://www.cisa.gov/news-events/directives/bod-25-01-implementing-secure-practices-cloud-services) pop up in my security news feeds recently. I'm currently building an email domain security scanner in my spare time, so naturally I added the BOD 25-01 requirements as a new compliance check.

That's when I discovered something interesting: US government domains were "failing" my new BOD 25-01 check, but their actual security posture was perfectly fine according to accepted industry best practice.

## The Requirement

BOD 25-01 mandates that US federal civilian executive branch (FCEB) agencies using Microsoft 365 configure their SPF records with a hard fail (`-all`) rather than the more common soft fail (`~all`). This requirement comes from CISA's Secure Cloud Business Applications (SCuBA) project, which provides secure configuration baselines for Microsoft 365. The directive must be implemented alongside DMARC in reject mode by June 2025.

## The Observation

Running my email security checker against the top ~100 federal .gov domains reveals an interesting split. This includes both FCEB agencies (who must comply with BOD 25-01) and legislative branch entities (who often follow CISA guidance voluntarily).

Of the 38 domains with valid configurations using Microsoft 365:

- **71% are compliant** with BOD 25-01/SCuBA - using SPF hard fail (`-all`) with DMARC reject
- **18% follow industry best practice** - using SPF soft fail (`~all`) with DMARC reject

The non-compliant 18% includes a mix of agency types:

**FCEB agencies (legally required to comply):**
- EPA (Environmental Protection Agency) - confirmed Microsoft 365 user
- Bureau of Labor Statistics (Department of Labor)

**Legislative branch entities (not required but often follow CISA guidance):**
- Congressional Budget Office (cbo.gov)
- Library of Congress (congress.gov)

Note that some agencies like the FDA (an FCEB agency under HHS), while having SPF soft fail, don't appear to be using Microsoft 365 and therefore wouldn't be subject to this specific SCuBA requirement.

## Why This Matters

The DMARC specification itself warns against using SPF hard fail with DMARC. While [RFC 7489](https://www.rfc-editor.org/rfc/rfc7489.html) was the original DMARC specification, it has been obsoleted by [RFC 9046](https://www.rfc-editor.org/rfc/rfc9046.html), which maintains the same fundamental guidance:

> "Some receiver architectures might implement SPF in advance of any DMARC operations. This means that a "-" prefix on a sender's SPF mechanism, such as "-all", could cause that rejection to go into effect early in handling, causing message rejection before any DMARC processing takes place."

Industry consensus from organisations like [RedSift](https://redsift.com/guides/email-protocol-configuration-guide/spf-failures-hard-fail-vs-soft-fail), [Mailhardener](https://www.mailhardener.com/blog/why-mailhardener-recommends-spf-softfail-over-fail), and [dmarcian](https://dmarcian.com/how-can-spfdkim-pass-and-yet-dmarc-fail/) is clear: use SPF soft fail (`~all`) with DMARC reject. This combination provides the same security outcome while maintaining operational flexibility.

## The Problem

SPF hard fail causes immediate SMTP-level rejection before DMARC evaluation. This means:

- Legitimate forwarded emails get blocked (forwarding typically breaks SPF, and while DKIM is more resilient to forwarding, it can also break if headers are modified)
- No DMARC reports are generated for SPF-rejected messages
- You lose visibility into what's being rejected
- DKIM can't compensate for SPF failures as DMARC intended

The whole point of DMARC is that it works if *either* SPF or DKIM passes. SPF hard fail undermines this design philosophy by making the decision at the SPF stage alone, before DKIM is even evaluated. This is confirmed by [Valimail's analysis](https://www.valimail.com/resources/guides/email-security-best-practices/spf-failure/) and extensive [community discussions](https://serverfault.com/questions/942586/does-an-spf-softfail-trigger-dmarc-reject).

## The Email Gateway Reality

In my experience with enterprise email security gateways like Proofpoint and Mimecast, the interaction between SPF and DMARC is more nuanced than it might first appear.

SPF operates on the SMTP envelope (the MAIL FROM address) during the initial SMTP transaction, while DMARC evaluates the message headers (the From: address) after the message is received. While modern gateways can perform unified DMARC checks that incorporate both SPF and DKIM results, the fundamental issue remains: SPF hard fail (`-all`) triggers an immediate rejection at the SMTP level, preventing the more sophisticated DMARC evaluation that considers both authentication methods.

The DMARC `p=reject` policy is designed to allow for a nuanced decision after both SPF and DKIM are evaluated. SPF hard fail preempts this evaluation, effectively reducing your authentication to a single point of failure. This architectural reality is exactly what RFC 9046 warns about - many email systems implement SPF checking before DMARC processing, making SPF hard fail particularly problematic.

## Contemporary Best Practice

Modern email authentication best practice is straightforward:

- **For active domains**: SPF soft fail (`~all`) + DMARC reject (`p=reject`)
- **For parked domains**: SPF record `v=spf1 -all` (no mechanisms, just hard fail)
- **Always**: Robust DKIM signing as your primary authentication method

*(Parked domains are domains that will never send legitimate email. For these, using `v=spf1 -all` prevents spoofing while acknowledging that no legitimate mail will ever originate from them)*

This approach maximises security while preserving email deliverability and troubleshooting capability, as detailed in [Microsoft's official guidance](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-dmarc-configure), [Google's troubleshooting documentation](https://support.google.com/a/answer/10032578), and the [M3AAWG Email Authentication Recommended Best Practices](https://www.m3aawg.org/sites/default/files/m3aawg-email-authentication-recommended-best-practices-09-2020.pdf). The M3AAWG (Messaging, Malware and Mobile Anti-Abuse Working Group) represents the industry consensus on email security, bringing together major ISPs, email providers, and security vendors.

## The Takeaway

Federal civilian executive branch agencies using Microsoft 365 face a frustrating choice: comply with BOD 25-01's SCuBA requirements or follow industry best practice and RFC 9046 guidance.

My quick analysis of .gov domains shows that both FCEB agencies (who must comply) and legislative branch entities (who often voluntarily follow CISA guidance) are choosing best practice over the directive. The deadline of June 20, 2025 to implement configurations has already expired.

When organisations like the EPA (FCEB) and the Congressional Budget Office (legislative) risk non-compliance rather than implement what they know is problematic, it validates what the secops community has been saying all along: sometimes compliance and security are two different things.