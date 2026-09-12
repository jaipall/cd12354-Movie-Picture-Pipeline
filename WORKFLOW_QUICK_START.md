# GitHub Actions Workflows - Quick Reference

## Workflow Files Location
All files are in `.github/workflows/` directory:
- `backend-ci.yaml` - Backend CI pipeline
- `backend-cd.yaml` - Backend CD pipeline  
- `frontend-ci.yaml` - Frontend CI pipeline
- `frontend-cd.yaml` - Frontend CD pipeline

## Quick Setup Checklist

### ✅ Step 1: GitHub Secrets Configuration
Add these secrets to your GitHub repository:
1. Go to Settings → Secrets and variables → Actions
2. Create these repository secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_REGION` (e.g., `us-east-1`)
   - `AWS_EKS_CLUSTER_NAME`
   - `AWS_ECR_REPOSITORY_BACKEND`
   - `AWS_ECR_REPOSITORY_FRONTEND`

### ✅ Step 2: AWS Setup
1. Create ECR repositories for both backend and frontend
2. Ensure EKS cluster exists with:
   - `backend-deployment` deployment
   - `frontend-deployment` deployment

### ✅ Step 3: Kubernetes Configuration
Update your K8s manifests if needed (in `starter/*/k8s/deployment.yaml`):
- Ensure container names are exactly: `backend` and `frontend`
- Ensure ports are correct: 5000 (backend), 3000 (frontend)

### ✅ Step 4: Test Workflows
1. Create a feature branch
2. Make changes to `starter/backend/**` or `starter/frontend/**`
3. Create a pull request to main
4. Workflows will automatically run CI pipeline
5. After merge to main, CD pipeline runs automatically

## Workflow Execution Flow

### On Pull Request (CI Only)
```
PR created with changes to starter/backend/** or starter/frontend/**
    ↓
Lint job runs
    ↓
Test job runs (parallel with lint)
    ↓
Both pass? → Build job runs
    ↓
All pass → Pull request approved for merge
```

### On Push to Main (CD Pipeline)
```
Code merged to main
    ↓
Lint + Test jobs run (same as CI)
    ↓
Both pass → Build job runs
    ├─ Build Docker image
    ├─ Login to ECR
    └─ Push with git SHA tag
    ↓
Build succeeds → Deploy job runs
    ├─ Get kubeconfig
    ├─ Apply K8s manifests
    └─ Update deployment image
    ↓
Deployment complete
```

## Docker Image Tags

**CI (Local Testing):**
- `mp-backend:latest`
- `mp-frontend:latest`

**CD (Production):**
- `<ecr-registry>/<repo>:<git-sha>`
- Example: `123456789.dkr.ecr.us-east-1.amazonaws.com/movie-backend:abc123def456`

## Key Commands Reference

### Backend
```bash
pipenv run lint         # Run linter (flake8)
pipenv run test         # Run tests (pytest)
pipenv run serve        # Run development server
```

### Frontend  
```bash
npm run lint            # Run linter (eslint)
npm test                # Run tests
npm run build           # Build for production
npm start               # Run development server
```

## Status Checks

GitHub will show workflow status on:
- Pull requests (CI must pass)
- Commit history (CD results)
- Actions tab (detailed logs)

All status checks must pass before PR can be merged.

## Monitoring Workflows

1. **View workflow runs:**
   - Go to Actions tab in GitHub
   - Click on workflow name to see all runs

2. **View specific run details:**
   - Click on run → View job summary
   - Click on failed step for error details

3. **View logs:**
   - Each step shows console output
   - Search for errors or specific output

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| Lint fails on PR | Fix code to match eslint/flake8 rules |
| Tests fail | Debug test failures locally first |
| ECR login fails | Check AWS credentials in secrets |
| Deployment fails | Verify EKS cluster access and deployment names |
| Tests pass locally but fail in CI | Check for environment-specific issues |

## Rollback Strategy

If deployment has issues:
1. Go to Actions tab
2. Click on failed CD run
3. Review the image SHA that was deployed
4. Use `kubectl set image` to rollback to previous image:
   ```bash
   kubectl set image deployment/backend-deployment backend=<registry>/<repo>:<previous-sha>
   ```
