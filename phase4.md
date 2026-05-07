## Phase 4: SOAR Automation with Shuffle & Discord Notifications

With detection and case management in place, the final phase focuses on **automation and alerting**. We will deploy **Shuffle** as our SOAR (Security Orchestration, Automation and Response) platform to build an automated workflow that:

1. Receives Mimikatz alerts from Wazuh
2. Extracts the file hash and queries it against **VirusTotal**
3. Creates a case in **TheHive**
4. Notifies the analyst via **Discord**

---

### Step 1 — Install Shuffle

On the third Ubuntu machine, install Docker and clone the Shuffle repository:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose git

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Clone the Shuffle repository
git clone https://github.com/Shuffle/Shuffle
cd Shuffle

# Set correct permissions for the database
sudo chown -R 1000:1000 shuffle-database

# Disable swap (required for Elasticsearch inside Shuffle)
sudo swapoff -a

# Start all Shuffle services
sudo docker compose up -d
```

Once the containers are running, access the Shuffle web interface at:

```
http://<YOUR_SERVER_IP>:3001
```

On first launch, create your admin account with a username and password of your choice. Once your credentials are set up, you will land on the Shuffle dashboard:

<img width="1526" height="774" alt="Shuffle dashboard after successful login" src="https://github.com/user-attachments/assets/bc1f8fd8-25b9-48c6-a11b-a2e9ff62c8a4" />

---

### Step 2 — Update the App Library

Navigate to the **Apps** section. By default, only a limited number of apps are available. To load the full library, click the **cloud icon** and select **Force Update**:

<img width="1528" height="815" alt="Clicking the cloud icon to trigger app update" src="https://github.com/user-attachments/assets/d0b7f48d-c069-44b3-b1fe-9bdbb0724c56" />

<img width="1524" height="805" alt="Selecting Force Update to refresh the app library" src="https://github.com/user-attachments/assets/3fad08a1-ecbb-4134-b88d-a3c94fcf7149" />

---

### Step 3 — Build the Automation Workflow

In the Shuffle dashboard, create a **New Workflow** and name it `Wazuh-Mimikatz-Response`:

<img width="1527" height="818" alt="Creating a new workflow in Shuffle named Wazuh-Mimikatz-Response" src="https://github.com/user-attachments/assets/bdefcfea-4448-4c7e-9c4d-6cadc4ad8e02" />

The workflow will follow this pipeline:

```
Wazuh Alert → Extract SHA256 Hash → VirusTotal Lookup → TheHive Case → Discord Notification
```

---

#### Node 1 — Webhook Trigger (Receive Wazuh Alerts)

Add a **Webhook** trigger as the starting node. This is the entry point that receives alerts forwarded from Wazuh:

<img width="1539" height="819" alt="Adding a Webhook trigger node as the workflow entry point" src="https://github.com/user-attachments/assets/89ae242c-2866-4bda-ae39-4be67eb009b5" />

<img width="1538" height="815" alt="Webhook trigger node configured in Shuffle" src="https://github.com/user-attachments/assets/5b823bbe-7afd-4547-911d-50e83b30a5ed" />

Copy the generated webhook URL — you will need it in the next step.

---

#### Connect Wazuh to Shuffle

On the Wazuh server, open `ossec.conf` and add the Shuffle integration block:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Insert the following inside the `<ossec_config>` block, replacing the webhook URL with the one copied from Shuffle:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://10.0.50.103:3001/api/v1/hooks/webhook_bc7e75a8-8f03-4c29-8148-c859fc79c7e7</hook_url>
  <rule_id>100002</rule_id>
  <alert_format>json</alert_format>
</integration>
```

<img width="1178" height="705" alt="Shuffle integration block added to Wazuh ossec.conf" src="https://github.com/user-attachments/assets/993b5360-48ca-4473-b534-4042ce7ee0dd" />

> **Note:** `rule_id 100002` corresponds to the custom Mimikatz detection rule configured earlier. This can be modified to capture additional rules or adjusted by severity level as needed.

Save the file and restart the Wazuh manager to apply the changes:

```bash
sudo systemctl restart wazuh-manager
```

---

#### Verify the Wazuh → Shuffle Connection

Run Mimikatz on the Windows 10 machine to generate a detection alert and verify it is successfully forwarded to Shuffle:

