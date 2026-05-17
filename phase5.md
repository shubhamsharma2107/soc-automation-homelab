## Phase 5: RDP Brute Force Attack Simulation & Automated Response

### Scenario Overview

In this phase we simulate a realistic external attack scenario:

> An attacker on the internet discovers our WAN IP address, performs reconnaissance
> using Nmap to identify open ports, finds RDP (port 3389) exposed via port
> forwarding, and attempts to brute force their way into the Windows 10 machine.
> Wazuh detects the attack, Shuffle orchestrates the response, and the analyst
> makes the final decision to block the attacker at the firewall.

The full attack and response pipeline:

```
Kali Linux (Attacker)
        ↓
Nmap Port Scan → Discovers RDP open on WAN IP
        ↓
Hydra RDP Brute Force → Multiple failed login attempts
        ↓
Windows Event ID 4625 → Wazuh Rule 60204 fires
        ↓
Custom Rule 100005 triggers after threshold crossed
        ↓
Shuffle receives alert → AbuseIPDB IP lookup
        ↓
Discord → Analyst YES / NO decision
        ↓
    ┌───┴───┐
   YES      NO
    ↓        ↓
OPNsense  TheHive
IP Block  Case Created
```

---

### Lab Setup Changes

Before running the attack, the following changes are required to accurately simulate
an external attacker coming in through the firewall rather than being on the same
internal network as the target machines.

The target network layout for this phase:

```
NAT Network (10.0.2.0/24)
├── Kali Linux        10.0.2.3   ← Attacker (external)
└── OPNsense WAN      10.0.2.15  ← Firewall perimeter
         ↓
    intnet LAN (10.0.50.0/24)
    ├── Windows 10    10.0.50.100 ← Target
    ├── Wazuh         10.0.50.x
    ├── TheHive       10.0.50.x
    └── Shuffle       10.0.50.x
```

This ensures:
- ✅ Kali can reach OPNsense WAN — simulating an internet attacker hitting the firewall
- ❌ Kali cannot directly reach any machines behind the firewall (`10.0.50.0/24`)
- ✅ All traffic from Kali to Windows must pass through OPNsense — giving full firewall visibility and control

---

#### 1. Create a NAT Network in VirtualBox

In VirtualBox go to **Tools → Network → NAT Networks → Create** and configure a
new NAT Network with the `10.0.2.0/24` range:

<img width="1065" height="690" alt="Creating a new NAT Network in VirtualBox with 10.0.2.0/24 range" src="https://github.com/user-attachments/assets/37a65a0d-499a-4878-961d-73d7901249bc" />

---

#### 2. Move OPNsense WAN to NAT Network

In the OPNsense VM settings, change **Adapter 1** from `NAT` to `NAT Network`
and select the newly created network. VirtualBox will automatically assign an IP
from the DHCP range:

<img width="776" height="514" alt="OPNsense Adapter 1 changed from NAT to NAT Network in VirtualBox settings" src="https://github.com/user-attachments/assets/d9829d94-6db2-4417-85ef-a217f28a4d35" />

---

#### 3. Place Kali on the Same NAT Network

In the Kali VM settings, set **Adapter 1** to the same NAT Network. This puts
Kali on the same segment as OPNsense WAN — simulating an external attacker on
the internet targeting the firewall perimeter:

<img width="772" height="515" alt="Kali Linux Adapter 1 placed on the same NAT Network as OPNsense WAN" src="https://github.com/user-attachments/assets/f0b0c0c8-d7f0-40e8-9a1a-f4b6e45a54e3" />

---

#### 4. Verify Network Connectivity

From the OPNsense dashboard, confirm the WAN interface has picked up an IP in
the `10.0.2.0/24` range (`10.0.2.15`). Verify connectivity by pinging Kali
from the OPNsense shell:

<img width="720" height="489" alt="OPNsense WAN showing IP 10.0.2.15 with successful ping to Kali at 10.0.2.3" src="https://github.com/user-attachments/assets/4b3b40e0-aa0b-49d4-80e0-dc90b01c2ae7" />

> **Note:** Pinging from Kali back to OPNsense will not return a response due
> to the default deny ICMP rule on the firewall — this is expected and
> demonstrates good security practice. The one-way ping from OPNsense to Kali
> is sufficient to confirm the setup is correct.

---

#### 5. Enable RDP on Windows 10

On the Windows 10 machine, enable Remote Desktop, allow it through the local
firewall, and disable Network Level Authentication (NLA) for testing:

```powershell
# Enable RDP
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
  -Name "fDenyTSConnections" -Value 0

# Allow RDP through Windows Firewall
netsh advfirewall firewall add rule name="Allow RDP" protocol=TCP dir=in localport=3389 action=allow

# Disable NLA to allow Hydra to attempt authentication
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
  -Name "UserAuthentication" -Value 0

# Verify RDP is listening on port 3389
netstat -an | findstr 3389
```

<img width="857" height="275" alt="PowerShell output confirming RDP is enabled and listening on port 3389" src="https://github.com/user-attachments/assets/8bebfc07-068d-4b03-8f3b-2da6abcbb6c8" />

---

#### 6. Create Port Forwarding Rule on OPNsense

Navigate to **Firewall → NAT → Destination NAT** and create a rule to redirect
inbound RDP traffic hitting the WAN IP to the Windows 10 machine internally:

| Field | Value |
|-------|-------|
| Interface | `WAN` |
| Protocol | `TCP` |
| Destination | `WAN Address` |
| Destination Port | `3389` |
| Redirect Target IP | `10.0.50.100` |
| Redirect Target Port | `3389` |
| Description | `RDP Port Forward → Windows 10` |

