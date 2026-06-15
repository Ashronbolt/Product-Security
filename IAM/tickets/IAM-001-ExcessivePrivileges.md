🎫 TICKET-IAM-001: User with Excessive Privileges Detected
Reported by: Automated, AWS Config Rule `iam-user-no-policies-check`  
Severity: High  
Account: Production (acc: 999988887777)
Problem
A developer `john.smith@company.com` was found to have the `AdministratorAccess` managed policy attached directly to their IAM user in the Production account. Discovered during a quarterly access review.
Investigation
Pulled IAM credential report to confirm the attachment
Checked IAM Access Advisor: only S3 and Lambda services used in last 90 days
Reviewed CloudTrail: no suspicious activity, user appears to be a legitimate developer
Checked how the policy was attached: done 8 months ago by a former admin (no longer at company)
Root Cause
Legacy permission granted during a project by a former admin. No periodic access review process was in place at the time to catch over-provisioning.
Resolution
Removed `AdministratorAccess` policy from the user
Attached a custom policy scoped to `s3:*` and `lambda:*` in the relevant regions only
Enrolled user in IAM Identity Centre (federated access). Direct IAM user to be deprecated within 30 days
Added SCP to deny `iam:AttachUserPolicy` in Production OU going forward
Created AWS Config rule to alert if any IAM user has admin policies attached
Lessons Learned
Quarterly access reviews must be automated. Manual reviews are too infrequent. IAM Access Analyser and Config rules now provide continuous detection.
