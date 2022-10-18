---
layout: single
title:  "Deploying Logic Apps (Preview) using Azure DevOps and GitHub Actions"
date:   2021-02-21 11:00:00 +0000
permalink: /functions-based-logic-apps-ci-cd
---

Single-tenant Logic Apps are now Generally Available!
Refer to Microsoft's [Set up DevOps deployment for single-tenant Azure Logic Apps](https://docs.microsoft.com/en-us/azure/logic-apps/set-up-devops-deployment-single-tenant-azure-logic-apps?tabs=github) documentation for instructions on deploying them using Azure DevOps / Github.
{: .notice--warning}

Last September the Azure Logic Apps team has [unveiled](https://techcommunity.microsoft.com/t5/azure-developer-community-blog/new-logic-apps-runtime-performance-and-developer-improvements/ba-p/1645335) a new Logic Apps runtime which is built as an extension of Azure Functions runtime, this means you can now run Logic Apps everywhere you can run Function Apps including App Service Plans, Functions Premium plan, and locally on your device. Due to runtime changes the process for building and deploying Logic Apps changes from a fully ARM based approach to one almost identical to building and deploying a Function App. 

You can the Logic Apps (Preview) documentation here: [Overview: Azure Logic Apps Preview - Microsoft Docs](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-overview-preview).

**⚠:**
As this runtime is preview it is not supported for Production workloads and includes some bugs and limitations. Make sure to familiarise yourself with [known issues](https://github.com/Azure/logicapps/blob/master/articles/logic-apps-public-preview-known-issues.md) before getting started.

All templates used in this guide can be found at [maciejporebski/function-based-logic-apps](https://github.com/maciejporebski/function-based-logic-apps).

# Developing

To get started with Logic Apps (Preview) download the Visual Studio Code extension [Azure Logic Apps (Preview)](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurelogicapps). The extension provides ability to scaffold projects and workflows. You'll notice the generated project is very similar to a Functions App project. 

Each workflow should be defined in a `workflow.json` file placed in a folder which name will determine the workflow name. The workflow folders must be copied to the output directory. If you generate a new workflow using the "Create Workflow..." feature of Logic Apps (Preview) extension the directory will be added to the .csproj, however if you are creating workflows manually or decided to rename a workflow directory make sure to update your .csproj file:
```xml
<ItemGroup>
    <None Update="CurrentTime\**\*.*">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
</ItemGroup>
```

# Building

The build process is exactly the same as for a .NET Core Azure Functions project. You will want to restore nuget packages, build the project and publish it using the dotnet CLI.
```bash
dotnet restore
dotnet build
dotnet publish
```

## Azure Pipelines

In Azure Pipelines use the `DotNetCoreCLI` task to restore, build, and publish the package.

```yml
- task: DotNetCoreCLI@2
  displayName: restore
  inputs:
    command: restore
    projects: '**/*.csproj'
- task: DotNetCoreCLI@2
  displayName: build
  inputs:
    projects: '**/*.csproj'
    arguments: '--configuration Release'
- task: DotNetCoreCLI@2
  displayName: publish
  inputs:
    command: publish
    publishWebProjects: false
    projects: '**/*.csproj'
    arguments: '--configuration Release --output $(build.artifactstagingdirectory)'
    zipAfterPublish: True
```
[.azure-devops/azure-pipelines.yml](https://github.com/maciejporebski/function-based-logic-apps/blob/main/.azure-devops/azure-pipelines.yml)

## GitHub Actions

In GitHub actions use the `actions/setup-dotnet` action to set up the SDK (currently the latest version supported by Azure Functions runtime is 3.1), then run the restore, build and publish commands.

```yml
- name: Setup .NET Core SDK
  uses: actions/setup-dotnet@v1.7.2
  with:
    dotnet-version: 3.1
- name: .NET Restore
  run: dotnet restore $GITHUB_WORKSPACE/src/src.csproj
- name: .NET Build
  run: dotnet build $GITHUB_WORKSPACE/src/src.csproj --configuration Release
- name: .NET Publish
  run: dotnet publish $GITHUB_WORKSPACE/src/src.csproj --configuration Release --output $GITHUB_WORKSPACE/output
- uses: actions/upload-artifact@v2
  with:
    name: logic-app-package
    path: output
```
[.github/workflows/deploy-logic-app.yml](https://github.com/maciejporebski/function-based-logic-apps/blob/main/.github/workflows/deploy-logic-app.yml)

# Building infrastructure

As previously mentioned Logic Apps (Preview) run on Azure Functions, this becomes obvious when deploying the Logic Apps (Preview) resource, which is a `Microsoft.Web/sites` resource, with few minor differences compared to a Functions App Resource. The properties requiring Logic Apps (Preview) specific values are:
- kind
- properties.siteConfig.cors.allowedOrigins
- properties.siteConfig.appSettings

The required values are:
```json
{
    "type": "Microsoft.Web/sites",
    "apiVersion": "2020-09-01",
    "name": "[parameters('logicAppName')]",
    "location": "[parameters('location')]",
    "kind": "functionapp,workflowapp",
    "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('appServicePlanName'))]",
        "siteConfig": {
            "cors": {
                "allowedOrigins": [
                    "https://afd.hosting.portal.azure.net",
                    "https://afd.hosting-ms.portal.azure.net",
                    "https://hosting.portal.azure.net",
                    "https://ms.hosting.portal.azure.net",
                    "https://ema.hosting.portal.azure.net"
                ]
            },
            "appSettings": [
                {
                    "name": "APP_KIND",
                    "value": "workflowApp"
                },
                {
                    "name": "AzureFunctionsJobHost__extensionBundle__id",
                    "value": "Microsoft.Azure.Functions.ExtensionBundle.Workflows"
                },
                {
                    "name": "AzureFunctionsJobHost__extensionBundle__version",
                    "value": "[1.*, 2.0.0)"
                },
                {
                    "name": "AzureWebJobsStorage",
                    "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', parameters('storageAccountName'), AccountKey=', listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName')'2019-06-01').keys[0].value)]"
                },
                {
                    "name": "FUNCTIONS_EXTENSION_VERSION",
                    "value": "~3"
                },
                {
                    "name": "FUNCTIONS_V2_COMPATIBILITY_MODE",
                    "value": "true"
                },
                {
                    "name": "FUNCTIONS_WORKER_RUNTIME",
                    "value": "dotnet"
                }
            ]
        }
    },
    "dependsOn": [...]
}
```
[function-based-logic-apps/infrastructure/arm-template.json](https://github.com/maciejporebski/function-based-logic-apps/blob/main/infrastructure/arm-template.json)

Aside from the Logic Apps (Preview) resource you will need to deploy or reference an existing hosting App Service Plan and Storage Account. It is also a good idea to deploy an Application Insights resource to monitor your Logic App. All of these resources are deployed by the full template: [arm-template.json](https://github.com/maciejporebski/function-based-logic-apps/blob/main/infrastructure/arm-template.json)

# Deploying

The deployment process consists of two parts:
- deploying the infrastructure
- deploying the built package to the Logic Apps (Preview) resource

## Azure Pipelines

In Azure Pipelines use the `AzureResourceManagerTemplateDeployment` task to deploy the ARM template created in the 'Building infrastructure' section. Then use the 'AzureRmWebAppDeployment' task to deploy the built package to the Logic Apps (Preview) resource. Set the 'appType' property to 'functionApp' as currently Logic Apps (Preview) does not have a dedicated appType value.

```yml
- task : AzureResourceManagerTemplateDeployment@3
  displayName: deploy_arm
  inputs:
    azureResourceManagerConnection: $(serviceConnection)
    resourceGroupName: '$(resourceGroup)'
    location: '$(location)'
    csmFile: '$(pipeline.workspace)/drop/arm-template.json'
    overrideParameters: '-logicAppName $(logicAppName) -storageAccountName $(storageAccountName) -appServicePlanName (appServicePlanName) -insightsName $(insightsName) -location $(location)'
- task: AzureRmWebAppDeployment@4
  displayName: deploy_logic_app
  inputs:
    ConnectionType: 'AzureRM'
    azureSubscription: $(serviceConnection)
    appType: 'functionApp'
    WebAppName: '$(logicAppName)'
    packageForLinux: '$(pipeline.workspace)/drop/src.zip'
```
[.azure-devops/azure-pipelines.yml](https://github.com/maciejporebski/function-based-logic-apps/blob/main/.azure-devops/azure-pipelines.yml)

## GitHub Actions

In GitHub Actions sign in to Azure by using the [`azure/login`](https://github.com/marketplace/actions/azure-login) action (you must configure service principal details in your repo secrets first), use the `Azure/arm-deploy` action to deploy the infrastructure, then finally use the `Azure/functions-action` action to deploy the package.

<!-- {% raw %} -->
```yml
- uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}  
- name: Deploy Infrastructure
  uses: Azure/arm-deploy@v1  
  with:
    scope: resourcegroup
    subscriptionId: ${{ secrets.SUBSCRIPTION_ID }}
    resourceGroupName: ${{ env.RESOURCE_GROUP }}
    template: infrastructure/arm-template.json
    deploymentMode: Incremental
    parameters: logicAppName=${{ env.LOGIC_APP_NAME }} storageAccountName=${{ env.STORAGE_ACCOUNT_NAME }} appServicePlanName=${{ env.ASP_NAME }} insightsName=${{ env.INSIGHTS_NAME }} location=${{ env.LOCATION }}  
- name: Deploy Logic App
  uses: Azure/functions-action@v1.3.1
  with:
    app-name: ${{ env.LOGIC_APP_NAME }}
    package: logic-app-package
```
<!-- {% endraw %} -->
[.github/workflows/deploy-logic-app.yml](https://github.com/maciejporebski/function-based-logic-apps/blob/main/.github/workflows/deploy-logic-app.yml)