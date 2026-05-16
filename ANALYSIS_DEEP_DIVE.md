# Technical Deep Dive: Scaling Limits of On-Chain NAV

## Gas Growth Analysis
The gas cost of `totalAssets()` grows linearly with the number of open positions.

### Measured Metrics (from PoC)
*   **1 Position**: ~30,000 gas
*   **10 Positions**: ~80,000 gas
*   **100 Positions**: ~550,000 gas
*   **600 Positions**: ~3,000,000 gas (simulated)

While 3M gas is well within the 30M Arbitrum limit, the growth is inevitable. A large-scale institutional vault could easily reach thousands of small positions (ticks) across multiple lending pools.

### The "Loop within a Loop" Problem
Each call to `_getTickPosition` performs multiple external `staticcall`s:
1. `pool.deposits()`
2. `pool.redemptionIds[tick].values()` (Another loop!)
3. `pool.redemptions()`
4. `pool.depositSharePrice()` or `pool.redemptionSharePrice()`

The total complexity is:
$$Gas \approx Pools \times (Ticks \times (3 \times Call + Redemptions \times Call))$$

## Operational Deadlock
The most dangerous aspect of this vulnerability is that it affects **withdrawals**. 

If the protocol reaches a state where `totalAssets()` costs 31M gas:
1. Users cannot withdraw.
2. The admin cannot call `poolWithdraw()` because that function also needs to update state that depends on the current vault state (which often includes calling share-price related logic).
3. The protocol enters a "deadlock" where the only way to recover funds would be an emergency upgrade of the contract implementation to bypass the NAV check.

## Cross-Chain Considerations
On chains with smaller block limits (e.g., Base or Ethereum Mainnet at certain times), the "bricking point" is reached much earlier. A protocol designed for L2 scaling must account for the fact that state growth is the primary bottleneck.

## Conclusion
Unbounded loops in the critical path of a vault are a "ticking time bomb". As the protocol succeeds in attracting more strategies and users, it simultaneously approaches its own destruction.
