# Enterprise Cyber Defense Operations & Automated Threat Monitoring Architecture 

## 📌 Executive Summary & Project Overview

The **Enterprise Cyber Defense Operations & Automated Threat Monitoring Architecture (ECDOMA-Lab)** is an end-to-end, enterprise-grade Security Operations Center (SOC) and SIEM/XDR environment designed and deployed using VMware Workstation. The core objective of this project is to simulate realistic threat actor TTPs (Tactics, Techniques, and Procedures), ingest multi-layered telemetry across network and host boundaries into a centralized SIEM platform (**Wazuh**), and execute tier-1/tier-2 SOC analyst workflows including threat hunting, event correlation, and incident triage.

Through this architecture, live host-based telemetry (Windows Security Auditing & Microsoft Sysmon), perimeter network firewall logs (**pfSense**), Network Intrusion Detection System alerts (**Suricata**), and external threat intelligence (**VirusTotal API**) are ingested, normalized, correlated, and visualized in real time.

---

## 🏗️ Architecture & Network Topography

```
                       +-------------------------+
                       |   Attacker Machine      |
                       |      (Kali Linux)       |
                       +------------+------------+
                                    |
                                    | [SSH Brute-Force / Network Traffic]
                                    v
+------------------+       +---------------------+       +-------------------+
|  pfSense Firewall| ----> |  Suricata IDS/IPS   | ----> |  Windows Endpoint |
|  (Syslog / UDP)  |       | (EVE JSON Alerts)   |       |   (Wazuh Agent)   |
+--------+---------+       +----------+----------+       +---------+---------+
         |                            |                            |
         | Network Logs               | IDS Alerts                 | Sysmon & Win Event Logs
         +--------------------+       |       +--------------------+
                              |       |       |
                              v       v       v
                       +-------------------------+       +-------------------+
                       |      Wazuh Server       | <---> |  VirusTotal API   |
                       | (Manager, Indexer, GUI) |       | (Threat Intel)    |
                       +-------------------------+       +-------------------+
```

### Technical Stack & Component Breakdown

| Role / Component | System Specification | Primary Responsibilities |
| :--- | :--- | :--- |
| **SIEM / XDR Engine** | Wazuh Manager v4.x (Linux VM) | Centralized log ingestion, correlation rules engine, OpenSearch indexer, and web dashboard. |
| **Monitored Endpoint** | Windows 11 Pro (`Yash`) | Endpoint target running Wazuh Agent, Microsoft Sysmon v15+, and OpenSSH Server daemon. |
| **Adversary Node** | Kali Linux | Offensive platform simulating threat actor TTPs (e.g., `hydra` SSH dictionary attacks). |
| **Perimeter Security** | pfSense 2.7.2 Router/Firewall | Network gateway handling packet filtering and remote syslog stream over UDP port 514. |
| **Network IDS/IPS** | Suricata Network Detection | Deep packet inspection engine generating structured `eve.json` threat alerts. |
| **Threat Intelligence** | VirusTotal API | Automated SHA-256 hash enrichment via Wazuh Integrator engine. |

---

## 🔄 End-to-End Telemetry Pipeline & Event Flow

This section details the exact lifecycle of security telemetry within the **ECDOMA-Lab**, tracing how a malicious network action is captured, transformed, indexed, and presented to a SOC analyst.

```
+-------------------+      +-------------------+      +-------------------+
| 1. Simulation     | ---> | 2. Generation     | ---> | 3. Agent Transport|
| Attacker (Kali)   |      | OS & Sysmon Logs  |      | Encrypted Stream  |
+-------------------+      +-------------------+      +-------------------+
                                                                |
+-------------------+      +-------------------+                |
| 6. Analyst Triage | <--- | 5. Indexing       | <--------------+
| Wazuh Dashboard   |      | OpenSearch Engine |   4. Server Processing & Decoders
+-------------------+      +-------------------+   (Rule Matching & Correlation)
```

1. **Phase 1: Adversarial Execution (Threat Simulation)**
   * The attacker executes a dictionary brute-force attempt using `hydra` targeting the Windows OpenSSH service (`192.168.230.1:22`).
2. **Phase 2: Local Telemetry Generation**
   * **Windows OS Security Auditing:** Generates **Event ID 4625** (Audit Failure) inside `C:\Windows\System32\Winevt\Logs\Security.evtx`.
   * **Sysmon Telemetry:** Captures process creation and child process executions (`sshd.exe` spawning authentication processes).
