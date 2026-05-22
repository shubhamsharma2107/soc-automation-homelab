## Phase 5: RDP Brute Force Simulation & Firewall Automation

### Scenario Overview

In this phase we simulate a realistic external attack scenario:

> An attacker on the internet discovers our WAN IP address, performs reconnaissance
> using Nmap to identify open ports, finds RDP (port 3389) exposed via port
> forwarding, and attempts to brute force their way into the Windows 10 machine.
> Wazuh detects the attack, Shuffle orchestrates the response, and the attacker
> IP gets blocked at the firewall.

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
Shuffle receives alert → VirusTotal IP lookup
        ↓
Discord → Analyst     
        ↓        
    OPNsense  
    IP Block
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

### 1. Create a NAT Network in VirtualBox

In VirtualBox go to **Tools → Network → NAT Networks → Create** and configure a
new NAT Network with the `10.0.2.0/24` range:

<img width="1065" height="690" alt="Creating a new NAT Network in VirtualBox with 10.0.2.0/24 range" src="https://github.com/user-attachments/assets/37a65a0d-499a-4878-961d-73d7901249bc" />

---

### 2. Move OPNsense WAN to NAT Network

In the OPNsense VM settings, change **Adapter 1** from `NAT` to `NAT Network`
and select the newly created network. VirtualBox will automatically assign an IP
from the DHCP range:

<img width="776" height="514" alt="OPNsense Adapter 1 changed from NAT to NAT Network in VirtualBox settings" src="https://github.com/user-attachments/assets/d9829d94-6db2-4417-85ef-a217f28a4d35" />

---

### 3. Place Kali on the Same NAT Network

In the Kali VM settings, set **Adapter 1** to the same NAT Network. This puts
Kali on the same segment as OPNsense WAN — simulating an external attacker on
the internet targeting the firewall perimeter:

<img width="772" height="515" alt="Kali Linux Adapter 1 placed on the same NAT Network as OPNsense WAN" src="https://github.com/user-attachments/assets/f0b0c0c8-d7f0-40e8-9a1a-f4b6e45a54e3" />

---

### 4. Verify Network Connectivity

From the OPNsense dashboard, confirm the WAN interface has picked up an IP in
the `10.0.2.0/24` range (`10.0.2.15`). Verify connectivity by pinging Kali
from the OPNsense shell:

<img width="720" height="489" alt="OPNsense WAN showing IP 10.0.2.15 with successful ping to Kali at 10.0.2.3" src="https://github.com/user-attachments/assets/4b3b40e0-aa0b-49d4-80e0-dc90b01c2ae7" />

> **Note:** Pinging from Kali back to OPNsense will not return a response due
> to the default deny ICMP rule on the firewall — this is expected and
> demonstrates good security practice. The one-way ping from OPNsense to Kali
> is sufficient to confirm the setup is correct.

---

### 5. Enable RDP on Windows 10

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

### 6. Configure Port Forwarding and Firewall Rules on OPNsense

This step exposes the internal Windows 10 machine's RDP port through the OPNsense firewall, allowing simulated external RDP brute force attacks to reach the target.

---

#### 6.1 Create a Destination NAT Rule (Port Forward)

Navigate to **Firewall → NAT → Destination NAT** and click **Add** to create a new rule.

| Field | Value |
|-------|-------|
| Interface | `WAN` |
| Protocol | `TCP` |
| Destination | `WAN Address` |
| Destination Port Range | `3389` |
| Redirect Target IP | `10.0.50.100` |
| Redirect Target Port | `3389` |
| Description | `RDP Port Forward → Windows 10` |
| Firewall Rule | `Manual` |

> **Why Manual?** Choosing manual rule creation gives you full control over the firewall rule that permits the forwarded traffic — rather than letting OPNsense auto-generate a permissive rule you cannot fine-tune.

Click **Save** then **Apply Changes**.

<img width="1789" height="1055" alt="image" src="https://github.com/user-attachments/assets/9e1712dc-0677-4dd4-994c-8cffdac4ec41" />

---

#### 6.2 Create the WAN Firewall Rule

Navigate to **Firewall → Rules → WAN** and click **Add** to create the matching inbound rule that permits the forwarded RDP traffic.

| Field | Value |
|-------|-------|
| Action | `Pass` |
| Interface | `WAN` |
| Direction | `in` |
| Protocol | `TCP` |
| Source | `any` |
| Destination | `WAN Address` |
| Destination Port | `3389` |
| Description | `Allow inbound RDP to Windows 10 (Port Forward)` |

