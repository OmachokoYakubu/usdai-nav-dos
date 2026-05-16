# Remediation Strategy: Net Asset Value DoS

## Vulnerability Overview
The current NAV calculation iterates through all pools, ticks, and redemptions. This $O(N)$ approach is unsustainable for a scaling protocol and leads to permanent DoS when the gas limit is exceeded.

## Mitigation Plan

### 1. Incremental NAV Accounting (Recommended)
Instead of calculating NAV from scratch every time, the protocol should maintain a cached `_totalNav` variable in storage.

**Implementation Details:**
1.  Add `uint256 internal _cachedNav` to `StakedUSDaiStorage.sol`.
2.  Update `_cachedNav` during:
    *   `poolDeposit()`: Add the value of the new deposit.
    *   `poolWithdraw()`: Subtract the value of the withdrawn assets.
    *   `Yield Accrual`: Periodically update the NAV to reflect share price increases in the underlying pools.

**Pros:**
*   Reduces `totalAssets()` gas cost to a single $O(1)$ SLOAD.
*   Enables protocol scaling to thousands of positions.

**Cons:**
*   Requires careful handling of yield. Since the underlying pool share prices change independently, the cached NAV will drift.

### 2. The "Hybrid" Approach
Keep the current iteration logic but limit the number of active pools/ticks. If the limit is reached, force older positions to be closed before new ones can be opened.

**Cons:**
*   Artificially limits protocol growth.
*   Doesn't fully solve the problem, only delays it.

### 3. Oracle-Based NAV
Use an off-chain keeper or oracle to push the latest NAV to the contract every few hours.

**Pros:**
*   Most gas-efficient for users.
*   Can handle extremely complex strategies.

**Cons:**
*   Introduces trust in the oracle/keeper.
*   Latency in share price updates (could lead to arbitrage).

## Verification of Fix
After implementing cached NAV, the `test_NAV_DoS_ProtocolBricked` PoC should show near-constant gas consumption (approx. 20,000 - 50,000 gas) regardless of the number of positions simulated.
