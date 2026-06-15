🔐 AWS IAM Organisation-Wide Security Guide
Domain: Identity & Access Management (IAM)  
Platform: Amazon Web Services (AWS)  
Level: Experienced | Scope: Enterprise / Multi-Account  
Portfolio: AM Consulting Cybersecurity Lab Series
---
📋 Table of Contents
What is Organisation-Wide IAM?
AWS Organisations: The Foundation
Account Structure & Design
Identity Strategy: Who Gets Access?
IAM Roles vs Users: The Right Approach
Service Control Policies (SCPs): The Guardrails
Least Privilege at Scale
Privileged Access Management (PAM)
IAM Access Analyser & Continuous Monitoring
Common Real-World Tickets & How to Solve Them
Interview Cheat Sheet
---
1. What is Organisation-Wide IAM?
In a small company, IAM is simple. A handful of users, a handful of policies.  
In an enterprise, you might have:
10–500+ AWS accounts (dev, staging, prod, security, logging, sandbox...)
Thousands of users spread across teams
Dozens of third-party services needing access
Compliance requirements (ISO 27001, SOC 2, PCI-DSS)
Organisation-wide IAM means answering one central question:
> *"Who can do what, where, and under what conditions, across every account we own?"*
Getting this wrong means:
❌ Data breaches from over-privileged users
❌ Compliance failures
❌ Accidental deletion of production resources
❌ Undetected lateral movement by attackers
---
2. AWS Organisations: The Foundation
AWS Organisations is the service that lets you manage multiple AWS accounts from a single Management (Root) Account.
```
Root (Management Account)
├── Security OU
│   ├── Security Tooling Account   ← GuardDuty, Security Hub, Config
│   └── Log Archive Account        ← All CloudTrail logs centralised here
├── Infrastructure OU
│   ├── Networking Account         ← VPCs, Transit Gateway, DNS
│   └── Shared Services Account    ← Active Directory, CI/CD
├── Workloads OU
│   ├── Production OU
│   │   ├── Prod Account (App A)
│   │   └── Prod Account (App B)
│   └── Development OU
│       ├── Dev Account (App A)
│       └── Dev Account (App B)
└── Sandbox OU
    └── Individual Dev Sandbox Accounts
```
Key Concepts
Term	What It Means
Management Account	The root account. Controls billing and SCPs. Never run workloads here.
Member Account	Every other account. Isolated blast radius.
Organisational Unit (OU)	A folder for grouping accounts. SCPs apply at OU level.
SCP	Service Control Policy. Defines the maximum permissions any account in an OU can ever have.
Why Multiple Accounts?
Think of each account as a separate vault. If an attacker compromises credentials in the Dev account, they cannot automatically get into Production. This is called blast radius reduction.
---
3. Account Structure & Design
The AWS Landing Zone / Control Tower Approach
AWS Control Tower automates this multi-account setup. It creates:
A Log Archive account for immutable CloudTrail logs from all accounts
An Audit account with read-only access for security teams
Pre-built SCPs and guardrails
Step-by-Step: Setting Up the Structure
Step 1: Create the Management Account
Enable MFA on the root user immediately
Never use root credentials day-to-day
Create a billing alert
Step 2: Enable AWS Organisations
Go to AWS Organisations → Create Organisation
Choose "All features" (not just consolidated billing)
Step 3: Create OUs
```
Organisations → Organise Accounts → Create OU
- Security OU
- Workloads OU
  - Production OU (nested under Workloads)
  - Development OU (nested under Workloads)
- Sandbox OU
```
Step 4: Create or invite member accounts
New accounts: Organisations → Add Account → Create Account
Existing accounts: Organisations → Add Account → Invite Account
Step 5: Move accounts into correct OUs
Drag-and-drop in the console, or use CLI:
```bash
aws organizations move-account \
  --account-id 123456789012 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-yyyyyyyy
```
---
4. Identity Strategy: Who Gets Access?
At enterprise scale, you never create individual IAM users in every account. That's unmanageable. Instead, you use one of these strategies:
Option A: AWS IAM Identity Centre (Recommended ✅)
Formerly called AWS SSO. This is the gold standard for human access.
```
Your Corporate Identity Provider (IdP)
        │
        │  (SAML 2.0 / OIDC federation)
        ▼
AWS IAM Identity Centre (Central hub)
        │
        ├──► Account A (assigns Permission Set = "Developer")
        ├──► Account B (assigns Permission Set = "ReadOnly")
        └──► Account C (assigns Permission Set = "Admin")
```
How it works:
Users log in once with their corporate credentials (e.g. Microsoft Entra ID / Okta)
They see a portal showing which accounts they have access to
They click an account → they get temporary credentials via an IAM Role
Credentials expire automatically (no long-lived access keys)
Permission Sets = reusable role templates you define once and assign to accounts.
Example Permission Sets:
`ReadOnly`: view all resources, change nothing
`Developer`: full access to dev accounts, read-only in prod
`NetworkAdmin`: manage VPCs, security groups, Route53
`SecurityAuditor`: read-only across all accounts for the security team
`BreakGlass-Admin`: emergency full admin, heavily monitored
Option B: Cross-Account IAM Roles (for automated/service access)
For machines, pipelines, Lambda functions, not humans.
```
CI/CD Pipeline (Account: DevOps)
        │
        │  AssumeRole
        ▼
DeploymentRole (Account: Production)
  └── Permissions: only deploy to ECS, only in eu-west-1
```
The pipeline assumes the role in the target account. It gets temporary credentials scoped to exactly what it needs.
What You NEVER Do at Scale
❌ Bad Practice	✅ Correct Approach
Create IAM users in every account	Use IAM Identity Centre with federated identities
Share access keys between people	One identity per person, short-lived credentials
Create permanent Admin access keys	Use roles + STS temporary credentials
Root account used for daily tasks	Root locked down, MFA enforced, only for billing/emergency
---
5. IAM Roles vs Users: The Right Approach
IAM Users
Long-lived credentials (username + password, access keys)
Should only exist for break-glass emergency or legacy service accounts
Every IAM user should have MFA enforced
Access keys should rotate every 90 days (or not exist at all)
IAM Roles
No long-lived credentials. Generates temporary tokens via STS
Used by: EC2 instances, Lambda, ECS tasks, CI/CD pipelines, humans via federation
Always prefer roles over users
The Role Trust Policy
Every role has two parts:
1. Trust Policy: who is allowed to assume this role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:role/CI-CD-Pipeline-Role"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-12345"
        }
      }
    }
  ]
}
```
2. Permission Policy: what this role can do once assumed
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices"
      ],
      "Resource": "arn:aws:ecs:eu-west-1:999988887777:service/my-app/*"
    }
  ]
}
```
Note the ExternalId condition. This prevents the "confused deputy" attack, where another AWS account tricks your role into giving them access.
---
6. Service Control Policies (SCPs): The Guardrails
SCPs are organisation-level controls. They define the maximum permissions any account in an OU can ever have, even if a local IAM admin tries to grant more.
Think of SCPs as the ceiling. IAM policies are what's actually granted within that ceiling.
```
SCP on Production OU:
  Deny: ec2:TerminateInstances (nobody can delete prod servers)
  Deny: iam:CreateUser (nobody can create local IAM users in prod)
  Deny: s3:DeleteBucket (nobody can delete S3 buckets)

Even if a local IAM Admin in the Prod account grants themselves
"s3:DeleteBucket", the SCP blocks it. The SCP wins.
```
Common Baseline SCPs
1. Deny leaving the Organisation
```json
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}
```
2. Deny disabling Security Hub / GuardDuty
```json
{
  "Effect": "Deny",
  "Action": [
    "guardduty:DeleteDetector",
    "guardduty:DisassociateFromMasterAccount",
    "securityhub:DisableSecurityHub"
  ],
  "Resource": "*"
}
```
3. Restrict to approved regions only (e.g. UK data residency)
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["eu-west-1", "eu-west-2"]
    }
  }
}
```
4. Require MFA for sensitive actions
```json
{
  "Effect": "Deny",
  "Action": [
    "iam:*",
    "sts:AssumeRole"
  ],
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}
```
5. Prevent root account usage
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringLike": {
      "aws:PrincipalArn": "arn:aws:iam::*:root"
    }
  }
}
```
---
7. Least Privilege at Scale
The principle: give every identity only the permissions it needs, nothing more.
How to Achieve It in Practice
Step 1: Start with AWS Managed Policies, then tighten
Use `ReadOnlyAccess`, `PowerUserAccess` as starting points
Review actual usage with IAM Access Analyser and AWS CloudTrail
Remove permissions that haven't been used in 90 days
Step 2: Use IAM Access Advisor
In IAM → any role/user → "Access Advisor" tab
Shows last time each service was accessed
Anything unused for 90 days → remove from the policy
Step 3: Permission Boundaries
Used when you're delegating IAM management (e.g. letting developers create their own roles) but you don't want them to exceed certain limits.
```
Permission Boundary on "Dev-Created-Roles":
  Allow: ec2:*, s3:*, lambda:*
  (Implicitly deny everything else)

Developer creates a role and attaches it to their Lambda.
Even if they attach AdministratorAccess to that role,
the permission boundary limits it to ec2, s3, lambda only.
```
Step 4: Attribute-Based Access Control (ABAC)
Tag-based access control. Instead of one policy per team, you use resource tags.
```json
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}"
    }
  }
}
```
This says: "You can manage EC2 instances that have the same Team tag as your identity." One policy scales to any number of teams.
---
8. Privileged Access Management (PAM)
For admin-level access (e.g. production database admins, security incident response), you never want standing privilege. Instead:
Just-In-Time (JIT) Access
Engineer requests elevated access via a ticketing system
Approval workflow triggers
IAM Identity Centre temporarily assigns the `BreakGlass-Admin` permission set
After the time window expires, access is automatically revoked
All activity during that window is logged
Tools that enable this: AWS IAM Identity Centre + Lambda automation, or third-party tools like CyberArk, BeyondTrust, Okta PAM.
Break-Glass Procedure
For genuine emergencies (e.g. locked out of all normal access):
Root account credentials stored in a physical safe (not digital)
Dual-control required (two people to access)
CloudTrail alert fires the moment root credentials are used
Post-incident review mandatory
---
9. IAM Access Analyser & Continuous Monitoring
IAM Access Analyser
Automatically identifies resources shared with external entities (other accounts, internet)
Flags unintended public S3 buckets, overly permissive role trust policies
Enable per account AND at the organisation level
```bash
# Enable organisation-level analyser (run from management account)
aws accessanalyzer create-analyzer \
  --analyzer-name org-wide-analyser \
  --type ORGANIZATION
```
CloudTrail: The Audit Log
Enable organisation-wide CloudTrail from management account
All logs centralised to the Log Archive account (tamper-proof)
Member accounts cannot disable or modify these logs (SCP enforces this)
Key events to alert on:
`ConsoleLogin` with `MFAUsed: No`
`AssumeRole` from unexpected source IPs
`CreateUser`, `AttachUserPolicy`
`DeleteTrail` or `StopLogging`
Any root account activity
AWS Config + Security Hub
AWS Config: records every change to IAM resources. Triggers rules like "MFA must be enabled on all IAM users"
Security Hub: aggregates findings from GuardDuty, Config, Access Analyser across all accounts into one dashboard
---
10. Common Real-World Tickets & How to Solve Them
---
🎫 TICKET-IAM-001: User with Excessive Privileges Detected
Reported by: Automated, AWS Config Rule `iam-user-no-policies-check`  
Severity: High  
Account: Production (acc: 999988887777)
Problem:  
A developer `john.smith@company.com` was found to have the `AdministratorAccess` managed policy attached directly to their IAM user in the Production account. This was discovered during a quarterly access review.
Investigation:
Pulled IAM credential report to confirm the attachment
Checked IAM Access Advisor: only S3 and Lambda services used in last 90 days
Reviewed CloudTrail: no suspicious activity, user appears to be a legitimate developer
Checked how the policy was attached: done 8 months ago by a former admin (no longer at company)
Root Cause:  
Legacy permission granted during a project by a former admin. No periodic access review process was in place at the time to catch over-provisioning.
Resolution:
Removed `AdministratorAccess` policy from the user
Attached a custom policy scoped to `s3:*` and `lambda:*` in the relevant regions only
Enrolled user in IAM Identity Centre (federated access). Direct IAM user to be deprecated within 30 days
Added SCP to deny `iam:AttachUserPolicy` in Production OU going forward
Created AWS Config rule to alert if any IAM user has admin policies attached
Lessons Learned:  
Quarterly access reviews must be automated. Manual reviews are too infrequent. IAM Access Analyser and Config rules now provide continuous detection.
---
🎫 TICKET-IAM-002: Service Account Access Key Exposed in GitHub
Reported by: AWS GuardDuty + GitHub Secret Scanning  
Severity: Critical  
Account: Development (acc: 111122223333)
Problem:  
A developer accidentally committed an IAM access key to a public GitHub repository. GuardDuty raised a `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` finding. GitHub's secret scanning also flagged it.
Investigation:
Identified the access key ID from the GuardDuty finding
Checked CloudTrail for all API calls made with this key in the last 72 hours
Found unusual `DescribeInstances` calls from an IP in Eastern Europe, 4 hours after the commit
Key belonged to a CI/CD service account `svc-deployer` with `PowerUserAccess`
Immediate Response (within 15 minutes):
Immediately disabled the exposed access key via IAM console
Created a new access key and rotated it in the CI/CD pipeline
Blocked the external IP at the network level (Security Group + WAF)
Full Remediation:
Investigated what the attacker accessed: `DescribeInstances` only (reconnaissance), no destructive actions confirmed
Reviewed all resources for unexpected changes (none found)
Replaced the long-lived access key entirely with an IAM Role for EC2 instance profile
Added AWS Secrets Manager for any remaining credentials
Enabled GuardDuty anomaly detection rules for credential abuse
Deployed git-secrets pre-commit hook to prevent future key commits
Policy Changes:
Added SCP: deny `iam:CreateAccessKey` in all accounts except the management account
All new service integrations must use IAM Roles. Access keys require security team approval
Lessons Learned:  
Access keys should not exist for automated workloads. IAM Roles with instance profiles or OIDC federation (for CI/CD) eliminate this risk entirely. Shift-left: add secret scanning to all repositories.
---
🎫 TICKET-IAM-003: Cross-Account Role with Overly Permissive Trust Policy
Reported by: IAM Access Analyser (External Access Finding)  
Severity: Medium  
Account: Production (acc: 999988887777)
Problem:  
IAM Access Analyser flagged a role `DataAnalyticsRole` in Production with a trust policy allowing any principal in account `555566667777` to assume it. Account `555566667777` is an old vendor account that the contract with ended 6 months ago.
Investigation:
Reviewed the trust policy: confirmed unrestricted `sts:AssumeRole` for entire account `555566667777`
Checked CloudTrail: last AssumeRole call from vendor account was 4 months ago
Verified with procurement: vendor contract ended, account should have no access
Checked what the role permits: `s3:GetObject` and `s3:ListBucket` on a sensitive analytics S3 bucket
Resolution:
Immediately updated the trust policy to remove account `555566667777`
Added a condition requiring a specific `ExternalId` for any future third-party role assumptions
Verified vendor had not accessed the S3 bucket since contract end (CloudTrail confirmed)
Implemented an offboarding checklist requiring access revocation as a mandatory step
Created a quarterly review process for all cross-account trust policies
Lessons Learned:  
Third-party access must be tracked in a register with contract end dates. Offboarding must include IAM cleanup. Access Analyser should run continuously and findings reviewed weekly.
---
11. Interview Cheat Sheet
Questions You Should Be Able to Answer
Q: How do you manage access across 50 AWS accounts?  
A: AWS Organisations with IAM Identity Centre. Federate with the corporate IdP (Entra ID/Okta). Define Permission Sets centrally. Users log in once and get temporary credentials per account. SCPs enforce guardrails at the OU level.
Q: What's the difference between an SCP and an IAM policy?  
A: SCPs are organisation-level ceilings. They define the maximum permissions any account can have. IAM policies grant permissions within that ceiling. If an SCP denies an action, no IAM policy can override it.
Q: How do you enforce least privilege at scale?  
A: Start with broad managed policies, use IAM Access Advisor to identify unused permissions, remove them after 90 days. Use Permission Boundaries when delegating IAM management. Use ABAC (tag-based) for dynamic, scalable access control.
Q: A developer says they need admin access to debug a production issue. What do you do?  
A: Implement Just-In-Time access. They raise a ticket, get time-limited elevated permissions via Identity Centre, all activity is logged in CloudTrail. Permissions auto-revoke after the window. Review the incident and patch the root cause so it doesn't happen again.
Q: An access key was leaked. Walk me through your response.  
A: 1) Disable the key immediately. 2) Check CloudTrail for what was done with it. 3) Rotate the key / replace with role-based access. 4) Assess impact. 5) Block attacker IP. 6) Post-incident review. 7) Prevent recurrence (git-secrets, secret scanning, remove long-lived keys).
Q: What is the confused deputy problem in IAM?  
A: When a trusted service is tricked into performing actions on behalf of a malicious party. Mitigated by using `aws:SourceAccount` and `sts:ExternalId` conditions on role trust policies.
---
📁 Repo Structure (GitHub)
```
cybersecurity-portfolio/
├── README.md
├── IAM/
│   ├── AWS_IAM_OrganisationWide_Guide.md   ← This file
│   ├── tickets/
│   │   ├── IAM-001-ExcessivePrivileges.md
│   │   ├── IAM-002-ExposedAccessKey.md
│   │   └── IAM-003-CrossAccountRoleMisconfiguration.md
│   └── policies/
│       ├── scp-deny-root-usage.json
│       ├── scp-restrict-regions.json
│       └── scp-deny-disable-security-services.json
├── Firewalls/
├── DLP/
├── Endpoint/
└── IncidentResponse/
```
---
