Data Loss Prevention (DLP) Guide
Domain: Data Loss Prevention
Level: Experienced, Enterprise Scope

---
Table of Contents
What is DLP?
Where DLP Controls Sit
Classifying Data
Building DLP Policies
Common Real-World Tickets
Interview Cheat Sheet
---
1. What is DLP?
Data Loss Prevention is a set of controls that detect and prevent sensitive data from leaving an organisation in unauthorised ways, whether by accident or intentionally.
The core question DLP answers is:
"Is sensitive data about to leave the organisation through this channel, and should it be allowed to?"
DLP is not one tool. It is a combination of policy, classification, and technical controls applied across several channels: email, web uploads, removable media, cloud storage, and endpoints.
Common drivers for DLP programmes:
Regulatory requirements (GDPR, PCI-DSS, HIPAA depending on sector)
ISO 27001 Annex A.8.12 (data leakage prevention) and A.5.34 (privacy and protection of personal data)
Protecting intellectual property and trade secrets
Insider threat mitigation
---
2. Where DLP Controls Sit
DLP can be applied at multiple points, often layered:
Layer	Example	What It Catches
Email Gateway	Microsoft Purview DLP, Proofpoint	Sensitive data sent as an attachment or in the email body
Endpoint	Endpoint DLP agent	Copying files to USB, printing, screen capture of sensitive documents
Cloud Apps (CASB)	Microsoft Defender for Cloud Apps, Netskope	Uploads to unsanctioned cloud storage, sharing links set to "Anyone with the link"
Network	Network DLP appliance	Sensitive data leaving over unencrypted protocols (FTP, plain HTTP)
Storage	Microsoft Purview, AWS Macie	Sensitive data sitting in repositories without correct access controls
A mature DLP programme covers multiple layers, since a single control point (e.g. only the email gateway) leaves other channels (USB, cloud uploads) uncovered.
---
3. Classifying Data
DLP policies only work if the organisation has a way of identifying what counts as sensitive. This usually means a data classification scheme:
Classification	Example	Typical Handling
Public	Marketing material, published reports	No restrictions
Internal	Internal memos, org charts	Internal distribution only
Confidential	Customer contracts, financial forecasts	Restricted distribution, encryption at rest
Highly Confidential / Restricted	Source code, M&A documents, health records, card data	Strict access controls, DLP enforcement, encryption in transit and at rest
How DLP Tools Identify Sensitive Data
Pattern matching: regular expressions for structured data like card numbers, National Insurance numbers, IBANs
Keyword and dictionary matching: terms like "Confidential", "NDA", project codenames
Fingerprinting: a hash or signature of a specific document, so any copy or near-copy is detected
Machine learning classifiers: trained to recognise document types such as contracts, resumes, or financial statements
Sensitivity labels: manually or automatically applied labels (e.g. Microsoft Information Protection labels) that DLP policies can act on directly
---
4. Building DLP Policies
A DLP policy generally defines:
Scope: which users, devices, or locations the policy applies to
Condition: what triggers the policy (e.g. content matches "card number" pattern, or has a "Highly Confidential" label)
Action: what happens when the condition is met
Exceptions: legitimate cases that should not trigger the policy
Example Policy: Card Data in Email
Field	Value
Scope	All users, outbound email
Condition	Email body or attachment contains content matching a card number pattern (PAN)
Action	Block the email, notify the user with the reason, notify the security team
Exception	Emails to the approved payment processor domain, where card data sharing is part of an approved business process
Policy Rollout Approach
Rolling out DLP in full block mode immediately tends to generate a high volume of false positives and user frustration. A staged approach is more common:
Audit mode: policy runs and logs matches, but does not block anything. Used to understand volume and false positive rate.
Notify mode: users are warned ("this email appears to contain card data, are you sure?") but can still proceed, often with a justification field.
Block mode: policy actively blocks the action, typically reserved for the highest-risk content types (e.g. card data, source code leaving via personal email).
---
5. Common Real-World Tickets
---
TICKET-DLP-001: Customer Database Export Attempted via Personal Email
Reported by: DLP policy alert, email gateway
Severity: High
Environment: Corporate email (Microsoft 365)
Problem
A DLP policy on the email gateway flagged an outbound email from an employee in the Sales team to their personal Gmail address, with an attachment identified as containing a large number of records matching the pattern for customer email addresses and phone numbers. The email was blocked by the policy, which was running in block mode for this content type.
Investigation
Reviewed the DLP alert details: attachment was an exported CSV from the CRM containing approximately 4,000 customer contact records
Spoke with the employee's manager: the employee was leaving the company in two weeks and had exported the data to "keep a copy of my contacts for my next role"
Checked whether any other exports had occurred recently from this account: found one earlier export attempt to the same personal address two days prior, which had also been blocked
No successful exfiltration occurred, both attempts were blocked by the DLP policy
Resolution
Confirmed with HR and the employee's manager that this was a policy violation, handled through HR process (this falls outside a pure technical remediation)
Reviewed the employee's CRM access and export permissions, export permissions were revoked immediately given the upcoming departure
Reviewed DLP logs for the past 90 days for this user to check for any other suspicious activity, none found
Flagged this as a case to include in the standard leaver checklist: export permissions should be reviewed or revoked as soon as a resignation is received, not just on the last working day
Lessons Learned
The DLP control worked exactly as intended and prevented data exfiltration in both attempts. The gap was procedural: access reduction for departing employees did not begin until their final week. This was added to the joiner/mover/leaver process so that sensitive export permissions are reviewed at the point a resignation is submitted, not just on exit.
---
TICKET-DLP-002: Sensitive Document Shared via Public Cloud Storage Link
Reported by: CASB alert, sharing link set to "Anyone with the link"
Severity: Medium
Environment: Cloud storage (SharePoint/OneDrive)
Problem
The CASB tool flagged a document labelled "Highly Confidential" (a draft financial forecast) that had been shared via a link set to "Anyone with the link can view", rather than restricted to named recipients or the organisation only.
Investigation
Identified the document owner and the sharing link creation event in the audit log
Contacted the owner: they had shared the document with an external auditor and selected the "Anyone with the link" option by mistake, intending to select "Specific people"
Checked link access logs: the link had been accessed twice, both from the same IP range associated with the external auditor's organisation, no other access detected
No evidence of broader exposure, but the link itself was technically accessible to anyone who obtained the URL during the period it was active
Resolution
Changed the sharing link to restrict access to the specific external auditor's email address only
Reviewed the document's sensitivity label settings: the label should have applied default sharing restrictions automatically but had not been configured to do so for this document type
Updated the sensitivity label policy so that "Highly Confidential" labelled documents cannot be shared via "Anyone with the link" links at all, this option is removed from the sharing menu for that label
Sent a reminder communication to staff about correct use of sharing options for confidential documents, with a short guide
Lessons Learned
User error caused the initial exposure, but the underlying control gap was that the sensitivity label did not technically prevent this sharing option from being selected. Fixing the label configuration removes the possibility of the same mistake recurring, rather than relying on user awareness alone.
---
6. Interview Cheat Sheet
Q: What is DLP and what problem does it solve?
A: DLP is a set of controls that detect and prevent sensitive data leaving an organisation through unauthorised channels, whether accidental or deliberate. It addresses the risk that data classified as confidential or restricted ends up somewhere it should not, such as a personal email account, an unsanctioned cloud service, or removable media.
Q: What channels should a DLP programme cover?
A: Email, endpoint (USB, printing, clipboard), cloud applications via a CASB, network traffic, and data at rest in storage repositories. Covering only one channel, such as email, leaves other paths for data to leave uncovered.
Q: How does a DLP tool know what counts as sensitive?
A: Through a combination of pattern matching for structured data like card numbers, keyword and dictionary matching, document fingerprinting for specific files, machine learning classifiers for document types, and sensitivity labels that can be applied manually or automatically and then referenced by DLP policies.
Q: How would you roll out a new DLP policy without disrupting the business?
A: Start in audit mode to understand the volume of matches and false positive rate without blocking anything. Move to notify mode, where users are warned but can proceed with justification. Reserve full block mode for the highest-risk content types once the policy has been tuned based on the audit and notify phases.
Q: An employee tried to email a customer database to their personal account and it was blocked. What's your response process?
A: Confirm the block worked and no data left. Review the context, who the employee is, whether they have a legitimate reason, and whether this is part of a pattern (check DLP logs for prior attempts). Involve HR if it appears to be a policy violation tied to departure or misconduct. Review and reduce the employee's data access where appropriate. Identify any procedural gap, such as access not being reduced early enough in the leaver process, and fix that gap going forward.
---
