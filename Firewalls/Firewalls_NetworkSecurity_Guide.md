Firewalls and Network Security Guide
Domain: Network Security
Level: Experienced, Enterprise Scope

---
Table of Contents
What is Network Security at Organisation Scale?
Network Segmentation
Firewall Types and Where They Sit
Firewall Rule Design
Common Real-World Tickets
Interview Cheat Sheet
---
1. What is Network Security at Organisation Scale?
At a small scale, network security might mean one firewall at the office edge. At enterprise scale, the network is made up of many zones, each with different trust levels, and firewalls sit at every boundary between them.
The core question network security answers is:
"What traffic is allowed to move between which parts of the network, and why?"
Getting this wrong leads to:
Lateral movement, where a compromised system in one zone can reach sensitive systems in another
Data exfiltration over allowed but unmonitored channels
Compliance failures (PCI-DSS, ISO 27001 Annex A.8.20-8.22 on networks)
---
2. Network Segmentation
Segmentation means dividing the network into zones based on trust level and function, so a breach in one zone does not automatically give access to others.
```
Internet
   |
[Perimeter Firewall]
   |
DMZ (public-facing web servers, reverse proxies)
   |
[Internal Firewall]
   |
Internal Network
   |--- Corporate Zone (user workstations, email, file shares)
   |--- Server Zone (application servers, databases)
   |--- Management Zone (admin access, jump hosts, monitoring tools)
   |--- OT/IoT Zone (if applicable: cameras, badge readers, sensors)
```
Why This Matters
If a workstation in the Corporate Zone is compromised by phishing, the attacker should not be able to directly reach the database in the Server Zone. A firewall between zones, with rules restricting traffic to only what is needed (e.g. application servers to database on port 1433 only), limits the blast radius.
Micro-Segmentation
In cloud environments, segmentation often happens at a much finer level using security groups or network security groups, where each workload has its own rule set rather than relying on a few large network zones. This is common in AWS (Security Groups), Azure (NSGs), and GCP (firewall rules tied to VPC).
---
3. Firewall Types and Where They Sit
Type	Description	Typical Placement
Perimeter / Edge Firewall	Controls traffic between the organisation and the internet	Network edge
Internal / Segmentation Firewall	Controls traffic between internal zones	Between Corporate, Server, Management zones
Web Application Firewall (WAF)	Inspects HTTP/HTTPS traffic for application-layer attacks (SQLi, XSS)	In front of public-facing web applications
Host-Based Firewall	Runs on individual servers/endpoints	Every server and workstation
Cloud Security Groups / NSGs	Stateful firewall rules attached to cloud resources	AWS EC2, Azure VMs, GCP Compute
Next-Generation Firewalls (NGFW)
Modern firewalls (Palo Alto, Fortinet, Check Point) go beyond port and protocol filtering. They can:
Identify the application generating traffic, not just the port (App-ID)
Inspect encrypted traffic via SSL/TLS decryption
Apply intrusion prevention (IPS) signatures
Enforce user-based policies (tied to Active Directory groups, not just IP addresses)
---
4. Firewall Rule Design
The Default Deny Principle
Every firewall should start from a default deny stance: block everything, then explicitly allow only what is needed. This is the inverse of "allow everything except known bad", which is far weaker.
Rule Structure
A firewall rule typically defines:
Source (where traffic comes from: IP, subnet, zone)
Destination (where traffic is going: IP, subnet, zone)
Service/Port (what protocol and port)
Action (allow or deny)
Logging (whether matches are logged)
Example rule set for a web application:
#	Source	Destination	Service	Action	Notes
1	Internet	DMZ Web Server	TCP/443	Allow	HTTPS only, no plain HTTP
2	DMZ Web Server	App Server Zone	TCP/8443	Allow	App API only
3	App Server Zone	Database Zone	TCP/5432	Allow	PostgreSQL, specific source/destination only
4	Any	Any	Any	Deny	Default deny, logged
Rule Hygiene
Over time, firewall rule sets accumulate "temporary" rules that are never removed. Common issues found during audits:
Rules with source/destination set to "Any" (overly broad)
Rules referencing decommissioned servers
Duplicate or shadowed rules (a broad rule earlier in the list makes a later, more specific rule unreachable)
No documented business justification for a rule
A periodic firewall rule review (e.g. every 6 months) should check each rule against: is it still needed, is it still scoped correctly, and is there an owner who can justify it.
---
5. Common Real-World Tickets
---
TICKET-FW-001: Unauthorised Inbound Access Detected on Database Server
Reported by: SIEM alert, unusual inbound connection to database server from outside expected subnet
Severity: High
Environment: On-premises data centre
Problem
The SIEM raised an alert showing inbound TCP/1433 (SQL Server) connections to a production database server from a subnet used by the guest Wi-Fi network. This subnet should have no route to the Server Zone at all.
Investigation
Reviewed the firewall rule base between the Guest Wi-Fi zone and the Server Zone
Found a rule allowing "Any to Any" on TCP/1433, added 14 months ago during a vendor's temporary testing window
The rule was never removed after testing concluded
Checked connection logs: a small number of connection attempts had come from a guest device, all rejected at the database's own host firewall (which had correct restrictions), so no actual data access occurred
Root Cause
A temporary firewall rule created for a vendor test was not time-bound and was never reviewed or removed. The database host firewall acted as a secondary control and prevented actual compromise, but the network-level exposure existed for over a year.
Resolution
Removed the overly broad rule immediately
Added a scoped rule allowing only the specific application server subnet to reach the database on TCP/1433
Reviewed all other rules created in the same time period for the same vendor, found and removed two similar temporary rules
Introduced a process requiring all temporary firewall rules to have an expiry date and an assigned owner
Added a quarterly firewall rule review to the security operations calendar
Lessons Learned
Defence in depth worked here: the host firewall prevented compromise even though the network firewall rule was wrong. However, network-level segmentation should not rely on host firewalls as the only control. Temporary rules need expiry dates enforced through the change management process, not just good intentions.
---
TICKET-FW-002: Outbound Connection to Known Malicious IP
Reported by: Firewall IPS module, signature match on outbound traffic
Severity: Critical
Environment: Corporate network
Problem
The perimeter firewall's IPS module flagged outbound traffic from a workstation in the Corporate Zone to an IP address on a threat intelligence blocklist, associated with a known command-and-control (C2) server for a commodity malware family.
Investigation
Identified the source workstation from the firewall log (internal IP, mapped to a specific user via DHCP logs)
Isolated the workstation from the network (moved to a quarantine VLAN) while investigation continued
Reviewed endpoint logs on the workstation: found a recently executed file in the user's Downloads folder, matching a known malware hash on VirusTotal
User confirmed they had opened an email attachment from an unexpected sender earlier that day
Checked firewall logs for any other workstations contacting the same C2 IP: none found, so this appeared contained to one device
Resolution
Workstation remained isolated, full malware scan and reimage performed
Blocked the malicious IP and associated domains at the firewall for the whole organisation (this was already partially covered by the IPS feed, but explicit block rules were added as well)
Reviewed email security logs to confirm whether other users received the same phishing email, found 3 other recipients, none of whom had opened the attachment
Those 3 users were notified and the email was retroactively removed from their mailboxes
Reported the phishing sender and indicators to the email security vendor for blocklisting
Lessons Learned
The IPS signature match was the first effective detection point, the email security control had not flagged this particular attachment. This was raised as a gap, attachment sandboxing/detonation for the email gateway was added to the security roadmap. Quarantine VLAN procedures worked well and contained the incident to one device.
---
6. Interview Cheat Sheet
Q: How would you design network segmentation for a new office?
A: Start with zones based on function and trust level: DMZ for anything internet-facing, a Server Zone for internal applications and databases, a Corporate Zone for user devices, and a Management Zone for administrative access. Place firewalls at each boundary with default-deny rules, and only open the specific ports needed between zones.
Q: What's the difference between a stateful and stateless firewall?
A: A stateless firewall evaluates each packet independently against rules (source, destination, port). A stateful firewall tracks the state of connections, so it knows that a response packet belongs to an established outbound connection and allows it automatically, without needing a separate inbound rule for every possible reply.
Q: How do you keep a firewall rule base clean over time?
A: Periodic rule reviews (commonly every 6 months), requiring a documented business justification and owner for every rule, expiry dates on temporary rules, and removing rules tied to decommissioned systems. Tools can also flag unused rules by checking hit counts over a period.
Q: A workstation is communicating with a known malicious IP. Walk me through your response.
A: Isolate the device from the network first to stop further communication. Identify the user and check endpoint logs for the cause (malware, phishing attachment). Block the malicious IP/domain at the perimeter for the whole organisation. Check if other devices contacted the same indicator. Remediate the affected device (scan or reimage). Review how the initial detection was missed by earlier controls (e.g. email security) and close that gap.
Q: What is a Web Application Firewall and how is it different from a network firewall?
A: A network firewall controls traffic based on IP, port, and protocol. A WAF operates at the application layer and inspects the actual content of HTTP/HTTPS requests, looking for attack patterns like SQL injection or cross-site scripting, which a network firewall would not detect because the traffic itself looks like normal HTTPS.
---
