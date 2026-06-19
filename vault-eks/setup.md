# HashiCorp Vault on AWS EKS

## Overview

This guide deploys:

- Amazon EKS
- Spot Worker Nodes
- AWS EBS CSI Driver
- HashiCorp Vault HA (3 Nodes)
- Integrated Raft Storage
- AWS KMS Auto Unseal
- AWS IAM OIDC Integration (IRSA)
- AWS Dynamic Secrets Engine

This setup is intended for learning and lab environments with minimal cost.

---

# Architecture

EC2 Jump Server
        |
        v
Amazon EKS
        |
        +----------------------+
        |                      |
        v                      v
Vault Pod 0              Vault Pod 1
        |
        v
Vault Pod 2

Storage:
Amazon EBS Volumes

Auto Unseal:
AWS KMS

Authentication:
IAM OIDC Provider

IAM Access:
IRSA

---

# Cost Optimization

This lab uses:

- Spot Instances
- t3.small / t3a.small
- 20GB node disks
- gp3 volumes
- Single Jump Server

Always complete cleanup steps before exiting.

---

# Prerequisites

## IAM Role for Jump Server

Create:

EKSAdminRole

Attach:

- AdministratorAccess

Launch EC2 with this role attached.

Verify:

```bash
aws sts get-caller-identity
```

Expected:

```json
{
  "Arn":"arn:aws:sts::<account-id>:assumed-role/EKSAdminRole/i-xxxx"
}
```

---

# EC2 Jump Server Setup

Update:

```bash
sudo dnf update -y
```

Install utilities:

```bash
sudo dnf install -y \
git \
curl \
wget \
jq \
unzip \
tar
```

Install kubectl:

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/latest/bin/linux/amd64/kubectl

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

Install eksctl:

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO \
https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz

tar -xzf eksctl_${PLATFORM}.tar.gz

sudo mv eksctl /usr/local/bin
```

Verify:

```bash
eksctl version
```

Install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

# Create KMS Key

Create:

```bash
aws kms create-key \
--description "Vault Auto Unseal"
```

Copy:

```text
KeyId
```

Create Alias:

```bash
aws kms create-alias \
--alias-name alias/vault-auto-unseal \
--target-key-id <KEY_ID>
```

Verify:

```bash
aws kms list-aliases
```

---

# Create Vault IAM Role (IRSA)

## Get OIDC Provider

```bash
aws eks describe-cluster \
--name vault-eks \
--query cluster.identity.oidc.issuer
```

Example:

```text
https://oidc.eks.ap-south-1.amazonaws.com/id/DEB052B9246A4C4A11572EF33B7FBAA5
```

OIDC ID:

```text
DEB052B9246A4C4A11572EF33B7FBAA5
```

---

## Trust Policy

Create trust relationship policy, by copying [trust-policy.json](https://github.com/SubbuDevasani/devops_training/blob/master/vault-eks/trust-policy.json) file:

```bash
vi trust-policy.json
```

Create Role:

```bash
aws iam create-role \
--role-name VaultKMSRole \
--assume-role-policy-document file://trust-policy.json
```

---

## IAM Permissions

Create IAM policy, by copying [kms-policy.json](https://github.com/SubbuDevasani/devops_training/blob/master/vault-eks/trust-policy.json) file:

```bash
vi kms-policy.json
```

Attach:

```bash
aws iam put-role-policy \
--role-name VaultKMSRole \
--policy-name VaultKMSPolicy \
--policy-document file://kms-policy.json
```

---

# Create EKS Cluster

Create EKS Cluster, by copying [cluster.yaml](https://github.com/SubbuDevasani/devops_training/blob/master/vault-eks/cluster.yaml) file:

```bash
vi cluster.yaml
eksctl create cluster -f cluster.yaml
```

---

# Verify OIDC

```bash
aws iam list-open-id-connect-providers
```

---

# Create Namespace

```bash
kubectl create namespace vault
```

---

# Vault values.yaml

Create values.yaml, by copying [vault-values.yml](https://github.com/SubbuDevasani/devops_training/blob/master/vault-eks/vault-values.yml) file:

```bash
vi vault-values.yml
```

---

# Install Vault

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com

helm repo update
```

Install:

```bash
helm install vault-helm hashicorp/vault \
-n vault \
-f vault-values.yaml
```

---

# Annotate Service Account

```bash
kubectl annotate sa sa-vault-demo \
-n vault \
eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/VaultKMSRole
```

