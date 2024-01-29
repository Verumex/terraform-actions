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

### use cache to optimize plugins download

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

### apply

- `terraform_version` - (optional) Terraform version used to run action.
  - type: String
  - optional
  - default: `"1.5.7"`
- `path` - (optional) Path to Terraform root module
  - type: String
  - optional
  - default: `"./"`
