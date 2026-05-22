# SOC Automation Home Lab

## 📌 Project Overview

A fully self-contained Security Operations Center built and operated entirely
within VirtualBox — designed to simulate a realistic corporate environment
where real attacks are launched, detected, investigated, and responded to
automatically.

The lab is segmented behind an OPNsense firewall with a dedicated internal
network (`10.0.50.0/24`), hosting a complete security stack: Wazuh for threat
detection, Shuffle for automated response orchestration, and TheHive for
incident case management. Attacks are simulated from a Kali Linux machine
positioned on the WAN side of the firewall — mirroring how a real external
adversary would approach the network.

---

## 🏗️ Architecture & Topology

| Component | Details |
|-----------|---------|
| **Hypervisor** | Oracle VirtualBox |
| **Firewall / Network Boundary** | OPNsense — WAN (`10.0.2.0/24`) → LAN (`10.0.50.0/24`) |
| **SIEM / EDR** | Wazuh |
| **SOAR** | Shuffle |
| **Incident Management** | TheHive |
| **Target Endpoints** | Windows Server 2025, Windows 11, Windows 10 |
| **Attacker Machine** | Kali Linux 2025.4 |

![SOC Automation Lab Topology Diagram](https://github.com/user-attachments/assets/7ef9ad5c-0320-471e-8f38-d0b265d115d8)

![Network Segmentation Overview](https://github.com/user-attachments/assets/394396c2-e71d-4c3b-a4ce-ea13ec449c63)

---

## 🎯 Key Objectives & Skills Demonstrated

**Network Engineering** ---
Designed and configured a bare-metal equivalent network topology using OPNsense
for routing, NAT, port forwarding, and strict firewall rulesets — isolating the
internal lab from the simulated external attacker with full traffic visibility
and control at the perimeter.

**Infrastructure Provisioning** ---
Deployed and managed a multi-OS environment across Windows Server, Windows
10/11, and Ubuntu — accurately reflecting the diverse attack surface of a
real corporate network.

**Threat Detection & Detection Engineering** ---
Deployed Wazuh agents across all target endpoints to ingest system telemetry,
wrote custom detection rules tuned to specific attack techniques, and validated
alert fidelity against live attack traffic.

**SOAR & Automated Incident Response** ---
Engineered an end-to-end automation pipeline in Shuffle that ingests Wazuh
alerts, parses JSON payloads, enriches indicators via VirusTotal, notifies
analysts through Discord, and automatically creates structured cases in TheHive
— with analyst-driven firewall blocking via the OPNsense API.

**Adversary Simulation** ---
Executed controlled attacks from Kali Linux — including credential dumping with
Mimikatz and RDP brute forcing with Hydra — to validate detection rules, test
SOAR playbook coverage, and confirm automated response actions fire correctly
end to end.
  
## 🛠️ Step-by-Step Implementation

For a detailed breakdown, configurations, and screenshots, click on each phase below:

---

| Phase | Description |
|-------|-------------|
| **[Phase 1: Network Architecture & Infrastructure Provisioning](./phase1.md)** | Provision the virtual environment — configure OPNsense as the perimeter firewall, spin up all virtual machines, and establish the core network architecture with isolated LAN and WAN segments. |
| **[Phase 2: Telemetry, SIEM & Case Management](./phase2.md)** | Deploy and configure Wazuh as the SIEM, install Sysmon on Windows endpoints for enriched telemetry, and set up TheHive for structured incident case management. |
| **[Phase 3: Mimikatz Execution & Wazuh Detection](./phase3.md)** | Execute Mimikatz credential dumping on the Windows target, write custom Wazuh detection rules to identify the attack, and verify alerts are firing correctly in the dashboard. |
| **[Phase 4: SOAR Integration & Automated Response](./phase4.md)** | Wire Shuffle, TheHive, VirusTotal, and Discord into a fully automated response pipeline — from alert ingestion through enrichment to analyst notification and case creation. |
| **[Phase 5: RDP Brute Force Simulation & Firewall Automation](./phase5.md)** | Simulate an external RDP brute force attack from Kali Linux, detect it with a custom Wazuh rule, and automate attacker IP blocking at the OPNsense firewall through the Shuffle SOAR pipeline. |
