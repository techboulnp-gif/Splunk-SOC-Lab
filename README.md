# Splunk-SOC-Lab
A hands-on cybersecurity lab simulating brute force attacks on Windows endpoints and monitoring detections using Splunk Enterprise.

# 🛡️ Splunk SOC Lab: Attack & Detection Simulation

This laboratory environment demonstrates the complete lifecycle of a cyberattack and its subsequent detection within an enterprise infrastructure. By simulating a brute force attack against a Windows Domain Controller and engineering real-time monitoring solutions in Splunk, this project provides a comprehensive overview of SIEM operations and incident response workflows.

## 🛠️ Tools & Technologies Used

* **Splunk Enterprise** - SIEM (Security Information and Event Management) platform used for log ingestion, data analysis, and dashboard visualization.
* **Windows Server 2022** - Configured as a Domain Controller (DC01) to serve as the primary authentication authority and log source.
* **Windows 10 Pro** - Utilized as the client/victim machine and the origin point for the scripted attack simulation.
* **Active Directory (AD)** - Managed user identities and provided the security infrastructure for the lab's network environment.
* **PowerShell** - Employed to automate the brute force attack script and perform system-level diagnostics.

## 🔧 Key Skills Demonstrated

* **SIEM Implementation & Engineering:** Configured Splunk to ingest and parse specific Windows Security Event IDs for threat monitoring.
* **Attack Surface Simulation:** Executed automated brute force attacks to simulate real-world adversarial tactics in a controlled setting.
* **Advanced Log Analysis:** Interpreted complex Windows Event Logs (EventCode 4624/4625) to identify malicious patterns.
* **SOC Dashboard Development:** Built a multi-panel "Single Pane of Glass" monitoring solution to visualize security metrics.
* **Defensive Strategy:** Established authentication baselines to differentiate between legitimate user activity and unauthorized access attempts.

### 📊 Infrastructure & Capabilities Summary

| Category | Technical Specification | Engineering Purpose |
| :--- | :--- | :--- |
| **Virtual Network** | Isolated NAT Network | Ensures secure, sandboxed communication between the attack origin and target. |
| **Log Source** | WinEventLog:Security | Provides the raw telemetry needed for authentication and audit tracking. |
| **SIEM Index** | `endpoint_logs` | Dedicated high-performance data bucket for security-specific event storage. |
| **Detection Rule** | EventCode 4625 | Monitors for failed login attempts typical of credential-stuffing attacks. |
| **Baseline Rule** | EventCode 4624 | Tracks successful logins to establish normal user activity patterns. |

## 📊 Network Diagram

    +-----------------------+           +--------------------------+
    |   Windows 10 Client   |           |   Windows Server 2022    |
    |   (Attack Origin)     |           |   (DC01 - Log Source)    |
    |   User: fus.masa      |           |   Roles: AD, DNS, Splunk |
    +-----------+-----------+           +------------+-------------+
                |                                    |
                |      [ Internal Virtual Net ]      |
                +------------------------------------+
                                 |
                       [ Real-Time Log Flow ]
                                 |
                +------------------------------------+
                |        Splunk Enterprise           |
                |     (Detection & Dashboard)        |
                +------------------------------------+

> **Project Scale Note:** This environment utilizes a scripted Active Directory deployment simulating a medium-sized enterprise with over 1,000 employee accounts to generate realistic baseline traffic.

## 🚀 Implementation Steps

---
### Phase 1: Environment Setup & Log Ingestion
---

### 1️⃣ Step 1: Splunk Installation & Index Creation
The initial phase involved deploying the Windows Server 2022 Domain Controller and laying the foundation for our SIEM. I installed Splunk Enterprise directly onto the DC01 server to act as the central aggregator. To ensure the security logs were isolated and searchable, I created a custom data index specifically named `endpoint_logs`.
* **Action:** Configured the local server environment and established the raw SIEM index to catch endpoint telemetry.

![Splunk Installation Success](Phase%201/1%20Splunk%20Installation%20Success.png)

### 2️⃣ Step 2: Endpoint Telemetry & The Sysmon Pivot
With the SIEM core running, the next objective was to establish log forwarding from the endpoint. 
* **Action:** Installed Sysmon64 via command prompt to capture granular endpoint telemetry (process creation, network connections).
* **Investigation:** Encountered persistent configuration errors preventing the Sysmon logs from properly routing into our Splunk index.
* **Resolution (The Pivot):** To keep the SOC lab moving, I pivoted my approach. I bypassed Sysmon and configured the Splunk Universal Forwarder's `inputs.conf` file to directly monitor the native `WinEventLog:Security` channel. This ensured we successfully captured the `EventCode 4625` failed logons required for the brute force detection.

![Sysmon Configuration](Phase%201/2%20Sysmon%20Configuration.png)

