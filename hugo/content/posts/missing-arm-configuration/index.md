---
title: "Missing Azure Resource Manager Configuration"
summary: "How to find ARM configuration when there are no examples, and export template isn't helping."
date: 2020-06-01
lastmod: 2020-06-05
author: Gary Jackson
draft: false
categories:
- Development
tags:
- "Azure"
- "ARM"
- "Azure Resource Manager"
---

## Overview
Azure Resource Manager is my preferred method for deploying Azure infrastructure.

I've found that sometimes the resource configuration I would like to set isn't visible in the ARM definition using my normal approaches.

This post will cover the methods I have been using up to this point, as well as a new method I've just learned.

## An Example
I have an Azure Front Door configured with these two routing rules.

{{< figure src="images/RoutingRules.PNG" alt="Front Door Routing Rules"  >}}

Now, I want to create an ARM template to automate it's creation.

## The Usual Suspects
### 1. Azure Portal - Export Template
The strategy in this case is to manually create the resource via the Azure portal, then to use the "Export template" menu option to view the template contents.

{{< figure src="images/ExportTemplate.PNG" alt="Export template menu option"  >}}

Exporting the template generates the following routingRules configuration.

```json
{
    "routingRules": [
        {
            "id": "[concat(resourceId('Microsoft.Network/frontdoors', 'ExampleFrontDoor'), '/RoutingRules/defaultForwardingRule')]",
            "name": "defaultForwardingRule",
            "properties": {
                "frontendEndpoints": [
                    {
                        "id": "[concat(resourceId('Microsoft.Network/frontdoors', 'ExampleFrontDoor'), concat('/FrontendEndpoints/', 'ExampleFrontDoor', '-azurefd-net'))]"
                    }
                ],
                "acceptedProtocols": [
                    "Https"
                ],
                "patternsToMatch": [
                    "/*"
                ],
                "forwardingProtocol": "HttpsOnly",
                "cacheConfiguration": {
                    "queryParameterStripDirective": "StripNone",
                    "dynamicCompression": "Enabled"
                },
                "backendPool": {
                    "id": "[concat(resourceId('Microsoft.Network/frontdoors', 'ExampleFrontDoor'), '/BackendPools/defaultPool')]"
                },
                "enabledState": "Enabled",
                "resourceState": "Enabled"
            }
        }
    ]
}
```

Notice anything?  

Yeah, my `httpToHttpsRedirect` has gone missing! We've gone from two rules to one.  

The `apiVersion` in this case, was `2018-08-01` - not shown, but more on this later.