Verify:

```bash
kubectl get sa sa-vault-demo -n vault -o yaml
```

---

# Initialize Vault

```bash
kubectl exec -it vault-helm-0 -n vault -- vault operator init
```

Save:

- Unseal Keys
- Root Token

---

# Join Remaining Nodes

```bash
kubectl exec -it vault-helm-1 -n vault -- \
vault operator raft join \
http://vault-helm-0.vault-helm-internal:8200
```

```bash
kubectl exec -it vault-helm-2 -n vault -- \
vault operator raft join \
http://vault-helm-0.vault-helm-internal:8200
```

---

# Verify Cluster

```bash
vault operator raft list-peers
```

Expected:

```text
vault-helm-0
vault-helm-1
vault-helm-2
```

---

# Configure AWS Secrets Engine

Enable:

```bash
vault secrets enable aws
```

Configure:

```bash
vault write aws/config/root \
access_key=<KEY> \
secret_key=<SECRET> \
region=ap-south-1
```

Create Role:

```bash
vault write aws/roles/ec2-role \
credential_type=iam_user \
policy_document=@aws-policy.json
```

Generate Credentials:

```bash
vault read aws/creds/ec2-role
```

---

# Troubleshooting

## KMS Access Denied

Error:

```text
kms:DescribeKey AccessDeniedException
```

Verify:

```bash
kubectl get sa sa-vault-demo -n vault -o yaml
```

Verify annotation exists.

Verify role:

```bash
aws iam get-role \
--role-name VaultKMSRole
```

Verify pod receives AWS variables:

```bash
kubectl exec -it vault-helm-0 -n vault -- env | grep AWS
```

---

# Cleanup

## Remove Vault

```bash
helm uninstall vault-helm -n vault
```

Delete Namespace:

```bash
kubectl delete namespace vault
```

---

## Delete Cluster

```bash
eksctl delete cluster \
--name vault-eks \
--region ap-south-1
```

---

## Delete KMS Alias

```bash
aws kms delete-alias \
--alias-name alias/vault-auto-unseal
```

---

## Schedule KMS Key Deletion

```bash
aws kms schedule-key-deletion \
--key-id <KEY_ID> \
--pending-window-in-days 7
```

---

## Delete IAM Role

```bash
aws iam delete-role-policy \
--role-name VaultKMSRole \
--policy-name VaultKMSPolicy
```

```bash
aws iam delete-role \
--role-name VaultKMSRole
```

---

# Cleanup Validation and Forced Resource Removal

## 1. EKS Clusters

Verify:

```bash
aws eks list-clusters
```

Expected:

```json
{
  "clusters": []
}
```

If any cluster exists:

```bash
eksctl delete cluster \
--name <CLUSTER_NAME> \
--region ap-south-1
```

Verify again:

```bash
aws eks list-clusters
```

---

## 2. Load Balancers

Verify:

```bash
aws elbv2 describe-load-balancers
```

Expected:

```json
{
  "LoadBalancers": []
}
```

If any Load Balancer exists:

List:

```bash
aws elbv2 describe-load-balancers \
--query "LoadBalancers[*].[LoadBalancerArn,LoadBalancerName]" \
--output table
```

Delete:

```bash
aws elbv2 delete-load-balancer \
--load-balancer-arn <LB_ARN>
```

Verify:

```bash
aws elbv2 describe-load-balancers
```

---

## 3. EBS Volumes

Verify:

```bash
aws ec2 describe-volumes \
--filters Name=status,Values=available \
--query "Volumes[*].[VolumeId,Size]" \
--output table
```

Example:

```text
vol-123456
vol-789012
```

Delete:

```bash
aws ec2 delete-volume \
--volume-id vol-123456
```

```bash
aws ec2 delete-volume \
--volume-id vol-789012
```

Verify:

```bash
aws ec2 describe-volumes \
--filters Name=status,Values=available
```

Expected:

```json
{
  "Volumes": []
}
```

---

## 4. Running EC2 Instances

Verify:

```bash
aws ec2 describe-instances \
--filters Name=instance-state-name,Values=running \
--query 'Reservations[*].Instances[*].[InstanceId,InstanceType]' \
--output table
```

If any worker nodes remain:

Terminate:

```bash
aws ec2 terminate-instances \
--instance-ids <INSTANCE_ID>
```

Example:

