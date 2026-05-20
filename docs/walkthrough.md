# Ghost in the Stack — Advanced Threat Hunt Walkthrough

> Copy this content into `docs/walkthrough.md` in VS Code.

---

## Project Overview

This walkthrough documents an advanced threat-hunting investigation against the host `GF-DEV01`. The investigation followed a live operator intrusion from initial authentication activity through implant deployment, persistence, lateral movement tooling, credential access behavior, enrichment, IOC extraction, containment planning, and final live-threat briefing.

The hunt used Microsoft Sentinel / Log Analytics KQL across Linux authentication, process, syscall, shell history, file, service, and network telemetry. Evidence was collected through screenshots and mapped to each investigation question.

---

## Repository Evidence Structure

Screenshots are stored in:

```text
Screenshots/
```

Supporting documentation is stored in:

```text
docs/
```

Planned supporting files:

```text
docs/walkthrough.md
docs/findings-summary.md
docs/lessons-learned.md
```

---

## Investigation Scope

**Affected host:** `GF-DEV01`  
**Primary compromised users observed:** `a.kumar`, `sancadmin`  
**Primary implant / payload artifacts:** `helix-update`, `helix-sync`, `hbsync.exe`, `wmi_exec.py`  
**Primary external IPs observed:** `194.36.110.139`, `104.21.57.185`  
**Primary techniques observed:** SSH access, process execution, implant persistence, systemd abuse, credential/file access, SMB staging, remote execution preparation, C2 infrastructure probing.

---

# Phase 1 — Initial Access and Authentication Review

## Linux Authentication Baseline

The investigation began by reviewing Linux authentication activity to identify suspicious logons and source IP behavior.

**Evidence:**

```md
![Linux Authentication Table](../Screenshots/1_Linux_Authentication_Table.jpg)
```

**Finding:** Authentication telemetry showed relevant activity tied to the affected host and users, establishing the access timeline for later correlation.

---

# Phase 2 — Implant Download and Execution Chain

## Implant Download

Process telemetry showed a curl-based download of the implant payload.

**Evidence:**

```md
![Implant Download Defender Process Tree](../Screenshots/2_Implant_Download_Defender_Process_Tree_pt_1.jpg)
![User Typed Installer Command](../Screenshots/2_User_Typed_Installer_Command_Sentinel_Shell_History_pt_2.jpg)
```

**Finding:** The operator used curl to retrieve an implant payload and staged it on the Linux host.

---

## Installer Process Tree

The installer activity was reconstructed through process telemetry.

**Evidence:**

```md
![Defender Process Tree Partial Installer Actions](../Screenshots/3_Defender_Process_Tree_Partial_Installer_Actions_pt_1.jpg)
![Sentinel LinuxProcess Full Installer Actions](../Screenshots/3_Sentinel_LinuxProcess_Full_Installer_Actions_pt_2.jpg)
```

**Finding:** Process telemetry provided the execution chain for the implant installation and supporting commands.

---

## PID / PPID Reconstruction

The implant process relationship was validated by confirming the PID and PPID.

**Answer:** `34616/1`

**Evidence:**

```md
![PID Chain Broad Filter](../Screenshots/4_PID_Chain_Query_Broad_Filter_pt_1%20(1).jpg)
![PID Chain Target and Acting Process IDs](../Screenshots/4_PID_Chain_Target_And_Acting_Process_IDs_pt_2.jpg)
![PID Chain Validated User Shell to Subshell](../Screenshots/4_PID_Chain_Validated_User_Shell_To_Subshell_pt_3.jpg)
![Helix Update PID PPID Validation](../Screenshots/7_Helix_Update_PID_PPID_Validation.jpg)
```

**Finding:** The implant process detached and continued under PID `34616` with PPID `1`, indicating it was no longer attached to the original shell process.

---

# Phase 3 — Watch Key and Tool Mapping

## Third Compromised Account Shell History

Shell history helped identify additional operator activity and another compromised account context.

