# 06 — Cloud Security: Tools & Real Usage

> 📋 **What You Will Do In This Section**
> - [ ] Use AWS CLI for cloud enumeration after credential theft
> - [ ] Run Pacu for automated AWS privilege escalation
> - [ ] Audit cloud configurations with ScoutSuite and Prowler
> - [ ] Enumerate Kubernetes clusters with kubectl and kube-hunter
> - [ ] Scan for secrets with trufflehog and gitleaks

---

## 🔴 AWS Tools

### AWS CLI (Enumeration)

> 💡 **Why This Matters**
> AWS CLI is your primary attack tool after obtaining credentials. Whether you stole keys via SSRF, found them in source code, or extracted them from a compromised instance — AWS CLI turns those credentials into actionable intelligence.

```bash
# Configure with stolen/found credentials
export AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
export AWS_DEFAULT_REGION=us-east-1

# Step 1: Who am I?
aws sts get-caller-identity

# Step 2: IAM Enumeration
aws iam list-users
aws iam list-roles
aws iam list-attached-user-policies --user-name USER
aws iam get-policy-version --policy-arn ARN --version-id v1

# Step 3: Resource Discovery
aws s3 ls
aws ec2 describe-instances | jq '.Reservations[].Instances[] | {id: .InstanceId, ip: .PublicIpAddress, role: .IamInstanceProfile.Arn}'
aws lambda list-functions
aws lambda get-function --function-name FUNC  # Gets code download URL!
aws secretsmanager list-secrets
aws secretsmanager get-secret-value --secret-id SECRET_NAME
aws ssm get-parameter --name "/prod/db/password" --with-decryption
```

#### 🧪 Try It Now — AWS CLI Quick Reference

```bash
echo "=== AWS Enumeration Workflow ==="
echo ""
echo "1. Identity:    aws sts get-caller-identity"
echo "2. Users:       aws iam list-users"
echo "3. Policies:    aws iam list-attached-user-policies --user-name USER"
echo "4. S3:          aws s3 ls"
echo "5. EC2:         aws ec2 describe-instances"
echo "6. Lambda:      aws lambda list-functions"
echo "7. Secrets:     aws secretsmanager list-secrets"
echo "8. Parameters:  aws ssm describe-parameters"
echo ""
echo "Install: pip3 install awscli"
```

> 🔧 **If Stuck**
> - `AccessDenied` errors → The credentials may have limited permissions. Try different services.
> - Wrong region → AWS resources are regional. Try `--region us-west-2`, `eu-west-1`, etc.
> - Session token missing → If credentials came from instance metadata, you also need to export `AWS_SESSION_TOKEN`

### Pacu (AWS Exploitation Framework)

> 💡 **Why This Matters**
> Pacu is Metasploit for AWS. It automates privilege escalation, persistence, and enumeration across all AWS services. When you have limited time on an engagement, Pacu checks all 21 privesc methods in minutes.

```bash
pip3 install pacu
pacu

# Inside Pacu:
import_keys --all           # Import AWS keys from environment
run iam__enum_permissions   # What can I do?
run iam__privesc_scan       # Check all 21 privilege escalation paths
run s3__bucket_finder       # Find S3 buckets
run ec2__enum               # Enumerate EC2 instances
run lambda__enum            # Enumerate Lambda functions
```

### ScoutSuite & Prowler (Cloud Auditing)

```bash
# ScoutSuite — Visual cloud audit report
pip3 install scoutsuite
scout aws --profile default
# Output: HTML report → Open scout-report/report.html

# Prowler — CIS Benchmark assessment
pip3 install prowler
prowler aws --severity critical high
prowler aws --service iam s3 ec2
prowler aws -M csv json html  # Multiple output formats
```

### enumerate-iam

```bash
# Quick brute-force permission enumeration
pip3 install enumerate-iam
python3 enumerate-iam.py --access-key AKIA... --secret-key xxxx...
# Tries every API call to determine what you can access
```

---

## 🔴 Kubernetes Tools

### kubectl (Enumeration & Attack)

