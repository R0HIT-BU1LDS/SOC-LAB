# 🛡️ ADVANCED SOC LAB — Wazuh + Suricata + Snort

A hands-on Security Operations Center (SOC) laboratory built with VirtualBox. This project demonstrates the complete blue-team workflow — **attack → network sensor → detection → telemetry → SIEM → alert → SOC investigation** — using real, tested components:

- **Wazuh** (Manager + Indexer + Dashboard) as the SIEM
- **Suricata** as the network intrusion detection engine (IDS mode)
- **Snort** (installed on the sensor; integration and testing are planned, not claimed)
- **Kali Linux** as the controlled attacker
- **Windows 10** as the monitored victim endpoint

This is a *practical SOC engineering project*: it documents an implemented, detected and verified attack scenario end-to-end, including the detection engineering trade-offs that come with real rules.

## Project Status

| Component | Status |
|---|---|
| VirtualBox Lab | 🟢 Complete |
| Network Configuration | 🟢 Complete |
| Wazuh SIEM | 🟢 Operational |
| Wazuh Agent — UBUNTU SENSOR | 🟢 Operational |
| Wazuh Agent — Windows 10 | 🟢 Operational |
| Suricata IDS | 🟢 Operational |
| Snort IDS | 🟢 Installed |
| Network Visibility | 🟢 Verified |
| Nmap Attack Simulation | 🟢 Verified |
| Custom Suricata Rule | 🟢 Working |
| Suricata EVE JSON Logging | 🟢 Working |
| Suricata → Wazuh Integration | 🟢 Working |
| Wazuh Correlation Rules | ⚪ Not Implemented |
| MITRE ATT&CK Mapping | ⚪ Not Implemented |
| Incident Response | ⚪ Not Implemented |
| Active Response | ⚪ Not Implemented |
| Sysmon | ⚪ Not Implemented |
| TheHive | ⚪ Not Implemented |

