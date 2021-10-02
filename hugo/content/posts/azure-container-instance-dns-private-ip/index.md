---
title: "Azure Container Instance - Private IP management using DNS"
summary: "Solving ACI dynamic IP assignment using a private DNS and an Azure CLI sidecar"
date: "2021-10-02T08:00:00+10:00"
lastmod: "2021-10-02T08:00:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Azure Container Instances"
  - "Azure Container Groups"
  - "Azure Private DNS"
  - "Azure CLI"
---

## Overview

My recent adventures with Azure Container Registry have presented a few side quests along the way.

Today I'll cover the limitations I've discovered with regards to private DNS and IP allocations.

## Context

When deploying Azure Container instances within a vNet, there are some limitations to be aware of.

### Static IP

You can't assign a static private IP address to your container instance.

There are a few reasons why the IP address might change, including manually restarting the app or even Microsoft performing platform upgrades or maintenance

### Private DNS

Unlike Azure VM's, container instances don't automatically register and synchronise the IP address with the associated DNS record.

### Managed Identity

Managed identity is not currently available for ACI in a private vNet.

## Solution

Given that I need to deploy ACI to a vNet, with a private IP address, I needed to work around these limitations.

### Components

#### Private DNS

I will deploy a private DNS zone into the virtual network.

Within the private zone, I create an A-Record pointing to the IP address of the container instance.

The A-record provides a stable communication location for services that need access to the container instance endpoint.

#### Container Group + Azure CLI Sidecar

Containers running within a group share the same lifecycle.
Alongside the regular service, I also deploy a custom image that inherits from the Azure CLI Docker image.

The custom image lets me run some Azure CLI commands when the sidecar container starts up.

Specifically, the Azure CLI commands to discover the IP address of the container instance and then update the DNS A-Record.

The sidecar image will be uploaded to an Azure Container Registry.

#### Service Principal

There are a few alternatives to a managed identity, in this case I'm using a service principle.

## How to

Alright, let's jump into the actual code.

{{< admonition type=note title="Note" open=true >}}

You might notice that I only deploy the sidecar in the following example.  
Azure CLI doesn't support multi-container deployments using the parameters directly.  
You will need to define an ARM template or similar to define the multi-container group.

{{< /admonition >}}

{{< admonition type=note title="Note" open=true >}}

The example deploy script provided below pulls the code for the Dockerfile, and the Azure CLI DNS update script from my Github repo

{{< /admonition >}}

### The sidecar

#### Dockerfile definition

```dockerfile
FROM mcr.microsoft.com/azure-cli AS base

WORKDIR /app

COPY az_cli.sh .

# Creates a non-root user with an explicit UID
RUN adduser -u 5678 --disabled-password --gecos "" appuser

# Assign the new user as the owner of the app folder
RUN chown -R appuser /app

# Give the new user execute permissions on the file
RUN chmod u+x az_cli.sh

USER appuser

CMD [ "./az_cli.sh" ]
```

#### Azure DNS update script

I've chosen to use environment variables for the parameters

```bash
#!/bin/bash

az --version

az login --service-principal -u $APP_ID -p $APP_PASSWORD --tenant $APP_TENANT_ID

ACI_IP=$(az container show --name $ACI_INSTANCE_NAME --resource-group $RESOURCE_GROUP --query ipAddress.ip --output tsv)

az network private-dns record-set a update --name $A_RECORD_NAME --resource-group $RESOURCE_GROUP --zone-name $DNS_ZONE_NAME --set aRecords[0].ipv4Address=$ACI_IP

echo "Done"

```

### Example deployment