<img width="837" height="167" alt="Executing Mimikatz on the Windows 10 machine" src="https://github.com/user-attachments/assets/719be4c9-eebb-4bcb-8c51-43cc110afa40" />

<img width="1540" height="775" alt="Mimikatz alert received and visible in Shuffle workflow" src="https://github.com/user-attachments/assets/b1959e65-b861-47a9-bd64-17bf4c4dfe8d" />

Back in Shuffle, confirm the workflow was triggered and the alert data was received successfully:

<img width="1537" height="818" alt="Shuffle workflow triggered by incoming Wazuh alert" src="https://github.com/user-attachments/assets/969c7f09-c512-4234-9067-0cec7147960c" />

<img width="1524" height="771" alt="Wazuh alert payload visible inside the Shuffle workflow execution" src="https://github.com/user-attachments/assets/d142ce25-cb5d-465b-bade-cd1a2a8b93d0" />

---

#### Node 2 — Extract the SHA256 Hash

With alert data flowing into Shuffle, the next step is to isolate the SHA256 file hash to use for threat intelligence lookup.

Add a **Regex Capture Group** node and set the input to the hashes field from the Wazuh alert:

```
$exec.text.win.eventdata.hashes
```

<img width="1541" height="819" alt="Configuring the input field to point to the Wazuh hashes event data" src="https://github.com/user-attachments/assets/fe34552d-7a15-45e6-a6e9-6a49b6da15ae" />

<img width="1535" height="820" alt="Raw hash value extracted from the incoming Wazuh alert payload" src="https://github.com/user-attachments/assets/8e9b6353-8c83-4bda-9da1-8fbe7060a2ab" />

Wazuh forwards process hashes in a combined format, for example:

```
SHA1=abc123...,MD5=def456...,SHA256=a1b2c3...
```

To extract only the SHA256 value, apply the following regex pattern in the capture group:

```regex
(?i)sha256=([A-Fa-f0-9]{64})
```

The `(?i)` flag makes the match case-insensitive, and the capture group `([A-Fa-f0-9]{64})` isolates exactly the 64-character hex hash, stripping the `SHA256=` prefix entirely.

<img width="1542" height="823" alt="Regex pattern configured to capture only the SHA256 hash value" src="https://github.com/user-attachments/assets/bc07b4c6-06c8-4f25-96c3-39a735e81488" />

Rename the node to something descriptive like `Extract_SHA256` for better visibility in the workflow canvas, then **save** the workflow and **run** it:

<img width="1539" height="824" alt="Node renamed to Extract_SHA256 for clarity in the workflow canvas" src="https://github.com/user-attachments/assets/17eeea27-2bf7-480d-a611-45ec1fb681c9" />

If configured correctly, the node will output a clean, isolated SHA256 hash ready to be passed directly into the VirusTotal lookup node:

<img width="1542" height="824" alt="Clean SHA256 hash successfully extracted and ready for VirusTotal lookup" src="https://github.com/user-attachments/assets/24a8f382-f259-4d68-b5d8-061959aa1128" />


#### Node 3 — VirusTotal Hash Lookup

The native VirusTotal app in Shuffle was not functioning as expected, so as a workaround an **HTTP node** was added to make direct API calls to VirusTotal instead. Rename the node to `VirusTotal` for clarity in the workflow canvas:

<img width="1541" height="821" alt="HTTP node added and renamed to VirusTotal in the workflow canvas" src="https://github.com/user-attachments/assets/0a4bdce1-bb60-484d-84ce-7e5b20b199d9" />

---

#### Configure the HTTP Node

Set the **Action** to `GET` and paste the following URL, which passes the extracted SHA256 hash from the previous node directly into the VirusTotal Files API endpoint:

```
https://www.virustotal.com/api/v3/files/$sha256-hash.group_0.#
```

<img width="1537" height="821" alt="HTTP GET request configured with the VirusTotal API URL and SHA256 hash variable" src="https://github.com/user-attachments/assets/8b0fb465-411e-4550-8cad-8068708ae20e" />

---

#### Add the VirusTotal API Key

VirusTotal requires authentication for all API requests. To get your API key:

