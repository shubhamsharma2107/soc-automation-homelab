# Phase 1: Network Architecture & Infrastructure Provisioning

This phase covers the foundational build of the home lab environment within VirtualBox. The goal was to establish a secure network boundary and deploy a functional Windows Active Directory environment, mirroring a standard corporate infrastructure before layering on the security stack.

## 1. Firewall & Network Boundary (OPNsense)
The first step was establishing the perimeter. I deployed an OPNsense virtual machine to act as the primary router and firewall for the entire lab.

* **Network Adapters:** * **WAN:** Bridged adapter connecting directly to my host internet.
  * **LAN:** Configured as a VirtualBox Internal Network (`10.0.50.0/24`) to completely isolate the lab environment from my personal host machine.
* **Configuration:** Established baseline firewall rules on the WAN interface to block incoming traffic while allowing LAN traffic to route out to the internet for updates and package downloads.

*(Placeholder: Add a screenshot of your OPNsense dashboard or interface assignments here)*

## 2. Active Directory & Core Services (Windows Server 2025)
To simulate a true enterprise environment, I deployed Windows Server 2025 and promoted it to a Domain Controller. This serves as the central management hub for the network.

* **Domain Setup:** Created a new forest and established a local domain (e.g., `soclab.local`).
* **DNS & DHCP:** * Configured the server to act as the primary DNS server for the `10.0.50.0/24` subnet.
  * Authorized a DHCP scope to automatically assign IP addresses to new workstations joining the internal network.
* **Organizational Units (OUs) & Users:** * Structured the directory by creating specific departments (e.g., IT, HR, Finance).
  * Provisioned standard user accounts and assigned them to relevant Security Groups to establish a foundation for Role-Based Access Control (RBAC) and future Group Policy Object (GPO) testing.

*(Placeholder: Add a screenshot of your Active Directory Users and Computers (ADUC) console showing the OUs and users)*

## 3. Endpoint Provisioning & Domain Join
With the domain established, I spun up two employee workstations to generate standard network traffic and serve as target endpoints.

* **Machines:** Deployed Windows 11 and Windows 10 virtual machines.
* **Network Configuration:** Ensured both machines pulled an IP address from the Windows Server DHCP scope and used the server's IP for DNS resolution.
* **Domain Integration:** Successfully joined both the Windows 10 and Windows 11 workstations to the domain and verified login functionality using the standard user accounts created in AD.

*(Placeholder: Add a screenshot showing the System Properties of one of the Windows machines confirming it is joined to the domain)*

## 4. Security Stack Preparation (Ubuntu)
To prepare for the deployment of the SOC tools, I provisioned the underlying Linux infrastructure.

* **Deployment:** Spun up three separate Ubuntu Server virtual machines on the `10.0.50.0/24` network.
* **Purpose:** These machines were given static IP addresses to ensure stability and will host the core security stack:
  * Ubuntu VM 1: Wazuh (SIEM/EDR)
  * Ubuntu VM 2: Shuffle (SOAR)
  * Ubuntu VM 3: TheHive (Incident Management)
* **Configuration:** Performed initial system updates and installed necessary dependencies (like Docker) to prep them for the software installations in Phase 2.

## 5. Attack Infrastructure (Kali Linux)
The final piece of the core infrastructure was setting up the adversary machine.

* **Deployment:** Installed Kali Linux 2025.4 on the internal network.
* **Purpose:** This machine will be used exclusively to simulate threat actor behavior, execute network scans, and launch controlled attacks (like brute-force or malware execution) against the Windows endpoints to validate the SOC's detection capabilities.

*(Placeholder: Add a screenshot showing a successful ping from your Kali Linux machine to your Windows Server to prove network connectivity)*

---
**Next Step:** Proceed to [Phase 2: SIEM Deployment & Log Ingestion](./Phase-2-SIEM.md) to see how Wazuh was configured to monitor this infrastructure.