**Evidence:**

```md
![Third Compromised Account Shell History](../Screenshots/5_Third_Compromised_Account_Shell_History.jpg)
```

---

## Closed False Positive DNS Alert Window

A false positive DNS alert was scoped and closed based on evidence.

**Evidence:**

```md
![Closed FP DNS Alert Footprint Window](../Screenshots/6_Closed_FP_DNS_Alert_Footprint_Window.jpg)
```

---

## Network Pivot and Node Carrier Activity

Network and process telemetry were used to identify pivot/node carrier behavior.

**Evidence:**

```md
![Broadened Network Pivot Node Carriers](../Screenshots/8_Broadened_Network_Pivot_Node_Carriers_pt_2.jpg)
![Helix Update PID No Direct Network Traffic](../Screenshots/8_Helix_Update_PID_No_Direct_Network_Traffic_pt_1.jpg)
![Node Carrier Process Command Lines](../Screenshots/8_Node_Carrier_Process_CommandLines_pt_3.jpg)
```

**Finding:** The implant itself did not directly create every network event. Related process activity helped reveal supporting node/carrier behavior.

---

## Linux Syscall Validation

Syscall telemetry confirmed lower-level activity visibility.

**Evidence:**

```md
![Linux Syscall Table Validation](../Screenshots/9_LinuxSyscall_Table_Validation.jpg)
```

---

## Tool-to-WatchKey Mapping

The investigation mapped tools to the artifacts or watch keys they accessed.

**Answer for Q10:**  
`aws:aws_creds, bash:ssh_user_keys, kubectl:kube_creds, ssh:ssh_user_keys`

**Evidence:**

```md
![Broad Tool to WatchKey Mapping](../Screenshots/10_Broad_Tool_To_WatchKey_Mapping_pt_1.jpg)
![Implant Scoped Tool to WatchKey Mapping](../Screenshots/10_Implant_Scoped_Tool_To_WatchKey_Mapping_pt_2.jpg)
```

---

## Implant Read-Most Finding

The implant itself most frequently accessed `claude_data`.

**Answer for Q11:** `claude_data`

**Evidence:**

```md
![Implant Reads Most Claude Data](../Screenshots/11_Implant_Reads_Most_Claude_Data.jpg)
```

---

## Audit Key and Proctitle Pivoting

Audit and syscall fields were decoded to pivot from raw telemetry into meaningful operator activity.

**Evidence:**

```md
![Implant Read AuditKeys](../Screenshots/12_Implant_Read_AuditKeys.jpg)
![AuditMsg Proctitle Decode Pivot](../Screenshots/13_AuditMsg_Proctitle_Decode_Pivot_pt_2.jpg)
![Proctitle Hex Decoded Comment](../Screenshots/13_Proctitle_Hex_Decoded_Comment_pt_3.jpg)
![SSH User Key Syscall Pivot](../Screenshots/13_SSH_User_Key_Syscall_Pivot_pt_1.jpg)
```

---

# Phase 4 — External Infrastructure and Enrichment

## WHOIS / ASN Enrichment

External source IP ownership was reviewed to enrich observed activity.

**Evidence:**

```md
![IP WHOIS ASN Provider](../Screenshots/14_IP_WHOIS_ASN_Provider_pt_2.jpg)
![TKT003 Source IP](../Screenshots/14_TKT003_Source_IP_pt_1.jpg)
```

---

## SSH Key Comment Attribution Signal

SSH key metadata provided an attribution signal.

**Evidence:**

```md
![SSH Key Comment Attribution Signal](../Screenshots/15_SSH_Key_Comment_Attribution_Signal.jpg)
```

---

## Event Volume and Logon Noise Closure

Event volume was grouped and reviewed to separate meaningful activity from noisy background events.

**Evidence:**

