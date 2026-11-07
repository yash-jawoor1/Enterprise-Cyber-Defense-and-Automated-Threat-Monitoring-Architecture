# 🛡️ Step 5: VirusTotal Threat Intelligence Integration & File Integrity Monitoring (FIM)

## 1. Overview & Objectives

In a modern Security Operations Center (SOC), alerts alone do not always provide sufficient context to determine whether a file is malicious or benign. **Threat Intelligence** solves this by enriching endpoint telemetry with external reputation data.

In this phase, **VirusTotal** is integrated with the **Wazuh Manager** to automatically check file hashes captured via **File Integrity Monitoring (FIM)** on the Windows endpoint (`WIN-ENDPOINT-01`).

### Core Objectives

- Integrate the VirusTotal API with the central Wazuh Manager.
    
- Configure real-time File Integrity Monitoring (FIM) on a dedicated Windows directory (`C:\VT-Test`).
    
- Automatically submit file hashes for VirusTotal threat intelligence lookups.
    
- Validate network routing and API connectivity (`ip route` configuration).
    
- Verify FIM alert generation (creation, modification, deletion) and VirusTotal enrichment in the Wazuh Dashboard.
    

## 2. Architecture & Integration Workflow

```
[Windows Endpoint: C:\VT-Test] 
       │  (File Creation / Modification)
       ▼
[Wazuh Agent (FIM)] 
       │  (Calculates MD5/SHA256 hashes & sends syscheck event)
       ▼
[Wazuh Manager (Linux)] 
       │  (Triggers integratord & queries VirusTotal API v2/v3)
       ▼
[VirusTotal API Gateway] 
       │  (Returns hash reputation & detection ratio)
       ▼
[Wazuh Dashboard] 
       │  (Displays enriched Rule 87103 alerts)
```

## 3. Step-by-Step Implementation

### Step 3.1: Configure VirusTotal Integration on Wazuh Manager

On the Linux Wazuh Manager, edit the global configuration file (`/var/ossec/etc/ossec.conf`) and append the integration block inside the main `<ossec_config>` root element:

XML

```
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VIRUSTOTAL_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

Validate configuration syntax and restart the manager services:

Bash

```
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

Verify that the integration process (`wazuh-integratord`) is active:

Bash

```
sudo /var/ossec/bin/wazuh-control status
# Expected output: wazuh-integratord is running
```

### Step 3.2: Configure Real-Time FIM on the Windows Endpoint

On the Windows endpoint (`WIN-ENDPOINT-01`), open the local Wazuh Agent configuration file (`C:\Program Files (x86)\ossec-agent\ossec.conf`) and add the real-time directory monitor:

XML

```
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>
  <directories realtime="yes">C:\VT-Test</directories>
</syscheck>
```

Restart the Windows agent via PowerShell (Administrator):

PowerShell

```
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

### Step 3.3: Network & Gateway Troubleshooting (Wazuh Server)

To ensure the Wazuh server can reach the VirusTotal API endpoints (`[www.virustotal.com](https://www.virustotal.com)`), verify the default gateway route:

Bash

```
# Check existing routes
ip route

# Add default route via pfSense LAN gateway if missing
sudo ip route add default via 192.168.1.1 dev eth0

# Verify connectivity & DNS resolution
ping -c 4 8.8.8.8
curl -I --connect-timeout 15 https://www.virustotal.com
```

## 4. Testing & Validation

### Step 4.1: Simulate File Lifecycle Events on Windows

Run the following PowerShell script to create, modify, and delete a test file in the monitored directory:

PowerShell

```
$file = "C:\VT-Test\fim-validation.docx"

# 1. Create file
Set-Content $file "Initial content for Wazuh FIM validation"
Start-Sleep -Seconds 3

# 2. Modify file
Add-Content $file "First modification"
Start-Sleep -Seconds 3

# 3. Delete file
Remove-Item $file
```

### Step 4.2: Verify FIM & VirusTotal Enrichment Alerts

Create a dedicated test file to trigger VirusTotal hash lookups:

PowerShell

```
Set-Content "C:\VT-Test\virustotal-check.txt" "VirusTotal integration test"
```

Expected Wazuh Rule Generation:

- **Rule 554:** File added to the system (Level 5)
    
- **Rule 550:** File modified (Level 7)
    
- **Rule 553:** File deleted (Level 7)
    
- **Rule 87103:** VirusTotal Alert — Enrichment response (Group: `virustotal`)
    

## 5. SOC Analyst Verification Checklist

- [x] VirusTotal API key configured securely in `/var/ossec/etc/ossec.conf`.
    
- [x] `wazuh-integratord` service running successfully.
    
- [x] Windows FIM enabled for real-time monitoring on `C:\VT-Test`.
    
- [x] Network default route verified via pfSense gateway (`192.168.1.1`).
    
- [x] File creation, modification, and deletion events successfully captured.
    
- [x] VirusTotal API enrichment returning valid JSON responses (`Rule 87103`) in the Wazuh Dashboard.
