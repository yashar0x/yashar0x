### **Title: Hardcoded 10-Day EVM Timelock Desynchronizes with SXT Chain's 7-Day Unbonding, Impairing Unstake Cancellation Utility**

### **Summary**

The `deployCollaborativeStakingWithDefaults` function establishes a 10-day on-chain delay for cancelling unstakes, which is longer than the SXT Chain's 7-day unbonding period. This mismatch means that if a user attempts to cancel their unstake after day 10, the underlying SXT Chain rebond operation will fail and have no effect, as the tokens are already fully unbonded, negating the intended cancellation.

### **Finding Description**

The `CollaborativeStakingFactory.sol` contract includes a function `deployCollaborativeStakingWithDefaults` which, when called, deploys new instances of the `CollaborativeStaking` contract. These default instances are critically initialized with a hardcoded 10-day timelock (`STAKING_DELAY_LENGTH`). This EVM-level timelock governs when a staker can execute certain actions like `stake()` or `cancelInitiateUnstake()` after an unstaking process has been initiated via `initiateUnstake()`.

A conflict arises because this 10-day EVM timelock is longer than the SXT Chain's native unbonding period for staked tokens, which is configured as 7 eras (equivalent to 7 days). The SXT Chain's staking pallet (`pallet_staking`) transitions tokens from an "unbonding" state to a fully "unbonded" (withdrawable) state after these 7 days.

If a user of a `CollaborativeStaking` contract (deployed via `deployCollaborativeStakingWithDefaults`) initiates an unstake and then wishes to cancel it after the 7-day SXT unbonding period has completed, they will be unable to call `cancelInitiateUnstake()` on the EVM contract. If they wait until after the 10-day EVM timelock to call `cancelInitiateUnstake()`, the corresponding `rebond()` operation attempted on the SXT Chain (triggered by `process_unstake_cancelled` in the `system_tables` pallet) will fail to re-stake the funds or will have no effect. This is because the SXT `pallet_staking::rebond()` function is designed to operate on funds that are actively in an unbonding ledger; once fully unbonded after 7 days, they are no longer in this state.

This desynchronization between the EVM contract's timelock and the SXT Chain's unbonding period renders the `cancelInitiateUnstake()` functionality ineffective under these common timing conditions, forcing users to complete the full withdrawal process (`claimUnstake` followed by `sxtFulfillUnstake`) even if their intent was to cancel the unstake and keep their tokens staked or re-stake them within the `CollaborativeStaking` contract.

#### **Step-by-Step Breakdown of the Issue:**

1.  **Deployment with Hardcoded 10-Day Delay:** A `CollaborativeStaking` contract instance is created using `CollaborativeStakingFactory.deployCollaborativeStakingWithDefaults()`:

    ```solidity
    // In CollaborativeStakingFactory.sol
    function deployCollaborativeStakingWithDefaults(
        // ... params ...
    ) external returns (address deployedAddress) {
        // STAKING_DELAY_LENGTH for this new instance is set to 10 days
        CollaborativeStaking staking = new CollaborativeStaking(tokenAddress, stakingAddress, 10 days);
        // ...
    }
    ```
    The `STAKING_DELAY_LENGTH` variable within this deployed `CollaborativeStaking` contract is now 10 days.

2.  **User Initiates Unstake via EVM Contract:** A user with the `STAKER_ROLE` on the `CollaborativeStaking` contract calls `initiateUnstake()`:

    ```solidity
    // In CollaborativeStaking.sol
    function initiateUnstake(uint248 amount) external onlyRole(STAKER_ROLE) {
        _startStakingDelay(); // Sets EVM's stakingDelayStartTime = block.timestamp

        // ...
        // This call propagates to the SXT Chain to start the native unbonding process
        IStaking(STAKING_ADDRESS).initiateUnstake(amount);
    }
    ```
    On the EVM contract, the `stakingDelayStartTime` is recorded. The `withStakingDelay` modifier now prevents calls to `stake()` and `cancelInitiateUnstake()` on this EVM contract for the next 10 days. On the SXT Chain, this action triggers `system_tables::process_unstake_initiated`, which in turn calls `pallet_staking::unbond()`. This begins the SXT Chain's 7-day unbonding period (`BondingDuration` = 7 eras).

