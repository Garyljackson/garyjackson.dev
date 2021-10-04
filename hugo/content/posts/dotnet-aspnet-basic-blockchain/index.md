---
title: "How to build a custom .NET core blockchain and PoW miner"
summary: "A basic blockchain using ASP.NET core and a example proof of work miner"
date: "2021-10-04T10:00:00+10:00"
lastmod: "2021-10-04T10:00:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Blockchain"
  - "PoW"
  - "Proof of work"
  - ".NET 6"
  - ".NET Core"
  - "ASP.NET Core"
---

## Overview

My curiosity around all things crypto has been gaining some momentum recently. I've always had what I would describe as a surface level understanding of how things work, but I've never sat down to codify my thoughts - until today.

If you're new to blockchain or also learning, maybe I can explain it in a way that resonates with how you think.

{{< admonition type=note title="Note" open=true >}}

Please note, I am still learning, and this is very much a work in progress.

Therefore, there is a chance that some of my assumptions might not be correct, and I fully expect my thoughts, opinions and understanding to iterate and evolve.

{{< /admonition >}}

## The Blockchain

The blockchain aspect of what we broadly refer to as crypto is only one component of many components layered together to form a functional system.

For this project, I've only dealt with a some of the components.

- A basic blockchain
- A simple miner using a proof of work algorithm

I have not implemented

- Adaptive proof of work difficulty
- A decentralised peer to peer network
- A decentralised consensus mechanism

## What is a Blockchain

In its most basic form, a blockchain is a collection of data blocks linked to the previous block using a cryptographic signature - thus, forming a verifiable chain of blocks.

This signature is called a hash

### The Hash

The cryptographic signature for each block uses the previous block's hash to generate its own hash. (among other inputs)

This hash signature chain means that if you try to modify any block in the chain, you automatically invalidate all successive blocks.

My mental model for this is like a linked list, but with added security and verifiable integrity.

## The Miner

The miner is a program that adds new blocks to the blockchain.

In the real world of crypto, the miners compete to win the right to add the next block to the chain.

To `Win` this right, the miners need to solve a cryptographic puzzle. The result of this puzzle is the SHA256 hash.
This hash is called the `proof of work` or `PoW`.

As more miners join the network, the overall network ability to solve the cryptographic hash becomes more likely.
The measure of this collective ability is called the `network hash rate`.

### Adaptive Hash Difficulty

The network adapts to the hash rate by varying the difficulty of the acceptable hash result.

The difficulty is varied by increasing or decreasing the number of zero's at the start of the hash value.

More leading zero's mean the miner has to iterate over many more hash variations to find a hash that meets the difficulty criteria.

{{< admonition type=note title="Note" open=true >}}
PoW is energy-intensive, and the community is working on less energy-intensive alternatives like 'Proof of Stake' or 'PoS'
{{< /admonition >}}

## Code Preview

I don't think its going to be helpful to include all the code in this post, but here are some previews.

You can access the [completed code here](https://github.com/Garyljackson/BasicBlockchain)

### The blockchain definition

```c#
using BlockchainApi.Contracts;
using BlockchainApi.Models;

namespace BasicBlockchain;

public class Blockchain : IBlockchain
{
    private readonly IBlockValidator _blockValidator;
    private readonly List<Block> _chain;

    public Blockchain(IBlockValidator blockValidator)
    {
        _blockValidator = blockValidator;
        _chain = new List<Block> {CreateGenesisBlock()};
    }

    public AppendBlockResult AppendBlock(Block block)
    {
        var previousBlock = GetChainHead();
        var validationResult = _blockValidator.ValidateBlock(block, previousBlock);

        if (validationResult.IsValid) _chain.Add(block);

        return new AppendBlockResult(validationResult);
    }

    public IReadOnlyCollection<Block> GetChain()
    {
        return _chain.AsReadOnly();
    }

    public Block GetChainHead()
    {
        return _chain.Last();
    }

    public ICollection<BlockValidationResult> ValidateChain()
    {
        var validationResults = new List<BlockValidationResult>();
        var previousBlock = _chain.First();

        foreach (var block in _chain.Where(block => block.Index != 0))
        {
            var result = _blockValidator.ValidateBlock(block, previousBlock);

            if (!result.IsValid) validationResults.Add(result);

            previousBlock = block;
        }

        return validationResults;
    }

    private static Block CreateGenesisBlock()
    {
        var block = new Block(0, DateTime.MinValue, "0", "0", 0, "");
        return block;
    }
}
```

### The hasher

```c#
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using BlockchainApi.Models;

namespace BlockchainApi;

public static class BlockCandidateHasher
{
    public static string HashBlockCandidate(this BlockCandidate block)
    {
        var blockCandidateJson = JsonSerializer.Serialize(block);

        using var sha256Hash = SHA256.Create();

        var bytes = sha256Hash.ComputeHash(Encoding.UTF8.GetBytes(blockCandidateJson));

        var builder = new StringBuilder();
        foreach (var t in bytes) builder.Append(t.ToString("x2"));

        return builder.ToString();
    }
}
```

### The miner (part)

```c#
using System.Net.Http.Json;
using BlockchainApi;
using BlockchainApi.Models;

namespace BlockchainMiner;

internal class BlockMiner
{
    private readonly HttpClient _httpClient;

    internal BlockMiner()
    {
        _httpClient = new HttpClient();
        _httpClient.BaseAddress = new Uri("https://localhost:7068/");
    }

    internal async Task<Block> MineBlockAsync(string data)
    {
        var blockMined = false;
        var minedBlockHash = string.Empty;
        uint currentNonce = 0;

        var chainHead = await _httpClient.GetFromJsonAsync<Block>("api/Blockchain/Head");

        var blockCandidate =
            new BlockCandidate(chainHead!.Index + 1, DateTime.UtcNow, chainHead.Hash, currentNonce, data);

        while (!blockMined)
        {
            blockCandidate = blockCandidate with {Proof = currentNonce};
            minedBlockHash = blockCandidate.HashBlockCandidate();

            if (minedBlockHash.StartsWith(BlockchainProperties.BlockDifficulty)) blockMined = true;

            currentNonce += 1;
        }

        return new Block(blockCandidate.Index, blockCandidate.Timestamp, minedBlockHash, blockCandidate.PreviousHash,
            blockCandidate.Proof, blockCandidate.Data);
    }
}
```

## The Example Solution

Requirements

- .NET 6 rc1 or above

The example I've created has the following components

- An ASP.NET 6 web API application
  - Host the blockchain instance
  - Enables interactions from the miner
- A miner console application
  - Solves the hash algorithm based on the difficulty
  - Submits the new block to the blockchain

To run the application

- Clone the [Github repository](https://github.com/Garyljackson/BasicBlockchain)
- Start up the web application
- Run the miner console application.

Running the web application will also open a swagger interface for you to interact with if you want.

## Related Links

[Github - Related Source Code](https://github.com/Garyljackson/BasicBlockchain)
