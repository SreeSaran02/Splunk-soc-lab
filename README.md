# 🏠 Splunk SOC Lab

A hands-on Security Operations Center home lab built from scratch with **Splunk Enterprise** as the SIEM. This is the second lab in my SOC portfolio, a companion to my [Wazuh SOC Lab](https://github.com/SreeSaran02/home-soc-lab), built specifically to get hands-on with SPL, Splunk's data pipeline (Universal Forwarders → Indexer → Search Head), and multi-source correlation using both Windows Security logs and Sysmon.

I'm a Mechanical Engineering graduate (UK Hons) making the switch to cybersecurity. This lab is where I simulate attacks, detect them in Splunk, and write investigation reports the same way a SOC L1 analyst would on the job.

---

## 🏗️ Lab Architecture

| VM                              | Role              | OS                | IP Address              |
| ------------------------------- | ----------------- | ----------------- | ----------------------- |
| Splunk Enterprise 10.4.2        | SIEM / Indexer    | Ubuntu Server 22.04 | 192.168.29.97 (static) |
| Windows 10 Pro (DESKTOP-9M1T24R)| Victim Endpoint   | Windows 10 (22H2) | 192.168.29.226          |
| Kali Linux 2026.2               | Attacker          | Kali Linux        | 192.168.29.163          |

All VMs run on Oracle VirtualBox with Bridged Adapter networking (192.168.29.x subnet).

---

## 🛠️ Tools Used

- **Splunk Enterprise 10.4.2** (self-hosted on Ubuntu, indexer + search head)
- **Splunk Universal Forwarder 10.4.2** on the Windows endpoint (port 9997)
- **Sysmon** from Sysinternals for advanced Windows endpoint logging
- **SwiftOnSecurity Sysmon Config** for comprehensive detection coverage
- **Nmap** for network reconnaissance
- **Hydra** for password brute-forcing (SMB and RDP)
- **Kali Linux** as the offensive security toolkit

---

## 🔍 Investigations

### 📌 1. Nmap Port Scan

**MITRE ATT&CK:** T1046 (Network Service Discovery)

Launched both SYN scans and TCP connect scans from Kali against the Windows endpoint. Discovered 6 open ports (135, 139, 445, 3389, 7680, 49668) and observed a striking detection gap: SYN scans were completely invisible to Sysmon Event ID 3 because they never complete a TCP handshake. Only the TCP connect scan produced logged network events. Documented the difference using the 6-Question Framework and mapped it to MITRE T1046.

📄 [Full Report](./reports/01_Nmap_Reconnaissance_Report.pdf)

---

### 📌 2. SMB Brute-Force Attack

**MITRE ATT&CK:** T1110.001 (Password Guessing), T1021.002 (SMB/Windows Admin Shares), T1078.003 (Valid Accounts: Local)

Used Hydra to brute-force SMB2 on port 445 with a 50-combination dictionary (5 users × 10 passwords). Attack completed in ~4 seconds and successfully cracked `testvictim / Password123`. Splunk captured 48 failed logon events (Windows Event ID 4625, Logon Type 3), all attributed to a single Source_Network_Address (192.168.29.163). Also documented a real-world forwarder stall triggered by the burst of events and the fix. Severity: HIGH. Full credential compromise.

📄 [Full Report](./reports/02_Hydra_SMB_Bruteforce_Investigation_Report.pdf)

---

### 📌 3. RDP Brute-Force Attack (with Multi-Source Correlation)

**MITRE ATT&CK:** T1110.001 (Password Guessing), T1021.001 (Remote Desktop Protocol), T1078.003 (Valid Accounts: Local)

Used Hydra to brute-force RDP on port 3389 with the same wordlist as the SMB exercise. Splunk captured 49 failed logon events, but critically all registered as **Logon Type 3 (Network)** rather than Logon Type 10 (RemoteInteractive). Hydra's RDP module authenticates via NLA, which Windows logs the same way as SMB. To confirm the protocol was RDP, correlated with **Sysmon EID 3** and found 51 network connections from Kali to port 3389. Hydra also confirmed the password for `testvictim` was valid, but Windows denied the RDP session because the account was not in the Remote Desktop Users group. This is a live example of defense-in-depth catching a failed control. Severity: HIGH. Credential compromised, session access denied by group policy.

📄 [Full Report](./reports/03_Hydra_RDP_Bruteforce_Investigation_Report.pdf)

---

## 📊 MITRE ATT&CK Coverage

| Technique                       | ID        | Tactic            | Investigation        |
| ------------------------------- | --------- | ----------------- | -------------------- |
| Network Service Discovery       | T1046     | Discovery         | #1 Nmap              |
| Brute Force                     | T1110     | Credential Access | #2 SMB, #3 RDP       |
| Password Guessing               | T1110.001 | Credential Access | #2 SMB, #3 RDP       |
| Remote Services: SMB/Admin Shares | T1021.002 | Lateral Movement  | #2 SMB               |
| Remote Services: RDP            | T1021.001 | Lateral Movement  | #3 RDP               |
| Valid Accounts: Local Accounts  | T1078.003 | Initial Access    | #2 SMB, #3 RDP       |

**6 techniques across 4 tactics detected and documented.**

---

## 🧠 Skills Demonstrated

- Built a Splunk Enterprise SIEM from scratch on Ubuntu Server (indexer + search head, static IP, receiving port)
- Deployed the Splunk Universal Forwarder on Windows and configured Security, System, Application, and Sysmon inputs
- Configured Sysmon with the SwiftOnSecurity community ruleset for endpoint visibility
- Wrote progressive-narrowing **SPL queries** with `stats`, `where`, `timechart`, and inline `rex` field extractions
- Correlated **two data sources** (Windows Security 4625 events + Sysmon EID 3 network connections) to prove the protocol behind a Logon Type 3 attack
- Identified and documented **detection gaps** honestly (SYN scans invisible to Sysmon EID 3; Logon_Type=10 ineffective for detecting Hydra RDP)
- Diagnosed environment issues in a home lab (Ubuntu clock drift, forwarder stall after brute-force flood, `XmlWinEventLog` vs `WinEventLog` sourcetype behavior)
- Correctly interpreted Windows Event IDs 4624/4625, Logon Types 3/10, Failure Reasons, and NTLM sub-status codes
- Mapped all activity to the MITRE ATT&CK framework
- Wrote professional incident reports using a 6-Question analyst framework, with severity justification, key observations, and remediation recommendations

## 📫 Contact

- **LinkedIn:** https://www.linkedin.com/in/sree-saran-s-s/
- **Email:** sreesaranraja@gmail.com

---

## 📖 How I Built This Lab

Want to build your own? Here's the step-by-step setup process.

<details>
<summary><b>👉 Click to expand the full build guide</b></summary>

### Prerequisites

- A laptop with at least 16 GB RAM
- Oracle VirtualBox installed
- Stable internet connection for downloads
- ~100 GB free disk space (dynamic disks)

### Step 1: Install Ubuntu Server (Splunk Host)

1. Download Ubuntu Server 22.04 LTS ISO from ubuntu.com
2. Create a new VM in VirtualBox: 4 GB RAM, 2 CPUs, 40 GB disk
3. Set Network to **Bridged Adapter**
4. Install Ubuntu Server, create a user account (`ssss` in this lab)
5. Assign a **static IP** by editing `/etc/netplan/00-installer-config.yaml`. This lab uses 192.168.29.97
6. Install OpenSSH server for file transfers: `sudo apt install openssh-server -y`

### Step 2: Install Splunk Enterprise

1. Download the Splunk Enterprise 10.4.2 `.deb` package from splunk.com (free trial converts to Free after 60 days)
2. Install: `sudo dpkg -i splunk-10.4.2-*.deb`
3. Start Splunk: `sudo /opt/splunk/bin/splunk start --accept-license`
4. Set admin password when prompted
5. Enable receiving on port 9997 for forwarders:
   ```
   sudo /opt/splunk/bin/splunk enable listen 9997
   ```
6. Enable boot-start: `sudo /opt/splunk/bin/splunk enable boot-start`
7. Access Splunk Web at `http://<splunk-ip>:8000`
8. Set the time zone to your local zone (Asia/Kolkata in this lab) via Settings → Server settings → General settings

### Step 3: Set Up Windows 10 (Victim)

1. Download Windows 10 ISO using the Media Creation Tool from Microsoft
2. Create a new VM in VirtualBox: 4 GB RAM, 2 CPUs, 50 GB disk
3. Set Network to Bridged Adapter
4. Install Windows 10 (skip the product key, choose Windows 10 Pro)
5. Enable Remote Desktop: Settings → System → Remote Desktop → On
6. Create a dummy target account for brute-force testing:
   ```
   net user testvictim Password123 /add
   ```

### Step 4: Install Sysmon on Windows

1. Download Sysmon from Microsoft Sysinternals
2. Download SwiftOnSecurity's `sysmonconfig-export.xml` from GitHub
3. In elevated PowerShell:
   ```
   .\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
   ```
4. Verify it's running: `Get-Service Sysmon64`

### Step 5: Install Splunk Universal Forwarder on Windows

1. Download the Splunk Universal Forwarder 10.4.2 MSI from splunk.com
2. Install as **Local System**, point the Deployment Server to blank, and set the Receiving Indexer to `<splunk-ip>:9997`
3. Edit `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf` to forward the log channels you care about:
   ```
   [WinEventLog://Security]
   disabled = 0
   index = main

   [WinEventLog://System]
   disabled = 0
   index = main

   [WinEventLog://Application]
   disabled = 0
   index = main

   [WinEventLog://Microsoft-Windows-Sysmon/Operational]
   disabled = 0
   renderXml = true
   index = main
   ```
4. Restart the forwarder:
   ```
   net stop SplunkForwarder
   net start SplunkForwarder
   ```
   > **Note:** with `renderXml = true`, Sysmon logs arrive with sourcetype `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational`. Field extraction inside the XML requires inline `rex`. See the RDP investigation report for a working example.

### Step 6: Set Up Kali Linux (Attacker)

1. Download the Kali Linux VirtualBox image from kali.org
2. Extract the `.7z` and double-click the `.vbox` file to import
3. Set Network to Bridged Adapter
4. Boot and log in with `kali / kali`
5. Verify connectivity to Windows: `ping 192.168.29.226`

### Step 7: Verify the Pipeline

On the Windows VM, generate a test event:
```
Start-Process notepad.exe
```
On Splunk, search:
```
host=DESKTOP-9M1T24R earliest=-5m
```
If events appear, the pipeline is working.

### 🔧 Troubleshooting Tips

- **Ubuntu clock drift** (no NTP by default): Run `date` at the start of every session; if it's off, `sudo date -s "YYYY-MM-DD HH:MM:SS"` then `sudo /opt/splunk/bin/splunk restart`. Long-term fix: `sudo apt install chrony -y && sudo systemctl enable --now chrony`.
- **No events after a heavy attack**: The forwarder can stall after a burst of failed-logon events. On Windows Admin cmd: `net stop SplunkForwarder && net start SplunkForwarder`.
- **Sysmon events not showing under `WinEventLog:...`**: With `renderXml = true` the sourcetype is `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational`. Use that exact string, and extract inner fields with regex: `| rex "<EventID>(?<eid>\d+)</EventID>"`.
- **IPs changing between sessions**: Windows and Kali are on DHCP by default. Check them each session with `ipconfig` and `ip a`. Consider static leases on your router.
- **Splunk not responding**: `sudo /opt/splunk/bin/splunk status`, then `sudo /opt/splunk/bin/splunk restart`.

</details>
