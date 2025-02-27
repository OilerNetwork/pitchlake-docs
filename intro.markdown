---
layout: default
title: Intro
nav_order: 1
has_children: true
---

- TOC
{:toc}

## Intro

Pitch Lake is an options market for Ethereum base fee built on Starknet. The repo for the Cairo contracts can be found [here](https://github.com/OilerNetwork/pitchlake_starknet).

## What are options

An option is a financial contract that gives the owner the right, *not the obligation*, to buy or sell an asset at a predetermined price. Options can be used to speculate or hedge against unfavorable price movements.

## Pitch Lake Options

In the context of Pitch Lake, we are referring to cash-settled call options. This means on or after the option round's settlement date, the option buyer (**OB**) has the ability to exercise their options to collect their payout. The **payout** is the difference between the **strike price** of the option round and price of the asset at the round's settlement date (**settlement price**). This means if the strike price is `10`, and the settlement price is `15`, the OB can exercise for `15-10=5`. This is essentially the same as exercising a physically-settled option to purchase base fee at `10` (the strike price) and then selling it for `15` (market price), for a profit of `5` (the payout).  Instead of using the actual price of base fee, Pitch Lake uses the 30 day TWAP on the settlement date. **TWAP** stands for Time Weighted Average Price, and is essentially an average of the basefee over 30 days.

The liquidity posted for a potential payout is locked at the start of an option round's auction. This means there is a cap to the maximum liquidity that can be paid out. This max payout per option is represented as a percentage above the strike price, this percentage is the cap level. In the above example (strike = 10, settlement price = 15), if the cap level is 25%, then instead of the payout being 15-10=5, it is capped to be 25% of 10 = 2.5at most. The total number of options that can be sold in an auction is the option round's starting liquidity divided by the max payout per option. 

### Why buy basefee options

Outside of retail trading and speculation, base fee options could be used by big gas spenders such as oracles and L2s to hedge their exposure to base fee's high volatility.

## Terminology

**LP(s)** - Liquidity provider(s); Referring to the account(s) that are funding/backing/selling the options.

**OB(s)** - Option bidder(s)/buyer(s); Referring to the account(s) that are buying/trading/exercising options.

**Starting Liquidity** - The amount of liquidity locked at the start of a round's auction.

**Unsold Liquidity** - The amount of liquidity that becomes unlocked at the end of a round's auction.

**TWAP** - Time Weighted Average Price; essentially an average price of base fee at a specific date spanning back a specific time interval.

**Settlement Price** - The TWAP of basefee at the settlement date of an option round.

**Strike Price** - The TWAP of base fee at which an option becomes exercisable.

**Cap Level** - The max percentage above the strike price an option will pay up to.

**Max Payout Per Option** - The maximum amount one option may be exercised for; `cap level x strike price`.

**Options Available** - The maximum number of options that can be be sold in the auction; `starting liquidity / max payout per option`

**Options Sold** - The number of options that sold in an auction.

**Reserve Price** - The minimum bid price for one option.

**Clearing Price** - The resulting price for one option after the auction.

**Premium** - The funds spent to purchase the options; total premium is `options sold x clearing price`. 

**Payout Per Option** - The exercisable amount for one option; total payout is `options sold x payout per option`.

**Locked Liquidity** - The liquidity that is locked inside of a vault as collateral for a potential payout when the vault's current round settles.

**Unlocked Liquidity** - The liquidity that is unlocked inside of a vault. This liquidity can be withdrawn until the next option round's auction starts. If it is not withdrawn before, then the unlocked liquidity will become locked for the next round.

**Stashed Liquidity** - The liquidity that is being held by a vault that no longer participates in the protocol. LPs can queue their position for withdrawal while a round is on-going, once the round settles these withdrawals are stashed aside to be collected at any time.

## High-Level

The Pitch Lake protocol is comprised of two contract types; vaults and option rounds. When a vault is deployed, its first options round is automatically deployed. Once a vault's current round is settled, its next round is automatically deployed.

Vaults are responsible for managing liquidity provider positions, and option rounds are responsible for managing the auctioning/minting/exercising of the options. The vault is the primary entry-point for both users (LPs & OBs), and anyone can be a liquidity provider and/or an option buyer.

### Round States

An option round will be in 1 of 4 states: **Open**, **Auctioning**, **Running** or **Settled**. 

- When it is deployed it is **Open**.
- Once its auction starts, it becomes **Auctioning**.
- Once its auction ends, it becomes **Running**.
- Once the round is settled, the state permanently becomes **Settled**.

In the same transaction that a round is settled the next round is deployed, and the vault's current round pointer is updated. This simplifies to: 

- A vault will always have a pointer to its **current round**
- The current round will be either **Open**, **Auctioning**, or **Running**
- All previous rounds will be **Settled**.

## Liquidity

There are three classifications of liquidity inside a vault: locked, unlocked, and stashed.

Unlocked liquidity is free to be withdrawn and will become locked once the next auction starts. Whenever an LP deposits, they are adding to their unlocked balance, and whenever they withdraw, they are taking from their unlocked balance.

Once an auction starts, all unlocked liquidity becomes locked. At this time, OBs can start placing bids for options. During the auction, all bid funds are held by the option round contract. 

Once the auction ends, all accepted bid funds (premiums) are sent to the vault as unlocked liquidity. Any unused bid funds are kept in the option round contract for OBs to refund at any time. If not all of the available options sell in the auction, then some of the locked liquidity becomes unlocked since it is no longer needed as collateral. 

- For example, a round's auction starts with 100 GWEI (locked). This round has a max payout of 2 GWEI per option; meaning there is a total of `100/2 = 50` options available in the auction. Once the auction ends, only 20 of the 50 options sell. This means `2 GWEI x 20 options = 40 GWEI` must remain locked, and the remaining `100 - 40 = 60 GWEI` becomes unlocked.

When the round settles, the total payout is sent from the vault's locked liquidity to the option round. At any point, an OB can exercise their options and claim their payout (even multiple rounds later). The remaining locked liquidity becomes unlocked, and will become locked once the next auction starts. 

This means the LP can either withdraw their funds before the next auction starts, or let them sit and rollover into the next round. However, if the LP knows they are going to withdraw some of their position once the current round settles, then they can queue this withdrawal before hand. This will stash aside their remaining position so that they can collect it at any time, instead of timing their withdrawal before the next auction starts.

- For example, Alice deposits 100 ETH for round 5. While round 5 is on-going, she decides to queue 50% of her position for withdrawal. Round 5 settles with a payout of 20 ETH. This means Alice's remaining liquidity is `100 - 20 = 80 ETH`. 40 ETH becomes stashed, and 40 ETH becomes unlocked. Once round 6 starts, if she has not withdrawn her 40 unlocked ETH, it gets locked for round 6. Her 40 stashed ETH will continue to be held by the vault until she collects it at any time.

### Liquidity Cycle Example

- Alice deposits 100 GWEI into a vault (`locked = 0, unlocked = 100`).
- The next auction starts with 100 options available (max payout = 1 GWEI per option) (`locked = 100, unlocked = 0`).
- Bob buys 33 of the 100 options for 25 GWEI total (`locked = 33, unlocked= 67 + 25 = 92`).
- Alice queues 50% of her position for withdrawal.
- The round settles with a total payout of 30 GWEI (`locked = 33 - 30 = 3, unlocked = 92`).
- 50% was queued for withdrawal, the rest becomes unlocked:
    
    `locked = 3 - 1.5 - 1.5 = 0, unlocked = 92 + 1.5 = 93.5, stashed = 1.5`
    

> **Outcome**: Alice has 1.5 gwei stashed in the vault that she can collect at any time and 93.5 gwei she can withdraw before the next auction starts. Bob has 33 options he can exercise for a total of 30 gwei at any time.
> 

### Tldr;

At the start of a round, liquidity provider funds become locked in the vault as collateral for the options. Option buyers bid for these options in an auction, and once the auction ends, the payments for the options (premiums) and any unsold liquidity are unlocked for liquidity providers. Upon settlement, pricing data from L1 is used to determine the payout of the options. The remaining funds become unlocked or stashed for the liquidity providers, and option buyers can exercise their options in exchange for their payout—proportional to the number of options they own.