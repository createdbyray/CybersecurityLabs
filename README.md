# Security + Digital Forensics Home Lab — Experience Log

A personal lab built to simulate enterprise environments and practise offensive and defensive security techniques hands-on. This repository is an honest log of what I've learned, what worked, what didn't, and where I'm aiming for.

---

## Lab Infrastructure

| Host | Role | Key Details |
|------|------|-------------|
| **Proxmox PC** | Hypervisor | Hosts Windows 10, Windows Server 2016 (AD DC), and an air-gapped Windows 10 instance for isolated analysis |
| **Windows 10 PC** | Attack targets | Runs intentionally vulnerable VMs sourced from [VulnHub](https://www.vulnhub.com/) |
| **Parrot OS Laptop** | Attack platform | Used for reconnaissance, exploitation, and post-exploitation against lab targets |

### Network Topology
- Proxmox environment simulates a small enterprise: domain controller, domain-joined clients, and a segmented air-gapped endpoint
- VulnHub machines run on an isolated host-only network — no external exposure
- Parrot OS attacks are contained entirely within the lab

---

## Skills & Tools

| Category | Tools / Concepts |
|----------|-----------------|
| **Reconnaissance** | Nmap, Netdiscover, enum4linux, OSINT |
| **Exploitation** | Metasploit, manual CVE exploitation, Burp Suite |
| **Active Directory** | BloodHound, PowerView, Kerberoasting, Pass-the-Hash, privilege escalation |
| **Post-exploitation** | Mimikatz, lateral movement, persistence techniques |
| **Blue team / Defence** | Log analysis, Windows Event Viewer, basic incident response |
| **Virtualisation** | Proxmox VE, VMware, VirtualBox, network segmentation |
| **Operating systems** | Kali/Parrot Linux, Windows 10, Windows Server 2016 |

---

## Machine Write-ups & Lab Exercises

Each entry follows the same structure: what I attempted, what worked, what didn't, and the key takeaway.

---

### ICA1 | Difficulty: Easy | [VulnHub Link](https://www.vulnhub.com/entry/ica-1,748/)

**Source:** VulnHub  
**Date:** 12/06/2026

**Objective**
> "According to information from our intelligence network, ICA is working on a secret project. We need to find out what the project is. Once you have the access information, send them to us. We will place a backdoor to access the system later. You just focus on what the project is. You will probably have to go through several layers of security. The Agency has full confidence that you will successfully complete this mission. Good Luck, Agent!"

**Approach**

**1. Reconnaissance**

Ran Nmap on the target IP address, found ports 22 (SSH), 80 (HTTP), 3306 and 33060 (MySQL) were open on the target VM.

**2. Foothold**

There was a vulnerability in the web application (qdPM 9.2). Through initial enumeration using Gobuster I found `/core`. Through analysis of this directory using the web browser, I eventually found the config file where the database credentials were stored. Using these credentials I logged into the SQL server, bypassing the SSL error using the `--skip-ssl` option.

Once logged in, I listed all the databases available — the most interesting being labelled `staff`. Upon inspection I found various logins corresponding to staff names. These were Base64 encoded, so I decoded them on my local Parrot OS machine using the terminal.

With the decoded credentials, I attempted to authenticate each one via the open SSH port using Hydra. Both Travis and Dexter succeeded.

**3. Privilege Escalation**

Travis did not have sudo access (`sudo apk update` to test) — the only file present was a redundant file with a text hint, so I moved on to Dexter. Dexter's account also lacked sudo, but contained a hint file which read along the lines of: *"I think one of these applications is suspicious."*

I then ran:
```bash
find / -type f -perm -04000 -ls 2>/dev/null
```
This revealed a standard list apart from a custom-built application at `/opt/get_access`. Running it produced:

```
############################
######## ICA ########
### ACCESS TO THE SYSTEM ###
############################

Server Information:

Firewall: AIwall v9.5.2
OS: Debian 11 "bullseye"
Network: Local Secure Network 2 (LSN2) v2.4.1
All services are disabled. Accessing to the system is allowed only within working hours.
```

Running `strings` on the binary revealed it executes `cat /root/system.info` — this had to be the way in.

I created a malicious `cat` file in `/tmp` and prepended it to `$PATH`, so the system would execute my version instead of the real one:

```bash
echo -e '#!/bin/bash\n/bin/bash' > /tmp/cat
chmod +x /tmp/cat
export PATH=/tmp:$PATH
```

Running `/opt/get_access` then dropped me into a root shell. Verified with:
```bash
root@debian:/root# ls
root.txt  system.info
```

**What worked**
- Successfully employed various pentesting tools including Hydra and Nmap
- Reinforced knowledge of privilege escalation via PATH hijacking

**What didn't work**
- Initially could not access the MySQL server despite using the correct credentials. After spending around 20 minutes looking for other vulnerabilities, I circled back and realised the issue was an active VPN connection — the VM's SQL server was configured to allow localhost connections only, so the VPN routing prevented access. Disconnecting from the VPN resolved it.

**Key takeaway**
> Instead of running aggressive Nmap scanning techniques from the outset, I would employ stealthier options to reduce the risk of early detection.

---

## Wins

- Set up a fully functional Active Directory domain from scratch including group policy, user accounts, and DNS
- Successfully completed various VulnHub machines from initial recon through to root
- Configured isolated network segments in Proxmox to prevent lab traffic touching the home network
- Successfully decompiled malware using Ghidra to analyse its effects (Petya/GoldenEye and the Mischa Ransomware)
- Successfully built a case report on a ransomware incident — case samples created via an intentionally compromised Windows VM using a hard drive cloning function, analysed using Autopsy
- Successfully built a case report on a mock cybercriminal's activity using their phone, PC, and location data — case samples sourced online, analysed using Autopsy

---

## Failures & Lessons

- **Network misconfiguration** — forgot to disable VPN in order to access localhost services
- **Accidental detection** — accidentally attempted to run `sudo` from the SSH terminal; the attempt was logged to a log file. In a real pentest, detection must be avoided at all times.

---

## Currently Learning

- [ ] Active Directory attack paths — BloodHound graph analysis
- [ ] Windows privilege escalation techniques (WinPEAS, manual checks)
- [ ] Linux privilege escalation techniques (PATH hijacking, etc.)
- [ ] Advanced malware analysis in the air-gapped environment
- [ ] Web application attacks (XSS, SQL injection) and mitigations — foundational knowledge is in place, but new vulnerabilities emerge constantly

---

## Goals

- [ ] Complete 10 VulnHub machines across varying difficulty levels
- [ ] Document a full red team exercise against the internal AD domain
- [ ] Introduce SIEM logging (e.g. Splunk Free or Wazuh) to the Proxmox environment for blue team practice
- [ ] Complete cybersecurity-focused courses

---

## Repository Structure

```
/
├── writeups/          # Individual machine and exercise write-ups
│   ├── vulnhub/       # VulnHub machine logs
│   └── ad-lab/        # Active Directory lab exercises
├── notes/             # Reference notes (tools, commands, techniques)
└── scripts/           # Custom scripts written during exercises
```

---

## ⚠️ Disclaimer

All activity documented here is performed in a private, isolated lab environment against systems I own or have explicit permission to test. Nothing here is used against live systems or external networks.

---

*This is a living document — updated as I learn, break things, and figure out why.*
