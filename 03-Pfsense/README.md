# 🌐 Step 3: pfSense Firewall & Wazuh SIEM Log Integration

## 1. Overview & Architecture Role

A **firewall** acts as the frontline network security gateway, monitoring and controlling incoming and outgoing traffic based on predefined security policies.

### Key Functions of pfSense Firewall

- **Perimeter Defense:** Prevents unauthorized network access and blocks malicious or suspicious traffic.
    
- **Traffic Segmentation:** Separates internal subnets (LAN) from external untrusted networks (WAN).
    
- **Audit Generation:** Produces structured system and packet filter logs (`filterlog`).
    

### Why Integrate pfSense with Wazuh?

While pfSense protects the network perimeter, **Wazuh SIEM** provides centralized log collection, parsing, correlation, and alerting. Integrating firewall telemetry into Wazuh creates an integrated SOC environment, allowing security analysts to investigate network events alongside host-level telemetry.

## 2. Lab Network Topology & Architecture

### 2.1 Component Requirements

- VMware Workstation Pro (Virtualization Platform)
    
- pfSense CE Firewall ISO image
    
- Running Wazuh Manager / Server instance (`v4.14.9`)
    
- Windows 11 Endpoint & Kali Linux testing VMs
    

### 2.2 Network Interface Configuration

pfSense utilizes two virtual network interfaces to separate internal assets from external traffic:

- **WAN (em0):** Connected via VMware NAT to the external network.
    
- **LAN (em1):** Connected to an isolated Host-Only network (`VMnet1`).
    

|**Component**|**Interface**|**IP Address / Subnet**|**Role**|
|---|---|---|---|
|**pfSense Firewall**|WAN (`em0`)|`<EXTERNAL_NAT_IP>`|Gateway / NAT interface|
|**pfSense Firewall**|LAN (`em1`)|`192.168.1.1`|Internal gateway / DHCP server|
|**Wazuh Server**|`eth0`|`<WAZUH_SERVER_IP>` (e.g., `192.168.1.128`)|Central SIEM & Analysis Engine|
|**Windows Endpoint**|`VMnet1`|`192.168.1.2`|Monitored Host (`Yash`)|

- **LAN Subnet:** `192.168.1.0/24`
    
- **DHCP Scope:** `192.168.1.100 – 192.168.1.199`
    

## 3. pfSense Web Interface Configuration

1. Access the pfSense management dashboard from a browser on the LAN network:
    
    Plaintext
    
    ```
    https://192.168.1.1
    ```
    
2. Log in using default credentials (`admin` / `pfsense`) and update them immediately post-installation.
    

## 4. Configuring Remote Syslog Forwarding

### Step 4.1: Enable Remote Logging in pfSense

1. Navigate to: **Status** $\rightarrow$ **System Logs** $\rightarrow$ **Settings**.
    
2. Configure remote syslog parameters to forward logs to the Wazuh server:
    

|**Setting**|**Configuration Value**|
|---|---|
|**Remote Syslog Server**|`<WAZUH_SERVER_IP>` (`192.168.1.128`)|
|**Protocol**|UDP|
|**Port**|`514`|
|**Firewall Log Source**|Selected (`Firewall` and `System` logs)|

### Step 4.2: Configure Wazuh Server Syslog Ingestion

On the Wazuh server, edit the global configuration file (`/var/ossec/etc/ossec.conf`) to open a UDP syslog listener socket inside the `<ossec_config>` block:

XML

```
<remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>192.168.1.1</allowed-ips>
    <local_ip><WAZUH_SERVER_IP></local_ip>
</remote>
```

### Step 4.3: Verify Incoming Syslog Traffic via tcpdump

Verify that raw syslog packets are successfully hitting the Wazuh network interface from the firewall gateway:

Bash

```
sudo tcpdump -ni eth0 -A -s0 udp port 514
```

_Sample Ingested Firewall Log:_

Plaintext

```
filterlog: 1,93,,100000101,em1,match,pass,in,4,0x0,,127,24609,0,DF,1,icmp,84,192.168.1.128,8.8.8.8,request,4,164
```

## 5. Custom Decoders & Detection Rules Engineering

Because pfSense outputs structured CSV-like filter logs, custom decoders and rules ensure precise field extraction and alerting.

### 5.1 Custom Decoder (`local_decoder.xml`)

Configuration path: `/var/ossec/etc/decoders/local_decoder.xml`

- **`pfsense-custom`**: Identifies incoming pfSense `filterlog` events.
    
- **`pfsense-fields`**: Extracts critical metadata including Event ID, interface, action (`pass`/`block`), traffic direction, protocol, source IP, and destination IP.
    

### 5.2 Custom Detection Rules (`local_rules.xml`)

Configuration path: `/var/ossec/etc/rules/local_rules.xml`

|**Rule ID**|**Description**|**Severity Level**|**Category**|
|---|---|---|---|
|**100111**|Generic pfSense log received|Level 3 (Informational)|General System Log|
|**100112**|Successful firewall login|Level 3 (Informational)|Authentication Success|
|**100113**|Allowed network traffic|Level 5 (Normal Alert)|Network Traffic Pass|
|**100114**|Blocked network traffic|Level 7 (Warning)|Firewall Block Drop|
|**100115**|Firewall authentication error|Level 10 (High Severity)|Security Incident / Auth Failure|

### 5.3 Configuration Syntax Validation

Validate the XML configuration and rules syntax before restarting the Wazuh analysis daemon:

Bash

```
sudo /var/ossec/bin/wazuh-analysisd -t
```

_(An empty output indicates zero syntax errors and successful validation)._

## 6. Testing and Validation (`wazuh-logtest`)

Using Wazuh's built-in testing utility (`sudo /var/ossec/bin/wazuh-logtest`), synthetic logs were evaluated against custom decoders:

- **Generic Log Test:** Matched Rule `100111` (Level 3) $\rightarrow$ **Passed**.
    
- **Allowed Traffic Test:** Matched Rule `100113` (Level 5) $\rightarrow$ **Passed** (Extracted traffic flow from internal source to external gateway).
    
- **Blocked Traffic Test:** Matched Rule `100114` (Level 7) $\rightarrow$ **Passed** (Correctly identified dropped packets).
    
- **Authentication Error Test:** Rule `100115` (Level 10) $\rightarrow$ _Pending refinement_ (Currently triggering default rule 2501; rule priority tuning in progress).
    

## 7. SOC Analyst Verification Checklist

- [x] pfSense virtual firewall deployed with WAN (NAT) and LAN (Host-Only) interfaces configured.
    
- [x] Remote syslog forwarding enabled over UDP port `514` pointing to the Wazuh Manager.
    
- [x] Live log ingestion verified using `tcpdump` packet capture on the SIEM server.
    
- [x] Custom decoders written and tested successfully to parse CSV filter logs.
    
- [x] Custom detection rules mapped for allowed (Level 5) and blocked (Level 7) traffic.
    
- [x] Analysis engine configuration validated using `wazuh-analysisd -t`.
