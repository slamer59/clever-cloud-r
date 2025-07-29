# Testing the PR Deployment Workflow

This guide provides step-by-step commands to test the PR deployment workflow locally and verify it works correctly.

## Prerequisites

Before testing, ensure you have:
- Git repository with the workflow file
- GitHub CLI (`gh`) installed
- Access to the repository with required secrets configured

## Step 1: Verify GitHub Secrets

First, check that all required secrets are configured:

```bash
# List all repository secrets (requires admin access)
gh secret list

# Verify specific secrets exist (will show if they exist, not their values)
gh secret list | grep -E "(CLEVER_TOKEN|CLEVER_SECRET|ORGA_ID)"
```

Expected secrets to be present:
- `CLEVER_TOKEN`
- `CLEVER_SECRET` 
- `ORGA_ID`
- `CC_CACHE_DEPENDENCIES`
- `CC_CGI_IMPLEMENTATION`
- `CC_NODE_DEV_DEPENDENCIES`
- `CC_WEBROOT`
- `HOST`
- `NODE_ENV`
- `PORT`
- `POSTGRESQL_ADDON_HOST`
- `POSTGRESQL_ADDON_DB`
- `POSTGRESQL_ADDON_USER`
- `POSTGRESQL_ADDON_PORT`
- `POSTGRESQL_ADDON_PASSWORD`
- `POSTGRESQL_ADDON_URI`

## Step 2: Create a Test Branch and PR

```bash
# Create and switch to a new test branch
git checkout -b test-pr-deployment

# Make a small change to trigger the workflow
echo "<!-- Test PR deployment -->" >> front/README.md

# Or create a test file if front/README.md doesn't exist
mkdir -p front
echo "# Test PR Deployment" > front/test-deployment.md

# Commit and push the changes
git add .
git commit -m "test: trigger PR deployment workflow"
git push origin test-pr-deployment
```

## Step 3: Create the Pull Request

```bash
# Create a PR using GitHub CLI
gh pr create \
  --title "Test PR Deployment Workflow" \
  --body "Testing the automated PR deployment to Clever Cloud" \
  --base main \
  --head test-pr-deployment

# Get the PR number for reference
PR_NUMBER=$(gh pr view --json number --jq .number)
echo "Created PR #$PR_NUMBER"
```

## Step 4: Monitor the Workflow

```bash
# Watch the workflow run
gh run list --workflow="pr-deploy.yml" --limit 1

# Get detailed logs of the latest run
gh run view --log

# Or watch in real-time
gh run watch
```

## Step 5: Verify Deployment

```bash
# Check if the deployment URL is accessible
PR_NUMBER=$(gh pr view --json number --jq .number)
DEPLOYMENT_URL="https://eclaireur-pr-$PR_NUMBER.cleverapps.io"
echo "Testing deployment at: $DEPLOYMENT_URL"

# Test the deployment URL (requires curl)
curl -I "$DEPLOYMENT_URL" || echo "Deployment not yet ready"

# Check PR comments for deployment status
gh pr view --comments
```

## Step 6: Test Manual Workflow Dispatch

```bash
# Trigger manual deployment with custom parameters
gh workflow run pr-deploy.yml \
  --field pr_number="$PR_NUMBER" \
  --field scale="S" \
  --field force_deploy="false"

# Monitor the manual run
gh run list --workflow="pr-deploy.yml" --limit 1
```

## Step 7: Test Different Scenarios

### Test Force Deployment
```bash
# Create another commit to test force deployment
echo "<!-- Force deployment test -->" >> front/test-deployment.md
git add .
git commit -m "test: force deployment scenario"
git push origin test-pr-deployment

# Trigger with force flag
gh workflow run pr-deploy.yml \
  --field pr_number="$PR_NUMBER" \
  --field scale="XS" \
  --field force_deploy="true"
```

### Test Different Instance Sizes
```bash
# Test each instance size
for SCALE in XS S M L XL XXL; do
  echo "Testing scale: $SCALE"
  gh workflow run pr-deploy.yml \
    --field pr_number="$PR_NUMBER" \
    --field scale="$SCALE" \
    --field force_deploy="false"
  
  # Wait a moment between runs
  sleep 30
done
```

## Step 8: Test Cleanup Process

```bash
# Close the PR to test cleanup
gh pr close "$PR_NUMBER"

# Verify cleanup workflow runs
gh run list --workflow="pr-deploy.yml" --limit 1

# Check that PR comment was updated
gh pr view "$PR_NUMBER" --comments
```

## Step 9: Verify Workflow File Syntax

```bash
# Validate workflow YAML syntax
yamllint .github/workflows/pr-deploy.yml

# Or use GitHub's action validation
gh workflow view pr-deploy.yml
```

## Step 10: Clean Up Test Resources

```bash
# Delete the test branch (locally and remote)
git checkout main
git branch -D test-pr-deployment
git push origin --delete test-pr-deployment

# Optionally delete the closed PR
gh pr delete "$PR_NUMBER" --yes
```

## Troubleshooting Commands

### Check Workflow Status
```bash
# List all workflow runs
gh run list --workflow="pr-deploy.yml"

# View specific run details
gh run view [RUN_ID] --log

# Check failed steps
gh run view [RUN_ID] --log | grep -A 10 -B 10 "Error\|Failed"
```

### Debug Secrets
```bash
# Test if secrets are accessible (don't log actual values)
gh secret list | wc -l  # Should show number of secrets

# Verify repository permissions
gh auth status
gh repo view --json permissions
```

### Check Clever Cloud Apps
```bash
# If you have clever-tools installed locally
clever login
clever applications

# Look for PR-specific apps
clever applications | grep "eclaireur-pr"
```

### Validate Environment
```bash
# Check if front/ directory exists and has content
ls -la front/

# Verify workflow file exists
ls -la .github/workflows/pr-deploy.yml

# Check git status
git status
git remote -v
```

## Expected Results

After successful testing, you should see:

1. ✅ Workflow runs without errors
2. ✅ PR comment is created/updated with deployment info
3. ✅ Deployment URL is accessible
4. ✅ Different instance scales work
5. ✅ Force deployment functions correctly
6. ✅ Cleanup process updates PR comment on close
7. ✅ Manual workflow dispatch works

## Common Issues and Solutions

### Issue: Workflow doesn't trigger
```bash
# Check if paths filter matches your changes
git diff --name-only main...HEAD | grep "front/"

# Ensure PR targets main branch
gh pr view --json baseRefName
```

### Issue: Deployment fails
```bash
# Check workflow logs for specific error
gh run view --log | grep -i error

# Verify all secrets are set
gh secret list | wc -l
```

### Issue: App creation fails
```bash
# Check if app name conflicts exist
# App names must be unique across Clever Cloud
gh run view --log | grep "eclaireur-pr"
```

This testing guide covers all major scenarios and provides commands to verify the PR deployment workflow works correctly.