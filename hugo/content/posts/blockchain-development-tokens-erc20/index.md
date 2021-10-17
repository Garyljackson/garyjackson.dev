---
title: "Smart Contract Development - Tokens"
summary: "What are ERC20 tokens, and how to create them"
date: "2021-10-17T11:00:00+10:00"
lastmod: "2021-10-17T11:00:00+10:00"
author: Gary Jackson
draft: false
categories:
  - "Development"
tags:
  - "Smart Contracts"
  - "Ethereum"
  - "Blockchain"
  - "ERC20"
---

# Intro

Tokens are a central concept within the crypto ecosystem.

Most of the currencies you see listed on places like [CoinMarketCap](https://coinmarketcap.com/) are essentially smart contracts.

Overwhelmingly these contracts are created on the Ethereum blockchain, but they're not exclusive to Ethereum.

# Tokens

Generally speaking, a token is a smart contract that adheres to a specific standard established through the Ethereum Request for Comments process (ERC)

Simply put, for a token to be considered an ERC20 token, it needs to implement specific predefined functionality defined within the ERC20 standard.

For any developer, ERC20 is an interface that defines the contract.

## Motivation

Having a standard allows for increased interoperability

For example, trading your tokens on existing trading platforms or decentralised exchanges.

## ERC Standards

**Here are the standards you've probably heard about**

- ERC20, used for fungible assets like tokens
- ERC721, used for non-fungible assets, more popularly known as NFT's
- ERC777, an improved version of the ERC20, with backwards compatibility.

Many of these standards are also supported by competing blockchain implementations.

## ERC20 Interface Definition

The ERC20 interface is defined as follows.

```solidity
pragma solidity ^0.6.0;

interface IERC20 {

    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);

    function transfer(address recipient, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);


    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

```

## OpenZeppelin

OpenZeppelin is an organisation that, among other things, publishes a collection of battle-tested contract implementations.

From the OpenZeppelin page:
`OpenZeppelin Contracts helps you minimise risk by using battle-tested libraries of smart contracts for Ethereum and other blockchains.`

Just about any smart contract you interact with these days will likely be based on the work freely available from OpenZeppelin.

I would strongly suggest that you leverage existing code and learnings from this project if you're looking to get into this space.

## Basic, Example token

For completeness, I've included the example basic ERC20 implementation provided by Ethereum.

```solidity
pragma solidity ^0.6.0;

interface IERC20 {

    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);

    function transfer(address recipient, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);


    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}


contract ERC20Basic is IERC20 {

    string public constant name = "ERC20Basic";
    string public constant symbol = "ERC";
    uint8 public constant decimals = 18;


    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);
    event Transfer(address indexed from, address indexed to, uint tokens);


    mapping(address => uint256) balances;

    mapping(address => mapping (address => uint256)) allowed;

    uint256 totalSupply_;

    using SafeMath for uint256;


   constructor(uint256 total) public {
    totalSupply_ = total;
    balances[msg.sender] = totalSupply_;
    }

    function totalSupply() public override view returns (uint256) {
    return totalSupply_;
    }

    function balanceOf(address tokenOwner) public override view returns (uint256) {
        return balances[tokenOwner];
    }

    function transfer(address receiver, uint256 numTokens) public override returns (bool) {
        require(numTokens <= balances[msg.sender]);
        balances[msg.sender] = balances[msg.sender].sub(numTokens);
        balances[receiver] = balances[receiver].add(numTokens);
        emit Transfer(msg.sender, receiver, numTokens);
        return true;
    }

    function approve(address delegate, uint256 numTokens) public override returns (bool) {
        allowed[msg.sender][delegate] = numTokens;
        emit Approval(msg.sender, delegate, numTokens);
        return true;
    }

    function allowance(address owner, address delegate) public override view returns (uint) {
        return allowed[owner][delegate];
    }

    function transferFrom(address owner, address buyer, uint256 numTokens) public override returns (bool) {
        require(numTokens <= balances[owner]);
        require(numTokens <= allowed[owner][msg.sender]);

        balances[owner] = balances[owner].sub(numTokens);
        allowed[owner][msg.sender] = allowed[owner][msg.sender].sub(numTokens);
        balances[buyer] = balances[buyer].add(numTokens);
        emit Transfer(owner, buyer, numTokens);
        return true;
    }
}

library SafeMath {
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
      assert(b <= a);
      return a - b;
    }

    function add(uint256 a, uint256 b) internal pure returns (uint256) {
      uint256 c = a + b;
      assert(c >= a);
      return c;
    }
}


```

## Related Links

[OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts)

[Basic ERC token example from Ethereum](https://ethereum.org/en/developers/tutorials/understand-the-erc-20-token-smart-contract/)

[More on the Ethereum Improvement process](https://ethereum.org/en/eips/)

[ERC20 Standard](https://eips.ethereum.org/EIPS/eip-20)

[A list of ERC's in their various phases](https://eips.ethereum.org/erc)
