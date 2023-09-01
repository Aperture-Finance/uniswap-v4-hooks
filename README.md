# Uniswap V4 Hooks

## JIT (Just-In-Time) Liquidity for Equitable Fee Distribution and Fair Trader Experience Hook

### User Story
As a liquidity provider (LP) on Uniswap, there can be two different roles:
- Passive LP
- JIT LP

As a passive LP, I want to offer liquidity in a passive manner and have a fair share of trading fee allocation.
As a JIT LP, I want to provide liquidity in a JIT manner and I don't want to be penalized too much if I'm helping traders to reduce price impact.


As a trader, I want to trade in a low price impact and greater liquidity environment.

Acceptance Criteria:
- Can balance and redistribute trading fees between different types of LPs.
- Traders experience minimal price impact and slippage.
- Liquidity readjustments should occur without major disruptions to the overall trading experience on Uniswap.

### Technical Discussion
The core issue of JIT LP vs. other types of LP is how to distribute trading fees in a equitable manner. We are proposing two potential technical directions to kick things off:

**Approach 1: Same-block fee redistribution**

The high level idea is to identify JIT add/remove transactions and redistribute its captured fee in a equitable way within in-range LPs.

With `beforeModifyPosition` and `afterModifyPosition` hooks, we will be able to keep track of when a position is added and removed. And we are also able to keep track of fees earned for such sequence of add/remove transactions. The detailed discussion of how accounting is done will be left to detailed engineering design doc later. Once we can calculate the associated JIT fees, a set of customizable knobs can be added to redistribute fees.

**Approach 2: Trading fee as liquidity mining rewards with governance**
Similar to the MasterChef contract from SushiSwap, instead of emitting farming tokens, trading fee, composed of two tokens, can be used as the liquidity reward to be distributed based on a few factors:
- Emission rate per block: this refers to how many tokens (two tokens in the trading pair) are given out per block.
- Allocation points: this defines who gets how much. Not all liquidity positions are the same. And this can be a good way to balance between passive LPs and JIT LPs.

For the 2nd approach, we can implement the core logic within a hook contract and intercept the trading fee to route to hook contract. Each `beforeModifyPosition()` and/or `afterModifyPosition` hook can help updating the state of trading fee distribution.

As for the knobs from 2nd approach, we can integrate a governance mechanism to fairly and democratically distribute the rewards.

The 1st approach may sound technically cleaner but the inner-workings of fee tracking can be more complicated than 2nd approach. More time would be needed to research and narrow down the optimal approach.

## Hook-based LP Automation
Aperture built and launched first of its kind non-custodial LP tooling to help with position rebalance, compound and limit order. With the help of hooks, we will be able to incorporate automation logic natively within Uniswap.

### User Story
As a LP, I want to have a smooth and low effort experience when providing and managing liquidity.
As a trader, I want to trade with food liquidity and low gas cost.

Acceptance Criteria:
- Can automate LP positions without involving off-chain infrastructure and do so natively within Uniswap.
- Won't incur high gas cost to traders.

### Technical Discussion
LP position management along with limit order can all be automated by creating a set of internal states to keep track of which position needs servicing. For instance, to automate position rebalance, we can:
- Keep track of position id to rebalance as well as its desired future position range in our hook contract: this will be an `external` function.
- In `afterSwap` hook, each tracked position will be scanned and rebalanced based on condition specified by the position owner. This step will incur extra gas cost for traders.

To offset extra gas paid by traders, we proposal a way to deduct a portion of the position being rebalanced to compensate traders. With this incentive mechanism, it's more likely to keep the pool price to be in align with external market price.

## Institutional Grade Liquidity Permission System

### User Story
As a liquidity provider, I would like to have finer granularity of control over positions delegated to a professional position manager and do so gaslessly. For instance, the current permit from Uniswap's `NonfungiblePositionManager` can either
- allow me to approve a single position to be automated by an external party, or
- allow me to assign an external party as an `operator` which grants that operator access to all of my positions.

The current permission system can be too narrow or too broad.

### Technical Discussion

We are proposing multiple improvements to existing `NonfungiblePositionManager` and move the permission system into hook contracts. Specifically,
- Hook contract will be the owner of pool liquidity: i.e. hook owned liquidity
- Utilize `before/afterModifyPosition()` hook to identify same-tx position transition and inherit permission from parent position. For instance, if position A is rebalanced into B, we'll track this and let B to inherit permissions from A.
- Implement more sophisticated permission functions to merge/prune multiple inheritance sequence
- Bake automation condition, such as price, into the gasless permit signing process to offer more security for institutions.
