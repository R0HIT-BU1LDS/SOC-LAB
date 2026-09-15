# Attack Simulation — Nmap TCP SYN Scan (T1046)

## Purpose

Demonstrate the detection pipeline end-to-end: attacker → network traffic → sensor → Suricata → Wazuh.

## Environment

- **Attacker:** Kali Linux — `192.168.56.12`
- **Target:** Windows 10 — `192.168.56.13`
- **Sensor observer:** UBUNTU SENSOR — `192.168.56.11` (promiscuous mode on the internal adapter)

## Command

```bash
nmap -sS -T3 -p 1-1000 192.168.56.13
```

- `-sS` — TCP SYN (stealth) scan
- `-T3` — normal timing
- `-p 1-1000` — first 1000 ports

## Raw observation (tcpdump)

On the sensor, the scan traffic is visible in real time:

```bash
sudo tcpdump -ni enp0s3 'src host 192.168.56.12 and dst host 192.168.56.13'
```

## Detection (Suricata)

The custom rule [`local.rules`](../suricata/local.rules) fires per SYN packet. Each alert is written to:

```
/var/log/suricata/eve.json
```

Observed event fields:

| Field            | Value                                                  |
|------------------|--------------------------------------------------------|
| Source IP        | `192.168.56.12`                                        |
| Destination IP   | `192.168.56.13`                                        |
| Protocol         | TCP                                                    |
| Signature ID     | `1000001` (Suricata SID)                               |
| Signature        | `SOC LAB - Kali TCP SYN scan detected`                 |
| Severity         | `3`                                                    |
| Action           | `allowed` (IDS mode — detection only, no blocking)     |

## Events in Wazuh

The full scan produced **~4,004 events** in the Wazuh Dashboard (`rule.groups:suricata`). Representative fields:

```json
{
  "agent.name": "ubuntusensor",
  "rule.description": "Suricata: Alert - SOC LAB - Kali TCP SYN scan detected",
  "rule.level": 3
}
```

## Result

The complete chain was confirmed:

Kali → Nmap SYN scan → Windows 10 → soclab network → UBUNTU SENSOR → Suricata → `eve.json` → Wazuh Agent → Wazuh Manager → Wazuh Indexer → Wazuh Dashboard → SOC alert ✓

## Why "allowed"?

Suricata runs in **IDS mode**, not inline IPS mode, so matched traffic is detected and logged, not dropped. This is the intended configuration for behavior-based detection in this lab.