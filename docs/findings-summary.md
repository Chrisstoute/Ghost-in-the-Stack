<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:050505,45:6B7280,100:F8FAFC&height=170&section=header&text=Ghost%20in%20the%20Stack&fontSize=42&fontColor=ffffff&animation=fadeIn" />
</p>

<h1 align="center">Findings Summary</h1>

## Executive Summary

This advanced threat hunt investigated a multi-stage Linux intrusion on **GF-DEV01** involving implant deployment, persistence, credential access, lateral movement preparation, and live operator activity. The investigation reconstructed the operator timeline from authentication, process, file, syscall, network, and service telemetry.

The hunt identified a live implant process, post-compromise tooling, persistence through systemd, external SSH access, and staged payloads written to the host. The final live threat brief confirmed three end-of-shift artifacts across separate system layers: a running process, a persistent credential artifact, and a dormant staged binary.

## Environment

| Item | Value |
|---|---|
| Host | GF-DEV01 |
| Primary User Context | sancadmin |
| Investigation Type | Advanced Threat Hunt / SecOps / Incident Response |
| Data Sources | LinuxAuth_CL, LinuxProcess_CL, LinuxFile_CL, LinuxNetwork_CL, LinuxSyscall_CL, Syslog |
| Main Tools Used | Microsoft Sentinel / Defender Advanced Hunting, KQL, OSINT, Sigma-style detection logic |

## Key Findings

### 1. Initial Linux Authentication Baseline

Authentication telemetry established the Linux authentication table as a key source for successful and failed SSH activity. This became the foundation for tracking external access, compromised account use, and attacker-controlled login activity.

**Evidence:**

![Linux Authentication Table](../Screenshots/1_Linux_Authentication_Table.jpg)

---

### 2. Implant Download and Execution Chain

Process telemetry showed the implant download and execution sequence. The operator used `curl` to retrieve the implant payload and then executed it on GF-DEV01.

Key observed behavior included:

- Download activity for `helix-update`
- Process execution tied to the implant
- PID/PPID validation showing the implant process relationship

**Evidence:**

![Implant Download Defender Process Tree](../Screenshots/2_Implant_Download_Defender_Process_Tree_pt_1.jpg)

![Defender Process Tree Partial Installer Actions](../Screenshots/3_Defender_Process_Tree_Partial_Installer_Actions_pt_1.jpg)

![Sentinel LinuxProcess Full Installer Actions](../Screenshots/3_Sentinel_LinuxProcess_Full_Installer_Actions_pt_2.jpg)

![Helix Update PID PPID Validation](../Screenshots/7_Helix_Update_PID_PPID_Validation.jpg)

---

### 3. Implant Process Became Detached and Live

The implant process was validated as a detached process with PID `34616` and PPID `1`, indicating it had separated from the initiating process and was running independently.

This supported the conclusion that the implant was still active and required immediate containment rather than additional passive investigation.

**Evidence:**

![Helix Update PID PPID Validation](../Screenshots/7_Helix_Update_PID_PPID_Validation.jpg)

![Implant Detach Time PID 34616](../Screenshots/26_Implant_Detach_Time_PID_34616_1.jpg)

![Process Telemetry Helix Update Running](../Screenshots/41_Process_Telemetry_Helix_Update_Running_pt_1.jpg)

---

### 4. Dwell Time Calculation

The dwell time was calculated from the implant detach timestamp to the first successful external SSH login by `sancadmin`.

Final dwell time:

```text
104 minutes
```

**Evidence:**

![Dwell Time 104 Minutes Calculation](../Screenshots/26_Dwell_Time_104_Minutes_Calculation.jpg)

![First External Sancadmin SSH](../Screenshots/26_First_External_Sancadmin_SSH.jpg)

![Implant Detach Time PID 34616](../Screenshots/26_Implant_Detach_Time_PID_34616_1.jpg)

---

### 5. External Access from Attacker Infrastructure

Authentication records identified successful SSH access by `sancadmin` from an external IP address. This confirmed external operator access into GF-DEV01.

**Evidence:**

![TKT003 Source IP](../Screenshots/14_TKT003_Source_IP_pt_1.jpg)

![First External Sancadmin SSH](../Screenshots/26_First_External_Sancadmin_SSH.jpg)

---

### 6. Network Pivoting and Infrastructure Enrichment

Network telemetry showed outbound activity to infrastructure associated with attacker staging and command activity. OSINT enrichment was used to determine that one observed destination belonged to Cloudflare-fronted infrastructure, explaining why blocking only the destination IP would be unreliable.

**Evidence:**

![IP WHOIS ASN Provider](../Screenshots/14_IP_WHOIS_ASN_Provider_pt_2.jpg)

![InstallSH Curl To Cloudflare Fronted IP](../Screenshots/29_InstallSH_Curl_To_Cloudflare_Fronted_IP.jpg)

![Late Session C2 Watchlist IOC](../Screenshots/30_Late_Session_C2_Watchlist_IOC.jpg)

---

### 7. Watch Key and Tool Activity

The operator used multiple tools and accessed several sensitive areas. Watch-key analysis mapped tooling to targeted areas, including SSH user keys, cloud credentials, Kubernetes credentials, and Claude-related data.

Key mapped activity included:

