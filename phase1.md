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

<img width="784" height="472" alt="Screenshot 2026-04-26 115001" src="https://github.com/user-attachments/assets/a0bf9fbd-9fb7-4011-bcdb-c53d08ba9879" />

* **Directory Structure (RBAC):**
    * Created standard OUs: `IT`, `HR`, `Sales` and `Finance`.
    * Provisioned standard user accounts and assigned them to role-based Security Groups for future GPO and RBAC testing.

<img width="784" height="472" alt="Screenshot 2026-04-26 114836" src="https://github.com/user-attachments/assets/7d8d1c10-ef5c-464b-866a-a647e8696b09" />

---

## 💻 3. Endpoint Provisioning & Domain Integration

Two Windows workstations were deployed to represent standard employee endpoints. These will serve as traffic generators and targets during attack simulations.

* **Endpoints Deployed:**
    * **`WIN-11-PRO`** *(Modern Enterprise Workstation)*: The designated analyst machine used to manage the lab infrastructure and review alerts.
    * **`WIN-10-PRO`** *(Legacy Endpoint)*: The designated target environment used exclusively for executing controlled attacks and generating security event logs.
* **Configuration:** Both machines successfully received IPs via DHCP and use the DC for DNS resolution.
* **Domain Join:** Both machines successfully joined `corp.local`. Standard domain user logins were verified.

<img width="744" height="554" alt="Screenshot 2026-04-26 120125" src="https://github.com/user-attachments/assets/68724a5c-e361-4b20-9e4e-e5b7b857bc19" />
<img width="744" height="554" alt="Screenshot 2026-04-26 120213" src="https://github.com/user-attachments/assets/9ad18539-e6fb-4197-90fd-97e665357627" />

---

## 🐧 4. Security Stack Infrastructure — Ubuntu Server

Three dedicated Ubuntu VMs were provisioned to host the Phase 2 SOC tooling. Service isolation simplifies troubleshooting and mirrors real-world deployment.

| VM | Assigned Role | Purpose |
| :--- | :--- | :--- |
| **Ubuntu VM 1** | `Wazuh` | SIEM / EDR — log aggregation, threat detection, agent telemetry. |
| **Ubuntu VM 2** | `Shuffle` | SOAR — automated response playbooks and alert orchestration. |
| **Ubuntu VM 3** | `TheHive` | Incident Management — case tracking and analyst workflow. |

* **Setup:** Assigned static IPs within `10.0.50.0/24`. Completed initial OS hardening, updates, and firewall config.

---

## ⚔️ 5. Attack Infrastructure — Kali Linux

The adversary machine provides a controlled platform for simulating threat actor behavior. **Assume Breach Scenario:** In this simulation, we are operating under the assumption that the attacker has already infiltrated the environment, connected a rogue device, received a DHCP IP lease, and is able to communicate laterally with other machines on the network.

* **Deployment:** `Kali Linux 2025.4`
* **Network:** Connected directly to the internal `10.0.50.0/24` subnet for unrestricted lateral visibility.
* **Purpose:** Offensive simulation scoped to validate SOC detection capabilities.
    * *Reconnaissance:* `nmap`
    * *Credential Attacks:* `hydra`, `medusa`
    * *Execution:* Controlled malware execution against Windows endpoints.

<img width="661" height="517" alt="image" src="https://github.com/user-attachments/assets/d495d83e-7ae8-42ef-b7b9-32404320650d" />

---

> **📝 Architectural Note:** > All virtual machines deployed in VirtualBox have their network adapters set strictly to the `Internal Network`. All traffic is forced through the OPNsense firewall, ensuring there is no direct connection to the host machine or the external internet. Additionally, because DNS and DHCP are handled centrally by the Windows Server DC, any new machine introduced to this environment will automatically receive a valid IP and be subject to the lab's centralized routing and security policies.

---
<div align="right">

[Next: Phase 2 — Telemetry, SIEM & Case Management →](https://github.com/shubhamsharma2107/soc-automation-homelab/blob/main/phase2.md)

</div>
