# SOC Automation Home Lab ##

## 📌 Project Overview
I built and deployed a comprehensive, locally hosted home lab to simulate a live corporate network. Built entirely within VirtualBox, this environment is segmented by an OPNsense firewall to secure a dedicated internal network (`10.0.50.0/24`). The primary objective of this project was to establish a functional Security Operations Center (SOC) to detect threats, automate responses, and manage incident ticketing in a realistic, isolated enterprise environment.

## 🏗️ Architecture & Topology
* **Hypervisor:** Oracle VirtualBox
* **Network Boundary:** OPNsense Firewall (WAN to Internet, LAN to `10.0.50.0/24`)
* **Security Stack (Ubuntu Linux):**
  * **SIEM / EDR:** Wazuh 
  * **SOAR:** Shuffle 
  * **Incident Management:** TheHive
* **Target Endpoints:** Windows Server 2025, Windows 11, Windows 10
* **Attacker Machine:** Kali Linux (2025.4)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7ef9ad5c-0320-471e-8f38-d0b265d115d8" />

<img width="616" height="610" alt="image" src="https://github.com/user-attachments/assets/394396c2-e71d-4c3b-a4ce-ea13ec449c63" />


## 🎯 Key Objectives & Skills Gained
* **Network Engineering:** Configured a bare-metal equivalent network topology utilizing OPNsense for routing, NAT, and strict firewall rules to isolate the lab environment.
* **Infrastructure Provisioning:** Deployed and managed a multi-OS environment (Windows Server, Windows 10/11, Ubuntu) to accurately reflect a diverse corporate attack surface.
* **Threat Detection:** Configured Wazuh agents across all target endpoints to ingest system telemetry and trigger alerts based on custom security rules.
* **Automated Incident Response:** Engineered an automation pipeline using Shuffle to parse JSON payloads from Wazuh, enrich the data, and automatically create cases in TheHive.
* **Adversary Simulation:** Utilized Kali Linux to execute controlled attacks against the Windows targets to validate detection engineering and SOAR playbooks.
  
## 🛠️ Step-by-Step Implementation

For a detailed breakdown, configurations, and screenshots, click on the phases below:

* **[Phase 1: Network Architecture & Infrastructure Provisioning](./phase1.md)** - Setting up the firewall, virtual machines, and core network architecture.
* **[Phase 2: Telemetry, SIEM, and Case Management](./phase2.md)** - Installing and configuring Wazuh, Sysmon, and TheHive.
* **[Phase 3: Execute Mimikatz & Verify Wazuh Detections](./phase3.md)** - Deploying Mimikatz, writing custom Wazuh detection rules, and verifying alerts.
* **[Phase 4: SOAR Integration & Automation](./phase4.md)** - Connecting Shuffle, TheHive, VirusTotal, and Discord into a fully automated response pipeline.

