
 

## 1. Introduction

A firewall is a critical network security solution that monitors and controls incoming and outgoing network traffic based on predefined security rules.

It acts as a security guard for a network by deciding which connections are allowed and which should be blocked.

### Key Functions of a Firewall

- Prevent unauthorized network access.
    
- Block malicious or suspicious traffic.
    
- Monitor incoming and outgoing connections.
    
- Segment networks to improve security.
    

### Project Overview

In this project, I built a virtual Security Operations Center (SOC) lab by integrating pfSense Firewall with Wazuh SIEM.

The objective is to collect firewall logs, analyze network events, and generate security alerts using custom decoders and detection rules.

**Technologies Used:**

|Technology|Purpose|
|---|---|
|pfSense CE 2.9.0|Firewall and network gateway|
|Wazuh 4.14.9|SIEM and security monitoring|
|VMware Workstation|Virtualization platform|
|Windows 11|Endpoint machine|
|Kali Linux|Security testing machine|

---

## 2. Lab Architecture and Requirements

### 2.1 Requirements

Before starting the integration, the following components are required:

1. VMware Workstation Pro.
    
2. pfSense ISO image.
    
3. A running Wazuh Manager.
    
4. Virtual machines for endpoint and security testing.
    

### 2.2 Network Configuration

The lab uses two network interfaces on pfSense:

- **WAN:** Connected through VMware NAT.
    
- **LAN:** Connected to an isolated Host-only network.
    

### Network Details

|Component|Interface|IP Address|
|---|---|---|
|pfSense|WAN (em0)|192.168.230.130|
|pfSense|LAN (em1)|192.168.1.1|
|Wazuh Server|eth0|192.168.1.128|
|Windows VM|VMnet1|192.168.1.2|

**Network Configuration:**

- LAN Network: `192.168.1.0/24`
    
- DHCP Range: `192.168.1.100 – 192.168.1.199`
    
- Default Gateway: `192.168.1.1`
    

### 2.3 pfSense Virtual Machine Configuration

The pfSense virtual machine was configured with:

- RAM: 2 GB recommended
    
- CPU: 2 cores
    
- Disk: 20 GB
    
- Network Adapter 1: NAT (WAN)
    
- Network Adapter 2: Host-only (LAN)
    

---

## 3. Accessing the pfSense Web Interface

After installing pfSense and assigning the network interfaces, access its web interface from a machine connected to the LAN network.

**URL:**

`https://192.168.1.1`

Default credentials:

- Username: `admin`
    
- Password: `pfsense`
    

Change the default credentials after the initial login.

The pfSense dashboard provides access to firewall rules, network interfaces, DHCP services, system logs, and other network settings.

---

## 4. Why Integrate pfSense with Wazuh?

pfSense and Wazuh perform different but complementary security functions.

### pfSense Firewall

- Monitors network traffic.
    
- Allows or blocks connections based on firewall rules.
    
- Generates firewall and system logs.
    
- Acts as the network security gateway.
    

### Wazuh SIEM

- Collects security logs.
    
- Analyzes events.
    
- Uses decoders to extract relevant fields.
    
- Applies detection rules.
    
- Generates security alerts.
    

### Integration Benefits

By integrating pfSense with Wazuh, firewall events can be centrally monitored and analyzed alongside endpoint security events.

This creates a small SOC environment for practicing security monitoring, log analysis, and threat detection.

---

## 5. Configuring pfSense Logs in Wazuh

### 5.1 Enable Remote Syslog in pfSense

Navigate to:

`Status → System Logs → Settings`

Enable remote logging and configure the Wazuh server as the remote syslog destination.

**Configuration:**

|Setting|Value|
|---|---|
|Remote Syslog Server|192.168.1.128|
|Protocol|UDP|
|Port|514|
|Firewall Log Source|pfSense|

Select the required log categories, including Firewall and System logs.

### 5.2 Configure Wazuh Syslog Input

On the Wazuh server, edit:

`/var/ossec/etc/ossec.conf`

Add the following remote syslog configuration inside the existing `<ossec_config>` root element:

```
<remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>192.168.1.1</allowed-ips>
    <local_ip>192.168.1.128</local_ip>
</remote>
```

**Configuration explanation:**

- `connection`: Configures the syslog connection.
    
- `port`: Specifies UDP port 514.
    
- `protocol`: Defines the transport protocol.
    
- `allowed-ips`: Restricts the permitted syslog sender.
    
- `local_ip`: Specifies the Wazuh server listening address.
    

