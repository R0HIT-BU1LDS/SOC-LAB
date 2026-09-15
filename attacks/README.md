# Attacks — Controlled Simulation Notes

All attack activity is **authorized, contained, and restricted to the isolated VirtualBox `soclab` network**. No Internet-facing targets are involved.

| Scenario                                                                                                    | Tool  | Target           | MITRE ATT&CK          | Status |
|-------------------------------------------------------------------------------------------------------------|-------|------------------|-----------------------|--------|
| SYN port scan of the lab Windows 10 endpoint  | Nmap  | 192.168.56.13    | T1046 — Network Service Scanning | ✅ Detected |

## Index

- [`nmap.md`](nmap.md) — TCP SYN scan simulation and observed detection telemetry