### 2. Azure Resource Explorer
[https://resources.azure.com](https://resources.azure.com)

This is a site that Microsoft provides to navigate through all your Azure resources, and displays the JSON configuration as you go.

You can either navigate to your resource using the tree on the left, or the search box at the top - the search is a good option.

Sadly, same results as above.

{{< figure src="images/AzureResourceExplorer.PNG" alt="Azure Resource Explorer" >}}


### 3. Azure Quickstart Templates
This repo contains all currently available Azure Resource Manager templates contributed by the community.

This is usually a good starting point also, but I find that sometimes the templates can be a bit outdated.

You can tell this by looking at the `$schema` JSON property listed at the top, or more specifically at the resource level, by looking at the `apiVersion` property of the resource you want to create.  

Some resource settings are only available on more recent `apiVersion`'s

If anything, it should at least give you a clue and point you in the right direction.

- [Quickstart Template GitHub Repo](https://github.com/Azure/azure-quickstart-templates)
- [Quickstart Template Search](https://azure.microsoft.com/en-us/resources/templates/)


but, sometimes there just isn't a quickstart example template for what you want to do.

## The New Strategy
### Azure PowerShell

To do this, you need to install [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-4.1.0)

Once you've installed Azure Powershell - grab the `id` for the resource you want to find.

You could build it yourself, but I find it easier to just copy it from Resource Explorer (no. 2 above)

Update the value for `$TheResourceId` in the code snippet below with your `id`.

Then paste the code into a PowerShell window, and run it.

```powershell
# This will log you into your Azure account via a browser - see the output after running
# It will give you a URL to open in your browser, and a temporary code to enter.
Connect-AzAccount

# This is the resource you want to see the ARM JSON definition for
$TheResourceId='/subscriptions/[YourSubscriptionId]/resourcegroups/[ResourceGroupName]/providers/Microsoft.Network/frontdoors/[ResourceIdentifier]'

# This gets the resource definition, and copies the output to your clipboard
Get-AzResource -ResourceId $TheResourceId -debug 5>&1 | clip
```

The first thing you'll see, is an output message in PowerShell directing you to use a browser, and navigate to a page.
{{< figure src="images/PowerShell.PNG" alt="PowerShell" >}}

Once the page opens, enter the code provided in the PowerShell output message.

{{< figure src="images/Auth.PNG" alt="Auth" >}}

Follow the login prompts, and when you're done, you should see the following.

{{< figure src="images/AuthSuccess.PNG" alt="Auth success" >}}

If you now return to your PowerShell window, the commands should have finished - and will have copied the output into your clipboard.

Open your favourite text editor and paste the contents of your clipboard.

It will output a lot of information, but right at the bottom you will see the ARM template definition.

Bingo!

```json
{
    "routingRules": [
        {
            "name": "defaultForwardingRule",
            "type": "Microsoft.Network/Frontdoors/RoutingRules",
            "properties": {
                "acceptedProtocols": ["Https"],
                "patternsToMatch": ["/*"],
                "enabledState": "Enabled",
                "resourceState": "Enabled",
                "routeConfiguration": {
                    "@odata.type": "#Microsoft.Azure.FrontDoor.Models.FrontdoorForwardingConfiguration",
                    "customForwardingPath": null,
                    "forwardingProtocol": "HttpsOnly",
                    "cacheConfiguration": {
                        "queryParameterStripDirective": "StripNone",
                        "queryParameters": null,
                        "dynamicCompression": "Enabled",
                        "cacheDuration": "P1D"
                    }
                }
            }
        },
        {
            "name": "httpToHttpsRedirect",
            "type": "Microsoft.Network/Frontdoors/RoutingRules",
            "properties": {
                "acceptedProtocols": ["Http"],
                "patternsToMatch": ["/*"],
                "enabledState": "Enabled",
                "resourceState": "Enabled",
                "routeConfiguration": {
                    "@odata.type": "#Microsoft.Azure.FrontDoor.Models.FrontdoorRedirectConfiguration",
                    "customFragment": null,
                    "customHost": "",
                    "customPath": "",
                    "redirectProtocol": "HttpsOnly",
                    "customQueryString": null,
                    "redirectType": "Found"
                },
            }
        }
    ]
}
```
I've removed some sensitive information, but you can clearly see that I now have both routing rules.

Also, helpful to know - if you look further up the output, you can find the currently supported `apiVersion` and `locations` for each resource.

```json
{
    "resourceType": "frontdoors",
    "locations": [
        "global",
        "Central US",
        "East US",
        "East US 2",
        "North Central US",
        "South Central US",
        "West US",
        "North Europe",
        "West Europe",
        "East Asia",
        "Southeast Asia",
        "Japan East",
        "Japan West",
        "Brazil South",
        "Australia East",
        "Australia Southeast"
    ],
    "apiVersions": [
        "2020-01-01",
        "2019-08-01",
        "2019-05-01",
        "2019-04-01",
        "2018-08-01"
    ],
    "defaultApiVersion": "2020-01-01",
    "capabilities": "SupportsTags, SupportsLocation"
}

```

Alright, that's about it from me - good luck.

{{< admonition type=note title="Update" open=true >}}
Since publishing this post, I've also discovered that you can dig throught the actual JSON definition files published on GitHub.

For instance, the current Front Door ARM JSON definition can be [seen here](https://github.com/Azure/azure-rest-api-specs/blob/master/specification/frontdoor/resource-manager/Microsoft.Network/stable/2020-05-01/frontdoor.json)
{{< /admonition >}}


## Resources
- [Azure Resource Explorer](https://resources.azure.com)
- [Quickstart Template GitHub Repo](https://github.com/Azure/azure-quickstart-templates)
- [Quickstart Template Search](https://azure.microsoft.com/en-us/resources/templates/)
- [Azure PowerShell Install](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-4.1.0)