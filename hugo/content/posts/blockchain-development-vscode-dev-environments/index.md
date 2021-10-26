---
title: "Blockchain Development - Dev Containers"
summary: "Use self contained development containers with Visual Studio Code for blockchain development"
date: "2021-10-26T18:00:00+10:00"
lastmod: "2021-10-26T18:00:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Smart Contracts"
  - "Ethereum"
  - "Blockchain"
---

# Intro

In my quest to learn smart contract development, I've been installing and trialling various tools, frameworks, runtimes and components.
 
Whilst doing so, I've also been trying to keep my computer as functional as possible without causing unnecessary headaches.
 
## Strategy

Some strategies I've employed are:
 - node.js: using NVM to install, manage and switch between the node versions
 - pipx: Isolated environments for python applications
 - Global tools: Using NPX whenever available to avoid global tool installations
 
 
Even with these strategies, I couldn't escape the feeling that I was bloating my dev environment with ever-increasing dependencies.

## Solution

What I would like is some sort of transient environment that already has the languages, frameworks, tools and dependencies I need, ideally, something I can easily discard or rebuild without too much fuss, and crucially, not impact the main OS.
 
This is where Visual Studio Code development containers come in.

### Visual Studio Code Development Containers

The Visual Studio Code documentation has an entire section dedicated to setting up a per-project development containers so I won't go into detail here.  
 

If you'd just like to jump in using one of my dev container definitions below, then just make sure you have installed all the required components detailed [here](https://code.visualstudio.com/docs/remote/containers#_installation)


VS Code can also guide you through the process - you can choose to create a new container from scratch or use one of the pre-created Microsoft dev containers.  

You can also browse the dev containers Microsoft has made available [here](https://aka.ms/vscode-dev-containers)


## My Blockchain Dev Containers
 
For my own explorations, I've created the following dev containers to play around with.
 
### Truffle:  
The OG JavaScript-based solidity development framework  
- [Github - Truffle Dev Container](https://github.com/Garyljackson/TruffleDevContainerTemplate)

### Brownie
For Python-based solidity development  
- [Github - Brownie Dev Container](https://github.com/Garyljackson/BrownieDevContainerTemplate)

### Hardhat
The new cool kid on the block (hehe) - also JavaScript-based but supports TypeScript.  
- [Github - Hardhat Dev Container](https://github.com/Garyljackson/HardhatDevContainerTemplate)


### Instructions

Assuming you've already been through the VS Code documentation for containers, all you need to do is clone the repo, open the folder with Visual Studio Code, and then when prompted, reopen the project in a dev container.

Then, proceed the same way you usually would when developing with any one of these tools.

I've also included a readme that details everything installed within the dev container, and the likely next commands you would use depending on the framework you're using.
 
{{< admonition type=note title="Note" open=true >}}
The first time you use a dev container, docker needs to build the various layers.
The layers are cached and subsequent runs will be much faster.
{{< /admonition >}}

## Opinions

### Dev Containers

I've honestly been very happy with the results - the integration with VS Code is very slick.  

I've also started using the containers for other projects at work - specifically the Azure CLI dev container.

One of the other benefits I'm enjoying is having a consistent terminal across my machines - in this case Ubuntu.

### Solidity Development Frameworks
I think for the moment, Hardhat is going to be my focus. 

Initially, I considered focusing on Brownie, mostly because I also would like to write more Python, but, when it comes to the front end UI, it seems most have settled on React, so I figure TypeScript will probably be a really happy medium for me.

Hardhat also has a really good community behind it and their documentation is fantastic.