# Azure DevOps Pipeline Template

A production-ready, multi-stage Azure DevOps YAML pipeline designed as a reusable baseline for CI/CD scenarios.

---

## Features

| Category | Capabilities |
|----------|-------------|
| **Triggers** | CI (branch + path filters), PR validation, Scheduled (nightly) |
| **Build** | Matrix builds, Caching, Artifact publishing |
| **Deploy** | Environment-based deployments, Azure CLI integration, Bicep/Terraform support |
| **Architecture** | Multi-repo checkout, Reusable templates, Conditional stages |

---

## Pipeline Flow

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Prep   │───▶│  Build  │───▶│  Test   │───▶│  Infra  │───▶│   App   │───▶│  Post   │
│         │    │         │    │(optional)│    │(optional)│    │         │    │(always) │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
     │              │              │              │              │              │
  checkout      matrix         unit tests     Bicep/TF       artifact       notify
  multi-repo    cache          pytest         deploy         deploy         cleanup
  set outputs   artifacts      dotnet test    az cli         app logic      work items
```

---

## Quick Start

1. Copy `az-pipeline.yaml` into your repository
2. Create environment variable files in `variables/`:
   ```
   variables/
   ├── vars-dev.yaml
   ├── vars-acc.yaml
   └── vars-prod.yaml
   ```
3. Update repository references in `resources.repositories`
4. Configure your Azure Service Connection
5. Run the pipeline manually to validate

---

## File Structure

```
├── az-pipeline.yaml          # Main pipeline definition
└── variables/
    ├── vars-dev.yaml         # Dev environment variables
    ├── vars-acc.yaml         # Acceptance environment variables
    └── vars-prod.yaml        # Production environment variables
```

---

## Parameters

Compile-time parameters for manual or triggered runs:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `environment` | string | `dev` | Target environment (`dev`, `acc`, `prod`) |
| `runInfra` | boolean | `true` | Toggle infrastructure deployment stage |
| `runTests` | boolean | `true` | Toggle test execution stage |
| `extraArgs` | string | `""` | Extra CLI args for infra deploy (e.g., `--what-if`) |

**Example manual trigger:**

```yaml
environment: acc
runInfra: true
runTests: false
extraArgs: "--what-if"
```

---

## Stages

### 1. Prep

Checkout repositories and set output variables for downstream stages.

- Checks out `self`, `infraRepo`, and `templatesRepo`
- Logs directory paths for debugging
- Exports output variables (e.g., git SHA)

### 2. Build

Build the application using matrix strategy with caching.

- Matrix job for parallel builds
- npm/pip cache restoration
- Publishes `drop` artifact

### 3. Test *(conditional)*

Execute unit tests. Only runs when `runTests=true`.

### 4. Infra *(conditional)*

Deploy infrastructure using Azure CLI. Only runs when `runInfra=true`.

- Uses deployment job with environment gates
- Executes Bicep/Terraform/ARM deployment
- Runs against configured Azure Service Connection

### 5. App

Deploy application using artifacts from Build stage.

- Downloads `drop` artifact
- Executes deployment logic

### 6. Post *(always runs)*

Finalization tasks that run regardless of previous stage results.

- Send notifications
- Create work items
- Cleanup resources

---

## Variables

### Inline Variables

```yaml
variables:
  vmImage: ubuntu-latest
  buildConfig: Release
```

### Variable Groups

Loaded from Azure DevOps Library:

```yaml
- group: shared-secrets
```

Store secrets like subscription IDs, service principals, and tokens.

### Environment Templates

Loaded per environment:

```yaml
- template: variables/vars-${{ parameters.environment }}.yaml
```

**Required variables in each environment file:**

```yaml
azureServiceConnection: "spoke-robin-dev"
adoEnvironmentName: "robin-dev"
location: "westeurope"
```

---

## Multi-Repo Checkout

### Resource Definition

```yaml
resources:
  repositories:
  - repository: infraRepo
    name: MyProject/infra-repo
  - repository: templatesRepo
    name: MyProject/pipeline-templates
```

### Checkout Usage

```yaml
- checkout: infraRepo
  path: shared-infra
```

### Path Variables

| Variable | Description |
|----------|-------------|
| `$(System.DefaultWorkingDirectory)` | Root of self repo |
| `$(Pipeline.Workspace)` | Root for all repos + artifacts |
| `$(Build.SourcesDirectory)` | Self repo root |

> **Note:** In deployment jobs, extra repos are located under `$(Pipeline.Workspace)/<path>`

**Example path setup:**

```bash
SELF_ROOT="$(System.DefaultWorkingDirectory)"
INFRA_ROOT="$(Pipeline.Workspace)/shared-infra"
TEMPLATES_ROOT="$(Pipeline.Workspace)/templates"
```

---

## Azure CLI Tasks

Infrastructure stage uses `AzureCLI@2`:

```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: $(azureServiceConnection)
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az group list -o table
```

Ensure your Service Connection has appropriate RBAC permissions on the target subscription.

---

## Artifacts

### Publishing (Build Stage)

```yaml
- publish: $(System.DefaultWorkingDirectory)/dist
  artifact: drop
```

### Downloading (App Stage)

```yaml
- download: current
  artifact: drop
```

Artifact location: `$(Pipeline.Workspace)/drop`

---

## Troubleshooting

### Debug Path Variables

Add this step to debug directory structure:

```bash
echo "Build.SourcesDirectory=$(Build.SourcesDirectory)"
echo "System.DefaultWorkingDirectory=$(System.DefaultWorkingDirectory)"
echo "Pipeline.Workspace=$(Pipeline.Workspace)"
pwd
find "$(Pipeline.Workspace)" -maxdepth 2 -type d
```

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `unexpected value stage` | `stage` used without `stages:` | Wrap top-level in `stages:` |
| Repo not found under `/s/` | Deployment jobs store extra repos elsewhere | Use `$(Pipeline.Workspace)` |
| File not found | Variable points to wrong repo | Correct variable paths |

---

## License

This pipeline template is intended for internal reuse. Fork and adapt per application or infrastructure stack.
