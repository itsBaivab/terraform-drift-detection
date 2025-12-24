# 🔍 Terraform Drift Detection & Auto-Remediation

Automated infrastructure drift detection system using GitHub Actions, Terraform, and AWS. Continuously monitors your cloud infrastructure for configuration drift and automatically applies corrections to maintain desired state.

[![Terraform](https://img.shields.io/badge/Terraform-1.10.3-623CE4?logo=terraform)](https://www.terraform.io/)
[![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?logo=amazon-aws)](https://aws.amazon.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?logo=github-actions)](https://github.com/features/actions)

---

## 📖 Overview

This project demonstrates enterprise-grade Infrastructure as Code (IaC) practices with automated drift detection and remediation. It deploys a production-ready AWS infrastructure including VPC, Auto Scaling Groups, Application Load Balancer, and S3, while continuously monitoring for configuration drift.

### 🎯 Key Features

- ⏰ **Scheduled Drift Detection** - Runs every minute (configurable)
- 🔄 **Automatic Remediation** - Auto-applies fixes when drift is detected
- 🌍 **Multi-Environment Support** - Separate dev and prod configurations
- 📊 **Comprehensive Notifications** - GitHub Issues + Slack alerts
- 🔐 **Secure State Management** - S3 backend with native locking (Terraform 1.10+)
- 📝 **Detailed Reporting** - Workflow summaries and drift analysis

---

## 🏗️ Infrastructure Components

### AWS Resources Deployed



- **VPC** - Custom VPC with public/private subnets across 2 AZs
- **Auto Scaling Group** - Dynamically scales EC2 instances
- **Application Load Balancer** - Distributes traffic across instances
- **Security Groups** - Firewall rules for ALB and EC2
- **S3 Bucket** - Storage for application data
- **NAT Gateway** - Outbound internet access for private subnets

---

## 🚀 Quick Start

### Prerequisites

- GitHub account
- AWS account with admin access
- AWS CLI configured locally (for backend setup)
- Terraform CLI 1.10+ (optional, for local testing)

### 1. Clone Repository

```bash
git clone <repository-url>
cd terraform-drift-detection
```

### 2. Setup Backend Storage

Run the backend setup script to create S3 bucket for Terraform state:

```bash
./scripts/setup-backend.sh <your-bucket-name> us-east-1
```

**Example:**
```bash
./scripts/setup-backend.sh my-terraform-state-bucket us-east-1
```

### 3. Update Backend Configuration

Edit `backend-dev.hcl` and `backend-prod.hcl` with your bucket name:

```hcl
bucket       = "your-bucket-name"
key          = "dev/terraform.tfstate"  # or "prod/terraform.tfstate"
region       = "us-east-1"
use_lockfile = true
encrypt      = true
```

### 4. Configure GitHub Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret Name | Description | Example |
|------------|-------------|---------|
| `AWS_ACCESS_KEY_ID` | AWS access key | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key | `wJalrXUtnFEMI/K7MDENG/...` |
| `SLACK_WEBHOOK_URL` | Slack webhook (optional) | `https://hooks.slack.com/...` |

### 5. Deploy Infrastructure

#### Option A: Manual Trigger (Recommended First Time)
1. Go to **Actions** → **Terraform Drift Detection**
2. Click **Run workflow**
3. Select branch (`dev` or `main`)

#### Option B: Local Deployment
```bash
# Initialize Terraform
terraform init -backend-config="backend-dev.hcl"

# Plan infrastructure
terraform plan

# Apply configuration
terraform apply
```

### 6. Enable Drift Detection

The workflow automatically runs every minute on the default branch (`main`). For manual runs:
- Navigate to **Actions** → **Terraform Drift Detection**
- Click **Run workflow**

---

## 🔄 CI/CD Workflow Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Actions Trigger                   │
│  • Schedule: Every minute (*/1 * * * *)                     │
│  • Manual: workflow_dispatch                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────▼─────────────┐
         │  Checkout Repository      │
         │  Determine Environment    │
         │  Configure AWS Creds      │
         └─────────────┬─────────────┘
                       │
         ┌─────────────▼─────────────┐
         │  Terraform Operations     │
         │  • Setup (v1.10.3)       │
         │  • Init (with backend)   │
         │  • Plan (drift check)    │
         └─────────────┬─────────────┘
                       │
         ┌─────────────▼─────────────┐
         │   Analyze Exit Code       │
         └──┬─────────┬─────────┬────┘
            │         │         │
    ┌───────▼──┐  ┌──▼─────┐  ┌▼──────────┐
    │  Exit 0  │  │ Exit 2 │  │  Exit 1   │
    │ No Drift │  │ DRIFT! │  │  Failed   │
    └───┬──────┘  └───┬────┘  └─────┬─────┘
        │             │              │
        │      ┌──────▼────────┐     │
        │      │  Auto-Apply   │     │
        │      │   Changes     │     │
        │      └──────┬────────┘     │
        │             │              │
    ┌───▼─────────────▼──────────────▼───┐
    │         Notifications                │
    │  • GitHub Issues                    │
    │  • Slack Alerts                     │
    │  • Workflow Summary                 │
    └─────────────────────────────────────┘
```

### Workflow Behavior

| Exit Code | Status | Action |
|-----------|--------|--------|
| `0` | ✅ No Drift | Close any open drift issues |
| `2` | ⚠️ Drift Detected | Auto-apply fixes + notify |
| `1` | ❌ Plan Failed | Exit with error |

---

## 📊 Drift Detection Process

### How It Works

1. **Scheduled Check** - GitHub Actions triggers every minute
2. **Terraform Plan** - Compares actual infrastructure vs. desired state
3. **Drift Analysis** - Examines plan exit code
   - Exit code `0` = Infrastructure matches configuration
   - Exit code `2` = Drift detected
   - Exit code `1` = Error in Terraform execution
4. **Auto-Remediation** - If drift detected:
   - Automatically runs `terraform apply`
   - Creates/updates GitHub issue with details
   - Sends Slack notification
5. **Verification** - Confirms fix applied successfully
6. **Cleanup** - Closes resolved drift issues

### Example Drift Scenarios

**Scenario 1: Manual EC2 Modification**
```
Someone manually changes an EC2 instance tag in AWS Console
→ Drift detected on next check
→ Terraform automatically restores original tag
→ Issue created and closed automatically
```

**Scenario 2: Security Group Rule Change**
```
Security group rule manually deleted
→ Drift detected
→ Auto-remediation recreates the rule
→ Slack notification sent
```

---

## 📁 Project Structure

```
terraform-drift-detection/
├── .github/
│   └── workflows/
│       └── drift_detection.yml      # GitHub Actions workflow
├── scripts/
│   ├── setup-backend.sh            # S3 backend setup script
│   └── user_data.sh                # EC2 initialization script
├── main.tf                         # Provider configuration
├── vpc.tf                          # VPC, subnets, IGW, NAT
├── security_groups.tf              # Security group rules
├── alb.tf                          # Application Load Balancer
├── asg.tf                          # Auto Scaling Group + Launch Template
├── s3.tf                           # S3 bucket
├── variables.tf                    # Input variables
├── outputs.tf                      # Output values
├── backend.tf                      # Backend configuration
├── backend-dev.hcl                 # Dev environment backend
├── backend-prod.hcl                # Prod environment backend
├── DEMO_GUIDE.md                   # Detailed demo walkthrough
└── README.md                       # This file
```

---

## 🔧 Configuration

### Adjust Drift Detection Frequency

Edit [.github/workflows/drift_detection.yml](.github/workflows/drift_detection.yml):

```yaml
on:
  schedule:
    - cron: "*/5 * * * *"  # Every 5 minutes
    # - cron: "*/15 * * * *"  # Every 15 minutes
    # - cron: "0 * * * *"     # Every hour
    # - cron: "0 */6 * * *"   # Every 6 hours
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `AWS_REGION` | AWS region for deployment | `us-east-1` |
| `SLACK_WEBHOOK_URL` | Slack notifications webhook | (optional) |

### Terraform Variables

Create `terraform.tfvars`:

```hcl
environment     = "dev"
region          = "us-east-1"
vpc_cidr        = "10.0.0.0/16"
instance_type   = "t3.micro"
desired_capacity = 2
min_size        = 1
max_size        = 4
```

---

## 📢 Notifications

### GitHub Issues

Drift detection automatically creates/updates GitHub issues:
- **Title**: 🚨 Terraform Drift Detected [env]
- **Labels**: `drift-detection`, `auto-fix`, `dev`/`prod`
- **Content**: Full Terraform plan output
- **Resolution**: Auto-closed when drift resolved

### Slack Integration

Configure Slack webhook for real-time alerts:

1. Create Slack webhook: https://api.slack.com/messaging/webhooks
2. Add to GitHub Secrets as `SLACK_WEBHOOK_URL`
3. Receive notifications for:
   - ✅ Drift detected & auto-fixed
   - ❌ Auto-fix failed (manual intervention needed)

---

## 🧪 Testing Drift Detection

### Test 1: Manual Resource Modification

```bash
# SSH into an EC2 instance and modify a tag
aws ec2 create-tags \
  --resources i-1234567890abcdef0 \
  --tags Key=ManualChange,Value=Test

# Wait for next scheduled run (< 1 minute)
# Check GitHub Actions for drift detection
```

### Test 2: Security Group Rule Change

```bash
# Remove a security group rule
aws ec2 revoke-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Workflow will detect and restore the rule
```

### Test 3: Manual Workflow Trigger

1. Go to **Actions** → **Terraform Drift Detection**
2. Click **Run workflow**
3. Observe the execution in real-time

---

## 🔒 Security Best Practices

- ✅ Use IAM roles with least privilege
- ✅ Store state files in encrypted S3 buckets
- ✅ Enable state locking to prevent conflicts
- ✅ Never commit AWS credentials to Git
- ✅ Use GitHub Secrets for sensitive data
- ✅ Enable MFA on AWS root account
- ✅ Regularly rotate access keys
- ✅ Review CloudTrail logs for unauthorized changes

---

## 🐛 Troubleshooting

### Workflow Not Running

**Problem**: Scheduled workflow doesn't trigger

**Solutions**:
1. Check if workflow is disabled in Actions tab
2. Verify `main` is the default branch
3. Look for "This scheduled workflow is disabled" banner
4. Click "Enable workflow" if present
5. Check GitHub Actions service status

### Apply Failures

**Problem**: Auto-apply fails during remediation

**Solutions**:
1. Check AWS credentials are valid
2. Verify IAM permissions are sufficient
3. Review Terraform state lock status
4. Check AWS service limits/quotas
5. Examine workflow logs for specific errors

### State Lock Issues

**Problem**: "Error acquiring state lock"

**Solutions**:
```bash
# Force unlock (use with caution)
terraform force-unlock <LOCK_ID>

# Or wait for lock to expire (S3 native locking)
```

### Backend Configuration Errors

**Problem**: Backend initialization fails

**Solutions**:
1. Verify S3 bucket exists
2. Check bucket name in `.hcl` files
3. Ensure bucket region matches configuration
4. Confirm AWS credentials have S3 access

---

## 📚 Additional Resources

- 📖 [Detailed Demo Guide](DEMO_GUIDE.md)
- 🔗 [Terraform Documentation](https://www.terraform.io/docs)
- 🔗 [GitHub Actions Documentation](https://docs.github.com/en/actions)
- 🔗 [AWS Best Practices](https://aws.amazon.com/architecture/well-architected/)
- 🔗 [Terraform Best Practices](https://www.terraform-best-practices.com/)

---

## 🧹 Cleanup

### Destroy Infrastructure

```bash
# Using Terraform CLI
terraform destroy -auto-approve

# Or delete via GitHub Actions
# (Create a destroy workflow if needed)
```

### Remove Backend Resources

```bash
# Empty and delete S3 bucket
aws s3 rm s3://your-bucket-name --recursive
aws s3api delete-bucket --bucket your-bucket-name
```

---

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## 👤 Author

**Baivab**

- 🐙 GitHub: [@itsBaivab](https://github.com/itsBaivab)

---

## ⭐ Show Your Support

Give a ⭐️ if this project helped you!

---

## 📞 Support

For issues and questions:
- Open a [GitHub Issue](../../issues)
- Check the [Demo Guide](DEMO_GUIDE.md)
- Review [Troubleshooting](#troubleshooting) section

---

**Made with ❤️ for the DevOps Community**
