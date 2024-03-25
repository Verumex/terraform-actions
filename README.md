## Example usage

### Run `terraform apply` on push to main branch

This also triggers when a pull request is merged

#### apply.yaml

```yaml
name: terraform apply

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  apply:
    runs-on: ubuntu-latest
    name: terraform apply
    env:
      # Terraform accepts assigning variables via environment variables in a
      # form of `TF_VAR_<name>`. https://developer.hashicorp.com/terraform/language/values/variables#environment-variables
      # Sensitive variables. Environment specific & repository level variables
      # will be available to the action.
      #
      # Environment secrets doc: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-an-environment
      # Repository secrets doc: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions
      TF_VAR_top_secret: ${{ secrets.TOP_SECRET }}
      # non-sensitive terraform variable
      TF_VAR_variable1: ${{ vars.variable1 }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - uses: Verumex/terraform-actions/apply@v1
```

### plan on pull request open

Open a PR to trigger `terraform plan`.

#### plan.yaml

```yaml
name: terraform plan

on:
  pull_request:
    # run plan only on PRs against 'main' branch
    branches:
      - main

permissions:
  # read PR diff
  contents: read
  # required for github-bot to write plan comment to a PR
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    name: terraform plan
    env:
      # Terraform accepts assigning variables via environment variables in a
      # form of `TF_VAR_<name>`. https://developer.hashicorp.com/terraform/language/values/variables#environment-variables
      # Sensitive variables. Environment specific & repository level variables
      # will be available to the action.
      #
      # Environment secrets doc: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-an-environment
      # Repository secrets doc: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions
      TF_VAR_top_secret: ${{ secrets.TOP_SECRET }}
      # non-sensitive terraform variable
      TF_VAR_variable1: ${{ vars.variable1 }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - uses: Verumex/terraform-actions/plan@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          terraform_version: "1.5.4"
```

### Plan with custom backend config

You can provide custom partial backend configuration to terraform during `terraform init` step. See https://developer.hashicorp.com/terraform/language/settings/backends/configuration#partial-configuration.
This is useful when you manage AWS resources and use S3 backend that belongs to another AWS account.

For example, you have an S3 backend config block.

```hcl
backend "s3" {
  bucket         = "bucket-name"
  key            = "path/to/state.tfstate"
  region         = "us-east-1"
  encrypt        = true
}
```

and the backend bucket lives in another AWS account. We can pass custom [partial backend configurations](https://developer.hashicorp.com/terraform/language/settings/backends/configuration#partial-configuration)
to load during init step without hard-coding or compromising secrets.

#### plan.yaml

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    name: terraform plan
    env:
      # Main AWS user credentials used to manage resources
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - uses: Verumex/terraform-actions/plan@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          terraform_version: "~1.5.0"
          backend_config: |
            # AWS IAM user credentials that owns the state S3 bucket
            access_key = "${{ secrets.STATE_BACKEND_AWS_ACCESS_KEY_ID }}"
            secret_key = "${{ secrets.STATE_BACKEND_AWS_SECRET_ACCESS_KEY }}"
```

Both `access_key` and `secret_key` will come from `backend_config` input, equivelent to.

```hcl
backend "s3" {
  bucket     = "bucket-name"
  key        = "path/to/state.tfstate"
  region     = "us-east-1"
  encrypt    = true
  # supplement configs from github action input `backend_config`
  access_key = "AX............"
  secret_key = "efym.........."
}
```

### Use cache to optimize plugins download

Similar to the previous example workflows, we may add an additional step to retrieve
plugin cache. The workflow environment `TF_PLUGIN_CACHE_DIR` is required for
caching to work.

#### plan.yaml

```yaml
env:
  TF_PLUGIN_CACHE_DIR: ${{ github.workspace }}/.terraform.d/plugin-cache

steps:
  - name: Checkout
    uses: actions/checkout@v4

  - name: Cache terraform providers
    uses: actions/cache@v3
    with:
      path: ${{ env.TF_PLUGIN_CACHE_DIR }}
      # Cache provider modules based on the lock file. Path is relative from
      # repository root.
      key: ${{ runner.os }}-${{ hashFiles('./.terraform.lock.hcl') }}

  - uses: Verumex/terraform-actions/plan@v1
    with:
      github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Use generated plan

The plan is uploaded as a GitHub Workflow Run Artifact and named per input `plan_artifact_name`. The action also has as outputs the typical:
- artifact-id
- artifact-url

## Inputs

### plan

- `github_token` - (**required**) A github action token. Used for writing a plan comment back to a pull request.
  - type: String
  - **required**
- `terraform_version` - (optional) Terraform version used to run action.
  - type: String
  - optional
  - default: `"1.5.7"`
- `path` - (optional) Path to Terraform root module
  - type: String
  - optional
  - default: `"./"`
- `plan_artifact_name` - (optional) The name of the GitHub Actions Run Artifact holding the uploaded plan
  - type: String
  - optional
  - default: `"artifact"`
- `custom_job_id` - (optional) The name of the GitHub Job to use to identify a plan comment initially and subsequently
  - type: String
  - optional
  - default: `""`; the action will use the default `github.job` [context variable](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context) if undefined
- `backend_config` - (optional) Additional backend configuration to use during 'terraform init'. See https://developer.hashicorp.com/terraform/language/settings/backends/configuration#partial-configuration.
  - type: String
  - optional
  - default: `""`

### apply

- `terraform_version` - (optional) Terraform version used to run action.
  - type: String
  - optional
  - default: `"1.5.7"`
- `path` - (optional) Path to Terraform root module
  - type: String
  - optional
  - default: `"./"`
- `plan_path` - (optional) Path to a Terraform Plan file to reuse
  - type: String
  - optional
- `backend_config` - (optional) Additional backend configuration to use during 'terraform init'. See https://developer.hashicorp.com/terraform/language/settings/backends/configuration#partial-configuration.
  - type: String
  - optional
  - default: `""`
