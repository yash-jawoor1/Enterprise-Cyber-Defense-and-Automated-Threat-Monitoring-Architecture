# 🛡️ Step 4: Suricata IDS/IPS Integration with Wazuh SIEM

## 1. Overview & Objectives

An **Intrusion Detection System (IDS)** and **Intrusion Prevention System (IPS)** form a core pillar of a layered cybersecurity architecture, continuously monitoring network traffic to identify suspicious activities, malicious connections, and active threats.

In this phase, **Suricata IDS** is deployed on the Windows endpoint (`WIN-ENDPOINT-01`) and integrated with the **Wazuh SIEM/XDR** platform to centralize network telemetry and security alerts.

### Core Objectives

- Install and configure Suricata on the Windows endpoint.
    
- Capture and inspect network traffic using **Npcap**.
    
- Configure Suricata detection rules (`local.rules` and Emerging Threats rules).
    
- Enable structured **EVE JSON** logging.
    
- Ingest Suricata logs via the local Windows **Wazuh Agent**.
    
- Monitor and verify network security events in the Wazuh Dashboard.
    

## 2. IDS vs. IPS Conceptual Overview

|**Feature**|**Intrusion Detection System (IDS)**|**Intrusion Prevention System (IPS)**|
|---|---|---|
|**Traffic Monitoring**|Yes (Passive)|Yes (Active inline)|
|**Threat Detection**|Yes|Yes|
|**Alert Generation**|Yes|Yes|
|**Traffic Blocking**|**No**|**Yes** (Drops malicious packets)|
|**Primary Purpose**|Detection, Logging & Analysis|Prevention, Mitigation & Blocking|

## 3. Suricata Installation & Configuration

### 3.1 Installation Procedure

1. Download the official Windows `.msi` installer from the [Suricata Download Page](https://suricata.io/download/).
    
2. Execute the installer and complete the setup wizard.
    
3. Default installation directory: `C:\Program Files\Suricata\`
    

### 3.2 Npcap Packet Capture Prerequisite

Suricata relies on **Npcap** to capture raw network packets from Windows network adapters.

1. Download and install Npcap with **WinPcap API-compatible mode** enabled and set to start at boot.
    
2. Verify the Npcap service status via PowerShell (Administrator):
    
    PowerShell
    
    ```
    Get-Service -Name npcap
    ```
    
    _Expected Output:_ Status should report **`Running`**.
    

## 4. Configuration File Setup (`suricata.yaml`)

Edit the main configuration file located at `C:\Program Files\Suricata\suricata.yaml`:

### 4.1 Interface Selection

Identify the active network interface name using PowerShell (`Get-NetAdapter | Select-Object Name, Status`), then bind Suricata using `af-packet`:

YAML

```
af-packet:
  - interface: "Wi-Fi"  # Replace with active adapter name (e.g., Ethernet or Wi-Fi)
```

### 4.2 Rule Files Configuration

Configure rule sets to include local definitions and Emerging Threats rules:

YAML

```
rule-files:
  - local.rules
  - emerging-all.rules
```

### 4.3 Enable EVE JSON Output

EVE JSON provides structured event logs optimized for SIEM ingestion:

YAML

```
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
```

_Log Output Path:_ `C:\Program Files\Suricata\log\eve.json`

## 5. Running Suricata & Verifying Logs

Open PowerShell as Administrator, navigate to the installation directory, and start Suricata:

PowerShell

```
cd "C:\Program Files\Suricata\"
.\suricata.exe -c .\suricata.yaml -i "Wi-Fi"
```

### Key Suricata Log Files (`C:\Program Files\Suricata\log\`)

- **`eve.json`**: Structured JSON log containing security alerts, DNS queries, TLS handshakes, and flow data.
    
- **`fast.log`**: Human-readable text file showing brief intrusion alerts.
    
- **`stats.log`**: Engine performance and packet-drop statistics.
    

## 6. Integrating Suricata with Wazuh SIEM

### Step 6.1: Configure Wazuh Agent Log Ingestion

On the Windows endpoint, open the Wazuh Agent configuration file (`C:\Program Files (x86)\ossec-agent\ossec.conf`) and add the following JSON localfile block inside the `<ossec_config>` root element:

XML

```
<localfile>
    <log_format>json</log_format>
    <location>C:\Program Files\Suricata\log\eve.json</location>
</localfile>
```

### Step 6.2: Restart the Wazuh Agent Service

Apply configuration changes by restarting the Windows agent service via PowerShell (Administrator):

PowerShell

```
Restart-Service -Name WazuhSvc
```

### Step 6.3: Generate Traffic & Verify EVE Log Ingestion

1. Generate test network traffic (e.g., `ping google.com` or browsing HTTPS sites).
    
2. Inspect the latest entries in the EVE JSON log:
    
    PowerShell
    
    ```
    Get-Content "C:\Program Files\Suricata\log\eve.json" -Tail 20
    ```
    
3. Navigate to the **Wazuh Dashboard** $\rightarrow$ **Threat Hunting** to filter and confirm that Suricata network alerts and event logs are actively populating the SIEM index.
    

## 7. SOC Analyst Verification Checklist

- [x] Suricata successfully installed (`C:\Program Files\Suricata\`).
    
- [x] Npcap packet capture driver verified and running (`Running`).
    
- [x] `suricata.yaml` configured with the correct network interface and rule files (`local.rules`, `emerging-all.rules`).
    
- [x] EVE JSON output enabled and writing to `eve.json`.
    
- [x] Wazuh agent `ossec.conf` updated to ingest JSON logs from the Suricata log directory.
    
- [x] Wazuh Agent service restarted successfully (`WazuhSvc`).
    
- [x] Suricata events verified in the Wazuh Dashboard Threat Hunting portal.