Items marked ⚪ are **discussed as future improvements** — they are not claimed as implemented anywhere in this document.

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Architecture](#architecture)
- [Network Design](#network-design)
- [Setup](#setup)
- [Lab Components](#lab-components)
- [Data Flow](#data-flow)
- [Network Visibility](#network-visibility)
- [Attack Simulation](#attack-simulation)
- [Custom Suricata Detection Rule](#custom-suricata-detection-rule)
- [Suricata Detection](#suricata-detection)
- [Wazuh + Suricata Integration](#wazuh--suricata-integration)
- [Wazuh Dashboard Detection](#wazuh-dashboard-detection)
- [SOC Investigation](#soc-investigation)
- [Detection Engineering](#detection-engineering)
- [MITRE ATT&CK](#mitre-attack)
- [Troubleshooting](#troubleshooting)
- [Screenshots](#screenshots)
- [Future Improvements](#future-improvements)
- [Project Report](#project-report)
- [Security Notice](#security-notice)
- [Author](#author)

## Overview

Four VirtualBox VMs form an isolated, internal-only simulation of a small enterprise network:

- **UBUNTU SIEM** — hosts the full Wazuh stack
- **UBUNTU SENSOR** — passive network monitoring with Suricata (and Snort)
- **Kali Linux** — attacker, running controlled scans
- **Windows 10** — victim endpoint under observation

The sensor's internal adapter runs in **promiscuous mode**, so it observes packet traffic between Kali and Windows even though the traffic is not addressed to it. Suricata converts that raw network traffic into structured JSON detection events (`eve.json`), which the Wazuh agent on the sensor forwards to the manager and indexer, surfacing them in the Wazuh Dashboard as analyst-reviewable alerts.

## Objectives

- Stand up a complete SOC detection pipeline on low-footprint lab hardware using only VirtualBox.
- Implement a custom Suricata detection rule and validate it against a controlled Nmap SYN scan.
- Prove end-to-end telemetry: raw packet → Suricata event → Wazuh event → dashboard alert.
- Practice SOC investigation workflows: triage fields, correlation, and incident response planning.
- Learn detection engineering trade-offs (alert volume, thresholding, correlation).
- Map observed activity to the MITRE ATT&CK framework.

## Architecture

```
                         ┌─────────────────────────────┐
                         │         UBUNTU SIEM         │
                         │        192.168.56.10        │
                         │                             │
                         │ Wazuh Manager              │
                         │ Wazuh Indexer              │
                         │ Wazuh Dashboard            │
                         └──────────────┬──────────────┘
                                        │
                                        │ Wazuh telemetry
                                        │
                         ┌──────────────▼──────────────┐
                         │       UBUNTU SENSOR         │
                         │        192.168.56.11        │
                         │                             │
                         │ Suricata                   │
                         │ Snort                      │
                         │ Wazuh Agent                │
                         └──────────────┬──────────────┘
                                        │
                              VirtualBox "soclab"
                                        │
                         ┌──────────────┴──────────────┐
                         │                             │
               ┌─────────▼──────────┐       ┌──────────▼─────────┐
               │     Kali Linux      │       │     Windows 10     │
               │   192.168.56.12    │       │   192.168.56.13    │
               │                    │       │                    │
               │      ATTACKER      │──────►│       VICTIM       │
               └────────────────────┘       └────────────────────┘
```

The red arrow represents the attack flow (Kali → Windows). The sensor sits on the same internal broadcast segment and observes that flow via promiscuous capture.

## Network Design

| VM              | IP            | OS               | Role / Components                          | Networking                  |
|-----------------|---------------|------------------|---------------------------------------------|-----------------------------|
| UBUNTU SIEM     | 192.168.56.10 | Ubuntu 24.04 LTS | Wazuh Manager, Indexer, Dashboard          | soclab internal + NAT       |
| UBUNTU SENSOR   | 192.168.56.11 | Ubuntu 24.04 LTS | Suricata, Snort, Wazuh Agent               | soclab internal + NAT (**promiscuous**) |
| Kali Linux      | 192.168.56.12 | Kali             | Attacker                                    | soclab internal + NAT       |
| Windows 10      | 192.168.56.13 | Windows 10       | Victim endpoint, Wazuh Agent               | soclab internal + NAT       |

Every VM uses two VirtualBox adapters:

- **Adapter 1 — VirtualBox Internal Network `soclab`**: isolated lab traffic, private `192.168.56.0/24` range.
- **Adapter 2 — NAT**: internet access for OS and package updates only.

> The UBUNTU SENSOR's internal adapter is set to **Promiscuous Mode: Allow All**, which is what lets it observe Kali → Windows traffic.

## Setup

Reproducibility details for the lab:

- **Hypervisor:** Oracle VirtualBox
- **Guest OS:** Ubuntu 24.04 LTS (SIEM and SENSOR), Kali Linux, Windows 10
- **Networking:** Adapter 1 = VirtualBox **Internal Network** named exactly `soclab`; Adapter 2 = **NAT** (internet/package updates only)
- **Sensor capture:** UBUNTU SENSOR Adapter 1 set to **Promiscuous Mode: Allow All**
- **Wazuh stack:** Wazuh 4.9.2 (Manager, Indexer, Dashboard) on UBUNTU SIEM
- **Suricata:** version 8.0.6 in IDS mode on UBUNTU SENSOR
- **Snort:** installed on UBUNTU SENSOR (integration/testing pending)
- **Wazuh Agents:** UBUNTU SENSOR (192.168.56.11) and Windows 10 (192.168.56.13)
- **Suricata → Wazuh:** sensor agent reads `/var/log/suricata/eve.json` via log collector

Static IP addressing on the internal `soclab` network (`192.168.56.0/24`):

| Host           | Static IP          |
|----------------|--------------------|
| UBUNTU SIEM    | `192.168.56.10/24` |
| UBUNTU SENSOR  | `192.168.56.11/24` |
| Kali Linux     | `192.168.56.12/24` |
| Windows 10     | `192.168.56.13/24` |

No gateway or DNS is defined for the internal SOC network; internet access is handled solely by the NAT adapter.

## Lab Components

### Wazuh

Open-source SIEM/XDR platform. The **Manager** centralizes events and applies Wazuh's built-in decoders/rules; the **Indexer** provides the datastore/search engine; the **Dashboard** gives the analyst UI. Observed Wazuh Manager version: **4.9.2**. Agents are installed on the sensor (Linux) and Windows 10 — no additional agents are claimed.

### Suricata

Network IDS that reads packets off the wire and matches them against signature rules, emitting structured events. Deployed in **IDS mode** on the sensor with a custom local rule (see below). Version used: **8.0.6**.

### Snort

Classic signature-based IDS/IPS. **Installed on the sensor but not yet fully integrated/tested** — see [Project Status](#project-status). No integration is claimed until it is verified.

### Kali Linux

Attacker VM used exclusively for **controlled, authorized simulations** within the isolated lab network.

### Windows 10

Monitored endpoint (victim). Runs the Wazuh agent so endpoint-side events are visible in the SIEM.

## Data Flow

End-to-end detection pipeline:

```
Attack
  ↓
Network Traffic
  ↓
Security Sensor
  ↓
Detection
  ↓
Telemetry / Logs
  ↓
Wazuh Agent
  ↓
Wazuh Manager
  ↓
Wazuh Indexer
  ↓
Wazuh Dashboard
  ↓
Alert
  ↓
SOC Investigation
  ↓
Correlation          (planned / future work — not implemented)
  ↓
Incident Response    (planned / future work — not implemented)
```

Concretely, for the Nmap scenario (see [attacks/nmap.md](attacks/nmap.md)):

```
Kali
 ↓
Nmap SYN scan
 ↓
Windows 10
 ↓
soclab network
 ↓
UBUNTU SENSOR
 ↓
Suricata
 ↓
eve.json
 ↓
Wazuh Agent
 ↓
Wazuh Manager
 ↓
Wazuh Indexer
 ↓
Wazuh Dashboard
 ↓
SOC Alert
```

## Network Visibility

A packet the sensor was never addressed to can still be seen because the sensor's VirtualBox internal adapter runs in **promiscuous mode ("Allow All")**. The kernel passes the received frames to userspace regardless of destination MAC, so Suricata (AF_PACKET) sees the full traffic on the segment — including Kali ↔ Windows.

Raw capture is verified with `tcpdump` on the sensor:

```bash
sudo tcpdump -ni enp0s3 'src host 192.168.56.12 and dst host 192.168.56.13'
```

This confirmed the scan packets were observable before any detection rule processed them — separating **network visibility** (packet capture) from **detection** (signature matching).

## Attack Simulation

A controlled Nmap SYN scan was run from Kali against the lab Windows machine only:

```bash
nmap -sS -T3 192.168.56.13
```

with the main demonstrated scan:

```bash
nmap -sS -T3 -p 1-1000 192.168.56.13
```

- `-sS` — TCP SYN scan
- `-T3` — normal timing
- `-p 1-1000` — scan ports 1 through 1000

The scans targeted no system outside `192.168.56.13`. Results are documented in [attacks/nmap.md](attacks/nmap.md).

## Custom Suricata Detection Rule

Custom rule file: `/var/lib/suricata/rules/local.rules` — see [`suricata/local.rules`](suricata/local.rules).

```suricata
alert tcp 192.168.56.12 any -> 192.168.56.13 any (msg:"SOC LAB - Kali TCP SYN scan detected"; flags:S; sid:1000001; rev:1;)
```

### Rule breakdown

| Rule part            | Meaning                                                        |
|----------------------|----------------------------------------------------------------|
| `alert`              | Action: log/notify when the rule matches (IDS mode)            |
| `tcp`                | Protocol being inspected (TCP)                                 |
| `192.168.56.12`      | Source IP — Kali, the attacker                                 |
| `any`                | Source port — match any source port                            |
| `->`                 | Directionality: source → destination                           |
| `192.168.56.13`      | Destination IP — Windows 10, the victim                        |
| `any`                | Destination port — match any destination port                  |
| `msg`                | Human-readable alert text: "SOC LAB - Kali TCP SYN scan detected" |
| `flags:S`            | Match packets with only the SYN flag set (a scan probe)        |
| `sid:1000001`        | **Suricata Signature ID (SID)** — unique rule identifier        |
| `rev:1`              | Rule revision (1)                                              |

> **`1000001` is the Suricata SID.** It identifies the signature in Suricata. Do not confuse it with the Wazuh **rule ID**, which is assigned by the Wazuh engine when the event is correlated (`rule.level`/`rule.id` are Wazuh concepts; `signature_id`/`severity` are Suricata concepts).

> SIDs `1000000+` are reserved for local/custom rules, so they do not collide with the emerging-threats/bundled rulesets.

## Suricata Detection

Suricata emits normalized JSON to the **EVE JSON** log:

```
/var/log/suricata/eve.json
```

Active rules directories:

| Path                                             | Purpose            |
|--------------------------------------------------|--------------------|
| `/var/lib/suricata/rules/suricata.rules`         | Bundled ruleset    |
| `/var/lib/suricata/rules/local.rules`            | Custom rules       |

Snapshot of an alert event for the custom rule:

| Field          | Value                                             |
|----------------|---------------------------------------------------|
| Source IP      | `192.168.56.12`                                   |
| Destination IP | `192.168.56.13`                                   |
| Protocol       | TCP                                               |
| Signature ID   | `1000001`                                         |
| Signature      | `SOC LAB - Kali TCP SYN scan detected`            |
| Severity       | `3`                                               |
| Action         | `allowed`                                         |

**`allowed` is expected behavior**: the existing deployment runs Suricata as an **IDS**, not an inline IPS, so matched packets are detected and logged rather than dropped. This is a deliberate configuration choice for a detection-focused lab.

## Wazuh + Suricata Integration

The Wazuh Agent on the UBUNTU SENSOR reads the Suricata EVE JSON file with its log collector. The relevant `ossec.conf` block:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Validation and restart:

```bash
sudo /var/ossec/bin/wazuh-logcollector -t   # config test: passed
sudo systemctl restart wazuh-agent
```

After restart the agent was verified as `status='connected'`, and Suricata events from the Nmap scan reached the Wazuh Dashboard.

## Wazuh Dashboard Detection

Searching in Wazuh **Discover**:

```
rule.groups:suricata
```

returned the Suricata events — approximately **4,004 events** from the single scan. A representative Wazuh event contains:

| Field                | Value                                                   |
|----------------------|---------------------------------------------------------|
| `agent.name`         | `ubuntusensor`                                          |
| `rule.description`   | `Suricata: Alert - SOC LAB - Kali TCP SYN scan detected`|
| `rule.level`         | `3`                                                     |

Full end-to-end chain confirmed:

```
Kali → Nmap SYN scan → Windows 10 → soclab network → UBUNTU SENSOR → Suricata → eve.json → Wazuh Agent → Wazuh Manager → Wazuh Indexer → Wazuh Dashboard → SOC Alert ✓
```

## SOC Investigation

A SOC analyst receiving this alert works through the classic **5W + How** questions.

### WHO?

- Attacker source IP: `192.168.56.12` (Kali) — internal actor.
- Reporting agent: `agent.name: ubuntusensor`, `agent.id` identifies the specific sensor node.

### WHAT?

- A TCP SYN probe targeting `192.168.56.13`.
- Rule: Suricata SID `1000001`, message `SOC LAB - Kali TCP SYN scan detected`.
- Wazuh `rule.description` and `rule.level` describe how the SIEM classified the event.

### WHEN?

- **`timestamp`** of the event — both the Suricata `timestamp` and the Wazuh ingestion timestamp should be compared to reconstruct scan timing and duration.

### WHERE?

- Source (`srcip`, `srcport`) → destination (`dstip`, `dstport`).
- Where in the architecture did detection occur: the sensor on `192.168.56.11`, observing internal traffic.
- `rule.groups` shows which rule family fired (`suricata`).

### HOW?

- Protocol: **`protocol: tcp`** with **`flags: S`** — a rapid series of SYN probes across ports is the fingerprint of scanning, not normal application traffic.
- Correlate `signature`, `signature_id` (Suricata) and `severity` with the Wazuh enrichment (`rule.id`, `rule.level`, `rule.groups`, `rule.description`).

### WHY?

- Reconnaissance: mapping open ports is the precursor to exploitation. In this lab it is controlled and authorized; in a real environment it would trigger escalation to [Incident Response](#project-report).

### Key alert fields

`timestamp`, `agent.name`, `agent.id`, `rule.id`, `rule.level`, `rule.groups`, `rule.description`, `srcip`, `srcport`, `dstip`, `dstport`, `protocol`, `signature`, `signature_id`, `severity`.

### Event types — do not confuse

| Term                | Meaning                                                                  |
|---------------------|--------------------------------------------------------------------------|
| Raw network event   | A packet observed on the wire (via `tcpdump`, never leaves the sensor)   |
| Suricata alert      | Signature match event emitted by Suricata in `eve.json` (SID, severity)  |
| Wazuh event         | The JSON document the Wazuh agent collected from `eve.json` and forwarded|
| Wazuh rule          | Wazuh's own correlation logic/decoder applied to the event (`rule.*`)    |
| SIEM alert          | The final analyst-facing alert surfaced in the Wazuh Dashboard           |

## Detection Engineering

The custom rule detects **individual SYN packets**. A single Nmap scan generates thousands of packets, so:

```
1 scan
 ↓
thousands of SYN packets
 ↓
thousands of alerts
 ↓
alert fatigue
```

This is the case documented here: **one scan ≈ 4,004 Wazuh-visible events.** It is intentionally useful as a learning exercise, but it is **not** ideal production detection.

Desirable improvements (all planned):

- Thresholding
- Correlation
- Suppression
- Time-based detection
- Wazuh custom rules
- Alert aggregation
- False-positive reduction
- Detection severity tuning

The eventual goal is a correlation outcome that compresses raw signals into one meaningful incident:

```
Thousands of SYN events
        ↓
Correlation
        ↓
"Possible Network Service Scanning"
```

## MITRE ATT&CK

> **Status: ⚪ Not Implemented as integration.** No automatic ATT&CK tagging or MITRE-enabled enrichment is deployed. The mapping below is an **analytical mapping** of the observed Nmap behavior to the framework for portfolio/research reference only.

| Technique | ID   | Observed behavior            |
|-----------|------|------------------------------|
| Network Service Scanning | [T1046](https://attack.mitre.org/techniques/T1046/) | Nmap `-sS` SYN scan of `192.168.56.13` from `192.168.56.12`, detected by the custom Suricata rule |

Only T1046 is mapped, and only analytically. Expanding the mapping is **planned** future work.

## Troubleshooting

| Problem | Diagnosis | Solution |
|---------|-----------|----------|
| Sensor sees traffic in `tcpdump` but not in Suricata logs | Rule not loaded / config not applied | Include `local.rules` in `suricata.yaml`, reload ruleset, re-run `suricata -T` |
| No Wazuh events for Suricata alerts | Log collector not watching EVE JSON | Add the `<localfile>` JSON block for `/var/log/suricata/eve.json` |
| Log collector config validation | Mislabelled `ossec.conf` | `sudo /var/ossec/bin/wazuh-logcollector -t` must return OK before restart |
| Agent shows disconnected | Agent service / key mismatch | Restart `wazuh-agent`, verify `status='connected'` |
| Suricata config test fails | Syntax error in `suricata.yaml` or rules | Fix entry, re-test with `suricata -T` |
| Massive alert volume | Per-packet rule, no correlation | See [Detection Engineering](#detection-engineering) |

## Screenshots

Captures from the running lab:

```
screenshots/
├── wazuh-dashboard.png
├── nmap-scan.png
├── virtualbox-vms.png
├── suricata-alert.png
└── wazuh-suricata-alert.png
```

### 1. Wazuh Dashboard

![Wazuh Dashboard](screenshots/wazuh-dashboard.png)

The Wazuh Dashboard overview showing the actively connected agents (`ubuntusensor`, Windows 10) and general SIEM monitoring state.

### 2. Nmap Scan

![Nmap Scan](screenshots/nmap-scan.png)

Kali terminal running the controlled SYN scan `nmap -sS -T3 -p 1-1000 192.168.56.13` against the lab Windows endpoint.

### 3. Lab VMs

![VirtualBox VMs](screenshots/virtualbox-vms.png)

All four VMs running together in VirtualBox: UBUNTU SIEM, UBUNTU SENSOR, Kali Linux, and Windows 10.

### 4. Suricata Alert

![Suricata Alert](screenshots/suricata-alert.png)

Suricata detection of the scan in `eve.json`: SID `1000001`, severity `3`, action `allowed`, Kali → Windows addressing.

### 5. Wazuh – Suricata Alert

![Wazuh Suricata Alert](screenshots/wazuh-suricata-alert.png)

Wazuh Discover showing the `rule.groups:suricata` events — the Suricata TCP SYN scan detection surfaced in the Wazuh Dashboard (`agent.name: ubuntusensor`).

> Note: a Snort screenshot is intentionally not included — no Snort alert pipeline is in place yet.

## Future Improvements

- [ ] Sysmon on Windows endpoint for deep host telemetry
- [ ] Wazuh Active Response (automated response actions)
- [ ] TheHive integration for case management
- [ ] DVWA as an additional controlled attack target
- [ ] Additional sensors (Zeek, additional Suricata nodes)
- [ ] Advanced attack scenarios (lateral movement, C2 simulation)
- [ ] Ansible/Vagrant automation of the lab
- [ ] Cloud SOC architecture
- [ ] Custom Wazuh correlation rules (thresholds/suppression)
- [ ] MITRE ATT&CK expansion beyond T1046

## Project Report

Planned report structure for the deliverables of this project:

1. Executive Summary
2. Lab Architecture and Network Design
3. Detection Engineering (rules, alerts, tuning)
4. Attack Scenario Walkthrough (Nmap SYN scan)
5. SOC Investigation Narrative (triage of the observed alert)
6. Incident Response Runbook (containment → eradication → recovery)
7. Lessons Learned and Future Work

## Security Notice

All attack simulations in this project are **restricted to the isolated, authorized VirtualBox `soclab` lab network**. No external or production systems were targeted. The documented IPs (`192.168.56.10`–`.13`) belong exclusively to this private lab environment. Do not reproduce these techniques outside of an environment you are authorized to test.

## Author

**ZONNYXXD**

Hands-on blue-team / SOC engineering project.
