
<div align="center">
  <img src="NordVPN%20Blocked%20GNS3%20in%20VirtualBox.png"
       alt="GNS3 VirtualBox networking troubleshooting - NordVPN blocking local network communication"
       width="850">
</div>
  


# 🛡️ GNS3 Network Troubleshooting

### VirtualBox Host-Only Networking & NordVPN

**A real-world cybersecurity and network troubleshooting case study**

![Windows 11](https://img.shields.io/badge/Windows-11-0078D4?logo=windows11&logoColor=white)
![GNS3](https://img.shields.io/badge/GNS3-Networking-blue)
![VirtualBox](https://img.shields.io/badge/VirtualBox-7.2.4-183A61?logo=virtualbox&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-Diagnostics-5391FE?logo=powershell&logoColor=white)
![Status](https://img.shields.io/badge/Network-Resolved-brightgreen)

</div>

---

## 📖 Project Overview

This project documents a real troubleshooting investigation into a GNS3 VM that could not communicate with its Windows 11 host through a VirtualBox Host-Only Adapter.

Despite correct IP addressing and routing, network traffic was blocked by an active NordVPN Windows Filtering Platform rule.

## 🔍 Troubleshooting Process

| Stage | Investigation |
|---|---|
| 1 | VirtualBox network configuration |
| 2 | IPv4 addressing and routing |
| 3 | Windows adapter diagnostics |
| 4 | Linux guest connectivity |
| 5 | Packet Monitor inspection drops |
| 6 | WFP blocking-rule analysis |
| 7 | NordVPN LAN policy |
| 8 | Connectivity verification |

## ✅ Verified Results

| Network Test | Result |
|---|---|
| Windows host-only adapter ping | ✅ Passed |
| GNS3 VM ping | ✅ Passed |
| GNS3 server TCP port 80 | ✅ Passed |
| Nested KVM with Memory Integrity enabled | ⚠️ Unresolved |

## 🛠️ Tools & Technologies

- Oracle VirtualBox
- GNS3
- Windows 11
- PowerShell
- Windows Packet Monitor (Pktmon)
- Windows Filtering Platform (WFP)
- NordVPN

## 📸 Troubleshooting Evidence

See the `screenshots/` folder for redacted screenshots demonstrating the failed connections, diagnostic process, and successful network tests.

See `logs/` for sanitized diagnostic excerpts.

## 🔐 Security Considerations

Personal identifiers, passwords, device GUIDs, and unique filter IDs have been removed from publicly shared material.

Windows Firewall and Memory Integrity were preserved for the final working host-only network configuration.

## 📚 Documentation

See the sections below for the complete technical troubleshooting walkthrough, commands, root-cause analysis, and references.

---

⭐ If this guide helped you troubleshoot a similar virtual networking issue, consider starring the repository.
