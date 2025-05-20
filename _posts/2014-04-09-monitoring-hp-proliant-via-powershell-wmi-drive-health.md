---
layout: post
title: "Monitoring HP ProLiant via PowerShell & WMI: Drive Health"
date: 2014-04-09
categories: technology powershell
tags: hp-proliant powershell prtg systems-administration wmi
author: "Daniel Streefkerk"
excerpt: "How to monitor HP ProLiant server drive health via PowerShell and WMI as an alternative to SNMP monitoring, with implementation details for PRTG Network Monitor."
---

> **Note:** This article was originally written in April 2014 and is now over 11 years old. Due to changes in PowerShell, Windows Server, and HP ProLiant hardware/software over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

This is the first in a series of posts about monitoring HP ProLiant system health via PowerShell and WMI. 

I chose to implement this as an alternative to SNMP since SNMP support in Windows Server is [being deprecated](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/hh831568(v=ws.11)).

> These scripts were written to be used in conjunction with [PRTG Network Monitor](https://www.paessler.com/prtg), but they can just as easily be adapted to display the information in a table by un-commenting a single line.

This functionality depends on the [HP Insight Management WBEM providers](https://support.hpe.com/connect/s/product?language=en_US&kmpmoid=3659620&tab=manuals) being installed on the monitored server. The account you're running the script as will also need permission to do remote WMI queries against the server in question.

Running the script in standalone mode will result in the following output:
*[Image removed during migration: Screenshot showing PowerShell ISE window with output from the script displaying disk health status for multiple drives with Port/Box/Bay information and SAS disk identifiers]*

Out of the box, the script will output the data in XML as needed by PRTG:

```xml
<prtg>
    <result>
        <channel>Port:1I Box:1 Bay:1 - SAS Disk P4W89D4A</channel>
        <value>2</value>
    </result>
    <result>
        <channel>Port:1I Box:1 Bay:2 - SAS Disk P4WA9UUA</channel>
        <value>2</value>
    </result>
    <result>
        <channel>Port:1I Box:1 Bay:3 - SAS Disk P4XWYJSA</channel>
        <value>2</value>
    </result>
    <result>
        <channel>Port:1I Box:1 Bay:4 - SAS Disk P4XX08ZA</channel>
        <value>2</value>
    </result>
    <result>
        <channel>Port:2I Box:1 Bay:5 - SAS Disk P4VX5AVA</channel>
        <value>2</value>
    </result>
    <result>
        <channel>Port:2I Box:1 Bay:6 - SAS Disk P4W9Y78A</channel>
        <value>2</value>
    </result>
</prtg>
```

## Setting up the script for PRTG

1. Copy the script to Custom Sensors\EXE\XML in your PRTG installation folder. Remember to do this on all probes if you have multiple. 
2. Add a new sensor of type **EXE/Script Advanced Sensor** to the device that represents the ProLiant server you wish to monitor. 
3. Make sure the following sensor settings are configured as per below: 
   1. Parameters: %host 
   2. Security Context: 'Use Windows credentials of parent device'

The script is configured to put the entire sensor into an error state if there's a problem with one of the channels. It will also set the sensor message to indicate which disk is experiencing the problem.

Here's the script:

<script src="https://gist.github.com/dstreefkerk/10224178.js"></script>

## Modern Alternatives

Since this post was written in 2014, several better alternatives have emerged:

1. **HPE iLO PowerShell Cmdlets**: HPE now provides official PowerShell modules for ProLiant management that offer more comprehensive monitoring capabilities. The latest version supports Gen8 through Gen11 servers and uses the Redfish API standard.

2. **iLOrest (HPE RESTful Interface Tool)**: An open-source Redfish client scripting tool available for Windows and Linux that provides advanced server management capabilities.

3. **Redfish API**: The industry-standard API for modern server management, which is fully implemented in HPE iLO 5 and 6. It's recommended for new implementations over WMI.

4. **OpenManage Essentials/Enterprise**: Dell EMC's management solution that also supports HP hardware in many cases.

5. **PowerShell Universal Dashboard**: Create interactive dashboards based on PowerShell scripts that can monitor server health.

6. **Container-based monitoring solutions**: Tools like Prometheus and Grafana can be used to create comprehensive monitoring dashboards with alerting capabilities.

## Security Considerations

The original approach has several security aspects to consider:

1. **WMI remote access**: Requires authentication and firewall considerations that may create security risks if not properly configured. Modern environments often restrict these ports.

2. **Credential storage**: The original implementation relies on Windows authentication which may have security implications. Modern solutions support more secure credential management methods.

3. **Outdated components**: The HP WBEM providers mentioned may no longer be maintained or updated. HPE has shifted focus to Redfish API support.

4. **Input validation**: The script should validate host input to prevent potential command injection.

5. **Limited encryption**: WMI communications may not be adequately encrypted, potentially exposing sensitive server health information.

6. **Access control**: Consider implementing additional access controls to restrict who can run these monitoring scripts.

Consider implementing modern security practices when adapting this script for current environments, including secure credential management and validation of all inputs.

## Additional Notes for 2025

- PRTG Network Monitor continues to be available with the latest version being 25.1.104.1961 (April 2025).
- SNMP is still technically available in Windows Server, but has remained a deprecated feature since Windows Server 2012.
- For Gen10 and newer servers, HPE no longer provides WBEM providers. The recommended approach is to use the Redfish API via HPE's PowerShell cmdlets or iLOrest tool.
- WMI-based monitoring is considered legacy and HPE recommends migrating to Redfish API for all server management operations.
- The HPE iLO PowerShell cmdlets module is available from the PowerShell Gallery, making installation and updates much easier than the original approach.