3. **Phase 3: Agent Collection & Forwarding**
   * The Windows Wazuh Agent continuously reads the `EventChannel` streams (`Security` & `Microsoft-Windows-Sysmon/Operational`) and forwards the telemetry over encrypted port 1514/UDP to the manager.
4. **Phase 4: Parsing, Decoding & Rule Correlation**
   * The Wazuh Manager passes raw logs through internal XML decoders (`windows_eventchannel`), extracting key-value pairs (Logon Type, Target Account, SubStatus).
   * The engine triggers **Rule 60122** (*Logon Failure - Unknown user or bad password*) at severity level 5 and maps the event to MITRE ATT&CK tactic **T1110 (Brute Force)**.
5. **Phase 5: Storage & Indexing**
   * The normalized alert object is stored in the OpenSearch Indexer (`wazuh-alerts-4.x-YYYY.MM.DD`).
6. **Phase 6: SOC Analyst Triage & Investigation**
   * The security analyst observes the high-velocity alert queue in the **Wazuh Threat Hunting** dashboard, opens the raw event fields, and validates the incident.

---

## 🛠️ Key Configurations & Technical Ingestion

### 1. Windows Endpoint & Sysmon Ingestion (`ossec.conf`)
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### 2. Suricata IDS Integration
```xml
<localfile>
  <location>C:\Program Files\Suricata\log\eve.json</location>
  <log_format>json</log_format>
</localfile>
```

### 3. File Integrity Monitoring (FIM) & VirusTotal API
```xml
<syscheck>
  <disabled>no</disabled>
  <directories check_all="yes" report_changes="yes" realtime="yes">C:\VT-Test</directories>
</syscheck>

<integration>
  <name>virustotal</name>
  <api_key>YOUR_VT_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

---

## ⚔️ Attack Simulation & Detection Scenario

### Scenario: SSH Dictionary Brute-Force Attack
* **Target:** Windows OpenSSH Server on `192.168.230.1`
* **Execution:**
  ```bash
  hydra -l administrator -P /usr/share/wordlists/rockyou.txt ssh://192.168.230.1 -t 4
  ```

### Dashboard Detection Summary
```
-----------------------------------------------------------------------------------
Timestamp             Agent   Rule Description                       Level   Rule ID
-----------------------------------------------------------------------------------
Nov 6, 2026 20:18:30  Yash    Logon Failure - Unknown user/bad pass   5       60122
Nov 6, 2026 20:18:30  Yash    Logon Failure - Unknown user/bad pass   5       60122
Nov 6, 2026 20:18:30  Yash    Logon Failure - Unknown user/bad pass   5       60122
Nov 6, 2026 20:18:31  Yash    Windows command prompt started          4       92052
Nov 6, 2026 20:18:31  Yash    Suspicious Windows cmd shell exec       3       92032
-----------------------------------------------------------------------------------
```

---

## 🔎 Deep-Dive Event Triage (Windows Event ID 4625 Breakdown)

| SIEM Data Field | Raw Event Value | SOC Analyst Analysis & Interpretation |
| :--- | :--- | :--- |
| `agent.name` | `Yash` | Identifies the target endpoint host. |
| `rule.id` | `60122` | Wazuh rule mapping for Windows authentication failures. |
| `data.win.system.eventID` | `4625` | Native Windows Security Audit ID for failed logon. |
| `data.win.eventdata.processName` | `.../OpenSSH/sshd.exe` | Proves the entry vector was the OpenSSH service daemon. |
| `data.win.eventdata.logonType` | `8` | NetworkCleartext logon (standard for SSH password auth). |
| `data.win.eventdata.targetUserName` | `administrator` | Identifies targeted administrative account. |
| `data.win.eventdata.subStatus` | `0xc000006a` | Specific NT STATUS code denoting "Bad Password". |
| `rule.firedtimes` | `6,886` | Confirms automated, high-velocity dictionary attack pattern. |

---

## 🎯 Key Competencies & Engineering Takeaways

1. **SIEM Engineering & Architecture:** Full setup of agent-manager client architectures, XML agent configurations, and JSON/EventChannel log collectors.
2. **SOC Incident Response:** Real-time alert triage, indicator extraction (IoCs), process lineage analysis, and log correlation.
3. **Adversarial Emulation:** Simulating real-world attack vectors using offensive tools (`hydra`) mapped to the **MITRE ATT&CK Framework**.
4. **Threat Intelligence Enrichment:** Automating hash verification via VirusTotal APIs to reduce mean time to detect (MTTD).
