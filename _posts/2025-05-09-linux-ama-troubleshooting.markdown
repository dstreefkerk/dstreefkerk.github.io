---
layout: post
title: "Linux Command Reference for System Troubleshooting"
date: 2025-05-09
categories: [linux, azure, monitoring, troubleshooting]
tags: [azure-monitor-agent, syslog, disk-space, networking, commands]
author: "Daniel Streefkerk"
excerpt: "Essential Linux commands for troubleshooting disk space, syslog, and Azure Monitor Agent connectivity issues, updated for May 2025."
---

This reference guide organises essential Linux commands for troubleshooting disk space, syslog, and Azure Monitor Agent connectivity issues. Updated for May 2025 to reflect current Azure Monitor Agent architecture.

## Table of Contents
- [Disk Space Analysis](#disk-space-analysis)
- [Log Rotation & Management](#log-rotation--management)
- [Syslog Configuration & Monitoring](#syslog-configuration--monitoring)
- [Azure Monitor Agent (AMA) Troubleshooting](#azure-monitor-agent-ama-troubleshooting)
- [VM Agent & Azure Connectivity](#vm-agent--azure-connectivity)
- [General System Troubleshooting](#general-system-troubleshooting)
- [File-Based Log Ingestion Troubleshooting](#file-based-log-ingestion-troubleshooting)
- [AMA Troubleshooting Tool](#ama-troubleshooting-tool)

## Disk Space Analysis

### Viewing Disk Usage
```bash
df -h               # Display filesystem usage in human-readable format
du -h --max-depth=1 # Show directory sizes in human-readable format
du -sh *            # Summarize disk usage of each file/directory
ncdu                # Interactive disk usage analyzer with ncurses interface
ls -lah             # List all files with sizes in human-readable format
ls -lahS            # List all files sorted by size (largest first)
find / -type f -size +100M # Find files larger than 100MB
lsof | grep deleted # Find deleted files still held open by processes
```

## Log Rotation & Management
```bash
logrotate -f /etc/logrotate.conf         # Force log rotation immediately
logrotate -d /etc/logrotate.conf         # Test log rotation configuration (debug mode)
cat /etc/logrotate.d/rsyslog             # View rsyslog rotation configuration
tail -f /var/log/syslog                  # Watch syslog file in real-time
grep -r "rotate" /etc/logrotate.d/       # Search for rotation settings in all config files
find /var/log -name "*.gz" -mtime +7     # Find compressed logs older than 7 days
journalctl --disk-usage                  # Show disk usage for systemd journal
```

## Syslog Configuration & Monitoring

### Syslog Service Management
```bash
systemctl status rsyslog               # Check rsyslog service status
systemctl restart rsyslog              # Restart rsyslog service
service rsyslog restart                # Alternative way to restart rsyslog
logger "Test message"                  # Generate a test syslog message
tail -f /var/log/syslog                # Monitor syslog in real-time
grep -i error /var/log/syslog          # Search for errors in syslog
cat /etc/rsyslog.conf                  # View main rsyslog configuration
ls -la /etc/rsyslog.d/                 # List all rsyslog configuration files
```

### Syslog Configuration
```bash
netstat -tuln | grep 514               # Check if rsyslog is listening on syslog ports
ss -tuln | grep 514                    # Alternative to check rsyslog listening ports
cat /var/log/syslog | wc -l            # Count lines in syslog file
grep -v "repeated" /var/log/syslog     # Filter out repeated message notices
journalctl -u rsyslog                  # View rsyslog service logs via journald
```

## Azure Monitor Agent (AMA) Troubleshooting

### AMA Status & Configuration
```bash
systemctl status azuremonitoragent     # Check Azure Monitor Agent service status (mdsd runs as child process)
ps aux | grep azuremonitoragent        # List running AMA processes
ls -la /var/opt/microsoft/azuremonitoragent/log/  # List AMA log files
cat /var/opt/microsoft/azuremonitoragent/log/mdsd.info  # View AMA main info logs
grep -i error /var/opt/microsoft/azuremonitoragent/log/mdsd.err  # Search for AMA errors
ls -la /etc/opt/microsoft/azuremonitoragent/config-cache/  # Check AMA configuration cache
ls -la /etc/opt/microsoft/azuremonitoragent/config-cache/configchunks/  # List DCR configuration fragments
jq '.' /etc/opt/microsoft/azuremonitoragent/config-cache/configchunks/*.json | less  # View DCR configuration fragments (requires jq)
# Get exact AMA extension version
grep '"'Version'"' /var/lib/waagent/$(ls -1 /var/lib/waagent | grep AzureMonitorLinuxAgent | sort | tail -1)/HandlerManifest.json  # Handy when chasing version-specific bugs
```

### AMA Connectivity Testing
```bash
curl -I https://global.handler.control.monitor.azure.com  # Test connectivity to AMA control endpoint
nc -zv global.handler.control.monitor.azure.com 443  # Test TLS reachability to the DCR control plane
curl -H Metadata:true --noproxy "*" "http://169.254.169.254/metadata/instance?api-version=2021-02-01"  # Test IMDS access with API version
systemctl restart azuremonitoragent    # Restart the AMA service to refresh configuration
curl -I https://management.azure.com   # Test connectivity to Azure Resource Manager
curl -I https://login.microsoftonline.com  # Test connectivity to Azure authentication
curl -I https://<region>.monitoring.azure.com  # Test connectivity to metrics endpoint (replace <region>)
cat /var/opt/microsoft/azuremonitoragent/log/fluentbit.log  # Check Fluent Bit logs (data collection)
```

## VM Agent & Azure Connectivity

### Azure VM Agent
```bash
systemctl status walinuxagent         # Check Azure VM Agent status
cat /var/log/waagent.log              # View Azure VM Agent logs (primary location)
cat /var/log/azure/waagent.log        # Alternative location on some distros
grep -i error /var/log/waagent.log    # Search for errors in Azure VM Agent logs
sudo waagent -deprovision+user        # Prepare VM for imaging (CAUTION: destructive)
ls -la /var/lib/waagent/              # List Azure extensions and configuration
ls -la /var/lib/waagent/Microsoft.Azure.Monitor.AzureMonitorLinuxAgent-*/  # Check AMA extension files
```

### Azure Connectivity
```bash
nc -zv login.microsoftonline.com 443  # Test TCP connectivity to Azure authentication
traceroute -T -p 443 management.azure.com  # Trace route to Azure Resource Manager
curl -H Metadata:true "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"  # Test managed identity
# Note: ICMP is usually blocked, prefer using nc or curl instead of ping
dig +short management.azure.com       # DNS resolution test for Azure endpoints
timedatectl status                    # Check system time synchronization (critical for TLS)
```

## General System Troubleshooting

### Process & Resource Monitoring
```bash
top                                   # Display system processes and resource usage
htop                                  # Interactive process viewer (more user-friendly)
ps aux | grep -i azure                # List all Azure-related processes
free -h                               # Display memory usage in human-readable format
uptime                                # Show system load and uptime
dmesg | tail                          # Display kernel messages (useful for hardware issues)
journalctl -f                         # Watch systemd journal in real-time
cat /proc/cpuinfo                     # Display CPU information
```

### Network Troubleshooting
```bash
ip addr                               # Show network interfaces and IP addresses
netstat -tulpn                        # List all listening ports and associated processes
ss -tulpn                             # Modern alternative to netstat
iptables -L                           # List firewall rules
ufw status                            # Check Ubuntu firewall status
cat /etc/resolv.conf                  # View DNS configuration
nslookup management.azure.com         # Test DNS resolution for Azure endpoints
curl -v https://login.microsoftonline.com  # Verbose output of connection to Azure
nc -zv login.microsoftonline.com 443  # More reliable connectivity test than ping
nc -zv global.handler.control.monitor.azure.com 443  # Test AMA control channel connectivity
```

### Systemd Journal (Alternative to Syslog)
```bash
journalctl -u azuremonitoragent       # View AMA service logs in journal
journalctl --disk-usage               # Check journal disk usage
journalctl -f                         # Follow all journal entries in real-time
journalctl -p err                     # Show only error entries
journalctl --vacuum-time=2d           # Clear journal entries older than 2 days
```

### File Operations
```bash
find /var/log -type f -mtime +30 -delete  # Delete log files older than 30 days (dangerous if logrotate keeps symlinks - review first)
grep -r "searchterm" /var/log/            # Search all logs for specific text
wc -l /var/log/syslog                     # Count lines in a log file
cat /var/log/syslog | awk '{print $5}'    # Extract specific column from log
head -n 20 /var/log/syslog                # View first 20 lines of log file
tail -n 20 /var/log/syslog                # View last 20 lines of log file
watch -n 1 "df -h"                        # Monitor disk usage in real-time
```

## File-Based Log Ingestion Troubleshooting

### Check Agent and Child Processes
```bash
systemctl status azuremonitoragent --no-pager    # Verify AMA service status with detailed output
ps -ef | grep -E 'fluent-bit|mdsd|amacoreagent'  # Confirm all child processes are running
pgrep -a 'fluent-bit'                            # Specifically check for Fluent Bit process
journalctl -u azuremonitoragent --since -1h      # Check last hour of AMA service logs
```

### Validate DCR Configuration
```bash
# Check for text and JSON log configurations in DCR fragments
jq '.dataSources | with_entries(select(.key|test("textLogs|jsonLogs")))' \
   /etc/opt/microsoft/azuremonitoragent/config-cache/configchunks/*.json

# List all DCR chunks downloaded to the VM
ls -la /etc/opt/microsoft/azuremonitoragent/config-cache/configchunks/

# Check DCR association from Azure CLI
az monitor data-collection rule association list \
   --scope $(az vm show -g <rg> -n <vm> --query id -o tsv) -o table
```

### Inspect Fluent Bit Configuration
```bash
# View Fluent Bit configuration paths
ls -la /etc/opt/microsoft/azuremonitoragent/config-cache/fluentbit/

# Check INPUT sections in Fluent Bit config (shows monitored files)
grep -A3 -B1 '\[INPUT\]' /etc/opt/microsoft/azuremonitoragent/config-cache/fluentbit/*.conf | less

# View full Fluent Bit configuration
cat /etc/opt/microsoft/azuremonitoragent/config-cache/fluentbit/td-agent.conf | less  # Note: name changed from fluent-bit.conf back in v1.27 - if you're on an earlier hot-fix branch look for that file instead

# Validate Fluent Bit configuration syntax
sudo /opt/microsoft/azuremonitoragent/bin/fluent-bit \
     -c /etc/opt/microsoft/azuremonitoragent/config-cache/fluentbit/td-agent.conf \
     --dry-run
```

### Monitor Log Collection and Processing
```bash
# Watch Fluent Bit logs in real-time
sudo tail -f /var/opt/microsoft/azuremonitoragent/log/fluentbit.log

# Monitor mdsd logs for custom logs processing
tail -f /var/opt/microsoft/azuremonitoragent/log/mdsd.info | grep -i 'Custom'

# Check internal Fluent Bit metrics if enabled
curl -s http://127.0.0.1:2020/api/v1/metrics | grep -E "azure|tail"

# Test file log ingestion
echo "AMA-TEST $(date --iso-8601=seconds)" | sudo tee -a /path/to/your/monitored/log.file
```

### Check for Backlogs and Disk Spooling
```bash
# Check AMA events disk cache size (near 10GB indicates issues)
du -sh /var/opt/microsoft/azuremonitoragent/events

# Check overall AMA directory size
du -sh /var/opt/microsoft/azuremonitoragent/
```

### KQL Queries for Log Analytics
```
# Run in Log Analytics to check DCR errors
DCRLogErrors
| where _ResourceId endswith "/<your-dcr-name>"
| project TimeGenerated, ErrorType, Details

# Test if ingested test data appeared
YourCustomLog_CL 
| where RawData has "AMA-TEST" 
| order by TimeGenerated desc
```

## AMA Troubleshooting Tool

The Azure Monitor Agent (AMA) comes with a built-in troubleshooting tool that can automate many diagnostic checks. Available in AMA versions 1.25.1 and newer, this tool is particularly useful for Ubuntu and Debian-based distributions.

### Locating and Running the Tool
```bash
# Find your AMA version
ls -ltr /var/lib/waagent | grep "Microsoft.Azure.Monitor.AzureMonitorLinuxAgent-"

# Navigate to the troubleshooter directory
cd /var/lib/waagent/Microsoft.Azure.Monitor.AzureMonitorLinuxAgent-<version>/ama_tst/

# Run the troubleshooter in interactive mode (diagnose and display results)
sudo sh ama_troubleshooter.sh -A

# Run the troubleshooter in log collection mode
sudo sh ama_troubleshooter.sh -L
# (Specify a directory like /tmp when prompted for output location)
```

### Manual Download (If Tool Not Present)
```bash
# Download the latest troubleshooter
wget https://github.com/Azure/azure-linux-extensions/raw/master/AzureMonitorAgent/ama_tst/ama_tst.tgz

# Extract the archive
tar -xzvf ama_tst.tgz

# Run the troubleshooter
sudo sh ama_troubleshooter.sh
```

### Checking Prerequisites
```bash
# Verify Python version (required for the troubleshooter)
python3 -V

# Install Python 3 if needed (for Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y python3
```

### Issues Diagnosed by the Troubleshooter

The AMA Troubleshooter can detect and help resolve:

1. **Installation Issues**
   ```bash
   # Check OS compatibility
   lsb_release -a
   
   # Check available disk space
   df -h /var/
   
   # Verify package manager
   which dpkg
   ```

2. **Connection Issues**
   ```bash
   # Test connectivity to required endpoints
   curl -I https://global.handler.control.monitor.azure.com
   
   # Check if IMDS is accessible
   curl -H Metadata:true --noproxy "*" "http://169.254.169.254/metadata/instance?api-version=2021-02-01"
   ```

3. **Heartbeat Issues**
   ```bash
   # Check all required services
   systemctl status azuremonitoragent
   ```

4. **Resource Usage Issues**
   ```bash
   # Check for log rotation
   cat /etc/logrotate.d/azuremonitoragent
   ```

5. **Syslog Collection Issues**
   ```bash
   # Check rsyslog configuration
   ls -la /etc/rsyslog.d/
   ```

6. **Custom Log Collection Issues**
   ```bash
   # Validate DCR configuration
   ls -la /etc/opt/microsoft/azuremonitoragent/config-cache/configchunks/
   ```

### Troubleshooter Output Location
```bash
# Navigate to the output location (if using -L option)
cd /tmp

# Extract the generated archive to review logs
tar -xzvf ama_logs_*.tgz

# Important logs in the archive include:
# - ama_troubleshooter.log: Results of automated checks
# - mdsd.err: MDSD error logs
# - fluentbit.log: Fluent Bit logs for custom log collection
# - waagent.log: VM agent logs
```