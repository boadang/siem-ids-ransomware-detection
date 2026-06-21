# SOC Lab: SIEM/IDS Monitoring & Early Ransomware Detection

## 1. Project Overview

This project builds a simulated enterprise security lab (**SOC Lab**) to monitor system activity, centralize logs, and detect suspicious behavior early using **SIEM/IDS** technologies. The main focus is the detection of behaviors associated with **ransomware attacks**, including abnormal process execution, mass file changes, and destructive actions such as shadow copy deletion.

The lab is designed around a segmented enterprise network model with **pfSense**, **Ubuntu Web Server**, **Windows Server**, and a centralized **Wazuh SIEM** node. Logs are collected from both Linux and Windows systems, normalized, and analyzed at the SOC layer to support alerting and future automated response.

---

## 2. Objectives

* Build a small enterprise-style SOC Lab with **network segmentation**
* Deploy a centralized **SIEM/IDS monitoring system** using **Wazuh**
* Collect logs from **Linux**, **Windows**, and **web services**
* Simulate realistic enterprise telemetry using **Active Directory**, **Sysmon**, **Auditd**, and **Nginx JSON logs**
* Prepare detection logic for ransomware-related activity
* Lay the groundwork for **active response** and attack simulation

---

## 3. Lab Architecture

The lab is divided into three main zones:

* **WAN**: external network / Internet access through PNetLab
* **DMZ**: public-facing services such as the Ubuntu Web Server
* **INTERNAL**: internal enterprise network hosting the Windows Server and SOC components

### Network Layout

```text
Internet
   |
[ pfSense Firewall ]
   |-------- WAN
   |-------- DMZ -------- Ubuntu Web Server (10.10.10.10)
   |
   |-------- INTERNAL --- Windows Server / AD / Sysmon
                     \
                      \-- Wazuh SOC Server
```

---

## 4. System Components

### 4.1 pfSense Firewall

The pfSense firewall is used to segment the lab network and control traffic between WAN, DMZ, and INTERNAL zones.

#### Interfaces

| Interface        | Role                                                          | IP / Network     |
| ---------------- | ------------------------------------------------------------- | ---------------- |
| WAN (`em0`)      | Receives DHCP from PNetLab and provides Internet connectivity | DHCP             |
| DMZ (`em1`)      | Isolates public-facing services                               | `10.10.10.1/24`  |
| INTERNAL (`em2`) | Internal enterprise network                                   | `192.168.1.1/24` |

#### Firewall Policy

* **DMZ → INTERNAL**: blocked by default to prevent lateral movement from the public zone into the internal network
* **DMZ → Internet**: allowed for package updates and outbound connectivity
* **DMZ → SOC**: allowed on Wazuh-related ports `1514` and `1515`
* **INTERNAL → Internet**: allowed to download required tools and packages

#### NAT / Port Forwarding

| Public Port | Internal Destination | Purpose                                      |
| ----------- | -------------------- | -------------------------------------------- |
| 80 / 443    | `10.10.10.10`         | Publish the web server                       |
| 2222        | Web Server port 22   | Remote SSH administration from host machine  |
| 8433        | SOC Ubuntu port 443  | Access Wazuh Dashboard from the host machine |

---

### 4.2 Ubuntu Web Server (DMZ)

The Ubuntu Web Server acts as a public-facing service inside the DMZ and also serves as a Linux log source for the SIEM platform.

#### Network Configuration

* **IP**: `10.10.10.10/24`
* **Gateway**: `10.10.10.1`
* **DNS**: `8.8.8.8`

#### Installed Services and Monitoring Components

* **Nginx Web Server**
  Configured to output access logs in **JSON format** for easier parsing by the SIEM pipeline.

* **OpenSSH Server**
  Enabled for remote administration from the host machine.

* **Auditd**
  Used to monitor file system activity and changes related to the web application or critical system files.

* **Wazuh Agent 4.8.0**
  Configured to forward:

  * Linux system logs
  * Auditd events
  * Nginx JSON access logs
    to the centralized Wazuh Manager.

