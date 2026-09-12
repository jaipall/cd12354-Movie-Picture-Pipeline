# Movie Picture Pipeline - CI/CD Workflow Setup Complete ✅

## Overview
All GitHub Actions workflows have been configured to automate testing, building, and deploying both the frontend (React) and backend (Flask) applications to AWS EKS.

## Workflow Configuration Details

### 1. Backend Continuous Integration (`backend-ci.yaml`)
**Triggers:** Pull requests to `main` branch, pushes to `main`, manual dispatch

**Jobs (Parallel where applicable):**
1. **Lint Job**
   - Sets up Python 3.10
   - Installs dependencies via pipenv
   - Runs: `pipenv run lint` (flake8)

2. **Test Job** (runs parallel with lint)
   - Sets up Python 3.10
   - Installs dependencies via pipenv
   - Runs: `pipenv run test` (pytest)

3. **Build Job** (conditional - only if lint and test pass)
   - Builds Docker image tagged as `mp-backend:latest`
   - Note: Local build only, no push to registry

### 2. Backend Continuous Deployment (`backend-cd.yaml`)
**Triggers:** Pushes to `main` branch, manual dispatch

**Jobs:**
1. **Lint Job** - Same as CI
2. **Test Job** - Same as CI  
3. **Build Job** (depends on lint & test)
   - Authenticates with AWS
   - Logs into Amazon ECR
   - Builds Docker image with git SHA tag: `${{ github.sha }}`
   - Pushes image to ECR at: `<registry>/<backend-repo>:<git-sha>`

4. **Deploy Job** (depends on build)
   - Authenticates with AWS
   - Updates kubeconfig for EKS cluster
   - Applies Kubernetes manifests from `starter/backend/k8s/`
   - Updates deployment image using `kubectl set image` with the newly built image
   - Waits for rollout to complete

### 3. Frontend Continuous Integration (`frontend-ci.yaml`)
**Triggers:** Pull requests to `main` branch, pushes to `main`, manual dispatch

**Jobs (Parallel where applicable):**
1. **Lint Job**
   - Sets up Node.js 18
   - Installs npm dependencies
   - Runs: `npm run lint` (eslint)

2. **Test Job** (runs parallel with lint)
   - Sets up Node.js 18
   - Installs npm dependencies
   - Runs: `npm test -- --passWithNoTests --watchAll=false`

3. **Build Job** (conditional - only if lint and test pass)
   - Installs dependencies
   - Builds React app: `npm run build`
   - Builds Docker image tagged as `mp-frontend:latest`
   - Note: Local build only, no push to registry

### 4. Frontend Continuous Deployment (`frontend-cd.yaml`)
**Triggers:** Pushes to `main` branch, manual dispatch

**Jobs:**
1. **Lint Job** - Same as CI
2. **Test Job** - Same as CI
3. **Build Job** (depends on lint & test)
   - Installs dependencies
   - Builds React app with production optimizations
   - Authenticates with AWS
   - Logs into Amazon ECR
   - Builds Docker image with git SHA tag: `${{ github.sha }}`
   - Pushes image to ECR at: `<registry>/<frontend-repo>:<git-sha>`

4. **Deploy Job** (depends on build)
   - Authenticates with AWS
   - Updates kubeconfig for EKS cluster
   - Applies Kubernetes manifests from `starter/frontend/k8s/`
   - Updates deployment image using `kubectl set image` with the newly built image
   - Waits for rollout to complete

## Key Features

✅ **Lint and Test in Parallel** - Both run simultaneously to save time
✅ **Conditional Build Execution** - Only builds if all lint/test checks pass
✅ **Git SHA Tagging** - Production images tagged with exact commit SHA for traceability
✅ **Proper Job Dependencies** - Ensures correct execution order (lint,test → build → deploy)
✅ **Kubernetes Deployment** - Automatic rollout with status checks
✅ **Pull Request Support** - Full CI pipeline runs on PRs to verify changes before merge
✅ **Manual Dispatch** - All workflows can be manually triggered

## Required GitHub Secrets

Configure these secrets in your GitHub repository settings (Settings → Secrets and variables → Actions):

```
AWS_ACCESS_KEY_ID           - AWS IAM access key
AWS_SECRET_ACCESS_KEY       - AWS IAM secret access key
AWS_REGION                  - AWS region (e.g., us-east-1)
AWS_EKS_CLUSTER_NAME        - Name of your EKS cluster
AWS_ECR_REPOSITORY_BACKEND  - ECR repository name for backend
AWS_ECR_REPOSITORY_FRONTEND - ECR repository name for frontend
```

## Kubernetes Requirements

The workflows expect these Kubernetes deployments to exist:
- `backend-deployment` in default namespace (or specified namespace)
- `frontend-deployment` in default namespace (or specified namespace)

The deployments must have:
- Container name matching: `backend` (for backend) or `frontend` (for frontend)
- Service ports: 5000 for backend, 3000 for frontend

## Workflow Paths

The workflows are configured to trigger only on specific path changes:

**Backend workflows trigger on:**
- `starter/backend/**`

**Frontend workflows trigger on:**
- `starter/frontend/**`

This prevents unnecessary workflow runs when only documentation or terraform config changes.

## Image Tagging Strategy

- **CI Workflows:** Images tagged as `latest` (local testing only)
- **CD Workflows:** Images tagged with git commit SHA (production) for:
  - Perfect traceability to source code
  - Ability to quickly rollback if needed
  - Avoiding tag collisions in ECR

## Local Testing

To test the workflows locally before pushing:

1. **Backend:**
   ```bash
   cd starter/backend
   pipenv install --dev
   pipenv run lint    # Run linting
   pipenv run test    # Run tests
   docker build -t mp-backend:latest .
   ```

2. **Frontend:**
   ```bash
   cd starter/frontend
   npm install
   npm run lint       # Run linting
   npm test -- --passWithNoTests --watchAll=false  # Run tests
   npm run build      # Build for production
   docker build -t mp-frontend:latest .
   ```

## Troubleshooting

### Workflow fails on ECR login
- Verify AWS credentials are correct
- Ensure IAM user/role has ECR permissions
- Check AWS region is correct

### Deployment fails with image not found
- Verify ECR repositories exist in AWS
- Check repository names match secrets configuration
- Ensure ECR repositories are in the same region

### Kubernetes deployment fails
- Verify EKS cluster is accessible
- Check kubeconfig credentials are valid
- Ensure deployments exist with correct container names
- Verify image pull secrets if using private ECR

### Tests fail in CI but pass locally
- Ensure test environment variables match CI configuration
- Check for filesystem/path dependencies
- Verify all dependencies are in Pipfile/package.json (not installed globally)
