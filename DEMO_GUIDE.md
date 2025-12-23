# Terraform Drift Detection Demo Guide

Complete walkthrough for demonstrating automated Terraform drift detection, auto-remediation, and infrastructure management across dev and production environments.

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Overview](#project-overview)
3. [Initial Setup](#initial-setup)
4. [Deploying Infrastructure](#deploying-infrastructure)
5. [Testing Drift Detection](#testing-drift-detection)
6. [Understanding the Workflows](#understanding-the-workflows)
7. [Cleanup](#cleanup)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Accounts & Tools
- ✅ GitHub account with repository access
- ✅ AWS account with admin access
- ✅ Terraform CLI installed locally (optional, for manual testing)
- ✅ Git CLI installed
- ✅ Slack workspace (optional, for notifications)

### AWS IAM Permissions Required
Your AWS credentials need permissions for:
- EC2 (VPC, Subnets, Internet Gateway, NAT Gateway, Route Tables)
- Auto Scaling Groups
- Application Load Balancer
- S3 Buckets
- Security Groups

---

## Project Overview

### Infrastructure Components

This demo creates a complete web application infrastructure:

```
┌─────────────────────────────────────────┐
│           Application Load Balancer      │
└─────────────┬───────────────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
┌───▼────┐         ┌────▼───┐
│  EC2   │         │  EC2   │
│Instance│         │Instance│
└────────┘         └────────┘
    │                   │
    └─────────┬─────────┘
              │
     ┌────────▼────────┐
     │   VPC Network    │
     │  - Public Subnet │
     │  - Private Subnet│
     │  - NAT Gateway   │
     └──────────────────┘
```

**Resources Created:**
- VPC with public/private subnets across 2 AZs
- Internet Gateway and NAT Gateway
- Application Load Balancer
- Auto Scaling Group (min: 1, max: 3 instances)
- S3 bucket for application data
- Security groups for ALB and EC2

### Environments

| Environment | Branch | Purpose | Drift Detection |
|-------------|--------|---------|-----------------|
| **Dev** | `dev` | Development/testing | ❌ Disabled |
| **Prod** | `main` | Production | ✅ Enabled (daily + auto-fix) |

### Workflows

| Workflow | File | Trigger | Purpose |
|----------|------|---------|---------|
| **CI/CD** | `terraform.yml` | Push to main/dev | Deploy infrastructure |
| **Drift Detection** | `drift_detection.yml` | Daily schedule (prod only) | Detect & auto-fix drift |
| **Destroy** | `destroy.yml` | Manual trigger | Safely destroy infrastructure |

---

## Initial Setup

### Step 1: Fork/Clone Repository

```bash
# Clone the repository
git clone https://github.com/itsBaivab/terraform-drift-detection.git
cd terraform-drift-detection

# Create dev branch
git checkout -b dev
git push origin dev
```

### Step 2: Verify Remote State Backend

**✅ Backend Already Configured!**

The project is pre-configured to use S3 for remote state storage:
- **S3 Bucket:** `techtutorialswithpiyush-terraform-state`
- **Dev State:** `dev/terraform.tfstate`
- **Prod State:** `prod/terraform.tfstate`
- **Locking:** S3 native locking (Terraform 1.10.3)

**Backend Configuration:**

```hcl
# backend-dev.hcl
bucket       = "techtutorialswithpiyush-terraform-state"
key          = "dev/terraform.tfstate"
region       = "us-east-1"
use_lockfile = true  # S3 native locking
encrypt      = true

# backend-prod.hcl
bucket       = "techtutorialswithpiyush-terraform-state"
key          = "prod/terraform.tfstate"
region       = "us-east-1"
use_lockfile = true
encrypt      = true
```

**S3 Native State Locking:**
- ✅ Uses Terraform 1.10.3 feature `use_lockfile = true`
- ✅ Creates `.tflock` files in S3 for state locking
- ✅ No DynamoDB needed - simpler and cheaper
- ✅ Lock files automatically created/deleted during operations

**Verify S3 Bucket Access:**

```bash
# Check if you have access to the bucket
aws s3 ls s3://techtutorialswithpiyush-terraform-state/

# Should show dev/ and prod/ folders after first deployment
```

### Step 3: Configure AWS Credentials

1. **Create IAM User** (or use existing)
   - Go to AWS Console → IAM → Users → Create User
   - Attach policy: `AdministratorAccess` (or custom policy with required permissions)
   - Create Access Key → Store securely

2. **Add GitHub Secrets**
   - Go to repository → Settings → Secrets and variables → Actions → New repository secret
   
   Add these secrets:
   
   | Secret Name | Value | Required |
   |-------------|-------|----------|
   | `AWS_ACCESS_KEY_ID` | Your AWS access key | ✅ Yes |
   | `AWS_SECRET_ACCESS_KEY` | Your AWS secret key | ✅ Yes |
   | `SLACK_WEBHOOK_URL` | Slack webhook URL | ⚠️ Optional |

   **Note:** The S3 bucket name (`techtutorialswithpiyush-terraform-state`) is hardcoded in the backend configuration files and doesn't need to be a secret.

### Step 4: Configure GitHub Environments

Create environments for approval gates (optional but recommended):

1. Go to Settings → Environments → New environment
2. Create two environments:
   - **dev** - No protection rules needed
   - **prod** - Add protection rules:
     - ✅ Required reviewers: Add yourself
     - ✅ Wait timer: 0 minutes

### Step 5: Create Slack Webhook (Optional)

If you want Slack notifications:

1. Go to https://api.slack.com/messaging/webhooks
2. Create a new webhook for your workspace
3. Copy the webhook URL
4. Add as `SLACK_WEBHOOK_URL` secret in GitHub

---

## Deploying Infrastructure

### Deploy Development Environment

1. **Create/Push to dev branch:**
   ```bash
   git checkout dev
   # Make any changes if needed
   git add .
   git commit -m "Deploy dev environment"
   git push origin dev
   ```

2. **Monitor Deployment:**
   - Go to Actions tab in GitHub
   - Watch "Terraform CI/CD" workflow
   - Plan job runs first
   - Apply job deploys infrastructure

3. **Verify Deployment:**
   - Check workflow summary for outputs
   - Or manually check AWS Console:
     - EC2 → Load Balancers
     - EC2 → Auto Scaling Groups
     - VPC → Your VPCs

### Deploy Production Environment

1. **Merge dev to main:**
   ```bash
   git checkout main
   git merge dev
   git push origin main
   ```

2. **Approve Deployment** (if using environment protection):
   - Go to Actions → Workflow run
   - Review and approve the deployment

3. **Verify Production:**
   - Same verification steps as dev
   - Note: This environment will have drift detection enabled

---

## Testing Drift Detection

### Understanding Drift Detection

Drift occurs when:
- Manual changes via AWS Console
- Changes by other automation/scripts
- External modifications to infrastructure

### Test Scenario 1: Manual Tag Modification

**Simulate drift by modifying a resource tag:**

```bash
# Get the ALB name from Terraform outputs
# In GitHub Actions output, or run locally:
terraform output alb_dns_name

# Manually add a tag via AWS CLI
aws elbv2 add-tags \
  --resource-arns arn:aws:elasticloadbalancing:us-east-1:ACCOUNT_ID:loadbalancer/app/YOUR-ALB-NAME \
  --tags Key=ManualTag,Value=DriftTest
```

Or via AWS Console:
1. EC2 → Load Balancers
2. Select your ALB
3. Tags → Manage tags
4. Add tag: `ManualTag=DriftTest`

**What happens:**
1. Next day at 8 AM UTC (or trigger manually), drift detection runs
2. Detects the unexpected tag
3. Creates GitHub issue: "🚨 Terraform Drift Detected"
4. Automatically runs `terraform apply` to remove the tag
5. Sends Slack notification (if configured)
6. Closes issue once fixed

### Test Scenario 2: Manual Instance Termination

**More dramatic drift test:**

```bash
# List ASG instances
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names YOUR-ASG-NAME

# Terminate one instance (ASG will recreate it)
aws ec2 terminate-instances --instance-ids i-xxxxx
```

**What happens:**
- ASG automatically recreates the instance (within minutes)
- Drift detection may not catch this (depends on timing)
- Good test of ASG self-healing

### Test Scenario 3: Security Group Rule Change

```bash
# Add unauthorized ingress rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

**What happens:**
- Drift detection identifies unauthorized SSH rule
- Auto-remediation removes the rule
- Issue tracks the change

### Manual Drift Detection Trigger

Don't want to wait for the daily schedule?

1. Go to Actions → "Terraform Drift Detection (Production Only)"
2. Click "Run workflow" → Run on main branch
3. Watch it detect and fix drift in real-time

---

## Understanding the Workflows

### Workflow 1: Terraform CI/CD (`terraform.yml`)

**Triggers:**
- Push to `main` or `dev` branches
- Pull request to `main` or `dev`

**Jobs:**

#### Plan Job (Always runs)
- ✅ Checks out code
- ✅ Determines environment (dev or prod)
- ✅ Runs `terraform plan`
- ✅ Comments plan on PR
- ✅ Uploads plan artifact

#### Apply Job (Only on push)
- ✅ Downloads plan artifact
- ✅ Runs `terraform apply`
- ✅ Outputs infrastructure info

**Environment Variable Logic:**
```yaml
main branch → prod environment
dev branch  → dev environment
```

### Workflow 2: Drift Detection (`drift_detection.yml`)

**Triggers:**
- Daily schedule: 8 AM UTC
- Manual trigger
- Push to `main` (optional, immediate check)

**Only runs on main branch (production)!**

**Flow:**
```
1. terraform plan -detailed-exitcode
2. Exit code 2? → Drift detected
3. Create/update GitHub issue
4. Run terraform apply -auto-approve
5. Send Slack notification
6. Close issue on success
```

**Exit Codes:**
- `0` = No drift (closes any open drift issues)
- `1` = Error (fails workflow)
- `2` = Drift detected (triggers auto-fix)

### Workflow 3: Destroy (`destroy.yml`)

**Triggers:**
- Manual only (workflow_dispatch)

**Safety Features:**
- ✅ Must type "DESTROY" exactly
- ✅ Environment selection (dev or prod)
- ✅ Requires environment approval (if configured)
- ✅ Creates issue tracking destruction

**Usage:**
1. Actions → "Terraform Destroy"
2. Run workflow
3. Select environment: dev or prod
4. Type "DESTROY" in confirmation field
5. Run workflow
6. Approve (if prod environment protection enabled)

---

## Cleanup

### Important: Backend State Files

Before destroying infrastructure, note that state files in S3 will remain. You can:
- Keep them for audit/history
- Delete them after infrastructure is destroyed
- The S3 bucket and DynamoDB table are NOT managed by Terraform

### Method 1: Using Destroy Workflow (Recommended)

**Destroy Dev Environment:**
1. Actions → "Terraform Destroy"
2. Run workflow
   - Environment: `dev`
   - Confirmation: `DESTROY`
3. Wait for completion

**Destroy Prod Environment:**
1. Actions → "Terraform Destroy"
2. Run workflow
   - Environment: `prod`
   - Confirmation: `DESTROY`
3. Approve the deployment
4. Wait for completion

### Method 2: Manual Terraform Destroy

If workflows fail:

```bash
# Clone repo locally
git clone https://github.com/itsBaivab/terraform-drift-detection.git
cd terraform-drift-detection

# Configure AWS credentials
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="us-east-1"

# Initialize and destroy
terraform init
terraform destroy -auto-approve
```

### Method 3: Manual AWS Cleanup

If Terraform fails, manually delete in AWS Console:

**Order matters! Delete in this order:**
1. Auto Scaling Group (wait for instances to terminate)
2. Target Groups
3. Load Balancer
4. Launch Template
5. NAT Gateway (wait ~5 min for release)
6. Elastic IPs
7. Internet Gateway (detach first)
8. Subnets
9. Route Tables
10. VPC
11. Security Groups
12. S3 Bucket (empty it first)

### Verify Cleanup

```bash
# Check for remaining resources
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=*terraform*"
aws elbv2 describe-load-balancers
aws s3 ls | grep terraform
```

### Cleanup Backend Resources (Optional)

**⚠️ Warning:** Only do this if you're completely done with the demo!

```bash
# 1. Delete state files from S3
aws s3 rm s3://techtutorialswithpiyush-terraform-state/dev/terraform.tfstate
aws s3 rm s3://techtutorialswithpiyush-terraform-state/prod/terraform.tfstate

# 2. Delete lock files (if any remain)
aws s3 rm s3://techtutorialswithpiyush-terraform-state/dev/terraform.tfstate.tflock
aws s3 rm s3://techtutorialswithpiyush-terraform-state/prod/terraform.tfstate.tflock

# 3. List remaining files
aws s3 ls s3://techtutorialswithpiyush-terraform-state/ --recursive

# 4. Only if empty and you own the bucket:
aws s3 rb s3://techtutorialswithpiyush-terraform-state --force
```

---

## Troubleshooting

### Issue: Workflow Fails with "403 Forbidden"

**Cause:** Missing GitHub permissions

**Solution:**
- Check workflow has `permissions:` block
- Verify `issues: write` permission exists
- Check GitHub Actions are enabled in repo settings

### Issue: Concurrent Terraform Runs

**S3 Native Locking (Terraform 1.10.0+):**
- Creates `.tflock` files in S3 during operations
- Prevents concurrent modifications automatically
- If locked, you'll see: "Error acquiring the state lock"

**How it works:**
```
terraform apply starts → Creates dev/terraform.tfstate.tflock
Another apply tries    → Sees lock file → Waits or fails
First apply completes  → Deletes .tflock file
```

**If lock gets stuck:**
```bash
# List lock files
aws s3 ls s3://techtutorialswithpiyush-terraform-state/ --recursive | grep tflock

# Manually remove stuck lock (only if you're sure no operation is running)
aws s3 rm s3://techtutorialswithpiyush-terraform-state/prod/terraform.tfstate.tflock
# Or for dev:
aws s3 rm s3://techtutorialswithpiyush-terraform-state/dev/terraform.tfstate.tflock
```

**State corruption recovery:**
```bash
# Restore from S3 version history
aws s3api list-object-versions \
  --bucket YOUR-BUCKET-NAME \
  --prefix prod/terraform.tfstate

# Download previous version
aws s3api get-object \
  --bucket YOUR-BUCKET-NAME \
  --key prod/terraform.tfstate \
  --version-id VERSION-ID \
  terraform.tfstate
```

### Issue: "Backend Configuration Changed"

**Cause:** Backend configuration was modified or state was moved

**Solution:**
```bash
# Reconfigure backend
terraform init -reconfigure -backend-config="backend-prod.hcl" \
  -backend-config="bucket=$TERRAFORM_STATE_BUCKET"

# Or migrate state
terraform init -migrate-state
```

### Issue: "Failed to Load State"

**Cause:** S3 bucket doesn't exist or wrong bucket name

**Check:**
```bash
# Verify bucket exists and you have access
aws s3 ls s3://techtutorialswithpiyush-terraform-state/

# Verify GitHub secret is set correctly
# Settings → Secrets → TERRAFORM_STATE_BUCKET
# Value should be: techtutorialswithpiyush-terraform-state
```

### Issue: Drift Detection Not Running

**Possible causes:**
1. Not on main branch (it only runs on prod)
2. GitHub Actions disabled
3. Schedule not reached yet

**Check:**
```bash
git branch  # Verify you're on main
```

### Issue: Auto-Apply Fails

**Common causes:**
- IAM permission insufficient
- Resource dependencies (e.g., can't delete VPC with resources)
- Rate limiting

**Solution:**
- Check workflow logs for specific error
- Verify AWS credentials have full permissions
- May need manual intervention

### Issue: S3 Bucket Not Deleting

**Cause:** Bucket not empty

**Solution:**
```bash
# Empty the bucket first
aws s3 rm s3://YOUR-BUCKET-NAME --recursive
# Then destroy
terraform destroy
```

### Issue: NAT Gateway Delete Timeout

**Cause:** NAT Gateway takes 3-5 minutes to delete

**Solution:**
- Be patient, this is normal
- Don't interrupt the destroy process
- If timeout occurs, run destroy again

### Issue: Slack Notifications Not Working

**Check:**
1. Webhook URL is correct and active
2. Secret name is exactly `SLACK_WEBHOOK_URL`
3. Webhook has permission to post to channel

---

## Cost Considerations

### Estimated Costs (us-east-1)

| Resource | Approximate Cost |
|----------|------------------|
| NAT Gateway | ~$0.045/hour (~$32/month) |
| ALB | ~$0.025/hour (~$18/month) |
| EC2 t2.micro (1-3) | ~$0.0116/hour each (~$8.5/month each) |
| Data Transfer | Varies |
| S3 | Minimal (< $1/month) |

**Total: ~$50-70/month if left running**

**Cost Saving Tips:**
- ✅ Destroy dev when not in use
- ✅ Use scheduled shutdown for dev instances
- ✅ Consider using AWS free tier (if eligible)
- ✅ Monitor with AWS Cost Explorer

---

## Demo Script

### Quick 10-Minute Demo

**Preparation (5 min):**
1. Deploy prod environment (let it run)
2. Open GitHub Actions, AWS Console, Slack

**Demo Flow (10 min):**

**Minute 1-2: Overview**
- "This is automated drift detection with auto-remediation"
- Show architecture diagram
- Explain dev/prod split

**Minute 3-4: Show Deployed Infrastructure**
- AWS Console → Show VPC, ALB, ASG
- Show healthy instances
- Copy ALB DNS, show in browser (if app deployed)

**Minute 5-6: Introduce Drift**
- AWS Console → ALB → Tags
- Add `ManualTag=DriftDemo`
- "This simulates unauthorized manual change"

**Minute 7-8: Trigger Drift Detection**
- GitHub Actions → Run drift detection manually
- Watch it detect drift
- Show issue being created

**Minute 9: Auto-Remediation**
- Watch terraform apply run
- Show Slack notification
- Refresh AWS Console → Tag is gone

**Minute 10: Wrap Up**
- Show issue closed
- Explain daily automated schedule
- Show destroy workflow for cleanup

---

## Advanced Topics

### Remote State Management

**Separate State Files:**
- Dev: `s3://techtutorialswithpiyush-terraform-state/dev/terraform.tfstate`
- Prod: `s3://techtutorialswithpiyush-terraform-state/prod/terraform.tfstate`

**S3 Native State Locking (Terraform 1.10.0+):**
- ✅ Uses S3 conditional writes for locking
- ✅ Creates `.tflock` files during operations
- ✅ Prevents concurrent modifications automatically
- ✅ No DynamoDB needed - simpler and cheaper
- ✅ Lock files: `dev/terraform.tfstate.tflock` and `prod/terraform.tfstate.tflock`

**How it works:**
1. `terraform apply` starts → Creates lock file in S3
2. Concurrent `terraform apply` → Detects lock file → Fails/waits
3. First operation completes → Deletes lock file

**State Versioning:**
- S3 versioning enabled automatically
- Can rollback to previous state if needed

```bash
# List state versions
aws s3api list-object-versions \
  --bucket techtutorialswithpiyush-terraform-state \
  --prefix prod/terraform.tfstate

# Restore previous version
aws s3api get-object \
  --bucket techtutorialswithpiyush-terraform-state \
  --key prod/terraform.tfstate \
  --version-id VERSION-ID \
  terraform.tfstate.backup
```

### Migrate Existing State to Remote Backend

If you have local state files:

```bash
# 1. Add backend configuration to your code
# (already done in backend.tf)

# 2. Initialize with migration
terraform init -migrate-state \
  -backend-config="backend-prod.hcl" \
  -backend-config="bucket=$TERRAFORM_STATE_BUCKET"

# 3. Confirm migration when prompted
```

### Multi-Region Deployment

Modify `variables.tf`:
```hcl
variable "aws_region" {
  default = "us-west-2"  # Change region
}
```

### Using OIDC Instead of Access Keys

For better security, use OIDC for GitHub Actions:

```yaml
# In workflow file
permissions:
  id-token: write
  contents: read

- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::ACCOUNT_ID:role/GitHubActionsRole
    aws-region: us-east-1
```

### State Backend Configuration (Already Implemented!)

The project now uses S3 backend:
```hcl
# backend.tf
terraform {
  backend "s3" {
    # Config provided at init time via backend-*.hcl files
  }
}
```

**Benefits:**
- ✅ Separate state files for dev/prod
- ✅ S3 native state locking (no DynamoDB)
- ✅ Versioning for state history
- ✅ Encryption at rest
- ✅ Simplified setup - only S3 needed
- ✅ Automatic lock management

### Disable Auto-Fix (Detection Only)

In `drift_detection.yml`, remove or comment out:
```yaml
- name: Auto-Fix Drift
  # Comment out this entire step
```

---

## Best Practices

### For Production Use:

1. **✅ Use Remote State Backend**
   - S3 + DynamoDB for locking
   - Enable versioning and encryption

2. **✅ Implement Proper IAM**
   - Use IAM roles, not access keys
   - OIDC for GitHub Actions
   - Least privilege principle

3. **✅ Add Plan Review**
   - Require approval for prod applies
   - Manual review before auto-fix

4. **✅ Sensitive Data**
   - Sanitize plan outputs
   - Don't log secrets
   - Use AWS Secrets Manager

5. **✅ Monitoring**
   - CloudWatch for AWS resources
   - GitHub Action notifications
   - Log aggregation

6. **✅ Testing**
   - Test in dev first
   - Validate plans before apply
   - Regular disaster recovery drills

---

## Additional Resources

- [Terraform Documentation](https://www.terraform.io/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS CLI Reference](https://docs.aws.amazon.com/cli/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

## Support

**Issues?**
- Check [Troubleshooting](#troubleshooting) section
- Review workflow logs in GitHub Actions
- Check AWS CloudTrail for API errors
- Open issue in this repository

---

## License

MIT License - Feel free to use and modify for your demos!

---

**Happy Drifting! 🚀**