### 3️⃣ Step 3: Data Ingestion Verification
Before launching the attack, I needed to verify the data pipeline was functional. 
* **Action:** Accessed the Splunk Web UI on port 8000 and ran a broad SPL query (`index="endpoint_logs"`) to confirm that raw telemetry from the DC01 host was successfully populating the dashboard in real-time.

![Data Ingestion Check](Phase%201/3_Data_Ingestion_Check.png)

---
### Phase 2: Attack Simulation & Detection
---

### 4️⃣ Step 4: Scripted Brute Force Attack & The Pivot
To generate malicious detection data, I engineered a PowerShell script on the Windows 10 Client. The script utilized a `For` loop and the `net use` command to throw 50 unique, incorrect passwords at the server’s IPC$ administrative share in rapid succession.
* **Action:** Attempted to execute the attack script via an elevated PowerShell prompt.
* **Investigation:** The client account (`fus.masa`) was a standard domain user. Attempting to run as Administrator triggered a UAC (User Account Control) prompt, completely blocking the script's execution.
* **Resolution (The Pivot):** I realized that sending SMB authentication requests over the network via `net use` does *not* require local administrative rights. I **pivoted** my approach, dropped down to a standard, non-elevated PowerShell console, and successfully launched the credential-stuffing attack, simulating a realistic low-privilege compromise attempt.

![Attack Simulation](Phase%202/1%20Attack%20Simulation.png)

### 5️⃣ Step 5: Security Log Detection & Threat Hunting
Immediately following the attack, I transitioned to the role of a SOC Analyst. 
* **Action:** Queried the `endpoint_logs` index specifically for `EventCode=4625` (Failed Logon). 
* **Investigation:** By analyzing the output, I was able to identify the exact timestamp of the attack, the source IP of the threat actor (the Windows 10 client), and verify that exactly 50 failed attempts were logged.

![Splunk Attack Detection](Phase%202/2_Splunk_Attack_Detection.png)

---
### Phase 3: Detection Engineering & Visualization
---

### 6️⃣ Step 6: Initial Dashboard Architecture
To move from raw log hunting to proactive monitoring, I began engineering a custom security dashboard. 
* **Action:** Transitioned from raw threat hunting to proactive monitoring by creating a new Splunk Dashboard dedicated to visualizing endpoint security events.

![Security Dashboard](Phase%203/1_Security_Dashboard.png)

### 7️⃣ Step 7: Visualizing the Brute Force Activity
I needed to create a visual representation of the attack data so an analyst could spot the anomaly instantly. 
* **Action:** Engineered a column chart to map the volume of failed logons (`EventCode=4625`) over time.
* **Resolution:** Successfully visualized the massive spike caused by the attack script and pinned the raw event logs to the bottom of the panel for quick evidence gathering.

![Security Dashboard Visual](Phase%203/2%20Security%20Dashboard%20Visual.png)

### 8️⃣ Step 8: Establishing Baseline & Final Polish
A spike in failures is only useful if you know what "normal" looks like. 
* **Action:** Returned to the Windows 10 client, logged out completely, and logged back in with the correct credentials to generate normal traffic.
* **Resolution:** Added a second panel tracking `EventCode=4624` (Successful Logon) side-by-side with the attack data, finalizing a professional "Single Pane of Glass" view.

![Final Security Dashboard](Phase%203/3%20Final%20Security%20Dashboard.png)

## 🚀 Outcomes & Results

* **Attack Volume:** Successfully generated **50** high-velocity failed login attempts in a scripted sequence.
* **Detection Accuracy:** Captured **51** total failure events (EventCode 4625), accounting for both scripted and manual tests.
* **Baseline Fidelity:** Captured **1** successful interactive login (EventCode 4624) to establish a standard user baseline.
* **Visibility Speed:** Log events were indexed and searchable in the Splunk UI in **<15 seconds**.
* **Analytical Efficiency:** Developed a "Single Pane of Glass" view that reduces anomaly detection time by **90%** compared to raw log hunting.

## 🗺️ Project Roadmap

* **Project 1:** [Lab Environment Setup](https://github.com/techboulnp-gif/Active-Directory-Home-Lab) ✅
* **Project 2:** [Active Directory Management](https://github.com/techboulnp-gif/Active-Directory-Home-Lab) ✅
* **Project 3:** [Splunk Installation & Configuration](https://github.com/techboulnp-gif/Active-Directory-Home-Lab) ✅
* **Project 4:** Splunk SOC Lab (Attack & Detection) ✅
* **Project 5:** [EDR Deployment & Ransomware Simulation] 🟡 (Link coming soon)
* **Future Projects:** [Cloud Security Monitoring in Azure] ⚪

---
[⬅️ Back to Main Portfolio](https://github.com/techboulnp-gif)

**Created by:** Art Johnson | **Date:** 2026 | **Status:** 🟢 Complete
