---
layout: post
title: "UniFi Networking: Patterns and Antipatterns"
date: 2025-05-14
categories: [networking, security, best-practices]
tags: [unifi, ubiquiti, vlan, wireless, network-design, security, ai-generated]
author: "Daniel Streefkerk"
excerpt: "A comprehensive guide to best practices and common pitfalls when designing and implementing UniFi network infrastructure, with practical examples for each pattern and antipattern."
---

# UniFi Networking: Patterns and Antipatterns

This is an AI-generated collation of various sources and YouTube videos that I use as a reference for volunteer network admin work that I do in my spare time.

Drawing from official Ubiquiti documentation and well-established community practices, this guide outlines effective patterns to follow and problematic antipatterns to avoid when working with UniFi networking equipment. Each pattern and antipattern includes practical examples to illustrate real-world application.

**Patterns** are proven, reusable solutions that effectively address recurring challenges in network design, while **antipatterns** are seemingly convenient approaches that ultimately create more problems than they solve, often resulting in security vulnerabilities, performance issues, or maintenance difficulties.

## VLAN Design

### Patterns

**Role-Based Network Segmentation**  
Divide your network into VLANs based on device role or user group (e.g. separate VLANs for IoT devices, guests, and staff). This improves security and performance by isolating traffic – for example, an IoT VLAN can be firewalled off from your main LAN to reduce risk. *Example:* Place all smart TVs, lights, and other untrusted devices on an **IoT VLAN** distinct from the corporate LAN, so a compromised IoT device cannot directly reach workstations.

**Dedicated Management VLAN**  
Use a specific VLAN for managing UniFi infrastructure (APs, switches, controllers) instead of the default LAN. This VLAN carries only management traffic and no regular client data, reducing exposure. *Example:* Keep the UniFi controller and device SSH/GUI interfaces on a **Management VLAN 10** that end-user devices do not join, and disallow internet access from this network for added safety.

**Explicit VLAN Trunk Configuration**  
Avoid using the "All" VLAN port profile except on true trunk links between network devices. Configure switch ports to carry only the VLANs needed: set the management VLAN as native (untagged) and tag specific VLANs for other networks on each trunk to APs or downstream switches. This prevents unintended VLAN bleeding across ports. *Example:* On an AP's switch port, native (untagged) VLAN 10 for management, and only tag VLAN 20 (Guest) and VLAN 30 (IoT) that correspond to the SSIDs it broadcasts, rather than allowing all VLANs.

### Antipatterns

**Flat Single Network for All Devices**  
Running everything on one LAN with no VLAN separation undermines security and performance. A flat network means IoT gadgets, guest users, and sensitive devices all intermix, allowing broad access and large broadcast domains. *Example:* Using the default `192.168.1.0/24` for all equipment, where a guest's infected laptop can freely scan or attack an employee's PC due to lack of segmentation.

**Using Default VLAN1 for Production Traffic**  
Relying on the UniFi default "LAN" (usually VLAN 1) for everything is discouraged. Best practice is to leave the default network unused for clients and create custom VLANs for all user/device networks. This avoids ambiguity and potential VLAN 1 leaks across devices. *Example:* Define VLAN 10 as "Office LAN", VLAN 20 as "Guest", etc., and **do not use VLAN 1** for any active devices (aside from possibly the management uplink), to prevent confusion and reduce targetability of the well-known default.

**Trunking All VLANs Everywhere**  
Setting all switch ports to the "All" VLAN profile or tagging every VLAN on every trunk is a bad practice. It's safer to allow only required VLANs per port. Unrestricted trunks can unintentionally extend networks where they shouldn't be, and make network behaviour harder to predict. Instead, explicitly define which VLANs traverse each uplink or trunk port. *Example:* Connecting an insecure IoT device to a port left in "All" (trunk) mode; the device could send tagged packets and access networks it shouldn't. Limiting that port to only the IoT VLAN would avoid this.

## IP Addressing

### Patterns

**One Subnet per VLAN**  
Plan a distinct IP subnet for each VLAN, with a clear logical range (UniFi will prompt for a subnet when you create a new VLAN network). This one-to-one mapping between VLAN and IP subnet ensures clean routing and simplifies firewall rule design. *Example:* Use **192.168.10.0/24** for VLAN 10 (Office LAN), **192.168.20.0/24** for VLAN 20 (IoT), etc., so that no two networks overlap and inter-VLAN traffic can be controlled.