---

### 4.3 Windows Server (INTERNAL)

The Windows Server represents the internal enterprise environment and provides both identity services and Windows telemetry for threat detection.

#### Network Configuration

* **IP**: `192.168.1.2/24`
* **Gateway**: `192.168.1.1`
* **DNS**: `192.168.1.2`

#### Installed Services and Monitoring Components

* **Active Directory Domain Services (AD DS)**
  The server is promoted to a **Domain Controller** for the internal lab domain:

  ```text
  soclab.com
  ```

* **BadBlood**
  Used to generate a realistic Active Directory environment with many OUs, groups, and users for a more practical monitoring scenario.

* **Sysmon v15.21**
  Installed with the **SwiftOnSecurity** configuration to capture rich endpoint telemetry such as:

  * **Event ID 1**: process creation
  * **Event ID 11**: file creation / file rename activity

* **Wazuh Agent 4.8.0**
  Configured to forward Sysmon logs from:

  ```text
  Microsoft-Windows-Sysmon/Operational
  ```

* **HoneyFolder / File Share Target**
  A shared folder is created at:

  ```powershell
  C:\Data_DoAn
  ```

  with permissive access for testing ransomware behavior against decoy data.

---

### 4.4 SOC Monitoring Server (Ubuntu)

The SOC server hosts the centralized Wazuh stack and acts as the SIEM core of the lab.

#### Network Configuration

* **IP**: `192.168.1.10/24`

#### Core Components

* **Wazuh Indexer**
* **Wazuh Manager**
* **Wazuh Dashboard**

These components are deployed in an **all-in-one Docker Compose architecture**. The server receives agent logs, stores them, and provides a web dashboard for monitoring and analysis.

#### Wazuh Ports

| Port         | Purpose                         |
| ------------ | ------------------------------- |
| 1514 TCP/UDP | Log ingestion                   |
| 1515 TCP     | Agent registration / enrollment |
| 443          | Web dashboard access            |

---

## 5. Deployment Steps

## 5.1 Deploy Wazuh on the SOC Server

```bash
# Increase virtual memory for OpenSearch / Wazuh Indexer
sudo sysctl -w vm.max_map_count=262144

# Generate certificates for the single-node deployment
cd ~/wazuh-docker/single-node/
sudo docker-compose -f generate-indexer-certs.yml up -d

# Start the Wazuh stack
sudo docker-compose up -d

# Restart Wazuh Manager after modifying rules or configuration
sudo docker exec -it single-node_wazuh.manager_1 /var/ossec/bin/wazuh-control restart
```

---

## 5.2 Configure Monitoring on the Ubuntu Web Server

### Step 1: Verify static network configuration

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

### Step 2: Enable Auditd

```bash
sudo systemctl start auditd
sudo systemctl enable auditd
```

### Step 3: Install Wazuh Agent 4.7.5

```bash
curl -so wazuh-agent_4.7.5-1_amd64.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb
sudo WAZUH_MANAGER='192.168.1.10' dpkg -i wazuh-agent_4.7.5-1_amd64.deb
```

### Step 4: Configure `ossec.conf` to collect Nginx JSON logs

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add:

```xml
<localfile>
  <location>/var/log/nginx/access.json</location>
  <log_format>json</log_format>
</localfile>
```

### Step 5: Restart the agent

```bash
sudo systemctl daemon-reload
sudo systemctl restart wazuh-agent
```

---

## 5.3 Configure Sysmon and Wazuh Agent on Windows Server

### Step 1: Download and install Sysmon

```powershell
New-Item -ItemType Directory -Force -Path "C:\Temp"
Set-Location -Path "C:\Temp"
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive -Path "Sysmon.zip" -DestinationPath "C:\Temp\Sysmon" -Force
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmonconfig.xml"
CD C:\Temp\Sysmon
.\Sysmon64.exe -i ..\sysmonconfig.xml -accepteula
Get-Service -Name Sysmon64
```