```bash
aws ec2 terminate-instances \
--instance-ids i-1234567890abcdef
```

Verify:

```bash
aws ec2 describe-instances \
--filters Name=instance-state-name,Values=running
```

Only your jump server should remain.

---

## 5. NAT Gateways

Verify:

```bash
aws ec2 describe-nat-gateways
```

Expected:

```json
{
  "NatGateways": []
}
```

If any NAT Gateway exists:

List:

```bash
aws ec2 describe-nat-gateways \
--query "NatGateways[*].[NatGatewayId,State]" \
--output table
```

Delete:

```bash
aws ec2 delete-nat-gateway \
--nat-gateway-id <NAT_GATEWAY_ID>
```

Verify:

```bash
aws ec2 describe-nat-gateways
```

---

## 6. Elastic IPs

Verify:

```bash
aws ec2 describe-addresses
```

Expected:

```json
{
  "Addresses": []
}
```

If any Elastic IP exists:

Release:

```bash
aws ec2 release-address \
--allocation-id <ALLOCATION_ID>
```

Verify:

```bash
aws ec2 describe-addresses
```

---

## 7. IAM OIDC Providers

Verify:

```bash
aws iam list-open-id-connect-providers
```

Expected:

```json
{
  "OpenIDConnectProviderList": []
}
```

If any provider exists:

```bash
aws iam delete-open-id-connect-provider \
--open-id-connect-provider-arn <OIDC_ARN>
```

Verify:

```bash
aws iam list-open-id-connect-providers
```

---

## 8. Vault IAM Role

Verify:

```bash
aws iam get-role \
--role-name VaultKMSRole
```

Delete Inline Policy:

```bash
aws iam delete-role-policy \
--role-name VaultKMSRole \
--policy-name VaultKMSPolicy
```

Delete Role:

```bash
aws iam delete-role \
--role-name VaultKMSRole
```

Verify:

```bash
aws iam get-role \
--role-name VaultKMSRole
```

Expected:

```text
NoSuchEntity
```

---

## 9. KMS Alias

Verify:

```bash
aws kms list-aliases
```

Delete Alias:

```bash
aws kms delete-alias \
--alias-name alias/vault-auto-unseal
```

Verify:

```bash
aws kms list-aliases
```

---

## 10. KMS Key

Verify:

```bash
aws kms list-keys
```

List Key Details:

```bash
aws kms describe-key \
--key-id <KEY_ID>
```

Schedule Deletion:

```bash
aws kms schedule-key-deletion \
--key-id <KEY_ID> \
--pending-window-in-days 7
```

Verify:

```bash
aws kms describe-key \
--key-id <KEY_ID>
```

Expected:

```text
PendingDeletion
```

---

## 11. CloudFormation Stacks

Verify:

```bash
aws cloudformation list-stacks
```

If any stack is not DELETE_COMPLETE:

Delete:

```bash
aws cloudformation delete-stack \
--stack-name <STACK_NAME>
```

Wait:

```bash
aws cloudformation wait stack-delete-complete \
--stack-name <STACK_NAME>
```

Verify:

```bash
aws cloudformation list-stacks
```

---

## 12. VPCs Created By EKS

Verify:

```bash
aws ec2 describe-vpcs
```

Look for:

```text
eksctl-vault-eks-cluster/VPC
```

Delete only if all dependent resources are removed.

Verify subnets:

```bash
aws ec2 describe-subnets
```

Delete:

```bash
aws ec2 delete-subnet \
--subnet-id <SUBNET_ID>
```

Delete VPC:

```bash
aws ec2 delete-vpc \
--vpc-id <VPC_ID>
```

---

## 13. Final Cost Verification

Run:

```bash
aws eks list-clusters

aws elbv2 describe-load-balancers

aws ec2 describe-nat-gateways

aws ec2 describe-addresses

aws iam list-open-id-connect-providers

aws kms list-keys

aws ec2 describe-volumes \
--filters Name=status,Values=available
```

Expected:

* No EKS Clusters
* No Load Balancers
* No NAT Gateways
* No Elastic IPs
* No OIDC Providers
* No KMS Keys
* No EBS Volumes

Only remaining resource:

* Jump Server EC2 (if not terminated yet)

Terminate the jump server when finished.

One additional recommendation for a zero-cost lab checklist:

```bash
aws ce get-cost-and-usage --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) --granularity MONTHLY --metrics UnblendedCost
```
