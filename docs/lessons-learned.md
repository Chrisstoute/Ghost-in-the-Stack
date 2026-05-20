# Lessons Learned

## Overview

This project reinforced how a real threat hunt is rarely solved from one table or one alert. The investigation required pivoting across authentication logs, process telemetry, file telemetry, network data, syscall records, service state, raw event fields, and OSINT enrichment.

The biggest lesson was that advanced threat hunting is not only about finding suspicious activity. It is about proving what happened, narrowing noisy data into defensible findings, and translating those findings into containment and detection actions.

## What Went Well

### 1. Strong Pivoting Across Multiple Data Sources

The investigation successfully moved across several telemetry sources:

- `LinuxAuth_CL` for SSH authentication activity
- `LinuxProcess_CL` for process execution and parent-child relationships
- `LinuxFile_CL` for file creation and staged artifacts
- `LinuxNetwork_CL` for destination IPs and ports
- `LinuxSyscall_CL` for audit and watch-key based Linux activity
- Raw packed telemetry for fields not immediately exposed in clean columns

This made it possible to reconstruct the attack path even when one table alone did not provide the full answer.

### 2. Process Lineage Was Critical

PID and PPID analysis helped confirm that `helix-update` detached and became a long-running implant process. This was one of the most important pieces of evidence because it changed the case from historical suspicious activity to a live threat.

### 3. KQL Iteration Improved the Investigation

The hunt required many query revisions. Early queries were often too broad, returned too much data, or focused on the wrong timestamp range. As the investigation progressed, the KQL became cleaner and more targeted.

The most useful improvements were:

- Using `project` to reduce noisy columns
- Using `has_any` to quickly scope to known artifacts
- Using `column_ifexists()` when table schemas differed
- Expanding the time window when expected results were missing
- Breaking large questions into smaller queries

### 4. OSINT Helped Explain Network Findings

The Cloudflare-fronted destination showed why enrichment matters. The destination IP alone did not fully explain the risk or response decision. OSINT helped show that blocking only the IP would be unreliable because the infrastructure was fronted behind a legitimate provider.

### 5. Detection Engineering Required the Right Abstraction

The Sigma-style rule was not about naming one exact file only. It required describing the behavior in a way that matched the framework, implant, action, and mechanism:

```text
sliver implant launched via systemd
```

This reinforced that good detections should describe behaviors and mechanisms, not just single indicators.

## Challenges

### 1. Too Much Raw Telemetry

Raw events were useful, but they were also overwhelming. Searching raw data without good filters created too much noise. The investigation became easier once raw searches were used only when necessary and paired with focused keywords or extracted fields.

### 2. Time Windows Were Easy to Misread

Several questions depended on exact timestamps. The alert UI displayed times that did not always match the Advanced Hunting tables because of ingestion delay. The correct approach was to rely on Advanced Hunting telemetry and calculate time differences directly from the source tables.

### 3. Naming and Formatting Were Strict

Several answers failed even when the concept was close because the required format was strict. This was especially true for:

- Artifact names
- State labels
- IP:port formatting
- Sigma-style rule naming
- Full absolute paths

For future work, the answer format should be treated as part of the technical requirement, not just a submission detail.

### 4. Similar Artifacts Created Confusion

Artifacts such as `helix-update`, `helix-sync`, `helix-sync.service`, `hbsync.exe`, and `wmi_exec.py` were easy to mix up. The investigation improved once each artifact was tied to a layer:

- Process telemetry
- File telemetry
- Service state
- Persistence
- Staging

## Key Technical Takeaways

### Live Threat Triage

A live implant should immediately move the case to incident response containment. Further investigation is still valuable, but containment becomes the priority once active compromise is confirmed.

### Process Telemetry

Process data can prove execution, parent-child relationships, detached processes, and live implant behavior.

### File Telemetry

File creation records can prove staged payloads, dropped tools, and dormant artifacts.

### Service State

Systemd activity can prove persistence attempts, service enablement, and service start/status behavior.

### Syscall Telemetry

Syscall and audit data can reveal low-level Linux activity that may not be obvious in cleaner process logs.

### OSINT Enrichment

IP ownership and provider context can explain whether an IP is attacker-owned, shared infrastructure, CDN-fronted, or otherwise unsuitable for simple perimeter blocking.

### Detection Engineering

Good detections should be behavior-focused. A useful Sigma-style title should be clear, descriptive, and tied to the mechanism used by the attacker.

## What I Would Do Differently Next Time

1. Start with a simple evidence tracker for each artifact.
2. Record every confirmed artifact with its source table and timestamp.
3. Keep a separate list of incorrect assumptions to avoid retrying them.
4. Normalize timestamps early in the investigation.
5. Avoid starting with broad raw searches unless the structured tables fail.
6. Capture proof screenshots immediately after solving each phase.
7. Save KQL queries alongside screenshots for easier walkthrough writing.
8. Separate findings by system layer: process, file, network, auth, service, syscall.
9. Validate answer formatting before submitting.
10. Use OSINT earlier when a question asks why an IP cannot simply be blocked.

## Skills Practiced

- Threat hunting methodology
- KQL query building
- Linux process analysis
- SSH authentication analysis
- PID and PPID validation
- File creation analysis
- Network enrichment
- Syscall and audit log interpretation
- Persistence analysis
- Incident response triage
- Detection engineering
- Sigma-style rule thinking
- Evidence-based reporting
- GitHub documentation

## Final Reflection

This hunt was challenging because it required both technical investigation and careful interpretation. The most important improvement was learning how to turn noisy telemetry into clear findings that support incident response decisions.

The project strengthened my ability to investigate live Linux threats, validate suspicious activity with evidence, and document the investigation in a way that can be used by SOC, incident response, and detection engineering teams.