### Step 2: Install Wazuh Agent and point it to the SOC server

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi" -OutFile "C:\wazuh-agent.msi"
Start-Process msiexec.exe -ArgumentList '/i C:\wazuh-agent.msi /q WAZUH_MANAGER="192.168.1.10"' -Wait
```

### Step 3: Edit the agent configuration if needed

```powershell
notepad "C:\Program Files\ossec-agent\ossec.conf"
```

### Step 4: Restart the Wazuh service

```powershell
Restart-Service -Name Wazuh
```

---

## 6. Log Sources Collected by the SIEM

The current lab collects and forwards telemetry from the following sources:

| Source System     | Data Source                             | Purpose                                                |
| ----------------- | --------------------------------------- | ------------------------------------------------------ |
| Ubuntu Web Server | Nginx access logs (JSON)                | Web request monitoring and suspicious access detection |
| Ubuntu Web Server | Auditd                                  | File integrity and Linux activity auditing             |
| Ubuntu Web Server | Linux system logs                       | General operating system monitoring                    |
| Windows Server    | Sysmon                                  | Process, file, and system behavior monitoring          |
| Windows Server    | Windows event telemetry via Wazuh Agent | Endpoint visibility and suspicious activity detection  |

---

## 7. Detection Use Case: Early Ransomware Monitoring

This project is specifically oriented toward building detection logic for ransomware-like activity. The expected detection path includes both **host-level telemetry** and **behavioral correlation**.

### Monitoring Goals

* Detect suspicious process execution on Windows endpoints
* Detect bulk file creation, file rename, or extension changes
* Detect attempts to delete shadow copies or backups
* Detect suspicious file system activity in decoy folders such as `C:\Data_DoAn`
* Centralize these events in Wazuh for alerting and future automated response

### Relevant Telemetry

| Event / Source     | Detection Value                                                                 |
| ------------------ | ------------------------------------------------------------------------------- |
| Sysmon Event ID 1  | Detect unusual process creation, scripting engines, or encryption tools         |
| Sysmon Event ID 11 | Detect file creation / rename patterns associated with mass encryption          |
| Auditd             | Detect Linux-side file modification and sensitive file access                   |
| Nginx logs         | Provide visibility into web access behavior and potential exploitation attempts |

---

## 8. Troubleshooting Log

During deployment, several infrastructure and configuration issues were encountered. The following table summarizes the root causes and resolutions.

| #  | Issue                                                                                       | Root Cause                                                                         | Resolution                                                                                   |
| -- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 1  | Linux host continuously sent DHCPDISCOVER but received no IP                                | Incorrect topology design; direct host-to-host links created network loop behavior | Switched to **static IP addressing** and defined network settings manually in Netplan        |
| 2  | `Network is unreachable` when pinging the Internet                                          | Missing default gateway in `/etc/netplan/00-installer-config.yaml`                 | Added `gateway4: 10.10.10.1`                                                                 |
| 3  | Internet ping failed with 100% packet loss despite local connectivity                       | pfSense WAN was blocking private networks in the nested lab environment            | Disabled **Block private networks and loopback addresses** on WAN                            |
| 4  | `403 Forbidden` while installing packages with APT                                          | Ubuntu mirror source pointed to an inaccessible repository                         | Replaced mirror source with the official Ubuntu repository and refreshed package metadata    |
| 5  | `Could not resolve 'archive.ubuntu.com'`                                                    | Typo in repository hostname and missing DNS configuration                          | Corrected the repository URL and manually set DNS to `8.8.8.8`                               |
| 6  | `Invoke-WebRequest` returned `404 Not Found` when downloading BadBlood                      | Incorrect GitHub URL                                                               | Corrected the download link                                                                  |
| 7  | Windows Server could ping the gateway but not access the Internet                           | Automatic Outbound NAT on pfSense did not correctly translate the INTERNAL subnet  | Switched to **Hybrid Outbound NAT** and created a manual NAT rule for `192.168.1.0/24`       |
| 8  | Windows Client could not join the domain because the domain membership field was greyed out | The VM was running a **Windows Home** edition                                      | Reinstalled the client using **Windows 10/11 Pro**                                           |
| 9  | Wazuh Indexer container kept crashing with `CPU does not support x86-64-v2`                 | PNetLab default CPU model `kvm64` did not expose required CPU instructions         | Changed the VM CPU model to **host / host passthrough**                                      |
| 10 | Ubuntu Wazuh Agent failed with `Invalid server address found: 'MANAGER_IP'`                 | Placeholder `MANAGER_IP` remained in the generated configuration                   | Replaced it with the SOC server IP `192.168.1.10` in `ossec.conf`                            |
| 11 | Windows Wazuh Agent attempted to connect to `192.168.1.2` and never appeared online         | Incorrect SOC server IP was used during agent configuration                        | Updated all relevant `<server>` and `<enrollment>` entries in `ossec.conf` to `192.168.1.10` |
| 12 | `Get-Service` could not find a service named `Sysmon`                                       | The installed service name was actually `Sysmon64`                                 | Used `Get-Service -Name Sysmon64`                                                            |

---

## 9. Current Status

At the current stage, the lab has successfully achieved the following:

* Network segmentation with **pfSense**
* A working **DMZ Web Server** with **Nginx**, **Auditd**, and **Wazuh Agent**
* A working **Windows Server** with **AD DS**, **BadBlood**, **Sysmon**, and **Wazuh Agent**
* A centralized **Wazuh SIEM** stack deployed on Ubuntu using **Docker Compose**
* Log forwarding from Linux and Windows systems into the SOC environment
* A decoy data folder prepared for future ransomware simulation

---

## 10. Next Implementation Phase

The next phase of the project focuses on turning the lab from a log collection environment into an active ransomware detection and response platform.

### Planned Tasks

* Build **custom Wazuh rules** in `local_rules.xml` to detect:

  * **Event ID 1** patterns related to malicious process execution or shadow copy deletion
  * **Event ID 11** patterns related to mass file creation / renaming / extension changes

* Configure **Active Response** on the Wazuh side to:

  * block malicious IP addresses
  * isolate compromised systems
  * trigger automated containment logic

* Execute a controlled **ransomware simulation scenario** by:

  * deleting or tampering with shadow copies
  * encrypting files inside the decoy folder `C:\Data_DoAn`
  * validating whether the SIEM can detect and alert on the attack chain

---

## 11. Suggested Future Improvements

To make the project stronger as a graduation project / GitHub portfolio project, the following improvements should be added:

1. **Screenshots / evidence**

   * Wazuh Dashboard overview
   * Agent registration status
   * Sysmon event visibility
   * Nginx log parsing results
   * Triggered alerts during the ransomware simulation

2. **Detection rule examples**

   * Include the actual `local_rules.xml` rules used for ransomware detection
   * Explain why each rule exists and which event patterns it targets

3. **Attack simulation procedure**

   * Document the exact commands or scripts used to simulate ransomware behavior
   * Separate safe simulation steps from destructive or unsafe actions

4. **Incident response workflow**

   * Describe how an alert moves from detection to triage, investigation, and containment inside the SOC lab

5. **Architecture diagram**

   * Replace the ASCII diagram with a proper network diagram showing pfSense, DMZ, INTERNAL, Wazuh, and log flows

---

## 12. Conclusion

This project establishes a practical SOC Lab for studying enterprise log collection, SIEM deployment, and ransomware-oriented detection engineering. By combining **pfSense segmentation**, **Wazuh**, **Sysmon**, **Auditd**, and **Active Directory telemetry**, the environment provides a realistic foundation for both blue-team monitoring and future security experimentation.

The current implementation already supports centralized monitoring and cross-platform log collection. The next step is to transform that telemetry into **actionable ransomware detections** and **automated defensive responses**.
