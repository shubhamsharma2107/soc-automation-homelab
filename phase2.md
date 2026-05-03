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
    Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.4-1.msi -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='10.0.50.50' 
    ```
    <img width="977" height="411" alt="Screenshot 2026-04-26 174402" src="https://github.com/user-attachments/assets/d03d67ff-1126-42e5-91be-4c4a10e05516" />

4.  **Service Initialization:** We need now to start the Wazuh Service.
    ```powershell
    NET START Wazuh
    ```
5.  On the Wazuh Dashboard we should see our machine online and information about it.

   <img width="1617" height="879" alt="image" src="https://github.com/user-attachments/assets/5f978082-aaa3-459b-8485-ae27fcdd53e6" />

6.  Follow the same steps for windows server and windows 10 machine.

---

## 4. Setting up Forwarding Sysmon Logs to Wazuh

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
4. Verify on Wazuh Dashboard if if wazuh agent is succesffully sending sysmon events.
   
    <img width="1622" height="914" alt="image" src="https://github.com/user-attachments/assets/f00f71e2-1267-451a-941c-1a625a15f76d" />

5. Follow same steps for Windows 10 and Windows Server Machine.

---

## 5. Case Management: TheHive Setup

To elevate this lab from just "detecting" to actual "responding," I deployed **TheHive** to act as the primary SOC ticketing and case management system.

---

### Step 1 — Update System & Install Dependencies

Update your system, install basic dependencies, then install Java:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget gnupg apt-transport-https software-properties-common -y
sudo apt install openjdk-11-jre-headless -y
java -version
```

---

### Step 2 — Install & Configure Apache Cassandra

Add the Cassandra repository and GPG key, then install:

```bash
wget -qO - https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive-keyring.gpg] https://debian.cassandra.apache.org 41x main" | sudo tee /etc/apt/sources.list.d/cassandra.list
sudo apt update
sudo apt install cassandra -y
```

#### Configure the Cluster Name

TheHive expects a specific cluster name. Open the config file:

```bash
sudo nano /etc/cassandra/cassandra.yaml
```

Locate `cluster_name` and update it:

```yaml
cluster_name: 'TheHive'
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

#### Reset & Initialize the Database

Because the cluster name was changed after installation, the existing system data must be cleared to prevent conflicts:

```bash
sudo systemctl stop cassandra
sudo rm -rf /var/lib/cassandra/*
sudo systemctl start cassandra
sudo systemctl enable cassandra
```

---

### Step 3 — Install & Configure Elasticsearch

Add the Elasticsearch repository and GPG key, then install:

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install elasticsearch -y
```

#### Configure Elasticsearch

Open the config file:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Apply the following settings:

```yaml
cluster.name: thehive
node.name: node-1
network.host: 127.0.0.1
http.port: 9200
discovery.type: single-node
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

Start and enable the service:

```bash
sudo systemctl restart elasticsearch
sudo systemctl enable elasticsearch
```

---

### Step 4 — Install & Start TheHive

Download and install the package:

```bash
wget -O /tmp/thehive_5.7.2-1_all.deb https://thehive.download.strangebee.com/5.7/deb/thehive_5.7.2-1_all.deb
sudo apt-get install /tmp/thehive_5.7.2-1_all.deb
```

Start and enable the service:

```bash
sudo systemctl start thehive
sudo systemctl enable thehive
```

---

### Step 5 — Verify All Services

Ensure all three components are actively running without errors:

```bash
sudo systemctl status cassandra elasticsearch thehive
```

---

### Step 6 — Access the Web Interface

Open your browser and navigate to: http://<YOUR_SERVER_IP>:9000

Log in with the default credentials: Username: `admin@thehive.local` & Password:`secret`

<img width="1370" height="1138" alt="image" src="https://github.com/user-attachments/assets/cfc7236c-1972-416c-be04-eb55c618c3bb" />
