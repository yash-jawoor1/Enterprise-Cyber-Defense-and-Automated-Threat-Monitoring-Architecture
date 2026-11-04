
# Suricata IDS/IPS Integration with Wazuh SIEM

## 1. Introduction

Intrusion Detection Systems (IDS) and Intrusion Prevention Systems (IPS) are important components of a layered cybersecurity architecture.

They monitor network traffic to identify suspicious activities, malicious connections, and potential security threats.

In this project, I integrated Suricata IDS with Wazuh SIEM to monitor network traffic and centralize security alerts.

### Objectives

- Install and configure Suricata on a Windows endpoint.
    
- Capture and inspect network traffic using Npcap.
    
- Configure Suricata rules for threat detection.
    
- Enable EVE JSON logging.
    
- Integrate Suricata logs with the Wazuh Agent.
    
- Monitor Suricata security events through the Wazuh Dashboard.
    

---

## 2. IDS vs. IPS Overview

### 2.1 Intrusion Detection System (IDS)

An IDS monitors network traffic and identifies suspicious or malicious activities.

Detection methods include:

- Signature-based detection using predefined rules.
    
- Indicators of Compromise (IoCs), including hashes, domains, URLs, and network fingerprints.
    
- Lua scripting for customized detection logic.
    

An IDS generates alerts and logs when suspicious activity is detected. It does not actively block traffic.

### 2.2 Intrusion Prevention System (IPS)

An IPS monitors network traffic and takes preventive action based on configured security policies.

Its functions include:

- Detecting malicious network traffic.
    
- Dropping suspicious or malicious packets.
    
- Allowing legitimate network traffic.
    
- Preventing certain attacks in real time.
    

### IDS vs. IPS Comparison

|Feature|IDS|IPS|
|---|---|---|
|Traffic monitoring|Yes|Yes|
|Threat detection|Yes|Yes|
|Alert generation|Yes|Yes|
|Traffic blocking|No|Yes|
|Primary purpose|Detection|Prevention|

---

## 3. About Suricata

Suricata is an open-source network threat detection engine capable of operating as:

- Intrusion Detection System (IDS)
    
- Intrusion Prevention System (IPS)
    
- Network Security Monitoring (NSM) engine
    

In IDS mode, Suricata monitors network traffic and generates alerts when configured detection rules match.

In IPS mode, it can take action against malicious traffic, including dropping packets.

### 3.1 Download and Install Suricata

I used the official Suricata download page:

