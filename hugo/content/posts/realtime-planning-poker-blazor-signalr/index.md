---
title: "Blazor Learnings - Realtime planning poker with Blazor and SignalR"
summary: "What is a contained database user, and how to use them?"
date: "2020-07-26T11:00:00+10:00"
lastmod: "2020-07-26T11:00:00+10:00"
author: Gary Jackson
draft: false
categories:
- "Development"
tags:
- "Blazor"
- "SignalR"
- ".NET Core"
---

## Overview

I took some time out over the past weekend to dive into a bit of Blazor Web Assembly development.
These are my learnings and takeaways from the experience.

## The App
Most people will already be familiar with planning poker, at a high level I wanted to support the following features
- Realtime remote collaboration
- Create new game or join existing game in progress
- Votes remain hidden until everyone has voted
- Reset the game for all participants

A demo version of the application is available [Here](https://realtimeplanningpoker.azurewebsites.net/)  
Or, if you'd like to jump into the source - you can find that [Here](https://github.com/Garyljackson/PlanningPoker)

{{< figure src="images/StartGame.png" alt="Game start" >}}
{{< figure src="images/Revealed.png" alt="Votes revealed" >}}

## Learnings
### How easy is it to get started?
This really couldn't have been any easier, assuming you already have the latest version of .NET Core, open your favourite terminal and run

```
dotnet new blazorwasm --name PlanningPoker --hosted true
```

The `--hosted true` paramater indicates that you would like a separate ASP.NET Core host for the Blazor WebAssembly app

The result will be a nicely separated project structure with the server, client and shared components
{{< figure src="images/SolutionStructure.png" alt="Solution Structure" >}}

And, if you hit run, you'll get the Blazer hello world application

{{< figure src="images/HelloWorld.png" alt="Hello World" >}}

### SignalR configuration
The SignalR package is included in the server project by default.
To get SignalR up and running, all you need is a hub, and some extra configuration sauce.

Example Hub
```c#
using Microsoft.AspNetCore.SignalR;
using PlanningPoker.Shared;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace PlanningPoker.Server.Hubs
{
    public class GameHub : Hub
    {
    }
}

```
In the `Startup.cs` file, include the following highlighted changes
```c# {linenos=table,hl_lines=[3,"6-10"]}
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddSignalR();
            services.AddControllersWithViews();
            services.AddRazorPages();
            services.AddResponseCompression(opts =>
            {
                opts.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
                    new[] { "application/octet-stream" });
            });
        }
```

```c# {linenos=table,hl_lines=[3,27]}
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseResponseCompression();

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
                app.UseWebAssemblyDebugging();
            }
            else
            {
                app.UseExceptionHandler("/Error");
                // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
                app.UseHsts();
            }

            app.UseHttpsRedirection();
            app.UseBlazorFrameworkFiles();
            app.UseStaticFiles();

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapRazorPages();
                endpoints.MapControllers();
                endpoints.MapHub<GameHub>("/GameHub");
                endpoints.MapFallbackToFile("index.html");
            });
        }
```
Besides adding your specific implementation for your `Hub` this is all that was needed to get SignalR up and running.  

The full implementation for my hub is available [here](https://github.com/Garyljackson/PlanningPoker/blob/master/Server/Hubs/GameHub.cs)

{{< admonition type=warning title="Note" open=true >}}
Hubs are transient:

- Don't store state in a property on the hub class. Every hub method call is executed on a new hub instance.
- Use `await` when calling asynchronous methods that depend on the hub staying alive. For example, a method such as `Clients.All.SendAsync(...)` can fail if it's called without `await` and the hub method completes before `SendAsync` finishes.

{{< /admonition >}}


### Component Composition
Using the default approach, components are composed of both markup and C# code.

`MyCoolThing.razor`
```c#
<h3>MyCoolThing</h3>

@code {
 // Your C# code here
}

```

To use this component from within another is simply a matter of referencing the component name in your markup.

```c#
<MyCoolThing></MyCoolThing>

@code {

}
```


### Code Behind
At this point you might be thinking the same as I was - "Yuk, I don't like having all my markup and c# code mixed together like this".

So, does Blazor support a sort of Code Behind concept?

Yes, yes it does! 

To do this you need to use `partial` classes - [Microsoft has some helpful documentation on the subject](https://docs.microsoft.com/en-us/aspnet/core/blazor/components/?view=aspnetcore-3.1#partial-class-support)

### Component Events
How can child components communicate events to a parent component?

This one turned out to be super simple too.

From with the child component, expose an `EventCallback` parameter

```c#
@using PlanningPoker.Shared

<button type="button" class="btn btn-secondary" @onclick="OnClicked">@Card.Text</button>

@code {
    [Parameter]
    public Card Card { get; set; }

    [Parameter]
    public EventCallback<Card> OnCardClicked { get; set; }

    private async Task OnClicked()
    {
        await OnCardClicked.InvokeAsync(Card);
    }
}

```
Then wherever you use the component, just wire up a handler method

```c#
<DeckCard Card="card" OnCardClicked="PlayCardAsync"></DeckCard>


@code {
    private async Task PlayCardAsync(Card card)
    {
        //Do stuff here
    }
}

```

### Blazor Dependency Injection
Registering services with Blazor WASM is very similar to registering services with any .NET Core application

```c# {linenos=table,hl_lines=[10]}
namespace PlanningPoker.Client
{
    public class Program
    {
        public static async Task Main(string[] args)
        {
            var builder = WebAssemblyHostBuilder.CreateDefault(args);
            builder.RootComponents.Add<App>("app");

            builder.Services.AddSingleton<GameStateManager>();
            builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });

            await builder.Build().RunAsync();
        }
    }
}
```

{{< admonition type=warning title="Note" open=true >}}
Blazor WebAssembly apps don't currently have a concept of DI scopes. `Scoped`-registered services behave like `Singleton` services. 
{{< /admonition >}}


Then, within the Blazor component itself, simply include the `@inject` Razor directive

```c# {linenos=table,hl_lines=[4,5]}
@using PlanningPoker.Client.Services
@using PlanningPoker.Shared

@inject NavigationManager NavigationManager
@inject GameStateManager StateManager
```

### Debugging
The debug configuration is all handled by the Dotnet new template, but it's helpful to know where this configuration lives.

If you have a look at the `launchSettings.json` file for the client app, you will see the following lines. (highlighted)

In case you're wondering, `wsProtocol` means WebSockets protocol

```json {linenos=table,hl_lines=[14,22]}
{
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:44424",
      "sslPort": 44325
    }
  },
  "profiles": {
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "inspectUri": "{wsProtocol}://{url.hostname}:{url.port}/_framework/debug/ws-proxy?browser={browserInspectUri}",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "PlanningPoker": {
      "commandName": "Project",
      "launchBrowser": true,
      "inspectUri": "{wsProtocol}://{url.hostname}:{url.port}/_framework/debug/ws-proxy?browser={browserInspectUri}",
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}

```

{{< admonition type=warning title="Note" open=true >}}

Debugging requires either of the following browsers.
- Google Chrome (version 70 or later) (default)
- Microsoft Edge (version 80 or later)

It most certainly doesn't work with FireFox - this one caught me out for a while
{{< /admonition >}}

### State Management
A Blazor application is largely composed of a bunch of Blazor components, many of those nested within each other.

How do we maintain state across various components?

As far as I can tell there are essentially two options.
1. Explicitly expose a paramater in each component, and manually plumb the data.
2. Use cascading parameters - In this case, Blazor automatically exposes the data to sub-components automatically.
3. Use dependency injection to create something to manage state for you.

In the case of this app, I went with option 3.

The trick is to create some kind of state manager service, and register that with the container as a singleton, then just inject it whereever you need it.

Though, I can't say I really put all that much thought into the pro's and con's for each approach.

### Wiring up SignalR
To get SignalR working with your app, you'll first need to add the `Microsoft.AspNetCore.SignalR.Client` Nuget package to your client.

Then, within the code section of your component, initialise a new hub, and subscribe to the relevant events.

```c#
@page "/ThePage"

@using Microsoft.AspNetCore.SignalR.Client
@implements IDisposable

<div>Your markup here</div>

@code {
    private HubConnection _hubConnection;

    protected override async Task OnInitializedAsync()
    {
        _hubConnection = new HubConnectionBuilder()
        .WithUrl(NavigationManager.ToAbsoluteUri("/YourHub"))
        .Build();

        _hubConnection.On<string, string>("ReceiveMessage", (user, message) =>
        {
            // Do stuff
            StateHasChanged();
        });

        await _hubConnection.StartAsync();
    }

    public void Dispose()
    {
        _ = _hubConnection.DisposeAsync();
    }
}
```

### Other stuff worth mentioning
#### Blazor Web Assembly code is not secure
I'm sure you already realise this, but it's worth mentioning just in case.

The code and files for your Blazor Web assembly app can easily be viewed on the client.

Treat Blazor WASM exactly how you would any other Single Page App like React. e.g.
- Don't include anything secret in your code
- Don't trust the client - validate everything on the server

#### Resharper causes Visual Studio crashes
My Visual Studio 2019 kept crashing, disabling the ReSharper plugin solved the issue.
At the time of writing, the versions were
- Visual Studio 2019: Version 16.6.5
- Resharper: 2020.1.4

{{< figure src="images/CrashProcess.png" alt="Crash Process" >}}
{{< figure src="images/CrashReason.png" alt="Crash Reason" >}}

#### Integrity Error
Sometimes, after modifying and running your code - the browser will report an Integrity Error in the developer tools console.

Doing a full rebuild of the solution solves this
{{< figure src="images/IntegrityError.png" alt="Integrity Error" >}}


Okay, I that's all I can think of at the moment, I'll be sure to update this if anything else comes to me