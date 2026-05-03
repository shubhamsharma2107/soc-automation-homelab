## Phase 3: Execute Mimikatz & Verify Wazuh Detections

The following steps are performed on the **Windows 10 endpoint**.

---

### Step 1 — Exclude the Downloads Folder from Windows Defender

To prevent Windows Defender from quarantining our payload before execution, we need to add the `Downloads` folder as an exclusion.

1. Open **Virus & Threat Protection** in Windows Security.
2. Navigate to **Manage settings** under *Virus & threat protection settings*.
3. Scroll down to **Exclusions** and click **Add or remove exclusions**.
4. Add your `Downloads` folder.
   
<br>
<img width="800" alt="Open Virus and Threat Protection" src="https://github.com/user-attachments/assets/66e4dc02-7266-4425-a67d-0889beb75b90" />
<img width="800" alt="Navigate to exclusions settings" src="https://github.com/user-attachments/assets/e0553469-969a-4eba-a7f1-6ecb1b263389" />
<img width="800" alt="Add folder exclusion" src="https://github.com/user-attachments/assets/c69af6f3-27b9-4315-ad1c-37cfd97b06f6" />
<img width="800" alt="Downloads folder added as exclusion" src="https://github.com/user-attachments/assets/668f077d-f89c-496d-b310-d9e877b2fbc0" />

---

### Step 2 — Download Mimikatz

Download the Mimikatz executable directly from the ParrotSec repository to the excluded folder:

🔗 **[Download mimikatz.exe (x64)](https://github.com/ParrotSec/mimikatz/blob/master/x64/mimikatz.exe)**

<img width="785" alt="Downloading Mimikatz from GitHub" src="https://github.com/user-attachments/assets/5b243989-70b5-4e52-b524-d8ba28ec593f" />

---

### Step 3 — Execute Mimikatz & Suppress Defender Alerts

Before running the executable, add `mimikatz.exe` as a process exclusion via PowerShell to prevent Defender from interfering during execution:

```powershell
Add-MpPreference -ExclusionProcess "mimikatz.exe"
```

<img width="837" height="184" alt="Adding mimikatz.exe as a Defender process exclusion" src="https://github.com/user-attachments/assets/72ed6c2a-0fbe-41df-a73a-b3dc6c12eeae" />

Now execute Mimikatz on the Windows 10 machine and then we will monitor the Wazuh dashboard for any generated alerts:

<img width="858" height="421" alt="image" src="https://github.com/user-attachments/assets/ebdc1d2d-db37-486b-9fe9-61afbccf04f3" />

<br>Returning to the Wazuh dashboard, Mimikatz-related alerts were successfully identified within the `wazuh-alerts-*` index. These detections were captured and forwarded by the Sysmon integration configured earlier.</br>

<img width="1541" height="864" alt="Mimikatz alerts visible in the Wazuh dashboard under wazuh-alerts index" src="https://github.com/user-attachments/assets/e57f8fc0-ab2a-4299-83d7-9dfeafd3a914" />

### Step 4 — Create a Custom Rule to Detect Mimikatz

Next, we will create a custom detection rule in Wazuh to ensure we are specifically alerting on Mimikatz usage. 

1. In the Wazuh dashboard, navigate to **Server management** > **Rules**.
2. Locate the Sysmon event logs we generated earlier. We need to identify the specific fields we want to base our custom rule on.

<br>
<img width="1542" alt="Navigate to Server Management" src="https://github.com/user-attachments/assets/a67408d1-e698-4b90-aa4a-7d09fb317d79" />
<img width="1543" alt="Navigate to Rules" src="https://github.com/user-attachments/assets/aa3ebc7b-cff9-4062-b41c-d1f3b549835d" />
<img width="1538" alt="View event logs" src="https://github.com/user-attachments/assets/ae90b2c8-cfec-474e-9376-e0f81093dfda" />
<img width="1538" alt="Copy log details for rule creation" src="https://github.com/user-attachments/assets/5bb7d9ff-c326-4d23-a3c8-73b26cf30331" />

3. Click on **Custom Rules** and click on edit an existing custom rules file **localrules.xml**  to create our detection logic.

<img width="1540" alt="Select Manage Rules" src="https://github.com/user-attachments/assets/e07f627d-c03f-401a-b5c2-90cdd4850427" />
<img width="1545" alt="Create new rule file" src="https://github.com/user-attachments/assets/3392c7b8-7996-4029-b3c4-2b3d43f9d6a2" />

Insert the following XML block into your custom rules file:

```xml
<rule id="100002" level="10">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
    <description>Mimikatz Usage Detected</description>
    <mitre>
      <id>T1003</id>
    </mitre>
</rule>
```
<img width="1540" alt="Edit custom rule" src="https://github.com/user-attachments/assets/99ecace9-e4bb-4b26-94e5-d7fadb01c70b" />

4. **Save** the file and click the **Restart** button to reload the Wazuh manager so the new rule takes effect.

<img width="1545" alt="Save and restart Wazuh manager" src="https://github.com/user-attachments/assets/c30da2f4-17cc-49c7-af23-bc992aaf0956" />

---

### Step 5 — Verify Detection Resilience (Renaming the Executable)

> **💡 Note:** In our custom rule, we specifically targeted the `win.eventdata.originalFileName` field. This refers to the original name embedded within the executable's metadata, which remains persistent even if the file is renamed on the host's disk. 

To test the robustness of our rule, let's rename `mimikatz.exe` to a disguised name, execute it again, and verify if Wazuh still triggers an alert. 

If our rule had relied solely on the `Image` field (the file path/name on disk), renaming the file would have successfully bypassed the detection. However, because we leveraged the file's internal metadata, the alert triggers successfully regardless of what the attacker calls the file.

<br>
<img width="856" alt="Renaming and executing mimikatz" src="https://github.com/user-attachments/assets/78f10772-afb7-432a-96a3-d02231463a1b" />
<img width="1544" alt="Wazuh dashboard alert view" src="https://github.com/user-attachments/assets/ed6cad34-e1d1-4f0f-b58c-7835d56b0436" />
<img width="1541" alt="Alert details showing originalFileName match" src="https://github.com/user-attachments/assets/b818c46b-7386-4547-94f6-4705477a6782" />
<img width="1545" height="823" alt="image" src="https://github.com/user-attachments/assets/8886302b-7295-4425-ba31-3801c35b8bab" />