**Consistent IP Scheme & Static Reservations**  
Use an organised IP addressing scheme, reserving a portion of each subnet for static IPs of important devices. It's common to assign the gateway `.1` and use, say, `.2–.50` for infrastructure (via static mapping or DHCP reservation) and the rest for DHCP clients. Ensure key UniFi components (gateway, switches, APs, controller) have stable addresses – either via static assignment or a DHCP reservation – so they're always reachable. *Example:* Reserve **192.168.10.2–10.10** for your UniFi controller and switches on the Office LAN, and configure those as fixed IPs (or DHCP reservations) to prevent IP changes disrupting connectivity.

**Avoid Common Home Subnets**  
Use uncommon private IP ranges to prevent conflicts, especially if you plan site-to-site VPNs or remote user access. Many consumer networks default to 192.168.0.0/24 or 192.168.1.0/24 – it's best to choose a different range (e.g. 10.x.y.0/24 or 172.16.x.0/24) to avoid overlaps. This ensures that when connecting to external networks (homes, clients, or cloud VPNs), you won't encounter IP collisions that break routing.

### Antipatterns

**Overlapping or Improper Subnetting**  
Reusing or overlapping IP ranges between networks will cause routing issues and confusion. Each LAN/VLAN must have a unique subnet. UniFi will even block setting a LAN IP that conflicts with the WAN network. Also avoid using IP blocks that encompass the gateway's own address inadvertently. *Example:* Configuring two different VLANs both as `192.168.50.0/24` will lead to unpredictability and clients not reaching the right gateway (UniFi will flag this as an error).

**All-Dynamic Addresses for Infrastructure**  
Letting critical devices (firewalls, controllers, servers) obtain IPs via DHCP without reservation can result in changed IPs after reboots or outages. This would break adoption and communication in a UniFi network (e.g. APs might lose contact with a moved controller). Always assign predictable IPs to infrastructure via either static mapping or DHCP reservation to avoid this issue. *Example:* The UniFi controller is on DHCP and gets a new IP, causing all UniFi devices to appear "disconnected" because they're still trying the old address.

## DHCP and DNS Configuration

### Patterns

**Single DHCP Server per Network**  
Configure one authoritative DHCP server for each subnet/VLAN – typically the UniFi gateway (UDM/USG) provides this by default when you create a network. Avoid having multiple DHCP sources on the same VLAN. Consistent DHCP ensures clients reliably get addresses and the correct DNS/gateway info. *Example:* If using a UniFi Security Gateway on VLAN 20, disable any DHCP service on other routers or devices in that VLAN so the USG's DHCP can operate without conflict.

**Reliable DNS Settings**  
Use a dependable DNS server for client name resolution. By default, UniFi gateways forward DNS requests to the ISP's server, but it's often better to specify a well-known public DNS (e.g. Cloudflare 1.1.1.1 or Google 8.8.8.8) for faster and more up-to-date responses. If you use a custom internal DNS (such as a Pi-hole or AD DNS server), ensure it forwards queries it cannot resolve – *especially for UniFi cloud services*. Ubiquiti warns that purely internal DNS servers might fail to resolve **ui.com** domains needed for updates or remote management.

**Local DNS Records for Internal Resources**  
Take advantage of UniFi's **Local DNS** feature (introduced in UniFi Network Application 7.2/8.0+) to create DNS hostnames for your important local devices. This lets clients reach services by a friendly name (e.g. **nas.home** instead of an IP). Pair this with DHCP reservations so the host's IP remains fixed. *Example:* Add a local DNS entry mapping **`nas.home`** to your NAS's IP (say 192.168.10.50) in the UniFi controller; ensure the NAS has a reserved address. Now any device using the UniFi gateway for DNS can simply use `nas.home` to connect, and the name will stay valid because the IP is fixed.

**Appropriate DHCP Lease Management**  
Use a reasonable DHCP lease time (e.g. 24 hours) and ensure your pool has enough addresses for all devices on that VLAN. For devices that require stable IPs (printers, cameras, etc.), use DHCP reservations so they always get the same address rather than setting static IPs on the device. This centralises IP management and avoids IP conflicts or expired leases causing downtime.