```text
aws:aws_creds
bash:ssh_user_keys
kubectl:kube_creds
ssh:ssh_user_keys
```

The implant itself most frequently read:

```text
claude_data
```

**Evidence:**

![Broad Tool To WatchKey Mapping](../Screenshots/10_Broad_Tool_To_WatchKey_Mapping_pt_1.jpg)

![Implant Scoped Tool To WatchKey Mapping](../Screenshots/10_Implant_Scoped_Tool_To_WatchKey_Mapping_pt_2.jpg)

![Implant Reads Most Claude Data](../Screenshots/11_Implant_Reads_Most_Claude_Data.jpg)

![Implant Read AuditKeys](../Screenshots/12_Implant_Read_AuditKeys.jpg)

---

### 8. Syscall and Raw Telemetry Pivoting

Linux syscall and raw telemetry were used to decode audit fields, extract process command lines, and pivot from noisy telemetry into meaningful operator behavior.

**Evidence:**

![AuditMsg Proctitle Decode Pivot](../Screenshots/13_AuditMsg_Proctitle_Decode_Pivot_pt_2.jpg)

![Proctitle Hex Decoded Comment](../Screenshots/13_Proctitle_Hex_Decoded_Comment_pt_3.jpg)

![SSH User Key Syscall Pivot](../Screenshots/13_SSH_User_Key_Syscall_Pivot_pt_1.jpg)

---

### 9. Credential and SSH Key Activity

The operator activity involved SSH key discovery and credential validation behavior. This supported the conclusion that the intrusion included credential access and potential persistence preparation.

**Evidence:**

![SSH Key Comment Attribution Signal](../Screenshots/15_SSH_Key_Comment_Attribution_Signal.jpg)

![Credential Validation Pattern Analysis](../Screenshots/17_Credential_Validation_Pattern_Analysis.jpg)

![Smbclient Cleartext Credential](../Screenshots/21_Smbclient_Cleartext_Credential.jpg)

---

### 10. Lateral Movement Preparation

The operator used `smbclient` with cleartext credentials and staged Windows-oriented payloads, including `hbsync.exe`, to remote Windows paths. This indicated preparation for cross-platform movement or payload transfer.

**Evidence:**

![Remote SAMR IPC Pipe Access](../Screenshots/20_Remote_SAMR_IPC_Pipe_Access.jpg)

![Smbclient Cleartext Credential](../Screenshots/21_Smbclient_Cleartext_Credential.jpg)

![Smbclient Variant Testing](../Screenshots/22_Smbclient_Variant_Testing.jpg)

![SFTP Server Wrote New Binaries](../Screenshots/38_SFTP_Server_Wrote_New_Binaries.jpg)

---

### 11. Ligolo Fingerprint / Agentless Scan Activity

Network and process evidence showed activity consistent with Ligolo-related pivoting or tunneling behavior, supporting the investigation’s conclusion that the operator used infrastructure to support internal movement.

**Evidence:**

![MDC Agentless Scan Ligolo Fingerprint](../Screenshots/24_MDC_Agentless_Scan_Ligolo_Fingerprint.jpg)

---

### 12. Containment Cleanup List

The containment phase identified files and units that required cleanup from DEV01. The cleanup list included the rogue SSH key, implant binary, Ligolo binary, systemd unit, and related state/cache artifacts.

**Evidence:**

![DEV01 Containment Cleanup Paths](../Screenshots/28_DEV01_Containment_Cleanup_Paths.jpg)

---

### 13. Detection Engineering

The spawn pattern was converted into a Sigma-style detection concept:

```text
sliver implant launched via systemd
```

The detection targeted Linux syscall-level process execution data captured through kernel watch keys.

**Evidence / Explanation:**

Detection logic is documented in the walkthrough narrative for Q31 and Q32.

---

### 14. Final Live Threat Brief

At the end of the hunt, three artifacts remained across different system layers:

```text
authorized_keys:persistent
helix-sync.service:dormant
helix-update:running
```

This shows persistence, dormant service state, and a currently running implant process.

**Evidence:**

![Process Telemetry Helix Update Running](../Screenshots/41_Process_Telemetry_Helix_Update_Running_pt_1.jpg)

![Service Persistence Helix Sync Service](../Screenshots/41_Service_Persistence_Helix_Sync_Service_pt_2.jpg)

![File Telemetry Hbsync Dormant](../Screenshots/41_File_Telemetry_Hbsync_Dormant_pt_3.jpg)

---

## Final Assessment

The activity on GF-DEV01 represented an active compromise involving a live implant, persistent access preparation, credential discovery, staging of additional tooling, and external operator access. The implant was confirmed as running, which shifted the case from investigation-only activity to immediate incident response and containment.

## Recommended Immediate Actions

1. Isolate GF-DEV01 from the network.
2. Preserve forensic evidence before cleanup.
3. Remove identified malicious files, units, and credential artifacts.
4. Revoke and rotate affected credentials.
5. Audit SSH authorized keys for all impacted users.
6. Review lateral movement targets referenced by smbclient activity.
7. Add confirmed IOCs to watchlists.
8. Deploy detections for systemd-launched implants and suspicious Linux syscall process activity.
9. Conduct post-containment validation scans.
10. Complete lessons learned and detection tuning.

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:F8FAFC,45:6B7280,100:050505&height=120&section=footer" />
</p>
