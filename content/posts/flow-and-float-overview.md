---
title: "An overview of the Flow blockchain and the Float project"
date: 2022-06-30T11:47:09+03:00
toc: true
tags: [crypto, flow]
GHissueID: 4
---

## Introduction
After some discussions in the chat about Float, I decided to make this short technical overview of the Flow blockchain with some introduction to the Float project. This allows the participants of the chat to quickly understand what is going on under the hood to troubleshoot possible errors without reading the whole documentation and the source code. So, I will refer to this post in further discussions in the related chats.

In this post we will consider:
* The key traits of the Flow blockchain, which set it apart from other networks.
* Float project implementation and common errors which the user may encounter.

This post could be useful for:
* Developers who need a quick introduction to understand what Flow is, before they start reading the technical papers and documentation. I have left links to all the resources used, so after spending a few minutes reading this small post, you will be able to return to them and study everything thoroughly.
* Curious users of Float project with some basic understanding of the blockchain technology, to get the basics of the technical implementation. This will allow to understand possible problems when working with Float, because they will conceptually understand what is going on under the hood.

## The Flow blockchain
Flow is a blockchain network created by [Dapper Labs](https://www.dapperlabs.com/) in 2017. It was created to solve the existing problems of the Ethereum such as high transactions costs and slow transactions.

![Image](/posts/flow-and-float-overview/flow-icon.png)

Flow is convenient for developers and implements some features that solves existing problems and makes the development process less painful. For example, it suggests a new smart-contract programming language, it supports [upgradable smart-contracts](https://docs.onflow.org/cadence/language/contract-updatability/) and a convenient API with [SDK packages](https://docs.onflow.org/sdks/) available for different languages.

### Flow architecture
The most important difference of the Flow blockchain is its pipelined architecture which increases network throughput and scalability in comparison with Ethereum.
The key idea here is to separate the process of ordering and selecting transactions from their computation. In the Flow blockchain, some nodes collect transactions, while other nodes lazily execute collected transactions.
So, Flow introduces a few types of nodes:
* *Collection* nodes implement RPC endpoints used by decentralized applications (dApps). They manage the transaction pool and select transactions which will be submitted to Consensus nodes.
* *Consensus* nodes which are responsible to selecting and ordering of transactions submitted by Collection nodes. The transactions are assembled into blocks that will be finalized using the consensus mechanism.
* *Execution* nodes which lazily execute computation jobs on the finalized blocks.
* *Verification* nodes are used to verify the work of execution nodes. They process executed transactions in parallel to make the verification process faster.
* *Access* nodes work as a proxy to the Flow network. They routes transactions to correct collection nodes.

![Image](/posts/flow-and-float-overview/flow-architecture.gif)

*Picture: Flow architecture ([source](https://www.onflow.org))*

So, this approach allows them to don't use a [sharding](https://101blockchains.com/blockchain-sharding/) mechanism. But it has some limitations and possible issues with scalability, which is described in the paper [Valtchanov et al. – Parallel Block Execution in SoCC Blockchains through Optimistic Concurrency Control](https://www.cs.montana.edu/mwittie/publications/Valtchanov21Parallel.pdf). It also suggests some improvements of this model, so you should read it to understand it better.

### Cadence
Another important feature of the Flow blockchain is their [Cadence](https://github.com/onflow/cadence) smart-contracts programming language.
It syntax is somewhat similar to Rust or JavaScript with [types](https://docs.onflow.org/cadence/language/type-hierarchy/) with addition of blockchain-specific features.

The most interesting feature of the language is the presence of [linear types](https://wiki.c2.com/?LinearTypes) in their type system adapted to the needs of the Flow. Linear types are used to represent valuable crypto assets as resources owned by an account.
This allows the developer to conveniently manage the [capabilities](https://docs.onflow.org/cadence/language/capability-based-access-control/) of the resources transferred between accounts.

Comparing to Ethereum, which must store these resources in a centralized storage – inside smart-contracts – this approach is safer and makes it not possible making errors that lead to vulnerabilities like [re-entrancy attack](https://hackernoon.com/hack-solidity-reentrancy-attack). So it just prohibits some common developer mistakes at the language level.

## Float project
[Float](https://floats.city/) is a proof of attendance platform by [Emerald City DAO](https://www.ecdao.org/). It intended to be used to share unique NFT to participants of any kind of event to allow them to proof that they were there.

![Image](/posts/flow-and-float-overview/float-icon.png)

They use the original idea of [POAP](https://poap.xyz/) but implement it in the Flow blockchain. That allows them to reduce transaction cost and make transactions faster.

### Technical overview
Essentially, it is a dApp which contained by a typical web frontend and a backend present as a smart-contract on the Flow blockchain.

The backend part is present as a set of the Cadence smart-contracts.
The description of FLOAT token is placed in the [FLOAT.cdc](https://github.com/muttoni/float/blob/5421ba35b6/src/cadence/float/FLOAT.cdc) smart-contract. It implements the [NonFungibleToken interface](https://github.com/muttoni/float/blob/5421ba35b6/src/cadence/core-contracts/NonFungibleToken.cdc) that have useful functions to manage NFT assets represented as Cadence resources. All logic related to common operations with FLOATs are present in [scripts](https://github.com/muttoni/float/tree/5421ba35b686f90df5c978a75fb5f86e9e7b7832/src/cadence/float/scripts) and [transactions](https://github.com/muttoni/float/tree/5421ba35b686f90df5c978a75fb5f86e9e7b7832/src/cadence/float/transactions) directories, so you could look there how it works under the hood.

The frontend is written in Svetle and JS, and you could read it in the [lib](https://github.com/muttoni/float/tree/5421ba35b686f90df5c978a75fb5f86e9e7b7832/src/lib) directory. Essentially, it just uses [Flow client libraries](https://docs.onflow.org/fcl/) to communicate with the backend's smart-contracts and contains some additional logic for encrypting and sending passwords to the blockchain so that no one can see the actual data in blockchain transactions.

### Troubleshooting
Now consider a few common errors occurs when working with the Float.
Most creepy error messages are recently [were improved](https://github.com/muttoni/float/commit/5421ba35b686f90df5c978a75fb5f86e9e7b7832), so for now they have more readable messages which makes it easier to understand what is going on.

#### Error 1103: Storage capacity

![Image](/posts/flow-and-float-overview/error-1103.png)

The current error message says: `The account with address X needs more FLOW token in their account (.1 FLOW will be enough).`.

The problem is in [storage capacity](https://docs.onflow.org/concepts/storage/) of the account.
In the Flow blockchain each account can store a certain amount of Cadence resources, depending on the number of FLOW tokens it has.
The exact number of tokens according to the documentation is as follows:
* 100 MB per 1 reserved FLOW
* 10 MB per 0.1 reserved FLOW
* 1 MB per 0.01 reserved FLOW
* 100 kB per 0.001 reserved FLOW

So, in order to store FLOATs, you need to have some $FLOW on your account. And the creator of the event must have at least 0.1 $FLOW to store information about the created event.

How many $FLOW do you need? In the current implementation the size of the single FLOAT is about 1kB. So you could store about 100 FLOATs on your fresh account that have default 0.001 $FLOW. If you need to store more than 100 FLOATs, you should add more tokens to your account.

But you need to have at least [0.003 $FLOW](https://github.com/muttoni/float/blob/9318ef8da68f8acd318c04893b9754816fccf15c/src/cadence/float/transactions/award_manually_many.cdc#L47) so that the event creator can manually send a FLOAT to you.

#### HTTP Request Error

![Image](/posts/flow-and-float-overview/error-http.png)

This issue is [well described](https://github.com/muttoni/float/blob/5421ba35b686f90df5c978a75fb5f86e9e7b7832/src/lib/flow/utils.js#L56) in the updated error message: `Flow Mainnet is currently undergoing maintenance. You can try to refresh the page or try again later.`.
The problem is that the API endpoint from the Flow blockchain doesn't answer. So, you should wait until the problem in the Flow network will be fixed. You could monitor the status of the network [here](https://docs.onflow.org/status/).

## Conclusions
In this post we have considered the most interesting features of the [Flow blockchain](https://www.onflow.org/) and took a look at the [Float](https://floats.city) project. To get the systematic and complete understanding of the internals, you should read their comprehensive [documentation](https://docs.onflow.org/) and technical papers. Now you have the basic understanding of features you may be interested, so you could take a look at the specific parts of documentation interesting to you to solve your problems.

I also hope that now someone will be able to better understand the Float project. Because when you conceptually understand how it is implemented internally, now you will not be afraid of another creepy error.

## Further reading
* [Flow documentation](https://docs.onflow.org/)
* [Flow Primer](https://www.onflow.org/primer) – The official overview of the key features of Flow.
* [Flow technical papers](https://www.onflow.org/technical-paper) – a series of technical papers about the Flow blockchain.
* [Valtchanov et al. – Parallel Block Execution in SoCC Blockchains through Optimistic Concurrency Control](https://www.cs.montana.edu/mwittie/publications/Valtchanov21Parallel.pdf) – limitations of the Flow architecture and suggested improvements.
* [Flow Blockchain vs. Ethereum. Which Is Better for NFT Development?](https://101blockchains.com/flow-blockchain-vs-ethereum/) – a short comparison between Ethereum and the Flow blockchain from the developer perspective.
* [Inside Flow: The Multi-Node Architecture that Scales to Millions](https://www.onflow.org/post/flow-blockchain-multi-node-architecture-advantages) – A comprehensive introduction to Flow architecture.
* [Resource-Oriented Programming: A Better Model for Digital Ownership](https://medium.com/dapperlabs/resource-oriented-programming-bee4d69c8f8e) and [Coblenz – Obsidian: A Safer Blockchain Programming Language](https://src.acm.org/binaries/content/assets/src/2018/michael-coblenz.pdf) – Introduce resource-oriented programming for smart-contracts.
* [Cadence bootcamps by Emerald DAO](https://academy.ecdao.org/) – Courses about fundamentals of Flow and Cadence programming language by Emerald DAO.
