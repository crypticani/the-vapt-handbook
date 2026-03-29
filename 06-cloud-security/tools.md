# 06 — Cloud Security: Tools & Real Usage

---

## 🔴 AWS Tools

### AWS CLI (Enumeration)
```bash
# Configure with stolen/found credentials
aws configure
# Or export directly:
export AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
export AWS_DEFAULT_REGION=us-east-1

# Identity
aws sts get-caller-identity

# IAM Enumeration
aws iam list-users
aws iam list-roles
aws iam list-groups
aws iam list-policies --only-attached
aws iam get-user-policy --user-name USER --policy-name POLICY
aws iam list-attached-user-policies --user-name USER
aws iam get-policy-version --policy-arn ARN --version-id v1

# S3
aws s3 ls
aws s3 ls s3://bucket-name --recursive
aws s3 cp s3://bucket/secret.txt ./

# EC2
aws ec2 describe-instances | jq '.Reservations[].Instances[] | {id: .InstanceId, ip: .PublicIpAddress, state: .State.Name, role: .IamInstanceProfile.Arn}'

# Lambda
aws lambda list-functions
aws lambda get-function --function-name FUNC_NAME  # Gets code download URL!

# Secrets Manager
aws secretsmanager list-secrets
aws secretsmanager get-secret-value --secret-id SECRET_NAME

# SSM Parameter Store
aws ssm describe-parameters
aws ssm get-parameter --name "/prod/db/password" --with-decryption

# RDS
aws rds describe-db-instances | jq '.DBInstances[] | {id: .DBInstanceIdentifier, engine: .Engine, endpoint: .Endpoint.Address}'
```

### Pacu (AWS Exploitation Framework)
```bash
# Install
pip3 install pacu

# Run
pacu

# Inside pacu:
import_keys --all  # Import AWS keys from environment
run iam__enum_permissions  # What can I do?
run iam__privesc_scan     # Privilege escalation paths
run s3__bucket_finder     # Find S3 buckets
run ec2__enum             # Enumerate EC2 instances
run lambda__enum          # Enumerate Lambda functions
run iam__backdoor_users_keys  # Create backdoor access keys
```

### ScoutSuite (Cloud Auditing)
```bash
# Install
pip3 install scoutsuite

# AWS audit
scout aws --profile default

# Output: HTML report with all misconfigurations
# Open: scout-report/report.html
```

### Prowler (AWS Security Assessment)
```bash
# Install  
pip3 install prowler

# Run full assessment
prowler aws

# Specific checks
prowler aws --severity critical high
prowler aws --service iam s3 ec2

# Output formats
prowler aws -M csv json html
```

### enumerate-iam
```bash
# Quick permission enumeration
pip3 install enumerate-iam
python3 enumerate-iam.py --access-key AKIA... --secret-key xxxx...
# Brute-forces API calls to determine what you can access
```

---

## 🔴 Kubernetes Tools

### kubectl (Enumeration & Attack)
```bash
# Cluster info
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces

# Enumerate everything
kubectl get all --all-namespaces
kubectl get pods --all-namespaces -o wide
kubectl get services --all-namespaces
kubectl get ingress --all-namespaces

# Check permissions
kubectl auth can-i --list
kubectl auth can-i create pods
kubectl auth can-i get secrets

# Read secrets
kubectl get secrets --all-namespaces
kubectl get secret SECRET -n NAMESPACE -o yaml
kubectl get secret SECRET -n NAMESPACE -o jsonpath='{.data}' | jq -r 'to_entries[] | .key + ": " + (.value | @base64d)'

# Execute into pods
kubectl exec -it POD_NAME -n NAMESPACE -- /bin/bash

# Get pod environment variables (might contain secrets)
kubectl exec POD_NAME -- env
```

### kube-hunter (Kubernetes Pentest)
```bash
# Install
pip3 install kube-hunter

# Remote scan
kube-hunter --remote TARGET_IP

# Internal scan (from within cluster)
kube-hunter --pod

# Active scanning (attempts exploitation)
kube-hunter --active
```

### kubeaudit
```bash
# Install
go install github.com/Shopify/kubeaudit@latest

# Audit all
kubeaudit all

# Specific checks
kubeaudit privileged   # Check for privileged containers
kubeaudit capabilities # Check for dangerous capabilities
kubeaudit rootfs       # Check for read-only root filesystem
kubeaudit nonroot      # Check if containers run as root
```

### peirates (Kubernetes Attack Tool)
```bash
# Install and run inside a pod
# https://github.com/inguardians/peirates
./peirates
# Automatically enumerates service accounts, secrets, and privilege escalation paths
```

---

## 🔴 CI/CD Tools

### trufflehog (Secret Scanner)
```bash
# Install
pip3 install trufflehog

# Scan Git repo
trufflehog git https://github.com/target-org/repo.git

# Scan GitHub org
trufflehog github --org target-org

# Scan filesystem
trufflehog filesystem /path/to/code

# Scan Docker image
trufflehog docker --image target/app:latest

# Only verified secrets (confirmed working)
trufflehog git https://github.com/target-org/repo.git --only-verified
```

### gitleaks
```bash
# Install
go install github.com/gitleaks/gitleaks/v8@latest

# Scan repo
gitleaks detect --source=/path/to/repo

# Scan specific commits
gitleaks detect --source=/path/to/repo --log-opts="--all"

# Generate report
gitleaks detect --source=/path/to/repo -f json -r report.json
```

### GitDorker
```bash
# GitHub dorking automation
# https://github.com/obheda12/GitDorker
python3 GitDorker.py -t YOUR_GITHUB_TOKEN -org target-org -d dorks/alldorks.txt
```

---

## 🔴 General Cloud Tools

### CloudFox (AWS/Azure/GCP Enumeration)
```bash
# Install
go install github.com/BishopFox/cloudfox@latest

# AWS enumeration
cloudfox aws --profile default all-checks

# Specific checks
cloudfox aws --profile default permissions
cloudfox aws --profile default instances
cloudfox aws --profile default s3
cloudfox aws --profile default env-vars
```

### Steampipe (Cloud SQL Queries)
```bash
# Install
brew install steampipe  # or download from steampipe.io

# Install AWS plugin
steampipe plugin install aws

# Query cloud resources with SQL
steampipe query "SELECT name, attached_policies FROM aws_iam_user"
steampipe query "SELECT name, acl FROM aws_s3_bucket WHERE bucket_policy_is_public"
steampipe query "SELECT instance_id, public_ip_address, iam_instance_profile_arn FROM aws_ec2_instance"
```

---

## 📋 Tool Selection Matrix

| Task | Tool | Complexity |
|------|------|------------|
| AWS credential enumeration | enumerate-iam, aws cli | Easy |
| AWS exploitation | Pacu | Medium |
| AWS audit | ScoutSuite, Prowler | Easy |
| Secret scanning | trufflehog, gitleaks | Easy |
| K8s pentest | kubectl, kube-hunter | Medium |
| K8s audit | kubeaudit | Easy |
| S3 bucket testing | aws s3 ls --no-sign-request | Easy |
| Cloud recon | CloudFox | Medium |
| CI/CD analysis | trufflehog, GitDorker | Easy |