### Antipatterns

**Rogue or Multiple DHCP Servers**  
Allowing more than one DHCP server on a VLAN (perhaps an ISP modem or another router left on) will lead to dueling offers and unpredictable client IPs. This can manifest as clients getting the wrong gateway or DNS settings. Always disable unused DHCP services and, if supported, enable **DHCP snooping/guard** on UniFi switches to block untrusted DHCP replies. *Example:* Leaving your old wireless router running as an AP **with its DHCP still active**, so devices randomly get an IP from the wrong device (and thus incorrect network settings).

**Misconfigured DNS**  
Pointing clients to a DNS server that is unreliable or filters too aggressively can break network services. For instance, using a custom internal DNS that doesn't know how to resolve Ubiquiti's cloud endpoints will prevent UniFi devices from phone-home updates. Similarly, not specifying any local DNS (and relying only on external DNS) means local device names won't resolve. The antipattern here is neglecting DNS – either by not setting a valid one (causing timeouts) or by using one that blocks domains your network gear needs. Always verify that the chosen DNS servers work for both internet and local name resolution.

## Device Management and Administration

### Patterns

**Strong Unique Credentials**  
Secure all UniFi devices and the controller with non-default, strong passwords. UniFi setup will prompt for admin credentials – use a complex passphrase and enable two-factor authentication on your UniFi Cloud account if using remote cloud access. Never leave devices on factory default login (e.g. **ubnt/ubnt**), as this is a well-known vulnerability. *Example:* Use a unique admin password for your UniFi controller and generate a separate device authentication token for SSH/console access to APs and switches; don't reuse these credentials elsewhere.

**Controller on Stable Always-On Hardware**  
Host the UniFi Network Application on a reliable, always-on device (cloud key, dedicated Raspberry Pi/VM, or an integrated UniFi OS Console). This ensures continuous monitoring and provisioning. Assign the controller a static/reserved IP and **do not allow it to change**. If the controller must be moved, use the proper migration process rather than just changing its IP. Regularly back up your UniFi controller configuration (especially before upgrades) – the UniFi community and documentation emphasise making periodic backups to recover from crashes or mistakes.

**Limited Management Access**  
Segregate management access and use secure methods to reach it. If managing remotely, avoid exposing the UniFi controller's GUI to the internet via port forwards. Instead, use UniFi's cloud access portal or a VPN into the network for administration (this drastically reduces attack surface). For device management, consider firewall rules so that only IT admin subnets or the Management VLAN can reach device login interfaces. *Example:* Use the **unifi.ui.com** cloud portal (which tunnels via HTTPS) or a secure VPN to access your controller, rather than opening port 8443 on your router – it's both simpler and far more secure (industry best practice is to prefer VPN over direct port exposure).

### Antipatterns

