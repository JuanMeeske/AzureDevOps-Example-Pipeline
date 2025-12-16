# Azure DevOps Kubernetes Pipeline Template

A comprehensive Azure DevOps pipeline template for deploying to Kubernetes clusters. Supports kubectl, Helm, and Kustomize deployment strategies.

## Files

| File | Description |
|------|-------------|
| `az-pipeline-k8s.yaml` | Full pipeline (all-in-one, no templates) |
| `az-pipeline-k8s-templated.yaml` | Main pipeline using reusable templates |
| `templates/install-k8s-tools.yaml` | Template: Install K8s tools (kubectl, helm, etc.) |
| `templates/get-k8s-credentials.yaml` | Template: Get K8s cluster credentials |

## Features

- **Tool Installation**: Automatically installs kubectl, Helm, Kustomize, and optional tools (kubeseal, k9s)
- **Multiple Deployment Strategies**: kubectl manifests, Helm charts, or Kustomize overlays
- **Environment Support**: Parameterized for dev, staging, and prod environments
- **Validation Stage**: Dry-run validation before deployment
- **Rollout Verification**: Automatic deployment status checks
- **Rollback Support**: Manual rollback stage for emergencies
- **Container Build**: Integrated Docker build and push to registry
- **Reusable Templates**: Modular templates for tools and credentials

## Prerequisites

### Azure DevOps Setup

1. **Service Connections**
   - **Kubernetes Service Connection**: Connect to your K8s cluster
     - Azure DevOps → Project Settings → Service Connections → New → Kubernetes
   - **Azure Resource Manager** (for AKS): If using Azure Kubernetes Service
   - **Container Registry**: Docker Hub, ACR, or other registry

2. **Variable Groups**
   Create a variable group named `k8s-secrets` with:
   ```
   kubernetesServiceConnection    # Name of your K8s service connection
   containerRegistryServiceConnection  # Name of your container registry connection
   containerRegistry              # e.g., myacr.azurecr.io
   containerRepository            # e.g., myapp
   ```

3. **Environments**
   Create environments in Azure DevOps for approval gates:
   - `dev-k8s`
   - `staging-k8s`
   - `prod-k8s`

4. **Variable Templates** (optional)
   Create environment-specific variable files:
   ```
   variables/
   ├── vars-dev.yaml
   ├── vars-staging.yaml
   └── vars-prod.yaml
   ```

   Example `vars-dev.yaml`:
   ```yaml
   variables:
     aksResourceGroup: rg-dev-aks
     azureServiceConnection: azure-dev-connection
     kubernetesServiceConnection: k8s-dev-connection
   ```

## Pipeline Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `environment` | string | `dev` | Target environment (dev/staging/prod) |
| `kubernetesCluster` | string | `aks-dev-cluster` | Target K8s cluster name |
| `namespace` | string | `default` | Target Kubernetes namespace |
| `deploymentStrategy` | string | `kubectl` | Deployment method: kubectl/helm/kustomize |
| `helmReleaseName` | string | `myapp` | Helm release name |
| `helmChartPath` | string | `charts/myapp` | Path to Helm chart |
| `kubectlManifestPath` | string | `k8s/manifests` | Path to K8s manifests |
| `imageTag` | string | `latest` | Container image tag |
| `runTests` | boolean | `true` | Run validation stage |
| `dryRun` | boolean | `false` | Skip actual deployment |

## Directory Structure

Expected project structure for this pipeline:

```
your-repo/
├── azure-pipelines.yaml      # Copy from az-pipeline-k8s.yaml
├── Dockerfile
├── src/
│   └── ...                   # Application source code
├── k8s/
│   └── manifests/            # For kubectl strategy
│       ├── deployment.yaml
│       ├── service.yaml
│       └── configmap.yaml
├── charts/
│   └── myapp/                # For Helm strategy
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
└── variables/
    ├── vars-dev.yaml
    ├── vars-staging.yaml
    └── vars-prod.yaml
```

## Usage

### Quick Start

1. Copy `az-pipeline-k8s.yaml` to your repository root
2. Rename to `azure-pipelines.yaml`
3. Create required service connections in Azure DevOps
4. Create variable group `k8s-secrets`
5. Customize parameters for your project
6. Push and run!

### Using kubectl Manifests

