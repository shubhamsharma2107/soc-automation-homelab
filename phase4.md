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

After the update, **VirusTotal** may not appear in your app list automatically. To add it:

1. Go to **Apps → Discover Public Apps**
2. Search for `VirusTotal`
3. Click **Activate** to add it to your organization's app library

<img width="1543" height="822" alt="Searching for and activating VirusTotal in public apps" src="https://github.com/user-attachments/assets/f29aabbb-9f5f-4768-99d9-ce56c3f04bdb" />

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

With the alert data flowing into Shuffle, the next step is to isolate the file hash for threat intelligence lookup. Add a **Regex Capture Group** node and set the input to:

```
$exec.text.win.eventdata.hashes
```

<img width="1541" height="819" alt="Configuring the execution argument to extract the hashes field" src="https://github.com/user-attachments/assets/fe34552d-7a15-45e6-a6e9-6a49b6da15ae" />

<img width="1535" height="820" alt="Hash value extracted from the Wazuh alert payload" src="https://github.com/user-attachments/assets/8e9b6353-8c83-4bda-9da1-8fbe7060a2ab" />

Since Wazuh forwards hashes in a combined format (e.g. `SHA1=...,MD5=...,SHA256=...`), apply the following regex to extract only the SHA256 value:

```regex
SHA256=([A-Fa-f0-9]{64})
```

<img width="1537" height="821" alt="Regex pattern configured to capture the SHA256 hash from the hashes field" src="https://github.com/user-attachments/assets/0fec5630-9f1e-4a68-91f9-60b72f6333ca" />

This will output a clean 64-character SHA256 hash, ready to be passed into the VirusTotal lookup node.

#### Node 3 — VirusTotal Hash Lookup

1. Add the **VirusTotal** app from the Shuffle app library
2. Connect your VirusTotal API key under app authentication
3. Set the action to **Get Hash Report**
4. Pass in the extracted hash from Node 2

Shuffle will query VirusTotal and return the detection ratio and threat verdict for the hash.

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

1. Add the **Discord** app or use an **HTTP** node with your Discord webhook URL
2. Set up a Discord webhook in your server under **Server Settings → Integrations → Webhooks**
3. Copy the webhook URL and paste it into Shuffle
4. Set the action to **Send Message** with a message body such as:

```
🚨 *Mimikatz Alert Detected*

🖥️ Host: $exec.text.agent.name
🔎 Hash: $exec.text.win.eventdata.hashes
🦠 VirusTotal: $virustotal.data.attributes.last_analysis_stats.malicious detections
📋 TheHive Case: Created Successfully
⏰ Time: $exec.text.timestamp
```

---

### Step 3 — Connect Wazuh to Shuffle

On the Wazuh manager, edit `ossec.conf` to forward alerts to the Shuffle webhook:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following integration block:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://<SHUFFLE_IP>:3001/api/v1/hooks/<YOUR_WEBHOOK_ID></hook_url>
  <rule_id>100002</rule_id>
  <alert_format>json</alert_format>
</integration>
```

> Replace `<SHUFFLE_IP>` and `<YOUR_WEBHOOK_ID>` with your actual values. `rule_id` should match the custom Mimikatz rule created earlier.

Restart the Wazuh manager to apply changes:

```bash
sudo systemctl restart wazuh-manager
```

---

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
