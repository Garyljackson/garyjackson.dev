---
title: "Accessing Azure Container Instances deployed within a Azure Virtual Network"
summary: "Accessing Azure Container Instances deployed within a Azure Virtual Network"
date: "2021-09-29T17:17:00+10:00"
lastmod: "2021-09-29T17:17:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Azure Container Instances"
  - "YARP"
  - "Azure vNet"
  - ".NET 6"
  - "ASP.NET 6"
  - "ASP.NET Minimal API"
---

## Overview

How to access an Azure Container Instance deployed within a vNet, from a public endpoint.

## Details

Azure Container Instances are a great way to fire up container-based applications with minimal effort.

Recently we decided to use ACI as a means to reduce our reliance on Azure Virtual Machines.

Besides moving away from infrastructure services, we had the following requirements.

- The container instance needs to be accessible from services via an azure virtual network.
- For any public interface like an admin UI, we need to have SSL.
- We need to be able to limit access using IP filtering.

### Blockers

During my initial discovery and proof of concept phase, I discovered that Azure Container Instances deployed to a virtual network could not be accessed using a public IP address using purely native ACI.

The ACI documentation does, however, suggest that public access can be achieved with some additional components, namely, Azure Firewall or Application Gateway.

I explored these options further, and my thoughts were

- Azure Firewall
  - Expensive (+- $1200/month)
- Azure Application Gateway
  - Have had mixed reliability results with AG before
  - Does not support platform managed SSL certificates
  - Not user friendly

I also explored the idea of using an Azure Point-to-site VPN, but that created more friction for anyone needing to access the service.

If you're not aware, Azure Function Apps natively support the ability to act as a proxy, but some other specifics about our solution made that inappropriate.

Our initial decision was to bite the bullet and go with the App gateway option, but be conscious that there is the additional responsibility to manage certificates.

### Huzzah

Fortunately, later that evening, it dawned on me that we could use Microsoft YARP (Yet another reverse proxy) within an ASP.NET core application deployed to an Azure Web Application.

Using this solution, we can leverage the existing App Service functionality to meet our requirements above.

Specifically:

- App Service vNet integration
- Managed custom domain certificates
- App service IP restrictions

Honestly, this worked out great; I'm thrilled with how it all turned out.

Here's how you can also implement this.

### Solution

#### Components

This solution example is deployed using the Azure CLI and is comprised of the following Azure resources.

- 1 x Azure Resource Group
- 1 x Virtual Network
- 2 x Subnets (You can't mix resource types inside the subnet)
  - 1 x subnet for ACI
  - 1 x subnet for the Web App vNet integration
- 1 x Azure Container Instance App
  - Deployed to the ACI subnet within the vNet
  - Running the Microsoft `aci-helloworld` demo application
- 1 x App service plan - this needs to be at least the `S1` sku to support vNet integration
- 1 x App service
- 1 x ASP.NET Core 6 `(rc1)` app
  - Implemented using the minimal API
  - YARP installed and configured
  - The custom app then deploys from my GitHub repo

#### Deployment

To deploy this solution you will need to do the following steps

- Open Powershell
- Login with the Azure CLI `az login`
- Copy the script below
- Update `$ProxyWebAppName` - this is the web app name and needs to be globally unique within Azure
- Execute the script
- The final CLI command will output the details for the proxy web application for you to use in a browser.

```powershell
$Location = 'australiaeast'
$ResourceGroupName = 'MyExampleResourceGroup'

$VNetName = 'MyExampleVNet'
$VNetCIDR = '10.0.0.0/16'

$DefaultSubnetName = 'Default'
$DefaultSubnetCIDR = '10.0.1.0/24'

$AciSubnetName = 'ExampleAciSubnet'
$AciSubnetCIDR = '10.0.2.0/24'

$WebAppIntegrationSubnetName = 'vNetIntegrationSubnet'
$WebAppIntegrationSubnet = '10.0.3.0/24'

$AppServicePlanName = 'ExampleAsp'
$ProxyWebAppName = 'ExampleAciProxy12345'

az group create --name $ResourceGroupName --location $Location

# Create the virtual network
az network vnet create `
    --name $VNetName `
    --resource-group $ResourceGroupName `
    --location $Location `
    --address-prefix $VNetCIDR `
    --subnet-name $DefaultSubnetName `
    --subnet-prefix $DefaultSubnetCIDR

# Create 2 subnets
# You can't mix resource types in a single resource group due to the delegations on each subnet
az network vnet subnet create `
    --name $AciSubnetName `
    --resource-group $ResourceGroupName `
    --vnet-name $VNetName   `
    --address-prefix $AciSubnetCIDR

az network vnet subnet create `
    --resource-group $ResourceGroupName `
    --vnet-name $vNetName `
    --name $WebAppIntegrationSubnetName `
    --address-prefixes $WebAppIntegrationSubnet `
    --delegations 'Microsoft.Web/serverFarms'

# Deploy the ACI instance to the virtual network
az container create `
    --name appcontainer `
    --resource-group $ResourceGroupName `
    --image mcr.microsoft.com/azuredocs/aci-helloworld `
    --vnet $VNetName `
    --subnet $AciSubnetName

# Create the proxy web app resources
# Needs to be S1 to support vNet integration
az appservice plan create `
    --resource-group $ResourceGroupName `
    --name $AppServicePlanName `
    --is-linux `
    --sku S1

az webapp create `
    --name $ProxyWebAppName `
    --resource-group $ResourceGroupName `
    --plan $AppServicePlanName `
    --runtime '"DOTNET|6.0"'

# Get the ip address for the ACI instance we deployed - this is used in the proxy web app settings
$AciIp = $(az container show `
        --name appcontainer `
        --resource-group $ResourceGroupName `
        --query ipAddress.ip --output tsv)

## Double underscores are needed for nested json settings in Azure
az webapp config appsettings set `
    --resource-group $ResourceGroupName `
    --name $ProxyWebAppName `
    --settings ReverseProxy__Clusters__MinimumCluster__Destinations__MyBackend__Address="http://$AciIp"

# Setup the vNet integration for the web app to its subnet
az webapp vnet-integration add `
    --resource-group $ResourceGroupName `
    --name $ProxyWebAppName `
    --vnet $VNetName `
    --subnet $WebAppIntegrationSubnetName

# Link the YARP proxy application from the GitHub repo
az webapp deployment source config `
    --branch master `
    --manual-integration `
    --name $ProxyWebAppName `
    --repo-url https://github.com/Garyljackson/vNetAciYarpProxyExample/ `
    --resource-group $ResourceGroupName

# Deploy the source code to the web app
az webapp deployment source sync `
    --name $ProxyWebAppName `
    --resource-group $ResourceGroupName

# Display the details to access the proxy application
az webapp config hostname list `
    --resource-group $ResourceGroupName `
    --webapp-name $ProxyWebAppName
```

## Related Links

[Example Solution GitHub Repo](https://github.com/Garyljackson/vNetAciYarpProxyExample/)

[The ACI Hello world docker app](https://hub.docker.com/_/microsoft-azuredocs-aci-helloworld)

[Azure Container Instance - Unsupported Networking Scenarios](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-virtual-network-concepts#unsupported-networking-scenarios)

[Azure Container Instance - Application Gateway](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-application-gateway)

[Azure Container Instance - Azure Firewall](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-egress-ip-address)

[Azure Function App - Proxy](https://docs.microsoft.com/en-us/azure/azure-functions/functions-proxies)
