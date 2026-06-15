🎫 TICKET-IAM-003: Cross-Account Role with Overly Permissive Trust Policy
Reported by: IAM Access Analyser (External Access Finding)  
Severity: Medium  
Account: Production (acc: 999988887777)
Problem
IAM Access Analyser flagged a role `DataAnalyticsRole` in Production with a trust policy allowing any principal in account `555566667777` to assume it. Account `555566667777` is an old vendor account that the contract with ended 6 months ago.
Investigation
Reviewed the trust policy: confirmed unrestricted `sts:AssumeRole` for entire account `555566667777`
Checked CloudTrail: last AssumeRole call from vendor account was 4 months ago
Verified with procurement: vendor contract ended, account should have no access.
Checked what the role permits: `s3:GetObject` and `s3:ListBucket` on a sensitive analytics S3 bucket
Resolution
Immediately updated the trust policy to remove account `555566667777`
Added a condition requiring a specific `ExternalId` for any future third-party role assumptions
Verified vendor had not accessed the S3 bucket since contract end (CloudTrail confirmed)
Implemented an offboarding checklist requiring access revocation as a mandatory step
Created a quarterly review process for all cross-account trust policies
Lessons Learned
Third-party access must be tracked in a register with contract end dates. Offboarding must include IAM cleanup. Access Analyser should run continuously and findings reviewed weekly.
