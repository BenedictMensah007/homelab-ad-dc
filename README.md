# Windows Server 2025 Active Directory + Ubuntu Domain-Join Lab

A homelab build on Hyper-V: a Windows Server 2025 Core Domain Controller, managed remotely via Windows Admin Center, with an Ubuntu Server web server joined to the domain and authenticating against it.

## Overview

This lab started as a way to get hands-on with Server Core (no GUI) rather than relying on Desktop Experience, and to practice a real-world pattern: a Windows AD environment with a Linux server as a domain-joined member, not just Windows-only. The goal was to build it, break it, and document every fix along the way — the troubleshooting is arguably the more useful part of this repo than the happy path.

## Architecture

```
Hyper-V Host (Windows 11)
│
├── Internal Virtual Switch + NAT (192.168.100.0/24)
│
├── Windows Server 2025 Datacenter (Core) — DC01
│   ├── Static IP: 192.168.100.10
│   ├── Roles: AD DS, DNS
│   ├── Domain: lab.local
│   └── Managed via Windows Admin Center
│
└── Ubuntu Server 26.04.1 LTS — Web Server
    ├── Static IP: 192.168.100.20
    ├── nginx installed
    └── Domain-joined via realmd/sssd
```

## Steps Taken

1. **Networking foundation** — Discovered the Hyper-V internal switch and VMs were falling back to APIPA (169.254.x.x) addresses with no route out. Configured a NAT gateway on the Hyper-V host (`New-NetIPAddress` + `New-NetNat`) to give the internal `192.168.100.0/24` network real internet access.
2. **Ubuntu install** — Installed Ubuntu Server with a manual static IP configuration, OpenSSH server enabled for remote management, and nginx as the intended web server role.
3. **Domain Controller promotion** — Renamed the Windows Server Core VM, installed the AD DS role, and promoted it to the first DC of a brand-new forest (`lab.local`) using `Install-ADDSForest`.
4. **Remote GUI management** — Installed Windows Admin Center on a separate management PC to get a browser-based GUI for the Core server without giving up the CLI-first workflow.
5. **Active Directory structure** — Created Organizational Units (`staff`, `servers`) and a test user account through WAC's Active Directory tool.
6. **Domain-joined the Ubuntu server** — Installed `realmd`, `sssd`, and related packages, then joined Ubuntu to `lab.local` with `realm join`.
7. **Verified end-to-end authentication** — Logged into the Ubuntu server as a domain user (`user@lab.local`), confirming the full chain: DC → DNS → domain join → Linux authentication.

## Challenges & Fixes

This is the part that actually matters — anyone can follow a checklist, but these are the issues that came up mid-build and how they got resolved.

| Issue | Cause | Fix |
|---|---|---|
| VMs had no internet, only local connectivity | Hyper-V internal switch defaulted to APIPA (link-local only, no gateway) | Set up NAT on the host via PowerShell (`New-NetNat`) and gave the switch a real gateway IP |
| `New-NetIPAddress` returned "Access is denied" | PowerShell wasn't running elevated | Re-ran PowerShell as Administrator |
| Locked out of the DC after a rename/reboot | Forgot the local Administrator credentials | Recovered by signing in as `LAB\Administrator` with the pre-promotion password (domain admin inherits the original local admin credentials) |
| Locked out of the management PC (Microsoft account) | Signed in via PIN, which doesn't validate the actual account password | Changed the password directly at the lock screen's password option to force credential validation |
| Windows Admin Center: "Connection error... WinRM cannot process the request" (0x8009030e) | Management PC wasn't domain-joined, so WinRM defaulted to Kerberos/Negotiate using the wrong (local) credentials | Added the DC's IP to `TrustedHosts` on the client, then used WAC's "Manage as" option to explicitly supply `LAB\Administrator` credentials instead of relying on SSO |
| `realm discover lab.local` → "No such realm found" | Ubuntu's DNS resolver (`systemd-resolved`) was prioritizing a public DNS server (1.1.1.1) over the DC, so AD's SRV records were never queried | Edited netplan to list only the DC as a nameserver and added `search: lab.local`, confirmed with `nslookup lab.local 192.168.100.10` |
| Domain user couldn't log in | The test AD account was disabled by default, and its SAM Account Name contained a space | Enabled the account in WAC's Active Directory tool and corrected the logon name |
| `su` login worked but no home directory created | Ubuntu doesn't auto-create home directories for domain users out of the box | Enabled `pam_mkhomedir` via `sudo pam-auth-update --enable mkhomedir` |

## Skills Demonstrated

- Hyper-V virtual networking (internal switches, NAT)
- Windows Server 2025 installation and Server Core administration via `sconfig` and PowerShell
- Active Directory Domain Services: forest creation, OUs, users, groups
- Windows Admin Center deployment and remote management (WinRM, Kerberos/NTLM troubleshooting)
- Linux-to-AD integration: `realmd`, `sssd`, PAM, netplan/DNS configuration
- Systematic troubleshooting across a mixed Windows/Linux environment

## Screenshots

| File | Description |
|---|---|
| `images/01-adds-overview.png` | Active Directory Domain Services overview in Windows Admin Center, confirming the `lab.local` domain |
| `images/02-ou-structure.png` | AD structure showing `staff` and `servers` OUs with a test user account |
| `images/03-realm-discover.png` | Successful `realm discover lab.local` from Ubuntu after fixing DNS |
| `images/04-realm-join.png` | Successful `realm join` and `realm list` confirming Ubuntu as a domain member (`kerberos-member`) |
| `images/05-domain-login-success.png` | Domain user authenticating successfully on the Ubuntu server |

## Configuration Files

See [`configs/`](./configs) for the sanitized netplan configuration and the PowerShell/bash commands used throughout this build.

---

*Built as a hands-on learning project. Feedback and suggestions welcome.*
