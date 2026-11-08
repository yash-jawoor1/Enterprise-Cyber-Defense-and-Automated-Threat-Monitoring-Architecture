# 🛡️ Step 6: Sysmon Installation, Configuration & Wazuh Log Ingestion

## 1. Overview & Objectives

**System Monitor (Sysmon)** is an advanced Windows Sysinternals tool that acts as a high-definition logging agent for Windows endpoints.

- **The Problem:** Native Windows event logs often lack deep process telemetry (such as parent-child process relationships, command-line arguments, hashes, and outbound network socket connections).
    
- **The Solution:** Sysmon fills this visibility gap by recording detailed system activity. By applying a tuned noise-reduction configuration (such as _SwiftOnSecurity's_ hardened configuration file), common background processes are filtered out, allowing security analysts to focus strictly on suspicious process execution and threat indicators.
    

### Core Objectives

- Install Sysmon and apply an enterprise configuration XML filter.
    
- Verify local event logging on the Windows endpoint using `wevtutil` and command-line execution (`whoami.exe`).
    
- Configure the local Wazuh Agent to ingest `Microsoft-Windows-Sysmon/Operational` event channels.
    
- Restart the Wazuh service and validate live telemetry ingestion in the Wazuh Dashboard.
    

## 2. Step-by-Step Implementation

### Step 2.1: Installing & Configuring Sysmon

Open **Command Prompt or PowerShell as Administrator** on your Windows endpoint and execute:

DOS

```
# Install Sysmon and accept the End-User License Agreement
sysmon.exe -i -accepteula

# Apply the hardened SwiftOnSecurity XML noise-reduction configuration
sysmon -c sysmonconfig.xml
```

### Step 2.2: Local Verification (Windows Endpoint Side)

Before forwarding logs to the SIEM, verify that local event logging is actively capturing process execution. Run the following command to query Event ID 1 (Process Creation):

DOS

```
wevtutil qe Microsoft-Windows-Sysmon/Operational "/q:*[System[(EventID=1)]]" /c:3 /rd:true /f:text
```

- **Exclusion Testing:** Opening benign utilities like `notepad.exe` will **not** generate an event (intentionally suppressed by the SwiftOnSecurity configuration to prevent log bloat).
    
- **Inclusion Testing:** Executing administrative utilities like `whoami.exe` or administrative command shells generates **Sysmon Event ID 1 (Process Create)**, confirming that local telemetry is operating properly.
    

### Step 2.3: Forwarding Sysmon Logs to Wazuh Manager

To ensure the central SIEM processes these telemetry streams, configure the local Wazuh Agent.

1. Open the agent configuration file with administrative privileges:
    
    PowerShell
    
    ```
    notepad.exe "C:\Program Files (x86)\ossec-agent\ossec.conf"
    ```
    
2. Insert the Sysmon `<localfile>` block directly **inside** the main `<ossec_config> ... </ossec_config>` tags:
    
    XML
    
    ```
    <ossec_config>
      <!-- Existing configurations -->
    
      <localfile>
        <location>Microsoft-Windows-Sysmon/Operational</location>
        <log_format>eventchannel</log_format>
      </localfile>
    
    </ossec_config>
    ```
    
3. Restart the Wazuh Agent service to apply the configuration:
    
    PowerShell
    
    ```
    Restart-Service -Name wazuh
    ```
    

### Step 2.4: Validating Telemetry on the Wazuh Dashboard

1. Log into the central **Wazuh Dashboard** web interface.
    
2. Navigate to **Modules** →\rightarrow→ **Threat Hunting** / **Discover**.
    
3. Apply a filter for your endpoint hostname (e.g., `agent.name: Yash` or `win.system.providerName: Microsoft-Windows-Sysmon`).
    
4. **Verification:** Confirm that live events (such as process creation, command-line arguments, and rule triggers like _"Suspicious Windows cmd shell execution"_) are actively populating the SIEM.
    

## 3. SOC Analyst Verification Checklist

- [x] Sysmon installed successfully with `sysmonconfig.xml` filter applied.
    
- [x] Local Event ID 1 generation verified via `wevtutil` during administrative process execution.
    
- [x] Wazuh Agent `ossec.conf` updated to ingest `Microsoft-Windows-Sysmon/Operational` via `eventchannel`.
    
- [x] Wazuh Agent service successfully restarted (`Restart-Service`).
    
- [x] Live Sysmon telemetry and process-creation alerts confirmed visible in the Wazuh Dashboard.
