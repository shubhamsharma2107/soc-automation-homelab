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
* **Framework Mapping:** Automatically tags incoming alerts with **MITRE ATT&CK** tactics and techniques, which is crucial for standardizing SOC triage and meeting compliance frameworks.

### Installation Workflow:
On the dedicated Ubuntu VM, the following steps were executed to provision the Wazuh infrastructure:

1.  **System Preparation:**
    ```bash
    sudo apt-get update && sudo apt-get upgrade -y
    ```

2.  **Execute the Installation Assistant:**
    For a streamlined lab architecture, the all-in-one script deploys the Indexer, Server, and Dashboard simultaneously.
    ```bash
    curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash wazuh-install.sh -a
    ```

3.  **UI Access:** Upon completion, I secured the `admin` credentials provided in the terminal. The dashboard is now accessible from the Windows 11 Analyst Workstation by navigating to `https://<Ubuntu_IP>`.

<img width="1204" height="713" alt="Wazuh Terminal Credentials" src="https://github.com/user-attachments/assets/b15b2c2d-e2e8-45c2-8ef4-9887b6c7fe4a" />
<br>
<img width="1740" height="873" alt="Wazuh Dashboard UI" src="https://github.com/user-attachments/assets/36b8ba18-db84-436f-9ec5-4467cfd35db1" />

---

## 2. Endpoint Telemetry Enrichment: Sysmon

Standard Windows Event Logging lacks the granularity required for modern threat hunting. **Sysmon (System Monitor)** bridges this gap by installing as a system service and device driver to log deep system activity, such as process creations, network connections, and changes to file creation time.

### Configuration & Deployment:

1.  Downloaded the sysmon from [Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon).
2.  To minimize log fatigue, I utilized a customized version of the [SwiftOnSecurity configuration](https://github.com/SwiftOnSecurity/sysmon-config), focusing strictly on high-value IOCs (Indicators of Compromise).
    <br>
    <img width="905" height="585" alt="Downloading Sysmon Config" src="https://github.com/user-attachments/assets/6c8d535c-56d8-4a88-b721-d0813ccdb912" />

3.  Extracted the downloaded Sysmon directory and placed the XML configuration file inside the same folder for streamlined execution.
    <br>
    <img width="859" height="586" alt="Sysmon Directory Setup" src="https://github.com/user-attachments/assets/2ba72b8d-3213-4b2b-800b-0dc1e8a8dfc0" />

4.  Opened the Powershell as Admin, Navigated to Sysmon folder and Execute the installation.
    ```powershell
    .\Sysmon64.exe -i sysmonconfig.xml -accepteula
    ```
    <img width="976" height="507" alt="Sysmon Install Command" src="https://github.com/user-attachments/assets/300c40bf-7e57-4a3f-9f24-59bb6cf2461f" />

5. Lets Verify Sysmon is installed on the System.
   ```powershell
    Get-Service Sysmon64
    ```
    <img width="977" height="507" alt="image" src="https://github.com/user-attachments/assets/ce5062fd-3538-4159-ae71-46795e5c1a76" />

6.  **Validation:** Verified event generation via `Event Viewer > Applications and Services Logs > Microsoft > Windows > Sysmon > Operational` (Filtering for Event ID 1: Process Creation).

    <img width="1676" height="876" alt="image" src="https://github.com/user-attachments/assets/08f6c548-b923-4872-b18f-b6adb6ec79e9" />

## 3. Wazuh Agent Integration

The Wazuh Agent establishes a secure, encrypted AES channel to forward endpoint logs to the manager.

1.  **Agent Generation:** Navigated to **Wazuh Dashboard > Agents > Deploy new agent**. Configured the payload for a Windows machine and entered the IP of ubuntu machine where wazuh is installed in Server Address.

    <br><img width="1606" height="703" alt="2026-04-26_17-41" src="https://github.com/user-attachments/assets/d2c2ab2e-79e1-4b77-8629-c9e3eec982e6" /></br>

    <br><img width="1449" height="643" alt="2026-04-26_17-39" src="https://github.com/user-attachments/assets/b6b51858-338f-437b-a98c-3053f31e49c8" /></br>
    
2.  **Execution:** Ran the command in PowerShell (Run as Administrator) on the target Windows VM to download and install the agent.
    ```powershell
    
    <img width="977" height="411" alt="Screenshot 2026-04-26 174402" src="https://github.com/user-attachments/assets/d03d67ff-1126-42e5-91be-4c4a10e05516" />

4.  **Service Initialization:** We need now to start the Wazuh Service.
    ```powershell
    NET START Wazuh
    ```
Follow the same steps for windows server and windows 10 machine.

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
    <img width="1621" height="917" alt="2026-04-26_17-58" src="https://github.com/user-attachments/assets/5957f703-bebd-400f-b514-3871bdaa46df" />

3.  **Apply Changes:** Open Powershell as Admin and run the command.
    ```powershell
    Restart-Service -Name wazuh
    ```
4. Verify on Wazuh Dashboard if if wazuh agenbt is usccesffully sending sysmon events.



5. Follow same stepos for Windows 10 and Windows Server Machine.

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
