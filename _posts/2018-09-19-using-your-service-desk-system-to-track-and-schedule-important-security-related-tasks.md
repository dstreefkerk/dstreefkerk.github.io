---
layout: post
title: "Using your service desk system to track and schedule important & security-related tasks"
date: 2018-09-19
categories: technology system-administration
tags: security service-desk
author: "Daniel Streefkerk"
excerpt: "Leveraging service desk systems to automate, track and schedule important security-related tasks like certificate renewals, password rotations, and domain registrations to ensure continuity regardless of staff turnover."
---

Most IT departments would have some type of service desk system in place, but are they using it for more than just the basic support scenarios and change control?

Any modern service desk system should also be able to schedule tickets and change requests, and perhaps even perform more advanced workflow functions.

I'm using the excellent [Freshservice SaaS app](https://freshservice.com/solutions/task-scheduler), and I've recently been taking advantage of the scheduling and workflow features to automatically generate tickets to:

- Remind about domain registration renewals
- Remind about [SSL certificate expiry](https://nakedsecurity.sophos.com/2017/10/03/how-forgetting-to-renew-a-domain-name-cost-3m/)
- Ensure that the Domain's [krbtgt account](https://adsecurity.org/?p=483) password is rotated frequently
- [Schedule password changes for the Azure AD Connect seamless single sign-on domain account](https://www.dsinternals.com/en/impersonating-office-365-users-mimikatz/) AZUREADSSSOACC$
- Schedule manual password changes for service accounts that don't support the use of Managed Service Accounts (or gMSAs)
- Schedule guest Wi-Fi network key rotation

> Moving these types of tasks out of the minds and calendars of individual staff is important. It ensures that these sometimes critical actions continue regardless of staff turnover.

Another benefit is that within each scheduled ticket you can include clear written instructions on how to carry out the task. You also gain a long-term audit trail and notes for each time the task was carried out.

One final related note - you could also look into pointing your email security and other notifications to the service desk if you aren't already doing so. Again, you'll get a clear owner for each outstanding task, an audit trail of what was done, and you can assign priorities and SLAs. For example:

- Email administrator notifications (quarantine notifications, content notifications, etc)
- Print device consumable alerts