> 💡 **Why This Matters**
> kubectl is both the admin tool and the attack tool for Kubernetes. Every command used for cluster management can be used for exploitation. If you find a kubeconfig file or service account token, kubectl gives you direct access to the cluster.

```bash
# Cluster overview
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
kubectl get all --all-namespaces

# Check permissions
kubectl auth can-i --list
kubectl auth can-i create pods
kubectl auth can-i get secrets

# Read secrets (decode from base64)
kubectl get secrets --all-namespaces
kubectl get secret SECRET -n NAMESPACE -o jsonpath='{.data}' | jq -r 'to_entries[] | .key + ": " + (.value | @base64d)'

# Execute into pods
kubectl exec -it POD_NAME -- /bin/bash
kubectl exec POD_NAME -- env  # Get environment variables (secrets!)
```

### kube-hunter (Kubernetes Pentest)

```bash
pip3 install kube-hunter

kube-hunter --remote TARGET_IP        # Remote scan
kube-hunter --pod                      # Internal scan (from inside cluster)
kube-hunter --active                   # Active scanning (attempts exploitation)
```

### kubeaudit

```bash
go install github.com/Shopify/kubeaudit@latest

kubeaudit all                  # Full audit
kubeaudit privileged           # Check for privileged containers
kubeaudit capabilities         # Check dangerous capabilities
kubeaudit nonroot              # Check if containers run as root
```

---

## 🔴 CI/CD Tools

### trufflehog (Secret Scanner)

> 💡 **Why This Matters**
> trufflehog scans Git history, GitHub orgs, Docker images, and filesystems for leaked secrets. Unlike grep, it understands credential formats and can verify if found secrets are still valid (active API keys, working passwords).

```bash
pip3 install trufflehog

# Scan Git repo
trufflehog git https://github.com/target-org/repo.git

# Scan GitHub org (all repos)
trufflehog github --org target-org

# Scan filesystem
trufflehog filesystem /path/to/code

# Scan Docker image
trufflehog docker --image target/app:latest

# Only verified (working) secrets
trufflehog git https://github.com/target-org/repo.git --only-verified
```

### gitleaks

```bash
go install github.com/gitleaks/gitleaks/v8@latest

gitleaks detect --source=/path/to/repo
gitleaks detect --source=/path/to/repo -f json -r report.json
gitleaks detect --source=/path/to/repo --log-opts="--all"  # All branches
```

### CloudFox (AWS/Azure/GCP Enumeration)

```bash
go install github.com/BishopFox/cloudfox@latest

cloudfox aws --profile default all-checks
cloudfox aws --profile default permissions
cloudfox aws --profile default instances
cloudfox aws --profile default env-vars
```

---

## 📋 Tool Selection Matrix

| Task | Tool | Complexity | Install |
|------|------|-----------|---------|
| AWS identity check | `aws sts get-caller-identity` | Easy | `pip3 install awscli` |
| AWS permission enum | enumerate-iam | Easy | `pip3 install enumerate-iam` |
| AWS exploitation | Pacu | Medium | `pip3 install pacu` |
| AWS audit (visual) | ScoutSuite | Easy | `pip3 install scoutsuite` |
| AWS CIS benchmark | Prowler | Easy | `pip3 install prowler` |
| K8s pentest | kubectl + kube-hunter | Medium | `pip3 install kube-hunter` |
| K8s audit | kubeaudit | Easy | `go install` |
| Secret scanning | trufflehog | Easy | `pip3 install trufflehog` |
| Git secret scan | gitleaks | Easy | `go install` |
| Cloud recon | CloudFox | Medium | `go install` |
| S3 bucket testing | `aws s3 ls --no-sign-request` | Easy | AWS CLI |

---

## 🧠 If You're Stuck

1. **AWS CLI not installed**: `pip3 install awscli` or download from aws.amazon.com/cli
2. **Pacu import fails**: Ensure `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are exported as environment variables
3. **kubectl not connecting**: Check `~/.kube/config` exists. For testing, use Minikube.
4. **trufflehog too many false positives**: Use `--only-verified` to filter to working credentials
5. **ScoutSuite report empty**: Ensure AWS credentials have read-only access to IAM, S3, EC2, Lambda