Click **Save** then **Apply Changes**.

<img width="1790" height="1057" alt="image" src="https://github.com/user-attachments/assets/febc4a1e-7f08-4a32-a3a3-7fb971919e05" />

---

#### 6.3 Disable Reply-To

Still on the same WAN rule, click **Show Advanced Options** and locate the **Reply-To** setting — **disable it**.

<img width="1563" height="366" alt="image" src="https://github.com/user-attachments/assets/0bacbe9c-14fa-47a5-a2e6-553585984c45" />

> **Why disable Reply-To?** By default, OPNsense uses Reply-To to force return
> traffic back through the WAN gateway. In a lab environment with NAT port
> forwarding, this causes asymmetric routing — response packets take a different
> path than expected, breaking the RDP session. Disabling it ensures return
> traffic follows the normal routing table instead.

Click **Save** then **Apply Changes**.

> Any inbound RDP connection hitting the OPNsense WAN interface on port `3389`
> will now be **transparently forwarded** to the Windows 10 machine at
> `10.0.50.100:3389` on the internal LAN segment.

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

---

### Building the Shuffle Workflow

#### Step 1 — Create the Workflow

In the Shuffle dashboard click **Workflows → New Workflow** and configure:

| Field | Value |
|-------|-------|
| Name | `RDP-BruteForce-Response` |
| Description | `Automated RDP brute force detection and analyst-driven IP blocking` |

<img width="678" height="477" alt="New RDP-BruteForce-Response workflow created in Shuffle dashboard" src="https://github.com/user-attachments/assets/6e2842ab-e147-4aa6-9297-de0ff67e7f08" />

---

#### Step 2 — Build the Initial Node Chain

Add the first three nodes and connect them in sequence on the workflow canvas:

1. **Webhook** — entry point that receives Wazuh alerts
2. **Repeat Back to Me** — extracts the attacker IP from the alert payload
3. **HTTP node** (renamed to `VirusTotal`) — queries VirusTotal for IP reputation

> For detailed steps on configuring the HTTP node as a VirusTotal integration
> refer to the [Phase 4 setup steps](./Phase-4-SOAR.md). The configuration here
> is identical — just rename the node to `VirusTotal`.

Once all three nodes are connected and saved the workflow canvas should look
like this:

<img width="1788" height="960" alt="Shuffle workflow showing Webhook, Repeat Back to Me, and VirusTotal HTTP nodes connected in sequence" src="https://github.com/user-attachments/assets/1ccdfef8-9679-4284-9051-82e01a602afc" />

---

#### Step 3 — Configure Each Node

##### 3a — Activate the Webhook & Connect Wazuh

Click the **Webhook** node and hit **Start Webhook** to activate it. Copy
the generated webhook URL:

<img width="1790" height="960" alt="Webhook node activated in Shuffle showing the generated webhook URL to copy" src="https://github.com/user-attachments/assets/644ca0cf-6f2f-4eda-90e3-119de472ede1" />

On the Wazuh manager open `ossec.conf` and add a new integration block for
this workflow alongside the existing Mimikatz integration:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

```xml
<!-- RDP Brute Force → Shuffle Webhook -->
<integration>
  <name>shuffle</name>
  <hook_url>http://<SHUFFLE_IP>:3001/api/v1/hooks/<YOUR_WEBHOOK_ID></hook_url>
  <rule_id>100005</rule_id>
  <alert_format>json</alert_format>
</integration>
```

<img width="823" height="584" alt="Shuffle integration block added to Wazuh ossec.conf for rule 100005" src="https://github.com/user-attachments/assets/17985f4b-2eba-457b-80da-745333f70f10" />

Restart Wazuh to apply the changes:

```bash
sudo systemctl restart wazuh-manager
```

---

##### 3b — Configure the Repeat Back to Me Node (Extract Attacker IP)

Click the **Repeat Back to Me** node and paste the following into the
**Call** field to extract the attacker IP directly from the Wazuh alert payload:

```
$exec.text.win.eventdata.ipAddress
```

<img width="1790" height="958" alt="Repeat Back to Me node configured with ipAddress field to extract attacker IP from alert" src="https://github.com/user-attachments/assets/01b07db7-5a5a-46fd-a51e-2d28e42a06cf" />

---

##### 3c — Configure the VirusTotal HTTP Node

Click the **VirusTotal** HTTP node and set:

| Field | Value |
|-------|-------|
| Action | `GET` |
| URL | `https://www.virustotal.com/api/v3/ip_addresses/$exec.text.win.eventdata.ipAddress` |