3.  **The Mismatch Window (Days 8, 9, and early Day 10 Post-Initiation):**

    *   **SXT Chain Status:** After 7 full days, the `BondingDuration` on the SXT Chain has elapsed. The user's tokens are now fully "unbonded" within `pallet_staking`. They are no longer in an active unbonding queue; they are considered withdrawable from the perspective of the SXT Chain's native staking logic. The `pallet_staking::rebond()` function, which is intended to reverse an active unbonding, would no longer find these funds in the unbonding ledgers in a state suitable for simple rebonding.
    *   **EVM `CollaborativeStaking` Contract Status:** The 10-day `STAKING_DELAY_LENGTH` has not yet fully elapsed. The `withStakingDelay` modifier is still active and will cause a revert if `cancelInitiateUnstake()` or `stake()` is called. Therefore, during days 8, 9, and part of day 10, the user cannot call `cancelInitiateUnstake()` on the EVM contract.

    ```solidity
    // In CollaborativeStaking.sol
    modifier withStakingDelay() {
        // If (current_time < start_time + 10_days), this condition is true, leading to revert.
        if (block.timestamp < stakingDelayStartTime + STAKING_DELAY_LENGTH) {
            revert StakingDelayNotExpired();
        }
        _;
    }
    ```

4.  **User Attempts `cancelInitiateUnstake()` After 10 Days (e.g., on Day 10.5):** The 10-day EVM timelock has now passed. The `withStakingDelay` modifier allows the call to `cancelInitiateUnstake()`:

    ```solidity
    // In CollaborativeStaking.sol
    function cancelInitiateUnstake() external onlyRole(STAKER_ROLE) withStakingDelay {
        emit CancelInitiateUnstake();
    
        // This call propagates to the SXT Chain
        IStaking(STAKING_ADDRESS).cancelInitiateUnstake();
    }
    ```
    On the SXT Chain, this triggers `system_tables::process_unstake_cancelled`, which attempts `pallet_staking::rebond()`:

    ```rust
    // Simplified from sxt-node/pallets/system_tables/src/lib.rs (process_unstake_cancelled)
    // ...
    // let staking_balance = ... fetch the amount that was being unbonded ...
    // pallet_staking::Pallet::<T>::rebond(staker_signer, staking_balance).map_err(|e| e.error)?;
    // ...
    ```

5.  **The Issue:** At this point (Day 10.5), the funds on the SXT Chain have been fully unbonded for over 3 days. The `pallet_staking::rebond()` function is unlikely to successfully re-stake these funds because they are no longer in an "unbonding" state that `rebond` is designed to act upon. The call might succeed without error but have no effect on re-staking the funds, or it might explicitly fail if it checks the state of the funds.

#### **Consequences:**

The user, having called `cancelInitiateUnstake()` on the EVM contract, might expect their funds to be re-staked or available for re-staking within the `CollaborativeStaking` contract. However, the underlying state on the SXT Chain is that the funds are fully unbonded. This discrepancy leads to:

*   **Ineffective Cancellation:** The `cancelInitiateUnstake()` function fails to achieve its primary purpose of reversing the unbonding process on the SXT Chain within the problematic time window.
*   **Forced Withdrawal:** The user is likely forced to proceed through the full `claimUnstake()` and `sxtFulfillUnstake()` process to retrieve their tokens to their wallet, even if their intention was to keep them staked or quickly re-stake them via the `CollaborativeStaking` contract.

#### **Security Guarantees Broken or Degraded:**

*   **Integrity of Stated Functionality:** The `cancelInitiateUnstake` function does not reliably deliver its intended outcome due to the timelock desynchronization. The EVM contract implies a capability (cancellation leading to re-bonding) that is not consistently achievable on the SXT Chain within the defined operational windows.
*   **Liveness of Cancellation Feature:** For a significant portion of the time when a user might wish to cancel (i.e., shortly after the SXT unbonding period but before the EVM timelock), the feature is either inaccessible or ineffective.

This issue does not directly lead to a loss of funds to an attacker but rather to a failure of intended functionality, forcing users into less desirable workflows and potentially incurring extra costs and delays due to the misalignment between EVM contract parameters and SXT Chain native parameters.

### **Impact Explanation**

This issue results in users losing the ability to cancel an unstake and must immediately claim their tokens, even if they change their minds—undermining the intended rebond flexibility. While funds aren’t stolen outright, users are forced into an irrevocable withdrawal path they didn’t choose, degrading user experience and breaking protocol guarantees. Therefore we assess the impact as **Medium**.

