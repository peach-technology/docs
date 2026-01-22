# Huam Docs

## 1. Overview

### 1.1. What is Huam

Huam turns DEX liquidity provision into a simple, yield-bearing stablecoin.

DEX LP positions offer some of the highest yields in DeFi, but capturing them is hard. It requires constant monitoring, active rebalancing, and sophisticated hedging to manage impermanent loss. 

Huam does this for you. The protocol continuously scans pools across multiple DEXs and networks, selects the best opportunities, and manages fully hedged positions. The yield flows back to you, with all the complexity abstracted away.

**Core components:**

- **USDhm**: A synthetic dollar, mintable and redeemable 1:1 with USDC.
- **sUSDhm**: A yield-bearing ERC-4626 token representing staked USDhm.

### 1.2. Why Huam

DEX liquidity provision is one of the most underexploited yield sources in crypto. Yet the vast majority of this yield remains uncaptured. Active management is demanding, impermanent loss is difficult to hedge, and opportunities are fragmented across dozens of DEXs, token pairs, and networks. 

This creates a persistent market inefficiency: high yields exist because few can systematically harvest them. Huam exists to capture this.

Compared to other yield sources commonly used by stablecoin protocols, DEX LP fees sit in a different category:

- **Basis arbitrage**: Funding rates compress as capital enters; yields typically range 5-15% APR
- **RWA-backed stablecoins**: Constrained by off-chain interest rates at 4-8% APR

These strategies prioritize stability and precise delta neutrality. Huam takes a different approach: optimized yield with managed risk. The underlying source is trading activity itself, which means higher return potential but also requires active position management and hedging.

| Value | Description |
| --- | --- |
| **High Yield** | Access to DEX LP fees—among the highest sustainable yield sources in DeFi |
| **Automated Management** | No active position management required; protocol handles rebalancing across pools and exchanges |
| **Risk Management** | Hedged positions minimize exposure to volatility of underlying assets and impermanent loss of LP position |
| **Transparency** | All positions, P&L, and hedging activity verifiable onchain |

## 2. Protocol Architecture

### 2.1. How does Huam Work

![image.png](Huam%20Docs/image.png)

The movement of funds through Huam follows a defined path:

1. **Deposit**: User mints USDhm with USDC. Collateral held in Minter contract.
2. **Stake**: User deposits USDhm into the Staking Vault, receiving sUSDhm. The corresponding USDC collateral transfers from Minter to Collector.
3. **Deployment**: Collector allocates funds between:
    - DEX LP positions (via Adaptors)
    - Exchange margin accounts (for hedging)
4. **Yield Collection**: Revenue from LP fees and funding payments flows to the Reward Manager.
5. **Distribution**: Reward Manager releases yield to the Staking Vault, increasing the sUSDhm exchange rate.

Unstaked USDhm remains fully backed by USDC in the Minter contract, ensuring immediate 1:1 redemption availability. 

Withdrawal from the vault takes reverse of above process. Collector sends corresponding collateral(USDC) to sUSDhm contract, and then directed to Minter contract again when user claims the withdrawal request.

### 2.2. Huam Stablecoin (hmUSD)

![image.png](Huam%20Docs/image%201.png)

Huam adpots a dual-token system to separate stability from yield accrual:

**USDhm** is the base synthetic dollar. It maintains a 1:1 peg with USDC through direct mint and redeem functionality. USDhm itself does not generate yield—it serves as stable collateral and the unit of account within the protocol.

**sUSDhm** is the yield-bearing token. Users stake USDhm into an ERC-4626 vault to receive sUSDhm. As protocol revenue accrues, the exchange rate between sUSDhm and USDhm increases. Users can unstake at any time to receive their proportional share of USDhm plus accumulated yield.

This separation ensures that users who need stability can hold USDhm, while those seeking yield can opt into sUSDhm.

**Minting**

Approved users deposit USDC to mint USDhm at a 1:1 ratio. The USDC collateral is held in the Minter contract until the corresponding USDhm is staked.

**Redeeming**

USDhm can be redeemed for USDC at any time at a 1:1 ratio, minus a 0.1% fee. This fee compensates for operational costs including unwinding hedged positions.

**Access**

- Minting is restricted to whitelisted addresses
- Staking and unstaking sUSDhm is permissionless
- Non-whitelisted users can acquire USDhm on secondary markets and stake freely

### 2.3. Staked hmUSD (shmUSD)

**Staking**

Users deposit USDhm into the Staking Vault and receive sUSDhm based on the current exchange rate. No minimum deposit. Yield accrues automatically—no claiming or compounding required.

**Unstaking**

A 7-day lockup applies to all staked positions. After initiating unstake, users wait 7 days before claiming their USDhm. This lockup:

