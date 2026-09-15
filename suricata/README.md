# Suricata — UBUNTU SENSOR

Suricata is installed on the **UBUNTU SENSOR** VM (`192.168.56.11`) and operates as an **Intrusion Detection System (IDS)** — packet capture and alerting only, no inline blocking.

## Profile

| Item  | Value                                                              |
|-------|--------------------------------------------------------------------|
| Host  | UBUNTU SENSOR (192.168.56.11)                                      |
| Role  | Network IDS                                                         |
| Version used | 8.0.6                                                        |
| Mode  | IDS (not inline IPS)                                               |
| Config test | `suricata -T` succeeds                                       |

## Files

| Path                                   | Purpose                                   |
|----------------------------------------|-------------------------------------------|
| `/etc/suricata/suricata.yaml`          | Main Suricata configuration               |
| `/var/log/suricata/eve.json`           | EVE JSON output (all detection events)    |
| `/var/lib/suricata/rules/suricata.rules` | Bundled ruleset                         |
| `/var/lib/suricata/rules/local.rules`  | **Custom rules** (see `local.rules`)      |

## Custom rule

See [`local.rules`](local.rules) — a per-SYN-packet detection rule aimed at the Kali → Windows traffic in the lab.

## Output

Suricata writes normalized JSON events to `eve.json`. Each alert event carries:

- `src_ip` / `src_port`
- `dest_ip` / `dest_port`
- `proto`
- `alert.signature_id` (the Suricata SID)
- `alert.signature`
- `alert.severity`
- `alert.action` (e.g. `allowed`)

> **`alert.action: allowed`** is expected: this deployment detects but does not block (IDS mode).

## Wazuh ingestion

The Wazuh Agent on the sensor reads `eve.json` (see [`../wazuh/README.md`](../wazuh/README.md)) and ships each event to the Wazuh Manager.