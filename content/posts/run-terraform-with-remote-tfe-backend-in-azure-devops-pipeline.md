+++ 
draft = false
date = 2021-01-20T21:25:59+11:00
title = "Run Terraform with remote TFE backend in Azure DevOps Pipeline"
description = "Run Terraform with remote TFE backend in Azure DevOps Pipeline
slug = ""
authors = ["ringoc"]
tags = ["terraform","azure pipeline"]
categories = ["cloud"]
externalLink = ""
series = []
+++

# Run Terraform with remote TFE backend in Azure DevOps Pipeline

Background

Recently I worked in a project in large enterprise cloud migration project which using Azure DevOps Pipeline (YAML) as the core CD/CI pipeline which integrate Github repository as the source of truth, Terraform Enterprise workspace performing the IaC workload and Azure AD for IAM.

There are a number of benefits:

1. Cloud-native approach with Terraform with massive number of providers

1. Enjoy all benefits TFE offered e.g. collaboration, security, policy, and governance

1. TFE exclusive feature e.g. Sentinel and ServiceNOW integration

1. Encapsulate the complexity if TFE from Ops and Application teams

However, there are plenty of example in internet demostrating how to run Terraform with marketplace tasks with Azure Storage as the backend but not with TFE backend. I thought it would be good to share my work to others. Hopefully this would give others some insights.

## Challenges

If you have tried running Terraform code with [TFE remote backend](https://www.terraform.io/docs/backends/types/remote.html), there are a couple of thing needed to be kept in mind.

1. Running Terraform job with TFE remote backend, it packaged up all files in current folder and upload to TFE workspace.

1. Terraform workspace version SHOULD be the same Terrafrom runtime version in the build agent in Azure Pipeline

1. Hence, no runtime variables is supported

1. Use token from [CLI configuration files](https://www.terraform.io/docs/commands/cli-config.html)

## Solutions

With the above assumptions, the following steps are required.

1. Checkout Github source code

1. Retrieve TFE workspace Terraform version

1. Install custom Terraform runtime with version matching TFE workspace version

1. Generate Terraform CLI configuration file and backend config

1. Initialize Terraform with custom backend config

1. Apply Terraform with custom backend config

## Step 1 — Retrieve TFE workspace Terraform version

As the Azure DevOps pipeline built-in Terraform task is always keep up to date, this may not match the Terraform Enterprise workspace version. It is the best to use the same version of Terraform between the runtime and workspace. To retrieve the [Terraform workspace version](https://www.terraform.io/docs/cloud/api/workspaces.html), we can fire up a CURL command in BASH with correct HTTP header. The return JSON will then selected by jq . e.g.

    curl -sS --insecure \
             --header "Authorization: Bearer <tfeToken>" \ 
             --header "Content-Type: application/vnd.api+json" \    "https://<tfeHost>/api/v2/organizations/<tfeOrg>/workspaces/<tfeWorkspace>" 
    | jq -r '.data.attributes["terraform-version"]'

We stored the version in the local variable and then pass it to the next step.

## Step 2 — Install Terraform version matching workspace version

Once the version is found, call a TerraformInstall task to install custom Terraform runtime.

    **task**: TerraformInstaller@0     
    **displayName**: 'Install Terraform'     
    **inputs**:       
      **terraformVersion**: <tfVersion>

## Step 3 – Generate Terraform CLI config and backend config

Since running Terraform remote backend, there are a few limitations:

* Runtime variable is not supported

* Plan file output is not supported

We need to dynamically generate the CLI and backend config on the fly by replacing the TFE host, organization, workspace prefix in the backend config. and TFE host and token in the CLI config.

*.config/backend.hcl*

    hostname     = "__ptfeHost__"
    organization = "__ptfeOrg__"
    workspaces  {
      prefix = "__tla__-"
    }

*.config/credentials.tfrc.json*

    {
      "credentials": {
        "__ptfeHost__": {
          "token": "__ptfeToken__"
        }
      }
    }

*azure-pipeline.yaml*

    **bash**: |       
      cp ./tf/env/<env>.auto.tfvars ./tf    
      sed -i 's/__tfeHost__/<tfeHost>/' ./tf/.config/backend.hcl       
      sed -i 's/__tfeOrg__/<tfeOrg>/' ./tf/.config/backend.hcl       
      sed -i 's/__tfeWorkspace__/<tfeWorkspace>/' ./tf/.config/backend.hcl       
      sed -i 's/__tfeHost__/<tfeHost>/' ./tf/.config/credentials.tfrc.json         
      sed -i 's/__tfeToken__/<tfeToken>/'            ./tf/.config/credentials.tfrc.json     
    **displayName**: 'Create TF CLI Config File'     
    **env**:       
      **TFE_TOKEN**: <tfToken> *# passed as secret variables*

## Step 4 – Terraform init with backend config

Once CLI and backend config is generated. Run terraform init with custom backend config, and a couple of runtime variables.

    bash: |       
      DIRECTORY="./tf"       
      cd $DIRECTORY       
      TF_WORKSPACE=<tfeWorkspace> \
      TF_CLI_CONFIG_FILE=.config/credentials.tfrc.json \
      terraform init -backend-config=.config/backend.hcl     displayName: 'TFE Init'\

## Step 5 – Terraform apply with correct CLI and backend config

Same for applying the terraform

    bash: |       
      DIRECTORY="./tf"       
      cd $DIRECTORY       
      TF_WORKSPACE=<tfeWorkspace> \    
      TF_CLI_CONFIG_FILE=.config/credentials.tfrc.json \
      terraform plan 
      TF_WORKSPACE=<tfeWorkspace> \
      TF_CLI_CONFIG_FILE=./.config/credentials.tfrc.json \
      terraform show -json > output-state.json       
      cat output-state.json | jq '.' | less     
    displayName: 'TFE Apply'

## Step 6 – Archive the plan as an artifact

As a requirement for change management, the plan/apply output are exported and packaged up as a GZIP file.

    task: ArchiveFiles@2     
      inputs:       
        rootFolderOrFile: '$(Build.SourcesDirectory)'            
        includeRootFolder: false       
        archiveType: 'tar'       
        tarCompression: 'gz'       
        archiveFile: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).tfplan.tgz'       
        replaceExistingArchive: true       
    displayName: 'Create Plan Artifact'

## Step 7 – Publish to the artifact container

Once the GZIP is created, publish to Azure DevOps artifact

    task: PublishBuildArtifacts@1     
      inputs:       
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'       
        ArtifactName: '$(Build.BuildId).tf.enterprise.plan'           
        publishLocation: 'Container'       
    displayName: 'Publish Plan Artifact'

## Summary

Now if you run the pipeline, it should go through tall the above steps. And the TFE workspace should be triggered by the pipeline task. However the workflow is infrastructure lead will review the plan in TFE workspace and confirm the plan by clicking on the Confirm button. Also, any change management can be referenced by TFE workspace history and the Azure artifact stored in Azure Pipeline.  