[https://suricata.io/download/](https://suricata.io/download/)

Installation steps:

1. Downloaded the Windows installer (`.msi`).
    
2. Executed the installer.
    
3. Followed the installation wizard.
    
4. Completed the installation using the selected installation settings.
    

**Default installation directory:**

`C:\Program Files\Suricata\`

### Important Files and Directories

|File / Directory|Purpose|
|---|---|
|`suricata.exe`|Main Suricata executable|
|`suricata.yaml`|Main configuration file|
|`rules\`|Directory containing detection rules|
|`log\`|Directory containing Suricata logs|

---

## 4. Npcap Installation and Verification

Suricata uses Npcap to capture network packets from Windows network interfaces.

### 4.1 Install Npcap

Downloaded Npcap from:

[https://npcap.com/#download](https://npcap.com/#download)

During installation, I selected:

- WinPcap API-compatible mode.
    
- Start the Npcap driver at boot time.
    

### 4.2 Verify Npcap Service

Opened PowerShell as Administrator and executed:

```
Get-Service -Name npcap
```

This command checks whether the Npcap packet capture driver service is installed and running.

**Expected status:** `Running`

If the service is stopped, it can be started using:

```
Start-Service -Name npcap
```

---

## 5. Configuring Suricata

### 5.1 Open the Configuration File

Opened the Suricata configuration file:

`C:\Program Files\Suricata\suricata.yaml`

This file controls network interfaces, rule files, logging, and other Suricata settings.

### 5.2 Identify Network Interfaces

To identify available Windows network adapters, executed:

```
Get-NetAdapter | Select-Object Name, Status
```

This command displays network adapter names and their operational status.

I used the appropriate active network interface for packet monitoring.

### 5.3 Configure Network Interface

Located the `af-packet` configuration section in `suricata.yaml`.

Example configuration:

```
af-packet:
  - interface: Wi-Fi
```

The interface name must match the actual Windows network adapter selected for monitoring.

### 5.4 Configure Rule Files

Located the `rule-files` section in `suricata.yaml`.

Configured the required rule files:

```
rule-files:
  - local.rules
  - emerging-all.rules
```

**Rule file explanation:**

- `local.rules`: Contains locally configured detection rules.
    
- `emerging-all.rules`: Contains Emerging Threats detection rules.
    

The rule files must exist in the configured rules directory.

### 5.5 Enable EVE JSON Logging

Configured EVE JSON output in `suricata.yaml`.

```
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
```

EVE JSON provides structured event information that can be consumed by SIEM platforms such as Wazuh.

The log file is generated in:

`C:\Program Files\Suricata\log\eve.json`

---

## 6. Running Suricata

Opened PowerShell as Administrator.

Navigated to the Suricata installation directory:

```
cd "C:\Program Files\Suricata\"
```

Started Suricata using its configuration file and selected network interface:

```
.\suricata.exe -c .\suricata.yaml -i "Wi-Fi"
```

The interface name should be replaced with the actual adapter name.

### 6.1 Suricata Log Files

Suricata stores its logs in:

`C:\Program Files\Suricata\log\`

Important log files:

|Log File|Purpose|
|---|---|
|`eve.json`|Structured JSON events and alerts|
|`fast.log`|Human-readable security alerts|
|`stats.log`|Suricata performance and statistics|

These logs provide information about network activity, detected threats, and engine performance.

---

## 7. Integrating Suricata with Wazuh SIEM

The next step was to integrate Suricata with Wazuh so that network security events could be centrally monitored.

### 7.1 Verify EVE JSON Output

Checked the Suricata log directory:

`C:\Program Files\Suricata\log\`

The primary file used for integration is:

`eve.json`

This file contains structured network events generated by Suricata.

### 7.2 Configure Wazuh Agent

On the Windows endpoint, opened the Wazuh Agent configuration file:

`C:\Program Files (x86)\ossec-agent\ossec.conf`

Added the following localfile configuration inside the existing `<ossec_config>` element:

```
<localfile>
    <log_format>json</log_format>
    <location>C:\Program Files\Suricata\log\eve.json</location>
</localfile>
```

**Configuration explanation:**

|Parameter|Purpose|
|---|---|
|`localfile`|Configures a log file for monitoring|
|`log_format`|Specifies JSON as the log format|
|`location`|Defines the path to the Suricata EVE JSON file|

This configuration allows the Wazuh Agent to monitor Suricata's EVE JSON logs and forward collected events to the Wazuh Manager.

### 7.3 Restart Wazuh Agent

Opened PowerShell as Administrator and executed:

```
Restart-Service -Name wazuh
```

This restarts the Wazuh Agent so that the updated configuration can be applied.

### 7.4 Generate Test Traffic

Generated network traffic using:

```
ping google.com
```

This produces ICMP traffic that can be observed by Suricata when the selected interface is capturing the traffic.

Note: A normal ping does not necessarily trigger a security alert. It can be used to check network visibility and event logging.

### 7.5 Verify Suricata Logs

Checked the Suricata EVE JSON file:

```
Get-Content "C:\Program Files\Suricata\log\eve.json" -Tail 20
```

This command displays the latest entries written to the EVE JSON log.

### 7.6 View Events in Wazuh Dashboard

Opened the Wazuh Dashboard and navigated to:

**Threat Hunting**

Used the available event filters to search for Suricata-related events.

The objective is to verify that Suricata events collected by the Windows Wazuh Agent are visible in the Wazuh SIEM.

---

## 8. Prerequisites and Dependencies

The following components are required for this integration:

1. A running Wazuh SIEM environment.
    
2. A connected Windows Wazuh Agent.
    
3. Suricata installed on the Windows endpoint.
    
4. Npcap installed and running.
    
5. A configured Suricata network interface.
    
6. Valid Suricata detection rule files.
    
7. EVE JSON logging enabled.
    

A VirusTotal API key is also required if VirusTotal integration is included in the broader lab workflow. It is not required for basic Suricata EVE JSON collection by the Wazuh Agent.

---

## 9. Conclusion

In this project, I configured Suricata as a network intrusion detection engine and integrated its EVE JSON logs with Wazuh SIEM.

The integration connects network-level threat monitoring with centralized security event analysis.

Suricata is responsible for inspecting network traffic and generating security events, while Wazuh collects and analyzes those events through its Windows Agent.

This setup demonstrates the integration of network security monitoring with SIEM-based log management and threat detection.

### Key Skills Demonstrated

- Network Intrusion Detection.
    
- Suricata IDS Configuration.
    
- Npcap Packet Capture.
    
- Network Traffic Monitoring.
    
- EVE JSON Logging.
    
- Windows Wazuh Agent Configuration.
    
- SIEM Log Integration.
    
- Security Event Monitoring.
  
  
  
  
  ![[Suricata-Installation.png]]
  
  
  ![[Npcap-Installation.png]]