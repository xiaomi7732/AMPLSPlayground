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
New-AzResourceGroupDeployment `
    -VNetName $vnetName `
    -location $location
```