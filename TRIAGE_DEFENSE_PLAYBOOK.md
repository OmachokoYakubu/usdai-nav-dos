# Triage Defense Playbook: Net Asset Value DoS

## 1. Triage Classification
*   **Vulnerability Type**: Denial of Service (DoS) / Gas Exhaustion
*   **Severity**: Critical (Permanent Protocol Bricking)
*   **Impacted Component**: `PoolPositionManager.sol` -> `_assets()` / `totalAssets()`

## 2. Evidence of Vulnerability
*   **Location**: `PoolPositionManager.sol#L159` and `L169`.
*   **Proof**: The nested loops iterate over `pools` and `ticks` in every call to `totalAssets()`, which is a gatekeeper for all deposits and withdrawals.

## 3. Anticipated Developer Counter-Arguments
*   *"The number of pools and ticks will be limited by the manager."*
    *   **Defense**: Relying on "Manager Discipline" is not a security guarantee. If the protocol is architecturally capable of bricking itself through normal usage (opening positions), it is a vulnerability. 
*   *"We will just use a high gas limit."*
    *   **Defense**: Blocks have a hard limit (e.g., 30M on Arbitrum). Our PoC shows that just 600 positions can push the gas cost into the millions.

## 4. Developer Masking Analysis
*   **Linear Thinking**: Developers tested with 1-5 positions and observed "acceptable" gas costs. They likely failed to perform **stress testing** or **asymptotic analysis** ($O(N^2)$ behavior).
*   **Separation of Concerns failure**: The protocol mixed "Strategy Management" (high complexity) with "Vault Accounting" (requires low complexity/constant time).

## 5. Critical Invariants to Monitor
*   **Invariant-01**: `totalAssets()` must complete within a constant or logarithmic gas bound relative to the number of users/positions.
*   **Invariant-02**: Withdrawal availability must not depend on the total state of all other strategies.