**Default or Shared Accounts**  
Using the default admin account for everything, or having multiple admins share one login, is poor practice. This makes auditing impossible and increases risk (e.g. if one person's credentials are compromised). Create individual accounts for each admin and assign appropriate roles. Likewise, forgetting to change default device passwords or leaving fallback accounts active is a serious security hole. *Example:* Leaving a UniFi AP in standalone mode with the default password – it could be accessed by anyone who knows it.

**Ignoring Firmware and Controller Updates**  
Running outdated UniFi Network Application versions or device firmware can leave known vulnerabilities unpatched. An admin should schedule regular updates (after backing up configs) to keep the environment secure and stable. UniFi's release notes often highlight security fixes; not applying these in a timely manner is an antipattern. *Example:* A critical fix for Wi-Fi security is released but the controller is never updated – months later the network is breached via that known exploit. Regular maintenance would have prevented this.

**Ad-hoc Controller Hosting**  
Hosting the UniFi controller on a laptop or desktop that is frequently turned off, or on an IP that changes, will lead to adoption problems and management blind spots. UniFi devices might go "offline" from the controller perspective whenever that machine is off or roaming. This setup is not suitable for a production environment. Always use a dedicated, always-on controller host and keep its IP reachable by the devices (via inform URL or DNS).

## Firewall Rules and Network Security

### Patterns

**Inter-VLAN Traffic Restrictions**  
Adopt a **"deny by default"** stance for traffic between VLANs (the UniFi equivalent of a Zero Trust approach). By default, UniFi allows all LAN networks to communicate; a best practice is to create firewall rules dropping unsolicited inter-VLAN traffic and then permit only what's necessary. You can either enable **"Network Isolation"** on a VLAN network (which automatically blocks it from reaching other local networks) or manually craft ACL rules to achieve the same with more granularity. *Example:* Implement a rule to **"Drop any traffic from IoT VLAN to Corporate VLAN"** (while still allowing IoT devices to reach the internet and specific services like a local DNS or printer if needed via separate allow rules). This contains potential breaches to their VLAN of origin.

**Use of Guest & Client Isolation for Untrusted Networks**  
Leverage UniFi's built-in guest controls for guest Wi-Fi or any untrusted network. Mark networks as **Guest** or enable **Network Isolation** to automatically limit access to only the internet. Additionally, enable **Client Device Isolation** on guest Wi-Fi SSIDs so that wireless clients cannot even talk to each other locally. These measures enforce containment at multiple layers (wireless and wired). *Example:* A "Guest" WLAN assigned to an isolated Guest VLAN will, by default, prevent guests from reaching your LAN subnets, especially if you checked "Apply guest policies" – this sets up firewall rules to block private subnet access and only allow DHCP/DNS for that VLAN.

**Firewall Grouping and Zones**  
Take advantage of UniFi's firewall groups or the newer **Zone-Based Firewall** capability on UniFi gateways. Group networks of similar trust levels into zones (e.g. a "LAN" zone for internal trusted networks vs. an "IoT" zone for all untrusted device VLANs) and apply inter-zone policies. This can simplify rules by applying one rule to an entire category of networks. Likewise, define **address groups** (aliases) for common destinations (like an RFC1918 "InternalNetworks" group) to use in multiple rules. This reduces error and keeps the rule-set concise and readable.

### Antipatterns

**All-Or-Nothing Trust**  
An antipattern is leaving the default stance of "all LAN networks can talk freely" in place for too long, effectively trusting everything inside. Without internal firewalls, a breach in one VLAN can pivot to others. Another antipattern is creating overly broad allow rules (like an allow **any**/**any** rule on LAN to LAN) that defeat the purpose of segmentation. This often happens if an admin tries to "fix" connectivity issues by opening all ports – it's better to identify required services and only allow those. Remember, start with isolation and add exceptions, not the reverse.

**Misusing the "Isolated" Network Toggle**  
While the **Network Isolation** setting is useful for quick setup, treating it as a one-click security solution without understanding it can be problematic. If you enable isolation on a network but then manually add liberal allow rules, you can completely negate the isolation. Additionally, the isolation toggle only affects lateral LAN access – an "isolated" network can still access any allowed internet destinations. Assuming it does more than it actually does is an antipattern. *Example:* Marking an IoT network as *isolated* but then creating a rule to allow IoT to query an internal server by IP; this custom rule will override the isolation and could inadvertently open more access than intended. In such cases, a more precise rule (allowing only a specific port to that server) or maintaining the default deny except that port is the proper approach.

**Weak Egress Controls**  
Not implementing any egress/outbound firewall rules can be a missed opportunity for security. By default, internal networks can reach out to anywhere on the internet. Best practice is to at least block known malicious destinations or unnecessary outbound services. UniFi's **Threat Management** (IDS/IPS) can help here if enabled. An antipattern is ignoring these features – for instance, allowing an IoT camera to make unrestricted outbound connections when it should only talk to a specific cloud service. Always consider tightening outbound policies for vulnerable networks (though carefully to avoid breaking legitimate usage).

## Wi-Fi Deployment

### Patterns

**Minimal SSIDs with VLAN Mapping**  
Keep the number of Wi-Fi SSIDs as low as possible to reduce overhead. Each additional SSID beacons and consumes airtime. It's more efficient to use one SSID per distinct user group or access policy, and use VLAN tagging to separate their traffic, rather than creating many SSIDs. As a rule of thumb, **3 SSIDs per band** is a reasonable upper limit before performance suffers. *Example:* Have one primary corporate SSID, one guest SSID, and perhaps one IoT SSID – each mapped to its own VLAN. Avoid scenarios like different SSIDs for every little subgroup or department.

**Strong Security and Modern Encryption**  
Use WPA2 or WPA3 security with a strong passphrase (or 802.1X authentication for enterprise) on all non-guest networks. Open networks should be restricted to truly public guest access and even then should be isolated. Do **not** use deprecated protocols like WEP or WPA/TKIP – these are insecure and not supported by UniFi APs in recent firmwares. If IoT devices only support older standards, place them on a separate SSID/VLAN with tight firewall rules. *Example:* Enable **WPA3** security on your main SSID if all clients support it (it offers stronger encryption), or use **WPA2-PSK AES** at minimum. For a guest hotspot, leave it open **only** if you have guest isolation and a captive portal in place (never open your primary Wi-Fi).

**Band Steering and Fast Roaming**  
Take advantage of UniFi's band steering and roaming features to optimise client experience. **Band Steering** pushes dual-band clients to 5 GHz, which usually has more capacity and less interference. **Fast Roaming (802.11r)** helps clients seamlessly hand off between APs. These should generally be kept enabled for most deployments. *Example:* With band steering on, a dual-band laptop will connect on 5 GHz instead of congested 2.4 GHz, and 802.11r allows a VoIP call to continue uninterrupted as the user moves between AP coverages.

**Proper Channel Planning and Width**  
Configure Wi-Fi channels to minimise interference. In low-density areas, using Auto channel is fine, but in high-density environments manually set 2.4 GHz channels to 1, 6, 11 (non-overlapping) for multiple APs. For 5 GHz, ensure adjacent APs don't all sit on the same channel. UniFi's AI auto-tuning can be helpful, but always verify it. Use **20 MHz width on 2.4 GHz** (to avoid overlap) and typically **40–80 MHz on 5 GHz** for a balance of speed and stability. Very wide channels (160 MHz) are usually not worth the interference unless you're in a clean spectrum environment. *Example:* In an office with three APs, set one AP's 2.4 GHz to channel 1, the second to 6, the third to 11. On 5 GHz, use channels like 36, 52, 149 to avoid co-channel interference. Keep 5 GHz at 80 MHz width for throughput – using 160 MHz in a city apartment block would conflict with neighboring networks.

**Sufficient AP Coverage with Controlled Power**  
Deploy enough access points to cover your area with overlapping coverage, but **turn down transmit power** on 2.4 GHz (and possibly 5 GHz) if APs are very close to each other. It's often better to have more APs at lower power than a few APs at max power, which can cause interference and client sticky issues. Use the UniFi coverage map or a mobile app like WiFiman to survey signal strengths. Aim for at least -65 to -70 dBm at edge of coverage for stable connections. *Example:* In a multi-floor office, put one NanoHD AP per floor instead of one AP on max power trying to cover two floors – this yields more uniform coverage and capacity. Set 2.4 GHz power to Low or Medium so that devices connect to the nearest AP instead of hearing a distant AP at high power.

**Wired Backhaul Preferred**  
Whenever possible, connect APs via Ethernet backhaul rather than mesh/repeater wireless links. Wired backhaul offers full bandwidth and zero relay latency. If you must use wireless uplinks (mesh), use them sparingly and ensure a strong signal between mesh nodes (at least -60 dBm or better). Each mesh hop will halve throughput, so limit to one hop if you can. *Example:* Instead of wirelessly meshing three APs in a chain, wire each AP to the switch if feasible. If a mesh is needed for one far AP in a shed, ensure the bridging AP is nearby with clear line-of-sight and consider using a dedicated wireless bridge device for that uplink to not steal capacity from client service.

### Antipatterns

**Overloading with SSIDs or Excessive Features**  
A common Wi-Fi antipattern is creating too many SSIDs "just in case" (for every department, user, or IoT type). This leads to channel overhead from beaconing on each SSID and can degrade performance. Similarly, enabling unnecessary advanced features or lower data rates can harm performance – for instance, leaving 2.4 GHz **802.11b** rates enabled (1 Mbps, 2 Mbps) lets very old devices connect but will slow down airtime for everyone. It's usually best to disable very low legacy rates so that even 2.4 GHz clients use 11 Mbps or higher. *Example:* Five separate IoT SSIDs for different device classes (thermostats, lights, cameras, etc.) – instead, use one IoT SSID and segregate devices by VLAN if needed at the IP level, not SSID.

**Improper Channel and Power Usage**  
Setting channels or widths without regard to the RF environment can cause self-interference. For instance, using a 40 MHz channel on 2.4 GHz will consume two-thirds of the band and overlap with your neighbors – generally a bad idea unless you have zero nearby networks. Likewise, blasting all APs at max power will cause them to hear each other and create a noisy environment with clients flapping between far APs. Running auto-channel scans at default settings in a very busy environment (e.g. Auto Optimisation at night) without manual tweaking can also yield suboptimal results. *Example:* Relying on out-of-the-box auto settings in a dense apartment complex – it might put all APs on the same channel or choose a wide channel that overlaps. The administrator should manually intervene or fine-tune thresholds in such cases.

**Ignoring Client Capabilities**  
Another antipattern is not considering the actual devices on your Wi-Fi. For example, enabling only WPA3 could cut off older clients, or using DFS channels (which some IoT clients don't support) might cause them not to connect. Or, an admin leaves all bands on for an IoT SSID even though those IoT devices only work on 2.4 GHz – the devices might waste time trying 5 GHz. Always match your Wi-Fi design to client needs. *Example:* If you have a bunch of 2.4 GHz-only IoT gadgets, **do not** combine them on an SSID that has band steering to 5 GHz – they'll never use it. Instead, you might create an IoT SSID locked to 2.4 GHz (with 20 MHz width) for those devices, and keep your main SSID dual-band for modern devices. In short, **know your clients** – design the WLAN (security, bands, rates) to suit what you actually have.

## References and Further Reading

- [Ubiquiti UniFi Network Controller Documentation](https://help.ui.com/hc/en-us/categories/200320654-UniFi-Network)
- [UniFi - Best Practices for Managing Firewall Rules](https://help.ui.com/hc/en-us/articles/115003077688-UniFi-Best-Practices-for-Managing-Firewall-Rules)
- [UniFi - Understanding Advanced Features](https://help.ui.com/hc/en-us/articles/360046183733-UniFi-Understanding-Advanced-Features)
- [UniFi - Understanding and Implementing VLANs](https://help.ui.com/hc/en-us/articles/219654087-UniFi-Using-VLANs-with-UniFi-Wireless-Routing-Switching-Hardware)
- [UniFi - Setting Up Guest Wireless Networks](https://help.ui.com/hc/en-us/articles/115000166827-UniFi-Guest-Network-Guest-Portal-and-Hotspot-System)
- [UniFi Network - Designing Networks for IoT Devices](https://help.ui.com/hc/en-us/articles/360015253653-UniFi-Best-Practices-for-IoT-Devices)
- [Ubiquiti Community Forums - VLAN Best Practices](https://community.ui.com/questions/VLAN-Best-Practices/8d4814c7-3bd0-4a16-93a7-0a0ff9c07225)
- [UniFi - Recommended Wireless Settings](https://help.ui.com/hc/en-us/articles/221321728-UniFi-Understanding-and-Implementing-Wireless-Settings)
- [UniFi - Zone-Based Firewall Implementation Guide](https://help.ui.com/hc/en-us/articles/8423790771991-UniFi-Network-How-to-Configure-Zone-Based-Firewall)
- [UniFi - WiFi Performance Optimization Guide](https://help.ui.com/hc/en-us/articles/221029967-UniFi-Troubleshooting-Wi-Fi-Connectivity)
- [Ubiquiti Community - UniFi Network Expert Diagrams & Topologies](https://community.ui.com/questions/Collection-of-UniFi-Diagrams-and-Topologies/13a693d6-7093-41da-83c7-9926928dba83)
- [Ubiquiti - Secure Network Design White Paper](https://dl.ui.com/qsg/UniFi-Security/UniFi_Network_Secure_Setup_Guide.pdf)
- [IEEE 802.11 Standards Reference](https://standards.ieee.org/ieee/802.11/7028/)
- [NIST SP 800-153: Guidelines for Wireless Network Security](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-153.pdf)
- [Wireless Spectrum Management Best Practices](https://www.cisco.com/c/en/us/td/docs/wireless/controller/8-5/config-guide/b_cg85/radio_resource_management.html)
- [Enterprise VLAN Security Implementation Guide](https://www.cisecurity.org/insights/white-papers/implementing-network-segmentation-and-segregation)