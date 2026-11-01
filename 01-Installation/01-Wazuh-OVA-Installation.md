# Wazuh VM – Network Configuration

## 1. Objective

Configure network connectivity for the Wazuh virtual machine using VMware Bridged Networking and identify its IP address.

## 2. Lab Environment

| Component         | Configuration      |
| ----------------- | ------------------ |
| SIEM Platform     | Wazuh 4.14.9       |
| Hypervisor        | VMware Workstation |
| Network Mode      | Bridged            |
| Physical Network  | Wi-Fi              |
| Network Interface | eth0               |
| VM IP Address     | 192.168.x.x        |

## 3. VMware Network Configuration

Configured VMware VMnet0 using Bridged Networking and selected the physical Wi-Fi adapter (Realtek RTL8821CE).

**Why Bridged Networking?**

It connects the virtual machine directly to the physical network, allowing other devices on the same network to communicate with the Wazuh server, subject to firewall rules.

## 4. Identify the IP Address

Executed the following command inside the Wazuh Linux terminal:

```
ifconfig
```

The command displays network interface details, including IP address, subnet mask, MAC address and network statistics. 

### Network Configuration Details

| Parameter    | Value         |
| ------------ | ------------- |
| Interface    | eth0          |
| IPv4 Address | 192.168.x.x   |
| Subnet Mask  | 255.255.255.0 |
| Broadcast    | 192.168.x.x   |
| MAC Address  | 00:0c:29:x:x  |

## 5. Access Wazuh Dashboard

Opened Chrome on the Windows host and accessed:

```
https://192.168.x.x
```

**Screenshot – Wazuh Dashboard**

![[Pasted image 20261101171059.png|462]]
  

## 6. Troubleshooting

### Issue 1: IPv4 address not assigned

**Cause:** VMware Bridged Networking was not correctly configured.

**Solution:** Configured VMnet0 and selected the physical Wi-Fi adapter.

### Issue 2: Dashboard server not ready

**Error:** Wazuh dashboard server is not ready yet.

**Solution:** Investigated the Wazuh services and resolved the startup issue.

## 7. Learning Outcomes

- Configured VMware Bridged Networking.
    
- Understood DHCP and IPv4 addressing.
    
- Used `ifconfig` to inspect network interfaces.
    
- Accessed the Wazuh Dashboard from Windows.
    
- Practised Linux network troubleshooting.
    

## 8. Next Steps

- Explore Wazuh Dashboard components.
    
- Deploy a Windows endpoint agent.
    
- Deploy a Linux endpoint agent.
    
- Generate and investigate security alerts.