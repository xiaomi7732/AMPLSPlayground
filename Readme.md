# Deploy resources for AMPLS Playground

## Create a target resource group (Azure PowerShell)

1. Login

```powershell
Connect-AzAccount
```

1. Pick your subscription. For example, `saars - Microsoft Azure Internal Consumption`

```powershell
$subscriptionName='saars - Microsoft Azure Internal Consumption'
Set-AzContext -Subscription $subscriptionName
```

1. Create a resource group to deploy resources. For example: "saars-ampls-playground-8-eastus"

```powershell
$location='eastus'
$rgName="saars-ampls-playground-8-$location"
New-AzResourceGroup -Name "$rgName" -Location "$location"
```

## Deploy VNet

```powershell
$vnetName="AppVNet1"
$appServiceSubnetName="appservice"
$defaultSubnetName="default"

New-AzResourceGroupDeployment `
    -ResourceGroupName $rgName `
    -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/VNet.jsonc `
    -VNetName $vnetName `
    -location $location `
    -appServiceSubnetName "$appServiceSubnetName" `
    -defaultSubnetName "$defaultSubnetName"
```

## Deploy Application Insights

```powershell
$logAnalyticsName="saars-ampls-pg8-la-$location"
$appInsightsComponentName="saars-ampls-pg8-ai-$location"

New-AzResourceGroupDeployment `
    -ResourceGroupName $rgName `
    -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/AppInsights.jsonc `
    -workspaceName $logAnalyticsName `
    -location $location `
    -appInsightsName $appInsightsComponentName
```

## Deploy a storage for BYOS

* Deploy the storage account with Role assignment

    ```powershell
    $trustedStorageAppObjectId = (Get-AzADServicePrincipal -DisplayName "Diagnostic Services Trusted Storage Access").id
    $storageName="saarsamplspg8sa$location"

    New-AzResourceGroupDeployment `
        -ResourceGroupName $rgName `
        -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/Storage.jsonc `
        -nameFromTemplate "$storageName" `
        -location "$location" `
        -diagServicesTrustedStorageAccessPrincipalId "$trustedStorageAppObjectId" `
        -vnetName "$vnetName" `
        -appServiceSubnetName "$appServiceSubnetName" `
        -defaultSubnetName "$defaultSubnetName"
    ```

* Enable BYOS by update linked storage

    ```powershell
    New-AzResourceGroupDeployment `
        -ResourceGroupName $rgName `
        -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/LinkedStorage.jsonc `
        -insightsComponentName "$appInsightsComponentName" `
        -storageAccountName "$storageName"
    ```

## App Service

* Deploy App Service with VNet integration

    ```powershell
    $appServiceName="saars-ampls-pg8-app-$location"

    New-AzResourceGroupDeployment `
        -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/AppService.jsonc `
        -ResourceGroupName $rgName `
        -location "$location" `
        -appServiceName "$appServiceName" `
        -vnetName "$vnetName" `
        -vnetIntegrationSubnetName "$appServiceSubnetName" `
        -privateLinkEndpointSubnetName "$defaultSubnetName"
    ```

## AMPLS

* Deploy AMPLS to link to the component and the workspace

    ```powershell
    $amplsName="saars-ampls-pg8-scope"

    New-AzResourceGroupDeployment `
        -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/AMPLS.jsonc `
        -ResourceGroupName "$rgName" `
        -amplsName "$amplsName" `
        -workspaceName "$logAnalyticsName" `
        -aiComponentName "$appInsightsComponentName"
    ```

* Deploy Private Link Endpoint for the AMPLS

    ```powershell
    $privateLinkEndpointName="saars-ampls-pg8-scope-eastus-ple"
    
    New-AzResourceGroupDeployment `
        -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/AMPLSPLEndpoint.jsonc `
        -ResourceGroupName "$rgName" `
        -location "$location" `
        -privateEndpointName "$privateLinkEndpointName" `
        -amplsScopeName "$amplsName" `
        -vnetName "$vnetName" `
        -plSubnetName "$defaultSubnetName"
    ```

## Notes

* Some manual steps currently needed
    * Somehow, the linked storage was overwritten in the application insights component. Add it back by running `Enable BYOS by update linked storage` step again.
        * To check the link, use resource explorer: https://resources.azure.com/
    * By default, the access mode for AMPLS is Open/Open. Change to:
        * Ingestion access mode: Private Only
        * Query access mode: Open
    * By default, for application insights resource, ingestion is allowed from public network. Change it to Private Link only:
        * Application Insights => Network Isolation => Virtual networks access configuration => Accept data ingestion from public networks not connected through a Private Link Scope => false

    * Permission to pull from ACR to the app service is missing.
        * Add a managed identity
        * Add role assignment to allow acrpull
        * Assign the identity to the app service deployment center and restart the app service

    * Add an availability test to the web app for continuous traffic

    * App service logging is by default disabled. Enable it from the portal

    * If app service stopped accepting public traffic, it could then be invoked through the private link endpoint in the subnet of `default`, by the jump box. For example:
        * watch -n 10 curl https://appname.azurewebsites.net


* Only if you want full private traffic, deploy Private Link Endpoint for BYOS storage

    ⚠️ DNS Zone for blob storage is reused from AMPLS configurations.

    ```powershell
    $storageEndpointName="saars-ampls-pg8-storage-eastus-ple"

    New-AzResourceGroupDeployment `
        -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/StorageEndpoint.jsonc `
        -ResourceGroupName "$rgName" `
        -location "$location" `
        -privateEndpointName "$storageEndpointName" `
        -storageAccountName "$storageName" `
        -vnetName "$vnetName" `
        -subnetName "$defaultSubnetName"
    ```