# Blue Team Wazuh Lab

A hands-on blue team learning lab where I deployed **Wazuh 4.14.5** (open-source SIEM/XDR) via Docker on a VPS, enrolled two monitored endpoints (Windows 10 + Ubuntu 22.04), and worked through several detection, response, and monitoring scenarios — from Sysmon telemetry to malware detection, FIM, VirusTotal integration, active response, MariaDB auditing, SBOM analysis, and Zeek network monitoring.

This repo documents the full journey: configs, custom rules, dashboard exports, screenshots, and notes.

## Lab Topology

```
+-------------------+        +-----------------------------+
|   VPS (2 vCPU,    |        |        Endpoints            |
|     8 GB RAM)     |        |                             |
|                   |        |  +-----------------------+  |
|  Wazuh Manager    |<-------+--| Windows 10 VM         |  |
|  Wazuh Indexer    |  1514  |  | (Sysmon + Wazuh agent)|  |
|  Wazuh Dashboard  |  1515  |  +-----------------------+  |
|  (Docker compose) |        |                             |
|                   |        |  +-----------------------+  |
|  Zeek (on Ubuntu  |<-------+--| Ubuntu 22.04 VM       |  |
|   VM, bridged)    |        |  | (auditd + Wazuh agent |  |
+-------------------+        |  |  + MariaDB + Zeek)    |  |
                             |  +-----------------------+  |
                             +-----------------------------+
```

## Repository Structure

| Folder                 | Purpose                                                                                                       |
| ---------------------- | ------------------------------------------------------------------------------------------------------------- |
| [`docs/`](docs/)       | Learning notes, references, and command cheatsheets                                                           |
| [`configs/`](configs/) | All configuration files — Wazuh agent/manager, Sysmon, MariaDB audit, and the full `wazuh-docker/` deployment |
| [`rules/`](rules/)     | Custom Wazuh detection rules (cipher.exe + DeerStealer malware)                                               |
| [`exports/`](exports/) | CSV exports from Wazuh Discover + an SBOM JSON sample                                                         |
| [`reports/`](reports/) | PDF reports generated from the Wazuh dashboard (FIM, MITRE, overview)                                         |
| [`mitre/`](mitre/)     | MITRE ATT&CK Navigator layer + screenshot                                                                     |
| [`images/`](images/)   | Screenshots documenting each step of the lab                                                                  |

## Documentation

- **[docs/what-i-learned.md](docs/what-i-learned.md)** — The main narrative of the entire lab journey, from Docker deployment through Zeek integration. Covers every decision, error, and fix along the way.
- **[docs/references.md](docs/references.md)** — External links and documentation used throughout the lab.
- **[docs/move-command.md](docs/move-command.md)** — Quick reference for copying files from the VPS to a local machine using `scp`.

## Scenarios Covered

1. **Docker Deployment** — Wazuh manager + indexer + dashboard via `docker-compose`, including persistent `vm.max_map_count` and cert generation.
2. **Agent Enrollment** — Wazuh agents on Windows 10 and Ubuntu 22.04 VMs.
3. **Sysmon Telemetry** — SwiftOnSecurity config, Event ID 1 vs Event ID 10, and `ossec.conf` integration.
4. **Custom Detection Rules** — `cipher.exe` execution (rules `119998`/`119999`) and DeerStealer malware (rules `111200`–`111206`) with MITRE ATT&CK mapping.
5. **File Integrity Monitoring (FIM)** — Realtime + `check_all` on Ubuntu via `agent.conf`.
6. **VirusTotal Integration** — Automated malware scanning triggered by FIM alerts (EICAR test).
7. **Active Response** — `remove-threat.sh` script with `jq`, `chown`, and `chmod 750`.
8. **MariaDB Auditing** — `50-audit.cnf`, decoders, and rules for authentication/DML/DLL operations.
9. **MITRE ATT&CK Mapping** — Navigator layer color-coded by detection coverage.
10. **SBOM + Dependency-Track** — CycloneDX BOM generation (npm + Python) and Wazuh alerting.
11. **Zeek Network Monitoring** — DNS query detection, Ubuntu 22.04 install tweaks, and interface fix (`eth0` → `enp0s3`).

## Key Configs

| File                                                                                                                                                   | Description                                                  |
| ------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| [`configs/wazuh-docker/single-node/docker-compose.yml`](configs/wazuh-docker/single-node/docker-compose.yml)                                           | Full Wazuh stack deployment (manager + indexer + dashboard)  |
| [`configs/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf`](configs/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf) | Manager config with VirusTotal integration + active response |
| [`configs/agent.conf`](configs/agent.conf)                                                                                                             | Centralized agent config for FIM on Linux                    |
| [`configs/sysmonconfig.xml`](configs/sysmonconfig.xml)                                                                                                 | SwiftOnSecurity Sysmon config (with Event ID 1 added)        |
| [`configs/50-audit.cnf`](configs/50-audit.cnf)                                                                                                         | MariaDB audit plugin configuration                           |

## Custom Rules

| File                                                         | Rule IDs          | Purpose                                                                                                                 |
| ------------------------------------------------------------ | ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| [`rules/cipher_rule_1.xml`](rules/cipher_rule_1.xml)         | `119998`          | Detect `cipher.exe` execution via Sysmon Event ID 1 (process creation)                                                  |
| [`rules/cipher_rule_10.xml`](rules/cipher_rule_10.xml)       | `119999`          | Detect `cipher.exe` execution via Sysmon Event ID 10 (process access)                                                   |
| [`rules/deerstealer_rules.xml`](rules/deerstealer_rules.xml) | `111200`–`111206` | DeerStealer malware detection — persistence, file creation, process creation, C2 network connection, registry tampering |

## Tech Stack

- **SIEM/XDR:** Wazuh 4.14.5 (Docker deployment)
- **Endpoints:** Windows 10 VM, Ubuntu 22.04 Server VM (VirtualBox, bridged adapter)
- **Telemetry:** Sysmon (SwiftOnSecurity), auditd
- **Network Monitoring:** Zeek
- **Database Auditing:** MariaDB + audit plugin
- **SBOM:** CycloneDX (npm + Python), Dependency-Track
- **Threat Intel:** VirusTotal API integration
- **Framework:** MITRE ATT&CK Navigator
