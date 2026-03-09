# AWS Audit Commands 

Runbook of auditing an AWS environment and the commands to run.

# Organization

An [Organization](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html) is the largest object in AWS. It contain one or more AWS Accounts. Normally, [Service Control Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) would be applied at this level and trickle down into the containing accounts. Examples of SCPs could be:

* Restrict IMDSv1 Usage
* Restrict the Usage of the Root Account
* Restrict Usage of `AdministratorAccess` IAM Role
* Force MFA usage

# Account

The AWS [Account] is contained within an Organization. It acts as a resource and security boundary for AWS services. Below are the components that should be audited.

## Ensure Cloudtrail Logging Is Set Up 

## Enforce MFA on Users

## Restrict the Usage of Regions

AWS has all regions enabled by default. The problem with this is that most organiations only use a single or a small number of regions and monitor those. However, an malicious actor could deploy resources to a previously unused region and go unnoticed. For this, set a policy to restrict non-approved regions.

# VPC
This section holds the plan for auditing VPC and the commands to do so. The VPC is the heart of it all. Here are the things to look for at the VPC level.

# EC2

This section holds the plan for auditing EC2 server instances and the commands to do so.

## Check for IMDSv2 Optional or Required

The AWS [IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) service provides metadata to instances. There is an SSRF vulnerability in [v1}(https://aws.amazon.com/blogs/security/how-to-use-policies-to-restrict-where-ec2-instance-credentials-can-be-used-from/) and it's best practice to use v2.  

## Check If a Instance Has a Public 

Check if an instnace has a public IP and what services are running on that instnace. If an instnace has public IP, then first ask why and if it needs one. Then look to see what applications are running on it. 

## Check the Security Groups for Instances

Check the security groups for an instance to see what traffic is allowed to reach the host.

## Check to See if Cloud Trail Logging Is Configured

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

## Nodes

This section contains commands and things to check for the Kubernetes Nodes.

### List out the Nodes and Versions

```bash
kubectl get nodes
```

### 
## Deployments, ReplicaSets, StatefulSets, DaemonSets, and Pods

## Services, Network Policies, and Ingress

## ConfigMaps and Secrets

## CRDs

## Roles, RoleBindings, ClusterRoles, and ClusterRoleBindings


