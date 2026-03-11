# AWS Audit Commands 

Runbook of auditing an AWS environment and the commands to run.

# CLI Cheat Sheet

This section is a cheat sheet for the [aws](https://github.com/aws/aws-cli) and [kubectl](https://kubernetes.io/docs/reference/kubectl/) command line interface (cli) tools used in this document.

## AWS CLI

The `aws` CLI is the primary way to interact with AWS from the command line.

### Check Your Current Identity

Before running any audit commands, confirm which account and role you are operating as. This is the AWS equivalent of `whoami` and should be the first thing you run after authenticating:

```bash
aws sts get-caller-identity
```

The output shows your `Account` ID, `UserId`, and the full `Arn` of the role or user you are acting as. If any of these are unexpected, stop and re-authenticate.

### Profiles

A profile is a named set of credentials and configuration stored in `~/.aws/credentials` and `~/.aws/config`. Profiles let you switch between accounts and roles without re-entering credentials.

List all configured profiles:

```bash
aws configure list-profiles
```

Run any command using a specific profile with the `--profile` flag:

```bash
aws sts get-caller-identity --profile <profile-name>
```

Set a profile for an entire terminal session so you do not have to pass `--profile` on every command:

```bash
export AWS_PROFILE=<profile-name>
```

### Switching Regions

Most `aws` commands operate against a single region. Either pass `--region` on every command or set a default for the session:

```bash
aws ec2 describe-instances --region us-west-2
```

```bash
export AWS_DEFAULT_REGION=us-west-2
```

Check which region is currently configured:

```bash
aws configure list | grep region
```

### Assuming a Role

Cross-account auditing often requires assuming a role in the target account. The output contains temporary credentials that can be exported to the shell:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<account-id>:role/<role-name> \
  --role-session-name audit-session
```

Write the following to a file and run it to assume a role and automatically export the credentials into the current shell session:

```bash
#!/usr/bin/env bash
OUTPUT=$(aws sts assume-role \
  --role-arn arn:aws:iam::<account-id>:role/<role-name> \
  --role-session-name audit-session)

export AWS_ACCESS_KEY_ID=$(echo "$OUTPUT" | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo "$OUTPUT" | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo "$OUTPUT" | jq -r '.Credentials.SessionToken')
```

Run `aws sts get-caller-identity` afterwards to confirm the role was assumed.

### Output Formats

The default output is JSON. Use `--output` to change it per command:

```bash
aws ec2 describe-instances --output table
```

```bash
aws ec2 describe-instances --output text
```

`table` is readable for quick spot-checks. `text` is useful for piping into other tools like `awk` or `cut`. `json` is best when piping into `jq`.

### Filtering Output with --query

The `--query` flag uses [JMESPath](https://jmespath.org/) to filter and reshape the response before it is printed. Every command in this document uses it. A few patterns worth knowing:

Select specific fields from a list of objects:

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name}'
```

Filter a list to only matching items using `?`:

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[?State.Name==`running`].InstanceId'
```

Pull a single value from a single object:

```bash
aws sts get-caller-identity --query 'Account' --output text
```

## kubectl

The `kubectl` client is the primary way to interact with a Kubernetes cluster from the command line.

### Check Your Current Context

A context in kubectl is a named combination of a cluster, a user, and a namespace. Always confirm which cluster you are pointed at before running commands:

```bash
kubectl config current-context
```

### List and Switch Contexts

List all contexts stored in your kubeconfig:

```bash
kubectl config get-contexts
```

Switch to a different context:

```bash
kubectl config use-context <context-name>
```

[kubectx](https://github.com/ahmetb/kubectx) is a faster alternative for switching contexts and is worth installing.

### Add an EKS Cluster to kubeconfig

Authenticate to an EKS cluster and add it to your local kubeconfig so kubectl can reach it:

```bash
aws eks update-kubeconfig \
  --name <cluster-name> \
  --region <region>
```

Pass `--profile` if the cluster is in a different account than your default profile.

### Set a Working Namespace

Passing `-n <namespace>` on every command gets repetitive. Set a default namespace for the current context:

```bash
kubectl config set-context --current --namespace=<namespace>
```

### Output Formats

The default output is a summary table. Use `-o` to change it:

```bash
kubectl get pods -o wide
```

```bash
kubectl get pod <pod-name> -o yaml
```

```bash
kubectl get pod <pod-name> -o json | jq .
```

`yaml` and `json` show the full resource spec including fields not shown in the default view, which is useful for auditing security contexts, labels, and annotations.

### Get More Detail on a Resource

`describe` gives a human-readable summary of a resource including events, which is useful for understanding what a resource is doing:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### List All Resource Types

If a resource type referenced in this document does not seem to exist in the cluster, check what resource types the cluster actually has. Not every cluster has every resource type:

```bash
kubectl api-resources -o wide
```

# Organization

An [Organization](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html) is the largest object in AWS. It contain one or more AWS Accounts. Normally, [Service Control Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) would be applied at this level and trickle down into the containing accounts.

Here are the key checks to walk through.

## Step 0: Verify SCPs Are Enabled and Attached

Confirm that SCPs are actually turned on for the organization. It's a feature that has to be explicitly enabled and it's easy to assume it's on when it isn't. Then verify that policies are attached to the org root or to OUs instead of just existing, but not attached to anything doing nothing.

**What to check:**
- The `SERVICE_CONTROL_POLICY` policy type shows as `ENABLED` on the org root
- There is at least one SCP attached to the root or to each OU
- No SCPs are sitting in the list unattached

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — foundational enforcement mechanism for controls **1.4** (no root access keys), **1.5** (root MFA), and **3.1** (CloudTrail in all regions) across all member accounts.

Confirm `SERVICE_CONTROL_POLICY` is enabled on the org root:

```bash
aws organizations list-roots \
  --query 'Roots[*].{Id:Id,PolicyTypes:PolicyTypes}'
```

List all SCPs in the organization:

```bash
aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[*].{Id:Id,Name:Name,Description:Description}'
```

Check which SCPs are attached to the root (substitute `<root-id>` from the output above):

```bash
aws organizations list-policies-for-target \
  --target-id <root-id> \
  --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[*].{Id:Id,Name:Name}'
```

## Check for a Root Account Restriction SCP

The AWS root account is the single most powerful identity in existence. It bypasses every IAM policy. An SCP that prevents root account actions across member accounts is one of the most important guardrails you can put in place. At the org level, you can't stop the management account's root user, but you can lock down every member account's root user so it can't be used. The management account's root user should have the password in a vault, hardware MFA locked away, and only used in a break-glass scenario. 

**What to check:**
- An SCP exists that denies all or specific high-risk actions when the principal is `root`
- The SCP is attached to the org root or all OUs, not just the management account
- The policy document uses `"Principal": {"AWS": "arn:aws:iam::*:root"}` or the `aws:PrincipalArn` condition key

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.4** (Ensure no root user account access key exists) and **1.5** (Ensure MFA is enabled for the root user account).

Inspect the full content of a specific SCP (substitute `<policy-id>` from the list above):

```bash
aws organizations describe-policy \
  --policy-id <policy-id> \
  --query 'Policy.Content' \
  --output text | jq .
```

To dump the contents of every SCP at once, write the following to a file and run it:

```bash
#!/usr/bin/env bash
for policy_id in $(aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[*].Id' \
  --output text); do
  echo "=== Policy: $policy_id ==="
  aws organizations describe-policy \
    --policy-id "$policy_id" \
    --query 'Policy.Content' \
    --output text | jq .
done
```

## Check for an IMDSv1 Restriction SCP

The IMDS SSRF vulnerability was already called out in the intro. The SCP is the most reliable way to enforce IMDSv2 at scale. Individual IAM policies can be overridden or misconfigured, but an SCP at the org root is a hard ceiling. Without it, a single misconfigured launch template can expose credentials to anyone who can reach the instance.

**What to check:**
- An SCP exists that denies `ec2:RunInstances` when the condition `ec2:MetadataHttpTokens` is not set to `required`
- The SCP covers all accounts by being attached to the root or all OUs
- There are no unintended exemptions for specific accounts or roles carved out of the SCP

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **5.6** (Ensure that EC2 Metadata Service only allows IMDSv2).

Write the following to a file and run it to scan all SCPs for any reference to IMDS or `MetadataHttpTokens`:

```bash
#!/usr/bin/env bash
for policy_id in $(aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[*].Id' \
  --output text); do
  content=$(aws organizations describe-policy \
    --policy-id "$policy_id" \
    --query 'Policy.Content' \
    --output text)
  if echo "$content" | grep -qi "MetadataHttpTokens\|imds"; then
    echo "Found IMDS policy: $policy_id"
    echo "$content" | jq .
  fi
done
```

## Check for a Region Restriction SCP

AWS enables all regions by default. A region restriction SCP limits deployments to approved regions only. This shrinks the monitoring surface and makes it much harder for a malicious actor to spin up resources in an unmonitored region and go unnoticed for months.

**What to check:**
- An SCP exists that denies all actions unless `aws:RequestedRegion` matches an approved list
- Global services (IAM, CloudFront, Route 53, STS, Support) are exempted since they don't use regional endpoints
- The approved region list matches what the organization actually uses

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — supports **3.1** (Ensure CloudTrail is enabled in all regions) by reducing the number of regions that need active monitoring and incident response coverage.

Check which regions are currently accessible from the account:

```bash
aws account list-regions \
  --region-opt-status-contains ENABLED ENABLED_BY_DEFAULT \
  --query 'Regions[*].{Region:RegionName,Status:RegionOptStatus}'
```

Write the following to a file and run it to scan all SCPs for a region restriction condition:

```bash
#!/usr/bin/env bash
for policy_id in $(aws organizations list-policies \
  --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[*].Id' \
  --output text); do
  content=$(aws organizations describe-policy \
    --policy-id "$policy_id" \
    --query 'Policy.Content' \
    --output text)
  if echo "$content" | grep -q "RequestedRegion"; then
    echo "Found region restriction in policy: $policy_id"
    echo "$content" | jq .
  fi
done
```

## Check for an Org-Wide CloudTrail Feeding a Central S3 Bucket

An audit trail is the foundation of incident response and compliance. CloudTrail records every API call made in every account. The org-wide trail consolidates those logs into a single, centralized S3 bucket under the security or logging account that is separate from the accounts being audited so that a compromised account cannot tamper with or delete its own logs. 

> This is a hard requirement for SOC2 and ISO 27001.

**What to check:**
- An organizational CloudTrail exists and is enabled in all regions
- Logs are delivered to an S3 bucket in a dedicated logging or security account, not the account being audited
- The destination S3 bucket has `Block Public Access` enabled and a bucket policy that denies `s3:DeleteObject` and `s3:PutBucketPolicy` to everyone except the logging service
- Log file validation is enabled so that tampered or deleted log files can be detected

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **3.1** (Ensure CloudTrail is enabled in all regions), **3.2** (Ensure CloudTrail log file validation is enabled), and **3.3** (Ensure the S3 bucket used to store CloudTrail logs is not publicly accessible).

List all trails and confirm one is an organizational trail. Note the `Name` and `S3BucketName` values for the next steps:

```bash
aws cloudtrail describe-trails \
  --include-shadow-trails \
  --query 'trailList[*].{Name:Name,IsOrganizationTrail:IsOrganizationTrail,HomeRegion:HomeRegion,S3BucketName:S3BucketName,LogFileValidationEnabled:LogFileValidationEnabled}'
```

Check the trail status to confirm logging is actually active (substitute `<trail-name>` from above):

```bash
aws cloudtrail get-trail-status \
  --name <trail-name> \
  --query '{IsLogging:IsLogging,LatestDeliveryTime:LatestDeliveryTime,LatestDeliveryError:LatestDeliveryError}'
```

Confirm the destination bucket blocks public access (substitute `<trail-bucket-name>` from the first command):

```bash
aws s3api get-public-access-block \
  --bucket <trail-bucket-name>
```

Check the bucket policy for any permissions that would allow deleting objects or modifying the policy:

```bash
aws s3api get-bucket-policy \
  --bucket <trail-bucket-name> \
  --output text | jq .
```

## Confirm GuardDuty Is Enabled Organization-Wide

GuardDuty is AWS's managed threat detection service. It watches CloudTrail, DNS logs, VPC Flow Logs, and more for signs of compromise. When enrolled at the org level through a delegated administrator, it automatically covers new member accounts as they're added. Without org-level enrollment, new accounts come in unmonitored.

**What to check:**
- GuardDuty has a delegated administrator account configured for the organization
- `AutoEnableOrganizationMembers` is set to `ALL` or `NEW` so new accounts are covered automatically
- All member accounts show a `Enabled` relationship status — look for any that are `Disabled` or `Resigned`

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **3.9** (Ensure that AWS GuardDuty is enabled).

Confirm a delegated administrator is set for GuardDuty:

```bash
aws organizations list-delegated-administrators \
  --service-principal guardduty.amazonaws.com \
  --query 'DelegatedAdministrators[*].{AccountId:Id,Status:Status}'
```

The remaining commands should be run from the delegated admin account. First, get the detector ID:

```bash
aws guardduty list-detectors \
  --query 'DetectorIds'
```

Check the org-wide configuration to confirm auto-enrollment is on (substitute `<detector-id>` from above):

```bash
aws guardduty describe-organization-configuration \
  --detector-id <detector-id> \
  --query '{AutoEnable:AutoEnable,AutoEnableOrganizationMembers:AutoEnableOrganizationMembers}'
```

List any member accounts that are not in an `Enabled` state:

```bash
aws guardduty list-members \
  --detector-id <detector-id> \
  --query 'Members[?RelationshipStatus!=`Enabled`].{AccountId:AccountId,Status:RelationshipStatus}'
```

# Account

The AWS [Account](https://docs.aws.amazon.com/accounts/latest/reference/accounts-welcome.html) is contained within an Organization. An organization can have many accounts. It acts as a resource and security boundary for AWS services. Below are the components that should be audited.

## Check the Root Account

The root account in each AWS account has unconditional access to everything. No IAM policy can restrict it. Two things must be true: 

* No access keys should exist for it
* MFA must be enabled

Also, check when it was last used. Root should almost never appear in normal operations. Any recent activity is worth investigating.

**What to check:**
* No access keys exist for the root account
* MFA is enabled for the root account
* The root account has not been used recently for day-to-day activity

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.4** (Ensure no root user account access key exists) and **1.5** (Ensure MFA is enabled for the root user account).

Pull a summary of the account's IAM posture. `AccountAccessKeysPresent` should be `0` and `AccountMFAEnabled` should be `1`:

```bash
aws iam get-account-summary \
  --query 'SummaryMap.{AccountMFAEnabled:AccountMFAEnabled,AccountAccessKeysPresent:AccountAccessKeysPresent}'
```

Generate a credential report to check last activity. The report takes a moment to produce, so run the generate command first and then retrieve it:

```bash
aws iam generate-credential-report
```

The API returns the report as a base64-encoded string. AWS does this because the report is a CSV file and raw CSV with its commas, newlines, and quotes would break the JSON response if embedded directly. The `--query 'Content'` pulls out the encoded string, and `base64 -d` decodes it back into the CSV. The final `grep root` filters down to just the root user's row:

```bash
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 -d | grep root
```

The output is a CSV row. The columns to focus on are `mfa_active` (field 8), `access_key_1_active` (field 9), and `password_last_used` (field 5). If `access_key_1_active` is `true` that is an immediate finding.

## Ensure CloudTrail Logging Is Set Up

CloudTrail records every API call made in the account. Without it, there is no audit trail and incident response becomes guesswork. Check that a trail exists, that it covers all regions, that log file validation is on, and that it is actually delivering logs without errors.

**What to check:**
* A trail exists and is enabled
* `IsMultiRegionTrail` is `true`
* `LogFileValidationEnabled` is `true`
* The trail is actively delivering logs with no recent delivery errors

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **3.1** (Ensure CloudTrail is enabled in all regions) and **3.2** (Ensure CloudTrail log file validation is enabled).

List all trails and review their configuration:

```bash
aws cloudtrail describe-trails \
  --query 'trailList[*].{Name:Name,MultiRegion:IsMultiRegionTrail,LogValidation:LogFileValidationEnabled,S3Bucket:S3BucketName,OrgTrail:IsOrganizationTrail}'
```

Check that logging is active and confirm no delivery errors (substitute `<trail-name>` from above):

```bash
aws cloudtrail get-trail-status \
  --name <trail-name> \
  --query '{IsLogging:IsLogging,LatestDeliveryTime:LatestDeliveryTime,LatestDeliveryError:LatestDeliveryError}'
```

## Enforce MFA on Users

A username and password with no MFA is a single credential away from account compromise. Every IAM user with console access needs an MFA device attached. The credential report is the fastest way to find users who are out of compliance.

**What to check:**
* All IAM users with a password (console access) have MFA enabled
* No users with console access have `mfa_active` set to `false`

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.10** (Ensure multi-factor authentication (MFA) is enabled for all IAM users that have a console password).

Generate a fresh credential report:

```bash
aws iam generate-credential-report
```

Write the following to a file and run it. The `base64 -d` decodes the CSV from the JSON response, and `awk` filters for users who have a password enabled but `mfa_active` set to false:

```bash
#!/usr/bin/env bash
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 -d | \
  awk -F',' 'NR>1 && $4=="true" && $8=="false" {print $1}'
```

Any name that appears in the output is a finding.

## Check for Stale Access Keys

Long-lived access keys are one of the most common sources of account compromise. A key created years ago by someone who has since left, attached to a role with broad permissions, is an easy target. CIS requires keys to be rotated every 90 days. In practice, teams create them and forget them. The credential report surfaces this fast and almost always turns up findings.

**What to check:**
* No active access keys are older than 90 days
* Keys that exist but have never been used should be disabled or deleted
* No user has more than one active access key at the same time

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.14** (Ensure access keys are rotated every 90 days or less).

Generate a fresh credential report first:

```bash
aws iam generate-credential-report
```

The API returns the report base64-encoded so it can travel safely inside the JSON response without the CSV's commas and newlines breaking the format. Write the following to a file and run it. `base64 -d` decodes the report back into CSV, and `awk` filters it down to active keys and their rotation and last-used dates:

```bash
#!/usr/bin/env bash
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 -d | \
  awk -F',' 'NR>1 {
    user=$1
    key1_active=$9
    key1_last_rotated=$10
    key1_last_used=$11
    key2_active=$14
    key2_last_rotated=$15
    key2_last_used=$16
    if (key1_active=="true") print user, "key1 rotated:", key1_last_rotated, "last used:", key1_last_used
    if (key2_active=="true") print user, "key2 rotated:", key2_last_rotated, "last used:", key2_last_used
  }'
```

Cross-reference the `rotated` dates against today. Any key older than 90 days is a finding. Any key where `last_used` is `N/A` has never been used and should be deleted.

# VPC

This section holds the plan for auditing VPC and the commands to do so. The VPC is the heart of it all. Here are the things to look for at the VPC level.

## Enable VPC Flow Logs

Flow Logs capture metadata about every network connection accepted or rejected within a VPC: source IP, destination IP, port, protocol, and whether the traffic was allowed or denied. Without them, there is no record of network-level activity. Detecting lateral movement, data exfiltration, or unexpected traffic between resources is not possible if flow logs are off.

**What to check:**
* Flow Logs are enabled on every VPC
* Logs are being delivered to CloudWatch Logs or S3 without errors
* The log format captures enough fields to be useful

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **3.9** (Ensure VPC flow logging is enabled in all VPCs).

List all VPCs in the account and note their IDs:

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[*].{VpcId:VpcId,Name:Tags[?Key==`Name`]|[0].Value,IsDefault:IsDefault,CidrBlock:CidrBlock}'
```

Check which VPCs have flow logs configured (substitute `<vpc-id>` from above):

```bash
aws ec2 describe-flow-logs \
  --filter Name=resource-id,Values=<vpc-id> \
  --query 'FlowLogs[*].{VpcId:ResourceId,Status:FlowLogStatus,Destination:LogDestinationType,DeliverLogsStatus:DeliverLogsStatus}'
```

Write the following to a file and run it to check flow log status across all VPCs at once:

```bash
#!/usr/bin/env bash
for vpc_id in $(aws ec2 describe-vpcs \
  --query 'Vpcs[*].VpcId' \
  --output text); do
  result=$(aws ec2 describe-flow-logs \
    --filter Name=resource-id,Values="$vpc_id" \
    --query 'FlowLogs[*].FlowLogStatus' \
    --output text)
  if [ -z "$result" ]; then
    echo "NO FLOW LOGS: $vpc_id"
  else
    echo "OK: $vpc_id - $result"
  fi
done
```

Any VPC printed with `NO FLOW LOGS` is a finding.

## Check Security Groups for Unrestricted Inbound Access

Security groups with inbound rules open to `0.0.0.0/0` on sensitive ports expose those resources directly to the internet. SSH (port 22) and RDP (port 3389) are the most critical. An open SSH port is actively scanned and attacked within minutes of being exposed. Any finding here should be treated as high severity.

**What to check:**
* No security group allows inbound `0.0.0.0/0` on port 22 (SSH)
* No security group allows inbound `0.0.0.0/0` on port 3389 (RDP)
* No security group allows inbound `0.0.0.0/0` on all ports (`-1` protocol)

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **5.2** (Ensure no security groups allow ingress from 0.0.0.0/0 to remote server administration ports).

### Check for security groups with unrestricted SSH access:

```bash
aws ec2 describe-security-groups \
  --filters Name=ip-permission.from-port,Values=22 \
            Name=ip-permission.cidr,Values=0.0.0.0/0 \
  --query 'SecurityGroups[*].{GroupId:GroupId,GroupName:GroupName,VpcId:VpcId}'
```

### Check for security groups with unrestricted RDP access:

```bash
aws ec2 describe-security-groups \
  --filters Name=ip-permission.from-port,Values=3389 \
            Name=ip-permission.cidr,Values=0.0.0.0/0 \
  --query 'SecurityGroups[*].{GroupId:GroupId,GroupName:GroupName,VpcId:VpcId}'
```

### Check for security groups that allow all inbound traffic from anywhere:

```bash
aws ec2 describe-security-groups \
  --filters Name=ip-permission.protocol,Values=-1 \
            Name=ip-permission.cidr,Values=0.0.0.0/0 \
  --query 'SecurityGroups[*].{GroupId:GroupId,GroupName:GroupName,VpcId:VpcId}'
```

## Check the Default VPC and Default Security Group

Every AWS account comes with a default VPC in every region. Its default security group allows unrestricted inbound traffic from other members of the same security group. Resources should not be deployed into the default VPC. It exists as a convenience for getting started and has none of the network architecture a production workload needs. The default security group should also restrict all traffic so that anything accidentally launched into it does not inherit open access.

**What to check:**
* No resources (EC2 instances, RDS, Lambda, etc.) are deployed in the default VPC
* The default security group in every VPC has no inbound or outbound rules beyond what is required
* Ideally the default VPC has been deleted in regions the organization does not use

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **5.3** (Ensure the default security group of every VPC restricts all traffic).

List all default VPCs across the account:

```bash
aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query 'Vpcs[*].{VpcId:VpcId,Region:OwnerId,CidrBlock:CidrBlock}'
```

Check the rules on the default security group (substitute `<vpc-id>`):

```bash
aws ec2 describe-security-groups \
  --filters Name=vpc-id,Values=<vpc-id> \
            Name=group-name,Values=default \
  --query 'SecurityGroups[*].{GroupId:GroupId,InboundRules:IpPermissions,OutboundRules:IpPermissionsEgress}'
```

## Check VPC Peering Route Tables

VPC peering connections link two VPCs so that traffic can flow between them. The risk is in the route tables. A broad route that allows any subnet in one VPC to reach any subnet in the other defeats the purpose of network segmentation. Each peering connection should route only the specific subnets that need to communicate, not entire VPC CIDR blocks.

**What to check:**
* VPC peering connections are intentional and documented
* Route tables for peering connections reference specific subnets, not full VPC CIDR blocks
* No peering connection routes traffic between a production VPC and a development or shared-services VPC without explicit justification

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **5.4** (Ensure routing tables for VPC peering are "least access").

List all active VPC peering connections:

```bash
aws ec2 describe-vpc-peering-connections \
  --filters Name=status-code,Values=active \
  --query 'VpcPeeringConnections[*].{Id:VpcPeeringConnectionId,Requester:RequesterVpcInfo.VpcId,Accepter:AccepterVpcInfo.VpcId}'
```

List all route tables and their routes to review which subnets are routed through each peering connection:

```bash
aws ec2 describe-route-tables \
  --query 'RouteTables[*].{RouteTableId:RouteTableId,VpcId:VpcId,Routes:Routes[?VpcPeeringConnectionId!=null]}'
```

# EC2

This section holds the plan for auditing [EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html) server instances and the commands to do so.

## Check for IMDSv2

The AWS [IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) service provides instance metadata and temporary credentials to applications running on the instance. IMDSv1 has a known [SSRF vulnerability](https://aws.amazon.com/blogs/security/how-to-use-policies-to-restrict-where-ec2-instance-credentials-can-be-used-from/) where an attacker who can make a server-side request to `169.254.169.254` can steal the instance's IAM credentials without any authentication. IMDSv2 requires a session token that must be fetched first, which blocks that attack. Every instance should have `HttpTokens` set to `required`.

**What to check:**
* Every instance has `MetadataOptions.HttpTokens` set to `required`, not `optional`
* The org-level SCP (covered in the Organization section) is in place to prevent new instances from launching with IMDSv1 enabled

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **5.6** (Ensure that EC2 Metadata Service only allows IMDSv2).

List all instances and their IMDS configuration. Any instance showing `optional` in the `HttpTokens` column is a finding:

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{InstanceId:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,HttpTokens:MetadataOptions.HttpTokens,State:State.Name}' \
  --output table
```

## Check EBS Encryption

[EBS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEBS.html) volumes hold the filesystem data for EC2 instances. Unencrypted volumes expose data at rest if a snapshot is shared accidentally or if someone gains access to the underlying storage. Two things need to be true: default encryption should be enabled so new volumes are encrypted automatically, and existing volumes should be checked for any that were created before the default was set.

**What to check:**
* Default EBS encryption is enabled for the account in this region
* No existing volumes are unencrypted
* Snapshot copies also inherit encryption

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **2.2.1** (Ensure EBS volume encryption is enabled in all regions).

Check if default encryption is turned on for new EBS volumes in this region. The result should be `true`:

```bash
aws ec2 get-ebs-encryption-by-default
```

List all unencrypted volumes currently in the account:

```bash
aws ec2 describe-volumes \
  --query 'Volumes[?Encrypted==`false`].{VolumeId:VolumeId,InstanceId:Attachments[0].InstanceId,Size:Size,State:State}'
```

## Check Instance Profiles

EC2 instances that need AWS API access should use [IAM instance profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html). Instance Profiles are an IAM role attached directly to the instance. Think of it as the user the instance runs as. The instance then retrieves short-lived credentials from IMDS rather than relying on hardcoded access keys baked into the application or stored in a config file. An instance with no profile that is making AWS API calls is a candidate for having credentials stored somewhere they should not be.

**What to check:**
* Every instance that makes AWS API calls has an IAM instance profile attached
* Instances with no profile attached are investigated for hardcoded credentials
* The permissions on each profile follow least privilege — no `AdministratorAccess` or wildcard policies

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.18** (Ensure that IAM Access analyzer is enabled for all regions) supports this by flagging overly permissive roles; least privilege is a foundational CIS principle throughout section 1.

List all instances and their attached profiles. Any instance showing `null` for `Profile` is worth investigating:

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{InstanceId:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,Profile:IamInstanceProfile.Arn,State:State.Name}' \
  --output table
```

## Check for Tags

Tags are how you attribute resources to teams, environments, and cost centers. During an incident, an untagged instance with no owner is a serious problem. No one knows who owns it, what it runs, or who to call. Good tagging also makes it easier to scope findings to a specific environment and avoid accidentally reporting a dev instance as a production risk.

Recommended tags for EC2 instances:
* `Name` — human-readable name for the instance
* `Environment` — `production`, `staging`, `development`
* `Owner` / `Team` — the team responsible for the instance
* `CostCenter` — for billing attribution
* `DataClassification` — what sensitivity of data the instance handles
* `ManagedBy` — `terraform`, `cloudformation`, `manual`, etc.
* `PatchGroup` — used by SSM Patch Manager to control patching schedules
* `ResponseSLA` — how quickly the owning team responds to incidents involving this resource

List all instances and their tags:

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{InstanceId:InstanceId,State:State.Name,Tags:Tags}'
```

Write the following to a file and run it to find instances with no tags at all:

```bash
#!/usr/bin/env bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{InstanceId:InstanceId,Tags:Tags}' \
  --output json | \
  jq '.[] | .[] | select(.Tags == null or .Tags == []) | .InstanceId'
```

# Networking

This section covers the controls around how traffic enters and exits the environment. These checks are distinct from the VPC-level checks where the VPC section focused on network configuration (flow logs, security groups, peering). This section focuses on the architecture decisions around how workloads are exposed and protected.

## Check for Instances with Public IPs

An EC2 instance with a public IP is directly reachable from the internet. Most production workloads should sit in private subnets and receive traffic only through a load balancer. A direct public IP bypasses the load balancer, the WAF, and any centralized logging of inbound requests. Every public IP on an instance should have a documented reason for existing.

**What to check:**
* Every instance with a public IP has a documented justification
* Public instances are covered by a security group that restricts inbound access to only required ports
* No instances in a private subnet have elastic IPs attached without explicit intent

List all running instances that have a public IP assigned:

```bash
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[*].Instances[?PublicIpAddress!=null].{InstanceId:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,PublicIP:PublicIpAddress,SubnetId:SubnetId}'
```

## Check Network Address Translation Gateway Usage

Private subnet instances need outbound internet access for things like package updates and API calls. A [Network Address Translation Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) (NAT) provides that outbound access without making the instance reachable from the internet. The alternative, giving instances public IPs, is the wrong answer. Check that private subnets route outbound traffic through a NAT Gateway, not directly to the internet gateway.

**What to check:**
* NAT Gateways exist and are in an `available` state
* Route tables for private subnets point to a NAT Gateway for `0.0.0.0/0`, not an internet gateway
* No active NAT instances are in use. NAT Gateways are the managed replacement and should be preferred

List all NAT Gateways and their state:

```bash
aws ec2 describe-nat-gateways \
  --filter Name=state,Values=available \
  --query 'NatGateways[*].{NatGatewayId:NatGatewayId,VpcId:VpcId,SubnetId:SubnetId,State:State}'
```

List route tables and check that private subnets route through NAT, not an internet gateway:

```bash
aws ec2 describe-route-tables \
  --query 'RouteTables[*].{RouteTableId:RouteTableId,VpcId:VpcId,Routes:Routes[?DestinationCidrBlock==`0.0.0.0/0`]}'
```

## Check Web Application Firewall Association with Load Balancers

A [Web Application Firewall](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html) (WAF) inspects HTTP/S traffic before it reaches the application and can block common attacks like SQL injection, cross-site scripting, and request flooding. A load balancer with no WAF attached is accepting all traffic directly. Check that every public-facing load balancer has a WAF Web ACL associated with it.

**What to check:**
* WAF Web ACLs exist in the account
* Every public-facing Application Load Balancer has a Web ACL associated
* Web ACLs have meaningful rules, not just the default allow-all

List all WAF Web ACLs in the region:

```bash
aws wafv2 list-web-acls \
  --scope REGIONAL \
  --query 'WebACLs[*].{Name:Name,Id:Id,ARN:ARN}'
```

List the resources associated with a Web ACL (substitute `<web-acl-arn>` from above):

```bash
aws wafv2 list-resources-for-web-acl \
  --web-acl-arn <web-acl-arn> \
  --query 'ResourceArns'
```

AWS WAF supports [Managed Rule Groups](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-rule-groups.html). These are pre-built rule sets that can be added to a Web ACL without writing rules from scratch. AWS publishes a free set called [AWS Managed Rules](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups.html) that covers common threats like the OWASP Top 10, known bad inputs, and IP reputation lists. Third-party rule groups from vendors like F5 and Fortinet are also available through AWS Marketplace. If a Web ACL has no managed rule groups and no custom rules, that is worth flagging — an empty Web ACL attached to a load balancer provides no protection.

## Check Load Balancer Configuration

[Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) (ELB) is the AWS service name, but it is an umbrella for three distinct load balancer types. Knowing which type you are looking at changes what you can and cannot configure:

**[Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) (ALB)** operates at Layer 7 (HTTP/S). It understands URLs, host headers, and request content, which is what makes path-based and host-based routing possible. A WAF can only be attached to an ALB, not the other types. This is the right choice for web applications, APIs, and any workload where you need the WAF or need to route based on request content.

**[Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html) (NLB)** operates at Layer 4 (TCP/UDP). It does not inspect request content. It just routes connections based on IP and port. It handles very high throughput with low latency and is the right choice for non-HTTP workloads like databases, game servers, or any TCP/UDP service. A WAF cannot be attached to an NLB.

**[Classic Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html) (CLB)** is the original and is considered legacy. If you see one in the environment, the recommendation is to migrate it to an ALB or NLB. New workloads should never use a CLB.

The `describe-load-balancers` command used below covers ALBs and NLBs (both use the `elbv2` API). Classic Load Balancers use a separate `elb` API and are listed separately.

Load balancers are the front door for most production traffic. Two things matter most from a security standpoint: all listeners should use HTTPS (not HTTP), and access logs should be enabled so there is a record of every request that comes through.

**What to check:**
* No listeners are configured for HTTP without a redirect rule pointing to HTTPS
* Access logs are enabled and delivering to an S3 bucket
* TLS policies on HTTPS listeners are not using deprecated protocols (TLS 1.0 or 1.1)

List all load balancers:

```bash
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[*].{Name:LoadBalancerName,Type:Type,Scheme:Scheme,State:State.Code,ARN:LoadBalancerArn}'
```

Check listeners on a load balancer for any using plain HTTP (substitute `<lb-arn>` from above):

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn <lb-arn> \
  --query 'Listeners[*].{Port:Port,Protocol:Protocol,SslPolicy:SslPolicy}'
```

Check if access logs are enabled for a load balancer:

```bash
aws elbv2 describe-load-balancer-attributes \
  --load-balancer-arn <lb-arn> \
  --query 'Attributes[?Key==`access_logs.s3.enabled`]'
```

# Simple Storage Service (S3)

This section holds the plan for auditing [S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) buckets and the commands to do so.

The AWS CLI has three separate sub-commands for S3 and it is worth knowing which one to reach for:

* `aws s3` is the high-level interface for working with objects inside buckets. It provides user-friendly commands like `cp`, `mv`, `sync`, `ls`, and `rm` that behave similarly to shell commands. This is what most people use day-to-day for moving data around.

* `aws s3api` maps directly to the S3 REST API and is used to inspect and configure the bucket itself. Bucket policies, ACLs, encryption settings, versioning, public access blocks, and logging are all configured through `s3api`. Most commands in this section use `s3api` for this reason.

* `aws s3control` handles account-level and organization-level S3 administrative features like S3 Access Points and S3 Batch Operations. Most audits do not require this unless the environment uses those specific features.

A simple way to think about it: `aws s3` is for what is inside a bucket, `aws s3api` is for the bucket itself.

Start by listing all buckets in the account. This is the inventory everything else is built from:

```bash
aws s3api list-buckets \
  --query 'Buckets[*].{Name:Name,Created:CreationDate}'
```

## Check Block Public Access

A bucket that is publicly readable exposes every object in it to anyone on the internet with no authentication required. Given that customer datasets live in these buckets, a public bucket is a critical finding. AWS has a four-setting [Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html) configuration that should be enabled on every bucket. All four settings should be `true`.

**What to check:**
* `BlockPublicAcls` is `true`
* `IgnorePublicAcls` is `true`
* `BlockPublicPolicy` is `true`
* `RestrictPublicBuckets` is `true`

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **2.1.4** (Ensure that S3 Buckets are configured with Block Public Access).

Check a single bucket (substitute `<bucket-name>` from the list above):

```bash
aws s3api get-public-access-block \
  --bucket <bucket-name>
```

Write the following to a file and run it to check every bucket at once. Any bucket printed with `MISSING` or showing a `false` value needs attention:

```bash
#!/usr/bin/env bash
for bucket in $(aws s3api list-buckets --query 'Buckets[*].Name' --output text); do
  result=$(aws s3api get-public-access-block --bucket "$bucket" 2>/dev/null)
  if [ $? -ne 0 ]; then
    echo "MISSING BLOCK PUBLIC ACCESS: $bucket"
  else
    echo "$bucket:" $(echo "$result" | jq -c '.PublicAccessBlockConfiguration')
  fi
done
```

## Inspect Bucket Policies and ACLs

Two separate controls govern access to a bucket. The bucket policy is an IAM-style JSON document that grants or denies access to specific principals. The bucket ACL is an older, simpler mechanism. Both need to be reviewed.

The most important thing to check in bucket policies is whether HTTP access is explicitly denied. A bucket that allows unencrypted HTTP connections can expose data in transit. Every bucket should have a policy statement that denies any request where `aws:SecureTransport` is `false`.

Check the bucket ACL for any grants to `AuthenticatedUsers` or `AllUsers`. `AuthenticatedUsers` does not mean users in your account. It means any authenticated AWS user from any account in the world. A grant to either of these groups is a critical finding.

**What to check:**
* The bucket policy contains a `Deny` statement on `aws:SecureTransport: false` (enforces HTTPS)
* No ACL grants exist for `AllUsers` or `AuthenticatedUsers`
* Cross-account access in the bucket policy is intentional and scoped to specific principals

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **2.1.1** (Ensure S3 Bucket Policy is set to deny HTTP requests).

Check the bucket policy and look for a `SecureTransport` deny condition:

```bash
aws s3api get-bucket-policy \
  --bucket <bucket-name> \
  --output text | jq .
```

Check the bucket ACL for any public or cross-account grants:

```bash
aws s3api get-bucket-acl \
  --bucket <bucket-name> \
  --query 'Grants[*].{Grantee:Grantee,Permission:Permission}'
```

## Check Encryption at Rest

All objects in buckets holding customer data should be encrypted at rest. S3 supports two main options: SSE-S3 (AWS-managed keys) and SSE-KMS (self-managed keys via AWS KMS). SSE-KMS is preferable for customer data because it gives independent control over the encryption key. The key can be rotated, restricted, or disabled separately from the bucket itself.

**What to check:**
* Every bucket has a default encryption configuration set
* Buckets holding customer data use SSE-KMS, not SSE-S3
* The KMS key used has automatic key rotation enabled

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **2.1.5** (Ensure that S3 Buckets are encrypted with AWS KMS).

Check the default encryption setting on a bucket:

```bash
aws s3api get-bucket-encryption \
  --bucket <bucket-name> \
  --query 'ServerSideEncryptionConfiguration.Rules[*].{Algorithm:ApplyServerSideEncryptionByDefault.SSEAlgorithm,KMSKeyId:ApplyServerSideEncryptionByDefault.KMSMasterKeyID}'
```

Write the following to a file and run it to check encryption status across all buckets at once:

```bash
#!/usr/bin/env bash
for bucket in $(aws s3api list-buckets --query 'Buckets[*].Name' --output text); do
  result=$(aws s3api get-bucket-encryption --bucket "$bucket" 2>/dev/null | \
    jq -r '.ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.SSEAlgorithm')
  echo "$bucket: ${result:-NOT ENCRYPTED}"
done
```

## Check for S3 VPC Endpoints

Traffic between EKS and S3 travels over the public internet by default, even though both are inside AWS. A [VPC Gateway Endpoint for S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html) routes that traffic directly through the AWS backbone without it ever leaving the network. This is both more secure (no public internet exposure) and faster (lower latency, no NAT Gateway per-GB cost). For workloads uploading and downloading large datasets continuously, this endpoint matters.

**What to check:**
* An S3 VPC Gateway Endpoint exists for every VPC where EKS nodes run
* The endpoint is in `available` state
* Bucket policies use the `aws:sourceVpce` condition to deny access from outside the VPC endpoint. This ensures EKS traffic stays on the private path and any accidental public access is blocked at the policy level

List all S3 VPC endpoints in the account:

```bash
aws ec2 describe-vpc-endpoints \
  --filters Name=service-name,Values=com.amazonaws.us-east-1.s3 \
  --query 'VpcEndpoints[*].{EndpointId:VpcEndpointId,VpcId:VpcId,State:State,Type:VpcEndpointType}'
```

Check whether a bucket policy enforces VPC endpoint access:

```bash
aws s3api get-bucket-policy \
  --bucket <bucket-name> \
  --output text | jq . | grep -i vpce
```

No output from the `grep` means there is no VPC endpoint restriction on the bucket policy. That is a finding for any bucket holding customer data.

## Check Versioning

[Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) keeps a full history of every object written to a bucket. If a computed dataset is overwritten with bad data, a pipeline bug corrupts an object, or an object is deleted accidentally, versioning makes recovery possible. For a pipeline where data integrity and availability are critical, an unversioned bucket is a risk.

**What to check:**
* Versioning is `Enabled` on all buckets holding raw and computed datasets
* MFA Delete is enabled on critical. This requires MFA to permanently delete a version, protecting against both accidents and malicious deletion

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **2.1.2** (Ensure MFA Delete is enabled on S3 buckets).

Write the following to a file and run it to check versioning status across all buckets:

```bash
#!/usr/bin/env bash
for bucket in $(aws s3api list-buckets --query 'Buckets[*].Name' --output text); do
  status=$(aws s3api get-bucket-versioning --bucket "$bucket" \
    --query '{Versioning:Status,MFADelete:MFADelete}')
  echo "$bucket: $status"
done
```

Any bucket showing `null` for `Versioning` has versioning disabled.

## Check for Tags

Tags on S3 buckets are how you know which team owns a bucket, what data it holds, and how sensitive that data is. `DataClassification` is especially important here. The appropriate access controls, encryption, and retention policy all follow from knowing what classification a bucket carries.

Recommended tags for S3 buckets:
* `DataClassification` — sensitivity level of the data in the bucket (`confidential`, `internal`, `public`)
* `Owner` / `Team` — the team responsible for the bucket
* `CostCenter` — for billing attribution
* `ManagedBy` — `terraform`, `cloudformation`, `manual`, etc.
* `ObjectTTL` — expected retention period for objects in the bucket

Check tags on a specific bucket:

```bash
aws s3api get-bucket-tagging \
  --bucket <bucket-name>
```

## Check for Access Logging

[S3 access logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html) records every request made to a bucket: who accessed it, what object, and when. This is separate from CloudTrail. For buckets holding customer data, access logs provide an audit trail for data access that is required for SOC2 and ISO 27001. Logs should deliver to a separate dedicated logging bucket, not back into the bucket being audited.

**What to check:**
* Access logging is enabled on all buckets holding customer data
* Logs deliver to a separate, dedicated logging bucket
* The logging bucket itself has logging disabled (to avoid an infinite loop) and is not publicly accessible

Check if access logging is configured on a bucket:

```bash
aws s3api get-bucket-logging \
  --bucket <bucket-name>
```

An empty response means logging is not enabled. That is a finding for any bucket holding customer data.

# IAM

This section holds the plan for auditing IAM and the commands to do so. The point of this section is to ensure that least-privilege is enfored. The use of the AWS builtin Roles or the use of wildcards `*.*` is too permissive.

## Inspect the List of Users and Ensure they Belong

Ensure the use of AWS-managed Roles and Groups aren't used. Enforce least-privilege where as the builtin roles and groups grant too many permissions.

## List Out Groups and Applied Permissions

## List Out Roles and the Trust Policies They Have


# EKS

This section holds the plan for auditing EKS cluster and the commands to do so. The objective here is to be able to go through the main parts of a [Kubernetes](https://kubernetes.io/docs/home/) cluster and identify the main components that need to be secured.

> Use api-resources to get a list of things to query the Kube API for in case the names can't be figured out

```bash
kubectl api-resources -o wide
```

## List Out Kubernetes Clusters and Versions

Check to see if the clusters are running supported versions of Kubernetes or End of Life.

## Add a Cluster to kubeconfing

Add the cluster to your [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) at this stage so that connection and auditing of it later can be achieved. 

> Use tools like [kubectx](https://github.com/ahmetb/kubectx) or [K9s](https://k9scli.io/) to manage your contexts.

## Check for Private Clusters 

The Kubernetes control plane should not be publicly exposed. Building private clusters is reccomended,. The following command(s) check to see if the cluster is prublic or private.

## Check to See if etcd Is Configured with Encryption at Rest 

[etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) stores data in plain text by default. Encrypted state should be set. 

## Nodes

This section contains commands and things to check for the Kubernetes Nodes.

### List out the Nodes and Versions

Check to ensure nodes and node groups are running supported versions of the kubelet.

```bash
kubectl get nodes
```

## Namespaces

A [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) is how Kubernetes organizes resources. 

## Deployments, ReplicaSets, StatefulSets, DaemonSets, and Pods

These are the ways a workload can run in a cluster. Besure to list them out and understand what's running.

### Check for Reccomended Tagging

The should have the following reccomended labels:
* Owning/Repsonsble Team
* Cost Center
* Data Classification
* Response SLA

### Resources

Kubernetes workloads needs to have [resource](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) limits and requests defined. Defining these helps Kubernetes know how much resourcs to give a Pod and how much to limit it so that it doesn't become a noisy neighbor.

## Autoscalers

Kubernetes has the ability to [autoscale](https://kubernetes.io/docs/concepts/workloads/autoscaling/) worklaods up or down. This can be a Horizatonal Pod Autoscaler that scales up and down the number of Pods in a cluster based on load or a Vertical Pod Autoscaler that increases or decreases the amount of resources a Pod can request and consume. Check to ensure all necessary workloads have these.

## Pod Security Context Pod and Container-Levels

Pods can have a [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) that defines privilege and access control. There are to place this can be defined in the Pod spec:

### 1. Pod Spec `.spec.securityContext`

The Pod spec contains settings applied to the entire Pod. The spec looks like so:

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    supplementalGroups: [4000]
```

Ensure the Pod is not running as root.

### 2. Container definitions `.spec.contianers[*].securityContext`. 

The containers spec defines settings applied to the individual container inside of the Pod. Pods can have one or more containers. The spec looks like so:

```yaml
  containers:
    ...
    securityContext:
      allowPrivilegeEscalation: false
```

## Services, Gateway, and Network Policies

These components affect the routing and control of traffic within and outside of the cluster.

### Services

Kubernetes uses [Services](https://kubernetes.io/docs/concepts/services-networking/service/) to expose groups of Pods for networking. There are multiple [types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) of services. Check the services to ensure they are routing to the proper Pods. A reccomendation would to use the ClusterIP type unless explicitly necessary. ClusterIPs route traffic internally without exposing it to the network outside of the cluster. A different resource is used routing traffic in and out of a cluster.

## Gateway

[Gateway](https://kubernetes.io/docs/concepts/services-networking/gateway/) is the newest version of Ingress. This is how traffic is routed into and out of the cluster. Gateways use providers much like Ingress. Ensure those components are up to date and supported. Remove any uncessary Gateway routes.

## ConfigMaps, Secrets, and Volumes

### ConfigMaps

[ConfigMap]s are used to inject configurations into a Pod. They can be mounted as enviorment variables or as a file on the container's filesystem. Read through the ConfigMaps to ensure no secrets or other sensative information is kept in them. 

### Secrets

[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) are a way to inject sensative information into a Pod. Reccomendations for securing these are taken right from the Kubernetes documentation:

> Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. Additionally, anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace; this includes indirect access such as the ability to create a Deployment.

> In order to safely use Secrets, take at least the following steps:

> * Enable Encryption at Rest for Secrets.
> * Enable or configure RBAC rules with least-privilege access to Secrets.
> * Restrict Secret access to specific containers.
>* Consider using external Secret store providers.

## Roles, RoleBindings, ClusterRoles, and ClusterRoleBindings

### Check If IRSA or EKS Pod Identites Is Being Used

If neither are in use, then reccomend EKS Pod Identities.

## CRDs

# RDS

[Relational Database Service](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) (RDS) runs managed relational databases. 

RDS should use IAM for permissions instead of relying on the underlying db. for example, the postgres user should have a role grant of `rds_iam` to tell it to accept an iam token instead of a username and password.

## Public Exposure

# Bedrock

[Bedrock](https://aws.amazon.com/bedrock/) is for building and hosting AI models.

# Route 53

[Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html) is AWS's DNS service.

# KMS

# TODO:

## AWS services this doc docesn't currently cover, but could in the future. 

* Lamnbda
* SQS
* SNS
* Dynamo

## Other TODOs:

* The first version is a single flat file. In the future, maybe break this out into child files and directories.