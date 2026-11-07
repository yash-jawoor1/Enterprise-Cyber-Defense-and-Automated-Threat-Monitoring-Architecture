# 🚨 Step 6: SSH Brute Force Attack Simulation, Detection & Analysis

## 1. Overview & Threat Context

A **brute force attack** is a trial-and-error method where an attacker systematically submits multiple password combinations until the correct credential is guessed. It remains one of the most common threat vectors used to gain unauthorized access to network services.

> **Disclaimer:** The following documentation describes a controlled security simulation performed within an isolated lab environment (`Kali Linux` attacker node targeting a `Windows 11` endpoint via OpenSSH). Unauthorized testing against production systems is illegal and strictly prohibited.

## 2. Attack Tooling & Wordlist Strategy (`rockyou.txt`)

### 2.1 Why Hydra?

**THC-Hydra** is a high-speed network login cracker supporting over 50 protocols (including SSH, FTP, HTTP, SMB, and RDP). It is widely used by security professionals to test authentication resilience against dictionary attacks.

### 2.2 The Industry Standard: `rockyou.txt`

Instead of using a trivial or custom file, the industry-standard **`rockyou.txt`** wordlist (containing over 14 million real-world passwords derived from historical data breaches, located at `/usr/share/wordlists/rockyou.txt` on Kali Linux) was utilized for the simulation.

## 3. Attack Simulation Procedure

### Step 3.1: Executing the Hydra Command from Kali Linux

From the attacker machine (Kali Linux), the network brute force attack was launched against the target Windows endpoint OpenSSH service over port 22:

Bash

```
hydra -l Yash -P /usr/share/wordlists/rockyou.txt ssh://<WINDOWS_IP>
```

- **`-l Yash`**: Specifies the targeted username on the target system.
    
- **`-P /usr/share/wordlists/rockyou.txt`**: Points to the comprehensive Kali Linux password wordlist.
    
- **`ssh://<WINDOWS_IP>`**: Defines the target protocol, port (22), and destination IP address.
    

### Step 3.2: Attack Execution Flow

- **Attacker Side (Kali Linux):** Reads passwords locally from `/usr/share/wordlists/rockyou.txt` and transmits network authentication requests across the internal network.
    
- **Target Side (Windows Endpoint):** The OpenSSH server daemon (`C:\Windows\System32\OpenSSH\sshd.exe`) receives the incoming connection attempts, processes authentication failures, and logs every failed try locally to the Windows Security Event Log.
    

## 4. Wazuh SIEM Detection & Windows Event Analysis

When Hydra floods the Windows SSH daemon, the local Windows Security Log captures the failed logon attempts under **Event ID 4625**, which are streamed in real-time to the central Wazuh Manager.

### 4.1 Windows Security Event ID 4625 Field Breakdown

|Field Name|Extracted Value|Analysis & Meaning|
|---|---|---|
|**`data.win.system.eventID`**|`4625`|An account failed to log on.|
|**`data.win.system.channel`**|`Security`|Source event log channel on Windows.|
|**`data.win.eventdata.processName`**|`C:\Windows\System32\OpenSSH\sshd.exe`|Identifies the targeted service daemon (OpenSSH).|
|**`data.win.eventdata.logonType`**|`8`|Network Cleartext logon (credentials transmitted in cleartext over network, standard for SSH brute forcing).|
|**`data.win.eventdata.status`**|`0xc000006d`|General logon failure status code.|
|**`data.win.eventdata.subStatus`**|`0xc0000064`|Unknown username or bad password.|
|**`rule.firedtimes`**|`14+`|Multiple repeated failures indicating an automated brute force pattern rather than an isolated typo.|

## 5. Defensive Countermeasures & Remediation

To mitigate network brute force risks against SSH services, implement the following hardening practices:

- **Enforce Key-Based Authentication:** Disable password authentication entirely in `sshd_config` and enforce strict SSH public keys.
    
- **Rate-Limiting & Account Lockout:** Configure account lockout policies or deploy intrusion prevention tools (e.g., `fail2ban`) to block malicious source IPs after repeated failures.
    
- **Network Segmentation:** Restrict SSH access (port 22) to trusted administrative subnet ranges using perimeter firewall rules (pfSense).
    
- **SIEM Monitoring:** Continuously monitor Windows Security Event ID 4625 spikes to detect active credential-harvesting attacks.