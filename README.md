# Product Security Portfolio-Ashish Mukherjee 

This repository is a practical portfolio of cybersecurity control implementations, incident write-ups, and threat models — built to demonstrate hands-on understanding of how security controls operate in real organisations, beyond audit/compliance documentation.

Each domain folder contains:
- A **conceptual guide** explaining how the control area works at an organisation-wide / enterprise scale
- **Realistic ticket write-ups** modelled on real SOC/security operations scenarios (Problem → Investigation → Root Cause → Resolution → Lessons Learned)
- Supporting **policy files / configs** where relevant

---

##  Repository Structure

\```
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
├── DLP/
├── Endpoint/
└── IncidentResponse/
\```

---



### IAM (Identity & Access Management)
- [AWS IAM Organisation-Wide Guide](IAM/AWS_IAM_OrganisationWide_Guide.md) — AWS Organisations, SCPs, IAM Identity Centre, least privilege at scale, PAM/JIT access
- 3 realistic incident tickets covering excessive privileges, leaked credentials, and cross-account trust misconfigurations
- 3 example SCP guardrail policies (JSON)

### Product Security
- [Smart Lock STRIDE Threat Model](ProductSecurity/threat-models/PRODUCT-SEC-001-SmartLock-STRIDE-ThreatModel.md) — full STRIDE analysis of an IoT device covering hardware, firmware, BLE, cloud API, and mobile app, mapped to IEC 62443, ISO/IEC 27400, and OWASP IoT Top 10

---

## 

- Firewalls & Network Security
- DLP (Data Loss Prevention)
- Endpoint Security
- Incident Response

---