1. Create a free account at [virustotal.com](https://www.virustotal.com)
2. Navigate to your **Profile → API Key**
3. Copy the key

Back in the HTTP node, expand **Optional Parameters → Headers** and add the following:

x-apikey:YOUR API KEY

<img width="1539" height="820" alt="VirusTotal API key added as x-apikey header in the HTTP node configuration" src="https://github.com/user-attachments/assets/08e429a3-1ec8-4144-9bb4-a5812db17224" />

---

#### Run & Verify

Save the workflow and run it again. A successful response will return a `200 OK` status, confirming that Shuffle has successfully queried VirusTotal with the extracted hash:

<img width="1538" height="817" alt="Workflow execution showing 200 OK response from the VirusTotal API" src="https://github.com/user-attachments/assets/75712ac6-402e-4d4c-9e36-532537e619ff" />

> ✅ A `200` status confirms the VirusTotal lookup is working correctly and the hash data is being returned for the next stage of the pipeline.

When checking attributes we can its identifrd malicious by 63 Secuirty Scanners.
<img width="1540" height="819" alt="image" src="https://github.com/user-attachments/assets/4d572580-3643-4950-b16d-5872d07c14cc" />


#### Node 4 — Create a Case in TheHive

1. Add the **TheHive** app
2. Authenticate using your TheHive URL and API key
3. Set the action to **Create Alert** or **Create Case**
4. Map the following fields from the Wazuh alert:

| TheHive Field | Wazuh Source |
|---------------|--------------|
| Title | `Mimikatz Detected on $exec.text.agent.name` |
| Severity | High |
| Description | Alert timestamp, agent name, file hash |
| Tags | `mimikatz`, `credential-dumping` |

#### Node 5 — Discord Notification

#### Set Up the Discord Webhook

In Discord, set up a dedicated alert channel and webhook by following these steps:

1. Create or open your Discord server
2. Create a text channel named `#soc-alerts`
3. Click the **gear icon** on the channel to open **Channel Settings**
4. Navigate to **Integrations → Webhooks → New Webhook**
5. Name it `Shuffle Automation`
6. Click **Copy Webhook URL** then hit **Save**

---

#### Configure the Discord Node in Shuffle

Add the **Discord** node to your workflow and configure it as follows:

| Field | Value |
|-------|-------|
| Action | `POST - Send a Message` |
| Webhook URL | Paste the URL copied from Discord |
| Body | See below |

Paste the following JSON into the **Body** field:

```json
{
  "content": "🚨 **Mimikatz Alert Detected**\n\n🖥️ **Host:** $exec.text.win.system.computer\n🔎 **Hash:** $sha256_hash.group_0.#\n🦠 **VirusTotal (Malicious):** $virustotal.#.body.data.attributes.last_analysis_stats.malicious\n📋 **TheHive Case:** Created Successfully\n⏰ **Time:** $exec.text.win.eventdata.utcTime"
}
```

<img width="1541" height="825" alt="Discord node configured in Shuffle with webhook URL and alert message body" src="https://github.com/user-attachments/assets/d69db864-b45f-415e-b673-7df40a0dfcb3" />

---

#### Test & Verify

Save the workflow and click **Test Action**. If everything is configured correctly, an alert message will appear in your `#soc-alerts` Discord channel within seconds:

<img width="1265" height="619" alt="image" src="https://github.com/user-attachments/assets/316a6638-2794-40d9-ae50-dba83cbb2226" />

The message will look like this:

```
🚨 Mimikatz Alert Detected

🖥️ Host: DESKTOP-WIN10
🔎 Hash: a1b2c3d4e5f6...
🦠 VirusTotal (Malicious): 65
📋 TheHive Case: Created Successfully
⏰ Time: 2025-01-15 14:32:10
```

> ✅ A message appearing in Discord confirms the full automation pipeline is working end to end — from Mimikatz execution on the Windows machine, through Wazuh detection, Shuffle orchestration, VirusTotal enrichment, all the way to analyst notification.

### Step 4 — Test the Full Pipeline

1. On the Windows 10 machine, execute Mimikatz
2. Wazuh detects the activity and fires the alert
3. The webhook forwards the alert to Shuffle
4. Shuffle extracts the hash and queries VirusTotal
5. A case is automatically created in TheHive
6. A Discord notification is sent to the analyst channel

If everything is configured correctly, within seconds of Mimikatz executing you should see a Discord message like this:

> 🚨 **Mimikatz Alert Detected**
> 🖥️ Host: `DESKTOP-WIN10`
> 🔎 Hash: `abc123...`
> 🦠 VirusTotal: `65/72 malicious detections`
> 📋 TheHive Case: Created Successfully
