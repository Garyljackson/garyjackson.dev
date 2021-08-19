---
title: "Azure Function App - Terraform: Requested feature is not available in resource group"
summary: "How to solve conflicting Azure App Service plans"
date: "2021-08-18T17:13:10+10:00"
lastmod: "2021-08-18T17:13:10+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Azure Function app"
  - "Azure"
  - "Terraform"
  - "Bytesized"
---

## Overview

I was attempting to deploy a collection of new resources to Azure via Terraform, and recieved the following error.

```bash
Error creating/updating App Service Plan "MySpecialFunc-asp" (Resource Group "ExampleRg1"):
web.AppServicePlansClient#CreateOrUpdate: Failure sending request: StatusCode=0 -- Original Error: Code="BadRequest" Message="Requested feature is not available in resource group ExampleRg1.
Please try using a different resource group or create a new one."

Details=
[
    {"Message":"Requested feature is not available in resource group ExampleRg1. Please try using a different resource group or create a new one."},
    {"Code":"BadRequest"},
    {"ErrorEntity":{"Code":"BadRequest","ExtendedCode":"59324","Message":"Requested feature is not available in resource group ExampleRg1. Please try using a different resource group or create a new one.", "MessageTemplate":"Requested feature is not available in resource group {0}. Please try using a different resource group or create a new one.","Parameters":["ExampleRg1"]}}
]
```

## Details

The important resources for the context of this post are:

- 1 Consumption based app service plan + 1 Function App
- 1 Basic (B1) app service plan + 1 Web Application

Just regular Azure resources, nothing special, and besides, I had already successfully manually deployed these resources via the Azure Portal to a separate resource group and everything was fine.

Looking at the resources within Azure, I could see that the Basic app service plan had been created successfully, so this was somehow related to the app service plan for the function.
I decided to try create the missing resource manually in the portal to see if an error also occurs - and there was.

```bash
{
  "code":"InvalidTemplateDeployment",
  "message":"The template deployment 'Microsoft.Web-FunctionApp-Portal-17a8e6f3-977e' is not valid according to the validation procedure. The tracking id is 'aaaaaaaa-bbbb-ccc-987b-e3d34b22bcc4'. See inner errors for details.",
  "details":[{"code":"ValidationForResourceFailed","message":"Validation failed for a resource. Check 'Error.Details[0]' for more information.","details":[{"code":"LinuxDynamicWorkersNotAllowedInResourceGroup","message":"Linux dynamic workers are not available in resource group ExampleRg1. Use this link to learn more http://go.microsoft.com/fwlink/?LinkId=825764."}]}]}
```

Based on that, my initial assumption was that I was trying to create both a Linux ASP, and a Windows ASP in the same resource group, and somehow this wasn't allowed.

I verified that both my ASP's were set to `Linux`, and then decided to follow the link provided in the error details - [Redirects here](https://github.com/Azure/Azure-Functions/wiki/Creating-Function-Apps-in-an-existing-Resource-Group)

The details provided by Microsoft are:

{{< admonition type=note title="Note" open=true >}}
In short the reason this happens is behind the scenes we have different pools of resources. Different SKUs require a different set of machines. For example, the Azure Functions Premium Plan requires modern and larger machines, whereas the consumption / dynamic plans do not. When you create an app in a resource group, that resource group is mapped and assigned to a specific pool of resources. If you try to create another plan in that resource group and the mapped pool (called ‘stamp’ internally) does not have the required resources, this error will occur. The best course of action is to create the plan in a new resource group. If you need the option to move between something like Windows Consumption and Windows Functions Premium, you could also create the Functions Premium Plan first in a resource group (which would map the resource group to a pool that has premium VMs available) and then create the consumption plan (which can run in nearly all Windows pools).

{{< /admonition >}}

Thinking back to the collection of resources I has created manually, I remembered that I had created the ASP for the Function App, BEFORE the ASP for the Web Application.

## Solution

On a hunch, I decided to try enforce the creation order for the ASP resources in my template by making the Web Apps ASP depend on the Function Apps ASP - and SUCCESS!

To put it simply, make sure you create the App Service Plans for you Function App, before you create any other App Service Plans.

### Example

```tf {linenos=table,hl_lines=[26]}
resource "azurerm_app_service_plan" "func" {
  name                = "MySpecialFunc-asp"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  kind                = "Linux"
  reserved            = true

  sku {
    tier = "Dynamic"
    size = "Y1"
  }
}

resource "azurerm_app_service_plan" "main" {
  name                = "MySpecialApp-asp"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  kind                = "Linux"
  reserved            = true

  sku {
    tier = "Basic"
    size = "B1"
  }

  depends_on = [azurerm_app_service_plan.func]
}
```
