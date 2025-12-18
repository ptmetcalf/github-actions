# github-actions

Reusable workflows and actions for the org.

## Contents

### OpenTofu Workflows

- **Reusable Workflows**
  - `tofu-plan.yml` - Run OpenTofu plan with integrated security scanning and optional cost estimation
  - `tofu-apply.yml` - Apply OpenTofu changes with optional plan artifact
  - `tofu-security-scan.yml` - Standalone security scanning (tfsec, Checkov, Trivy)
  - `tofu-cost-estimate.yml` - Standalone cost estimation with Infracost

- **Composite Actions**
  - `actions/setup-tofu` - Install OpenTofu and common tooling

### Bicep Workflows

- **Reusable Workflows**
  - `bicep-whatif.yml` - Run Bicep what-if analysis with integrated security scanning (Checkov, PSRule)
  - `bicep-deploy.yml` - Deploy Bicep templates to Azure
  - `bicep-security-scan.yml` - Standalone security scanning (Checkov, PSRule, Trivy)

### Code Quality

- **Reusable Workflows**
  - `pre-commit-check.yml` - Run pre-commit hooks for code quality and formatting

- **Composite Actions**
  - `actions/setup-bicep` - Install Bicep CLI and Azure CLI

## Usage

### OpenTofu

Reference these workflows from your IaC repositories:

```yaml
jobs:
  plan:
    uses: YOUR_ORG/github-actions/.github/workflows/tofu-plan.yml@v1.0.0
    with:
      environment: dev
      stack_dir: infra/stacks/app
      var_file: env/dev.tfvars
    secrets: inherit
```

### Bicep

Reference Bicep workflows for Azure deployments:

```yaml
jobs:
  whatif:
    uses: YOUR_ORG/github-actions/.github/workflows/bicep-whatif.yml@v1.0.0
    with:
      environment: dev
      bicep_file: infra/main.bicep
      parameters_file: env/dev.bicepparam
      deployment_scope: resourceGroup
      resource_group_name: rg-myapp-dev
    secrets: inherit

  deploy:
    uses: YOUR_ORG/github-actions/.github/workflows/bicep-deploy.yml@v1.0.0
    with:
      environment: prod
      bicep_file: infra/main.bicep
      parameters_file: env/prod.bicepparam
      deployment_scope: resourceGroup
      resource_group_name: rg-myapp-prod
    secrets: inherit
```

## Security Scanning

All plan/what-if workflows include integrated security scanning by default.

### OpenTofu Security Tools

- **tfsec**: Fast security scanner for Terraform/OpenTofu code
- **Checkov**: Policy-as-code scanner with 1000+ built-in checks
- **Trivy**: Comprehensive vulnerability scanner (standalone workflow)

Scans run during the plan phase and fail gracefully (soft_fail: true) to provide warnings without blocking deployments.

### Bicep Security Tools

- **Checkov**: Policy-as-code scanner with Bicep support
- **PSRule for Azure**: Azure-specific best practices and security rules
- **Trivy**: Comprehensive vulnerability scanner (standalone workflow)

### Disabling Security Scans

To disable security scanning, set `enable_security_scan: false` in your workflow:

```yaml
uses: YOUR_ORG/github-actions/.github/workflows/tofu-plan.yml@v1.0.0
with:
  enable_security_scan: false
  # ... other parameters
```

### Uploading to GitHub Security Tab

The standalone security scan workflows support SARIF upload to GitHub Advanced Security:

```yaml
security-scan:
  uses: YOUR_ORG/github-actions/.github/workflows/tofu-security-scan.yml@v1.0.0
  with:
    stack_dir: infra/stacks/app
    upload_sarif: true  # Requires GitHub Advanced Security
```

## Cost Estimation

OpenTofu workflows support integrated cost estimation with Infracost.

### Features

- **Monthly cost estimates** for infrastructure changes
- **Cost breakdown** by resource type
- **PR comments** with cost diff (before/after)
- **Free tier available** at https://infracost.io

### Enabling Cost Estimation

1. **Get Infracost API Key**: Sign up at https://infracost.io (free tier available)