<img width="1295" height="1002" alt="OPNsense Destination NAT rule forwarding WAN port 3389 to Windows 10 at 10.0.50.100" src="https://github.com/user-attachments/assets/cac262d7-adc7-48ed-9dd5-6ab232df18e8" />

> Any inbound RDP connection hitting `10.0.2.15:3389` (OPNsense WAN) will now
> be transparently forwarded to the Windows 10 machine at `10.0.50.100:3389`
> on the internal LAN.

---

### Step 1 — Reconnaissance: Nmap Port Scan

With the lab configured, the attack begins with reconnaissance. From Kali,
scan the OPNsense WAN IP to check if RDP is accessible:

**Before the port forward rule — port shows as filtered:**

```bash
nmap -p 3389 10.0.2.15
```

<img width="645" height="508" alt="Nmap scan showing RDP port 3389 as filtered before port forward rule was created" src="https://github.com/user-attachments/assets/da75a93f-af86-4a44-bf2b-6fa4d819eb67" />

**After the port forward rule — port now shows as open:**

```bash
nmap -p 3389 10.0.2.15
```

<img width="645" height="510" alt="Nmap scan showing RDP port 3389 as open after port forward rule was applied" src="https://github.com/user-attachments/assets/0afee1fa-f7b4-4dfe-9305-dbf6a1a6e569" />

> 🔍 From an attacker's perspective, an open RDP port on a public-facing IP
> is a high-value target. This confirms the port forward is working and the
> Windows machine is now reachable through the firewall.

---

### Step 2 — Attack: RDP Brute Force with Hydra

With RDP confirmed open, the attacker launches a brute force attack using Hydra
against the Administrator account:

```bash
hydra -l Administrator -P /usr/share/wordlists/fasttrack.txt rdp://10.0.2.15 -V -f -t 4
```

| Flag | Meaning |
|------|---------|
| `-l Administrator` | Target username |
| `-P fasttrack.txt` | Password wordlist |
| `-V` | Verbose — show each attempt |
| `-f` | Stop on first successful login |
| `-t 4` | 4 parallel threads (RDP max recommended) |

<img width="949" height="490" alt="Hydra RDP brute force running against OPNsense WAN IP targeting Administrator account" src="https://github.com/user-attachments/assets/5b6f2d45-77bd-403c-ab1f-e4ad0055f029" />

---

### Step 3 — Detection: Wazuh Alerts

While the brute force runs, log into the Wazuh dashboard and navigate to
**Threat Intelligence → Threat Hunting → Events**, filtered by the Windows 10 agent.

Wazuh detects the failed login attempts via **Windows Event ID 4625** and the
built-in rule **60204** fires:

<img width="1931" height="977" alt="Wazuh Threat Hunting dashboard showing rule 60204 Multiple Windows Logon Failures firing" src="https://github.com/user-attachments/assets/133911b6-231e-4136-8fbe-b044104cf551" />

Expanding an individual alert reveals detailed forensic information about the attack:

| Field | Value |
|-------|-------|
| **Event ID** | `4625` — Failed Logon |
| **Logon Type** | `3` — Network logon |
| **Target Username** | `Administrator` |
| **Target Computer** | `win10.corp.local` |
| **Attacker Workstation** | `KALI` |
| **Attacker IP** | `10.0.2.3` |
| **Provider** | `Microsoft-Windows-Security-Auditing` |

<img width="966" height="1025" alt="Wazuh alert detail showing Event ID 4625 with attacker IP 10.0.2.3 and target Administrator account" src="https://github.com/user-attachments/assets/0ab97995-02b4-465d-b2a8-a64374550069" />

> **Note:** Rule `60204` is a built-in Wazuh rule that groups failed login
> attempts and triggers after every 8 failures based on Windows Security
> Event ID `4625`. No custom configuration was needed for initial detection —
> Wazuh detected the brute force out of the box.

---

### Step 4 — Custom Detection Rule

While rule `60204` detects individual failure groups, a custom rule is needed
to trigger only after a significant brute force threshold is crossed. This
custom rule is what forwards the alert to Shuffle to kick off the automated
response pipeline.

On the Wazuh manager open the local rules file:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add the following rule:

```xml
<!-- Fires when rule 60204 triggers 5 times within 60 seconds = ~40 failed attempts -->
<rule id="100005" level="12" frequency="5" timeframe="60">
  <if_matched_sid>60204</if_matched_sid>
  <same_field>win.eventdata.ipAddress</same_field>
  <description>RDP Brute Force Attack Detected — Multiple failures from $(win.eventdata.ipAddress) on $(win.system.computer)</description>
  <mitre>
    <id>T1110.001</id>
  </mitre>
  <group>authentication_failures,rdp,brute_force,shuffle_alert</group>
</rule>
```

Save and restart Wazuh to apply the rule:

```bash
sudo systemctl restart wazuh-manager
```

Run Hydra again from Kali and confirm rule **100005** now appears in the
Wazuh dashboard alongside rule 60204:

<img width="1914" height="206" alt="Custom rule 100005 RDP Brute Force firing in Wazuh security events dashboard" src="https://github.com/user-attachments/assets/202c8fb9-ba5d-439a-a4e2-ad1a9123255c" />

> ✅ Rule `100005` firing confirms Wazuh is correctly identifying the brute
> force attack and is ready to forward the alert to Shuffle for automated
> response. The next step is building the Shuffle workflow to handle the
> analyst decision and OPNsense block.
