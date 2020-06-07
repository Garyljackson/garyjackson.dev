---
title: "Automate Azure Front Door HTTPS"
summary: "How to automate enabling HTTPS for Azure Front Door"
date: "2020-06-07T09:35:00+10:00"
lastmod: "2020-06-07T09:35:00+10:00"
author: Gary Jackson
draft: false
categories:
- "Development"
tags:
- "Azure"
- "Azure Front Door"
- "HTTPS"
- "ARM"
- "Azure CLI"
- "Devops"
---

## Overview

These are the details of a recent attempt to automate the HTTPS configuration for Azure Front door.

Spoilers: I tried to use ARM, it didn't work, I had to resort to Azure CLI as a secondary step in the deploy process.


## Azure Front Door HTTPS
Providing a secure connection via HTTPS has come along nicely in the past couple years. 

Largely thanks to the efforts of companies like [Let's Encrypt](https://letsencrypt.org/), you can now create the certificates needed to secure your communications free of charge.

Assuming you have a custom domain associated with your Front Door instance, Azure has two options for enabling HTTPS communications for your custom domain.
1. "Front Door managed"
2. "Use my own certificate"

The "Front Door managed" version is the variant that I'll be using for this post.

## Front Door managed certificates
By choosing this option, Azure Front door will handle all certificate management tasks for you.

This includes the provisioning of the initial certificate, as well as making sure to renew the certificate whenever appropriate.

No more stress about keeping tabs on certificate expiry and renewal! :raised_hands:

### CNAME configuration/domain validation Gotcha
Before diving right into the attempts, lets talk about a gotcha I had along the way.

And by gotcha, I mean, the documentation clearly states it, but I skimmed over it... :roll_eyes:

Having previously [configured](https://docs.microsoft.com/en-au/azure/frontdoor/front-door-custom-domain#map-the-temporary-afdverify-subdomain) the `afdverify` CNAME entry to add a custom domain to my front door instance, I had assumed that the same entry would be used to validate the domain prior to provisioning the certificate.

Don't fall into this trap, it doesn't work like that, as documented [here](https://docs.microsoft.com/en-au/azure/frontdoor/front-door-custom-domain-https#custom-domain-is-not-mapped-to-your-front-door)

Front Door will only automatically validate your domain and provision a certifiate IF the CNAME for the domain is mapped DIRECTLY to your Front Door URI.  

For example:

This WILL work: `CNAME: www.contoso.com -> contoso.azurefd.net`

This will NOT work: `CNAME: afdverify.www.contoso.com -> afdverify.contoso.azurefd.net`

In my case, there was a CloudFlare proxy in the middle, hiding the azurefd.net origin URI, and preventing the automated validation.


### Attempt 1 - ARM (Fail)
After following [my usual process for discoverying ARM configuration]({{< ref "missing-arm-configuration" >}}), I landed on something like this extract from the `frontendEndpoints` part of the ARM template.


```JSON
{
    "frontendEndpoints": [
        {
            "name": "www-contoso-com",
            "properties": {
                "hostName": "www.contoso.com",
                "sessionAffinityEnabledState": "Disabled",
                "sessionAffinityTtlSeconds": 0,
                "customHttpsConfiguration": {
                    "certificateSource": "FrontDoor",
                    "minimumTlsVersion": "1.2",
                    "protocolType": "ServerNameIndication",
                    "frontDoorCertificateSourceParameters": {
                        "certificateType": "Dedicated"
                    }
                },
                "resourceState": "Enabled"
            }
        }
    ]
}
```

Unfortunately, while this doesn't produce any kind of errors when deployed, it also produces no result.
I've also found the following [Stack Overflow](https://stackoverflow.com/questions/58180861/enable-https-on-azure-front-door-custom-domain-with-arm-template-deployment) question with similar results.

### The Solution - Azure CLI
Ultimately my solution is a hybrid ARM + Azure CLI deployment.

I have two tasks in my Azure Devops Release Pipeline
1. [Azure resource group deployment](https://github.com/microsoft/azure-pipelines-tasks/blob/master/Tasks/AzureResourceGroupDeploymentV2/README.md)
2. [Azure CLI](https://github.com/microsoft/azure-pipelines-tasks/blob/master/Tasks/AzureCLIV2/Readme.md)


The first task deploys the majority of the Azure resources, the second task configures anything not specifically supported by ARM at this time.

{{< admonition type=note title="Subsequent ARM deployments" open=true >}}
I have verified that if I deploy just the ARM template after enabling HTTPS, then HTTPS remains enabled even without any certificate configuration in the ARM template itself.
{{< /admonition >}}


Here is a slightly modified version of the code I'm using for the Azure CLI task.

```PowerShell
# This adds the extension needed to interact with Front Door
az extension add --name front-door

# The Front Door instance name
$frontDoorName = 'contoso-fd'
# The resource group the Front Door instance is in
$resourceGroupName = 'contoso-rg'
# The name of the frontend/domain
$frontendEndpoint = 'www-contoso-com'

write-output "frontDoorName: $frontDoorName"
write-output "resourceGroupName: $resourceGroupName"
write-output "frontendEndpoint: $frontendEndpoint"

# Gets the current HTTPS enablement state
$customDomainHttpsProvisioningState = (az network front-door frontend-endpoint show --front-door-name $frontDoorName --name $frontendEndpoint --resource-group $resourceGroupName --query 'customHttpsProvisioningState' -o tsv)

# 'Enabling' is also a possible provisioning status, so use 'Disabled' directly
$customDomainEnableHttps = ($customDomainHttpsProvisioningState -eq 'Disabled') 

if($customDomainEnableHttps) {
   az network front-door frontend-endpoint enable-https --front-door-name $frontDoorName --name $frontendEndpoint --resource-group $resourceGroupName
}else{
    write-output "HTTPS is already enabled/enabling for $frontendEndpoint"
}

```

Success!

{{< figure src="images/Success.png" alt="Success" >}}