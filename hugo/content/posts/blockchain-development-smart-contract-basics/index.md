---
title: "Smart Contract Development - The Basics"
summary: "The basic concepts for smart contract development"
date: "2021-10-10T18:00:00+10:00"
lastmod: "2021-10-10T18:00:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Smart Contracts"
  - "Ethereum"
---

# Intro

I'm going to be doing a series on smart contract development.

The topics I would like to cover in the follow-up post are

- Dev tools and environment
- Create your very own Token/Cryptocurrency
- Build a Decentralized finance (DeFi) application
  - A stretch goal, but hey, they're all the rage right now.

# Getting Started

To get started, we need to establish a common understanding.

For this post, I will be covering the concept of smart contracts.

## What is a smart contract?

A smart contract is a program deployed to the blockchain.

You can think of a smart contract as being very similar to a state machine.

In the case of this blockchain "State Machine", you change the state using transactions.

## Transactions and the EVM

When you interact with a contract via a transaction, the blockchain nodes execute the program using a virtual machine. For Ethereum, this is the Ethereum Virtual Machine or EVM.

This means that Ethereum is a massively distributed compute environment.

However, the nodes don't collectively share the execution of the application - each node runs the entire program independently.

No single node or bad actor can influence the result, and the consensus among the nodes will prevail.

## Transaction Costs

### State Modification

Execution of a smart contract that modifies the state incurs a cost.
Given that the method of interaction is a transaction, this is called the transaction fee.

The fee will vary depending on the code executed.
Things like the types used, the structure, and the efficiency of the code itself all impact the transaction fee.

In the case of Ethereum, the cost of the transaction is called gas. Gas is priced in fractions of an ETH, commonly referred to as Gwei.

If this sounds confusing, think of Gwei as being what cents are to a dollar.
1 Gwei is one nano (one-billionth) ETH, or 1 ETH equals 1 billion Gwei.

This page has a nice comparision of the various [ETH fractions](https://academy.binance.com/en/glossary/gwei)

### Read Only Operations

Read-only interactions do not incur a transaction fee.

## Blockchain Support

Ethereum was the OG smart contract blockchain, but there are a more available now.

## Public vs private blockchains

Some companies would like the benefits of blockchain but want their data to remain private.

### Public Blockchains

- Ethereum
- Binance Smart Chain
- Cardano

{{< admonition type=note title="Note" open=true >}}
Binance Smart Chain is not decentralised
{{< /admonition >}}

### Private Blockchains

- Hyperledger
- Corda
- Quorum

# Development

## Solidity

The programming language you use to program your smart contracts on Ethereum is called Solidity.

[Solidity Doco](https://docs.soliditylang.org/en/v0.8.9/)

## Conclusion

I think that's about is at this stage, this stuff will solidify over the comings posts :-)
