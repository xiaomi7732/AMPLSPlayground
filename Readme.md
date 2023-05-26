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
        -diagServicesTrustedStorageAccessPrincipalId "$trustedStorageAppObjectId"
    ```

* Enable BYOS by update linked storage

    ```powershell
    New-AzResourceGroupDeployment `
        -ResourceGroupName $rgName `
        -TemplateUri https://raw.githubusercontent.com/xiaomi7732/AMPLSPlayground/main/deploy/LinkedStorage.jsonc `
        -insightsComponentName "$appInsightsComponentName" `
        -storageAccountName "$storageName"
    ```

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