+++
title = "Move on IOTA Rebased: Building a Decentralized Exchange Smart Contract. Part 1: Liquidity Book Pools"
date = 2025-02-17
+++

# Introduction

One of the most important services in any financial system is the *exchange*. These facilitate capital to flow between different assets. Most people entering the cryptocurrency world will be familiar with the centralized exchanges (CEX) where you can buy and sell cryptocurrencies. These are companies, third parties, who act as intermediaries. Users pay CEX's a fee for handling the exchange between a seller and a buyer of a particular asset. {pretentious?}

With the rise of cryptocurrencies smart contracts, one of the biggest 

In this article, we will be creating a decentralized exchange with Move for the IOTA Rebased network. This decentralized exchange will be based on the **Liquidity Book** model, created by the [LFJ](https://lfj.gg/) team. 

## Prerequisites

This is an intermediate tutorial aimed at developers wanting to get a deeper understanding of Move. You should have some working familiarity with the basics of the Move language (shared/owned objects, struct abilities, how to publishing modules etc.) If you have never programmed with Move before, I recommend working through [this article](../iota-rebased-sc/) and [this article](../iota-move-raffle-tutorial/) first. These should give you enough understanding to to follow along with here.

To write, test, and deploy our smart contract, we will need:

- [IOTA CLI](https://github.com/iotaledger/iota) - compiling from source takes quite a while, so consider installing one of [the pre-built binaries](https://docs.iota.org/developer/getting-started/install-iota#install-from-binaries)
- Strongly recommended: Using Visual Studio Code with the [IOTA Move extension](https://marketplace.visualstudio.com/items?itemName=iotaledger.iota-move)

## How Decentralized Exchanges Work

- Liquidity providers earning fees
- trades happen whenever the trader wants
- Liquidity Book model, refer to [LFJ docs](https://docs.lfj.gg/concepts/concentrated-liquidity), recommend reading most of the 'concepts'. Explain that we'll be implementing a simplified fee model (in the next article though, not this one)

# A quick note on representing prices

Because we will be exchanging tokens, we will need to be able to represent prices. In particular, we will need to represent *fractional* prices (e.g. IOTA = 0.01 ETH). 

The only fractional number type in the IOTA Move framework is the [`fixed_point32`](https://docs.iota.org/references/framework/move-stdlib/fixed_point32) type. It would probably work for our purposes, but it has limited precision and a limited API. Since I felt it was necessary to have a better fixed-point number type, I created the [`UFP256`](https://github.com/teunvw14/move-fixed-point/blob/main/sources/ufp256.move) type to use in this project. It  works identically to `fixed_point32`, but has higher precision and a more extensive API. 

(You might be familiar with [floating-point numbers](https://en.wikipedia.org/wiki/Floating-point_arithmetic), as they are used in most programming languages to represent fractional values. In contrast, you will most likely never see them in any smart contract programming contexts. The reason for this is that floating-point numbers require a lot more computation than fixed-point numbers, and in smart contracts, more computation mean paying more gas. So floating-point numbers are a no-go.)

# Let's get Move-ing

## Setup 

Start off by creating a new project using the IOTA CLI:

```bash
$ iota move new iota-rebased-dex
$ cd iota-rebased-dex
```

Then make sure to download [`ufp256.move`](https://github.com/teunvw14/move-fixed-point/blob/main/sources/ufp256.move) and place the file into the `sources` directory. This well let you use the `UFP256` type, which we will use for token exchange rates. 