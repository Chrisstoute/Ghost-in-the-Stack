<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0B1220,50:334155,100:DC2626&height=170&section=header&text=Ghost%20in%20the%20Stack&fontSize=42&fontColor=ffffff&animation=fadeIn" />

# Ghost in the Stack — Advanced Threat Hunt

![Threat Hunting](https://img.shields.io/badge/Threat%20Hunting-Advanced-red?style=for-the-badge)
![Microsoft Sentinel](https://img.shields.io/badge/Microsoft%20Sentinel-KQL-blue?style=for-the-badge)
![Microsoft Defender](https://img.shields.io/badge/Microsoft%20Defender-Advanced%20Hunting-0078D4?style=for-the-badge)
![Incident Response](https://img.shields.io/badge/Incident%20Response-Live%20Threat-orange?style=for-the-badge)
![Linux](https://img.shields.io/badge/Linux-Telemetry-black?style=for-the-badge)

## Overview

**Ghost in the Stack** is an advanced threat hunting and incident response investigation focused on reconstructing a multi-phase Linux intrusion against `GF-DEV01`.

The investigation followed attacker activity across authentication logs, process telemetry, file telemetry, Linux shell history, network events, persistence artifacts, and service state evidence. The goal was to identify the operator’s path from initial access through implant deployment, persistence, lateral movement preparation, payload staging, and live-threat containment planning.

This project demonstrates practical SOC and threat hunting skills using KQL, timeline reconstruction, evidence validation, detection engineering, and incident response decision-making.

---

## Investigation Focus

The hunt centered on answering the following:

- Which account was compromised?
- How was the implant downloaded and launched?
- What process chain confirmed the implant execution path?
- What persistence mechanisms were created?
- What tools and watch keys did the implant interact with?
- What infrastructure did the operator use?
- What artifacts needed containment and cleanup?
- What detection logic could identify this behavior in the future?
- What live threat state remained on the host at end of shift?

---

## Tools and Data Sources

| Category | Tools / Tables |
|---|---|
| SIEM / Hunting | Microsoft Sentinel, Defender Advanced Hunting |
| Query Language | KQL |
| Host Telemetry | LinuxProcess_CL, LinuxFile_CL, LinuxAuth_CL, LinuxNetwork_CL, LinuxShellHistory_CL |
| Investigation Artifacts | Process telemetry, file events, shell history, authentication logs, network events |
| Detection Engineering | Sigma-style rule design |
| Incident Response | Containment scoping, artifact cleanup planning, live threat brief |

---

## Key Findings

### Compromised Host

The affected Linux host was:

```text
GF-DEV01
```

### Compromised / Abused User Context

The investigation identified attacker activity involving the `sancadmin` account during the later intrusion phase.

### Implant Execution

The implant was launched as:

```text
/tmp/helix-update
```

The validated process relationship showed the implant running with:

```text
PID: 34616
PPID: 1
```

This supported that the implant detached from the original shell and became its own long-running process.

### External Access

The first external SSH activity for `sancadmin` came from:

```text
194.36.110.139
```

### Dwell Time

The calculated dwell time from implant detach to first external `sancadmin` SSH was:

```text
104 minutes
```

### Network / Infrastructure Finding

The `install.sh` download activity showed curl traffic to:

```text
104.21.57.185:443
```

OSINT enrichment showed the destination was associated with Cloudflare infrastructure, supporting the finding that blocking the IP directly would be risky because the adversary was hiding behind fronted CDN infrastructure.

### Watchlist / IOC Finding

The late-session operator probe identified the infrastructure IOC:

```text
194.36.110.139:9080
```

---

## Detection Engineering

A Sigma-style detection title for the implant launch pattern:

```text
sliver implant launched via systemd
```

The matching Linux logsource service was based on syscall-level exec telemetry captured through kernel watch keys.

---

## Live Threat Brief

At end of shift, the live-threat state was summarized across three layers:

```text
authorized_keys:persistent, helix-sync.service:dormant, helix-update:running
```

This separated the remaining artifacts into:

- **File telemetry:** persistent SSH key material
- **Service state:** dormant systemd service artifact
- **Process telemetry:** running implant process

---

## Repository Structure

```text
Ghost-in-the-Stack/
│
├── README.md
├── Walkthrough.md
│
├── docs/
│   ├── walkthrough.md
│   ├── findings-summary.md
│   └── lessons-learned.md
│
├── queries/
│
├── reports/
│
└── Screenshots/
    ├── 1_Linux_Authentication_Table.jpg
    ├── 2_Implant_Download_Defender_Process_Tree_pt_1.jpg
    ├── 3_Defender_Process_Tree_Partial_Installer_Actions_pt_1.jpg
    ├── 4_PID_Chain_Query_Broad_Filter_pt_1 (1).jpg
    ├── 7_Helix_Update_PID_PPID_Validation.jpg
    ├── 26_Dwell_Time_104_Minutes_Calculation.jpg
    ├── 28_DEV01_Containment_Cleanup_Paths.jpg
    ├── 29_InstallSH_Curl_To_Cloudflare_Fronted_IP.jpg
    ├── 30_Late_Session_C2_Watchlist_IOC.jpg
    ├── 41_Process_Telemetry_Helix_Update_Running_pt_1.jpg
    ├── 41_Service_Persistence_Helix_Sync_Service_pt_2.jpg
    └── 41_File_Telemetry_Hbsync_Dormant_pt_3.jpg
```

---

## Screenshots

### Implant PID / PPID Validation

![Helix Update PID PPID Validation](Screenshots/7_Helix_Update_PID_PPID_Validation.jpg)

### Dwell Time Calculation

![Dwell Time 104 Minutes](Screenshots/26_Dwell_Time_104_Minutes_Calculation.jpg)

### Containment Cleanup Paths

![Containment Cleanup Paths](Screenshots/28_DEV01_Containment_Cleanup_Paths.jpg)

### Install Script Cloudflare-Fronted Destination

![Cloudflare Fronted IP](Screenshots/29_InstallSH_Curl_To_Cloudflare_Fronted_IP.jpg)

### Late-Session IOC Extraction

![Late Session C2 Watchlist IOC](Screenshots/30_Late_Session_C2_Watchlist_IOC.jpg)

### Live Threat Brief Evidence

![Process Telemetry Helix Update Running](Screenshots/41_Process_Telemetry_Helix_Update_Running_pt_1.jpg)

![Service Persistence Helix Sync Service](Screenshots/41_Service_Persistence_Helix_Sync_Service_pt_2.jpg)

![File Telemetry Hbsync Dormant](Screenshots/41_File_Telemetry_Hbsync_Dormant_pt_3.jpg)

---

## Skills Demonstrated

- Advanced KQL hunting
- Process chain reconstruction
- Linux authentication analysis
- Linux shell history analysis
- File telemetry investigation
- Network telemetry enrichment
- OSINT-driven infrastructure analysis
- IOC extraction
- Detection engineering
- Sigma-style rule naming
- Incident response containment scoping
- Evidence-based reporting
- Live threat briefing

---

## Lessons Learned

This investigation reinforced the importance of validating findings across multiple telemetry sources. A single table rarely tells the full story. The strongest conclusions came from correlating process events, shell history, file creation events, service activity, authentication logs, and network telemetry.

It also highlighted why infrastructure enrichment matters. Some destinations may resolve to legitimate shared services or CDN providers, meaning blocking an IP at the perimeter may create operational risk without actually stopping the adversary.

---

## Final Summary

The Ghost in the Stack investigation identified a live Linux implant, validated its process lineage, reconstructed attacker behavior, extracted IOCs, scoped persistence, and produced containment-focused findings. The hunt moved from raw telemetry to actionable incident response decisions and detection engineering recommendations.

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:DC2626,50:334155,100:0B1220&height=120&section=footer" />
