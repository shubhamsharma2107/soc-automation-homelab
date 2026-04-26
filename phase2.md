# 🛡️ Phase 2: Telemetry, SIEM, and Case Management
## Deploying Wazuh, Sysmon, and TheHive for Incident Response

### 📌 Phase Overview

This phase transitions the lab from basic logging to a fully operational Security Operations Center (SOC) monitoring environment. By integrating **Wazuh SIEM** with high-fidelity **Sysmon** telemetry, we establish deep endpoint visibility. Furthermore, deploying **TheHive** introduces a dedicated case management platform, enabling structured incident response workflows.

---
## 1. Wazuh SIEM Deployment (Ubuntu 22.04)

For the central intelligence hub of this SOC lab, I deployed **Wazuh** on an Ubuntu 22.04 LTS server. Wazuh is an industry-leading, open-source platform that unifies SIEM (Security Information and Event Management) and XDR (Extended Detection and Response). Rather than just acting as a passive log sink, Wazuh provides real-time visibility, active threat hunting, and automated incident response across the environment.

### Core Capabilities of Wazuh:
* **Endpoint Security (EDR):** Utilizes a lightweight, low-resource agent on the Windows machine to continuously monitor for malware, rootkits, and behavioral anomalies.
* **Deep Log Analysis:** Aggregates and parses complex telemetry (like Sysmon) from endpoints and network devices into structured, actionable security alerts.
* **File Integrity Monitoring (FIM):** Tracks unauthorized modifications to critical system files, directories, and registry keys in real-time.
* **Vulnerability Detection:** Automatically cross-references installed endpoint software against global databases to identify known CVEs and insecure configurations.
* **Framework Mapping:** Automatically tags incoming alerts with **MITRE ATT&CK** tactics and techniques, which is crucial for standardizing SOC triage and meeting compliance frameworks (like GDPR and PCI-DSS).

### Installation Workflow:
In one of the Ubuntu VM we dedicated for wazuh we wll do below steps

1.  **System Preparation:**
    ```bash
    sudo apt-get update && sudo apt-get upgrade -y
    ```
2.  **Execute the Installation Assistant:**
    For a streamlined lab architecture, the all-in-one script deploys the Indexer, Server, and Dashboard simultaneously.
    ```bash
    curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash wazuh-install.sh -a
   
    ```
3.  **UI Access:** Upon completion, copy the `admin` credentials provided in the terminal and navigate to `https://<Ubuntu_IP>`. Here the ubuntu ip is the Ubuntu vm ip which you set manually and from analyst workstation win11 you can access wazuh dashboard
<img width="1204" height="713" alt="Screenshot 2026-04-26 131800" src="https://github.com/user-attachments/assets/b15b2c2d-e2e8-45c2-8ef4-9887b6c7fe4a" />

<br> <img width="1740" height="873" alt="Screenshot 2026-04-26 132043" src="https://github.com/user-attachments/assets/36b8ba18-db84-436f-9ec5-4467cfd35db1" /> <br/>


---

## 2. Endpoint Telemetry Enrichment: Sysmon

Standard Windows Event Logging lacks the granularity required for modern threat hunting. **Sysmon** 

### Configuration & Deployment:
1.  **Acquisition:** Download the latest binary from [Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon).
2.  **Tuning:** To minimize log fatigue, I utilized a customized version of the [SwiftOnSecurity configuration](https://github.com/SwiftOnSecurity/sysmon-config), focusing strictly on high-value IOCs (Indicators of Compromise).
3.  **Silent Installation (PowerShell Admin):**
    ```powershell
    .\Sysmon64.exe -i sysmonconfig.xml -accepteula
    ```
4.  **Validation:** Verified event generation via `Event Viewer > Applications and Services Logs > Microsoft > Windows > Sysmon > Operational` (Filtering for Event ID 1: Process Creation).

---

## 3. Wazuh Agent Integration

The Wazuh Agent establishes a secure, encrypted AES channel to forward endpoint logs to the manager.

1.  **Agent Generation:** Navigated to **Wazuh Dashboard > Agents > Deploy new agent**. Configured the payload for a Windows architecture pointing to the Ubuntu server IP.
2.  **Execution:** Ran the provided API command in PowerShell (Run as Administrator) on the target Windows VM.
3.  **Service Initialization:**
    ```powershell
    Start-Service -Name wazuh
    ```

---

## 4. Engineering the Telemetry Pipeline

By default, Wazuh does not monitor the Sysmon event channel. The local configuration must be modified to bridge this gap.

1.  **Modify OSSEC Config:** Open `C:\Program Files (x86)\ossec-agent\ossec.conf` as an Administrator.
2.  **Inject Localfile Block:** Append the following within the `<ossec_config>` module:
    ```xml
    <localfile>
      <location>Microsoft-Windows-Sysmon/Operational</location>
      <log_format>eventchannel</log_format>
    </localfile>
    ```
3.  **Apply Changes:**
    ```powershell
    Restart-Service -Name wazuh
    ```

---

## 5. Case Management: TheHive Setup

To elevate this lab from just "detecting" to actual "responding," I deployed **TheHive** to act as the primary SOC ticketing and case management system.

### Docker Deployment Steps:
1. **Install Prerequisites:** Ensure Docker and Docker Compose are installed on the Linux host.
2. **Compose Configuration:** Created a `docker-compose.yml` file configuring TheHive alongside Cassandra and Elasticsearch (its backend dependencies).
3. **Container Initialization:**
    ```bash
    docker-compose up -d
    ```
4. **Access Verification:** Reached TheHive web interface via `http://<Ubuntu_IP>:9000` to establish the initial administrator account and organization setup.

---

## 6. Verification & SOC Visualization

With the infrastructure online, I verified the telemetry flow within the Wazuh **Security Events** module.

- **Query Executed:** `data.source: "Microsoft-Windows-Sysmon/Operational"`
- **Result:** Successful ingestion of deep system internals, including:
  - Parent-child process relationships
  - Network callbacks
  - File hash (SHA256) logging

*This project is part of my ongoing preparation for the **SC-200: Microsoft Security Operations Analyst** certification, demonstrating practical application of SIEM engineering and incident response tooling.*