Under **Headers** add:

| Header | Value |
|--------|-------|
| `x-apikey` | `YOUR_VIRUSTOTAL_API_KEY` |

<img width="1790" height="959" alt="VirusTotal HTTP node configured with IP reputation API URL using extracted attacker IP variable" src="https://github.com/user-attachments/assets/e8760493-8919-48d5-9af2-1ada8713b466" />

---

#### Step 4 — Verify the Pipeline is Working

Save the workflow and run Hydra from Kali to trigger rule `100005` and confirm
data is flowing through all three nodes correctly:

```bash
hydra -l Administrator -P /usr/share/wordlists/fasttrack.txt rdp://10.0.2.15 -V -t 4
```

In Shuffle click **Show Execution Results** on each node and verify:

- ✅ **Webhook** — Wazuh alert JSON payload received
- ✅ **Repeat Back to Me** — Attacker IP `10.0.2.3` extracted cleanly
- ✅ **VirusTotal** — `200` status returned with IP reputation data

<img width="1791" height="962" alt="Shuffle workflow execution results showing successful data flow through all three nodes with 200 status from VirusTotal" src="https://github.com/user-attachments/assets/f29604eb-ce54-473d-ad4a-e83d689c26ec" />

> **Note:** Since `10.0.2.3` is a private/internal IP address, VirusTotal will
> return a clean or inconclusive result — this is expected. In a real-world
> scenario where the attacker is coming from a public IP, VirusTotal would
> return meaningful threat intelligence including malicious detection counts,
> known associations, and geographic data. The integration is working correctly
> regardless of the result for private IPs.

---

### Step 5 — Discord Notification Node

Add a **Discord** node to send the enriched alert to the analyst:

1. Add a **Discord** node to the canvas
2. Connect **VirusTotal → Discord_Alert**
3. Rename it `Discord_Alert`
4. Copy your Discord webhook URL from the Discord server and paste it into
   the URL field — refer to [Phase 4](./Phase-4-SOAR.md) for detailed steps
   on setting up the Discord webhook if needed

<img width="1456" height="958" alt="Discord node added and configured in the Shuffle workflow connected after VirusTotal" src="https://github.com/user-attachments/assets/ea6ad8fd-1984-4da7-aecc-8ee0272bb954" />

Paste the following into the **Body** field:

```json
{
  "content": "🚨 **RDP Brute Force Detected**\n\n🖥️ **Target Host:** $exec.all_fields.data.win.system.computer\n🌐 **Attacker IP:** $exec.text.win.eventdata.ipAddress\n🔁 **Rule Fired Times:** $exec.all_fields.rule.firedtimes\n🦠 **Malicious Detections:** $virustotal.body.data.attributes.last_analysis_stats.malicious\n⏰ **Time:** $exec.timestamp\n\n🛡️ **Action: Attacker IP will be added to the OPNsense firewall blocklist**"
}
```

Click **Test Action** to verify the Discord message is delivered successfully:

<img width="893" height="312" alt="Discord test action returning success with RDP brute force alert message delivered to soc-alerts channel" src="https://github.com/user-attachments/assets/57be1ef8-2875-4014-b84a-73d96244bb27" />

---

### Step 6 — Block IP at OPNsense Firewall

#### 6.1 Create the Blocklist Alias

Before configuring the Shuffle node, a dedicated alias needs to be created in
OPNsense to store the attacker IPs that will be blocked. This alias acts as a
dynamic blocklist that the firewall rule references.

In OPNsense navigate to **Firewall → Aliases → + Add** and configure:

| Field | Value |
|-------|-------|
| Name | `sblocklist` |
| Type | `Host(s)` |
| Description | `IPs blocked via Shuffle SOAR automation` |

<img width="1516" height="780" alt="OPNsense alias sblocklist created as Host type to store attacker IPs from Shuffle" src="https://github.com/user-attachments/assets/ab951d85-3958-46ac-9468-b302b30395a5" />

---

#### 6.2 Configure the OPNsense Node in Shuffle

The native OPNsense app in Shuffle was not functioning as expected so an
**HTTP node** is used instead to make direct API calls — the same workaround
used for VirusTotal in Phase 4.

Add an **HTTP** node to the canvas and configure it:

1. Add an **HTTP** node
2. Connect **Discord_Alert → OPNsense_Block**
3. Rename it `OPNsense_Block`
4. Configure it:

| Field | Value |
|-------|-------|
| Action | `POST` |
| URL | `https://10.0.50.1/api/firewall/alias_util/add/sblocklist` |

