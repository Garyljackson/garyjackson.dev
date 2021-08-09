---
title: "Azure Function App - appsettings.json and user secrets"
summary: "How to add support for appsettings.json and user secrets to a .NET 5 isolated Azure function app"
date: "2021-08-09T17:00:00+10:00"
lastmod: "2021-08-09T17:00:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Azure Function app"
  - ".NET 5"
  - ".NET Core"
  - "User Secrets"
  - "App settings"
---

## Overview

I recently started using the new .NET 5 Azure Function app (Isolated version).

I was a bit surprised that there isn't support right out the box for stuff we often use with ASP.NET - specifically `user secrets` and the `appsettings.json` files.

This is an attempt to document the process for anyone looking to do the same.

If you're just looking for the tldr version, then you can find the example source code [here](https://github.com/Garyljackson/AzureFunctionWithSettings)

## The Process

### Initial Project Creation

Before you begin, you must have the following:

- An Azure account with an active subscription
- [.NET 5.0 SDK](https://dotnet.microsoft.com/download)
- [Azure Functions Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local#v2) version 3.0.3381, or a later version.
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) version 2.20, or a later version.

The very first thing you need to do is log into Azure, running the following will provide a link and a code to autheticate with Azure

```powershell
az login
```

Next, navigate to a folder where you would like to create the Function App itself and run

```powershell
func init AzureFunctionWithSettings --worker-runtime dotnetisolated
```

Now lets add an HttpTrigger example function to work with

```powershell
func new --name HttpExample --template "HTTP trigger" --authlevel "anonymous"
```

{{< admonition type=warning title="Note" open=true >}}
`No templates found matching: 'HTTPtrigger`:

If you get this error then you likely have the .NET 6 preview SDK installed.

The workaround is to add a `global.json` file, as detailed on [Adam Storr's Post](https://adamstorr.azurewebsites.net/blog/azure-functions-cli-failing-after-preview-install)

{{< /admonition >}}

To verify that your new function is, well, functional, run the following.

This will start the function and provide a URL for you to open to access the new HttpTrigger endpoint.

```powershell
func start
```

### appsettings.json and User Secrets

Alright, now that we have the basic function up and running, let's add the bits and modifications.

First up, let's add the User Secret nuget package

```powershell
dotnet add package Microsoft.Extensions.Configuration.UserSecrets
```

Initialise the secret store and add a secret.

The colon in the name represents a `type:property` relationship.

You can also edit the file directly to have a normal parent/child json structure

```powershell
dotnet user-secrets init
dotnet user-secrets set "ExampleServiceOptions:ServiceApiKey" "abcSecretB"
```

While we're at it, create a new `appsettings.json` file also, and add the this:

```json
{
  "ExampleServiceOptions": {
    "Timeout": "00:00:30"
  }
}
```

Make sure to indicate that you want the appsettings file included in the build output.

To do this, add the CopyToOutput directive to the `AzureFunctionWithSettings.csproj` file

```xml
  <ItemGroup>
    <None Update="appsettings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
    <None Update="host.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
    <None Update="local.settings.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      <CopyToPublishDirectory>Never</CopyToPublishDirectory>
    </None>
  </ItemGroup>
```

Okay, next up, create a type to represent the settings, name it `ExampleServiceOptions.cs`

The `ServiceApiKey` setting will come from the `user secrets`, and the `Timeout` setting will come from the `appsettings.json`

```c#
using System;

namespace AzureFunctionWithSettings
{
    public class ExampleServiceOptions
    {
        public string ServiceApiKey { get; set; }
        public TimeSpan Timeout { get; set; }
    }
}

```

### Project Updates

Now, this is where the secret sauce comes in - update the `Program.cs` file with the following.

Note the new `ConfigureAppConfiguration` section, ABOVE the `ConfigureFunctionsWorkerDefaults()`.

The order of these registrations is important because the collection of settings providers override each other (Last wins), and if you're using docker containers, you will probably want the environment variable settings provider to override the others.

At this point that we're also registering the `ExampleServiceOptions` with our DI container.

```c#
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using System.Reflection;
using Microsoft.Extensions.DependencyInjection;

namespace AzureFunctionWithSettings
{
    public class Program
    {
        public static void Main()
        {
            var host = new HostBuilder()
                .ConfigureAppConfiguration(builder =>
                {
                    builder.AddJsonFile("appsettings.json", true, true)
                        .AddUserSecrets(Assembly.GetExecutingAssembly(), true);
                })
                .ConfigureFunctionsWorkerDefaults()
                .ConfigureServices(services =>
                {
                    services.AddOptions<ExampleServiceOptions>().Configure<IConfiguration>((settings, configuration) =>
                    {
                        configuration.GetSection(nameof(ExampleServiceOptions)).Bind(settings);
                    });
                })
                .Build();

            host.Run();
        }
    }
}
```

Now update the `HttpExample.cs` with the following:

Things to note:

- The `static` modifiers have been removed from both the `class` definition and the `Run` method
- There is now a constructor to accept the `IOptions<ExampleServiceOptions>` application settings - this will be injected by the DI container.
- The settings from the User Secret, and the appSettings file are being written out.

```c#
using System.Net;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace AzureFunctionWithSettings
{
    public class HttpExample
    {
        private readonly IOptions<ExampleServiceOptions> _config;

        public HttpExample(IOptions<ExampleServiceOptions> config)
        {
            _config = config;
        }

        [Function("HttpExample")]
        public HttpResponseData Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req,
            FunctionContext executionContext)
        {
            var logger = executionContext.GetLogger("HttpExample");
            logger.LogInformation("C# HTTP trigger function processed a request.");

            var response = req.CreateResponse(HttpStatusCode.OK);
            response.Headers.Add("Content-Type", "text/plain; charset=utf-8");


            response.WriteString("Welcome to Azure Functions! \n");
            response.WriteString($"The ServiceApiKey setting is: {_config.Value.ServiceApiKey} \n");
            response.WriteString($"The Timeout setting is is: {_config.Value.Timeout} \n");

            return response;
        }
    }
}

```

If you now run the function, and navigate to the URL it provides, then you should see your appsettings/user secrets being output from the service.

```
func run
```

### Other Stuff

I noticed that the `local.settings.json` file is excluded from the source control by the `.gitignore` file created by the Azure Functions Core Tools

However, if someone else clones the repo without this file, then you get warnings in the CLI.

Given that there shouldn't be anything sensitive in this file now, I've removed the exclusion from the `.gitignore`

```
# Azure Functions localsettings file
# local.settings.json

```