2. **Add GitHub Secret**: Store `INFRACOST_API_KEY` in your repository or organization secrets

3. **Enable in workflow**:
```yaml
plan:
  uses: YOUR_ORG/github-actions/.github/workflows/tofu-plan.yml@v1.0.0
  with:
    enable_cost_estimate: true
    # ... other parameters
  secrets: inherit
```

### Standalone Cost Estimation

Use the standalone workflow for more control:

```yaml
cost-estimate:
  uses: YOUR_ORG/github-actions/.github/workflows/tofu-cost-estimate.yml@v1.0.0
  with:
    stack_dir: infra/stacks/app
    var_file: env/dev.tfvars
    currency: USD
    pr_comment: true
  secrets:
    INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
```

### Bicep Cost Estimation

**Note**: Cost estimation for Bicep is currently limited. Azure native cost estimation tools are under development. Consider:
- **Azure Pricing Calculator** (manual)
- **Azure Cost Management** (post-deployment)
- **Infracost** has experimental Bicep support (ARM template conversion)

## Pre-commit Hooks

All generated repositories include pre-commit hook configurations for local code quality checks.

### Features

- **Format checking** - Ensures consistent code formatting
- **Validation** - Runs tofu/bicep validation before commit
- **Security scanning** - tfsec/Checkov checks before commit
- **Documentation** - Auto-generates terraform-docs
- **Linting** - YAML and Markdown linting

### Setup (Local Development)

```bash
# Install pre-commit
pip install pre-commit

# Install hooks in your repo
cd your-iac-repo
pre-commit install

# Run manually
pre-commit run --all-files
```

### OpenTofu Hooks

- `terraform_fmt` - Format tofu code
- `terraform_validate` - Validate configuration
- `terraform_docs` - Generate documentation
- `terraform_tfsec` - Security scanning
- `terraform_checkov` - Policy scanning
- `trailing-whitespace` - Remove trailing spaces
- `check-yaml` - YAML syntax validation
- `detect-private-key` - Prevent committing secrets

### Bicep Hooks

- `bicep-lint` - Lint Bicep files
- `bicep-build` - Build/validate Bicep
- `psrule-azure` - Azure best practices
- `checkov` - Policy scanning
- `trailing-whitespace` - Remove trailing spaces
- `check-yaml` - YAML syntax validation
- `detect-private-key` - Prevent committing secrets

### CI Integration

Pre-commit checks run automatically in CI on every PR:

```yaml
pre-commit:
  uses: YOUR_ORG/github-actions/.github/workflows/pre-commit-check.yml@v1.0.0
```

### Skipping Hooks

Skip specific hooks when needed:

```bash
# Skip a specific hook
SKIP=terraform_tfsec git commit -m "message"

# Skip all hooks (not recommended)
git commit -m "message" --no-verify
```

## Releasing

- Create git tags for versions (v1.0.0, v1.1.0, ...)
- Caller repos pin `uses:` to a tag for safe upgrades
- Test changes in a branch before tagging a new version

## Setup

### OpenTofu Workflows

Before using these workflows, configure:

1. **Cloud Authentication**: Set up Azure OIDC authentication for state backend access
   - See [BACKEND_SETUP.md](../BACKEND_SETUP.md) for Azure Storage backend setup and OIDC configuration
   - See [PROVIDER_AUTH.md](../PROVIDER_AUTH.md) for provider-specific authentication (azurerm, azuread, databricks, github, etc.)
2. **GitHub Environments**: Create environments (dev, prod) with protection rules
3. **Secrets**: Store Azure credentials as GitHub secrets (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID)

### Bicep Workflows

Before using Bicep workflows, configure:

1. **Azure OIDC Authentication**: Set up federated credentials in Azure AD
   - Create an App Registration in Azure AD
   - Add federated credentials for your GitHub repo
   - Store `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` as GitHub secrets
2. **Azure RBAC**: Grant appropriate permissions to the service principal
   - Contributor role on resource groups/subscriptions
   - Or custom roles with specific permissions
3. **GitHub Environments**: Create environments (dev, prod) with protection rules
4. **Resource Groups**: Pre-create resource groups or use subscription-level deployments
