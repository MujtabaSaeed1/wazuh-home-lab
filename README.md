# Wazuh Home Lab: SIEM, File Integrity Monitoring & Automated Brute-Force Response

A self-built home SOC lab: a Wazuh manager on Ubuntu Server monitors a Windows 11 endpoint, detects file tampering in real time, and automatically blocks an SSH brute-force attack launched against the manager itself, all inside VirtualBox on a single machine.

## Architecture

```
┌─────────────────────────┐        Host-only network        ┌──────────────────────────┐
│   Windows 11 (Host)     │◄────────192.168.56.0/24─────────►│  Ubuntu Server 24.04 LTS │
│                          │                                  │   (VirtualBox VM)        │
│  Wazuh Agent             │──── FIM + logs (TCP 1514) ──────►│   Wazuh Manager           │
│  Watches: D:\...\Test    │                                  │   Wazuh Indexer           │
│                          │◄──── Dashboard (HTTPS) ──────────│   Wazuh Dashboard         │
└─────────────────────────┘                                  └──────────────────────────┘
        192.168.56.1                                                 192.168.56.101
```

- **Manager VM:** Ubuntu Server 24.04.5 LTS, VirtualBox, dual-NIC (NAT for internet, Host-only for agent traffic)
- **Agent:** Windows 11, Wazuh agent v4.12.0, deployed via the dashboard's auto-enrollment command
- **Wazuh stack:** all-in-one install (manager + indexer + dashboard) via the official `wazuh-install.sh -a`

## What it does

### 1. File Integrity Monitoring (FIM)

Real-time monitoring of a Windows directory. Every create, modify, and delete is captured with before/after MD5, SHA1, and SHA256 hashes.

### 2. Brute-Force Detection

Simulated an SSH password-guessing attack against the manager (8 loops, roughly 24 failed logins). Wazuh correctly classified it under **MITRE ATT&CK T1110 (Brute Force)** and **T1110.001 (Password Guessing)**.

### 3. Automated Active Response

Configured Wazuh's `firewall-drop` active response to trigger on rule `5712` (sshd brute-force correlation rule). On detection, Wazuh automatically firewalled the attacking IP at the OS level, confirmed by losing my own SSH and dashboard access to the manager mid-attack.

## Setup Summary

1. Built an Ubuntu Server 24.04 VM in VirtualBox (8GB RAM, 2 vCPU, 50GB disk)
2. Configured dual networking: **NAT** (internet access) and **Host-only Adapter** (agent communication), on `192.168.56.0/24`
3. Installed Wazuh 4.12 all-in-one (`wazuh-install.sh -a -i`)
4. Deployed the Windows agent via the dashboard's generated enrollment command (auto-registers and auto-keys, no manual `manage_agents` needed)
5. Added a custom `<directories realtime="yes">` entry to `ossec.conf` for FIM
6. Configured `<active-response>` and `<command>` blocks in the manager's `ossec.conf` to auto-block brute-force source IPs

## Screenshots

| #   | File                          | What it shows                                                                                     |
| --- | ----------------------------- | ------------------------------------------------------------------------------------------------- |
| 1   | `01-dashboard-overview.png`   | Agent connected, severity breakdown                                                               |
| 2   | `02-endpoints-active.png`     | WindowsHost agent, active status, version, IP                                                     |
| 3   | `03-fim-events.png`           | File integrity monitoring: add/modify/delete events with rule IDs 550/553/554                     |
| 4   | `04-attack-terminal.png`      | SSH brute-force attempt from PowerShell, ends in `Connection timed out` once blocked              |
| 5   | `05-threat-hunting-mitre.png` | Threat Hunting dashboard: 44 alerts, MITRE ATT&CK breakdown (Password Guessing, SSH, Brute Force) |
| 6   | `06-dashboard-blocked.png`    | My own dashboard access timing out, proof the block was IP-wide, not just SSH                     |
| 7   | `07-rule-5712-events.png`     | Four distinct brute-force detections (rule 5712, level 10) across separate attack runs            |

## Skills Applied

- **SIEM administration:** Wazuh manager, indexer, and dashboard installation and configuration (all-in-one deployment)
- **Linux system administration:** Ubuntu Server 24.04 install, disk partitioning (LVM), user/SSH setup, systemd service management, log analysis
- **Windows administration:** endpoint agent deployment, service management, directory permissions
- **PowerShell:** scripted the brute-force simulation, service restarts, log filtering, and connectivity testing on the Windows host
- **Bash / Linux CLI:** log grepping and filtering, `systemctl`, `journalctl`, `ip a`, `netplan`, config editing with `nano`
- **Networking:** dual-NIC VM networking (NAT + Host-only), DHCP troubleshooting, static routing concepts, firewall rule inspection
- **Firewall / active response:** `nftables` rule inspection and cleanup, Wazuh active-response scripting
- **Virtualization:** VirtualBox VM creation, disk management, network adapter configuration
- **Detection engineering:** Wazuh rule correlation (XML ruleset analysis), active-response command wiring, MITRE ATT&CK mapping
- **Troubleshooting / root cause analysis:** diagnosed and resolved config load-order errors, a missing command definition, and a script-name mismatch using service logs

## Debugging Notes

Getting active response to actually fire took real troubleshooting:

1. **Wrong rule IDs.** Initially wired active response to rules `5763`/`5716` (generic "authentication failed"), which never correlate on a non-existent user. The correct escalation rule for this attack pattern was `5712` ("brute force ... Non existent user"), found by dumping the full `0095-sshd_rules.xml` ruleset.
2. **Missing `<command>` definition.** `wazuh-analysisd` rejected the config with `Invalid command 'firewall-drop' in the active response`. The `<active-response>` block referenced a command that hadn't been declared yet, because it was defined later in the file. Wazuh's XML parser reads top to bottom, so a reference before its definition is a config error.
3. **Wrong executable name.** The install ships the script as `firewall-drop`, not `firewall-drop.sh`, a one-character mismatch that produced a silent `Active response command not present` log entry instead of a hard error.
4. **Manual firewall cleanup.** After successfully blocking myself (including from the dashboard), the scripted `firewall-drop ... delete` call didn't remove the `nft` rules cleanly. Resolved with `sudo nft flush table ip filter` after locating the rule via `nft list ruleset`.

## Stack

`Wazuh 4.12` `Ubuntu Server 24.04 LTS` `VirtualBox` `Windows 11` `PowerShell` `MITRE ATT&CK`