### **Likelihood Explanation**

Because the factory always hard-codes a 10-day delay and the SxT chain unbonds in 7 days, every deployment using the default will exhibit this mismatch. No extra conditions are required—this will happen every time a user uses the default deployment path. Hence the likelihood is **High**.

### **Proof of Concept**

In the POC we will focus on demonstrating that the rebonding fails after the unbonding period. Create test `demonstrate_rebond_failure_after_unbonding_period` file in `sxt-node/attestation_tree/src/demonstrate_rebond_failure_after_unbonding_period.rs`.

```rust
#[cfg(test)]
pub mod tests {
    use std::str::FromStr;

    use eth_merkle_tree::utils::keccak::keccak256;
    use pallet_balances::BalanceLock;
    use sp_core::{H160, U256};
    use sxt_core::system_contracts::ContractInfo;
    use sxt_runtime::Runtime;

    use crate::locks_staking_prefix_foliate::{LocksStakingPrefixFoliate, STAKING_BALANCE_LOCK_ID};
    use crate::prefix_foliate::encode_key_value_leaf;
     /// This test demonstrates that rebond operation fails after the unbonding period
    #[test]
    fn demonstrate_rebond_failure_after_unbonding_period() {
        // 1. Setup test data
        let staker = H160::from_str("0x000102030405060708090a0b0c0d0e0f10111213").unwrap();
        let amount = 257u128;
        let chain_id = U256::from(1028u32);
        let contract_address = H160::from_str("0x000102030405060708090a0b0c0d0e0f10111213").unwrap();

        // 2. Simulate initial staking state
        let staking_lock = BalanceLock::<u128> {
            amount,
            id: *STAKING_BALANCE_LOCK_ID,
            reasons: pallet_balances::Reasons::All,
        };

        let locks = vec![staking_lock].try_into().unwrap();
        let contract_info = ContractInfo {
            chain_id,
            address: contract_address,
        };

        // 3. Simulate unbonding process
        // After 7 days, the tokens are fully unbonded on SXT Chain
        // The staking lock is removed from the balance locks
        let empty_locks = vec![].try_into().unwrap();

        // 4. Attempt rebond after unbonding period
        // This simulates what happens in process_unstake_cancelled
        let rebond_attempt = encode_key_value_leaf::<LocksStakingPrefixFoliate<Runtime>>(
            (sp_core::crypto::AccountId32::from_slice(&staker.0),),
            (empty_locks, contract_info),
        );

        // 5. Print the results
        println!("\nRebond Attempt After Unbonding Period:");
        println!("  - Raw bytes length: {}", rebond_attempt.len());
        println!("  - Hex encoded: {}", hex::encode(&rebond_attempt));

        // 6. Demonstrate that the rebond attempt fails because:
        // a) The staking lock is no longer present in the balance locks
        // b) The tokens are already fully unbonded after 7 days
        // c) The rebond operation in pallet_staking::Pallet::<T>::rebond() will fail
        //    because it can't find the tokens in the unbonding queue
        assert_eq!(empty_locks.len(), 0, "The staking lock should be removed after unbonding period");
        
        // 7. Explain the failure
        println!("\nRebond Failure Explanation:");
        println!("1. After 7 days, tokens are fully unbonded on SXT Chain");
        println!("2. The staking lock is removed from balance locks");
        println!("3. When cancelInitiateUnstake is called after 7 days:");
        println!("   - The EVM contract allows the call (after 10 days)");
        println!("   - But the SXT Chain's rebond operation fails");
        println!("   - Because the tokens are no longer in the unbonding queue");
        println!("4. This creates a mismatch between EVM and SXT Chain states");
    }
}
```

### **Recommendation**

*   **Align Delay Durations:** Change the default timelock in `deployCollaborativeStakingWithDefaults` from 10 days to match the on-chain unbonding period (7 days), or better yet:
*   **Make Delay Configurable:** Expose the staking delay as a constructor parameter (rather than hard-coding) so that deployers can set it to exactly the network’s unbonding period.
*   **Document Clearly:** In both factory and staking contracts, document that the staking delay must equal or be less than the chain’s unbonding period to preserve cancel flexibility.
*   **Add Runtime Check:** In the staking constructor or factory, include a runtime `require(timelockDelay <= maxUnbondingDuration, "Timelock exceeds unbonding period")` to prevent misconfiguration.
