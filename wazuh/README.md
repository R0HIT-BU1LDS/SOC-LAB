# Wazuh — SIEM Deployment

Wazuh provides the SIEM layer of the lab. The Manager, Indexer and Dashboard run on **UBUNTU SIEM** (`192.168.56.10`); lightweight agents run on the sensor and on Windows 10.

## Components

| Component               | Host           | Version observed |
|-------------------------|----------------|------------------|
| Wazuh Manager           | UBUNTU SIEM    | 4.9.2            |
| Wazuh Indexer           | UBUNTU SIEM    | 4.9.2            |
| Wazuh Dashboard         | UBUNTU SIEM    | 4.9.2            |
| Wazuh Agent (Linux)     | UBUNTU SENSOR  | -                |
| Wazuh Agent (Windows)   | Windows 10     | -                |

## Agents

| Agent name    | Host            | Status |
|---------------|-----------------|--------|
| `ubuntusensor`| UBUNTU SENSOR   | ✅ connected (`status='connected'`) |
| Windows agent | Windows 10      | ✅ active in dashboard |

## Suricata log collection (UBUNTU SENSOR)

The sensor agent ships Suricata events by reading its EVE JSON log. This block is added to the agent's `ossec.conf`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Validation and restart:

```bash
sudo /var/ossec/bin/wazuh-logcollector -t
sudo systemctl restart wazuh-agent
```

The test passed (`-t` returns OK) and the agent came back with `status='connected'`.

## Dashboard detection

Suricata alert events appear in **Discover** via:

```
rule.groups:suricata
```

The Nmap simulation produced **~4,004 Wazuh-indexed events**. A representative event contains:

```json
{
  "agent.name": "ubuntusensor",
  "rule.description": "Suricata: Alert - SOC LAB - Kali TCP SYN scan detected",
  "rule.level": 3
}
```

## Rules

Custom Wazuh correlation rules are **planned** (see repository README, *Detection Engineering*). No custom Wazuh rules are claimed as implemented.