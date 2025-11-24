Azure DevOps Pipeline – Comprehensive Reusable Template

This repository contains a multi-stage Azure DevOps YAML pipeline designed as a reusable baseline for most CI/CD scenarios.
It supports:
	•	CI triggers (branch + path filters)
	•	PR validation
	•	Scheduled (nightly) runs
	•	Multi-repo checkout
	•	Matrix builds
	•	Caching
	•	Artifact publishing & consumption
	•	Conditional stages (tests/infra)
	•	Azure infrastructure deployment using AzureCLI@2
	•	Environment-based deployments
	•	Reusable templates

⸻

Pipeline File

Main pipeline is defined in:

pipeline.yml

Environment-specific variables are loaded from:

variables/
  vars-dev.yaml
  vars-acc.yaml
  vars-prod.yaml


⸻

Stages Overview

1. Prep

Goal: checkout repos, print working dirs, set output variables.
	•	Checks out:
	•	self repo
	•	infraRepo
	•	templatesRepo
	•	Logs directory roots for debugging.
	•	Exports output variables (example: git SHA).

2. Build

Goal: build your application.
	•	Runs as a matrix job (example for future parallel OS builds).
	•	Restores/creates cache (example uses npm).
	•	Publishes artifacts named drop.

3. Test (optional)

Goal: run unit tests.
	•	Only runs if runTests=true.

4. Infra

Goal: deploy infrastructure (Bicep, Terraform, ARM, etc.).
	•	Uses a deployment job
	•	Runs against Azure DevOps environment
	•	Performs sanity check (az group list)
	•	Executes infra deployment (example uses Bicep)

5. App

Goal: deploy application using published artifacts.
	•	Downloads the drop artifact from Build stage.
	•	Runs your deploy logic.

6. Post

Goal: always-run finalization.
	•	Runs even if previous stages fail (condition: always()).
	•	Can send notifications, create work items, cleanup, etc.

⸻

Parameters

Pipeline supports compile-time parameters:

Parameter	Type	Default	Description
environment	string	dev	Target environment (dev, acc, prod)
runInfra	boolean	true	Toggle infra stage
runTests	boolean	true	Toggle test stage
extraArgs	string	""	Extra CLI args passed into infra deploy

Example manual run:

environment: acc
runInfra: true
runTests: false
extraArgs: "--what-if"


⸻

Variables & Templates

Inline variables

Defined inside the YAML:

variables:
  vmImage: ubuntu-latest
  buildConfig: Release

Variable group

Loaded from Azure DevOps Library:

- group: shared-secrets

Typical use: store secrets like subscription IDs, service principals, tokens, etc.

Environment templates

Loaded per environment:

- template: variables/vars-${{ parameters.environment }}.yaml

Expected variables inside each env file:

azureServiceConnection: "spoke-robin-dev"
adoEnvironmentName: "robin-dev"
location: "westeurope"


⸻

Multi-Repo Checkout

This pipeline checks out multiple repos:

resources:
  repositories:
  - repository: infraRepo
    name: MyProject/infra-repo
  - repository: templatesRepo
    name: MyProject/pipeline-templates

Checkout usage:

- checkout: infraRepo
  path: shared-infra

✅ Important deployment-job path rule
In a deployment: job, extra repos are located under:

$(Pipeline.Workspace)/<path>

Use these variables safely:

Variable	Meaning
$(System.DefaultWorkingDirectory)	Root of self repo
$(Pipeline.Workspace)	Root for all repos + artifacts
$(Build.SourcesDirectory)	Self repo root (mostly same as DefaultWorkingDirectory)

Example:

SELF_ROOT="$(System.DefaultWorkingDirectory)"
INFRA_ROOT="$(Pipeline.Workspace)/shared-infra"
TEMPLATES_ROOT="$(Pipeline.Workspace)/templates"


⸻

Azure CLI Tasks

Infra stage uses AzureCLI@2:

- task: AzureCLI@2
  inputs:
    azureSubscription: $(azureServiceConnection)
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az group list -o table

Make sure your Service Connection has the right RBAC role on the subscription.

⸻

Artifacts

Build publishes an artifact named drop:

- publish: $(System.DefaultWorkingDirectory)/dist
  artifact: drop

App stage downloads it:

- download: current
  artifact: drop

The artifact ends up in:

$(Pipeline.Workspace)/drop


⸻

How to Use This Template in a New Project
	1.	Copy pipeline.yml into your repo
	2.	Update:
	•	resources.repositories names
	•	variable templates in /variables
	•	Azure Service Connection name
	•	build/test/deploy scripts
	3.	Run pipeline manually once for validation
	4.	Gradually remove unused stages

⸻

Troubleshooting Tips

Print all useful paths

Add this step when debugging:

echo "Build.SourcesDirectory=$(Build.SourcesDirectory)"
echo "System.DefaultWorkingDirectory=$(System.DefaultWorkingDirectory)"
echo "Pipeline.Workspace=$(Pipeline.Workspace)"
pwd
find "$(Pipeline.Workspace)" -maxdepth 2 -type d

Common errors

Error	Cause	Fix
unexpected value stage	stage used without stages:	wrap top-level in stages:
repo not found under /s/	deployment jobs store extra repos elsewhere	use $(Pipeline.Workspace)
file not found	variable points to self repo but file is in fork repo	correct variable paths


⸻

License / Usage

This pipeline template is intended for internal reuse.
Fork and adapt per application or infra stack.

⸻

If you want, paste your real repo names + folder layout and I’ll produce a “final” README version that matches your exact project (with your Bicep/Terraform paths + naming conventions).