- Prevents sudden capital flight during market stress
- Allows the protocol to unwind positions in an orderly manner
- Protects remaining stakers from forced liquidations at unfavorable prices

After the lockup period, users can claim USDhm at any time. Redemption to USDC is always available without delay.

- Reward Distribution
    - target APY = (current collateralization ratio - desired collateralization ratio) * rewad distribution interval / 365
    - actual APY = last distribution APY * smoothing factor + target APY * (1 - smoothing factor)

### 2.4. Smart Contract Architecture

The protocol is deployed across a main network of the protocol and multiple EVM chains where LP opportunities exist.

**Main Network**

| Contract | Function |
| --- | --- |
| **Minter** | Handles mint/redeem of USDhm against USDC collateral. Transfers collateral to Collector when USDhm is staked. |
| **Staking Vault** | ERC-4626 vault for USDhm → sUSDhm conversion. Receives yield from Reward Manager. |
| **Reward Manager** | Buffers incoming yield, controls distribution velocity, absorbs short-term losses. |

**Per-Chain Deployment**

| Contract | Function |
| --- | --- |
| **Collector** | Holds deployed assets. Routes funds to LP positions and exchange margin accounts. Handles cross-chain bridging and swaps. |
| **Adaptors** | DEX-specific interfaces (e.g., Uniswap V3 Adaptor, Aerodrome Slipstream Adaptor). Each holds LP position NFTs and executes position management. Only callable by Collector. |

This modular design allows Huam to deploy to any EVM-compatible network and integrate with any concentrated liquidity DEX without modifying core protocol contracts.

### 2.5. Layered Protection

- Reserve → Reward Manager → Staked Assets

The Reward Manager acts as a buffer between raw protocol earnings and user distributions. This serves two purposes:

- **Reward Smoothing**: DEX LP returns are inherently variable. The Reward Manager accumulates excess yield during high-revenue periods and maintains distribution during lower periods, reducing volatility in sUSDhm APY.
- **Loss Absorption**: If positions incur short-term losses (from hedging costs, rebalancing, or adverse market conditions), the buffer absorbs these before they affect stakers. Only if accumulated losses exceed the buffer would a loss be passed to the vault.

- Staked Asset = Debt in point of view of protocol
- Peg stability of USDhm is not affected at all by risks of portfolio management of Huam (regardless of shmUSD staker’s loss)

## 3. Protocol Mechanism

### 3.1. PnL Profile of a DEX LP Position

This chapter explains how Huam hedges DEX LP positions to generate managed yield while minimizing risk and volatility from underlying assets. It assumes familiarity with Uniswap V3 style concentrated liquidity LP positions and basic options concepts.

---

To understand Huam's hedging approach, let's walk through a concrete example.

Assume we enter an ETH-USDC pool on Uniswap V3 with a symmetric ±10% price range centered on the current price. Our initial deposit is 1 ETH and 3,000 USDC (assuming ETH price is $3,000).

This LP position earns trading fees from swaps that occur within our range. However, it is also exposed to **impermanent loss (IL)**—the difference in value between holding the LP position versus simply holding the original assets. As price moves away from the entry point in either direction, IL increases.

![image.png](Huam%20Docs/image%202.png)

![image.png](Huam%20Docs/image%203.png)

*Figure: PnL profile of LP position vs. price]*

The most intuitive way to remove directional exposure is to add a short position on the underlying asset. If we short 1 ETH, we eliminate the risk of ETH price movements affecting our portfolio value—at least in a linear sense.

However, even with this short hedge in place, the portfolio still experiences losses when price moves within the price range. This happens because impermanent loss has a **nonlinear, concave profile**. A static short position, which moves linearly with price, cannot fully offset this curved loss profile.

For a symmetric price range, the maximum impermanent loss at either boundary is approximately 25% of the range width.

$$
IL_{max} \approx \frac{r}{4}
$$

where $r$ is the price range width. For a ±10% range, expect ~2.5% loss if price reaches either bound.

This is the fundamental dynamic of DEX LP positions. Evaluating profitability comes down to a simple comparison, where the position is profitable when fee revenue exceeds impermanent loss, and unprofitable when fees fall short.

### 3.2. Hedging Impermanent Loss with Options

The challenge with hedging impermanent loss is its nonlinearity and convexity. Spot positions and perpetual futures move linearly with price—they gain or lose value in direct proportion to price changes. Impermanent loss, by contrast, is convex: the loss accelerates as price moves further from the entry point.

This mismatch means linear instruments alone cannot neutralize IL. We need an instrument that also has convexity—one that gains value at an accelerating rate as price moves in either direction.