```md
![Credential Validation Pattern Analysis](../Screenshots/17_Credential_Validation_Pattern_Analysis.jpg)
![Event Volume Cluster By Minute](../Screenshots/18_Event_Volume_Cluster_By_Minute_pt_1.jpg)
![First Logon Noise Closure Cluster](../Screenshots/18_First_Logon_Noise_Closure_Cluster_pt_2.jpg)
```

---

# Phase 5 — Operator Activity, Pivoting, and Staging

## SSH Port Forwarding

The operator used SSH port forwarding behavior.

**Evidence:**

```md
![SSH Port Forward Command](../Screenshots/19_SSH_Port_Forward_Command_via_GFDEV01.jpg)
```

---

## Remote SAMR / IPC Pipe Access

Remote access attempts included SAMR / IPC pipe access.

**Evidence:**

```md
![Remote SAMR IPC Pipe Access](../Screenshots/20_Remote_SAMR_IPC_Pipe_Access.jpg)
```

---

## SMB Client Credential Exposure

SMB client activity exposed cleartext credential use.

**Evidence:**

```md
![Smbclient Cleartext Credential](../Screenshots/21_Smbclient_Cleartext_Credential.jpg)
![Smbclient Variant Testing](../Screenshots/22_Smbclient_Variant_Testing.jpg)
```

---

## Ligolo Fingerprint

Ligolo-related staging or fingerprint behavior was observed.

**Evidence:**

```md
![MDC Agentless Scan Ligolo Fingerprint](../Screenshots/24_MDC_Agentless_Scan_Ligolo_Fingerprint.jpg)
```

---

## Sancadmin Pivot Attempts

Pivot attempts were reviewed and scoped.

**Evidence:**

```md
![Sancadmin Pivot Attempts No Success](../Screenshots/25_Sancadmin_Pivot_Attempts_No_Success.jpg)
```

---

# Timeline Analysis

## Q26 — Dwell Time

The dwell time was calculated from the implant detach timestamp to the first external `sancadmin` SSH login.

**Answer:** `104`

**Evidence:**

```md
![Dwell Time 104 Minutes Calculation](../Screenshots/26_Dwell_Time_104_Minutes_Calculation.jpg)
![First External Sancadmin SSH](../Screenshots/26_First_External_Sancadmin_SSH.jpg)
![Implant Detach Time PID 34616](../Screenshots/26_Implant_Detach_Time_PID_34616_1.jpg)
```

**Finding:** The implant detached, became persistent, and the first external `sancadmin` SSH login occurred 104 whole minutes later.

---

# Containment Planning

## Q28 — DEV01 Cleanup Paths

The cleanup list required every file or unit that had to be removed from `DEV01`.

**Evidence:**

```md
![DEV01 Containment Cleanup Paths](../Screenshots/28_DEV01_Containment_Cleanup_Paths.jpg)
```

**Finding:** The cleanup list included the rogue SSH key, implant binary, Ligolo binary, systemd persistence unit, and state/cache artifacts identified during the earlier phases.

---

# Enrichment

## Q29 — Install.sh Destination Infrastructure

The install script download connected to `104.21.57.185:443`, which resolved to Cloudflare-owned infrastructure. The correct conclusion was that the malicious infrastructure was hidden behind a fronting-related CDN/proxy layer rather than a simple single-origin host that could be safely blocked at the perimeter.

**Evidence:**

```md
![InstallSH Curl To Cloudflare Fronted IP](../Screenshots/29_InstallSH_Curl_To_Cloudflare_Fronted_IP.jpg)
```

**Finding:** Blocking the destination IP directly could affect legitimate shared CDN infrastructure.

---

# IOC Extraction

## Q30 — Late Session C2 Watchlist IOC

Late-session operator activity revealed an infrastructure probe against their own listener.

**Format:** `IP:PORT`

**Evidence:**

```md
![Late Session C2 Watchlist IOC](../Screenshots/30_Late_Session_C2_Watchlist_IOC.jpg)
```

---

# Detection Engineering

## Q31 — Sigma Rule Title

**Answer:** `sliver implant launched via systemd`

