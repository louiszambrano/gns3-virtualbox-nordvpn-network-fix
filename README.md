<div align="center">

<img src="assets/NordVPN%20Blocked%20GNS3%20in%20VirtualBox.png" alt="Illustrated thumbnail: NordVPN blocked GNS3 VirtualBox host-only networking" width="850" />

# 🛡️ GNS3 Network Troubleshooting

### VirtualBox Host-Only Networking & NordVPN

**A real-world Windows 11 networking and cybersecurity troubleshooting case study**

![Windows 11](https://img.shields.io/badge/Windows-11-0078D4?logo=windows11&logoColor=white)
![GNS3](https://img.shields.io/badge/GNS3-2.2.61-blue)
![VirtualBox](https://img.shields.io/badge/VirtualBox-7.2.4-183A61?logo=virtualbox&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-Diagnostics-5391FE?logo=powershell&logoColor=white)
![Network status](https://img.shields.io/badge/Host--only_Network-Resolved-brightgreen)

*Thumbnail is a conceptual illustration, not an original packet capture.*

</div>

---

## 📖 Project Overview

My GNS3 VM had valid host-only IP settings, but Windows could not ping the VM, connect to its web service, **or even ping its own host-only address**. After checking VirtualBox, IP routing, the guest, and Windows packet inspection, I traced the outbound block to a **NordVPN local-network WFP filter**. Connectivity was later restored with NordVPN's **Stay invisible on LAN** setting Off.

> **Scope:** This fixes host-only networking. **KVM acceleration remained unresolved** while Windows Memory Integrity was enabled. These are separate issues.

## 🔍 Troubleshooting Process

| Stage | Investigation |
|---|---|
| 1 | VirtualBox host-only and NAT configuration |
| 2 | Windows IPv4 addressing and routing |
| 3 | Local TCP/IP and VirtualBox driver diagnostics |
| 4 | Linux guest networking and neighbor discovery |
| 5 | Windows Packet Monitor (`pktmon`) drops |
| 6 | Windows Filtering Platform (WFP) blocking rule |
| 7 | NordVPN local network policy |
| 8 | Repeatable connectivity verification |

## ✅ Verified Results

| Network test | Before | After |
|---|---|---|
| Ping Windows host-only adapter | General failure | ✅ 0% loss |
| Ping GNS3 VM | General failure | ✅ 0% loss |
| TCP to GNS3 web service (port 80) | Failed | ✅ `TcpTestSucceeded : True` |
| KVM with Memory Integrity enabled | False | ⚠️ Still unresolved |

## 🛠️ Tools & Technologies

Windows 11 · GNS3 · Oracle VirtualBox · PowerShell · `pktmon` · WFP (`netsh wfp`) · NordVPN

## 📸 Evidence and privacy

The `screenshots/` and `logs/` directories can contain **redacted** screenshots and short diagnostic excerpts. If those folders have not yet been uploaded, the case study below is still complete without them. Do **not** upload unreviewed ETL, WFP XML exports, full packet captures, or screenshots showing VM passwords, Windows user profiles, MAC addresses, or account details.

## 📑 Contents

- [Environment](#environment)
- [Initial connectivity failure](#1-initial-connectivity-failure)
- [Packet-drop investigation](#7-identify-packet-drops-using-pktmon)
- [WFP investigation](#8-investigate-windows-filtering-platform)
- [NordVPN root cause](#9-identify-the-blocking-filter)
- [Verified fix](#11-verify-the-solution)
- [KVM limitation](#12-separate-finding-nested-virtualization-and-kvm)
- [Security notes](#privacy-and-security-notes)

---

## 🔬 Full Technical Walkthrough

## Environment

| Component | Configuration |
|---|---|
| Host operating system | Windows 11 |
| Virtualization platform | Oracle VirtualBox 7.2.4 |
| Virtual machine | GNS3VM |
| GNS3 version | 2.2.61 |
| Guest OS | Ubuntu-based GNS3 appliance |
| Adapter 1 | Host-only |
| Adapter 2 | NAT |
| VPN software | NordVPN |
| VM allocation | 4 vCPUs, approximately 8 GB RAM |

Personal identifiers and credentials are removed. The `192.168.56.x` addresses and `Ethernet 6` interface in this guide are private lab examples; substitute your own values. WFP filter identifiers are redacted and differ across systems.

## 1. Initial Connectivity Failure

I started by testing connectivity from Windows to the GNS3 VM.

```powershell
ping 192.168.56.104
```

The result was:

```text
General failure.
General failure.
General failure.
General failure.

Packets: Sent = 4, Received = 0
Lost = 4 (100% loss)
```

Next, I tested the GNS3 web service:

```powershell
Test-NetConnection 192.168.56.104 -Port 80
```

The connection failed:

```text
PingSucceeded    : False
TcpTestSucceeded : False
```

This indicated that the Windows host could not communicate with the guest using the expected network.

## 2. Validate VirtualBox Host-Only Networking

I confirmed the VM's network configuration in VirtualBox Manager:

- Adapter 1: Host-only Adapter
- Adapter 2: NAT
- VirtualBox host-only DHCP server: Enabled
- Host and guest: Same IPv4 subnet

I then inspected the Windows interfaces:

```powershell
Get-NetAdapter |
Format-Table Name,InterfaceDescription,Status
```

And their IPv4 addresses:

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
Format-Table InterfaceAlias,IPAddress,PrefixLength,AddressState
```

The host-only interface had the expected IPv4 address, and Windows reported it as active.

I also discovered a second host-only adapter with a link-local address. Temporarily disabling that additional adapter did not resolve the connection problem.

**Lesson:** An unusual adapter configuration may be worth investigating, but it should not automatically be treated as the root cause.

## 3. Verify Windows Routing

I checked the route to the host-only subnet:

```powershell
Get-NetRoute -AddressFamily IPv4 |
Format-Table DestinationPrefix,InterfaceAlias,NextHop,RouteMetric
```

Windows was selecting the expected host-only interface.

I also verified its IP address:

```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet 6" -AddressFamily IPv4
```

The host-only address was valid and marked `Preferred`.

Basic addressing and routing appeared correct.

## 4. Test the Local Windows Network Stack

I tested Windows loopback:

```powershell
ping 127.0.0.1
```

Result:

```text
Reply from 127.0.0.1: bytes=32 time<1ms TTL=128
```

Loopback worked.

I then tested the Windows host-only interface's own address:

```powershell
ping 192.168.56.1
```

Result:

```text
General failure.
```

That was an important clue.

The failure was not limited to communication with the Linux guest. Windows could not successfully send ICMP traffic to its own host-only IP.

## 5. Verify the Linux Guest Network

Inside the GNS3 VM, I checked the interfaces:

```bash
ip -br addr
```

The host-only interface had an address in the expected subnet, while the second interface had a NAT-assigned address.

I examined the route to Windows:

```bash
ip route get 192.168.56.1
```

The VM selected its host-only interface.

I also checked the neighbor table:

```bash
ip neigh show 192.168.56.1
```

The output showed a cached MAC address entry in the `STALE` state.

This indicated that the guest had learned the host's link-layer address previously, but it did not establish that current IP connectivity was functional.

Pinging the Windows host still failed.

## 6. Investigate Adapter Drivers and Network Filters

I inspected the Windows host-only adapter driver:

```powershell
Get-NetAdapter -Name "Ethernet 6" |
Format-List Name,Status,DriverProvider,DriverVersion
```

Windows reported an active Oracle VirtualBox network adapter without a Device Manager error.

I also inspected the bound network components:

```powershell
Get-NetAdapterBinding -Name "Ethernet 6" |
Format-Table DisplayName,ComponentID,Enabled
```

The adapter had additional packet-filtering components installed, including Npcap and a Realtek lightweight filter.

Each was temporarily tested individually, then restored to its original state.

Neither test fixed the connection.

Restarting the host-only adapter also failed to resolve the problem.

At this point, I stopped changing adapter configuration and focused on packet diagnostics.

## 7. Identify Packet Drops Using Pktmon

Windows Packet Monitor (`pktmon`) provides packet-capture and drop-inspection capabilities.

From Administrator PowerShell:

```powershell
pktmon filter remove
pktmon filter add -i 192.168.56.1
pktmon start --capture --comp all --pkt-size 0
```

I reproduced the problem:

```powershell
ping 192.168.56.1
ping 192.168.56.104
```

Then stopped the capture:

```powershell
pktmon stop
pktmon counters --drop-reason
```

The results showed:

```text
Component: TCPIP
Direction: Tx
Packets Dropped: 8
Drop Reason: Inspection drop
```

This established that the failed test packets were being dropped during Windows TCP/IP inspection.

To inspect individual events, I converted the capture:

```powershell
pktmon etl2txt "PktMon.etl" --out "PktMon.txt"
```

The logs showed dropped ICMP requests involving both the Windows host-only address and the guest.

## 8. Investigate Windows Filtering Platform

Windows Filtering Platform (WFP) provides packet-classification and filtering infrastructure used by Windows and third-party security applications.

I collected a WFP capture:

```powershell
netsh wfp capture start cab=off traceonly=off file="GNS3-WFP.cab"
```

I reproduced the failed network tests and then stopped the capture:

```powershell
netsh wfp capture stop
```

I exported the network events:

```powershell
netsh wfp show netevents file="GNS3-WFP-Events.xml" timewindow=300
```

I then investigated ICMP events associated with the guest:

```powershell
netsh wfp show netevents protocol=1 localaddr=192.168.56.1 remoteaddr=192.168.56.104 file="GNS3-ICMP-Events.xml"
```

The relevant events contained:

```xml
<type>FWPM_NET_EVENT_TYPE_PUBLIC_CLASSIFY_DROP</type>
<classifyDrop>
    <filterId>FILTER_ID_REDACTED</filterId>
    <layerId>48</layerId>
</classifyDrop>
```

The associated action was:

```xml
<actionType>FWP_ACTION_BLOCK</actionType>
```

These events established that a WFP filter was actively blocking the ICMP traffic.

**Important:** WFP filter IDs are runtime-specific and can change. Do not copy a filter ID from another computer and assume it identifies the same rule.

## 9. Identify the Blocking Filter

Next, I exported the current WFP state:

```powershell
netsh wfp show state file="GNS3-WFP-State.xml"
```

I searched that file for the filter ID from the drop event:

```powershell
Select-String -Path "GNS3-WFP-State.xml" -Pattern "<filterId>YOUR_FILTER_ID</filterId>" -Context 100,5
```

The filter's identifying details revealed:

```text
Name:
NordVPN Local Network - Block (AuthConnect v4)

Description:
NordVPN local-network block.

Layer:
FWPM_LAYER_ALE_AUTH_CONNECT_V4

Action:
FWP_ACTION_BLOCK
```

The rule covered private IPv4 network ranges, including the range used by the GNS3 host-only adapter.

This was the key finding.

**A NordVPN network-filtering policy was responsible for the observed ICMP drops.**

NordVPN's application was not actively open, but its background networking services remained present.

## 10. Restore Local Network Connectivity

Rather than manually removing WFP filters or disabling Windows Firewall, I investigated NordVPN's supported local-network settings.

In NordVPN:

**Settings → Connection and security → Stay invisible on LAN**

The setting was shown as Off in the final configuration.

After the NordVPN local-network policy was no longer blocking communication, the network tests succeeded.

The available evidence identifies the responsible filter and confirms the successful final state. It does not establish the precise internal event that caused NordVPN to stop applying the block.

Security note: Allowing local-network access can increase exposure to other devices on the same network. Evaluate the setting before using it on untrusted or public networks.

## 11. Verify the Solution

I repeated the original tests.

**Windows host-only adapter:**

```powershell
ping 192.168.56.1
```

Result:

```text
Packets: Sent = 4, Received = 4
Lost = 0 (0% loss)
```

**GNS3 virtual machine:**

```powershell
ping 192.168.56.104
```

Result:

```text
Packets: Sent = 4, Received = 4
Lost = 0 (0% loss)
```

**GNS3 web service:**

```powershell
Test-NetConnection 192.168.56.104 -Port 80
```

Result:

```text
TcpTestSucceeded : True
```

The successful ICMP and TCP tests confirmed that Windows-to-GNS3VM host-only connectivity had been restored.

## 12. Separate Finding: Nested Virtualization and KVM

During this session, I also investigated an unrelated GNS3VM issue:

```text
KVM support available: False
```

Nested hardware virtualization was enabled in VirtualBox using:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "GNS3VM" --nested-hw-virt on
```

VirtualBox reported the setting as enabled.

Tests also showed that Windows Memory Integrity affected the availability of KVM within the guest.

Disabling Memory Integrity made KVM report `True`, but that reduces an important Windows security protection.

KVM availability therefore remained a separate compatibility issue in the configuration that preserved Memory Integrity.

**Fixing host-only network connectivity did not fix KVM acceleration.**

## Key Lessons

1. **Correct IP addressing and routing do not guarantee connectivity.** Network filters can still block traffic.

2. **Test the host-only interface itself.** Failure to ping the Windows host-only IP provided an important diagnostic clue.

3. **Use packet evidence before changing drivers.** Pktmon and WFP diagnostics helped identify exactly where traffic was being blocked.

4. **VPN software can affect local networking even when the application is closed.** Background filtering policies may remain active.

5. **Avoid removing security controls as a first response.** The networking issue was resolved without disabling Windows Firewall or requiring Memory Integrity to be turned off.

6. **Verify the fix.** Repeat the original ping and TCP tests to ensure the problem is actually resolved.

## Privacy and Security Notes

The following information has deliberately been excluded or anonymized:

- Windows account names and administrative usernames
- Device-specific MAC addresses and interface GUIDs
- Exact diagnostic filter IDs
- Authentication credentials and passwords
- VPN account information
- Personal directories and machine identifiers

Screenshots and diagnostic captures should also be reviewed before publication, since raw logs may contain account information, application paths, security-policy details, or other machine-specific metadata.

Do not publish full WFP state files or packet captures without auditing them.

## References

- [Oracle VirtualBox User Guide](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/)
- [Microsoft — Windows Packet Monitor](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [Microsoft — netsh wfp](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-wfp)
- [NordVPN — Local Network Invisibility](https://support.nordvpn.com/hc/en-us/articles/25202605194257-Invisibility-on-a-local-network-why-we-had-to-build-a-broken-feature)

## Conclusion

The GNS3 host-only network was configured correctly, but communication was blocked by an active NordVPN local-network filtering policy.

The combination of PowerShell, Packet Monitor, and Windows Filtering Platform diagnostics helped isolate the responsible rule and distinguish a VPN filtering problem from VirtualBox addressing and routing problems.

**The main lesson: When host-only virtual networking fails despite correct IP settings, investigate host-level filtering policies before rebuilding virtual adapters or weakening system-wide security protections.**

---

*This case study documents a real troubleshooting session. Account names, hardware identifiers, paths, and unique filter identifiers have been sanitized for public distribution.*
