---
layout: post
title: "FortiAnalyzer CEF and the Case of the Missing Logs"
date: 2025-05-23
categories: [security, azure, sentinel]
tags: [fortianalyzer, sentinel, azure-monitor-agent, cef, rsyslog, syslog-ng, log-ingestion, fortinet]
author: "Daniel Streefkerk"
excerpt: "How to fix FortiAnalyzer's non-compliant CEF messages that lack syslog PRI headers when ingesting to Microsoft Sentinel via Azure Monitor Agent, supporting both rsyslog and syslog-ng environments."
---

I recently helped a client set up FortiAnalyzer CEF log ingestion to Sentinel via AMA. We ran into a bunch of issues, and I ended up coming up with what I thought was a decent workaround that:

* Didn't mess with the standard AMA/rsyslog configuration, thereby putting the AMA into a potentially unsupported state and impacting other CEF ingestion data flows
* Didn't require the client to fall back to older and more fragile syslog-based log ingestion
* Works with both rsyslog and syslog-ng environments

## The Problem: When Standards Collide

FortiAnalyzer has this interesting quirk where it sends CEF messages that are *technically* valid CEF, but completely ignore the syslog RFC standards that Azure Monitor Agent (AMA) expects. It's a classic case of two vendors implementing standards differently, and there's not much hope of Fortinet changing anything as issues like this have [been a thing](https://community.graylog.org/t/fortinet-cef-formatting-issues/12017) for [some time now](https://www.syslog-ng.com/community/b/blog/posts/parsing-fortigate-logs-and-other-syslog-ng-3-31-news).

The core issue that I encountered was **Missing PRI Headers**. FortiAnalyzer sends raw CEF messages without the RFC3164/RFC5424 compliant syslog PRI headers.

A PRI header is the priority field in syslog messages (like <164>) that combines the facility and severity levels into a single number, telling receiving systems what type of log it is and how urgent it should be treated-and without it, Azure Monitor Agent simply refuses to process the message.

## The Battlefield: Understanding the Data Flow

Before diving into the solution, let's map out the data flow:

```
FortiAnalyzer (malformed CEF)
        ↓ UDP/TCP
    Port 1514
        ↓
rsyslog/syslog-ng (adds PRI header)
        ↓ TCP
    Port 28330
        ↓
Azure Monitor Agent
        ↓
    Sentinel
```

The challenge is intercepting those raw CEF messages and transforming them into something AMA will accept, while simultaneously filtering out the noise before it hits your ingestion costs.

## The Solution: Supporting Both rsyslog and syslog-ng

During implementation across multiple clients now, we discovered that some environments use syslog-ng instead of rsyslog (Ubuntu 24.04 defaults to rsyslog, but administrators may install syslog-ng for its advanced features or simply because that's what they're used to). Here's how to implement the solution for both daemons.

## Step-by-Step Implementation

### Prerequisites

1. **Verify which syslog daemon is running:**
   ```bash
   # Check for rsyslog
   systemctl status rsyslog
   
   # Check for syslog-ng
   systemctl status syslog-ng
   ```

2. **Configure Azure Monitor Agent:**
   - Install AMA on your log collection server
   - Configure the Data Collection Rule (DCR) for LOG_LOCAL4:WARNING
   - Use the "Common Event Format (CEF) via AMA" data connector in Sentinel
   - Ensure the DCR has "Collect messages without PRI header (facility and severity)" checked

### Option A: Using rsyslog

For environments running rsyslog (default on most Linux distributions):

1. **Create configuration file:** `/etc/rsyslog.d/40-fortianalyzer-cef.conf`
2. **Add the configuration** (see script below)
3. **Test syntax:** `rsyslogd -N1`
4. **Reload service:** `sudo systemctl reload rsyslog`

#### Key Configuration Elements

**Receive and Validate:**
```
input(type="imudp" port="1514" ruleset="forti-force-pri-cef")
```

**Force PRI Header Addition:**
```
template(name="FortiCEF_ForcePRI" type="string" 
         string="<164>%TIMESTAMP% %HOSTNAME% FortiAnalyzer: %rawmsg-after-pri%\n")
```

The `<164>` represents local4.warning, matching the DCR configuration. You can adjust this to your own DCR's config with the help of references like [this one](https://techdocs.broadcom.com/us/en/symantec-security-software/identity-security/privileged-access-manager/4-2/reference/messages-and-log-formats/syslog-message-formats/syslog-priority-facility-severity-grid.html).

**Implement Filtering:**
```
if ($rawmsg contains 'type="utm"') then {
    # Forward only UTM security events
}
```

### Option B: Using syslog-ng

For environments running syslog-ng:

1. **Create configuration file:** `/etc/syslog-ng/conf.d/40-fortianalyzer-cef.conf`
2. **Add the configuration** (see embedded script below)
3. **Test syntax:** `sudo syslog-ng -s`
4. **Reload service:** `sudo systemctl reload syslog-ng`

Key differences from rsyslog:
- Uses source/filter/destination/log blocks instead of rulesets
- `flags(no-parse)` preserves raw messages
- `disk-buffer` provides reliable queueing
- `flags(final)` stops further processing

## Update: Severity Filtering Strategy

Initially, I filtered FortiAnalyzer logs at the rsyslog level based on their threat scoring fields (only forwarding logs with `ad.crlevel=critical`, `high`, or `medium`). The goal was to reduce noise and ingestion costs by dropping lower-severity events before they reached Azure.

However, a client implementing this configuration reported they weren't seeing their "Threat logs" in Sentinel. After investigation, we discovered a fundamental misunderstanding about how Fortinet structures its logs.

### The Real Problem: Not All Security Events Have Threat Scores

Here's what I learned: Fortinet doesn't have separate "Threat logs". What people call "Threat logs" are actually just UTM logs that happen to have threat weight scoring fields (`crlevel`, `crscore`, `craction`). 

My original filter:
```bash
if re_match($rawmsg, "ad.crlevel=(critical|high|medium)") then {
    # Forward only logs with threat scoring
}
```

This was excluding many important UTM security events because:
- Not all UTM events trigger threat weight scoring
- Many security events (like certain IPS detections or web filtering) don't get `crlevel` fields
- The filter was looking for something that simply wasn't there in many security logs

### The Working Solution

The fix was simple - forward ALL UTM logs regardless of threat scoring:

```bash
if ($rawmsg contains 'type="utm"') then {
    # Forward all security-relevant events
}
```

This captures:
- UTM logs WITH threat scoring (what the client called "Threat logs")
- UTM logs WITHOUT threat scoring (equally important security events)
- All IPS, antivirus, web filtering, app control, and other security functions

Traffic and system logs are still filtered out, which is where the real noise comes from.

### The Lesson Learned

Don't filter security logs based on severity or threat scores at the ingestion point. You'll inevitably miss important events because vendor severity classifications rarely align with what's actually important for your security posture. 

If you need to reduce volume, do it at the source (FortiAnalyzer) where you have full context: https://docs.fortinet.com/document/fortianalyzer/7.6.3/administration-guide/19991/configuring-log-forwarding

## Testing and Validation

### 1. Verify Port Listeners
```bash
# Check if listening on port 1514
sudo netstat -tlnp | grep 1514
# or
sudo ss -tlnp | grep 1514
```

### 2. Send Test Messages

Simulate FortiAnalyzer's malformed CEF format:

```bash
# Test UTM event (should be forwarded)
echo 'May 23 10:00:00 fortianalyzer CEF:0|Fortinet|FortiGate|7.0.0|13002|Virus Detected|10|type="utm" src=10.0.0.1 dst=8.8.8.8 act=blocked' | nc -w1 localhost 1514

# Test non-UTM event (should be filtered)
echo 'May 23 10:00:00 fortianalyzer CEF:0|Fortinet|FortiGate|7.0.0|00001|Traffic Log|1|type="traffic" src=10.0.0.1 dst=8.8.8.8' | nc -w1 localhost 1514
```

### 3. Monitor Processing

**For rsyslog:**
```bash
# Check debug log if enabled
sudo tail -f /var/log/forti-force-pri.log
```

**For syslog-ng:**
```bash
# View statistics
sudo syslog-ng-ctl stats | grep -E "forti|ama"
```

### 4. Verify AMA Reception
```bash
# Check AMA is running
sudo systemctl status azuremonitoragent

# Verify AMA is listening
sudo ss -tlnp | grep 28330
```

### 5. Validate in Sentinel

Wait 5-15 minutes for ingestion, then query:
```kusto
CommonSecurityLog
| where TimeGenerated > ago(30m)
| where DeviceVendor == "Fortinet"
| where DeviceProduct == "FortiGate"
```

## Troubleshooting

### No logs appearing in Sentinel?
1. Enable debug logging (uncomment line in config)
2. Send test message and check debug file
3. Verify message has 'type="utm"' field
4. Check AMA is running: `systemctl status azuremonitoragent`
5. Remember ingestion delay can be 10-15 minutes

### Syslog-ng syntax errors?
- Remove version/include statements if conflicts occur
- Ensure no nested log statements
- Use `syslog-ng -s` to validate

### High memory usage?
- Adjust queue/buffer sizes in configuration
- Monitor with `syslog-ng-ctl stats` (syslog-ng) or check rsyslog queue stats

### Messages not being filtered correctly?
- Verify the type field in your FortiAnalyzer messages
- Check if FortiAnalyzer is sending the expected CEF format
- Enable debug logging to see raw messages

## Results

After implementing this configuration, the logs are actually making their way through to Sentinel, and we get the following benefits as a side effect:

- **Reduction** in log volumes hitting Sentinel by 70-90% (filtering out traffic logs)
- **Cost savings** on ingestion - only security-relevant events
- **Improved signal-to-noise ratio** for security analysts
- **Support for both rsyslog and syslog-ng** environments

## Gotchas and Lessons Learnt

1. **Test with Debug Logging**: Redirect logs to a local file to debug properly if you run into issues. You'll want visibility into what's actually hitting your syslog daemon before assuming it's working.

2. **Monitor Ingestion Lag**: Remember that Sentinel has an ingestion delay of several minutes. Don't immediately assume logs haven't ingested-wait at least 10-15 minutes before troubleshooting.

3. **DCR Configuration**: Ensure your Data Collection Rule is expecting LOG_LOCAL4:LOG_WARNING and has "Collect messages without PRI header" enabled. There's potential to waste hours troubleshooting if it's configured for a different facility/severity combination.

4. **Choose the Right Daemon**: While both work, and syslog-ng offers more advanced features like disk buffering and better performance under high load, consider your environment's needs when choosing. Don't just switch to syslog-ng for shiggles.

## Wrapping Up

While this is essentially a bit of a hack/workaround, we're addressing a vendor compatibility issue that really shouldn't exist. In the real world of enterprise security, these kinds of workarounds are often necessary to get disparate systems talking to each other.

If you're facing similar FortiAnalyzer integration challenges, hopefully this saves you some of the trial and error that I went through.

## Configuration Scripts

### rsyslog Configuration
<script src="https://gist.github.com/dstreefkerk/07e2c942136f27dff13d04b3f5f33f77.js"></script>

### syslog-ng Configuration
<script src="https://gist.github.com/dstreefkerk/24d120459df9029e3c82fb40dbf9ac6c.js"></script>

Changelog:

- Updated on 11 September 2025 to factor in Syslog-NG configuration as well