#  Security+Digital Forensics Home Lab — Experience Log

A personal lab built to simulate enterprise environments and practise offensive and defensive security techniques hands-on. This repository is an honest log of what I've learned, what worked, what didn't, and where I'm aiming for.

---

##  Lab Infrastructure

| Host | Role | Key Details |
|------|------|-------------|
| **Proxmox PC** | Hypervisor | Hosts Windows 10, Windows Server 2016 (AD DC), and an air-gapped Windows 10 instance for isolated analysis |
| **Windows 10 PC** | Attack targets | Runs intentionally vulnerable VMs sourced from [VulnHub](https://www.vulnhub.com/) |
| **Parrot OS Laptop** | Attack platform | Used for reconnaissance, exploitation, and post-exploitation against lab targets |

### Network topology
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

 ica1 | Difficulty: Easy | link:https://www.vulnhub.com/entry/ica-1,748/

**Source:** VulnHub  
**Date:** 12/06/2026

**Objective**
> "According to information from our intelligence network, ICA is working on a secret project. We need to find out what the project is. Once you have the access information, send them to us. We will place a backdoor to access the system later. You just focus on what the project is. You will probably have to go through several layers of security. The Agency has full confidence that you will successfully complete this mission. Good Luck, Agent!"

**Approach**
**1. Reconnaissance — tools used, what I found**
     INITIALLY:Ran nmap on the target ip address, found Ports 22 for SSH, 80 for http, 3306 and 33060 for mysql were open on the target VM.
**2. Foothold — vulnerability identified, exploit used**
     FOUND: There was a vunerability in the web application(qdPM 9.2), through initial enumeration using gobuster i found /core, through anaylsis of this directory utilising the web browser i eventually found the config               file where the database credentials were stored, using these credentials i logged into the sqlserver bypassing the ssl error using the -skip--ssl option.

     ONCE_LOGGED_IN: I listed all the databases available, the most interesting being labeled "staff", upon inspection of "staff" i found various logins corresponding to the staff names, these were base64 encoded                               therefore i decoded it on my local Parrot OS machine using the 'konsole'.

     CREDENTIALS_FOUND: Once i had access to the various logins i attempted to use all of them through the open SSH port(using hydra) Both Travis and Dexter succeeded.

**3. Privilege escalation — method and path to root/SYSTEM**

   
     TESTING_USERS: Travis did not have sudo(sudo apk update to test) the only file i saw was a redundant file with a text hint therefore i moved onto Dexter. Dexters account also did not have sudo, however it did have a
                    hint too, after reading the hint "cat .txt" which contained something along the lines of, i think one of these applications is suspicious. I then proceeded to run "find / -type f -perm -04000 -ls                           2>/dev/null" which revealed a ordinary list apart from a custom built application(/opt/get_access) after anaylisis of this application(running it) it provided this text:
                                   dexter@debian:~$ /opt/get_access

                                    ############################
                                    ######## ICA #######
                                    ### ACCESS TO THE SYSTEM ###
                                    ############################
                                    
                                    Server Information:
                                    
                                    Firewall: AIwall v9.5.2
                                    OS: Debian 11 "bullseye"
                                    Network: Local Secure Network 2 (LSN2) v 2.4.1 ` All services are disabled. Accessing to the system is allowed only within working hours.
                   I then ran the strings command to properly anaylse what this program was doing, from this i found out it runs the command cat /root/system.info this had to be the way in.
   FINAL: I ran the following commands to create a malicious cat file that can be used to gain root access(Privillege elevation), this worked by shifting where path first looks for cat, it now looked in temp first which             ran my modified application rather than the official one providing me with root access

                   echo -e '#!/bin/bash\n/bin/bash' > /tmp/cat
                  chmod +x /tmp/cat
                  export PATH=/tmp:$PATH

          In order to verify this i used, root@debian:/root# ls which returned:
                                                          root.txt system.info

        
**What worked**
- Successfully employed the use of various pentesting tools such as HYDRA and NMAP 
- Reinforced knowledge about Privillege Escalation

**What didn't work**
- Initially i could not access the mysql server regardless of the fact they were the correct credentials, after spending around 20 minutes looking for other vunerablities i decided to circle back and realised the issue     was the fact i was using a vpn, which with the virtual machine sql server being configured to allow localhost connections only would not work. After disconnecting from the vpn it finally worked and i could progress.

**Key takeaway**
> One concrete thing I'd do differently: i would instead of running agressive nmap scanning techniques employ more stealthy ones to ensure there is no early detection.

---
---

##  Wins

- Set up a fully functional Active Directory domain from scratch including group policy, user accounts, and DNS
- Successfully completed various VulnHub machines from initial recon through to root
- Configured isolated network segments in Proxmox to prevent lab traffic touching the home network
- Successfully decompiled malware using GHIDRA to anaylse its effects(Petra/Goldeneye and the Mischa Ransomware)
- Successfully built a case report on a ransomware case(Case samples built via my intentionally compromised windows VM using a hard drive cloning function), anaylsed using "Autopsy"
- Successfully built a case report on a mock cybercriminals activity using their Phone, PC and location data(Case samples found online, anaylsed using "Autopsy"
- 

---

##  Failures & Lessons

- **network misconfiguration** forgot to turn off vpn in order to access local host
- **accidental detection** accidentally attempted to run sudo from the ssh terminal, the attempt was logged to a fiction file however in a real pentest detection must be avoided.
- 
---

##  Currently Learning

- [ ] Active Directory attack paths — BloodHound graph analysis
- [ ] Windows privilege escalation techniques (WinPEAS, manual checks)
- [ ] Linux privilege escalation techniques (Path, etc)
- [ ] Improve advanced malware analysis in the air-gapped environment
- [ ] Improve knowledge on web application attacks(XSS, SQL-injection) and how to prevent them-- this is already learnt however there is always new vunerablities that come about.

---

##  Goals

- [ ] Complete 10 VulnHub machines across varying difficulty levels
- [ ] Document a full red team exercise against the internal AD domain
- [ ] Introduce SIEM logging (e.g. Splunk Free or Wazuh) to the Proxmox environment for blue team practice
- [ ] Complete Cybersecurity Focused Courses

---

##  Repository Structure

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
