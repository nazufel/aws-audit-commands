# AWS Audit Commands 

Runbook of auditing an AWS environment and the commands to run.

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

# EC2

This section holds the plan for auditing EC2 server instances and the commands to do so.

## Check foe IMDSv2 Optional or Required

The AWS [IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) service provides metadata to instances. There is an SSRF vulnerability in [v1}(https://aws.amazon.com/blogs/security/how-to-use-policies-to-restrict-where-ec2-instance-credentials-can-be-used-from/) and it's best practice to use v2.  

## Check If a Instance Has a Public IP 

Check if an instnace has a public IP and what services are running on that instnace. If an instnace has public IP, then first ask why and if it needs one. Then look to see what applications are running on it. 

## Check the Security Groups for Instances

Check the security groups for an instance to see what traffic is allowed to reach the host.

## Check to See if Cloud Trail Logging Is Configured

## Check the Use of NAT

## Check the Use of a WAF


## Check for Tags

Check to see if the instance has necessary tagging:
* Cost Center
* Owning/Responsible Team
* Response SLA
* Service Level
* Managed-by

## Check ELBs

Check the ELBs to see what running instances are behind them and their configurations.

## Flow Logs

Check to see if Flow Logging is enabled on the ELBs.

# S3

This section holds the plan for auditing S3 buckets and the commands to do so.

## List Out Buckets

## Inspect the Permissions of Each Bucket 

Check to see if it has uniform perissions, inheritance, or fine-grained access control. Also check to see if any buckets are public. Check to see if any buckets allow `AuthenticatedUsers` because this could be any AWS user from any account, not just the owning account.

## Check to for Tags

Check for necessary tags:
* Data Classification
* Owning/Repsonsble Team
* Cost Center
* Object TTL
* Managed-by

## Check for Configured Access Logging

Check to see if access logging is configured and there is a separate bucket for that.

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