**Options provide exactly this property.** For a symmetric LP range, a **straddle position** (buying both an at-the-money put and an at-the-money call) creates a payoff profile that mirrors the shape of impermanent loss. As price moves away from the strike in either direction, the straddle gains value, offsetting the IL incurred by the LP position.

![image.png](Huam%20Docs/image%204.png)

*[Figure: Combined PnL of LP + Short + Straddle]*

When we combine all three components—the LP position, the short perpetual hedge, and the straddle—the resulting portfolio has a flattened PnL profile across a wide range of prices. The position captures fee revenue while neutralizing both directional exposure (via the short) and impermanent loss (via the straddle).

Unlike a portfolio with only a short hedge, this fully hedged position does not suffer losses when price moves to the range boundaries. The options payoff compensates for the IL that the linear short cannot cover.

**Selecting Option Maturity**

However, introducing options requires us to specify a maturity date, which brings us to the next consideration.

To determine the appropriate option tenor, we estimate how long the LP position is expected to remain in range. If price exits the range, the LP position stops earning fees and must be closed or repositioned.

Assuming the underlying asset follows a driftless geometric Brownian motion (GBM), we can derive the expected time to reach a given price boundary. For a return level rr
r (the distance to the range boundary as a percentage), the expected time is:

$$
T = \frac{[\ln(1+r)]^2}{\sigma^2}
$$

where $\sigma$ is annualized volatility.

**Example**: ETH volatility = 60%, price range = ±20%

$$
T = \frac{[\ln(1.2)]^2}{0.6^2} = \frac{0.0332}{0.36} \approx 0.092 \text{ years} \approx 34 \text{ days}
$$

This calculation guides our option maturity selection: we match the option tenor to the expected duration of the LP position. Tighter ranges mean shorter expected durations and therefore shorter-dated options.

**Evaluating Profitability**

With the hedging framework established, the profitability condition becomes straightforward:

$$
\text{LP Fee Revenue} + \text{Funding Income} > \text{Straddle Premium}
$$

If the fees earned from the LP position plus any funding payments received from the short perpetual exceed the cost of the options hedge, the position is profitable.

It's important to note that adding options does not magically create yield where none exists. If a pool's fee revenue is fundamentally insufficient relative to the volatility of its underlying assets, the hedge simply reshapes the risk profile without improving the economics. The option premium reflects the expected cost of the volatility exposure we're hedging away.

However, this framework provides two significant benefits:

1. **Variance reduction**: By hedging IL, we dramatically reduce the volatility of returns. The position generates more consistent yield across different price paths, rather than being highly profitable in some scenarios and deeply unprofitable in others.
2. **Rigorous evaluation**: Using established option pricing models allows us to evaluate opportunities on a sound theoretical basis. Rather than naively selecting pools with the highest displayed APY, we can compare fee revenue against the true cost of the associated risks.

### 3.3. Replicating Options

The hedging approach described above requires options with convex payoffs. In traditional markets, we would simply purchase these options from an exchange or dealer. In crypto, this is often not practical.

Crypto options markets are illiquid and limited to a handful of major assets—primarily BTC and ETH. Even for these assets, liquidity is concentrated at specific strikes and expirations, and spreads can be wide.

For exotic token pairs like CRV-WETH or BRETT-cbBTC, options markets simply don't exist. There is no venue where we could purchase the straddles needed to hedge these positions.

![Source: Unified Approach for Hedging Impermanent Loss of Liquidity Provision, Alex Lipton et al.](Huam%20Docs/image%205.png)

Source: Unified Approach for Hedging Impermanent Loss of Liquidity Provision, Alex Lipton et al.

Rather than buying options directly, Huam replicates the desired option payoffs using perpetual futures. This approach, known as delta hedging or dynamic replication, is a well-established technique from traditional finance.

The core idea is that any option's payoff can be approximated by continuously adjusting a position in the underlying asset (or a derivative like a perpetual) to match the option's delta—its sensitivity to price changes.

Using an option pricing model such as Black-Scholes, we calculate the theoretical delta of the straddle at any given price and time. We then maintain a perpetual position sized to match this delta. As price moves and time passes, the delta changes, and we rebalance the perpetual position accordingly.

For a straddle, the delta starts near zero when price is at the strike (the ATM puts and calls have offsetting deltas). As price moves away from the strike, the delta shifts: positive if price rises (the call dominates), negative if price falls (the put dominates). Near expiry, these delta shifts become more pronounced.

By continuously adjusting our perpetual position to track these delta changes, we replicate the option's payoff without ever holding the actual option.

**Tradeoffs and Considerations**

Replication is not a perfect substitute for holding actual options:

- **Path dependency**: The cost of replication depends on the realized path of the underlying price, not just the final outcome. Frequent large moves (high realized volatility) increase rebalancing costs.
- **Discrete rebalancing**: In theory, perfect replication requires continuous adjustment. In practice, we rebalance at discrete intervals, introducing some tracking error.
- **Transaction costs**: Each rebalancing trade incurs fees and potentially slippage, which add up over the life of the position.
- **Model risk**: The replication is based on theoretical option deltas. If the model assumptions don't match reality, the hedge may be imperfect.

Despite these limitations, replication using perpetuals is often more cost-effective than purchasing options—particularly when implied volatility is elevated relative to subsequent realized volatility. It also enables hedging on assets where no other solution exists.

In practice, applying the option delta to our portfolio often results in a net position close to the LP's underlying token amounts, providing intuitive alignment between the hedge and the exposure.

### 3.4. Portfolio Allocations

Huam continuously scans pools across multiple DEXs and networks to identify the most attractive opportunities. The allocation process combines quantitative estimation with systematic risk management.

For each candidate pool, we estimate two key inputs:

1. **Fee APR**: Projected based on historical trading volume, current liquidity depth, and the fee tier of the pool. Volume and liquidity fluctuate, so we use statistical methods that account for mean reversion and recent trends.
2. **Underlying Volatility**: Estimated using models like HAR (Heterogeneous Autoregressive), which capture volatility dynamics across multiple time horizons. Accurate volatility estimation is critical because it directly determines the cost of hedging.

With these estimates, we calculate the expected return for each pool:

The hedge cost corresponds to the theoretical option premium required to neutralize impermanent loss. This premium is proportional to the volatility of the underlying assets—higher volatility means more expensive hedging.

A pool with 50% APR but 80% underlying volatility may be less attractive than a pool with 25% APR but 30% volatility. The framework allows us to make these comparisons on a consistent, risk-adjusted basis.

**Position Selection and Diversification**

Based on expected returns, we:

1. Rank all candidate pools by risk-adjusted expected return
2. Select the top N pools for deployment, where N scales with total protocol AUM
3. Apply concentration limits per pool, per DEX, and per network to manage idiosyncratic risks

Diversification across multiple pools smooths returns and reduces the impact of any single position underperforming.

## 4. Resources

### 4.1 FAQ

- **How does USDhm maintain its peg?**
    
    USDhm is minted and redeemed 1:1 with USDC. There is no algorithmic mechanism or fractional reserve—every USDhm in circulation is backed by USDC held in protocol contracts. The only trust assumptions are USDC's own peg stability and the security of the underlying blockchain.
    

---

- **Can I lose money staking sUSDhm?**
    
    Yes. Huam runs an active trading strategy with inherent risks. While positions are hedged, costs can exceed revenue in certain conditions: hedging costs during high volatility, estimation errors in pool selection, or transaction costs during rebalancing.
    
    Short-term losses are absorbed by the Reward Manager buffer. However, if losses exceed the buffer, they are passed to the Staking Vault, reducing the sUSDhm exchange rate. Stakers should understand this is a managed yield strategy, not a risk-free return.
    

---

- **Who can mint USDhm?**
    
    Minting is restricted to whitelisted addresses. Non-whitelisted users can acquire USDhm on secondary markets (e.g., DEX pools) and stake it permissionlessly. Staking, unstaking, and redeeming are open to all users.
    

---

- **Can I withdraw at any time?**
    
    Unstaking sUSDhm requires a 7-day lockup period. After initiating unstake, you wait 7 days before claiming USDhm. Once you have USDhm, redeeming to USDC is available immediately with no delay, subject to a 0.1% fee.
    

---

- **Is Huam audited?**
    
    Huam is currently in MVP phase. Smart contracts have not been fully audited. Users should not interact with the protocol without understanding the associated implementation risks. Audit status will be updated as the protocol matures.
    

---

- **How is this different from other yield-bearing stablecoins?**
    
    Most yield-bearing stablecoins derive returns from basis arbitrage (funding rates) or off-chain assets (T-bills, money markets). Huam sources yield from DEX LP fees—a higher but more variable yield source. The tradeoff is increased complexity in position management and hedging, which the protocol handles automatically.
    

---

- **How can I verify protocol holdings and performance?**
    
    All LP positions, hedge positions, and P&L are verifiable onchain. The Collector and Adaptor contracts hold protocol assets directly—users can inspect positions, collateral levels, and historical performance at any time. Exchange margin balances used for hedging are held through OES (Off-Exchange Settlement) providers, keeping assets in custody while delegating trading access to exchanges.
    

### 4.2 Risks

참고 - [Midas Risk Disclosure](https://docs.midas.app/resources/risk-disclaimer)

- Third Party Risks - Failure of Infrastructure providers, exhcnages
- Strategy Risks
- Market Risks
- Execution Risks
- Operational Risks
- Liquidation Risks

### 4.3 Contract Addresses

TBD