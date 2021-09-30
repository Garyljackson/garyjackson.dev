---
title: "Gitpod OpenVSCode Server on Azure ACI"
summary: "Remote development with Gitpod OpenVSCode Server with Azure Container Instances"
date: "2021-09-30T17:17:00+10:00"
lastmod: "2021-09-30T17:17:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Azure Container Instances"
  - "Gitpod"
---

## Overview

The kind folks over at [Gitpod](https://www.gitpod.io/blog/openvscode-server-launch) recently open-sourced their OpenVSCode Server project.

Gitpod OpenVSCode is a project that lets you create cloud-based development environments using Visual Studio Code in your browser.

There are a bunch of guides on how to fire up an instance across various environments; however, there is no guide for Azure Container Instances.

Given my very recent experience with ACI, I wanted to give that a try.

## Resources

The following Azure resources will be created.

- 1 x Resource Group
- 1 x Storage Account
- 1 x File Share in the Storage Account
  - This is where the container volume will mount
- 1 x Azure Container Instance

## How to

I've tried to make this whole process as seamless as possible and I've actually measured the provisioning time.

You can be up and running in under 80 seconds - pretty damn cool!

{{< admonition type=note title="Note" open=true >}}

Disclaimer: I haven't played around with the interface much at this stage, but on the surface all seemed fine.

Let me know if you come across anything.

{{< /admonition >}}

#### Steps

- Open Azure Cloud shell - [https://shell.azure.com/](https://shell.azure.com/)
- Make sure you're in `Bash` mode
- Copy, paste and run the following script
- Wait 80 seconds
- Click on the link displayed at the end
- Profit

```bash
# Define the variables
RESOURCE_SUFFIX=$RANDOM
RESOURCE_GROUP=GitpodOpenVSCode$RESOURCE_SUFFIX
LOCATION=australiaeast
SHARE_NAME=openvscode
STORAGE_ACCOUNT_NAME=openvscode$RESOURCE_SUFFIX
DNS_LABEL=gitpod-openvscode-$RESOURCE_SUFFIX
CONTAINER_NAME=$DNS_LABEL

# Create the resource group
az group create \
  --location $LOCATION \
  --name $RESOURCE_GROUP

# Create the storage account
az storage account create \
  --resource-group $RESOURCE_GROUP \
  --name $STORAGE_ACCOUNT_NAME \
  --location $LOCATION \
  --sku Standard_LRS

# Create the file share
az storage share create \
  --name $SHARE_NAME \
  --account-name $STORAGE_ACCOUNT_NAME

# Get the storage account key
STORAGE_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP --account-name $STORAGE_ACCOUNT_NAME --query "[0].value" --output tsv)

# Create the container
az container create \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_NAME \
  --image gitpod/openvscode-server \
  --dns-name-label $DNS_LABEL \
  --ports 3000 \
  --azure-file-volume-account-name $STORAGE_ACCOUNT_NAME \
  --azure-file-volume-account-key $STORAGE_KEY \
  --azure-file-volume-share-name $SHARE_NAME \
  --azure-file-volume-mount-path /home/workspace

# Get the URI to access the application
ACI_HOST_NAME=$(az container show --resource-group $RESOURCE_GROUP --name $CONTAINER_NAME --query ipAddress.fqdn --output tsv)
echo http://$ACI_HOST_NAME:3000
```

## Other Considerations

### https

This example does not support `https`, but it should be possible using the same strategy [I blogged about here]({{< relref "accessing-azure-container-instance-within-vnet" >}})

Assuming there are no other ports that need to be proxied

## Related Links

[Example Solution GitHub Repo](https://github.com/Garyljackson/GitpodOpenVSCodeAzure)

[Gitpod announcement](https://www.gitpod.io/blog/openvscode-server-launch)
