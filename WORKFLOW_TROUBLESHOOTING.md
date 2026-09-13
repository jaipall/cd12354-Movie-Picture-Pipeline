# GitHub Actions Workflow Issues - Resolution Guide

## Issues Fixed ✅

### 1. Node.js 20 Deprecation - FIXED
**What was wrong:** Actions/checkout@v3 and amazon-ecr-login@v1 target Node.js 20, which is deprecated
**What was fixed:** 
- Updated `actions/checkout@v3` → `actions/checkout@v4`
- Updated `aws-actions/amazon-ecr-login@v1` → `aws-actions/amazon-ecr-login@v2`
- All workflows now run on Node.js 24 ✅

### 2. Docker Password Not Masked - FIXED
**What was wrong:** ECR login credentials were visible in logs
**What was fixed:**
- Added `mask-password: 'true'` to all ECR login actions
- Docker credentials now properly masked in workflow logs ✅

---

## Remaining Issue: AWS IAM Authorization Error

### Error Message
```
User: arn:aws:iam::***:user/AdministratorAccess is not authorized to perform: 
ecr:GetAuthorizationToken on resource: * with an explicit deny in a service control
```

### Root Cause
The AWS IAM user/role doesn't have the required permissions to authenticate with ECR, likely due to:
1. Missing `ecr:GetAuthorizationToken` permission
2. Service Control Policy (SCP) explicitly denying the action
3. Permission boundary restricting ECR access

### How to Fix (AWS Administrator)

#### Option 1: Attach ECR Permissions to IAM User
1. Go to AWS IAM Console
2. Find the user: `AdministratorAccess` (in the ARN shown in error)
3. Click "Add permissions" → "Attach policies directly"
4. Search for and attach: `AmazonEC2ContainerRegistryPowerUser` or `AmazonEC2ContainerRegistryFullAccess`
5. **Alternative:** Create inline policy with:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ecr:GetDownloadUrlForLayer",
           "ecr:BatchGetImage",
           "ecr:GetAuthorizationToken",
           "ecr:BatchCheckLayerAvailability",
           "ecr:PutImage",
           "ecr:InitiateLayerUpload",
           "ecr:UploadLayerPart",
           "ecr:CompleteLayerUpload"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

#### Option 2: Check Service Control Policies (SCP)
1. Go to AWS Organizations (if enabled)
2. Check if any SCP is denying `ecr:*` actions
3. Update the SCP to allow ECR permissions

#### Option 3: Check Permission Boundaries
1. Go to IAM → Users → AdministratorAccess
2. Check "Permission boundary" section
3. If set, ensure it allows ECR actions

### Verification Steps

After fixing permissions, verify with these commands:

```bash
# Check IAM permissions for GetAuthorizationToken
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT_ID:user/USERNAME \
  --action-names ecr:GetAuthorizationToken \
  --resource-arns "*"

# Test ECR login
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```

### What to Check in GitHub Actions

1. **Verify GitHub Secrets are correct:**
   - Go to Settings → Secrets and variables → Actions
   - Ensure `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are for the correct user

2. **Verify AWS Credentials:**
   - Confirm these are active access keys (not disabled)
   - Check they match the IAM user: `AdministratorAccess`

3. **Check AWS Region:**
   - Ensure `AWS_REGION` secret matches where your ECR repositories are located

---

## All Workflow Versions Updated

| Component | Old | New | Status |
|-----------|-----|-----|--------|
| actions/checkout | v3 | v4 | ✅ |
| aws-actions/amazon-ecr-login | v1 | v2 | ✅ |
| aws-actions/configure-aws-credentials | v2 | v2 | ✅ (already current) |
| Password masking | ❌ None | ✅ Added | ✅ |

---

## Testing the Fix

Once AWS IAM permissions are configured:

1. **Push a test commit** to a feature branch:
   ```bash
   git checkout -b test-ci
   # Make a small change
   git commit -m "test: verify CI pipeline"
   git push origin test-ci
   ```

2. **Create a Pull Request** to main
   - CI pipeline should run without errors
   - Check Actions tab for successful completion

3. **Merge to main** (after PR is approved)
   - CD pipeline should run
   - Docker image should build and push to ECR successfully
   - Deployment to EKS should complete

4. **Verify in AWS Console:**
   - Go to ECR
   - Check if new images appear with git SHA tags
   - Verify timestamps match your push time

---

## Quick Troubleshooting Checklist

- [ ] AWS IAM user has `ecr:GetAuthorizationToken` permission
- [ ] No Service Control Policies (SCP) deny ECR actions
- [ ] No Permission boundaries restrict ECR access
- [ ] GitHub Secrets contain correct AWS credentials
- [ ] AWS credentials are active (not disabled)
- [ ] GitHub Secrets region matches ECR repository region
- [ ] ECR repositories exist in AWS account
- [ ] EKS cluster exists and is accessible
- [ ] Kubernetes deployments exist: `backend-deployment`, `frontend-deployment`

---

## Still Having Issues?

### Check Workflow Logs
1. Go to GitHub → Actions tab
2. Click on the failed workflow run
3. Expand the failed job/step
4. Look for error details in console output

### Common Error Messages & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `UnrecognizedClientException` | Invalid AWS credentials | Verify Access Key ID & Secret in secrets |
| `NoSuchRepository` | ECR repo doesn't exist | Create ECR repositories in AWS |
| `AccessDenied` | Missing IAM permissions | Add ecr:* permissions to IAM user |
| `ImageNotFound` | Wrong region | Verify AWS_REGION in secrets |
| `EKSAccessDenied` | No EKS cluster access | Ensure kubeconfig permissions are set |

### Get Help
1. Check GitHub Actions documentation: https://docs.github.com/en/actions
2. Check AWS ECR documentation: https://docs.aws.amazon.com/ecr/
3. Review workflow logs for exact error timestamps
