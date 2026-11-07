## 🛠️ Step 1: Wazuh OVA Installation & Dashboard Access

### 1. Overview & Architecture Role

The **Wazuh All-in-One Virtual Appliance (OVA)** serves as the central command hub for the entire security monitoring ecosystem. It houses the **Wazuh Manager** (rules engine & agent coordinator), **Wazuh Indexer** (OpenSearch-based data storage), and **Wazuh Dashboard** (visual analyst portal) packaged on an optimized AlmaLinux 9 base system.

### 2. Prerequisites & Resource Allocation

Before deploying the virtual machine, ensure your hypervisor (VirtualBox / VMware Workstation / Proxmox) meets the minimum resource allocations for smooth log indexing:

|**Resource**|**Minimum Requirement**|**Recommended Setting**|
|---|---|---|
|**CPU Cores**|4 Cores|4 - 8 Cores|
|**RAM**|8 GB|8 GB - 16 GB|
|**Storage**|50 GB (Thick/Thin provisioned)|100 GB SSD|
|**Network Mode**|Bridged / Host-Only|Bridged (Same subnet as endpoints & Kali)|

### 3. Step-by-Step Deployment Procedure

#### Step 3.1: Download & Import Virtual Machine

1. Download the official pre-built Wazuh OVA file (Version 4.x) from the official repository.
    
2. Open your hypervisor and select **Import Appliance**.
    
3. Map the network interface to your designated lab network adapter (**Bridged Adapter** recommended so the Windows agent and Kali attacker can communicate directly with the manager IP).
    

#### Step 3.2: First Boot & Credential Setup

1. Power on the virtual machine and wait for the system boot sequence to complete.
    
2. Log into the local shell interface using default system credentials (if prompted) or observe the auto-generated IP address assigned by your DHCP server.
    
3. Locate the management IP address using the interface command:
    
    Bash
    
    ```
    ip a
    ```
    
    _(Note down the IP address, e.g., `192.168.1.100` or `192.168.230.X`)_.
    

#### Step 3.3: Initial Service Verification

Verify that all core Wazuh services are active and running in the background before launching the web UI:

Bash

```
# Verify Manager status
systemctl status wazuh-manager

# Verify Indexer status
systemctl status wazuh-indexer

# Verify Dashboard status
systemctl status wazuh-dashboard
```

### 4. Accessing & Validating the Wazuh Dashboard

1. Open a web browser on your host machine and navigate to the HTTPS endpoint:
    
    Plaintext
    
    ```
    https://<WAZUH_SERVER_IP>
    ```
    
    _(Accept the self-signed TLS security warning)_.
    
2. Enter the default administrator credentials:
    
    - **Username:** `admin`
        
    - **Password:** _(Retrieved from setup console / output script)_
        
3. **Dashboard Health Check:**
    
    - Navigate to **Management** $\rightarrow$ **Cluster / Nodes**.
        
    - Confirm that `wazuh-server` reports an **Active** status.
        
    - Verify the **Security Events** baseline tab loads without database connection errors.
        

### 5. SOC Analyst Verification Checklist

- [x] Wazuh OVA successfully imported with 4+ CPU cores and 8GB+ RAM.
    
- [x] Static / DHCP IP assigned and pingable from Kali and Windows nodes.
    
- [x] All core services (`wazuh-manager`, `wazuh-indexer`, `wazuh-dashboard`) running without errors.
    
- [x] Web GUI accessible via `https://<IP>` with active administrative privileges.