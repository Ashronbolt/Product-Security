Endpoint Security Guide
Domain: Endpoint Security
Level: Experienced, Enterprise Scope

---
Table of Contents
What is Endpoint Security?
Core Endpoint Controls
EDR: How Detection and Response Works
Patch and Vulnerability Management
Common Real-World Tickets
Interview Cheat Sheet
---
1. What is Endpoint Security?
Endpoints are the devices where users actually work: laptops, desktops, servers, and increasingly mobile devices. They are also the most common initial point of compromise, since they are where users open email attachments, browse the web, and run applications.
The core question endpoint security answers is:
"If this device is compromised, how quickly can we detect it, and how do we limit what the attacker can do next?"
Endpoint security is not just antivirus. A modern endpoint security stack includes prevention, detection, response, and configuration controls working together.
---
2. Core Endpoint Controls
Control	Purpose
Antivirus / Anti-malware	Signature and heuristic-based detection of known malicious files
EDR (Endpoint Detection and Response)	Behavioural monitoring, alerting, and response capability (isolate, investigate, remediate)
Host-based firewall	Controls inbound/outbound connections at the device level
Disk encryption	Protects data at rest if the device is lost or stolen (BitLocker, FileVault)
Application control / allowlisting	Restricts which applications can execute
Patch management	Keeps OS and applications updated against known vulnerabilities
Mobile Device Management (MDM)	Enforces configuration baselines, encryption, remote wipe on mobile devices
USB/removable media control	Restricts or blocks data transfer via USB devices
Defence in Depth at the Endpoint
No single control catches everything. A phishing email might bypass the email gateway, the malicious attachment might not match an antivirus signature, but EDR behavioural detection might catch the process trying to establish a connection to a C2 server, and application allowlisting might prevent the payload from executing in the first place. Each layer reduces the chance that an attack succeeds all the way through.
---
3. EDR: How Detection and Response Works
EDR agents run on endpoints and continuously collect telemetry: process creation, network connections, file changes, registry changes, and more. This telemetry is sent to a central platform (cloud-hosted in most modern EDR products) where it is analysed against detection rules and behavioural models.
Typical EDR Workflow
```
Endpoint Activity (process runs, file written, network connection made)
        |
        v
EDR Agent collects telemetry
        |
        v
Cloud EDR Platform analyses against detection rules
        |
        v
Alert generated if suspicious pattern matches
        |
        v
Analyst investigates: timeline, related processes, network connections
        |
        v
Response action: isolate host, kill process, quarantine file, or close as benign
```
Common Detection Patterns
A process spawning from an unusual parent (e.g. Microsoft Word spawning PowerShell)
PowerShell or command-line activity with obfuscated or encoded commands
A process attempting to disable security tools or modify startup configuration
Outbound connections to IPs or domains on threat intelligence feeds
Lateral movement indicators, such as use of remote admin tools (PsExec, WMI) from a workstation that does not normally use them
Host Isolation
When EDR confirms a host is compromised, isolation is often the first response action. The EDR agent blocks all network traffic except the connection back to the EDR management console, so the analyst can still investigate and remediate while the host cannot communicate with the rest of the network or the internet.
---
4. Patch and Vulnerability Management
Unpatched software is one of the most common ways attackers gain initial access or escalate privileges. A patch management programme typically includes:
Vulnerability scanning: regular scans (e.g. weekly) identify missing patches and known vulnerabilities (CVEs) across the endpoint estate
Risk-based prioritisation: not every vulnerability needs the same urgency. Factors include CVSS score, whether the vulnerability is actively exploited in the wild, and whether the affected system is internet-facing
Patch deployment: patches are tested in a staging group before wider rollout, then deployed in waves
Verification: a follow-up scan confirms the patch was applied successfully
SLA Example
Severity	Patch SLA
Critical, actively exploited	24-72 hours
Critical, not yet exploited	7 days
High	14 days
Medium	30 days
Low	Next scheduled patch cycle
---
5. Common Real-World Tickets
---
TICKET-EP-001: EDR Alert for Suspicious PowerShell Activity on Finance Workstation
Reported by: EDR platform, behavioural detection
Severity: High
Environment: Corporate laptop, Finance department
Problem
EDR raised an alert for a Finance department laptop showing PowerShell launched as a child process of Microsoft Excel, executing an encoded command. This pattern is commonly associated with malicious macros.
Investigation
Reviewed the EDR process tree: Excel opened a file named "Invoice_Q3_Statement.xlsm", which spawned PowerShell with a base64-encoded argument
Decoded the PowerShell command: it attempted to download a file from an external URL and execute it
Checked EDR network telemetry: the download attempt was blocked by the host firewall's outbound rules, the destination IP was not on any allowlist and the connection was refused
Reviewed email logs: the spreadsheet had arrived as an attachment from an external sender that morning, the user had opened it and enabled macros when prompted
Resolution
Isolated the laptop via EDR while investigation continued
Confirmed no successful payload download occurred (network block was effective)
Ran a full scan and reviewed persistence locations (scheduled tasks, registry run keys, startup folder), none found
Returned the laptop to normal operation after confirming no compromise occurred
Blocked the sender's domain at the email gateway
Reviewed macro execution settings organisation-wide: macros from the internet were not blocked by default, this was identified as a configuration gap
Lessons Learned
The host firewall's default-deny outbound policy prevented this from becoming a full compromise even though the user opened the file and enabled macros. The longer-term fix was a policy change: block macros in files originating from the internet by default (a setting available in Microsoft Office), which removes this entire attack pattern rather than relying on the user not enabling macros or the firewall catching the follow-on connection.
---
TICKET-EP-002: Critical Vulnerability Actively Exploited, Patch Deployment Tracking
Reported by: Vulnerability management team, vendor advisory plus active exploitation reports
Severity: Critical
Environment: Server estate, mixed Windows and Linux
Problem
A critical remote code execution vulnerability was disclosed in a widely used component, with public reports of active exploitation within days of disclosure. The organisation needed to identify affected systems and patch within the 24-72 hour SLA for critical, actively exploited vulnerabilities.
Investigation
Ran an emergency vulnerability scan against the server estate to identify systems with the vulnerable component installed
Identified 23 affected servers: 18 internal application servers, 5 internet-facing servers
Prioritised the 5 internet-facing servers as the highest risk, since the vulnerability was being exploited via internet-accessible services
Checked whether a vendor patch was available: yes, released alongside the advisory
Checked for a temporary mitigation in case patching could not be completed immediately: a configuration change was available that disabled the vulnerable feature with minimal impact to the affected applications
Resolution
Applied the temporary mitigation to all 23 servers within 4 hours, reducing immediate risk while patches were prepared
Patched the 5 internet-facing servers within 24 hours, following emergency change approval (out-of-cycle given the severity)
Patched the remaining 18 internal servers within the 72-hour SLA window, in two waves with verification scans between each
Ran a full scan after all patches were applied to confirm no systems remained vulnerable
Reviewed firewall and IDS logs for the period between disclosure and patching for any signs of exploitation attempts against the organisation's systems, none found
Lessons Learned
Having a documented temporary mitigation step available bought time to patch in a controlled way rather than rushing emergency changes onto all 23 servers simultaneously. The emergency change process worked as designed for the internet-facing systems. One gap identified: the vulnerability scanner's asset inventory was missing 2 of the 23 affected servers initially, they were found through a manual cross-check against the CMDB. Asset inventory accuracy was raised as a follow-up item.
---
6. Interview Cheat Sheet
Q: What's the difference between antivirus and EDR?
A: Antivirus primarily relies on signatures and known-bad file detection. EDR focuses on behaviour: it monitors process activity, network connections, and system changes over time, so it can detect malicious activity even from files that have never been seen before, and gives analysts the ability to investigate and respond (isolate a host, kill a process) directly.
Q: A laptop has been flagged by EDR for suspicious activity. What's your immediate process?
A: Isolate the host via EDR to stop any further communication while preserving the ability to investigate. Review the process tree and telemetry to understand what happened and whether any payload executed successfully. Check whether other controls (firewall, email gateway) already blocked part of the chain. Remediate if needed (clean, reimage). Identify any configuration gap that allowed the initial step (e.g. macros enabled by default) and fix it so the same pattern does not succeed again.
Q: How do you prioritise vulnerability patching when you can't patch everything at once?
A: Use a risk-based approach: consider the CVSS severity, whether the vulnerability is being actively exploited in the wild, and whether the affected system is internet-facing or holds sensitive data. Internet-facing systems with actively exploited critical vulnerabilities get patched first, often within 24-72 hours, while lower severity or internal-only systems follow longer SLAs.
Q: What's host isolation and why is it useful?
A: Host isolation is an EDR response action that blocks all network traffic from a device except the connection to the EDR management console. It stops a compromised device from communicating with attackers or moving laterally, while still allowing analysts to investigate and remediate remotely.
Q: What layers of defence might prevent a malicious email attachment from compromising a device?
A: Email gateway scanning and sandboxing, macro execution policies (blocking macros from internet-sourced files), antivirus scanning the file on write, application allowlisting preventing unexpected processes from running, host firewall blocking outbound connections to unknown destinations, and EDR behavioural detection catching the activity if earlier layers are bypassed.
---
