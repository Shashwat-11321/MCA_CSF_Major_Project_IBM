# Digital Forensics Investigation of a Compromised System
### CVE-2024-3094 · XZ Utils Backdoor · Supply Chain Attack Analysis

> A controlled forensic laboratory built on Proxmox VE to emulate, acquire, and analyze evidence from a memory-resident supply chain compromise. Developed as part of the UPES Internship / Major Project 2026 under the IBM Innovation Center for Education.

---

## Overview

This project constructs a sandboxed digital forensics platform to reproduce and investigate **CVE-2024-3094** — the XZ Utils backdoor discovered in March 2024. The backdoor, introduced via a long-term social engineering supply chain attack, targeted `liblzma` and modified SSH daemon authentication behavior to enable unauthenticated remote code execution on affected Linux systems.

The lab enables end-to-end forensic investigation: from controlled exploit detonation to volatile memory acquisition, packet analysis, and the generation of detection rules mapped to the MITRE ATT&CK framework.

---

## Team

| Name | Enrollment |
|---|---|
| Jai Maniktalla | 590010183 |
| Anandshankar Iyer | 590010637 |
| Shashwat Tripathi | 590011321 |

**Guided by:** Dr. Sumitra (UPES) · Mr. Mohsin Quresh (Industry Mentor)  
**School:** School of Computer Science (SoCS), UPES Dehradun  
**Session:** Internship / Major Project 2026

---

## The Vulnerability

CVE-2024-3094 is a CVSS 10.0 critical supply chain backdoor embedded in `xz-utils` versions **5.6.0** and **5.6.1**. The attacker ("Jia Tan") spent nearly two years building trust as an open-source contributor before injecting malicious code into the release tarballs. The backdoor targeted systemd-linked `sshd` on x86-64 Debian and Fedora systems, hooking `RSA_public_decrypt` via `liblzma` to allow unauthenticated RCE using a hardcoded ED448 private key.

The compromise was caught before reaching stable major distributions by a developer who noticed a 500ms SSH login delay.

---

## Lab Architecture

```
Production Proxmox VE (bare metal)
│
├── Main pfSense          ← PPPoE WAN, production LAN, forensics transit (/30)
│
└── Virtual Proxmox VE   ← Nested hypervisor, isolated lab environment
    │
    ├── Forensics pfSense ← WAN: transit link, LAN: lab segment
    │
    ├── VM: Kali Linux (Attacker)     ← xzbot, Ed448 key generation, exploit delivery
    ├── VM: Victim Linux (Debian)     ← liblzma 5.6.1-1, sshd, forensic target
    └── VM: REMnux (Forensics)        ← Volatility 3, Wireshark, analysis tools
```

Network isolation is enforced at two firewall boundaries. The victim and attacker VMs have no internet access. Only the forensics VM is permitted outbound connectivity for tool acquisition.

---

## Methodology

```
1. Provision & Isolate    →  Proxmox + pfSense lab setup, VLAN segmentation
2. Baseline & Hash        →  SHA-256 all artifacts before compromise
3. Install Backdoor       →  Deploy liblzma 5.6.1-1 from Debian snapshot archive
4. Patch PoC Framework    →  Generate Ed448 keys, patch liblzma.so with xzbot
5. Trigger Exploit        →  Crafted SSH handshake from attacker VM
6. Acquire Evidence       →  RAM dump (hypervisor-level) + .pcap (pfSense)
7. Analyze Artifacts      →  Volatility 3 memory analysis, Wireshark/TShark
8. Correlate & Report     →  Super timeline, MITRE ATT&CK mapping, YARA/Snort rules
```

---

## Tools & Technologies

| Layer | Tool | Purpose |
|---|---|---|
| Hypervisor | Proxmox VE | VM hosting, snapshots, memory acquisition |
| Network | pfSense | Segmentation, packet capture, firewall logging |
| Attacker | Kali Linux + xzbot | Ed448 key generation, exploit delivery |
| Victim | Debian (liblzma 5.6.1-1) | Compromised target system |
| Forensics | REMnux / Ubuntu | Analysis environment |
| Memory | Volatility 3 | Process reconstruction, hook detection, memory parsing |
| Network | Wireshark / TShark | Packet dissection, anomaly identification |
| Detection | YARA + Snort/Suricata | Signature generation and IOC correlation |
| RE (optional) | Ghidra / GDB | Binary and static analysis |

---

## Key Forensic Artifacts

- **liblzma binary signature** — hexdump pattern match confirming backdoor presence
- **SSH timing anomaly** — ~0.30s vs ~0.13s login time delta indicating hook overhead
- **RAM dump** — memory-resident hooks on `RSA_public_decrypt` with no disk trace
- **PCAP evidence** — crafted SSH handshake payload captured at pfSense boundary
- **System logs** — pre/post exploitation correlation for super timeline construction

---

## Detection

### Backdoor Detection Script

```bash
#!/bin/bash
set -eu
path="$(ldd $(which sshd) | grep liblzma | grep -o '/[^ ]*')"
if [ "$path" == "" ]; then echo "probably not vulnerable"; exit; fi
if hexdump -ve '1/1 "%.2x"' "$path" | grep -q \
  f30f1efa554889f54c89ce5389fb81e7000000804883ec28488954241848894c2410
then echo "probably vulnerable"
else echo "probably not vulnerable"
fi
```

### MITRE ATT&CK Mapping

| Technique | ID | Description |
|---|---|---|
| Supply Chain Compromise | T1195.001 | Malicious code in upstream tarball |
| Hijack Execution Flow | T1574 | liblzma hooking RSA_public_decrypt |
| Pre-OS Boot / Firmware | — | Build-time injection via M4 macro |
| Valid Accounts (SSH) | T1078 | Bypassing sshd authentication |

---

## Repository Structure

```
.
├── docs/
│   ├── HLD.pdf                  # High Level Design document
│   └── LLD.pdf                  # Low Level Design document
├── lab-setup/
│   ├── proxmox-config.md        # Proxmox bridge and VM configuration
│   └── pfsense-rules.md         # pfSense firewall rule reference
├── exploit/
│   ├── detect.sh                # Backdoor detection script
│   └── xzbot-setup.md           # xzbot Ed448 patch procedure
├── forensics/
│   ├── volatility-commands.md   # Volatility 3 plugin reference
│   ├── wireshark-filters.md     # TShark/Wireshark filter cheatsheet
│   └── timeline/                # Super timeline outputs
├── detection/
│   ├── rules.yar                # YARA detection rules
│   └── signatures.rules         # Snort/Suricata signatures
└── README.md
```

---

## References

- [Andres Freund — Original Disclosure (oss-security)](https://www.openwall.com/lists/oss-security/2024/03/29/4)
- [Tukaani Project — XZ Backdoor Statement](https://tukaani.org/xz-backdoor/)
- [amlweems/xzbot — PoC Framework](https://github.com/amlweems/xzbot)
- [Kali Linux — XZ Backdoor Getting Started Guide](https://www.kali.org/blog/xz-backdoor-getting-started/)
- [Debian Snapshot Archive — xz-utils 5.6.1-1](https://snapshot.debian.org/package/xz-utils/5.6.1-1/)
- [Volatility 3 Documentation](https://volatility3.readthedocs.io/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- NVD — CVE-2024-3094

---

## Disclaimer

This project is conducted exclusively within a sandboxed academic environment for educational and research purposes. All exploit execution is contained within isolated virtual machines with no exposure to public networks or production systems. The work complies with UPES academic guidelines and ethical research standards.