**Walkthrough Explanation:**  
The rule title follows Sigma-style naming: framework name, implant, action verb, and mechanism. The activity matched a Sliver implant launched through systemd on Linux.

---

## Q32 — Sigma Logsource Service

The rule targets Linux syscall-level execution telemetry captured through kernel watch keys.

**Walkthrough Explanation:**  
The detection should target the logsource service aligned to syscall/auditd-style Linux execution events rather than a generic process log.

---

## Q33 — Systemd-as-PPID Field

The Sigma selection used an image/path condition for the implant and a second field to match the parent process as `/sbin/init`.

**Walkthrough Explanation:**  
This catches the pattern where the implant is represented as detached and systemd/init becomes the parent process.

---

## Q34 — Severity Level

The severity should be calibrated high enough because this is not a noisy heuristic. It detects a real implant landing and persistence pattern.

---

## Stream Differentiator Fields

Fields used to differentiate stream/log types were reviewed.

**Evidence:**

```md
![Stream Differentiator Fields](../Screenshots/36_Stream_Differentiator_Fields.jpg)
```

---

# Additional Operator Activity

## Sancadmin Successful Logons

**Evidence:**

```md
![Sancadmin Successful Logons SrcIpAddr](../Screenshots/37_Sancadmin_Successful_Logons_SrcIpAddr.jpg)
```

---

## SFTP Server Writes New Binaries

**Evidence:**

```md
![SFTP Server Wrote New Binaries](../Screenshots/38_SFTP_Server_Wrote_New_Binaries.jpg)
```

---

## Decoded Payload and WinRM Command

**Evidence:**

```md
![Decoded Payload URL and Drop Path](../Screenshots/39_Decoded_Payload_URL_and_DropPath_pt_2.jpg)
![WinRM Encoded Payload Command](../Screenshots/39_WinRM_Encoded_Payload_Command_pt_1.jpg)
```

---

## Sancadmin Two-Phase Persistence

**Evidence:**

```md
![Sancadmin Two Phase Session Persistence First](../Screenshots/40_Sancadmin_Two_Phase_Session_Persistence_First.jpg)
```

---

# Live Threat Brief

## Q41 — Final Live Threat State

**Answer:** `authorized_keys:persistent, helix-sync.service:dormant, helix-update:running`

The final brief required three artifacts, each in a different state:

| Artifact | State | Evidence Layer |
|---|---:|---|
| `authorized_keys` | persistent | File / credential persistence |
| `helix-sync.service` | dormant | Service state |
| `helix-update` | running | Process telemetry |

**Evidence:**

```md
![Process Telemetry Helix Update Running](../Screenshots/41_Process_Telemetry_Helix_Update_Running_pt_1.jpg)
![Service Persistence Helix Sync Service](../Screenshots/41_Service_Persistence_Helix_Sync_Service_pt_2.jpg)
![File Telemetry Hbsync Dormant](../Screenshots/41_File_Telemetry_Hbsync_Dormant_pt_3.jpg)
```

**Finding:** At the end of the hunt, the active process layer showed the implant still running, the service layer showed persistence present but not actively running, and file telemetry showed staged/dropped tooling that was no longer actively executing.

---

# Final Summary

This hunt reconstructed a multi-stage Linux intrusion involving implant deployment, detached execution, systemd-based persistence, SSH abuse, credential and file access, SMB staging, and external infrastructure enrichment. The investigation moved from triage to containment by identifying the active implant, dormant/staged tooling, persistence mechanisms, and cleanup paths needed for incident response.

---

# Lessons Learned

- Break large hunts into small evidence-backed pivots.
- Avoid relying only on alert UI timestamps when Advanced Hunting / Log Analytics tables provide better event timing.
- Validate process state, file state, and service state separately.
- Use shell history, process telemetry, file telemetry, network telemetry, and syscall data together.
- For CDN-fronted infrastructure, do enrichment before recommending broad perimeter blocks.
- Screenshots should show both the query and the result whenever possible.
