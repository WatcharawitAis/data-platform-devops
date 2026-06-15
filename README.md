# Databricks CI/CD Reusable Workflows

This repository provides **reusable GitHub Actions workflows** for deploying Databricks Asset Bundles (DABs) across multiple projects. Instead of duplicating CI/CD logic in every project, teams can call these centralized workflows to maintain consistency and simplify maintenance.

## What This Repository Contains

* **Reusable Workflow** (`.github/workflows/reusable-deploy.yml`) - A parameterized workflow that other repositories can call to deploy their Databricks resources
* **Example Workflow** (`.github/workflows/deploy.yml`) - A sample implementation showing direct deployment
* **Usage Documentation** (`USAGE_EXAMPLE.md`) - Detailed examples and configuration options
* **CODEOWNERS** - Defines who reviews changes to the workflows

## Quick Start for Calling Projects

### 1. In Your Databricks Project Repository

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Databricks

on:
  push:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, staging, prod]

jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: dev
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_DEV }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_DEV }}

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@main
    with:
      environment: prod
      run_tests: true
    secrets:
      DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST_PROD }}
      DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN_PROD }}
```

**Replace `YOUR_ORG` with your GitHub organization name.**

### 2. Configure Secrets in Your Project

In your calling repository, add these secrets:

**Settings → Secrets and variables → Actions → New repository secret**

* `DATABRICKS_HOST_DEV` - Dev workspace URL (e.g., `https://your-workspace.cloud.databricks.com`)
* `DATABRICKS_TOKEN_DEV` - Dev access token
* `DATABRICKS_HOST_PROD` - Prod workspace URL
* `DATABRICKS_TOKEN_PROD` - Prod access token

### 3. Ensure Your Project Has `databricks.yml`

Your calling repository must have a Databricks Asset Bundle configuration:

```yaml
bundle:
  name: my-project

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
      # ... job configuration
```

## Workflow Features

### Reusable Workflow (`reusable-deploy.yml`)

**Inputs:**
* `environment` (required) - Target environment: `dev`, `staging`, or `prod`
* `databricks_bundle_target` (optional) - Override bundle target name
* `working_directory` (optional) - Path to `databricks.yml` (default: `.`)
* `python_version` (optional) - Python version (default: `3.10`)
* `run_tests` (optional) - Run post-deployment tests (default: `false`)
* `validation_only` (optional) - Only validate without deploying (default: `false`)

**Secrets:**
* `DATABRICKS_HOST` (required) - Databricks workspace URL
* `DATABRICKS_TOKEN` (required) - Access token
* `SLACK_WEBHOOK_URL` (optional) - Slack notifications

**Steps:**
1. Checkout calling repository
2. Set up Python and Databricks CLI
3. Configure authentication
4. Validate bundle
5. Deploy bundle (if not validation-only)
6. Run tests (if enabled)
7. Report status

## Benefits

✅ **Centralized Maintenance** - Update CI/CD logic in one place, all projects benefit  
✅ **Consistency** - All projects use the same deployment process  
✅ **Reduced Duplication** - No need to copy workflow code to each project  
✅ **Version Control** - Pin to specific versions with `@v1.0.0` or use `@main` for latest  
✅ **Security** - Secrets managed per project, workflow logic centralized

## Repository Structure

```
data-platform-devops/
├── .github/
│   ├── CODEOWNERS                      # Code review assignments
│   └── workflows/
│       ├── reusable-deploy.yml         # Main reusable workflow
│       └── deploy.yml                   # Example implementation
├── .gitignore
├── README.md                            # This file
└── USAGE_EXAMPLE.md                     # Detailed usage examples
```

## Advanced Usage

For more examples and advanced configurations, see [USAGE_EXAMPLE.md](USAGE_EXAMPLE.md):

* Multi-environment deployments with dependencies
* PR validation without deployment
* Custom working directories
* Post-deployment testing
* Workflow versioning strategies

## Maintenance

### Updating the Workflow

Changes to workflows in this repository will affect all projects that reference `@main`. For breaking changes:

1. Create a new version tag (e.g., `v2.0.0`)
2. Update calling projects to reference the new version
3. Deprecate old versions with clear migration guides

### Testing Changes

Test workflow changes in a separate branch before merging to `main`:

```yaml
# In your test project
uses: YOUR_ORG/data-platform-devops/.github/workflows/reusable-deploy.yml@feature/my-changes
```

## Prerequisites for Calling Projects

Each project using these workflows must have:

1. **Databricks Asset Bundle** - `databricks.yml` configuration file
2. **GitHub Secrets** - Workspace URLs and access tokens
3. **Proper Permissions** - Service principals or tokens with deployment rights
4. **Target Environments** - Defined in `databricks.yml` (`dev`, `staging`, `prod`)

## Troubleshooting

**Error: "workflow was not found"**
* Verify repository name and workflow path in `uses:` statement
* Ensure workflow exists in the referenced branch/tag
* Check repository visibility (public or same organization)

**Error: "secret not found"**
* Verify secrets are configured in the **calling repository**
* Secret names are case-sensitive
* Use environment-specific secret names (e.g., `DATABRICKS_HOST_DEV`)

**Deployment succeeds but resources not updated**
* Check bundle target matches environment
* Verify workspace URL and permissions
* Review Databricks workspace for actual resource state

## Additional Resources

* [Databricks Asset Bundles Documentation](https://docs.databricks.com/dev-tools/bundles/index.html)
* [GitHub Reusable Workflows Documentation](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
* [Databricks CLI Reference](https://docs.databricks.com/dev-tools/cli/index.html)

## Support

For issues or questions about these workflows:
* Open an issue in this repository
* Contact the Data Platform team
* Review [USAGE_EXAMPLE.md](USAGE_EXAMPLE.md) for detailed examples