```yaml
# Run with default kubectl strategy
parameters:
  environment: dev
  deploymentStrategy: kubectl
  kubectlManifestPath: k8s/manifests
  namespace: myapp-dev
```

### Using Helm Charts

```yaml
parameters:
  environment: staging
  deploymentStrategy: helm
  helmChartPath: charts/myapp
  helmReleaseName: myapp
  namespace: myapp-staging
  imageTag: v1.2.3
```

### Using Kustomize

```yaml
parameters:
  environment: prod
  deploymentStrategy: kustomize
  kubectlManifestPath: k8s/base
  namespace: myapp-prod
```

Expected kustomize structure:
```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

## Pipeline Stages

| Stage | Description |
|-------|-------------|
| **Prep** | Checkout code, install kubectl/Helm/Kustomize, set variables |
| **Build** | Build container image, push to registry, publish artifacts |
| **Validate** | Lint and dry-run validation of K8s resources |
| **Deploy** | Deploy to Kubernetes using selected strategy |
| **Verify** | Check rollout status, get deployment info, health check |
| **Post** | Summary, notifications, optional cleanup |
| **Rollback** | Manual rollback (triggered by variable) |

## Tools Installed

| Tool | Version | Purpose |
|------|---------|---------|
| Azure CLI | latest | Azure resource management |
| kubectl | 1.31.0 | Kubernetes CLI |
| Helm | 3.16.3 | Kubernetes package manager |

## Service Connection Options

### Option 1: Kubernetes Service Connection (Recommended)

Works with any Kubernetes cluster:
- AKS, EKS, GKE, on-premises, etc.
- Uses kubeconfig or service account

### Option 2: Azure AKS Direct

For AKS clusters, uncomment the AzureCLI task in Deploy stage:
```yaml
- task: AzureCLI@2
  displayName: "Get AKS credentials"
  inputs:
    azureSubscription: $(azureServiceConnection)
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az aks get-credentials \
        --resource-group $(aksResourceGroup) \
        --name ${{ parameters.kubernetesCluster }} \
        --overwrite-existing
```

## Manual Rollback

To trigger a rollback, run the pipeline with:
```
triggerRollback: true
```

Or use Azure DevOps UI → Run Pipeline → Variables → Add `triggerRollback = true`

## Customization

### Add Custom Helm Values

Modify the HelmDeploy task arguments:
```yaml
arguments: >
  --set image.tag=${{ parameters.imageTag }}
  --set replicas=3
  --set ingress.enabled=true
  -f values-${{ parameters.environment }}.yaml
```

### Add Image Pull Secrets

Add a step before deployment:
```yaml
- task: Kubernetes@1
  displayName: "Create image pull secret"
  inputs:
    connectionType: 'Kubernetes Service Connection'
    kubernetesServiceEndpoint: $(kubernetesServiceConnection)
    namespace: ${{ parameters.namespace }}
    command: 'apply'
    secretType: 'dockerRegistry'
    secretName: 'acr-secret'
    dockerRegistryEndpoint: $(containerRegistryServiceConnection)
```

### Add ConfigMaps from Files

```yaml
- task: Kubernetes@1
  displayName: "Create ConfigMap"
  inputs:
    connectionType: 'Kubernetes Service Connection'
    kubernetesServiceEndpoint: $(kubernetesServiceConnection)
    namespace: ${{ parameters.namespace }}
    command: 'create'
    arguments: 'configmap myapp-config --from-file=config/ -o yaml --dry-run=client | kubectl apply -f -'
```

## Troubleshooting

### Common Issues

1. **kubectl not found**: Ensure KubectlInstaller task runs before kubectl commands
2. **Connection refused**: Verify service connection permissions and cluster accessibility
3. **Image pull errors**: Check container registry credentials and image tags
4. **RBAC errors**: Ensure service account has necessary cluster permissions

### Debug Mode

Add to your pipeline for verbose output:
```yaml
variables:
  system.debug: true
```

### Get Pod Logs

Add this step for debugging failed deployments:
```yaml
- bash: |
    kubectl logs -l app=${{ parameters.helmReleaseName }} -n ${{ parameters.namespace }} --tail=100
  displayName: "Get pod logs"
  condition: failed()
