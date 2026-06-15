🎫 TICKET-IAM-002: Service Account Access Key Exposed in GitHub
Reported by: AWS GuardDuty + GitHub Secret Scanning  
Severity: Critical  
Account: Development (acc: 111122223333)
Problem
A developer accidentally committed an IAM access key to a public GitHub repository. GuardDuty raised a `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` finding. GitHub's secret scanning also flagged it.
Investigation
Identified the access key ID from the GuardDuty finding
Checked CloudTrail for all API calls made with this key in the last 72 hours
Found unusual `DescribeInstances` calls from an IP in Eastern Europe, 4 hours after the commit
Key belonged to a CI/CD service account `svc-deployer` with `PowerUserAccess`
Immediate Response (within 15 minutes)
Immediately disabled the exposed access key via IAM console
Created a new access key and rotated it in the CI/CD pipeline
Blocked the external IP at the network level (Security Group + WAF)
Full Remediation
Investigated what the attacker accessed: `DescribeInstances` only (reconnaissance), no destructive actions confirmed
Reviewed all resources for unexpected changes (none found)
Replaced the long-lived access key entirely with an IAM Role for EC2 instance profile
Added AWS Secrets Manager for any remaining credentials
Enabled GuardDuty anomaly detection rules for credential abuse
Deployed git-secrets pre-commit hook to prevent future key commits
Policy Changes
Added SCP: deny `iam:CreateAccessKey` in all accounts except the management account
All new service integrations must use IAM Roles. Access keys require security team approval
Lessons Learned
Access keys should not exist for automated workloads. IAM Roles with instance profiles or OIDC federation (for CI/CD) eliminate this risk entirely. Shift-left: add secret scanning to all repositories.
