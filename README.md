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

## CRDs
