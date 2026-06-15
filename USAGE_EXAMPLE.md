# How to Use This Reusable Workflow in Your Project

This document shows how other projects can call this centralized CI/CD workflow for Databricks deployments.

## Quick Start

In your project repository, create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Databricks

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod

jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop' || github.event_name == 'pull_request'
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: dev
      validation_only: ${{ github.event_name == 'pull_request' }}
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_DEV }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_DEV }}

  deploy-prod:
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: prod
      run_tests: true
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_PROD }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_PROD }}

  deploy-manual:
    if: github.event_name == 'workflow_dispatch'
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: ${{ github.event.inputs.environment }}
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
```

## Configuration Options

### Required Inputs

* `environment` - Target environment: `dev`, `staging`, or `prod`

### Optional Inputs

* `databricks_bundle_target` - Override bundle target name (default: uses `environment` value)
* `working_directory` - Path to databricks.yml (default: `.`)
* `python_version` - Python version (default: `3.10`)
* `run_tests` - Run post-deployment tests (default: `false`)
* `validation_only` - Only validate, don't deploy (default: `false`)

### Required Secrets

* `DATABRICKS_HOST` - Databricks workspace URL
* `DATABRICKS_TOKEN` - Access token or service principal token

### Optional Secrets

* `SLACK_WEBHOOK_URL` - Slack webhook for notifications

## Examples

### Example 1: Simple Auto-Deploy

```yaml
name: Auto Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: prod
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
```

### Example 2: Multi-Environment with Tests

```yaml
name: Deploy All Environments

on:
  push:
    branches: [main]

jobs:
  deploy-dev:
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: dev
      run_tests: true
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_DEV }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_DEV }}

  deploy-staging:
    needs: deploy-dev
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: staging
      run_tests: true
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_STAGING }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_STAGING }}

  deploy-prod:
    needs: deploy-staging
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: prod
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_PROD }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_PROD }}
```

### Example 3: PR Validation Only

```yaml
name: Validate PR

on:
  pull_request:
    branches: [main]

jobs:
  validate:
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: dev
      validation_only: true
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_DEV }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_DEV }}
```

### Example 4: Custom Working Directory

```yaml
name: Deploy from Subdirectory

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: prod
      working_directory: ./databricks-project
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
```

## Prerequisites for Calling Projects

### 1. Databricks Bundle Configuration

Your project must have a `databricks.yml` file:

```yaml
bundle:
  name: my-databricks-project

targets:
  dev:
    mode: development
    workspace:
      host: https://your-workspace.cloud.databricks.com
  
  prod:
    mode: production
    workspace:
      host: https://your-workspace.cloud.databricks.com

resources:
  jobs:
    my_job:
      name: ${bundle.target}_my_job
      tasks:
        - task_key: main
          notebook_task:
            notebook_path: ./notebooks/main.py
```

### 2. GitHub Secrets

Configure these secrets in your project repository:

**Settings → Secrets and variables → Actions**

* `DATABRICKS_HOST` or `DATABRICKS_HOST_DEV`/`DATABRICKS_HOST_PROD`
* `DATABRICKS_TOKEN` or `DATABRICKS_TOKEN_DEV`/`DATABRICKS_TOKEN_PROD`

### 3. GitHub Environments (Optional)

For production protection:

**Settings → Environments → New environment**

* Create `prod` environment
* Add required reviewers
* Limit deployment branches to `main`

## Post-Deployment Tests (Optional)

If you enable `run_tests: true`, create `scripts/run_tests.sh` in your project:

```bash
#!/bin/bash
TARGET=$1

echo "Running tests for target: $TARGET"

# Example: Run a test job
databricks bundle run test_job -t $TARGET

# Example: Validate data quality
# python scripts/validate_data.py --env $TARGET

exit 0
```

## Benefits of Using This Reusable Workflow

✅ **Centralized Maintenance** - Update CI/CD logic in one place
✅ **Consistency** - All projects use the same deployment process
✅ **Reduced Duplication** - No need to copy workflow code to each project
✅ **Versioning** - Pin to specific versions with `@v1` or use `@main` for latest
✅ **Security** - Secrets managed per project, workflow logic centralized

## Versioning

Reference specific versions of the workflow:

```yaml
# Use latest from main branch
uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main

# Use specific version tag
uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@v1.0.0

# Use specific commit
uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@abc1234
```

## Troubleshooting

### Error: "workflow was not found"

* Verify the repository name and workflow path
* Ensure the workflow file exists in the referenced branch/tag
* Check repository visibility (must be public or in same org)

### Error: "secret not found"

* Verify secrets are configured in the calling repository
* Secret names are case-sensitive

### Deployment fails but validation passes

* Check Databricks workspace permissions
* Verify token has necessary privileges
* Review Databricks bundle configuration

## Support

For issues with the reusable workflow, contact the data platform team or open an issue in the `data-platform-devops` repository.