Open the [Azure Shell](https://shell.azure.com/) and run the following

```bash
resource_suffix=$RANDOM
location=australiaeast
resource_group_name=ExampleResourceGroup$resource_suffix
vnet_name=ExampleVNet$resource_suffix
vnet_cidr=10.0.0.0/16
default_subnet_name=Default
default_subnet_cidr=10.0.1.0/24
aci_subnet_name=ExampleAciSubnet$resource_suffix
aci_subnet_cidr=10.0.2.0/24
aci_container_name=example-container-$resource_suffix
aci_sidecar_image_name=example-sidecar-$resource_suffix
acr_name=ExampleAcr$resource_suffix
dns_zone=private.internal$resource_suffix
dns_a_record=example$resource_suffix
ad_rbac_service_principal=AciAzureCliServicePrincipal$resource_suffix

# Create the resource group
az group create --name $resource_group_name --location $location

# Create a service principle
service_principal=$(az ad sp create-for-rbac \
    --name $ad_rbac_service_principal \
    --role "Private DNS Zone Contributor")

# Extract the details
service_principal_app_id="$(echo $service_principal | jq -r '.appId')"
service_principal_password="$(echo $service_principal | jq -r '.password')"
service_principal_tenant="$(echo $service_principal | jq -r '.tenant')"

# assign the reader role so that it can read the container instance ip address
az role assignment create --assignee $service_principal_app_id --role "Reader"

# Create the virtual network
az network vnet create \
    --name $vnet_name \
    --resource-group $resource_group_name \
    --location $location \
    --address-prefix $vnet_cidr \
    --subnet-name $default_subnet_name \
    --subnet-prefix $default_subnet_cidr

# Create the ACI subnet
az network vnet subnet create \
    --name $aci_subnet_name \
    --resource-group $resource_group_name \
    --vnet-name $vnet_name \
    --address-prefix $aci_subnet_cidr

# Create an azure container registry
az acr create \
    --resource-group $resource_group_name \
    --name $acr_name \
    --sku Basic

# Get the login server for the ACR instance
acr_login_server=$(az acr show \
    --name $acr_name \
    --query loginServer \
    --output tsv)

# Enable admin before you can retrieve the password
az acr update -n $acr_name --admin-enabled true

# Retrieve the credentials for the container registry
acr_credentials=$(az acr credential show --name $acr_name)
acr_username="$(echo $acr_credentials | jq -r '.username')"
acr_password="$(echo $acr_credentials | jq -r '.passwords[0].value')"

# Deploy the image to ACR
az acr build \
    --registry $acr_name \
    --image examples/$aci_sidecar_image_name:latest \
    https://github.com/Garyljackson/PrivateAciDnsUpdate.git

# Create a private dns
az network private-dns zone create \
    --resource-group $resource_group_name \
    --name $dns_zone

# Link the private dns to the virtual network
az network private-dns link vnet create \
    --resource-group $resource_group_name \
    --name $resource_group_name \
    --zone-name $dns_zone \
    --virtual-network $vnet_name \
    --registration-enabled false

# Add an initial placeholder A record
az network private-dns record-set a add-record \
    --record-set-name $dns_a_record \
    --resource-group $resource_group_name \
    --zone-name $dns_zone \
    --ipv4-address 0.0.0.0

# Deploy the ACI instance to the virtual network
az container create \
    --name $aci_container_name \
    --resource-group $resource_group_name \
    --image $acr_login_server/examples/$aci_sidecar_image_name:latest \
    --vnet $vnet_name \
    --subnet $aci_subnet_name \
    --registry-login-server $acr_login_server \
    --registry-username $acr_username \
    --registry-password $acr_password \
    --environment-variables \
    ACI_INSTANCE_NAME=$aci_container_name \
    RESOURCE_GROUP=$resource_group_name \
    A_RECORD_NAME=$dns_a_record \
    DNS_ZONE_NAME=$dns_zone \
    --secure-environment-variables \
    APP_ID=$service_principal_app_id \
    APP_PASSWORD=$service_principal_password \
    APP_TENANT_ID=$service_principal_tenant

# Get the ip address of the container instance
aci_ip=$(az container show \
    --name $aci_container_name \
    --resource-group $resource_group_name \
    --query ipAddress.ip --output tsv)

echo $aci_ip

```

Once the script has completed running, navigate to the container instance, and verify that the AZ CLI commands were executed.

Double check that the DNS record has been updated to match the ACI IP address.

## Related Links

[Github - Related Source Code](https://github.com/Garyljackson/PrivateAciDnsUpdate)

[ACI - Static Private IP + DNS](https://docs.microsoft.com/en-us/answers/questions/200596/azure-container-instance-in-a-vnet-doesn39t-suppor.html)

[ACI - Container Group lifecycle](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-container-groups#what-is-a-container-group)

[ACI - Managed Identity](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-virtual-network-concepts#other-limitations)

[ACI - Multi Container Deployment](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-multi-container-yaml)

[Docker Hub - Azure CLI image](https://hub.docker.com/_/microsoft-azure-cli)

[Azure Portal - Service Principles](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps) (Click the "All Applications" link)
