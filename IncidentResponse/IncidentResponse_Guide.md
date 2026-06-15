Incident Response Guide
Domain: Incident Response
Level: Experienced, Enterprise Scope

---
Table of Contents
What is Incident Response?
The Incident Response Lifecycle
Severity Classification
Roles During an Incident
Common Real-World Tickets
Interview Cheat Sheet
---
1. What is Incident Response?
Incident response is the structured process an organisation follows when a security incident occurs: detecting it, understanding its scope and impact, containing it, removing the cause, restoring normal operation, and learning from it afterwards.
The core question incident response answers is:
"Something has gone wrong. What happened, how bad is it, and what do we do right now to limit the damage?"
A good IR process is defined and rehearsed before an incident happens. During an actual incident is not the time to be deciding who has authority to isolate a production server or who needs to be informed.
---
2. The Incident Response Lifecycle
A widely used model (based on NIST SP 800-61) breaks IR into phases:
```
Preparation
     |
     v
Detection and Analysis
     |
     v
Containment, Eradication, Recovery
     |
     v
Post-Incident Activity (Lessons Learned)
```
Preparation
Before any incident occurs: having logging and monitoring in place, defined playbooks for common scenarios, contact lists, access to forensic tools, and a tested communication plan.
Detection and Analysis
An alert fires, or a report comes in (from a user, a third party, or a monitoring tool). The analyst confirms whether this is a real incident, determines its scope (which systems, users, data are affected), and classifies its severity.
Containment, Eradication, Recovery
Containment: stop the incident from getting worse. Isolate affected systems, disable compromised accounts, block malicious IPs.
Eradication: remove the root cause. Delete malware, close the vulnerability that was exploited, remove unauthorised access.
Recovery: restore affected systems to normal operation, with monitoring in place to confirm the issue does not recur.
Post-Incident Activity
A lessons learned review, usually within a week or two of the incident closing. This looks at what happened, what worked well, what did not, and what changes (technical, procedural) should be made to prevent recurrence or improve response next time.
---
3. Severity Classification
Classifying severity early helps determine who needs to be involved and how urgently.
Severity	Example	Typical Response
Critical (P1)	Active ransomware encrypting production systems, confirmed large-scale data breach	Immediate, all-hands response, executive notification, possible regulatory notification
High (P2)	Compromised privileged account, malware on a server with sensitive data, but contained	Urgent response within the working day, security team leads, management informed
Medium (P3)	Malware on a single workstation, contained and not yet spread	Standard response within agreed SLA, handled by security operations
Low (P4)	Policy violation with no evidence of compromise, isolated phishing click with no payload execution	Logged and reviewed, may inform awareness training
Severity can change as an incident is investigated. A ticket that starts as Medium might be escalated to High if scope turns out to be larger than initially thought, or de-escalated if the impact is found to be smaller.
---
4. Roles During an Incident
For anything above a Low severity incident, having clear roles avoids confusion:
Role	Responsibility
Incident Commander	Coordinates the overall response, makes key decisions, manages communication
Technical Lead / Analyst	Performs investigation, containment, and eradication actions
Communications Lead	Manages internal updates and, if needed, external/customer/regulatory communication
Scribe	Records a timeline of actions taken, decisions made, and evidence collected, important for both the post-incident review and any legal/regulatory follow-up
For smaller organisations, one person may hold multiple roles, but it should still be clear who is doing what during the incident rather than everyone acting independently.
---
5. Common Real-World Tickets
---
TICKET-IR-001: Phishing Email Leads to Credential Compromise
Reported by: User report, "I think I clicked something I shouldn't have"
Severity: High (initially), de-escalated to Medium after investigation
Environment: Microsoft 365 corporate environment
Problem
A user reported that they had clicked a link in an email that appeared to be from IT support, asking them to verify their account by logging in. The page looked like the normal Microsoft login page. The user entered their username and password, then noticed the page did not look quite right and closed it.
Detection and Analysis
Confirmed the email was a phishing attempt: the link led to a credential harvesting page hosted on a domain registered the day before
Checked the user's account sign-in logs: a sign-in had occurred from an unfamiliar location shortly after the user entered their credentials on the phishing page, approximately 6 minutes before the user reported the incident
The sign-in from the unfamiliar location had access to the user's mailbox briefly: logs showed it accessed the inbox but no evidence of mailbox rule changes, forwarding rules, or mass email access
Containment
Disabled the user's account immediately to stop any further access
Revoked all active sessions and tokens for the account, forcing re-authentication everywhere
Reset the user's password
Eradication and Recovery
Reviewed the account for any changes made during the brief unauthorised access: no mailbox rules, no forwarding rules, no changes to MFA settings, no other accounts accessed from the same session
Re-enabled the account with a new password and confirmed MFA was correctly configured (it had not been bypassed, the attacker only obtained the password, MFA prompt had not been completed since the user closed the page before continuing)
Searched for the phishing email across all mailboxes: identified 11 other recipients of the same email, removed it from their mailboxes before it was opened by anyone else
Blocked the phishing domain at the email gateway and web proxy
Lessons Learned
The user reporting the issue quickly was the most important factor, it limited the unauthorised access window to about 6 minutes and meant the attacker only got as far as briefly viewing the inbox before the account was disabled. MFA being enabled meant the stolen password alone was not enough to fully take over the account; the attacker would have needed to also complete an MFA prompt, which they had not managed before access was cut off. This was used as a positive example in security awareness training, framed around "reporting quickly is exactly the right response and made a real difference here."
---
TICKET-IR-002: Ransomware Detected on File Server, Contained Before Encryption Completed
Reported by: EDR alert, mass file modification pattern detected
Severity: Critical
Environment: On-premises file server
Problem
EDR raised a critical alert for a file server, detecting a process rapidly renaming and modifying a large number of files in a shared folder, a pattern consistent with ransomware encryption activity.
Detection and Analysis
EDR had automatically isolated the file server from the network as part of its automated response policy for this detection pattern, within seconds of the pattern being identified
The Incident Commander role was activated given the Critical severity, technical lead, communications lead, and scribe roles were assigned
Reviewed the affected shared folder: approximately 1,200 files had been renamed with a new extension and appeared encrypted, out of an estimated 85,000 files on that share. The automated isolation had stopped the process partway through
Identified the source: the malicious process had been launched from a service account used by a backup application, which had recently had its credentials changed and the new credentials were found, in plaintext, in a configuration file that had broader read access than intended
Reviewed how the service account's credentials might have been obtained: the configuration file was on a file share that, due to a misconfigured permission inherited from a parent folder, was readable by a much larger group of users than intended
Containment
File server remained isolated (already done automatically by EDR)
Disabled the backup service account while investigation continued
Reviewed other systems for the same malicious process or related indicators: none found on other servers, this appeared to be contained to the one file server
Eradication and Recovery
Identified and removed the malicious binary and its persistence mechanism from the file server
Restored the approximately 1,200 affected files from the previous night's backup, verified backup integrity before restoring
Reset the backup service account credentials and corrected the file share permissions so the configuration file was no longer broadly readable
Reviewed all service accounts for similarly exposed credentials in configuration files across the environment, found and remediated 2 additional cases
Returned the file server to the network after a full scan confirmed no remaining indicators of compromise
Lessons Learned
The automated EDR isolation was the single most important factor in limiting impact, roughly 1,200 files affected out of 85,000 rather than the entire share. The root cause was a permissions misconfiguration that allowed broad read access to a file containing service account credentials, this was a pre-existing weakness that had not been identified by previous access reviews because the review process focused on user accounts rather than service account credential exposure. A new check was added to the access review process specifically looking for credentials or secrets stored in configuration files on broadly accessible shares.
---
6. Interview Cheat Sheet
Q: Walk me through the incident response lifecycle.
A: Preparation happens before anything occurs: logging, monitoring, playbooks, and contact lists in place. Detection and analysis is where an alert or report is confirmed as a real incident and its scope and severity are determined. Containment, eradication, and recovery cover stopping the incident from spreading, removing the root cause, and restoring normal operation. Post-incident activity is a lessons learned review to improve for next time.
Q: How do you decide the severity of an incident?
A: Based on impact and scope: is it affecting production systems, is sensitive data involved, is it contained to one system or spreading, and could it require executive or regulatory notification. Severity can change as more information becomes available during the investigation.
Q: A user reports they clicked a phishing link and entered their password. What do you do first?
A: Containment first: disable the account, revoke active sessions and tokens, and reset the password, to cut off any access the attacker may have gained. Then analyse sign-in logs to see if the credentials were actually used and what was accessed during that window. Then check whether MFA prevented full takeover, search for and remove the phishing email from other mailboxes, and block the phishing domain.
Q: What roles are useful to define for incident response?
A: An Incident Commander to coordinate and make decisions, a technical lead to perform investigation and containment, a communications lead to manage updates internally and externally, and a scribe to record a timeline of actions and evidence. In smaller teams one person may cover multiple roles, but the responsibilities should still be clearly assigned.
Q: Why is the post-incident review important, and what should come out of it?
A: It is where the organisation turns an incident into improvements: identifying what control gaps allowed the incident to happen or to have the impact it did, and converting those into specific actions, such as configuration changes, new detection rules, or process updates. Without this step, the same root cause can lead to a repeat incident.
---
