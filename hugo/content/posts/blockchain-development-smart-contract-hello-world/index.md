---
title: "Smart Contract Development - Hello World"
summary: "How to set up your local development environment and then create, test and deploy your first smart contract"
date: "2021-10-13T18:00:00+10:00"
lastmod: "2021-13-10T18:00:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Smart Contracts"
  - "Ethereum"
  - "Blockchain"
---

# Overview

In this post, I'll be covering how to get a local dev environment setup and configured for doing smart contract development with Ethereum, and also a quick hello world app to get our very first smart contract under the belt

## Prerequisites

You're going to need to download and install the following.

**Visual Studio Code**

https://code.visualstudio.com/download

**Visual Studio Code Solidity extension**

https://marketplace.visualstudio.com/items?itemName=JuanBlanco.solidity

**Ganache**

Ganache is the local development blockchain emulator.  
I've gone with the UI version for this post, but there is also a CLI version available.  
https://www.trufflesuite.com/ganache

**Node.js**

For Node.js you can install it directly, but I would suggest that you use something like node version manager to manage multiple node version installations.
I'm using version 16.11.1 currently.

https://nodejs.org/en/

**Truffle**

Truffle describes itself as "A world class development environment, testing framework and asset pipeline for blockchains using the Ethereum Virtual Machine (EVM), aiming to make life as a developer easier."

To install truffle, open a terminal window and run  
`npm install truffle -g`

{{< admonition type=note title="Note" open=true >}}
This step produces a lot of warnings, I ignored them and moved on.
{{< /admonition >}}

Alright, now that all that's done, let's move on to creating our first smart contract.

## Hello World

### Start Ganache

First up, we need to start our local blockchain.  
Once Ganache has started, on the `Create a workspace` screen, select the `Quickstart` button.  
This will launch you onto the main app and display a list of wallet addresses.

### Initialise the project

Open a terminal window, navigate to a folder for your source code and run

```bash
mkdir DappHelloWorld
cd .\DappHelloWorld\
code .
```

Now, from within the visual studio terminal, run the following.

```bash
truffle init
```

You will end up with a folder structure like this.

{{< figure src="images/TruffleInitFolders.png" alt="Init Folder Structure" >}}

Compile the project as a quick smoke test.

```bash
truffle compile
```

{{< figure src="images/InitTruffleCompile.png" alt="Init Compile Output" >}}

### Configure Ganache

We need to configure truffle to use our local development blockchain.  
Edit the `truffle-config.js` file  
Uncomment the `networks/development` json node and its contents.  
Update the port number to the port your Ganache app is running on, you can see this on the Ganasche UI under the `RPC SERVER` title.

The result should look something like this (Note, this is just a partial excerpt)

```json
...
  networks: {
    // Useful for testing. The `development` name is special - truffle uses it by default
    // if it's defined here and no other network is specified at the command line.
    // You should run a client (like ganache-cli, geth or parity) in a separate terminal
    // tab if you use this network and you must also set the `host`, `port` and `network_id`
    // options below to some value.
    //
    development: {
      host: "127.0.0.1", // Localhost (default: none)
      port: 7545, // Standard Ethereum port (default: none)
      network_id: "*", // Any network (default: none)
    }
...
```

While we're in the truffle config, let's also set up the solidity language version we will be using.  
Uncomment the compilers/solc/version node, and set its value to `>=0.7.0 <0.9.0`  
The result should look like this.

```json
...
  compilers: {
    solc: {
      version: ">=0.7.0 <0.9.0",
      // docker: true,        // Use "0.5.1" you've installed locally with docker (default: false)
      // settings: {          // See the solidity docs for advice about optimization and evmVersion
      //  optimizer: {
      //    enabled: false,
      //    runs: 200
      //  },
      //  evmVersion: "byzantium"
      // }
    },
  }
  ...
```

### Define a test

Before we create our smart contract, lets define a test.  
If you've defined any sort of JavaScript tests in the past, this should all be very familiar.  
I think it might be [mocha](https://mochajs.org/), but I'm not sure.

The test below checks that our (yet to be defined) greeting variable is populated when our smart contract is deployed.
The test then verifies that the greeting can be updated.

Create a new file inside the tests folder named `helloworld.js`

Place the following code into the new file

```JavaScript
const HelloWorld = artifacts.require("HelloWorld");

contract("HelloWorld", (accounts) => {
  it("should initialise the greeting", async () => {
    const HelloWorldInstance = await HelloWorld.deployed();
    const greeting = await HelloWorldInstance.greeting.call();

    assert.equal(
      greeting,
      "Initial Greeting",
      "The initial greeting was not set"
    );
  });

  it("should update the greeting", async () => {
    const HelloWorldInstance = await HelloWorld.deployed();
    await HelloWorldInstance.setGreeting("Howzit!");

    const greeting = await HelloWorldInstance.greeting.call();
    assert.equal(greeting, "Howzit!", "The greeting was not updated");
  });
});

```

#### Running the test

run `truffle test`

Hey! errors, just what we wanted.

{{< figure src="images/ContractNotExistsTest.png" alt="First test run results" >}}

### The Smart Contract

Finally, let's write that smart contract!  
Create a new file within the contracts folder named `HelloWorld.sol`

Add the following code.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.7.0 <0.9.0;

contract HelloWorld {
    string public greeting;

    constructor() {
        greeting = "Initial Greeting";
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }
}

```

Assuming you've done some development in the past, this will be easy to grok; however, there are some things worth mentioning.

- The licence line is optional but encouraged.
- All contracts are defined within a contract - similar to a class.
- The `pragma` line is required, and defines the version of solidity being used.
  - This needs to align with what you configured in the `truffle-config.js` file earlier.
- Public variables, like `greeting`, are automatically given a `getter` method.
- The `memory` data annotation is needed for reference types
  - In this case, it means that the parameter itself does not need to be persisted to the blockchain state.
  - If you want something to be persisted to the blockchain, you use the `storage` annotation.

Okay, let's give that test another try and see what happens now.

Run `truffle test` again

Okaaaay, the error has changed, but why are we still getting an error??

{{< figure src="images/ContractNotMigrated.png" alt="Not migrated test result" >}}

**This brings us to the migrations.**

### Contract migrations

We need to create a migration for our new smart contract to publish it to the blockchain.  
These migrations are similar in concept to database migrations like Entity Framework.

Create a new file named `2_deploy_contract.js` in the migrations folder and add the following code.

```JavaScript
const HelloWorld = artifacts.require("HelloWorld");

module.exports = function (deployer) {
  deployer.deploy(HelloWorld);
};

```

Now, once again, let's run `truffle test`  
ayyyy!! success! Congratulations, you just created, deployed and tested your very first smart contract.

{{< figure src="images/TestSuccess.png" alt="Success test result" >}}

## Related Links

[Github - Related Source Code](https://github.com/Garyljackson/DappHelloWorld)

[Solidity Doco](https://docs.soliditylang.org/)
