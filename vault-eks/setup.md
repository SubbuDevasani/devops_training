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

Create:

trust-policy.json

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":{
        "Federated":"arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/<OIDC_ID>"
      },
      "Action":"sts:AssumeRoleWithWebIdentity",
      "Condition":{
        "StringEquals":{
          "oidc.eks.ap-south-1.amazonaws.com/id/<OIDC_ID>:sub":"system:serviceaccount:vault:sa-vault-demo"
        }
      }
    }
  ]
}
```

Create Role:

```bash
aws iam create-role \
--role-name VaultKMSRole \
--assume-role-policy-document file://trust-policy.json
```

---

## IAM Permissions

Create:

kms-policy.json

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":[
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource":"*"
    }
  ]
}
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

cluster.yaml

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: vault-eks
  region: ap-south-1
  version: "1.34"

autoModeConfig:
  enabled: false

iam:
  withOIDC: true

addons:
  - name: aws-ebs-csi-driver

managedNodeGroups:
  - name: spot-workers

    spot: true

    instanceTypes:
      - t3.small
      - t3a.small

    desiredCapacity: 3
    minSize: 3
    maxSize: 4

    volumeType: gp3
    volumeSize: 20
```

Create:

```bash
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

(Place your final working values.yaml here)

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

# Cleanup Validation

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

Verify:

```bash
aws kms list-keys
```

Expected:

```json
{
  "Keys": []
}
```

Verify:

```bash
aws ec2 describe-volumes \
--filters Name=status,Values=available
```

Expected:

```text
No EBS volumes related to Vault
```

Terminate:

- Jump Server EC2
- Unused EBS Volumes

Lab cleanup complete.
