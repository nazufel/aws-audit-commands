# AWS Audit Commands 

Runbook of auditing an AWS environment and the commands to run.

# EC2

This section holds the plan for auditing EC2 server instances and the commands to do so.

## Check for IMDSv2 Optional or Required

The AWS IMDS service provides metadata 

# S3

This section holds the plan for auditing S3 buckets and the commands to do so.

# IAM

This section holds the plan for auditing IAM and the commands to do so.

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