```

## Using Templates (Recommended)

The templated version (`az-pipeline-k8s-templated.yaml`) uses reusable templates for cleaner, DRY pipelines.

### Template: Install K8s Tools

**File**: `templates/install-k8s-tools.yaml`

Installs kubectl, Helm, and optionally Kustomize, kubeseal, k9s.

```yaml
steps:
  - template: templates/install-k8s-tools.yaml
    parameters:
      kubectlVersion: '1.31.0'
      helmVersion: '3.16.3'
      installAzureCLI: true       # Install Azure CLI (default: true)
      useAzureDevOpsTasks: true   # Use Azure DevOps tasks vs manual install
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kubectlVersion` | string | `1.31.0` | kubectl version |
| `helmVersion` | string | `3.16.3` | Helm version |
| `installAzureCLI` | boolean | `true` | Install Azure CLI |
| `useAzureDevOpsTasks` | boolean | `true` | Use Azure DevOps tasks or manual curl install |

### Template: Get K8s Credentials

**File**: `templates/get-k8s-credentials.yaml`

Gets credentials for connecting to Kubernetes clusters. Supports multiple providers.

```yaml
steps:
  # Option 1: Kubernetes Service Connection
  - template: templates/get-k8s-credentials.yaml
    parameters:
      connectionType: serviceConnection
      kubernetesServiceConnection: 'my-k8s-connection'
      namespace: 'myapp'

  # Option 2: Azure AKS
  - template: templates/get-k8s-credentials.yaml
    parameters:
      connectionType: aks
      azureServiceConnection: 'azure-subscription'
      aksResourceGroup: 'rg-aks-prod'
      aksClusterName: 'aks-prod-cluster'
      aksAdmin: false
      namespace: 'myapp'

  # Option 3: Kubeconfig variable
  - template: templates/get-k8s-credentials.yaml
    parameters:
      connectionType: kubeconfig
      kubeconfigVariable: 'KUBECONFIG_CONTENT'
      namespace: 'myapp'

  # Option 4: Google GKE
  - template: templates/get-k8s-credentials.yaml
    parameters:
      connectionType: gke
      gkeProject: 'my-gcp-project'
      gkeCluster: 'gke-cluster-name'
      gkeZone: 'us-central1-a'
      namespace: 'myapp'

  # Option 5: AWS EKS
  - template: templates/get-k8s-credentials.yaml
    parameters:
      connectionType: eks
      awsServiceConnection: 'aws-connection'
      eksClusterName: 'eks-cluster-name'
      awsRegion: 'us-east-1'
      namespace: 'myapp'
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `connectionType` | string | `serviceConnection` | Type: serviceConnection, aks, kubeconfig, gke, eks |
| `kubernetesServiceConnection` | string | | K8s service connection name |
| `azureServiceConnection` | string | | Azure service connection (for AKS) |
| `aksResourceGroup` | string | | AKS resource group |
| `aksClusterName` | string | | AKS cluster name |
| `aksAdmin` | boolean | `false` | Use admin credentials for AKS |
| `gcpServiceConnection` | string | | GCP service connection (for GKE) |
| `gkeProject` | string | | GCP project ID |
| `gkeCluster` | string | | GKE cluster name |
| `gkeZone` | string | | GKE zone |
| `awsServiceConnection` | string | | AWS service connection (for EKS) |
| `eksClusterName` | string | | EKS cluster name |
| `awsRegion` | string | `us-east-1` | AWS region |
| `kubeconfigVariable` | string | `KUBECONFIG_CONTENT` | Variable containing kubeconfig |
| `namespace` | string | `default` | Target namespace |
| `verifyConnection` | boolean | `true` | Verify cluster connectivity |

### Using Templates from Another Repository

Store templates in a shared repository for organization-wide reuse:

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: MyOrg/pipeline-templates
      ref: refs/heads/main

stages:
  - stage: Deploy
    jobs:
      - job: deploy
        steps:
          - template: k8s/install-tools.yaml@templates
            parameters:
              kubectlVersion: '1.31.0'
              
          - template: k8s/get-credentials.yaml@templates
            parameters:
              connectionType: aks
              aksClusterName: 'my-cluster'
```

## Related Resources

- [Azure DevOps Kubernetes Tasks](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/kubernetes)
- [Helm Deploy Task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/helm-deploy)
- [KubernetesManifest Task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/kubernetes-manifest)
- [Azure DevOps Templates](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates)
