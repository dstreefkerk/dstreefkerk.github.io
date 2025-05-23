---
layout: post
title: "FortiAnalyzer CEF and the Case of the Missing Logs"
date: 2025-05-23
categories: [security, azure, sentinel]
tags: [fortianalyzer, sentinel, azure-monitor-agent, cef, rsyslog, syslog, log-ingestion, fortinet]
author: "Daniel Streefkerk"
excerpt: "How to fix FortiAnalyzer's non-compliant CEF messages that lack syslog PRI headers when ingesting to Microsoft Sentinel via Azure Monitor Agent, while reducing ingestion costs through intelligent filtering."
---

I recently helped a client set up FortiAnalyzer CEF log ingestion to Sentinel via AMA. We ran into a bunch of issues, and I ended up coming up with what I thought was a decent workaround that:

* Didn't mess with the standard AMA/rsyslog configuration, thereby putting the AMA into a potentially unsupported state and impacting other CEF ingestion data flows.
* Didn't require the client to fall back to older and more fragile syslog-based log ingestion.

## The Problem: When Standards Collide

FortiAnalyzer has this interesting quirk where it sends CEF messages that are *technically* valid CEF, but completely ignore the syslog RFC standards that Azure Monitor Agent (AMA) expects. It's a classic case of two vendors implementing standards differently, and there's not much hope of Fortinet changing anything as issues like this have [been a thing](https://community.graylog.org/t/fortinet-cef-formatting-issues/12017) for [some time now](https://www.syslog-ng.com/community/b/blog/posts/parsing-fortigate-logs-and-other-syslog-ng-3-31-news).

The core issue that I encountered was **Missing PRI Headers**. FortiAnalyzer sends raw CEF messages without the RFC3164/RFC5424 compliant syslog PRI headers.

A PRI header is the priority field in syslog messages (like <164>) that combines the facility and severity levels into a single number, telling receiving systems what type of log it is and how urgent it should be treated-and without it, Azure Monitor Agent simply refuses to process the message.

## The Battlefield: Understanding the Data Flow

Before diving into the solution, let's map out the data flow:

```
FortiAnalyzer → rsyslog (UDP 514) → Processing → AMA (TCP 28330) → Sentinel
```

The challenge is intercepting those raw CEF messages and transforming them into something AMA will accept, while simultaneously filtering out the noise before it hits your ingestion costs.

## The Solution: rsyslog as the Middle Layer

After examining how the default AMA rsyslog configuration handles incoming syslog traffic, I took inspiration from its approach and adapted it for FortiAnalyzer's specific quirks. The default AMA config already has the framework for receiving, processing, and forwarding syslog messages-I just needed to extend it for headerless CEF.

We ended up with the following data flow instead:

```
FortiAnalyzer → rsyslog (UDP 1514) → Processing → AMA (TCP 28330) → Sentinel
```

### 1. Receive and Validate

First, we need rsyslog to listen for FortiAnalyzer's UDP traffic on a non-standard port (1514) to avoid conflicts:

```
input(type="imudp" port="1514" ruleset="forti-force-pri-cef")
```

### 2. Force PRI Header Addition

This is where we diverge from the standard AMA config. I created a template that *always* adds the PRI header, no conditional logic:

```
template(name="FortiCEF_ForcePRI" type="string" 
         string="<164>%TIMESTAMP% %HOSTNAME% FortiAnalyzer: %rawmsg-after-pri%\n")
```

The `<164>` represents local4.warning, matching the DCR configuration that we'd implemented. You can adjust this to your own DCR's config with the help of references like [this one](https://techdocs.broadcom.com/us/en/symantec-security-software/identity-security/privileged-access-manager/4-2/reference/messages-and-log-formats/syslog-message-formats/syslog-priority-facility-severity-grid.html).

### 3. Implement Severity Filtering

Because FortiAnalyzer isn't sending a valid PRI, the default AMA rsyslog config has no way to figure out what which events to filter. It'll just forward everything up to Sentinel. 

Here's where we save on ingestion costs. Instead of forwarding everything, we only pass through security-relevant events:

```
if ($rawmsg contains "deviceSeverity=critical" or 
    $rawmsg contains "deviceSeverity=high" or 
    $rawmsg contains "deviceSeverity=medium") then {
    # Forward to AMA
}
```

### 4. Forward to AMA

Finally, we forward the processed messages to AMA on localhost, using the same port that the default configuration expects:

```
action(type="omfwd"
       template="FortiCEF_ForcePRI"
       target="127.0.0.1"
       port="28330"
       protocol="tcp"
       queue.type="LinkedList"
       queue.size="25000" 
       queue.workerThreads="100")
```

The queue configuration here is crucial. FortiAnalyzer can generate significant log volumes during security events, and you need enough buffer to handle bursts without dropping messages.

## Results

After implementing this configuration, the logs are actually making their way through to Sentinel, and we get the following benefits as a side effect:

- **Reduction** in log volumes hitting Sentinel
- **Cost savings** on ingestion
- **Improved signal-to-noise ratio** for security analysts

## Gotchas and Lessons Learnt

1. **Test with Debug Logging**: Redirect logs to a local file to be able to debug properly if you run into issues. In the below script, that's line 58. You'll want visibility into what's actually hitting rsyslog before assuming it's working.

2. **Monitor Ingestion Lag**: Remember that Sentinel has an ingestion delay of several minutes.

3. **DCR Configuration**: Ensure your Data Collection Rule is expecting LOG_LOCAL4:LOG_WARNING. There's potential to waste hours troubleshooting if it's configured for a different facility/severity combination.

## Wrapping Up
This solution might feel a bit hacky—and honestly, it is. We're essentially working around a vendor compatibility issue that really shouldn't exist. But in the real world of enterprise security, these kinds of workarounds are often necessary to get disparate systems talking to each other.

The upside is that we've turned a compatibility headache into an opportunity to implement intelligent filtering, reducing both ingestion costs and analyst noise. Sometimes the best solutions come from making the best of a less-than-ideal situation.

If you're facing similar FortiAnalyzer integration challenges, hopefully this saves you some of the trial and error I went through.

<script src="https://gist.github.com/dstreefkerk/07e2c942136f27dff13d04b3f5f33f77.js"></script>