### 5.3 Verify Incoming Syslog Traffic

To verify that pfSense is forwarding logs to Wazuh, execute:

```
sudo tcpdump -ni eth0 -A -s0 udp port 514
```

Firewall events were successfully received from pfSense.

Example firewall log:

```
filterlog: 1,93,,100000101,em1,match,pass,in,4,0x0,,127,24609,0,DF,1,icmp,84,192.168.1.128,8.8.8.8,request,4,164
```

This event represents allowed ICMP traffic from the Wazuh server to 8.8.8.8.

---

## 6. Creating a Custom Decoder and Rules for pfSense

pfSense firewall logs use a structured CSV-like format. Custom decoders help Wazuh extract relevant information from these events.

### 6.1 Custom Decoder

**Configuration file:**

`/var/ossec/etc/decoders/local_decoder.xml`

Two custom decoders were created:

- `pfsense-custom`: Identifies pfSense filterlog events.
    
- `pfsense-fields`: Extracts relevant firewall fields.
    

The decoder extracts information such as:

- Firewall event ID
    
- Interface
    
- Action
    
- Traffic direction
    
- Protocol
    
- Source IP
    
- Destination IP
    

The regular expression was adapted to the actual pfSense log format and tested using Wazuh Logtest.

### 6.2 Custom Detection Rules

**Configuration file:**

`/var/ossec/etc/rules/local_rules.xml`

Custom rules were created to identify different pfSense events.

|Rule ID|Description|Level|
|---|---|---|
|100111|Generic pfSense log received|3|
|100112|Successful login|3|
|100113|Allowed traffic|5|
|100114|Blocked traffic|7|
|100115|Authentication error|10|

### Rule Severity Levels

- **Level 3 – Informational:** General events and successful login messages.
    
- **Level 5 – Normal alert:** Allowed network traffic.
    
- **Level 7 – Warning:** Blocked network traffic.
    
- **Level 10 – High severity:** Authentication errors.
    

### 6.3 Configuration Validation

After creating the custom decoder and rules, validate the Wazuh analysis configuration:

```
sudo /var/ossec/bin/wazuh-analysisd -t
```

No output indicates that the configuration passed this validation.

---

## 7. Testing and Validation

Wazuh Logtest was used to verify decoder field extraction and custom detection rules.

Start the testing utility:

```
sudo /var/ossec/bin/wazuh-logtest
```

### 7.1 Generic pfSense Log

Expected rule: `100111`

Level: 3

Result: Passed.

### 7.2 Successful Login

Expected rule: `100112`

Level: 3

Result: Rule matched successfully. The source IP was not extracted in the synthetic test event.

### 7.3 Allowed Traffic

Expected rule: `100113`

Level: 5

Result: Passed.

Example alert:

```
pfSense: Allowed traffic from 192.168.1.128 to 8.8.8.8
```

### 7.4 Blocked Traffic

Expected rule: `100114`

Level: 7

Result: Passed.

Example alert:

```
pfSense: Blocked traffic from 192.168.1.128 to 8.8.8.8
```

### 7.5 Authentication Error

Expected rule: `100115`

Level: 10

Result: Pending.

During testing, Wazuh generated built-in Rule 2501 (Level 5) instead of the custom Rule 100115. Further rule-priority investigation is required.

---

## 8. Project Status

The following components have been configured and tested:

- pfSense virtual firewall installation.
    
- WAN and LAN interface configuration.
    
- Network connectivity between pfSense and Wazuh.
    
- Remote syslog forwarding over UDP port 514.
    
- Incoming firewall log verification using tcpdump.
    
- Custom pfSense decoder creation.
    
- Custom detection rule creation.
    
- Wazuh Logtest validation.
    

**Remaining validation:**

- Resolve custom authentication error rule 100115.
    
- Verify real-time alerts from live pfSense traffic.
    
- Verify alerts in the Wazuh dashboard.
    

---

## 9. Conclusion

This project demonstrates the integration of pfSense Firewall with Wazuh SIEM in a virtual SOC lab.

The integration enables centralized collection and analysis of firewall logs. Custom decoders and detection rules were created to identify allowed and blocked network traffic.

This hands-on lab provides practical experience with firewall administration, syslog integration, SIEM monitoring, log analysis, and custom security detection engineering.




![[Pasted image 20261103233924.png]]


![[Interface.png]]


![[Remote logging.png]]


![[04-pfsense-logs-received04-pfsense-logs-receive.png]]