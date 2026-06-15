🛡️ Product Security Portfolio 

This repository is a practical portfolio of cybersecurity control implementations, incident write-ups, and threat models. It demonstrates hands-on understanding of how security controls operate in real organisations, beyond audit and compliance documentation.
Each domain folder contains:
A conceptual guide explaining how the control area works at an organisation-wide / enterprise scale
Realistic ticket write-ups modelled on real SOC/security operations scenarios (Problem → Investigation → Root Cause → Resolution → Lessons Learned)
Supporting policy files / configs where relevant
---
📂 Repository Structure
```
product-security-portfolio/
├── README.md
├── IAM/
│   ├── AWS_IAM_OrganisationWide_Guide.md
│   ├── tickets/
│   │   ├── IAM-001-ExcessivePrivileges.md
│   │   ├── IAM-002-ExposedAccessKey.md
│   │   └── IAM-003-CrossAccountRoleMisconfiguration.md
│   └── policies/
│       ├── scp-deny-root-usage.json
│       ├── scp-restrict-regions.json
│       └── scp-deny-disable-security-services.json
├── ProductSecurity/
│   └── threat-models/
│       └── PRODUCT-SEC-001-SmartLock-STRIDE-ThreatModel.md
├── Firewalls/
│   └── Firewalls_NetworkSecurity_Guide.md
├── DLP/
│   └── DLP_Guide.md
├── Endpoint/
│   └── Endpoint_Security_Guide.md
└── IncidentResponse/
    └── IncidentResponse_Guide.md
```
---
Completed
IAM (Identity & Access Management)
AWS IAM Organisation-Wide Guide: AWS Organisations, SCPs, IAM Identity Centre, least privilege at scale, PAM/JIT access
3 realistic incident tickets covering excessive privileges, leaked credentials, and cross-account trust misconfigurations
3 example SCP guardrail policies (JSON)
Product Security
Smart Lock STRIDE Threat Model: full STRIDE analysis of an IoT device covering hardware, firmware, BLE, cloud API, and mobile app, mapped to IEC 62443, ISO/IEC 27400, and OWASP IoT Top 10
Firewalls and Network Security
Firewalls and Network Security Guide: segmentation, firewall types, rule design, plus 2 tickets covering a stale firewall rule and a malware C2 connection
Data Loss Prevention (DLP)
DLP Guide: DLP layers, data classification, policy rollout stages, plus 2 tickets covering a data export attempt by a departing employee and a misconfigured sharing link
Endpoint Security
Endpoint Security Guide: core endpoint controls, EDR workflow, patch management, plus 2 tickets covering a macro-based attack and a critical vulnerability patch rollout
Incident Response
Incident Response Guide: IR lifecycle, severity classification, IR roles, plus 2 tickets covering a phishing credential compromise and a ransomware containment
---
