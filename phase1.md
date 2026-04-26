# 🏗️ Phase 1: Network Architecture & Infrastructure Provisioning

> **Objective:** Establish the foundational layer of the home lab—a fully isolated, enterprise-grade virtual environment built inside VirtualBox. This phase constructs a realistic corporate network topology complete with perimeter security, centralized identity management, and endpoint infrastructure, paving the way for the security monitoring stack.

## 🛡️ 1. Firewall & Network Boundary — OPNsense

The first priority is establishing a hard network boundary. OPNsense is deployed as the primary router and stateful firewall, controlling all edge traffic. To ensure stable routing, the firewall was manually assigned a static IP address to serve as the network's default gateway. This address is intentionally excluded from the upcoming DHCP scope to prevent any future IP conflicts.

<img width="722" height="484" alt="image" src="https://github.com/user-attachments/assets/f3ffaf3c-9bf4-4383-a243-2531c57ae66f" />


### 🔌 Interface Configuration
* **`WAN`** `[Bridged Adapter]`: Direct connectivity to the host machine's internet uplink.
  
  <img width="549" height="162" alt="image" src="https://github.com/user-attachments/assets/46471464-d36b-48fe-b87d-2da091b62a40" />

* **`LAN`** `[Internal Network]`: Fully isolated segment bound to `10.0.50.0/24` with no direct path to the host OS.

  <img width="553" height="167" alt="image" src="https://github.com/user-attachments/assets/5c006fb4-0c64-4add-8ede-ca6792c96631" />

### 🚦 LAN Firewall Policy: Strict Egress Filtering
To enforce a secure network boundary, the OPNsense LAN interface was configured with a top-down, least-privilege ruleset. By restricting outbound access, we simulate a hardened corporate environment:

* **✅ Permitted Outbound Services:** DNS (Port 53), HTTP/HTTPS (Ports 80/443), NTP (Port 123), and ICMP (Ping).
* **❌ Default Deny (Catch-All):** A final drop rule blocks all other IPv4/IPv6 traffic. This ensures that rogue services, peer-to-peer traffic, or potential malware deployed during later lab phases cannot communicate outside the designated network.
<img width="1734" height="396" alt="image" src="https://github.com/user-attachments/assets/5ecd50d6-020a-4a62-af99-59c2b2e86a5c" />

---

## 🏢 2. Active Directory & Core Services — Windows Server 2025

To accurately replicate an enterprise environment, Windows Server 2025 was deployed as the Domain Controller (DC). It acts as the authoritative hub for authentication, DNS resolution, and IP management.

* **Domain:** `corp.local`
* **DNS & DHCP:** * Configured as the primary DNS resolver for the `10.0.50.0/24` subnet.
    * DHCP scope authorized to auto-issue IP leases to joining workstations.
 


* **Directory Structure (RBAC):**
    * Created standard OUs: `IT`, `HR`, and `Finance`.
    * Provisioned standard user accounts and assigned them to role-based Security Groups for future GPO and RBAC testing.

📸 *[Screenshot: ADUC console displaying OU hierarchy and provisioned users]*

---

## 💻 3. Endpoint Provisioning & Domain Integration

Two Windows workstations were deployed to represent standard employee endpoints. These will serve as traffic generators and targets during attack simulations.

* **Endpoints Deployed:**
    * `WIN-11-PRO` (Modern enterprise workstation)
    * `WIN-10-PRO` (Legacy endpoint)
* **Configuration:** Both machines successfully received IPs via DHCP and use the DC for DNS resolution.
* **Domain Join:** Both machines successfully joined `soclab.local`. Standard domain user logins were verified.

📸 *[Screenshot: System Properties panel confirming domain membership under soclab.local]*

---

## 🐧 4. Security Stack Infrastructure — Ubuntu Server

Three dedicated Ubuntu Server VMs were provisioned to host the Phase 2 SOC tooling. Service isolation simplifies troubleshooting and mirrors real-world deployment.

| VM | Assigned Role | Purpose |
| :--- | :--- | :--- |
| **Ubuntu VM 1** | `Wazuh` | SIEM / EDR — log aggregation, threat detection, agent telemetry. |
| **Ubuntu VM 2** | `Shuffle` | SOAR — automated response playbooks and alert orchestration. |
| **Ubuntu VM 3** | `TheHive` | Incident Management — case tracking and analyst workflow. |

* **Setup:** Assigned static IPs within `10.0.50.0/24`. Completed initial OS hardening, updates, firewall config, and dependency installs (e.g., Docker).

---

## ⚔️ 5. Attack Infrastructure — Kali Linux

The adversary machine provides a controlled platform for simulating threat actor behavior.

* **Deployment:** `Kali Linux 2025.4`
* **Network:** Connected directly to the internal `10.0.50.0/24` subnet for unrestricted lateral visibility.
* **Purpose:** Offensive simulation scoped to validate SOC detection capabilities.
    * *Reconnaissance:* `nmap`
    * *Credential Attacks:* `hydra`, `medusa`
    * *Execution:* Controlled malware execution against Windows endpoints.

📸 *[Screenshot: Successful ICMP ping from Kali Linux to Windows Server]*

---

## 🏁 Phase 1 Summary & Next Steps

### 📋 Checklist
- [x] Provision OPNsense firewall and establish `10.0.50.0/24` subnet.
- [x] Deploy Windows Server 2025 DC (`soclab.local`).
- [x] Configure DNS and DHCP scopes.
- [x] Provision and domain-join Windows 10/11 endpoints.
- [x] Setup dedicated Ubuntu VMs for the SOC stack.
- [x] Deploy Kali Linux for offensive simulations.

**➡️ Next Up:** Phase 2 covers the deployment and integration of the SIEM, SOAR, and incident management platforms onto the Ubuntu infrastructure.
