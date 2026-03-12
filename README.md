# SOC-Automation-Lab
Automated SOC pipeline using Wazuh, Shuffle, and TheHive for real-time threat detection and response
# SOC Automation Lab 🛡️

## Overview
A fully automated Security Operations Center (SOC) pipeline built 
using open-source tools to detect, enrich, and respond to 
cybersecurity threats in real time — without any manual intervention.

When Mimikatz runs on the monitored Windows machine, the entire 
pipeline triggers automatically within seconds.

---

## Architecture
```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   🖥️ Windows 10 VM                                 │
│   Mimikatz executed                                 │
│          │                                          │
│          │ Sysmon captures process creation         │
│          ↓                                          │
│   🔍 Wazuh Agent                                   │
│          │                                          │
│          │ Ships logs to Wazuh Manager              │
│          ↓                                          │
│   🖥️ Wazuh Server (Ubuntu 22.04)                   │
│   Custom rule 100002 fires — Level 15 Alert         │
│          │                                          │
│          │ Webhook to Shuffle                       │
│          ↓                                          │
│   🤖 Shuffle SOAR (shuffler.io — cloud)            │
│          │                                          │
│    ______|______________________                    │
│    │                           │                   │
│    ↓                           ↓                   │
│ 🦠 VirusTotal            🌐 ngrok tunnel           │
│ Hash enrichment          Exposes local TheHive      │
│                               │                    │
│                               ↓                    │
│                        📋 TheHive Server           │
│                        (Ubuntu 22.04)              │
│                        Case created                │
│                               │                    │
│                               ↓                    │
│                        📧 Email Alert              │
│                        Sent to analyst             │
└─────────────────────────────────────────────────────┘
```

---

## Why ngrok?
Shuffle runs in the **cloud** (shuffler.io). TheHive runs **locally** 
on a private VM with no public IP. ngrok creates a secure tunnel that 
gives TheHive a temporary public URL so Shuffle can reach it and 
create cases automatically.

---

## Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| Wazuh | 4.7 | SIEM + EDR — detects threats |
| TheHive | 5 | Case Management — tracks incidents |
| Shuffle | Latest | SOAR — automates the response |
| Sysmon | Latest | Windows telemetry — rich logging |
| VirusTotal API | v3 | Threat intelligence — hash lookup |
| ngrok | Latest | Tunnel — exposes TheHive to Shuffle |

---

## Infrastructure

| VM | OS | RAM | Disk | Role |
|----|-----|-----|------|------|
| Wazuh-Server | Ubuntu 22.04 | 4GB | 50GB | SIEM Manager |
| TheHive-Server | Ubuntu 22.04 | 6GB | 50GB | Case Management |
| Windows10-Client | Windows 10 | 4GB | 50GB | Monitored Endpoint |

All VMs run on VMware Workstation using NAT networking.

---

## How It Works — Step by Step

### 1. Attack Simulation
Mimikatz is executed on the Windows 10 VM.
Sysmon captures the process creation event including:
- Original filename
- SHA256 hash
- User account
- Timestamp
- Parent process

### 2. Detection
Wazuh Agent ships the Sysmon log to Wazuh Manager.
Custom rule 100002 matches on `originalFileName: mimikatz.exe`
using PCRE2 regex — even if the file is renamed.
Alert fires at **Level 15** (maximum severity).
Maps to **MITRE ATT&CK T1003 — OS Credential Dumping**.

### 3. Automation
Wazuh sends the alert JSON to Shuffle via webhook.
Shuffle workflow executes automatically:

**Step 1** — Webhook receives Wazuh alert JSON

**Step 2** — Regex extracts SHA256 hash from the alert

**Step 3** — VirusTotal API looks up the hash
- Confirms: `trojan.mimikatz/hack`
- Reputation score: -66
- 5 sandbox verdicts: all malicious

**Step 4** — Shuffle calls TheHive via ngrok tunnel
- ngrok exposes local TheHive VM to the cloud
- Case created automatically with full enriched data

