

# Brute Force Attack: Simulation, Detection & Defense

## 33. Introduction

A brute force attack is a method where an attacker tries multiple password or key combinations repeatedly until the correct one is found. It is one of the simplest yet most common attack techniques used to gain unauthorized access to systems, applications, or accounts.

> **Disclaimer:** The following content is for educational and ethical purposes only. These techniques should only be executed in a controlled lab environment with explicit permission. Unauthorized use against systems you do not own or manage is illegal and punishable by law.

## 34. Types of Brute Force Attacks

- **Simple brute force attack:** Directly attempting all possible combinations of passwords.
    
- **Dictionary attack:** Using a predefined list of likely passwords (a “dictionary”) instead of trying every possible combination.
    
- **Hybrid brute force attack:** Combining dictionary attacks with variations (e.g., adding numbers or symbols).
    
- **Reverse brute force attack:** Starting with a single commonly used password and trying it against many usernames.
    
- **Credential stuffing:** Using leaked username-password pairs from previous data breaches to attempt logins on other systems.
    

## 35. Popular Brute Force Tools

- **John the Ripper:** A fast password cracker for Unix, Windows, and other platforms.
    
- **Hashcat:** A powerful GPU-based password recovery tool, ideal for cracking password hashes.
    
- **Aircrack-ng:** Focused on cracking Wi-Fi network keys.
    
- **L0phtCrack:** Used to audit and recover Windows passwords.
    

## 36. Hydra for Brute Force Attacks

Hydra is one of the most widely used tools for network login brute forcing. It can perform rapid dictionary-based attacks against more than 50 protocols, including HTTP, SSH, FTP, RDP, Telnet, and more.

### 36.1 Hydra Command Structure

Bash

```
hydra -L <user_file> -P <pass_file> <service>://<target_ip>
```

- `-L`: Specifies the file containing a list of usernames.
    
- `-P`: Specifies the file containing a list of passwords.
    

## 37. Brute Force SSH

A brute force SSH attack attempts to gain unauthorized access to a remote machine by repeatedly trying different username and password combinations against the SSH service (port 22 by default).

### 37.1 Example Hydra Command

Bash

```
hydra -l Yash -P passwords.txt ssh://<WINDOWS_IP>
```

## 38. System Requirements

- **Attacker Machine:** Kali Linux
    
- **Target Machine:** Windows system (with SSH enabled via OpenSSH Server)
    

## 39. Simulating an SSH Brute Force Attack with Hydra

### 39.1 Step 1: Preparing the Password List

On the attacker machine (Kali Linux), we first create a list of possible passwords in a file named `passwords.txt`. This file will be used by Hydra to attempt login combinations against the target system.

### 39.2 Step 2: Running the Hydra Command

Since we already know the username of the target system is `Yash`, we use the `-l` option to specify it directly. If we did not know the username, we could use the `-L` option along with a file containing a list of usernames.

Bash

```
hydra -l Yash -P passwords.txt ssh://<WINDOWS_IP>
```

### 39.3 Step 3: Observing the Attack

Hydra will automatically attempt multiple login attempts using the given username and password list. Each failed login attempt is logged by the target’s SSH service (`sshd.exe`).

## 40. Detection in Wazuh

After simulating the brute force attack, we switch to the Wazuh dashboard. In the Threat Hunting section, we can observe key indicators:

- Multiple failed login attempts for the SSH service.
    
- Alerts triggered by Wazuh security rules detecting brute force activity.
    
- Event details, such as the source IP (attacker machine) and the affected service (SSH).
    

### Windows Security Event Analysis (Event ID 4625)

| Field Name                            | Field Value                            | Meaning                                                                                                   |
| ------------------------------------- | -------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `data.win.eventdata.failureReason`    | `%%2313`                               | Unknown username or bad password.                                                                         |
| `data.win.eventdata.keyLength`        | `0`                                    | No encryption key was used.                                                                               |
| `data.win.eventdata.logonProcessName` | `Advapi`                               | Typical for services, not interactive logins.                                                             |
| `data.win.eventdata.logonType`        | `8`                                    | Logon type 8 = Network Cleartext. Credentials were sent in cleartext over the network (common with SSHD). |
| `data.win.eventdata.processId`        | `0x65d8`                               | The process ID (`0x65d8`) of the process that attempted the logon.                                        |
| `data.win.eventdata.processName`      | `C:\Windows\System32\OpenSSH\sshd.exe` | Confirms someone tried to connect to this machine over SSH.                                               |
| `data.win.eventdata.status`           | `0xc000006d`                           | Status = Logon failure.                                                                                   |
| `data.win.eventdata.subStatus`        | `0xc0000064`                           | User logon with unknown username OR bad password.                                                         |
| `data.win.system.channel`             | `Security`                             | The event comes from the Windows Security log.                                                            |
| `data.win.system.eventID`             | `4625`                                 | Event ID 4625 = An account failed to log on.                                                              |
| `data.win.system.eventRecordID`       | `919137`                               | A unique sequential ID for this event in the event log.                                                   |
| `data.win.system.systemTime`          | `2026-11-05T12:00:00.0000000Z`         | Event timestamp (UTC).                                                                                    |
| `decoder.name`                        | `windows_eventchannel`                 | Wazuh parsed this as a Windows Event Channel log.                                                         |
| `input.type`                          | `log`                                  | Confirms this came from a log source (Windows logs).                                                      |
| `rule.firedtimes`                     | `14`                                   | At least 14 failed logins of this type have occurred.                                                     |

### Summary of Findings

- The Windows machine received multiple SSH login attempts via `sshd.exe`.
    
- Authentication failed with error _Unknown username or bad password_ (`0xc0000064`).
    
- This was logged locally as **Event ID 4625 (failed logon)**.
    
- Wazuh classified it under failed authentication events, mapped it to compliance frameworks, and recorded multiple rule firings, indicating an active brute force pattern rather than an isolated typo.
    

## 41. Defensive Countermeasures

While detection is critical, prevention strengthens system defenses against brute force attempts. Recommended strategies include:

- Use strong and unique passwords.
    
- Enable multi-factor authentication (MFA).
    
- Regularly monitor login activity via SIEM solutions.
    
- Implement rate-limiting, account lockout policies, or fail2ban mechanisms to restrict login attempts after repeated failures.
  
  
  
  
  ![[Brute Force Attack.png]]
  
  