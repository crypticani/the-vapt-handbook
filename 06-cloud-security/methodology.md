# 06 — Cloud Security: Methodology

> Systematic approach for assessing cloud infrastructure security.

> 📋 **What You Will Do In This Section**
> - [ ] Follow a 6-phase cloud pentest methodology
> - [ ] Map cloud attack surfaces across AWS, K8s, and CI/CD
> - [ ] Document findings using the cloud security report template
> - [ ] Provide IaC remediation examples in your reports

---

## 🔴 Cloud Pentest Methodology

> 💡 **Why This Matters**
> Cloud environments are vast — hundreds of services, thousands of configurations. Without a methodology, you'll miss critical misconfigurations. This framework ensures systematic coverage across identity, storage, compute, and network layers.

```
PHASE 1: SCOPE & ACCESS (15 min)
  ├── What cloud provider(s)? (AWS, GCP, Azure, multi-cloud?)
  ├── What credentials/roles provided?
  ├── What services are in scope?
  └── Can I create resources? (avoid accidental charges)

PHASE 2: IDENTITY ENUMERATION (30 min)
  ├── Who am I? (sts get-caller-identity / whoami)
  ├── What can I do? (enumerate permissions)
  ├── What roles exist? (list-roles)
  └── What trust relationships exist? (cross-account access)

PHASE 3: RESOURCE DISCOVERY (1 hour)
  ├── Compute: EC2, Lambda, ECS, Pods
  ├── Storage: S3, EBS, RDS, Secrets Manager
  ├── Network: VPCs, Security Groups, Load Balancers
  └── Management: CloudTrail, CloudWatch, Config

PHASE 4: VULNERABILITY ASSESSMENT (2 hours)
  ├── Public S3 buckets
  ├── Overly permissive IAM
  ├── Exposed services (dashboards, APIs, databases)
  ├── Unencrypted data at rest/in transit
  ├── Missing logging
  └── Default configurations

PHASE 5: EXPLOITATION (varies)
  ├── Privilege escalation paths
  ├── Credential harvesting (metadata, secrets manager)
  ├── Data exfiltration
  ├── Lateral movement (cross-account, cross-service)
  └── Persistence establishment

PHASE 6: REPORTING
  ├── Cloud-specific severity (public S3 = Critical)
  ├── Compliance mapping (CIS Benchmarks, SOC2)
  ├── Remediation steps with IaC examples
  └── Architecture recommendations
```

---

## 🔴 Phase Execution Commands

### Phase 2: Identity Enumeration

```bash
echo "=== IDENTITY ENUMERATION ==="
echo ""
echo "--- AWS ---"
echo "  aws sts get-caller-identity"
echo "  aws iam list-users"
echo "  aws iam list-roles"
echo "  aws iam list-attached-user-policies --user-name USER"
echo ""
echo "--- Kubernetes ---"
echo "  kubectl auth can-i --list"
echo "  kubectl get clusterrolebindings"
echo "  kubectl get serviceaccounts --all-namespaces"
echo ""
echo "--- Automated ---"
echo "  python3 enumerate-iam.py --access-key KEY --secret-key SECRET"
echo "  pacu → run iam__enum_permissions"
```

### Phase 3: Resource Discovery

```bash
echo "=== RESOURCE DISCOVERY ==="
echo ""
echo "--- AWS ---"
echo "  aws s3 ls"
echo "  aws ec2 describe-instances"
echo "  aws lambda list-functions"
echo "  aws rds describe-db-instances"
echo "  aws secretsmanager list-secrets"
echo "  aws ssm describe-parameters"
echo ""
echo "--- Kubernetes ---"
echo "  kubectl get all --all-namespaces"
echo "  kubectl get secrets --all-namespaces"
echo "  kubectl get services --all-namespaces"
echo "  kubectl get ingress --all-namespaces"
```

---

## 🔴 Cloud Security Report Template

```markdown
## Cloud Security Assessment Finding

### Finding: [Title — e.g., "S3 Bucket Publicly Accessible"]

**Service**: AWS S3 / IAM / EC2 / K8s / etc.
**Severity**: Critical / High / Medium / Low
**CIS Benchmark**: [e.g., CIS AWS 2.1.5 — Ensure S3 Bucket Policy is Set]

### Description
The S3 bucket `company-database-backup` is publicly accessible
without authentication. Anyone with the bucket name can list and
download all contents, including daily database backups containing
customer PII (names, emails, phone numbers, hashed passwords).

### Evidence
```bash
$ aws s3 ls s3://company-database-backup --no-sign-request
2024-01-14 03:00:00    524288000 backup_2024-01-14.sql.gz
2024-01-13 03:00:00    523264000 backup_2024-01-13.sql.gz
```

### Impact
- 1.2 million customer records exposed (PII)
- Database credentials in backup files
- GDPR/CCPA violation → potential regulatory fines
- Reputational damage if data is leaked publicly

### Remediation

**Immediate**: Block public access
```bash
aws s3api put-public-access-block --bucket company-database-backup \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

**IaC Fix (Terraform)**:
```hcl
resource "aws_s3_bucket_public_access_block" "backup" {
  bucket = aws_s3_bucket.backup.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**Long-term**:
1. Enable S3 bucket logging
2. Implement bucket policy with explicit deny for public access
3. Enable AWS Config rule `s3-bucket-public-read-prohibited`
4. Add to CI/CD pipeline: scan IaC for public S3 buckets
```

---

## 📋 Cloud Security Audit Checklist

```
AWS:
□ sts get-caller-identity
□ IAM permissions enumerated
□ S3 bucket enumeration + public access check
□ Lambda function code reviewed
□ Secrets Manager / SSM Parameter Store checked
□ EC2 metadata accessible (IMDSv1 check)?
□ IAM privilege escalation paths tested
□ CloudTrail logging verified
□ VPC security groups reviewed
□ CIS Benchmark compliance checked

KUBERNETES:
□ API server access tested
□ Dashboard exposure checked
□ RBAC permissions audited
□ Secrets enumerated and decoded
□ Pod security standards checked
□ Network policies verified
□ Service account tokens analyzed
□ Privileged pod creation tested

CI/CD:
□ Git history scanned for secrets
□ Pipeline configuration reviewed
□ Secret management analyzed
□ Supply chain risks assessed
□ Docker images scanned for secrets
```

---

## 🧠 If You're Stuck

1. **Don't know which phase to start**: Always start with Phase 2 (Identity). You need to know WHO you are before testing anything.
2. **Overwhelmed by AWS services**: Focus on IAM, S3, EC2, and Lambda first — these cover 80% of cloud findings.
3. **No cloud environment available**: Use CloudGoat, flaws.cloud, or Kubernetes Goat for practice.
4. **Report doesn't feel impactful**: Always include business impact (GDPR fines, data breach costs, regulatory violations).
5. **Client wants IaC remediation**: Include Terraform/CloudFormation fixes — it shows domain expertise.