**Step 5** — Email sent to analyst with:
- Date and time of attack
- Affected device name
- User account involved
- Malicious file detected

---

## Detection Rule
```xml
<group name="sysmon,">
  <rule id="100002" level="15">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.originalFileName" 
           type="pcre2">(?i)mimikatz\.exe</field>
    <description>
      Mimikatz Usage Detected on $(win.system.computer)
    </description>
    <mitre>
      <id>T1003</id>
    </mitre>
  </rule>
</group>
```

---

## ngrok Setup

ngrok was used to expose the local TheHive instance to 
Shuffle running in the cloud.
```bash
# Install ngrok
snap install ngrok


# Expose TheHive port 9000
ngrok http 9000
```

The generated ngrok URL was used as TheHive's 
base URL inside the Shuffle workflow.

Example:
```
https://abc123.ngrok.io → http://192.168.75.20:9000
```

---

## Shuffle Workflow

The workflow contains 5 nodes connected in sequence:
```
[Webhook] → [Extract Hash] → [VirusTotal] → [TheHive] → [Email]
```

| Node | Tool | Action |
|------|------|--------|
| 1 | Webhook | Receives Wazuh alert |
| 2 | Regex | Extracts SHA256 hash |
| 3 | VirusTotal | Gets hash report |
| 4 | TheHive | Creates alert case |
| 5 | Email SMTP | Notifies analyst |

---

## Screenshots

### Wazuh Dashboard
![Wazuh Dashboard]
<img width="2227" height="1185" alt="Screenshot 2026-03-12 131305" src="https://github.com/user-attachments/assets/f42e6e0e-47b0-4a66-9a95-ba277107163c" />

### Wazuh Agent Connected
![Wazuh Agent]
<img width="2217" height="1156" alt="Screenshot 2026-03-12 131151" src="https://github.com/user-attachments/assets/0059aa1b-efff-43b6-97c6-8ebedcaf2477" />

### Mimikatz Alert Fired
![Wazuh Alert]
<img width="1112" height="628" alt="8" src="https://github.com/user-attachments/assets/c7f3b083-4ac2-4ca1-a3f8-ddde90ecb86c" />

### Shuffle Workflow
![Shuffle Workflow]
<img width="1660" height="1049" alt="Screenshot 2026-03-12 131416" src="https://github.com/user-attachments/assets/20d6ed45-7325-4399-9dd0-514468d584ab" />

### VirusTotal Results
![VirusTotal]
<img width="845" height="573" alt="4" src="https://github.com/user-attachments/assets/cbd5c4b9-de81-4134-b916-c6616759823a" />

### TheHive Case Created
![TheHive Case]
<img width="1115" height="568" alt="6" src="https://github.com/user-attachments/assets/58dfb981-8761-4492-9044-b9bc2579b8b1" />

### Email Alert Received
![Email Alert]
<img width="410" height="146" alt="1" src="https://github.com/user-attachments/assets/822db808-dd25-42a9-b507-dae9ad7412a2" />

<img width="848" height="465" alt="2" src="https://github.com/user-attachments/assets/66fe0d0b-052e-49e3-8b60-5b331fecbe61" />

---

## Key Skills Demonstrated

- Linux server administration (Ubuntu 22.04)
- SIEM deployment and custom rule writing
- SOAR workflow automation
- API integration (VirusTotal, TheHive)
- Threat intelligence enrichment
- Incident case management
- Network tunneling with ngrok
- MITRE ATT&CK framework mapping
- Windows endpoint monitoring with Sysmon
- VMware virtualization

---

## References

| Resource | Link |
|----------|------|
| Wazuh Docs | https://docs.wazuh.com |
| TheHive Docs | https://docs.strangebee.com |
| Shuffle Docs | https://shuffler.io/docs |
| MITRE T1003 | https://attack.mitre.org/techniques/T1003 |
| Sysmon Config | https://github.com/SwiftOnSecurity/sysmon-config |

---

## Author
Kenilkumar Prajapati
