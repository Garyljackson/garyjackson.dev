---
title: "Extracting GitHub Repository Details via REST with PowerShell"
summary: "How to extract information about your organisations GitHub repositories using the GitHub REST API (v3) and PowerShell"
date: "2020-06-23T19:27:00+10:00"
lastmod: "2020-06-23T19:27:00+10:00"
author: Gary Jackson
draft: false
categories:
- "Development"
tags:
- "GitHub"
- "PowerShell"
---

## Overview

This week I decided to do some GitHub housekeeping at my workplace.

We currently have about 120+ repositories spread across three separate GitHub organisation accounts.

This is a pain for several reasons
1. Locating source code for any particular project
2. Managing users across multiple organisations
3. Configuring external integrations - e.g. Azure Devops
4. Cost

The first thing I need to do is create a consolidated list of all the repositories across all the organisations so we can decide what to move, archive, delete etc.

This post is specifically about that.

## Ze Details

Fortunately for us, Microsoft has already done a lot of the work for us in the form of the `PowerShellForGitHub` PowerShell Module

The process we're going to follow is
1. Create a GitHub personal access token
2. Install the PowerShellForGitHub PowerShell Module
3. Use our new PowerShell Module to set our GitHub Auth details
4. Query GitHub to see the repositories and the properties available
5. Refine and Export the data to CSV

### Creating a GitHub personal access token
Assuming you're already logged into your GitHub account in your browser - navigate to the [New personal access token page](https://github.com/settings/tokens/new)

Provide a name for the access token in the `Note` box, and then select the `repo` scope.

{{< admonition type=warning title="Be warned" open=true >}}
Selecting the top level `repo` scope is probably overkill for this use case - but I didn't try to restrict it further.
{{< /admonition >}}

{{< figure src="images/PersonalAccessToken.png" alt="Person access token" >}}

Once you've clicked save, your fresh new token will be displayed - save it somewhere safe, you need it later, and you won't be able to see it again.

{{< figure src="images/PersonAccessTokenSaved.png" alt="Person access token saved"  >}}

### Install the PowerShellForGitHub PowerShell Module
Open a PowerShell session, paste and run the following script.

It will prompt you with an Untrusted Repository warning - use the `A` (All) option to continue
```PowerShell
Install-Module -Name PowerShellForGitHub
```

{{< figure src="images/InstallModule.png" alt="Install Module" >}}

### Set your GitHub Auth credential
This one has a slight oddity to it, when you run this script, it will prompt you for your `username` - you can enter anything, it's just ignored.

However, when prompted for the password - paste in the personal access token you created in step 1


```PowerShell
Set-GitHubAuthentication
```
{{< figure src="images/SetAuth.png" alt="Set GitHub Authentication" >}}

### View all the repositories and the available properties
Now, to list your repositories, run the following.

On the first run, it might download some additional nuget packages before executing.

```PowerShell
Get-GitHubRepository -Visibility All
```
The result will list out all your repositories and show the available properties.
{{< figure src="images/ListAll.png" alt="List all repository details" >}}

### Refine and Export the data to CSV
Alright, once you've decided on the properties you want to export, update the script below and run it.

In my case, I wanted the `name`, `description`, `fork`, `archived`, `created_at` and `updated_at` property values, but then also a SubString from the `full_name` property, that I'm aliasing to `organisation`

This is done with a calculated property

```PowerShell
Get-GitHubRepository -Visibility All | Select-Object -Property @{
    label      = 'organisation'
    expression = { ($_.full_name).SubString(0, $_.full_name.IndexOf("/")) }
}, name, description, fork, archived, created_at, updated_at | Export-Csv -Path .\Repositories.csv -Force 
```

You will find the `Repositories.csv` file in the same path as your PowerShell session - in my case that was `D:\Repositories.csv`

## Bonus Detail
You can copy the debug output stream from your script to the clipboard automatically using `Redirection`

For example, using the script from earlier, if you append `5>&1 | clip` - this will take the output from the PowerShell debug stream, redirect it to the success stream, and pipe it to your clipboard.

```PowerShell
Get-GitHubRepository -Visibility All 5>&1 | clip
```

After running it, just head on over to your favourite text editor and do a paste.

## References
- [GitHub - REST API v3](https://developer.github.com/v3/)
- [GitHub - PowerShellForGitHub PowerShell Module](https://github.com/microsoft/PowerShellForGitHub)
- [PowerShell - About Redirection](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_redirection?view=powershell-7)
- [PowerShell - Calculated Property](https://docs.microsoft.com/en-us/powershell/scripting/samples/selecting-parts-of-objects--select-object-?view=powershell-7)