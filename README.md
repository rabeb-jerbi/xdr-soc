<div align="center">

<img src="https://img.shields.io/badge/Type-PFE%20Security%20Engineering-blue?style=flat-square" />
<img src="https://img.shields.io/badge/Platform-Google%20Cloud%20Platform-4285F4?style=flat-square&logo=google-cloud&logoColor=white" />
<img src="https://img.shields.io/badge/XDR-Wazuh%20Open%20Source-orange?style=flat-square" />
<img src="https://img.shields.io/badge/SOAR-TheHive%20%2B%20Cortex-informational?style=flat-square" />
<img src="https://img.shields.io/badge/Threat%20Intel-MISP-red?style=flat-square" />
<img src="https://img.shields.io/badge/Status-Academic%20Project-purple?style=flat-square" />

# 🛡️ Mise en place d'une solution XDR open source

**Deploying an Open-Source XDR Solution — SOC Architecture with Active Response**

*Full SOC architecture on Google Cloud Platform combining Wazuh XDR/SIEM, Suricata NIDS, pfSense firewall, MISP threat intelligence, TheHive incident management, and Cortex automation — validated against 4 real attack scenarios.*

[Overview](#-overview) · [Architecture](#-architecture) · [Stack Comparison](#-comparative-study) · [Implementation](#-implementation) · [Attack Tests](#-attack-tests--active-response) · [MITRE ATT&CK](#-mitre-attck-mapping) · [Tools](#-tools-used)

</div>

---

## 🎯 Overview

This project designs and deploys a complete **SOC (Security Operations Center) architecture** for the **Agence Technique des Télécommunications (ATT)** using exclusively **open-source tools**. The solution integrates an **XDR (Extended Detection and Response)** stack capable of detecting, analyzing, and responding to real-world cyber threats automatically.

**Key results:** 4 attack scenarios were simulated and successfully detected and blocked using Wazuh active responses:
1. Malicious actor / brute force (IP 220.78.28.115 blocked) — T1110.001
2. SQL Injection on Windows & Ubuntu — T1190
3. Shellshock (Bash CVE-2014-6271) — remote code execution blocked
4. Nmap port scan — detected by Suricata + Wazuh, attacker IP blocked

---

## 🎓 Academic Context

> **Projet de Fin d'Études (PFE)** — ISET'COM, Licence Appliquée en Sciences et Technologies de l'Information et de la Communication, Spécialité Réseaux et Systèmes, 2023–2024.

--

## 🏗️ Architecture

### System Overview

The XDR architecture is deployed entirely on **Google Cloud Platform (GCP)** with a VPC network (`192.168.0.0/16`) hosting all components:

```
            ┌─────────────────────────────────────────────────┐
            │          Google Cloud Platform VPC               │
            │              192.168.0.0/16                      │
            │                                                  │
  ┌──────┐  │  ┌──────────┐  ┌───────────────────────────┐   │
  │ Kali │  │  │ pfSense  │  │       Wazuh Cluster        │   │
  │ Agent│◄►│  │ Firewall │  │  Manager  Worker1  Worker2 │   │
  └──────┘  │  │192.168.  │  │  .0.2     .0.3     .0.31  │   │
  ┌──────┐  │  │  0.38    │  └──────────────────────────── ┘  │
  │Ubuntu│  │  └──────────┘         ▲                          │
  │Agent1│  │  ┌──────────┐         │ (Nginx Load Balancer     │
  │.0.4  │  │  │ Suricata │         │  192.168.0.5)            │
  └──────┘  │  │  NIDS    │    ┌────┴────────────────┐         │
  ┌──────┐  │  │  .0.9    │    │  MISP + TheHive +   │         │
  │Win   │  │  └──────────┘    │  Cortex             │         │
  │Agent2│  │                  │  192.168.0.26 / .36 │         │
  │.0.56 │  │                  └─────────────────────┘         │
  └──────┘  └─────────────────────────────────────────────────┘
```

### Component IP Table

| Component | IP Address | Specs | OS |
|---|---|---|---|
| Wazuh Manager (Master) | 192.168.0.2 | 2 vCPUs, 4 GB RAM, 30 GB | Ubuntu 20.04 LTS |
| Wazuh Worker 1 | 192.168.0.3 | 2 vCPUs, 4 GB RAM, 30 GB | Ubuntu 20.04 LTS |
| Wazuh Worker 2 | 192.168.0.31 | 2 vCPUs, 4 GB RAM, 30 GB | Ubuntu 20.04 LTS |
| Nginx Load Balancer | 192.168.0.5 | 2 vCPUs, 4 GB RAM, 30 GB | Ubuntu 20.04 LTS |
| Suricata NIDS | 192.168.0.9 | 2 vCPUs, 4 GB RAM, 30 GB | Ubuntu 20.04 LTS |
| pfSense Firewall | 192.168.0.38 | 4 vCPUs, 2 GB RAM, 10 GB | FreeBSD (pfSense) |
| TheHive + MISP | 192.168.0.26 | 4 vCPUs, 16 GB RAM, 35 GB | Ubuntu 20.04 LTS |
| Cortex | 192.168.0.36 | 4 vCPUs, 16 GB RAM, 20 GB | Ubuntu 20.04 LTS |
| Agent 1 (Ubuntu) | 192.168.0.4 | 2 vCPUs, 4 GB RAM, 15 GB | Ubuntu 20.04 LTS |
| Agent 2 (Windows) | 192.168.0.56 | 2 vCPUs, 8 GB RAM, 50 GB | Windows Server 2019 |

### Data Flow

```
Endpoints (Ubuntu + Windows)
    └─► Wazuh Agents ──► Nginx Load Balancer ──► Wazuh Cluster (Manager + Workers)
                                                         │
Suricata NIDS ──────────────────────────────────────────┘
pfSense Firewall ───────────────────────────────────────┘
                                                         │
                                        ┌────────────────▼──────────────┐
                                        │        Alert Processing        │
                                        │                                │
                                        │  Wazuh Alert ──► TheHive Case │
                                        │                       │        │
                                        │              Cortex Analyze   │
                                        │              VirusTotal + IoC  │
                                        │                       │        │
                                        │              MISP TI Feed     │
                                        │                                │
                                        │         Active Response        │
                                        │    firewall-drop (block IP)   │
                                        └───────────────────────────────┘
```

---

## 🔬 Comparative Study

### XDR Platform

| Solution | Type | Chosen? | Reason |
|---|---|---|---|
| **Wazuh** | Open source | ✅ **YES** | Flexible, free, integrates SIEM + XDR + EDR, strong community |
| Microsoft Defender XDR | Commercial | ❌ | High cost, vendor lock-in |
| Palo Alto Cortex XDR | Commercial | ❌ | Very high cost |

### SIEM

| Solution | Type | Chosen? | Reason |
|---|---|---|---|
| **Wazuh** | Open source | ✅ **YES** | Integrated with XDR, no additional cost |
| Splunk | Commercial | ❌ | Powerful but expensive |
| IBM QRadar | Commercial | ❌ | High cost, complex |

### NIDS

| Solution | Type | Chosen? | Reason |
|---|---|---|---|
| **Suricata** | Open source | ✅ **YES** | High performance, multithreading, handles large traffic volumes |
| Zeek | Open source | ❌ | Deep protocol analysis but less real-time detection |
| Snort | Open source | ❌ | Signature-based, less capable at scale |

### SOAR

| Solution | Type | Chosen? | Reason |
|---|---|---|---|
| **Cortex** | Open source | ✅ **YES** | Free, integrates tightly with TheHive, artifact automation |
| Cortex XSOAR | Commercial | ❌ | Expensive, complex |
| Splunk Phantom | Commercial | ❌ | High cost |

### Threat Intelligence Platform

| Solution | Type | Chosen? | Reason |
|---|---|---|---|
| **MISP** | Open source | ✅ **YES** | Collaborative sharing, integrates with Wazuh + TheHive |
| Recorded Future | Commercial | ❌ | High cost |
| ThreatConnect | Commercial | ❌ | High cost |

### Firewall

| Solution | Type | Chosen? | Reason |
|---|---|---|---|
| **pfSense** | Open source | ✅ **YES** | FreeBSD-based, rich features, free, Wazuh agent compatible |
| Cisco ASA | Commercial | ❌ | High cost |
| Fortinet FortiGate | Commercial | ❌ | High cost |

### Load Balancer

| Solution | Type | Chosen? | Reason |
|---|---|---|---|
| **Nginx** | Open source | ✅ **YES** | Dual-role (web server + load balancer), low resource usage |
| HAProxy | Open source | ❌ | Reliable but single-purpose |
| F5 BIG-IP | Commercial | ❌ | High cost, overkill |

---

## ⚙️ Implementation

### 1. Wazuh Cluster Configuration

A 3-node Wazuh cluster (1 Manager + 2 Workers) was deployed for high availability:

**Cluster config (`/var/ossec/etc/ossec.conf`):**
```xml
<cluster>
  <name>wazuh</name>
  <node_name>master-node</node_name>
  <node_type>master</node_type>
  <key>[shared-secret-key]</key>
  <port>1516</port>
  <bind_addr>0.0.0.0</bind_addr>
</cluster>
```

Cluster verified with: `sudo systemctl status wazuh-manager` → `active (running)`

### 2. Nginx Load Balancer

Nginx configured to distribute agent traffic across Wazuh nodes (ports 1514 and 1515):

```nginx
stream {
  upstream wazuh_cluster {
    server 192.168.0.2:1514;
    server 192.168.0.3:1514;
    server 192.168.0.31:1514;
  }
  server {
    listen 1514;
    proxy_pass wazuh_cluster;
  }
}
```

### 3. MISP — Threat Intelligence

- Organization: `pfe-soc`
- Threat feeds activated: CIRCL OSINT Feed, Botvrij.eu, and multiple community feeds
- API integrated with Wazuh for IoC enrichment of alerts in TheHive

### 4. TheHive — Incident Management

- Organization: `pfe-soc`
- Integrated with MISP (threat intelligence) and Cortex (response automation)
- Wazuh alerts forwarded to TheHive as cases automatically

### 5. Cortex — Analysis Automation

- Organization: `XDR`
- VirusTotal integration enabled (IP and hash analysis)
- Analyzers configured for automatic enrichment of TheHive observables

### 6. Suricata NIDS

Suricata configured on dedicated Ubuntu VM, integrated with Wazuh:

```yaml
# /etc/suricata/suricata.yaml
vars:
  address-groups:
    HOME_NET: "[192.168.0.0/16]"
    EXTERNAL_NET: "any"
af-packet:
  - interface: ens4
```

Wazuh agent on Suricata host reads `/var/log/suricata/eve.json` and ships alerts to manager.

### 7. pfSense Integration

Wazuh agent installed on pfSense to ship firewall logs to Wazuh Manager for centralized analysis.

---

## ⚔️ Attack Tests & Active Response

### Test 1: Malicious Actor / Brute Force

**Technique:** MITRE T1110.001 (Brute Force: Password Guessing) + T1021.004 (Remote Services: SSH)

**Attack simulation:**
```bash
curl http://192.168.20.3   # from Kali with unknown/malicious IP
```

**Detection:**
- Wazuh alert raised: suspicious connection from IP `220.78.28.115`
- Alert forwarded to TheHive as a new case
- Cortex ran VirusTotal analyzer → IP confirmed malicious, flagged for phishing + SSH brute force

**Active Response configured in `ossec.conf`:**
```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5710,5763</rules_id>
  <timeout>300</timeout>
</active-response>
```

**Result:** Attacker IP automatically blocked by firewall-drop rule. Re-test confirmed block.

---

### Test 2: SQL Injection

**Technique:** MITRE T1190 (Exploit Public-Facing Application) + T1190.001 (SQL Injection)

**Attack simulation:** Malicious SQL query sent to web application on both Ubuntu and Windows agents.

**Detection:**
- Wazuh detected injection attempt on both platforms
- Alerts appeared in TheHive with severity level

**Active Response:**
```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>[sql-injection rule IDs]</rules_id>
  <timeout>60</timeout>
</active-response>
```

**Result:** Attacker IP blocked after first injection attempt. Confirmed with re-test — connection refused.

---

### Test 3: Shellshock (CVE-2014-6271)

**Vulnerability:** Critical Bash vulnerability allowing remote arbitrary command execution via environment variables.

**Wazuh rule triggered:** Rule 31168 — Shellshock CGI attack

**Active Response:**
```xml
<active-response>
  <command>firewall-drop-Shellshock-attack</command>
  <location>local</location>
  <rules_id>31168</rules_id>
  <timeout>180</timeout>
</active-response>
```

**Result:** Attacker IP blocked for 180 seconds after Shellshock attempt detected. Alert escalated to TheHive.

---

### Test 4: Nmap Port Scan

**Technique:** MITRE T1046 (Network Service Scanning)

**Attack simulation (from Kali Linux):**
```bash
nmap -Pn -p- -sV 192.168.0.4
```

**Detection:**
- **Suricata** detected SYN scan patterns and generated alert in `eve.json`
- **Wazuh** ingested Suricata logs and raised alert with attacker IP and targeted services
- Alert details: attacker IP, services scanned, vulnerability types detected

**Active Response:**
```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>[nmap detection rule]</rules_id>
  <timeout>300</timeout>
</active-response>
```

**Result:** Attacker IP blocked after scan detected — no further information gathering possible.

---

## 🗺️ MITRE ATT&CK Mapping

| Technique ID | Name | Attack Simulated | Result |
|---|---|---|---|
| T1110.001 | Brute Force: Password Guessing | SSH brute force from malicious IP | ✅ Detected + IP blocked |
| T1021.004 | Remote Services: SSH | Remote SSH connection attempt | ✅ Blocked |
| T1190 | Exploit Public-Facing Application | SQL injection on web app | ✅ Detected on Ubuntu + Windows |
| T1190.001 | SQL Injection | Malicious SQL query execution | ✅ Detected + IP blocked |
| T1059.004 | Unix Shell (Shellshock) | CVE-2014-6271 Bash exploit | ✅ Detected + blocked (180s) |
| T1046 | Network Service Scanning | Nmap `-p-` port scan | ✅ Detected by Suricata + blocked |

---

## 🛠️ Tools Used

| Tool | Role |
|---|---|
| **Wazuh** | XDR + SIEM — log aggregation, alert generation, active response |
| **Suricata** | NIDS — real-time network traffic analysis |
| **pfSense** | Firewall — network traffic control, log forwarding to Wazuh |
| **Nginx** | Load balancer — distributes agent traffic across Wazuh cluster |
| **MISP** | Threat Intelligence Platform — IoC feeds, threat sharing |
| **TheHive** | Incident management — case creation, alert triage |
| **Cortex** | SOAR automation — observable analysis, response orchestration |
| **VirusTotal** | IP/hash enrichment integrated via Cortex |
| **Google Cloud Platform** | Infrastructure — VPC, VM instances |
| **Kali Linux** | Attacker machine for attack simulations |
| **Windows Server 2019** | Monitored endpoint (Agent 2) |
| **Ubuntu 20.04 LTS** | Base OS for all backend components |

---

## 📁 Repository Contents

```
xdr-soc-pfe/
├── README.md                  # This file — full project documentation
└── report/
    └── rapport_pfe.pdf        # Full technical report
```

---

## 🔮 Perspectives & Future Work

- **AI/ML integration**: Anomaly detection using behavioral models for zero-day threat identification
- **SOAR expansion**: More automated playbooks in TheHive/Cortex for faster incident response
- **Threat hunting**: Proactive threat hunting using MISP IoC feeds and Wazuh query capabilities
- **Scalability**: Horizontal scaling of the Wazuh cluster for larger enterprise environments

---

## ⚠️ Disclaimer

This project was conducted for academic purposes at ISET'COM (2023–2024). All attack simulations were performed in an isolated, controlled environment (Google Cloud Platform VPC) for educational purposes only. No production systems or external networks were targeted.

---

<div align="center">

**🛡️ Open-Source XDR SOC** — Wazuh · Suricata · MISP · TheHive · Cortex · pfSense on GCP

*ISET'COM — Licence SR, Projet de Fin d'Études, 2023–2024*

</div>
