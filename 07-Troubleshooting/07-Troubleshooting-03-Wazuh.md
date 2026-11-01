

# Wazuh Manager Startup Timeout and API Connectivity Troubleshooting

## 1. Issue Overview

During the initial Wazuh deployment, the Wazuh Dashboard displayed a server readiness issue, and the Wazuh Manager service failed to start correctly through systemd.

The following issues were identified:

- Wazuh Manager service reported a startup timeout.
    
- Wazuh API was initially unavailable.
    
- Dashboard API connection showed Offline.
    
- The systemd service had a startup timeout configured to 45 seconds.
    

## 2. Initial Service Investigation

Checked the Wazuh Manager service:

```
sudo systemctl status wazuh-manager --no-pager
```

The service showed:

```
Active: failed (Result: timeout)
```

Inspected the Wazuh Manager processes:

```
sudo /var/ossec/bin/wazuh-control status
```

The Manager processes were initially stopped.

## 3. Manually Starting Wazuh Manager

Started the Manager manually to investigate whether its components could initialize successfully:

```
sudo /var/ossec/bin/wazuh-control start
```

The command completed successfully and started the main Wazuh components, including:

- Wazuh API
    
- Wazuh Analysisd
    
- Wazuh Modulesd
    
- Wazuh Database
    
- Wazuh Remoted
    
- Wazuh Authd
    
- Wazuh Syscheckd
    
- Wazuh Logcollector
    

Verified the running processes:

```
sudo /var/ossec/bin/wazuh-control status
```

## 4. Verifying Wazuh API Connectivity

Checked whether the Wazuh API was listening on port 55000:

```
sudo ss -lntp | grep 55000
```

The API was listening successfully.

Tested the API locally:

```
curl -k -I https://127.0.0.1:55000
```

Received:

```
HTTP/1.1 401 Unauthorized
server: Wazuh
```

**Interpretation:** The API was reachable and responding. The HTTP 401 response was expected because the request did not include authentication credentials.

## 5. Verifying Wazuh Network Ports

Checked the main Wazuh communication ports:

```
sudo ss -lntp | grep -E '55000|1514|1515'
```

Verified listeners:

|Port|Purpose|Status|
|---|---|---|
|1514|Wazuh agent communication|Listening|
|1515|Agent enrollment|Listening|
|55000|Wazuh API|Listening|

## 6. Fixing the systemd Startup Timeout

Inspected the existing service configuration:

```
sudo systemctl cat wazuh-manager
```

The original configuration contained:

```
TimeoutSec=45
```

Created a systemd override directory:

```
sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
```

Created an override configuration:

```
printf '[Service]\nTimeoutStartSec=180\nTimeoutStopSec=180\n' | sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf
```

Reloaded systemd:

```
sudo systemctl daemon-reload
```

Verified the timeout settings:

```
sudo systemctl show wazuh-manager -p TimeoutStartUSec -p TimeoutStopUSec
```

Cleared the previous failed service state:

```
sudo systemctl reset-failed wazuh-manager
```

Started the Manager through systemd:

```
sudo systemctl start wazuh-manager
```

Checked its service status:

```
sudo systemctl status wazuh-manager --no-pager
```

## 7. Dashboard API Connection Verification

Navigated to:

**Server Management → Settings → API Connections**

The default API connection displayed:

|Setting|Value|
|---|---|
|Connection ID|default|
|Manager|wazuh-server|
|Host|[https://127.0.0.1](https://127.0.0.1/)|
|Port|55000|
|Username|wazuh-wui|
|Connection status|Online|

This confirmed successful communication between the Wazuh Dashboard and the Wazuh API.

  

## 8. Wazuh Dashboard Overview

After restoring API connectivity, opened the Wazuh Dashboard Overview.

The dashboard displayed:

- Agent registration summary
    
- Security alerts for the last 24 hours
    
- Critical, high, medium, and low severity alert counts
    
- Endpoint Security features
    
- Threat Intelligence features
    

The overview showed:

|Alert Severity|Count|
|---|---|
|Critical|0|
|High|0|
|Medium|5|
|Low|60|

The dashboard also indicated that no agents had been registered yet.

  

## 9. Troubleshooting Outcome

The investigation established that:

- Wazuh Manager components could be started manually.
    
- Wazuh API was responding on port 55000.
    
- Agent communication and enrollment ports were listening.
    
- The Dashboard API connection was Online.
    
- The Wazuh Dashboard Overview was accessible.
    

The systemd startup timeout was increased from 45 to 180 seconds using a service override.

**Next step:** Deploy a Wazuh agent and begin endpoint monitoring.

## 10. Key Learnings

1. Used systemd commands to investigate service startup failures.
    
2. Learned to inspect Wazuh Manager processes using `wazuh-control`.
    
3. Verified API availability using `curl`.
    
4. Identified the purpose of Wazuh ports 1514, 1515, and 55000.
    
5. Learned how to create systemd service overrides without modifying the original vendor service file.
    
6. Verified Dashboard-to-API connectivity through the API Connections page.
    

---

**Environment:** VMware Workstation  
**Platform:** Wazuh OVA  
**Wazuh Version:** 4.14.9  
**Operating System:** Amazon Linux 2023  
**Documentation:** Wazuh SOC Lab