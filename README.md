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
- All member accounts show an `Enabled` relationship status. Look for any that show `Disabled` or `Resigned`

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
* The permissions on each profile follow least privilege with no `AdministratorAccess` or wildcard policies

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

AWS WAF supports [Managed Rule Groups](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-rule-groups.html). These are pre-built rule sets that can be added to a Web ACL without writing rules from scratch. AWS publishes a free set called [AWS Managed Rules](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups.html) that covers common threats like the OWASP Top 10, known bad inputs, and IP reputation lists. Third-party rule groups from vendors like F5 and Fortinet are also available through AWS Marketplace. If a Web ACL has no managed rule groups and no custom rules, that is worth flagging as a finding. An empty Web ACL provides no protection.

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

# Identity and Access Management (IAM)

This section holds the plan for auditing [IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) and the commands to do so. The point of this section is to ensure that least-privilege is enfored. The use of the AWS builtin Roles or the use of wildcards `*.*` is too permissive.

## Audit IAM Users

Every IAM user in the account should have a known owner and a reason to exist. Users who have left the organization, service accounts no longer in use, and users created for one-off tasks that were never cleaned up are all attack surface. The ideal state for most organizations is to have very few or no IAM users at all. Human access should go through an identity provider via SSO (Okta, EntraID), and workloads should use IAM roles. Tools like [Teleport](https://goteleport.com/) layer on top of an SSO provider to manage and audit privileged infrastructure access with short-lived credentials, but that is out of scope for this document. IAM users with long-lived credentials are the fallback, not the default.

**What to check:**
* Every user has a documented owner and purpose
* Users with console access also have MFA enabled (covered in the Account section; cross-reference findings here)
* No user has both console access and active access keys at the same time
* Users who have not logged in or used their keys in 90 days should be disabled or removed

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.3** (Ensure credentials unused for 90 days or greater are disabled) and **1.12** (Ensure credentials unused for 45 days or greater are disabled).

List all IAM users and when they last used their password:

```bash
aws iam list-users \
  --query 'Users[*].{UserName:UserName,Created:CreateDate,PasswordLastUsed:PasswordLastUsed}'
```

Generate a fresh credential report and retrieve it to see the full picture: console access, access keys, MFA status, and last activity all in one place:

```bash
aws iam generate-credential-report
```

```bash
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 -d
```

## Check Groups and Attached Policies

Groups are how permissions should be assigned to IAM users. Attach a policy to a group and add users to the group, rather than attaching policies directly to individual users. The audit here has two goals: confirm that groups are being used as intended, and check that none of them carry AWS-managed policies that are too permissive.

`AdministratorAccess` and `PowerUserAccess` are the most common offenders. `AdministratorAccess` grants unrestricted access to every AWS service. `PowerUserAccess` grants full access to all services except IAM and Organizations. Neither should be attached to a group that regular users belong to.

**What to check:**
* Users are assigned permissions through groups, not through policies attached directly to the user
* No group has `AdministratorAccess` or `PowerUserAccess` attached
* Each group's policies reflect the actual job function of its members

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.15** (Ensure IAM policies are attached only to groups or roles) and **1.16** (Ensure IAM policies that allow full administrative privileges are not attached).

List all groups in the account:

```bash
aws iam list-groups \
  --query 'Groups[*].{GroupName:GroupName,GroupId:GroupId,Created:CreateDate}'
```

Check what managed policies are attached to a group (substitute `<group-name>` from above):

```bash
aws iam list-attached-group-policies \
  --group-name <group-name> \
  --query 'AttachedPolicies[*].{PolicyName:PolicyName,PolicyArn:PolicyArn}'
```

Check which users, groups, and roles have `AdministratorAccess` attached. Any result here is a finding worth documenting:

```bash
aws iam list-entities-for-policy \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --query '{Users:PolicyUsers[*].UserName,Groups:PolicyGroups[*].GroupName,Roles:PolicyRoles[*].RoleName}'
```

## Check Roles and Trust Policies

An IAM role is an identity that can be assumed by a user, a service, or another AWS account. The trust policy controls who is allowed to assume it. Overly broad trust policies are one of the most common and dangerous IAM misconfigurations. A role that any AWS account can assume, or that trusts a wildcard principal, can be assumed by anyone with AWS credentials.

**What to check:**
* No role has a trust policy with `"Principal": "*"` or `"AWS": "*"` without restrictive conditions
* Cross-account trust entries reference specific, known account IDs and not wildcards
* Service roles trust only the specific AWS service that needs them (e.g. `ec2.amazonaws.com`, not `*.amazonaws.com`)
* Roles that have not been used in 90 days should be reviewed for removal

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — least privilege principles throughout section 1; trust policy review supports **1.16**.

List all roles and their creation date:

```bash
aws iam list-roles \
  --query 'Roles[*].{RoleName:RoleName,Created:CreateDate,Description:Description}'
```

Inspect the trust policy of a specific role (substitute `<role-name>` from above):

```bash
aws iam get-role \
  --role-name <role-name> \
  --query 'Role.AssumeRolePolicyDocument' \
  --output text | jq .
```

Write the following to a file and run it to print the trust policy for every role at once. Review the output for any `"Principal": "*"` entries:

```bash
#!/usr/bin/env bash
for role in $(aws iam list-roles --query 'Roles[*].RoleName' --output text); do
  echo "=== $role ==="
  aws iam get-role \
    --role-name "$role" \
    --query 'Role.AssumeRolePolicyDocument' \
    --output text | jq .
done
```

## Check for Wildcard Permissions in Policies

A policy statement with `"Action": "*"` or `"Resource": "*"` grants unrestricted access to either every action in a service or every resource of a type. These are the IAM equivalent of root. They should not appear in any customer-managed policy. AWS-managed policies like `AdministratorAccess` contain wildcards by design, which is exactly why the recommendation is not to use them.

This check scans all customer-managed policies (policies your organization wrote, not AWS-provided ones) for wildcard actions or resources on Allow statements.

**What to check:**
* No customer-managed policy has `"Action": "*"` in an Allow statement
* No customer-managed policy has `"Resource": "*"` paired with sensitive actions like `iam:*`, `s3:*`, or `ec2:*`
* Inline policies attached directly to users, groups, or roles are also checked

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.16** (Ensure IAM policies that allow full administrative privileges are not attached) and the least privilege principle throughout section 1.

List all customer-managed policies in the account:

```bash
aws iam list-policies \
  --scope Local \
  --query 'Policies[*].{PolicyName:PolicyName,Arn:Arn,DefaultVersionId:DefaultVersionId}'
```

Inspect the document for a specific policy (substitute `<policy-arn>` and `<version-id>` from above):

```bash
aws iam get-policy-version \
  --policy-arn <policy-arn> \
  --version-id <version-id> \
  --query 'PolicyVersion.Document' | jq .
```

Write the following to a file and run it to scan every customer-managed policy for wildcard Allow statements:

```bash
#!/usr/bin/env bash
for policy_arn in $(aws iam list-policies --scope Local --query 'Policies[*].Arn' --output text); do
  version=$(aws iam get-policy \
    --policy-arn "$policy_arn" \
    --query 'Policy.DefaultVersionId' \
    --output text)
  doc=$(aws iam get-policy-version \
    --policy-arn "$policy_arn" \
    --version-id "$version" \
    --query 'PolicyVersion.Document')
  matches=$(echo "$doc" | jq '[.Statement[] | select(.Effect=="Allow") | select(.Action=="*" or (.Action | arrays | any(. == "*")))] | length')
  if [ "$matches" -gt 0 ]; then
    echo "WILDCARD ACTION FOUND: $policy_arn"
  fi
done
```

## Check Permissions on EC2 Instance Profile Roles

The EC2 section covered whether an instance profile exists. This check goes one level deeper and looks at what the role inside that profile is actually allowed to do. A compromised EC2 instance or EKS node inherits every permission on its attached role. An instance profile with `AdministratorAccess` or broad S3 and IAM permissions means a single compromised workload can access the entire account.

This is one of the highest-risk findings in a cloud environment. In this company's case, EKS nodes and EC2 instances have IAM roles that likely carry S3 permissions for the data pipeline. Those roles should be scoped to exactly the buckets and actions each workload needs. Anything broader is a finding.

Note: how IAM roles are assigned to individual pods inside EKS (IRSA and EKS Pod Identities) is covered in the EKS section.

**What to check:**
* No instance profile role has `AdministratorAccess` or `PowerUserAccess` attached
* S3 permissions are scoped to specific bucket ARNs, not `arn:aws:s3:::*`
* IAM permissions are absent from instance profile roles unless there is a documented reason
* Roles are not shared across multiple applications or workloads with different permission needs

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — least privilege principles throughout section 1; directly supports **1.16** and **1.22** (Ensure access to AWSCloudShellFullAccess is restricted).

List all instance profiles and their attached roles:

```bash
aws iam list-instance-profiles \
  --query 'InstanceProfiles[*].{ProfileName:InstanceProfileName,Roles:Roles[*].RoleName}'
```

List the policies attached to a specific instance profile role (substitute `<role-name>` from above):

```bash
aws iam list-attached-role-policies \
  --role-name <role-name> \
  --query 'AttachedPolicies[*].{PolicyName:PolicyName,PolicyArn:PolicyArn}'
```

Inspect the full document of a policy attached to the role to review its actual permissions (substitute `<policy-arn>` and `<version-id>`):

```bash
aws iam get-policy-version \
  --policy-arn <policy-arn> \
  --version-id <version-id> \
  --query 'PolicyVersion.Document' | jq .
```

Write the following to a file and run it to check every instance profile role for `AdministratorAccess` in one pass:

```bash
#!/usr/bin/env bash
for profile in $(aws iam list-instance-profiles \
  --query 'InstanceProfiles[*].InstanceProfileName' --output text); do
  roles=$(aws iam get-instance-profile \
    --instance-profile-name "$profile" \
    --query 'InstanceProfile.Roles[*].RoleName' --output text)
  for role in $roles; do
    policies=$(aws iam list-attached-role-policies \
      --role-name "$role" \
      --query 'AttachedPolicies[*].PolicyArn' --output text)
    if echo "$policies" | grep -q "AdministratorAccess"; then
      echo "ADMIN ACCESS ON INSTANCE PROFILE: $profile / $role"
    fi
  done
done
```

## Check IAM Access Analyzer

[IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) continuously scans resource policies across the account and flags anything that grants access to an external principal, which could be another AWS account, a public endpoint, or an unknown federated identity. It also has a policy validation mode that checks policies for overly permissive statements before they are deployed. Think of it as automated, ongoing IAM review running in the background.

The audit check here is simple: confirm it is enabled, and if it is, review any active findings. An Access Analyzer that has been enabled but has unreviewed findings sitting open is a problem in itself.

**What to check:**
* At least one analyzer exists and is in an `ACTIVE` state
* There are no unreviewed findings, or all findings have been acknowledged with a documented reason
* The analyzer type is `ACCOUNT` to cover all resources in the account, or `ORGANIZATION` if delegated from the management account

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **1.20** (Ensure that IAM Access analyzer is enabled for all regions).

Check if Access Analyzer is enabled and its current status:

```bash
aws accessanalyzer list-analyzers \
  --query 'analyzers[*].{Name:name,Status:status,Type:type,Region:arn}'
```

List any active findings from the analyzer (substitute `<analyzer-arn>` from above):

```bash
aws accessanalyzer list-findings \
  --analyzer-arn <analyzer-arn> \
  --filter '{"status": {"eq": ["ACTIVE"]}}' \
  --query 'findings[*].{Id:id,ResourceType:resourceType,Resource:resource,Condition:condition}'
```

Any finding with a status of `ACTIVE` that has not been reviewed is a finding in the audit report.

# Elastic Kubernetes Service (EKS)

This section holds the plan for auditing an [EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) cluster and the commands to do so. The objective is to walk through the main components of a [Kubernetes](https://kubernetes.io/docs/home/) cluster and identify what needs to be secured. Kubernetes has many moving parts. This section focuses on the ones that carry the most security risk.

If a resource type referenced below does not seem to exist in the cluster, use the following command to see what resource types the cluster actually has:

```bash
kubectl api-resources -o wide
```

## List Clusters and Check Versions

Running a Kubernetes version that is past its end-of-life means no security patches are being issued for it. EKS supports a specific set of Kubernetes minor versions at any given time and drops support on a regular schedule. The kubelet version on each node must also match or be within one minor version of the control plane.

**What to check:**
* The cluster is running a Kubernetes version currently supported by EKS
* Node groups are running a kubelet version compatible with the control plane
* The EKS platform version is current

List all EKS clusters in the account:

```bash
aws eks list-clusters \
  --query 'clusters'
```

Describe a cluster to see its Kubernetes version and platform version (substitute `<cluster-name>` from above):

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --query 'cluster.{Name:name,Version:version,PlatformVersion:platformVersion,Status:status}'
```

List all node groups and their AMI and kubelet versions:

```bash
aws eks list-nodegroups \
  --cluster-name <cluster-name> \
  --query 'nodegroups'
```

```bash
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query 'nodegroup.{Status:status,AmiType:amiType,Version:version,ReleaseVersion:releaseVersion}'
```

## Add a Cluster to kubeconfig

Add the cluster to the local [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) so that `kubectl` commands can reach it. Pass `--profile` if the cluster is in a different account than the default profile:

```bash
aws eks update-kubeconfig \
  --name <cluster-name> \
  --region <region>
```

> Use tools like [kubectx](https://github.com/ahmetb/kubectx) or [K9s](https://k9scli.io/) to manage multiple cluster contexts efficiently.

## Check for a Private Control Plane

The Kubernetes API server is the control plane endpoint that `kubectl` communicates with. A publicly exposed API server is reachable from the internet and is a target for credential stuffing and API exploits. The recommended configuration is private-only access, where the API server is only reachable from within the VPC. If public access must remain on, it should be restricted to specific CIDR blocks.

> Tools like [Teleport](https://goteleport.com) also can make private endpoints accessable via their proxy, so no need for VPN connections.

**What to check:**
* `endpointPublicAccess` is `false`, or if `true`, `publicAccessCidrs` does not contain `0.0.0.0/0`
* `endpointPrivateAccess` is `true`
* Node groups are deployed in private subnets, not public ones

Check the cluster's endpoint configuration:

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --query 'cluster.resourcesVpcConfig.{PublicAccess:endpointPublicAccess,PrivateAccess:endpointPrivateAccess,PublicCIDRs:publicAccessCidrs}'
```

Check which subnets each node group is using and confirm they are private subnets:

```bash
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query 'nodegroup.{NodegroupName:nodegroupName,Subnets:subnets}'
```

Cross-reference the subnet IDs against the VPC section findings to confirm they are private and route through a NAT Gateway rather than an internet gateway.

## Check etcd Encryption at Rest

[etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) is the key-value store that holds all Kubernetes cluster state, including Secrets. Without envelope encryption configured, Secrets are stored in plaintext in etcd. Anyone who gains access to the etcd data through a backup, a snapshot, or direct storage access can read every Secret in the cluster.

**What to check:**
* An `encryptionConfig` is defined on the cluster
* The resource type `secrets` is listed in the encryption configuration
* The encryption provider references a KMS key, not a static key

Check the cluster's encryption configuration. An empty result means no encryption is configured:

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --query 'cluster.encryptionConfig'
```

## Check Control Plane Logging

EKS can send control plane component logs to CloudWatch. There are five log types: `api` (API server requests), `audit` (every call to the Kubernetes API with the caller identity and outcome), `authenticator` (authentication decisions made by AWS IAM Authenticator), `controllerManager`, and `scheduler`. The `audit` log is the most security-relevant. It is the Kubernetes equivalent of CloudTrail and records who called what API and whether it was allowed or denied. Without it, there is no way to investigate suspicious activity in the cluster after the fact.

**What to check:**
* The `audit` log type is enabled and sending to CloudWatch
* Ideally all five log types are enabled for full visibility
* CloudWatch log groups for the cluster have an appropriate retention period set

Check which log types are currently enabled on the cluster:

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --query 'cluster.logging.clusterLogging'
```

The output shows two entries: `types` that are enabled and `types` that are disabled. Any configuration where `audit` appears in the disabled list is a finding.

## Check EKS Add-on Versions

EKS managed add-ons run inside the cluster and handle critical functions: `vpc-cni` manages pod networking, `kube-proxy` handles service routing, `coredns` handles cluster DNS, and `eks-pod-identity-agent` supports Pod Identities. Each add-on has its own release cycle and carries its own CVEs when outdated. AWS publishes the latest supported version for each add-on per Kubernetes minor version.

**What to check:**
* All installed add-ons are at the latest or near-latest version for the cluster's Kubernetes version
* No add-on is in a `DEGRADED` or `CREATE_FAILED` state
* The VPC CNI add-on is installed and managed (not self-managed), so AWS can update it

List all add-ons installed on the cluster:

```bash
aws eks list-addons \
  --cluster-name <cluster-name>
```

Describe a specific add-on to see its current version and status (substitute `<addon-name>` such as `vpc-cni`, `kube-proxy`, `coredns`):

```bash
aws eks describe-addon \
  --cluster-name <cluster-name> \
  --addon-name <addon-name> \
  --query 'addon.{Name:addonName,Version:addonVersion,Status:status,UpdatedAt:modifiedAt}'
```

Check what the latest available versions are for a specific add-on and Kubernetes version:

```bash
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version <k8s-version> \
  --query 'addons[*].addonVersions[0].addonVersion'
```

## Nodes

Node groups are the EC2 instances that run workloads. The node section focuses on version currency, placement, operating system choice, and the IAM permissions the nodes carry.

**What to check:**
* All nodes show a `Ready` status with no persistent `NotReady` nodes
* Node AMI versions are current and not significantly behind the latest release
* Nodes are spread across at least three availability zones
* Node groups are in private subnets (see the control plane check above for the subnet cross-reference)

List all nodes and their status, version, and which zone they are in:

```bash
kubectl get nodes \
  -o wide
```

Write the following to a file and run it to list each node group and confirm its nodes are in private subnets:

```bash
#!/usr/bin/env bash
for ng in $(aws eks list-nodegroups \
  --cluster-name <cluster-name> \
  --query 'nodegroups' --output text); do
  echo "=== Nodegroup: $ng ==="
  aws eks describe-nodegroup \
    --cluster-name <cluster-name> \
    --nodegroup-name "$ng" \
    --query 'nodegroup.{Subnets:subnets,AmiType:amiType,Version:version}'
done
```

### Check Node Groups Span Multiple Availability Zones

A node group confined to a single AZ has no resilience. If that AZ has an outage, every node in the group goes down simultaneously. The recommended pattern is one private subnet per AZ, with three AZs, so the scheduler can distribute pods across zones. Kubernetes does not automatically rebalance pods across AZs. Spreading is done at scheduling time, so nodes must exist in all target AZs before workloads are deployed.

**What to check:**
* Each node group's subnet list covers at least three different availability zones
* The subnets are distributed evenly across AZs, not weighted toward one

Write the following to a file and run it to check which AZs each node group's subnets are in:

```bash
#!/usr/bin/env bash
for ng in $(aws eks list-nodegroups \
  --cluster-name <cluster-name> \
  --query 'nodegroups' --output text); do
  echo "=== Nodegroup: $ng ==="
  subnet_ids=$(aws eks describe-nodegroup \
    --cluster-name <cluster-name> \
    --nodegroup-name "$ng" \
    --query 'nodegroup.subnets' --output text)
  aws ec2 describe-subnets \
    --subnet-ids $subnet_ids \
    --query 'Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,Public:MapPublicIpOnLaunch}'
done
```

Any node group with subnets in fewer than three AZs, or with `MapPublicIpOnLaunch` set to `true`, is a finding.

### Check the Node IAM Role

Each node group has an IAM role that every node in the group assumes. This role gives the node the permissions it needs to join the cluster, pull container images from ECR, and call AWS APIs for networking. The role needs three specific managed policies and nothing more. Any additional permissions on the node role are permissions that every workload running on that node could potentially access if the pod escapes its container boundary.

The required policies are:
* `AmazonEKSWorkerNodePolicy` — allows nodes to call EKS APIs to register with the cluster
* `AmazonEC2ContainerRegistryReadOnly` — allows nodes to pull images from ECR
* `AmazonEKS_CNI_Policy` — allows the VPC CNI plugin to manage ENIs (this can instead be moved to the VPC CNI service account via IRSA, which is the more locked-down approach)

**What to check:**
* The node role has exactly the three required policies and nothing additional
* The node role does not have `AdministratorAccess`, broad S3 permissions, or any IAM permissions
* The `AmazonEKS_CNI_Policy` has been moved to the VPC CNI service account via IRSA rather than staying on the node role

Get the node role ARN for a node group:

```bash
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query 'nodegroup.nodeRole'
```

List the policies attached to the node role (substitute the role name from the ARN above):

```bash
aws iam list-attached-role-policies \
  --role-name <node-role-name> \
  --query 'AttachedPolicies[*].{PolicyName:PolicyName,PolicyArn:PolicyArn}'
```

### Check Node OS: Bottlerocket vs Amazon Linux

The default AMI type for EKS node groups is Amazon Linux 2 (`AL2_x86_64`). [Bottlerocket](https://aws.amazon.com/bottlerocket/) is an AWS-built, open-source operating system designed specifically for running containers. It has a significantly smaller attack surface than a general-purpose Linux distribution.

Key security properties of Bottlerocket:
* Immutable root filesystem — the OS cannot be modified at runtime
* No package manager, no SSH by default — dramatically reduces what an attacker can do with node access
* Read-only root partition with dm-verity integrity verification
* Automatic updates applied atomically, with rollback on failure
* SELinux enforcing by default

**What to check:**
* Node groups use `BOTTLEROCKET_x86_64` or `BOTTLEROCKET_ARM_64` as the AMI type
* If Amazon Linux 2 is in use, document the reason and note it as a recommendation to migrate

Check the AMI type for each node group:

```bash
aws eks describe-nodegroup \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --query 'nodegroup.{AmiType:amiType,ReleaseVersion:releaseVersion}'
```

## Namespaces

[Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) are the primary way Kubernetes organizes and isolates resources. The `default` namespace is a common dumping ground for workloads that were deployed without a namespace specified. Production workloads should live in dedicated namespaces with their own RBAC, resource quotas, and network policies. The `kube-system` namespace contains cluster-level system components and should be treated as off-limits for application workloads.

**What to check:**
* No application workloads are deployed in the `default` namespace
* Each application namespace has a `ResourceQuota` defined to prevent a single workload from consuming all cluster resources
* Each namespace has a `LimitRange` to set default resource requests and limits for pods that do not define their own

List all namespaces:

```bash
kubectl get namespaces
```

Check what is running in the default namespace. Any application workload here is a finding:

```bash
kubectl get all -n default
```

Check which namespaces are missing a ResourceQuota:

```bash
kubectl get resourcequota \
  --all-namespaces
```

Check which namespaces are missing a LimitRange:

```bash
kubectl get limitrange \
  --all-namespaces
```

## Workloads

Workloads are the applications running inside the cluster. [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), and standalone [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) are the main types. List them out across all namespaces and understand what is running before auditing individual configurations.

List all workloads across all namespaces:

```bash
kubectl get deployments,statefulsets,daemonsets,pods \
  --all-namespaces
```

### Labels

Kubernetes labels are the equivalent of AWS tags. They are used for identifying ownership, routing traffic, applying policies, and incident attribution. The following labels are recommended on all workloads:

* `app.kubernetes.io/name` — the name of the application
* `app.kubernetes.io/owner` or `team` — the owning team
* `environment` — `production`, `staging`, `development`
* `data-classification` — sensitivity of data the workload handles
* `cost-center` — for billing attribution
* `response-sla` — how quickly the owning team responds to incidents

Check labels on all pods in a namespace (substitute `<namespace>`):

```bash
kubectl get pods \
  -n <namespace> \
  --show-labels
```

### Resource Limits and Requests

Kubernetes workloads need [resource limits and requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) defined. A pod with no limits can consume all CPU and memory on a node and starve other workloads. A pod with no requests cannot be scheduled accurately. Both are required for the cluster to function predictably under load.

**What to check:**
* Every container has `resources.requests.cpu` and `resources.requests.memory` defined
* Every container has `resources.limits.cpu` and `resources.limits.memory` defined
* No container has limits set dramatically higher than its requests without a documented reason

Write the following to a file and run it to find pods missing resource definitions in a namespace:

```bash
#!/usr/bin/env bash
kubectl get pods -n <namespace> -o json | \
  jq -r '.items[] | select(
    .spec.containers[].resources.limits == null or
    .spec.containers[].resources.requests == null
  ) | .metadata.name'
```

## Pod Security Context

Pods can have a [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) that defines privilege and access control settings. These settings exist at two levels: the pod spec (applying to all containers in the pod) and the individual container spec (applying to a single container). Security contexts are one of the most impactful areas to audit because a single privileged container can be enough to escape the container boundary and gain access to the host node.

### Pod-Level Security Context

The pod-level spec applies to all containers in the pod. The most important settings are:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
```

**What to check:**
* `runAsNonRoot` is `true` — the container process should never run as UID 0
* `runAsUser` is set to a non-zero UID
* No pod has `hostPID: true`, `hostIPC: true`, or `hostNetwork: true`. These settings share the node's process, IPC, or network namespaces with the container and are rarely legitimate

### Container-Level Security Context

The container-level spec applies to a single container and can override the pod-level settings:

```yaml
containers:
  - name: my-app
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

**What to check:**
* `allowPrivilegeEscalation` is `false` on every container
* `privileged` is `false`. A privileged container has nearly full access to the host
* `readOnlyRootFilesystem` is `true` where possible
* `capabilities` drops `ALL` and adds back only what is explicitly required

Write the following to a file and run it to find any privileged containers or containers allowing privilege escalation across the cluster:

```bash
#!/usr/bin/env bash
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .metadata.name as $pod |
  .spec.containers[] |
  select(
    .securityContext.privileged == true or
    .securityContext.allowPrivilegeEscalation == true
  ) |
  "\($ns)/\($pod): privileged=\(.securityContext.privileged) escalation=\(.securityContext.allowPrivilegeEscalation)"
'
```

These configurations can be enforced by Pod Security Admission, described below.

### Pod Security Admission

[Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) (PSA) is the Kubernetes-native way to enforce security context standards at the namespace level. Rather than relying on developers to set the right fields, PSA gives the cluster a way to reject pods that do not meet a minimum security bar before they are ever scheduled.

PSA works by adding labels to a namespace. When a pod is submitted to that namespace, the admission controller checks the pod spec against the policy level defined in the label. If the pod fails the check, the behavior depends on the mode configured:

* `enforce` — the pod is rejected outright and will not be created
* `audit` — the pod is created but a violation is logged to the audit log. Useful for detecting what would break before switching to enforce
* `warn` — the pod is created but the API server returns a warning to the caller. Helpful during migrations

Each mode is independent and can be set to a different policy level on the same namespace. A common rollout pattern is to start with `warn` and `audit` at `restricted`, observe what breaks, fix violations, and then move to `enforce`.

The three policy levels are:

* `privileged` — no restrictions, used for system-level namespaces like `kube-system`
* `baseline` — blocks the most dangerous configurations (privileged containers, hostPID, etc.)
* `restricted` — the strictest policy, requires most of the security context settings above

An example namespace label that enforces `restricted` and also warns on violations looks like:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

Check whether PSA labels are applied to namespaces:

```bash
kubectl get namespaces \
  -o json | jq -r '.items[] | "\(.metadata.name): \(.metadata.labels | to_entries | map(select(.key | startswith("pod-security"))) | from_entries)"'
```

Any namespace running application workloads that shows no `pod-security` labels is operating with no enforcement. That is worth flagging as a finding.

PSA is built into Kubernetes and requires no additional installs, but it has limits. It can only enforce the fields defined by the [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/). It cannot enforce custom rules like "all images must come from our private registry" or "no pods may use the `latest` tag." For those use cases, a third-party admission controller is the right tool.

### Admission Controllers

An [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) is a piece of code that runs in the Kubernetes API server pipeline after a request is authenticated and authorized, but before the resource is written to etcd. Every `kubectl apply` passes through all registered admission controllers before anything is persisted. There are two types:

* **Validating admission controllers** inspect the request and either allow or deny it. They cannot change the resource.
* **Mutating admission controllers** can modify the resource before it is persisted. They can inject values, add labels, or set default fields automatically.

PSA is a validating admission controller built into Kubernetes. Beyond PSA, teams commonly deploy external admission controllers using the [Admission Webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) mechanism, which lets any service in the cluster act as an admission controller.

Common third-party admission controllers to look for:

**[Kyverno](https://kyverno.io/)** is a Kubernetes-native policy engine. Policies are written as Kubernetes YAML resources rather than a separate policy language. Kyverno can validate, mutate, and generate resources. A typical policy would enforce image registry restrictions, require specific labels on all workloads, or auto-inject a security context if one is missing. Check if it is installed:

```bash
kubectl get pods \
  -n kyverno
```

```bash
kubectl get clusterpolicies
```

**[OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)** uses [Open Policy Agent](https://www.openpolicyagent.org/) (OPA) as its policy engine. Policies are written in a language called [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/). OPA is more expressive than Kyverno but has a steeper learning curve. It is a good fit for organizations that already use OPA elsewhere (for example, in Terraform or service mesh authorization). Check if it is installed:

```bash
kubectl get pods \
  -n gatekeeper-system
```

```bash
kubectl get constrainttemplates
```

**[Istio](https://istio.io/)** is a service mesh that uses a mutating admission controller to inject a sidecar proxy container into every pod at scheduling time. The sidecar handles mTLS between services, traffic routing, and observability without the application needing to be aware of it. Istio is not primarily a security policy tool, but because it is a mutating admission controller it touches every pod in the mesh. Check if the sidecar injector is running:

```bash
kubectl get pods \
  -n istio-system
```

```bash
kubectl get mutatingwebhookconfigurations
```

```bash
kubectl get validatingwebhookconfigurations
```

Any `MutatingWebhookConfiguration` or `ValidatingWebhookConfiguration` on the cluster represents a service that is intercepting all or some API requests. Review what is registered and whether it is expected. A webhook that is misconfigured or pointing to a service that is down can cause all pod creation in the cluster to fail.

## Services

Kubernetes uses [Services](https://kubernetes.io/docs/concepts/services-networking/service/) to expose groups of pods on the network. The service type determines who can reach it. `ClusterIP` is reachable only within the cluster. `NodePort` opens a port on every node. `LoadBalancer` provisions an AWS load balancer and exposes the service externally. Most internal services should be `ClusterIP`.

**What to check:**
* No services are of type `LoadBalancer` unless they are intentionally public-facing and covered by the WAF/ALB architecture from the Networking section
* No services are of type `NodePort` without a documented reason
* Services are selecting the correct pods via their label selectors

List all services and their types across all namespaces:

```bash
kubectl get services \
  --all-namespaces \
  -o wide
```

Write the following to a file and run it to flag any `LoadBalancer` or `NodePort` services:

```bash
#!/usr/bin/env bash
kubectl get services --all-namespaces -o json | jq -r '
  .items[] |
  select(.spec.type == "LoadBalancer" or .spec.type == "NodePort") |
  "\(.metadata.namespace)/\(.metadata.name): \(.spec.type)"
'
```

## CNI

CNI stands for Container Network Interface. It is the plugin responsible for giving every pod an IP address and connecting it to the network. When the kubelet schedules a pod onto a node, it calls the CNI plugin to set up the pod's network interface, assign an IP, and configure the routing rules so that pod can send and receive traffic. Without a CNI, pods have no network.

On EKS, AWS ships its own CNI called the [VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s) (`aws-node`). It assigns each pod a real IP address pulled directly from the VPC subnet. This means pods are first-class citizens on the VPC network, reachable by other AWS services and resources without NAT or an overlay network. It is the default CNI on every EKS cluster.

Other CNIs seen in Kubernetes environments:

* **[Calico](https://www.tigera.io/project-calico/)** — a widely used CNI with strong network policy enforcement. Often installed alongside the VPC CNI on EKS specifically to gain its policy engine
* **[Cilium](https://cilium.io/)** — uses eBPF in the Linux kernel for high-performance networking. Supports Layer 7 HTTP-aware network policies beyond what standard Kubernetes NetworkPolicy resources can express
* **[Weave](https://www.weave.works/oss/net/)** — an older overlay-based CNI, less common on EKS today

The CNI has a direct security impact because it is responsible for enforcing network policies. If the CNI does not support network policy enforcement, any NetworkPolicy resources in the cluster are decorative and have no effect on traffic. This is covered in the Network Policies section below.

Check which CNI is running and its version:

```bash
kubectl get pods \
  -n kube-system \
  -o wide
```

```bash
kubectl describe daemonset aws-node \
  -n kube-system | grep -i image
```

Look for `aws-node` pods for the VPC CNI, `calico-node` for Calico, or `cilium` for Cilium. The version reported by `describe` should be checked against the [EKS add-on release notes](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) for known CVEs and whether the version is still supported for the cluster's Kubernetes version.

## Network Policies

Kubernetes applies no network restrictions between pods, by default. Every pod in the cluster can reach every other pod on any port. [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) are the Kubernetes resource that implements microsegmentation, restricting which pods can communicate with which. Without them, a compromised pod can freely scan and connect to any other pod in the cluster, including databases, internal APIs, and the Kubernetes API server.

Network policies require a CNI that supports enforcement. If the CNI does not, the resources can be created but will have no effect. Creating policies against an unsupporting CNI gives a false sense of security, so confirm the CNI first using the section above.

List all network policies across all namespaces to get a baseline picture:

```bash
kubectl get networkpolicies \
  --all-namespaces \
  -o wide
```

## Check for a Default-Deny Policy in Every Namespace

The most important network policy pattern is a default-deny. Without it, any pod can reach any other pod even if additional allow policies exist. The allow policies are additive, so the only way to ensure a namespace has a closed posture is to start with a policy that denies everything and then explicitly allow only the traffic that is needed.

A compliant default-deny policy looks like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

The empty `podSelector: {}` matches every pod in the namespace. Listing both `Ingress` and `Egress` in `policyTypes` with no rules denies all traffic in both directions. A policy that only lists `Ingress` will still allow all egress, which is a partial control.

Write the following to a file and run it to identify namespaces with no network policies defined:

```bash
#!/usr/bin/env bash
all_ns=$(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}')
for ns in $all_ns; do
  count=$(kubectl get networkpolicies -n "$ns" --no-headers 2>/dev/null | wc -l)
  if [ "$count" -eq 0 ]; then
    echo "NO NETWORK POLICIES: $ns"
  fi
done
```

Write the following to a file and run it to check which namespaces have policies but are missing a default-deny that covers both ingress and egress:

```bash
#!/usr/bin/env bash
kubectl get networkpolicies --all-namespaces -o json | jq -r '
  .items |
  group_by(.metadata.namespace) |
  .[] |
  . as $policies |
  ($policies[0].metadata.namespace) as $ns |
  if ($policies | map(
    select(
      .spec.podSelector == {} and
      (.spec.policyTypes | (contains(["Ingress"]) and contains(["Egress"]))) and
      (.spec.ingress == null or .spec.ingress == []) and
      (.spec.egress == null or .spec.egress == [])
    )
  ) | length) == 0
  then "\($ns): no default-deny-all policy found"
  else empty
  end
'
```

> **CIS Reference:** CIS Kubernetes Benchmark v1.8 — **5.3.2** (Ensure that all Namespaces have Network Policies defined).

## Check for Overly Permissive Policies

A network policy with an empty `podSelector` and no port or peer restrictions on an allow rule is functionally equivalent to having no policy. This pattern can appear when a developer creates a policy to "fix" a connectivity issue without understanding what it opens up.

Write the following to a file and run it to find policies that allow ingress or egress from all pods with no port restriction:

```bash
#!/usr/bin/env bash
kubectl get networkpolicies --all-namespaces -o json | jq -r '
  .items[] |
  . as $pol |
  (.metadata.namespace + "/" + .metadata.name) as $name |
  (
    (.spec.ingress // [] | map(select(
      (.from == null or .from == []) and
      (.ports == null or .ports == [])
    )) | length > 0) or
    (.spec.egress // [] | map(select(
      (.to == null or .to == []) and
      (.ports == null or .ports == [])
    )) | length > 0)
  ) |
  if . then "\($name): overly permissive rule detected" else empty end
'
```

## Check for Namespace Isolation Between Tenants

If the cluster hosts multiple teams or environments in separate namespaces, network policies should prevent cross-namespace traffic unless explicitly required. A pod in a staging namespace should not be able to reach a pod in the production namespace. Without explicit namespace selectors in allow rules, nothing stops this traffic.

Check whether any policies use `namespaceSelector` to allow cross-namespace traffic:

```bash
kubectl get networkpolicies \
  --all-namespaces \
  -o json | jq -r '
    .items[] |
    select(
      (.spec.ingress // [] | map(select(.from // [] | map(.namespaceSelector) | any)) | length > 0) or
      (.spec.egress // [] | map(select(.to // [] | map(.namespaceSelector) | any)) | length > 0)
    ) |
    "\(.metadata.namespace)/\(.metadata.name): allows cross-namespace traffic"
  '
```

Review each result and confirm whether the cross-namespace access is intentional and documented.

## Verify the CNI Supports Network Policies

Creating network policies against a CNI that does not enforce them is a false control. The policies will exist in etcd and look compliant, but no traffic will actually be blocked. Check which CNI is running in the cluster:

```bash
kubectl get pods \
  -n kube-system \
  -o wide
```

Look for pods with names like `calico`, `cilium`, `weave`, or `aws-node` (the AWS VPC CNI). The AWS VPC CNI that ships with EKS by default does not support network policies on its own. EKS added [network policy support to the VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy.html) starting in version 1.14 of the add-on, but it must be explicitly enabled. If the cluster is using the VPC CNI and network policies are defined, confirm the network policy feature is actually enabled:

```bash
kubectl describe daemonset aws-node \
  -n kube-system | grep -i "enable-network-policy"
```

If neither Calico, Cilium, nor an enabled VPC CNI network policy feature is present, all existing network policies are decorative and every namespace is effectively open.

> **CIS Reference:** CIS Kubernetes Benchmark v1.8 — **5.3.1** (Ensure that the CNI in use supports Network Policies) and **5.3.2** (Ensure that all Namespaces have Network Policies defined).

## Gateway

[Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/) is the current standard for routing traffic into and out of the cluster, replacing the older Ingress resource. Gateways use a controller (such as the AWS Load Balancer Controller) to provision and manage external load balancers. The Gateway and its routes define exactly which traffic is allowed in and where it goes.

**What to check:**
* The Gateway controller is running a supported, up-to-date version
* No `HTTPRoute` or `GRPCRoute` resources expose internal services that should not be externally reachable
* All routes require TLS, meaning no unencrypted HTTP routes in production
* Unused Gateway resources are removed

List all Gateways and their status:

```bash
kubectl get gateways \
  --all-namespaces \
  -o wide
```

List all HTTPRoutes and what they route to:

```bash
kubectl get httproutes \
  --all-namespaces \
  -o wide
```

## ConfigMaps

[ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) inject configuration into pods as environment variables or files on the container filesystem. The most common finding is secrets stored in ConfigMaps (database passwords, API keys, tokens) because it was convenient and the developer did not want to deal with Kubernetes Secrets. ConfigMaps are not encrypted and anyone with read access to the namespace can read them in plaintext.

**What to check:**
* No ConfigMap contains passwords, tokens, API keys, or any value that should be a Secret
* No ConfigMap is deployed in the `default` namespace

List all ConfigMaps across all namespaces:

```bash
kubectl get configmaps \
  --all-namespaces
```

Inspect a specific ConfigMap for sensitive values (substitute `<name>` and `<namespace>`):

```bash
kubectl get configmap <name> \
  -n <namespace> \
  -o yaml
```

Write the following to a file and run it to scan ConfigMaps for common patterns that suggest secrets are stored in them:

```bash
#!/usr/bin/env bash
kubectl get configmaps --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)" as $cm |
    .data // {} | to_entries[] |
    select(.key | test("password|secret|token|key|credential"; "i")) |
    "\($cm): suspicious key: \(.key)"'
```

## Secrets

[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) are the Kubernetes resource for injecting sensitive information into pods. They are base64-encoded, not encrypted. Anyone with API access can retrieve and decode them. The Kubernetes documentation states this directly:

> Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. Additionally, anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace; this includes indirect access such as the ability to create a Deployment.

> In order to safely use Secrets, take at least the following steps:
> * Enable Encryption at Rest for Secrets.
> * Enable or configure RBAC rules with least-privilege access to Secrets.
> * Restrict Secret access to specific containers.
> * Consider using external Secret store providers.

**What to check:**
* etcd encryption is configured for Secrets (covered in the etcd section above)
* RBAC restricts which service accounts and users can read Secrets
* An external secrets operator is in use. Tools like [External Secrets Operator](https://external-secrets.io/) or [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/) pull secrets from AWS Secrets Manager or Parameter Store at runtime, keeping them out of etcd entirely

List all Secrets across all namespaces:

```bash
kubectl get secrets \
  --all-namespaces
```

Check whether an external secrets solution is installed:

```bash
kubectl get pods \
  --all-namespaces \
  -l 'app.kubernetes.io/name in (external-secrets,secrets-store-csi-driver)'
```

## RBAC

[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (Role-Based Access Control) governs what every identity in the cluster is allowed to do. The core resources are `Role` and `ClusterRole` (which define permissions), and `RoleBinding` and `ClusterRoleBinding` (which assign those permissions to identities). The scope difference: a `Role` is namespace-scoped, a `ClusterRole` applies across the entire cluster.

### ClusterRoles and ClusterRoleBindings

A `ClusterRoleBinding` that assigns `cluster-admin` to any non-system identity gives that identity unrestricted access to every resource in every namespace. This is the Kubernetes equivalent of `AdministratorAccess` in IAM. It is a common finding in clusters where operators needed quick access and granted cluster-admin rather than creating a scoped role.

**What to check:**
* `cluster-admin` is bound only to system-level service accounts (`system:masters`, `system:node`, etc.) and specific documented admin identities
* No `ClusterRole` grants wildcard verbs (`*`) on sensitive resources like `secrets`, `pods/exec`, or `*`
* `RoleBindings` and `ClusterRoleBindings` reference identities that still exist and are still needed

List all ClusterRoleBindings and what they grant to whom:

```bash
kubectl get clusterrolebindings \
  -o wide
```

Check exactly which subjects have `cluster-admin` bound to them:

```bash
kubectl get clusterrolebindings \
  -o json | jq -r '
    .items[] |
    select(.roleRef.name == "cluster-admin") |
    "\(.metadata.name): \(.subjects // [] | map("\(.kind)/\(.name)") | join(", "))"
  '
```

List all ClusterRoles that grant wildcard verbs:

```bash
kubectl get clusterroles \
  -o json | jq -r '
    .items[] |
    select(.rules[]?.verbs[]? == "*") |
    .metadata.name
  '
```

### IRSA and EKS Pod Identities

Pods that need AWS API access (reading from S3, calling Bedrock, writing to DynamoDB) need AWS credentials. The wrong answers are hardcoding access keys in the pod spec or relying on the node's IAM role (all pods on the node would share those permissions). The right answer is to give each workload its own IAM role scoped to exactly what it needs. AWS provides two mechanisms to do this:

* [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
* [EKS Pod Identities](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)

Below the sections will discuss both options and how to identify which are used in the cluster. Either are acceptable or using a third-party identity provider like Teleport. Hard-coded credentials in the environment or secret files are not options and should be flagged as critical findings. 

#### IRSA

IRSA uses [OIDC federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html). The full setup has four parts that must all be in place for it to work:

**1. The OIDC provider is registered in IAM.** The EKS cluster has an OIDC issuer URL. That URL is registered in IAM as a trusted identity provider, which tells AWS "tokens signed by this cluster's control plane are valid credentials."

**2. The IAM role has a trust policy scoped to a specific service account.** The trust policy allows the OIDC provider to assume the role, but only when the token's `sub` claim matches a specific Kubernetes service account in a specific namespace.

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:my-namespace:my-service-account"
    }
  }
}
```

The `Condition` clause is the critical scoping mechanism. Without it, any pod in any namespace using any service account in the cluster could assume the role.

**3. The Kubernetes service account is annotated with the IAM role ARN.** This is what links the Kubernetes identity to the IAM role.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: my-namespace
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-role
```

**4. The pod spec references the service account.** This is the final step and the one most commonly missing or misconfigured. The pod must explicitly declare `serviceAccountName` in its spec. If it does not, it gets the `default` service account, which has no role annotation and no AWS permissions.

```yaml
spec:
  serviceAccountName: my-service-account
  containers:
    - name: my-app
      image: my-image
```

When the pod is created, the [EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook) (a mutating admission controller running in the cluster) sees the service account annotation and automatically injects two things into the pod:

* An environment variable `AWS_ROLE_ARN` set to the role ARN from the annotation
* An environment variable `AWS_WEB_IDENTITY_TOKEN_FILE` pointing to a projected volume mount at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token`

The AWS SDK reads these environment variables automatically. It picks up the token file, calls STS `AssumeRoleWithWebIdentity`, and receives short-lived temporary credentials. None of this requires the developer to write any credential handling code.

To confirm a running pod has had IRSA credentials injected, inspect its environment:

```bash
kubectl exec -n <namespace> <pod-name> -- env | grep AWS
```

You should see `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`. If you see `AWS_ACCESS_KEY_ID` instead, that is a different (and worse) credential mechanism and should be flagged.

#### EKS Pod Identities

EKS Pod Identities is the newer mechanism (launched 2023) and is simpler to operate at scale. The full setup has three parts:

**1. The `eks-pod-identity-agent` add-on is installed.** This runs as a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) on every node. It is the agent that intercepts credential requests from pods and fulfills them using the association mapping. Without it, no Pod Identity associations will work.

**2. A Pod Identity Association is created in EKS.** Instead of annotating service accounts, the mapping is stored in EKS directly. An association ties a specific namespace and service account name to an IAM role. The IAM role's trust policy only needs to trust the `pods.eks.amazonaws.com` service principal, and that trust policy works for any cluster without modification.

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "pods.eks.amazonaws.com"
  },
  "Action": [
    "sts:AssumeRole",
    "sts:TagSession"
  ]
}
```

**3. The pod spec references the service account.** Just like IRSA, the pod must declare `serviceAccountName` in its spec to pick up the association. The service account itself does not need any annotation for Pod Identities; the mapping lives entirely in EKS.

```yaml
spec:
  serviceAccountName: my-service-account
  containers:
    - name: my-app
      image: my-image
```

When the pod starts on a node, the `eks-pod-identity-agent` intercepts any AWS SDK credential request destined for the local credential endpoint (`169.254.170.23`). It looks up whether there is a Pod Identity Association for the pod's namespace and service account, assumes the mapped role via STS, and returns short-lived credentials to the SDK. The developer does not need to do anything else. No webhook injects environment variables; the agent handles everything at the network level on the node.

To confirm a running pod is receiving credentials through the Pod Identity agent, check whether the agent is healthy and then inspect the pod's service account:

```bash
kubectl get daemonset eks-pod-identity-agent \
  -n kube-system
```

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.spec.serviceAccountName}'
```

Then verify that service account has a Pod Identity Association:

```bash
aws eks list-pod-identity-associations \
  --cluster-name <cluster-name> \
  --namespace <namespace> \
  --service-account <service-account-name>
```

**When to use which:** New workloads should use Pod Identities. It is simpler, does not require per-cluster OIDC provider setup, scales better across multiple clusters, and is AWS's recommended path going forward. Existing IRSA setups do not need to be migrated immediately, but note them and recommend eventual migration.

#### How to Detect Which Is in Use

Check whether the cluster has an OIDC issuer configured (indicates IRSA is possible):

```bash
aws eks describe-cluster \
  --name <cluster-name> \
  --query 'cluster.identity.oidc'
```

Check whether that OIDC provider is registered in IAM:

```bash
aws iam list-open-id-connect-providers
```

Check whether service accounts have the IRSA role annotation:

```bash
kubectl get serviceaccounts \
  --all-namespaces \
  -o json | jq -r '
    .items[] |
    select(.metadata.annotations."eks.amazonaws.com/role-arn" != null) |
    "\(.metadata.namespace)/\(.metadata.name): \(.metadata.annotations."eks.amazonaws.com/role-arn")"
  '
```

Check whether the Pod Identity agent add-on is installed:

```bash
kubectl get daemonset eks-pod-identity-agent \
  -n kube-system
```

List all Pod Identity Associations for the cluster:

```bash
aws eks list-pod-identity-associations \
  --cluster-name <cluster-name>
```

Describe a specific association to see which service account it maps to and which role it uses:

```bash
aws eks describe-pod-identity-association \
  --cluster-name <cluster-name> \
  --association-id <association-id> \
  --query 'association.{Namespace:namespace,ServiceAccount:serviceAccount,RoleArn:roleArn}'
```

#### Auditing the Permissions

Once the IAM role ARN is identified (either from the service account annotation for IRSA or from the Pod Identity Association), the IAM audit steps from the IAM section apply. Check the role for wildcard policies, `AdministratorAccess`, and any permissions that go beyond what the specific workload needs.

For IRSA, also verify the trust policy condition is scoped to the specific service account and not open to all service accounts in the cluster:

```bash
aws iam get-role \
  --role-name <role-name> \
  --query 'Role.AssumeRolePolicyDocument' \
  --output text | jq .
```

If the trust policy has no `Condition` block, or the `sub` condition uses a wildcard, any pod in the cluster can assume that role. That is a critical finding.

### What to Look for When Neither IRSA nor Pod Identities Are in Use

If the audit finds no IRSA configuration and no Pod Identity associations, workloads are still making AWS API calls somehow. The AWS SDK works through a fixed credential resolution chain in order until it finds something that works:

1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`)
2. Shared credentials file (`~/.aws/credentials`) on the container filesystem
3. Web identity token file (IRSA)
4. EKS Pod Identity agent
5. Instance metadata service (IMDS) — the node's IAM role

If neither IRSA nor Pod Identities are configured, the SDK falls all the way to step 5 and uses the node's IAM role. This means every pod on that node implicitly inherits the node's AWS permissions with no annotation, no configuration, and nothing visible in the pod spec. This is the most dangerous case because it is invisible. No credentials appear anywhere in the cluster, but every pod has AWS access through the node role.

The following checks cover all the ways a workload might be authenticating with AWS improperly.

#### Check for Hardcoded Credentials in Pod Environment Variables

The most obvious misconfiguration is a pod spec with `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set directly as plaintext environment variables. These are visible to anyone who can read the pod spec and are stored unencrypted in etcd.

Write the following to a file and run it to scan all pods for AWS credential environment variables set directly in the spec:

```bash
#!/usr/bin/env bash
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .metadata.name as $pod |
  .spec.containers[].env[]? |
  select(.name | test("AWS_ACCESS_KEY_ID|AWS_SECRET_ACCESS_KEY|AWS_SESSION_TOKEN")) |
  select(.value != null) |
  "\($ns)/\($pod): PLAINTEXT CREDENTIAL in env var: \(.name)"
'
```

Any output from this script is a critical finding.

#### Check for AWS Credentials Injected from Kubernetes Secrets

A slightly less obvious pattern is using a Kubernetes Secret to hold the access key and secret, then injecting the Secret values as environment variables via `valueFrom.secretKeyRef`. The credentials are not visible in plaintext in the pod spec, but they are still long-lived static keys that can be extracted by anyone who can read the Secret.

Write the following to a file and run it to find pods pulling environment variables from Secrets where the key name suggests AWS credentials:

```bash
#!/usr/bin/env bash
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .metadata.name as $pod |
  .spec.containers[].env[]? |
  select(.valueFrom.secretKeyRef != null) |
  select(.name | test("AWS_ACCESS_KEY|AWS_SECRET|AWS_SESSION"; "i")) |
  "\($ns)/\($pod): credential from Secret \(.valueFrom.secretKeyRef.name) key \(.valueFrom.secretKeyRef.key)"
'
```

#### Check Secrets for AWS Credential Content

Search the Secrets themselves for values that look like AWS access keys. AWS access key IDs always start with `AKIA` (long-term) or `ASIA` (temporary/STS). Finding one in a Secret means a long-lived credential is stored in etcd.

Write the following to a file and run it to scan Secret data for AWS access key patterns:

```bash
#!/usr/bin/env bash
kubectl get secrets --all-namespaces -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .metadata.name as $name |
  (.data // {}) | to_entries[] |
  select(.value != null) |
  .value |= (. | @base64d) |
  select(.value | test("AKIA|ASIA")) |
  "\($ns)/\($name): possible AWS access key in key \(.key)"
'
```

Note that Secret values are base64-encoded in the API response, which is why the script decodes them before checking. This is the same reason `base64 -d` was used in the credential report commands in the Account section.

#### Check for AWS Credential Files Mounted as Volumes

Some workloads mount an AWS credentials file directly into the container from a Secret or ConfigMap volume. The file is typically placed at `/root/.aws/credentials` or `/home/<user>/.aws/credentials` inside the container. Check for Secret or ConfigMap volumes with names that suggest AWS configuration.

Write the following to a file and run it to find pods with volumes referencing Secrets or ConfigMaps with AWS-related names:

```bash
#!/usr/bin/env bash
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .metadata.name as $pod |
  .spec.volumes[]? |
  select(
    (.secret.secretName | strings | test("aws|credential"; "i")) or
    (.configMap.name | strings | test("aws|credential"; "i"))
  ) |
  "\($ns)/\($pod): volume \(.name) references \(.secret.secretName // .configMap.name)"
'
```

#### Check for the Node Role Fallback

If none of the above checks produce findings but the cluster is still making AWS API calls, the workloads are using the node's IAM role through IMDS. This is the silent misconfiguration. Every pod on the node has the node's permissions. There is no annotation, no secret, and nothing visible in the pod spec.

Confirm the node role and its permissions using the commands in the Node IAM Role section above. Also check whether IMDSv2 is required (covered in the EC2 section). If IMDSv1 is still allowed, pods can reach IMDS without the token hop-limit restriction, making credential theft easier.

Check whether any service accounts in application namespaces have no IRSA annotation and no Pod Identity association, meaning they fall through to the node role:

```bash
kubectl get serviceaccounts \
  --all-namespaces \
  -o json | jq -r '
    .items[] |
    select(.metadata.namespace | test("kube-system|kube-public|kube-node-lease") | not) |
    select(.metadata.annotations."eks.amazonaws.com/role-arn" == null) |
    "\(.metadata.namespace)/\(.metadata.name): no IRSA annotation"
  '
```

Cross-reference this list against the Pod Identity associations retrieved earlier. Any service account that appears here and has no Pod Identity association is using the node role for AWS access.

## CRDs

[Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) extend the Kubernetes API with custom resource types. Most cluster add-ons (cert-manager, the AWS Load Balancer Controller, External Secrets Operator) install CRDs. The security concern is CRDs from unknown or unmanaged sources, and CRDs whose controllers have not been updated.

**What to check:**
* All CRDs have a known, documented source
* CRD controllers are running supported versions

List all CRDs and their creation date:

```bash
kubectl get crds \
  -o wide
```

# Relational Database Service (RDS)

[RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) runs managed relational databases as a fully managed service. The underlying EC2 instances, patching, and backups are handled by AWS. The audit focuses on what the customer controls: network exposure, encryption, access control, and backup configuration.

Start by listing all RDS instances in the account:

```bash
aws rds describe-db-instances \
  --query 'DBInstances[*].{Identifier:DBInstanceIdentifier,Engine:Engine,Version:EngineVersion,Status:DBInstanceStatus,PubliclyAccessible:PubliclyAccessible,StorageEncrypted:StorageEncrypted,MultiAZ:MultiAZ}'
```

## Check Public Accessibility

An RDS instance with `PubliclyAccessible` set to `true` has a DNS endpoint that resolves to a public IP address. Even if a security group restricts inbound access, a publicly accessible database is a higher-risk configuration and requires justification. Most RDS instances should be in private subnets with no public endpoint.

**What to check:**
* `PubliclyAccessible` is `false` on every instance
* Instances are in private subnets, not public ones
* Security groups allow inbound on the database port only from specific sources (application security groups or VPC CIDR blocks), not from `0.0.0.0/0`

List all publicly accessible instances:

```bash
aws rds describe-db-instances \
  --query 'DBInstances[?PubliclyAccessible==`true`].{Identifier:DBInstanceIdentifier,Engine:Engine,Endpoint:Endpoint.Address}'
```

Check the security groups attached to a specific instance to see what can reach it (substitute `<db-identifier>`):

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-identifier> \
  --query 'DBInstances[*].VpcSecurityGroups[*].VpcSecurityGroupId'
```

Then use the VPC section commands to inspect the rules on each of those security group IDs.

## Check Encryption at Rest

RDS storage encryption uses KMS to encrypt the underlying EBS volumes, automated backups, read replicas, and snapshots. Encryption must be enabled at creation time and cannot be added to an existing unencrypted instance without creating a new encrypted snapshot and restoring from it.

**What to check:**
* `StorageEncrypted` is `true` on every instance
* The KMS key in use is a customer-managed key, not the AWS-managed `aws/rds` default key
* Automated snapshots are also encrypted (they inherit the instance's encryption setting)

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **2.3.1** (Ensure that encryption-at-rest is enabled for RDS Instances).

List all unencrypted instances:

```bash
aws rds describe-db-instances \
  --query 'DBInstances[?StorageEncrypted==`false`].{Identifier:DBInstanceIdentifier,Engine:Engine}'
```

Check which KMS key an encrypted instance uses:

```bash
aws rds describe-db-instances \
  --db-instance-identifier <db-identifier> \
  --query 'DBInstances[*].{KmsKeyId:KmsKeyId,StorageEncrypted:StorageEncrypted}'
```

## Check IAM Database Authentication

RDS instances, by default, use the underlying database's authentication mechanism which is usually username and password. These are long-lived credentials stored in a config file, a Kubernetes Secret, or an environment variable, and they must be rotated manually. IAM database authentication replaces the password with a short-lived token generated by the AWS SDK using the caller's IAM identity. No password is stored, the token expires in 15 minutes, and access is controlled through IAM policies.

For PostgreSQL, the database user must be granted the `rds_iam` role. For MySQL, the user is created with `IDENTIFIED WITH AWSAuthenticationPlugin`.

**What to check:**
* `IAMDatabaseAuthenticationEnabled` is `true` on every instance
* Applications are configured to use IAM tokens, not static passwords
* The IAM role used to generate tokens follows least privilege (scoped to `rds-db:connect` on specific databases and users)

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **2.3.2** (Ensure that IAM Authentication is enabled for RDS Instances).

List all instances where IAM authentication is disabled:

```bash
aws rds describe-db-instances \
  --query 'DBInstances[?IAMDatabaseAuthenticationEnabled==`false`].{Identifier:DBInstanceIdentifier,Engine:Engine}'
```

## Check Automated Backups and Multi-AZ

Automated backups and Multi-AZ are availability and recovery controls. Automated backups enable point-in-time recovery. Multi-AZ maintains a synchronous standby replica in a separate availability zone and promotes it automatically if the primary fails. Both are required for any production database.

**What to check:**
* `BackupRetentionPeriod` is at least 7 days for production instances
* `MultiAZ` is `true` for production instances
* The backup window and maintenance window are set to off-peak hours

List instances with short or no backup retention:

```bash
aws rds describe-db-instances \
  --query 'DBInstances[?BackupRetentionPeriod < `7`].{Identifier:DBInstanceIdentifier,BackupRetentionPeriod:BackupRetentionPeriod,MultiAZ:MultiAZ}'
```

# Bedrock

[Bedrock](https://aws.amazon.com/bedrock/) is AWS's managed service for building and hosting generative AI applications using foundation models. The security concerns here are different from a traditional service. Bedrock can be used to process customer data, generate content on behalf of users, and in some configurations be fine-tuned on proprietary datasets. Each of these introduces data confidentiality and access control risks specific to AI workloads.

## Check IAM Permissions for Model Invocation

`bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` are the permissions that allow a principal to call a foundation model. These should be scoped to specific model ARNs and granted only to the identities that need them. A policy granting `bedrock:*` to a broad principal gives that identity the ability to invoke any model, list models, and manage Bedrock resources.

**What to check:**
* No IAM policy grants `bedrock:*` as a wildcard action
* `bedrock:InvokeModel` is scoped to specific model ARNs in the `Resource` field, not `*`
* The EKS pod roles or EC2 instance profiles that call Bedrock have the minimum necessary permissions

List all foundation models currently available in the account (this shows what models have been enabled for use):

```bash
aws bedrock list-foundation-models \
  --query 'modelSummaries[*].{ModelId:modelId,Provider:providerName,Status:modelLifecycle.status}'
```

Check which models have been granted access in the account:

```bash
aws bedrock list-foundation-model-agreements \
  --query 'modelSummaries[*].{ModelId:modelId,AgreementStatus:agreementStatus}'
```

## Check Model Invocation Logging

Invocation logging records every request sent to a Bedrock model, including the prompt and the response. Without it, there is no audit trail of what data was sent to a model or what was generated. This is a compliance requirement for SOC2 and ISO 27001 when Bedrock is processing customer data.

**What to check:**
* Invocation logging is enabled
* Logs are delivered to a CloudWatch log group or S3 bucket
* The log destination is in the central logging account, same as CloudTrail logs

Check the current invocation logging configuration. An empty or disabled result is a finding:

```bash
aws bedrock get-model-invocation-logging-configuration
```

## Check VPC Endpoint for Bedrock

Bedrock API calls travel over the public internet by default, even though the workload making the call is inside a VPC. A VPC endpoint for Bedrock keeps that traffic within the AWS network. This is the same principle as the S3 VPC endpoint which is more secure and has lower latency.

**What to check:**
* A VPC endpoint exists for `com.amazonaws.<region>.bedrock-runtime`
* EKS pods and EC2 instances calling Bedrock are in subnets that route through the endpoint

Check for Bedrock VPC endpoints:

```bash
aws ec2 describe-vpc-endpoints \
  --filters Name=service-name,Values=com.amazonaws.us-east-1.bedrock-runtime \
  --query 'VpcEndpoints[*].{EndpointId:VpcEndpointId,VpcId:VpcId,State:State}'
```

# Route 53

[Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html) is AWS's DNS service. It handles domain registration, public DNS resolution, and private DNS within VPCs. Security issues in DNS are high impact because DNS is foundational. A misconfigured or compromised DNS record can redirect traffic, break authentication, or expose internal infrastructure.

## Check Domain Transfer Lock

Route 53 registered domains can have a transfer lock enabled that prevents unauthorized domain transfers to another registrar. An unlocked domain can be transferred away from the account without additional verification. Every registered domain should have transfer lock on.

**What to check:**
* Every registered domain has `TransferLock` enabled
* Contact information for each domain is accurate and up to date

List all registered domains and their transfer lock status:

```bash
aws route53domains list-domains \
  --query 'Domains[*].{DomainName:DomainName,TransferLock:TransferLock,AutoRenew:AutoRenew}'
```

Get full details on a specific domain including contacts and lock status (substitute `<domain-name>`):

```bash
aws route53domains get-domain-detail \
  --domain-name <domain-name> \
  --query '{TransferLock:StatusList,AdminContact:AdminContact.Email,Tech:TechContact.Email}'
```

## Check DNSSEC

[DNSSEC](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring-dnssec.html) cryptographically signs DNS records so that resolvers can verify the response has not been tampered with. Without DNSSEC, an attacker in a position to intercept DNS responses can substitute malicious records. Route 53 supports DNSSEC signing for public hosted zones.

**What to check:**
* DNSSEC signing is enabled on public hosted zones for production domains
* The key-signing key (KSK) is in an `ACTIVE` state
* The DS record has been registered with the parent zone (without this, DNSSEC validation does not chain up to the root)

List all hosted zones and check which have DNSSEC enabled:

```bash
aws route53 list-hosted-zones \
  --query 'HostedZones[*].{Name:Name,Id:Id,Private:Config.PrivateZone}'
```

Check DNSSEC status for a specific hosted zone (substitute `<zone-id>`):

```bash
aws route53 get-dnssec \
  --hosted-zone-id <zone-id> \
  --query '{Status:Status,KeySigningKeys:KeySigningKeys[*].{Name:Name,Status:Status}}'
```

## Check DNS Query Logging

Route 53 query logging records every DNS query that reaches a public hosted zone. These logs are valuable for security monitoring. DNS is a common channel for command-and-control communication and data exfiltration, and query logs can surface that activity. For private hosted zones inside VPCs, query logging goes through Route 53 Resolver logging.

**What to check:**
* Query logging is enabled on all public hosted zones
* Logs are delivered to a CloudWatch log group in the central logging account
* Resolver query logging is enabled for VPCs using private hosted zones

Check which hosted zones have query logging configured:

```bash
aws route53 list-query-logging-configs \
  --query 'QueryLoggingConfigs[*].{Id:Id,HostedZoneId:HostedZoneId,Destination:CloudWatchLogsLogGroupArn}'
```

Check which VPCs have Resolver query logging enabled:

```bash
aws route53resolver list-resolver-query-log-config-associations \
  --query 'ResolverQueryLogConfigAssociations[*].{VpcId:ResourceId,Status:Status,ConfigId:ResolverQueryLogConfigId}'
```

# Key Management Service (KMS)

[KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) manages encryption keys used across AWS services. Most of the encryption checked in earlier sections such as: EBS volumes, S3 objects, RDS storage, etcd Secrets. These services ultimately depends on a KMS key. KMS is therefore a dependency for the security of the entire environment. A misconfigured key policy or a key with rotation disabled affects everything encrypted with it.

Start by listing all customer-managed keys in the account. AWS-managed keys (prefixed with `aws/`) are managed by AWS and do not appear in this list:

```bash
aws kms list-keys \
  --query 'Keys[*].KeyId'
```

## Check Key Rotation

AWS KMS supports automatic annual rotation for symmetric customer-managed keys. Rotation generates new key material while the old material is retained to decrypt data encrypted with it. Without rotation, a key that has been in use for years represents a larger blast radius if the key material is ever compromised.

**What to check:**
* Automatic key rotation is enabled on all customer-managed symmetric keys
* Keys used for high-sensitivity data (S3 customer datasets, RDS, EBS, etcd) are on the rotation list

> **CIS Reference:** CIS AWS Foundations Benchmark v3.0.0 — **3.8** (Ensure rotation for customer-created symmetric CMKs is enabled).

Write the following to a file and run it to check rotation status across all customer-managed keys:

```bash
#!/usr/bin/env bash
for key_id in $(aws kms list-keys --query 'Keys[*].KeyId' --output text); do
  status=$(aws kms get-key-rotation-status \
    --key-id "$key_id" \
    --query 'KeyRotationEnabled' \
    --output text 2>/dev/null)
  meta=$(aws kms describe-key \
    --key-id "$key_id" \
    --query 'KeyMetadata.{Alias:AliasArn,Manager:KeyManager,State:KeyState}' 2>/dev/null)
  if [ "$status" = "False" ]; then
    echo "ROTATION DISABLED: $key_id"
    echo "$meta"
  fi
done
```

## Check Key Policies

Every KMS key has a [key policy](https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html) that controls who can use and administer it. Key policies are separate from IAM policies, even if an IAM policy grants `kms:*`, the key policy must also allow access. A key policy with `"Principal": "*"` makes the key usable by anyone, which is a critical finding.

**What to check:**
* No key policy has `"Principal": "*"` without restrictive conditions
* Cross-account access in key policies is intentional and scoped to specific account IDs and roles
* Key administrators (who can manage the key) and key users (who can encrypt/decrypt with it) are separate identities
* No IAM user or role has `kms:*` on a key that encrypts production data

Retrieve the key policy for a specific key (substitute `<key-id>`):

```bash
aws kms get-key-policy \
  --key-id <key-id> \
  --policy-name default \
  --output text | jq .
```

Write the following to a file and run it to scan all key policies for wildcard principals:

```bash
#!/usr/bin/env bash
for key_id in $(aws kms list-keys --query 'Keys[*].KeyId' --output text); do
  policy=$(aws kms get-key-policy \
    --key-id "$key_id" \
    --policy-name default \
    --output text 2>/dev/null)
  if echo "$policy" | jq -e '.Statement[] | select(.Principal == "*" or .Principal.AWS == "*")' > /dev/null 2>&1; then
    echo "WILDCARD PRINCIPAL IN KEY POLICY: $key_id"
  fi
done
```

## Check Key Aliases and Inventory

Key aliases give KMS keys human-readable names and make it easier to understand what each key is used for. A key with no alias and no description is difficult to attribute to a service or team. During an audit, every key should have a known purpose.

**What to check:**
* Every key has an alias that reflects its purpose (e.g. `alias/prod-s3-customer-data`, `alias/prod-rds`)
* Keys that are in a `PendingDeletion` state are reviewed before deletion (data encrypted with them may still need to be decrypted)
* No keys exist with no alias, no description, and no recent use

List all key aliases:

```bash
aws kms list-aliases \
  --query 'Aliases[*].{AliasName:AliasName,KeyId:TargetKeyId}'
```

Describe a key to see its full metadata including state, creation date, and key usage:

```bash
aws kms describe-key \
  --key-id <key-id> \
  --query 'KeyMetadata.{KeyId:KeyId,State:KeyState,Created:CreationDate,Description:Description,Usage:KeyUsage,Manager:KeyManager}'
```

# TODO:

## AWS services this doc docesn't currently cover, but could in the future. 

* Lamnbda
* SQS
* SNS
* Dynamo

## Other TODOs:

* The first version is a single flat file. In the future, maybe break this out into child files and directories.