> Replace `10.0.50.1` with your actual OPNsense LAN IP if different.

Set the **Body** to:

```json
{
  "address": "$exec.text.win.eventdata.ipAddress"
}
```

<img width="1788" height="1055" alt="HTTP node configured as OPNsense block with POST request to alias_util API endpoint" src="https://github.com/user-attachments/assets/3c0932bc-36c8-47dc-8dfe-6cc7081ca7db" />

---

#### 6.3 Generate the OPNsense API Key

The OPNsense API requires authentication. To generate an API key:

1. In OPNsense go to **System → Access → Users**
2. Click on your admin user
3. Scroll down to **API Keys → + Add**
4. OPNsense will download a file containing your **Key** and **Secret**

<img width="1793" height="583" alt="OPNsense API key generation page under System Access Users" src="https://github.com/user-attachments/assets/f96b7c44-c823-480a-8f14-76e74e5312c5" />

Open the downloaded file and in the Shuffle HTTP node set:

| Field | Value |
|-------|-------|
| Username | `key` value from the downloaded file |
| Password | `secret` value from the downloaded file |

<img width="1784" height="1048" alt="OPNsense API key and secret entered as username and password in the Shuffle HTTP node" src="https://github.com/user-attachments/assets/11103652-a673-4c2a-886a-4aa125547508" />

---

#### 6.4 Test & Verify the Block

Save the workflow and click **Test Action** on the OPNsense node. A successful
response will return a result similar to:

```json
{
  "result": "done"
}
```

<img width="904" height="648" alt="OPNsense API returning done result confirming attacker IP was successfully added to sblocklist" src="https://github.com/user-attachments/assets/050d4ac1-a869-418a-b23e-8b8743a1e954" />

To confirm the IP was actually added, navigate to **Firewall → Diagnostics →
Aliases** and look for `sblocklist` — the attacker IP `10.0.2.3` should now
appear in the list:

<img width="1789" height="654" alt="OPNsense Aliases diagnostics showing attacker IP 10.0.2.3 successfully added to sblocklist" src="https://github.com/user-attachments/assets/19f8ebb7-9978-40e7-9a83-c5c15c3b1f6d" />

> ✅ The IP appearing in the `sblocklist` alias confirms the full Shuffle →
> OPNsense API integration is working correctly. Any firewall rule referencing
> this alias will now automatically block all traffic from the attacker IP at
> the perimeter — without any manual intervention required.

---

### Step 7 — Create the Floating Firewall Block Rule

With the alias in place, the final step is creating a firewall rule that
actually **drops traffic** from any IP in the `sblocklist` alias.

Navigate to **Firewall → Rules → Floating** and click **Add** to create the rule:

| Field | Value |
|-------|-------|
| Action | `Block` |
| Direction | `in` |
| Protocol | `any` |
| Source | `sblocklist` (the alias) |
| Destination | `any` |
| Description | `sblocklist drop rule — block IPs added by Shuffle SOAR` |

Click **Save** then **Apply Changes**.

<img width="1791" height="1061" alt="image" src="https://github.com/user-attachments/assets/32389cbc-17ce-4c33-a992-cea362cbf601" />

> **Why a Floating Rule?** Floating rules sit at the top of the rule evaluation
> order and are processed before interface-specific rules — both manually created
> and auto-generated ones. This guarantees the block takes effect immediately
> regardless of other rules in the ruleset, making it the correct choice for a
> dynamic SOAR-driven blocklist.

---

### Step 8 — End-to-End Verification

With all components in place, run Hydra from Kali one final time to validate
the full pipeline fires end to end:

```bash
hydra -l Administrator -P /usr/share/wordlists/fasttrack.txt rdp://10.0.2.15 -V -t 4
```

The side-by-side view below shows Hydra attempting the brute force on the left
while OPNsense drops the packets in real time on the right — confirming the
automated block is working:

<img width="2551" height="1314" alt="image" src="https://github.com/user-attachments/assets/a8a83afc-9042-42ad-bcd4-dc9e98cb6c23" />

> To validate the full pipeline with a fresh IP, change the Kali Linux IP
> address and rerun the attack — confirming the detection, enrichment,
> notification, and blocking chain works from start to finish on a new attacker.

---

<div align="center">

[← Previous: Phase 4 — SOAR Integration & Automated Response](./phase4.md) &nbsp;&nbsp;&nbsp;&nbsp; [🏠 Back to Main README](https://github.com/shubhamsharma2107/soc-automation-homelab)

</div>

