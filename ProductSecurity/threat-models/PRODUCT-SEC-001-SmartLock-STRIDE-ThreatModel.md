🔐 PRODUCT-SEC-001: Threat Model: Smart Door Lock (IoT Device)
Domain: Product Security, Hardware/IoT  
Methodology: STRIDE Threat Modelling  
Standards Referenced: IEC 62443, ISO/IEC 27400 (IoT Security), OWASP IoT Top 10  
Severity: High (multiple Critical findings)  

---
1. Background
Product: "SecureHome Pro", a Wi-Fi enabled smart door lock  
Components:
Embedded MCU (microcontroller) running firmware
Wi-Fi module for cloud connectivity
Bluetooth Low Energy (BLE) for local pairing/setup
Mobile app (iOS/Android) for remote control
Cloud backend (REST API + MQTT broker for real-time lock status)
Physical keypad + biometric fingerprint sensor
Request:  
The product team requested a security review before the device moves from prototype to mass production. A threat model is required to identify risks across the full product lifecycle: hardware, firmware, communication, mobile app, and cloud.
---
2. Scope & Assets
Asset	Description	Why It Matters
Firmware	Controls lock mechanism, Wi-Fi/BLE stack	Compromise = physical access to home
Encryption keys	Device-unique keys for cloud auth & BLE pairing	Key extraction = clone/impersonate device
User credentials	Stored on mobile app and cloud	Credential theft = account takeover
Lock/Unlock command channel	MQTT messages between app ↔ cloud ↔ device	Tampering = unauthorised unlock
Debug/JTAG interface	Used during manufacturing	Physical access = full firmware extraction
Firmware update mechanism (OTA)	Pushes new firmware to deployed devices	Malicious update = backdoor on every device
---
3. System Architecture (Data Flow)
```
[Mobile App] ──(HTTPS/TLS)──► [Cloud API + MQTT Broker] ──(MQTT/TLS)──► [Smart Lock Device]
                                                                              │
                                                            [BLE] ◄───────────┘ (local pairing)
                                                              │
                                                       [Physical Keypad/
                                                        Fingerprint Sensor]
                                                              │
                                                        [Lock Actuator/Motor]

      Trust Boundary 1: Internet ↔ Cloud
      Trust Boundary 2: Cloud ↔ Device (over Wi-Fi)
      Trust Boundary 3: Device ↔ User (BLE/Keypad/Biometric)
      Trust Boundary 4: Device internals ↔ Physical access (debug ports)
```
---
4. STRIDE Threat Analysis
STRIDE = Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege
---
S: Spoofing
Threat 4.1: Device Impersonation via Cloned Credentials
Description: If every device ships with the same hardcoded MQTT credentials (common shortcut in cheap IoT manufacturing), an attacker who extracts credentials from one device can impersonate any device on the fleet, sending fake "door unlocked" status or receiving unlock commands meant for other users.
Attack Vector: Extract firmware via JTAG/UART on one purchased unit → find hardcoded MQTT username/password → connect to broker as any device ID.
Likelihood: High (JTAG extraction is a known, documented technique for this device class)
Impact: Critical. Fleet-wide compromise
STRIDE Category: Spoofing
Affected Standard: OWASP IoT Top 10 (I1 - Weak/Hardcoded Credentials)
Threat 4.2: BLE Pairing Spoofing
Description: During initial setup, the mobile app pairs with the lock over BLE. If pairing uses "Just Works" mode (no PIN/passkey verification), an attacker nearby can intercept the pairing and impersonate either the app or the lock (man-in-the-middle).
Attack Vector: Attacker's device intercepts BLE advertisement during setup window, completes pairing before legitimate app does.
Likelihood: Medium (requires physical proximity during the setup window, typically a few minutes)
Impact: High. Attacker gains BLE control of the lock
STRIDE Category: Spoofing
---
T: Tampering
Threat 4.3: Firmware Tampering via Unsigned OTA Updates
Description: If OTA (over-the-air) firmware updates are not cryptographically signed, an attacker who can intercept the update channel (or compromise the update server) can push malicious firmware containing a backdoor.
Attack Vector: MITM on Wi-Fi update channel, or compromise of the firmware distribution server.
Likelihood: Medium
Impact: Critical. Full device takeover, persistent backdoor across entire fleet
STRIDE Category: Tampering
Affected Standard: IEC 62443-4-2 (Component Requirements - Secure Update)
Threat 4.4: Lock Command Tampering on MQTT Channel
Description: If MQTT messages between the cloud and device aren't using TLS with certificate pinning, an attacker on the local network (e.g. via a compromised router) can intercept and modify "lock" commands to "unlock" commands.
Attack Vector: ARP spoofing on home network → intercept MQTT traffic → modify command payload.
Likelihood: Medium
Impact: Critical. Unauthorised physical access
STRIDE Category: Tampering
---
R: Repudiation
Threat 4.5: Insufficient Audit Logging of Lock Events
Description: If the device/cloud doesn't maintain tamper-evident logs of who unlocked the door and when, there's no way to investigate after a break-in, and a malicious user (e.g. ex-partner with old access) could deny unlocking the door.
Attack Vector: N/A. This is a design gap, not an active attack
Likelihood: High (common gap in budget IoT products)
Impact: Medium. Hinders incident investigation, no accountability
STRIDE Category: Repudiation
---
I: Information Disclosure
Threat 4.6: Encryption Key Extraction via Hardware Debug Ports
Description: If JTAG/UART debug interfaces are left enabled on production units, an attacker with brief physical access (e.g. a delivery person, or buying a unit to reverse-engineer) can dump the entire firmware including embedded private keys used for cloud authentication.
Attack Vector: Connect to exposed test points on PCB → dump flash memory via JTAG.
Likelihood: High. Extremely common in prototype-to-production transitions where debug interfaces are not disabled
Impact: Critical. Keys extracted can be used to impersonate device to cloud, or extract algorithm/IP
STRIDE Category: Information Disclosure
Affected Standard: IEC 62443-4-2 (Hardware Security Requirements)
Threat 4.7: Sensitive Data in Mobile App Local Storage
Description: If the mobile app stores auth tokens or the user's home Wi-Fi credentials (used during device setup) in plaintext in local storage/shared preferences, a malicious app on the same phone or a lost/stolen phone could extract them.
Attack Vector: Malicious app with storage permissions reads SecureHome app's data directory.
Likelihood: Medium
Impact: High. Wi-Fi credential exposure, account token theft
STRIDE Category: Information Disclosure
---
D: Denial of Service
Threat 4.8: BLE/Wi-Fi Jamming Causing Lockout
Description: An attacker with a cheap RF jammer can disrupt Wi-Fi/BLE communication, preventing the legitimate user from remotely locking/unlocking, or preventing the device from reporting status to the cloud (masking an intrusion).
Attack Vector: RF jamming device near the property.
Likelihood: Medium
Impact: Medium. Availability impact, could mask physical intrusion (door physically forced while comms jammed so no alert sent)
STRIDE Category: Denial of Service
Threat 4.9: Cloud API Rate-Limit Abuse
Description: If the cloud API doesn't rate-limit unlock attempts, an attacker can brute-force PIN codes via the API or repeatedly trigger lock/unlock cycles, draining battery (battery-powered locks) and potentially causing motor wear/failure (physical DoS).
Attack Vector: Scripted API calls against `/device/{id}/unlock` endpoint.
Likelihood: Medium
Impact: Medium-High. Battery drain DoS, or PIN brute force leading to unauthorised access
STRIDE Category: Denial of Service + Spoofing overlap
---
E: Elevation of Privilege
Threat 4.10: Local Keypad Firmware Bypass via Buffer Overflow
Description: If the keypad PIN-entry firmware routine has a buffer overflow vulnerability (e.g. no bounds checking on PIN length input), an attacker entering a crafted long PIN sequence could potentially execute arbitrary code on the MCU, bypassing authentication entirely.
Attack Vector: Crafted input sequence via physical keypad.
Likelihood: Low-Medium (requires firmware vulnerability + reverse engineering)
Impact: Critical. Full authentication bypass, direct unlock
STRIDE Category: Elevation of Privilege
Threat 4.11: Privilege Escalation via Shared Cloud Tenant
Description: If the cloud backend uses a multi-tenant architecture without proper tenant isolation (e.g. IDOR, Insecure Direct Object Reference, where device IDs are sequential integers), one user might be able to query or send commands to another user's lock by changing an ID in the API request.
Attack Vector: `GET /api/device/1002/status` → change to `/api/device/1003/status` → access another user's device.
Likelihood: Medium
Impact: Critical. Cross-tenant access to other users' locks
STRIDE Category: Elevation of Privilege
---
5. Risk Summary Table
ID	Threat	STRIDE	Likelihood	Impact	Risk Rating
4.1	Device impersonation (hardcoded creds)	Spoofing	High	Critical	Critical
4.2	BLE pairing spoofing	Spoofing	Medium	High	High
4.3	Unsigned OTA firmware tampering	Tampering	Medium	Critical	Critical
4.4	MQTT command tampering	Tampering	Medium	Critical	Critical
4.5	Insufficient audit logging	Repudiation	High	Medium	Medium
4.6	Debug port key extraction	Info Disclosure	High	Critical	Critical
4.7	Insecure mobile app storage	Info Disclosure	Medium	High	High
4.8	RF jamming DoS	DoS	Medium	Medium	Medium
4.9	API rate-limit abuse	DoS	Medium	Medium-High	High
4.10	Keypad buffer overflow	Elevation of Privilege	Low-Medium	Critical	High
4.11	IDOR cross-tenant access	Elevation of Privilege	Medium	Critical	Critical
---
6. Remediation Recommendations
ID	Recommendation	Priority	Standard Reference
4.1	Provision unique per-device credentials/certificates during manufacturing (e.g. X.509 client certs per device)	Critical	IEC 62443-4-2
4.2	Use BLE LE Secure Connections with passkey/numeric comparison, not "Just Works" pairing	High	OWASP IoT Top 10
4.3	Implement signed firmware updates (code signing with hardware-backed key verification before flashing)	Critical	IEC 62443-4-2
4.4	Enforce TLS 1.2+ with certificate pinning on all MQTT/cloud communication	Critical	ISO/IEC 27400
4.5	Implement tamper-evident audit logs (hash-chained log entries) for all lock/unlock events, stored in cloud	Medium	ISO 27001 A.8.15
4.6	Disable/fuse JTAG and UART debug interfaces on production silicon (set security fuse bits)	Critical	IEC 62443-4-2
4.7	Use platform secure storage (iOS Keychain / Android Keystore) for tokens and Wi-Fi credentials, never plaintext	High	OWASP MASVS
4.8	Implement local fallback (keypad/biometric continues to work offline) and alert on prolonged comms loss	Medium	N/A
4.9	Implement rate limiting and exponential backoff on unlock endpoints; account lockout after failed attempts	High	OWASP API Security Top 10
4.10	Conduct firmware fuzz testing on all input parsers (keypad, BLE, Wi-Fi); implement bounds checking	High	IEC 62443-4-1 (Secure Development Lifecycle)
4.11	Use non-sequential, unguessable device identifiers (UUIDs); enforce server-side authorisation checks on every request	Critical	OWASP API Security Top 10 (BOLA)
---
7. Lifecycle Recommendations (Secure Product Development Lifecycle)
Mapping to IEC 62443-4-1 secure development practices:
SDLC Phase	Activity
Requirements	Define security requirements (e.g. "all comms must use TLS 1.2+", "no hardcoded credentials")
Design	Threat modelling (this document) conducted at architecture stage, before hardware finalisation
Implementation	Secure coding standards for firmware (MISRA-C), static analysis (SAST) on firmware codebase
Verification	Penetration testing on production-representative hardware, fuzz testing on all input interfaces
Manufacturing	Per-device credential provisioning, disable debug interfaces before shipping
Deployment	Signed OTA update pipeline established and tested
Maintenance	Vulnerability disclosure programme, defined patching SLA, end-of-life/support policy communicated to customers
---
8. Lessons Learned / Summary for Stakeholders
This threat model identified 5 Critical and 4 High risk findings, primarily clustered around:
Hardware security fundamentals: debug interfaces and credential provisioning are the highest-impact, highest-likelihood risks and must be addressed before mass production. This is the cheapest point to fix them; a recall is far more expensive.
Communication security: TLS/MQTT hardening and signed OTA updates are non-negotiable for any internet-connected lock.
Cloud-side authorisation: IDOR-style vulnerabilities are common in IoT backends and require systematic server-side authorisation checks, not just client-side assumptions.
Recommended next steps:
Remediate all Critical findings before moving to mass production tooling
Conduct a follow-up penetration test on production hardware once fixes are implemented
Establish a recurring threat model review cadence (at each major firmware version)
---
