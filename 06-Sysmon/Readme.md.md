

# Sysmon Setup & Wazuh Ingestion

## 1. What is Sysmon?

- **Definition:** Sysmon (System Monitor) is a free tool from Microsoft Sysinternals that acts as an advanced, high-definition "security camera" for Windows.
    
- **Why We Need It:** Native Windows event logs miss deep process details. Sysmon records program execution, network connections, and file changes.
    
- **Why Use a Config File?** Sysmon records _everything_, creating massive noise. We applied **SwiftOnSecurity's XML config** (`sysmonconfig.xml`) to ignore normal background stuff (like `notepad.exe`) and only log important security activity.
    

## 2. Step 1: Installing & Configuring Sysmon

Executed in Windows Command Prompt / PowerShell as Administrator:

1. **Install Sysmon:**
    
    DOS
    
    ```
    sysmon.exe -i -accepteula
    ```
    
2. **Apply SwiftOnSecurity Noise Filter:**
    
    DOS
    
    ```
    sysmon -c sysmonconfig.xml
    ```
    

## 3. Step 2: Local Verification (Windows Side)

Before sending logs to Wazuh, we verified that Windows was actually recording Sysmon events locally.

- **Checking Events via Command Prompt (`cmd.exe`):**
    
    DOS
    
    ```
    wevtutil qe Microsoft-Windows-Sysmon/Operational "/q:*[System[(EventID=1)]]" /c:3 /rd:true /f:text
    ```
    
- **Key Finding (Exclusion Rule):**
    
    - Opening `notepad.exe` did **not** generate an event because SwiftOnSecurity intentionally excludes simple text editors to save space.
        
    - Running administrative commands like `whoami.exe` or background services (like Splunk binaries) generated **Event ID 1 (Process Create)**, proving local logging works.
        

## 4. Step 3: Forwarding Sysmon Logs to Wazuh

Wazuh needs to know _where_ to look for Sysmon events on the Windows machine.

1. **Open Wazuh Agent Config:**
    
    PowerShell
    
    ```
    notepad.exe "C:\Program Files (x86)\ossec-agent\ossec.conf"
    ```
    
2. **Add Sysmon Channel (CRITICAL FIX):** The `<localfile>` block **must** go _inside_ the main `<ossec_config> ... </ossec_config>` tags (placing it outside causes the parser to ignore it):
    
    XML
    
    ```
    <ossec_config>
      ...
      <localfile>
        <location>Microsoft-Windows-Sysmon/Operational</location>
        <log_format>eventchannel</log_format>
      </localfile>
    </ossec_config>
    ```
    
3. **Restart the Wazuh Agent:**
    
    PowerShell
    
    ```
    Restart-Service -Name wazuh
    ```
    

## 5. Step 4: Validating on the Wazuh Dashboard

1. Logged into the **Wazuh Web Console**.
    
2. Navigated to **Modules -> Threat Hunting / Discover**.
    
3. Set the filter to search for the Windows hostname (`Yash`) or Sysmon events:
    
    Plaintext
    
    ```
    agent.name: Yash
    ```
    
4. **Result:** Confirmed **68+ hits** coming through live from the Windows agent, including rule triggers like _"Suspicious Windows cmd shell execution"_ and process creation alerts.




































































![[Event viewer.png]]


![[Sysmon-events.png]]