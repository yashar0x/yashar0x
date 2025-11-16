### **Title: Users Trapped in `UnstakeClaimed` State with No Recovery Path After SXT Chain Reorganization**

### **Summary**

Users can become permanently stuck in the `UnstakeClaimed` state within the EVM staking contract if an SXT Chain reorganization invalidates the off-chain proofs needed for `sxtFulfillUnstake`. The contract offers no recovery path from this state, effectively locking user funds intended for withdrawal.

### **Finding Description**

The `Staking.sol` contract implements a multi-step process for users to unstake their SXT tokens. A critical vulnerability exists in this process: if the SXT Chain (the Layer 1 blockchain providing data and proofs for unstake fulfillment) undergoes a blockchain reorganization after a user has transitioned their state to `UnstakeClaimed` on the EVM contract but before the SXT-provided fulfillment transaction (`sxtFulfillUnstake`) is successfully executed, the user's funds can become permanently locked within the EVM staking contract. This is because the cryptographic proofs tied to the pre-reorg SXT Chain state become cleared, and the EVM contract lacks any mechanism to reset the user's state or handle such cross-chain desynchronization.

#### The Staking and Unstaking Lifecycle (Simplified):

1.  **Staking:** A user calls `stake(amount)` on the EVM `Staking.sol` contract.

    ```solidity
    function stake(uint248 amount) external requireStateNotUnstakeClaimed(msg.sender) {
        // ... (checks for amount) ...
        if (stakerState[msg.sender] == StakerState.UnstakeInitiated) {
            _cancelInitiateUnstake(msg.sender); // Auto-cancels previous unstake attempt
        }
        stakerState[msg.sender] = StakerState.Staked; // User is now Staked
        IERC20(TOKEN_ADDRESS).safeTransferFrom(msg.sender, STAKING_POOL_ADDRESS, amount);
        emit Staked(msg.sender, amount); // Event signals SXT Chain
    }
    ```
    The `Staked` event informs the SXT Chain about this action.

2.  **Initiating Unstake:** The user decides to unstake and calls `initiateUnstake(amount)`:

    ```solidity
    function initiateUnstake(uint248 amount) external requireState(msg.sender, StakerState.Staked) whenNotPaused {
        initiateUnstakeRequestsTimestamp[msg.sender] = uint64(block.timestamp); // Records EVM timestamp
        stakerState[msg.sender] = StakerState.UnstakeInitiated; // State changes
        emit UnstakeInitiated(msg.sender, amount); // Event signals SXT Chain
    }
    ```
    The SXT Chain receives this, and its internal `pallet_staking` begins the 7-day unbonding period.

3.  **Claiming Unstake (Transition to `UnstakeClaimed`):** After the 7-day unbonding period (enforced by `UNSTAKING_UNBONDING_PERIOD` on the EVM side, checked against `initiateUnstakeRequestsTimestamp`), the user calls `claimUnstake()`:

    ```solidity
    function claimUnstake() external requireState(msg.sender, StakerState.UnstakeInitiated) whenNotPaused {
        uint64 earliestPossibleClaimUnstakeRequest =
            initiateUnstakeRequestsTimestamp[msg.sender] + UNSTAKING_UNBONDING_PERIOD;
        if (block.timestamp < earliestPossibleClaimUnstakeRequest) revert UnstakeNotUnbonded();

        stakerState[msg.sender] = StakerState.UnstakeClaimed; // CRITICAL STATE CHANGE
        emit UnstakeClaimed(msg.sender); // Event signals SXT Chain (or its oracles)
    }
    ```
    At this point, the user's state in the EVM contract is `UnstakeClaimed`. They are now entirely dependent on an external call to `sxtFulfillUnstake` (made by an SXT relayer/oracle) to receive their tokens. This `sxtFulfillUnstake` call will contain proofs derived from a specific state of the SXT Chain.

#### **The Vulnerability Scenario: SXT Chain Reorganization After `claimUnstake`**

1.  **SXT Chain Prepares Proofs:**
    *   Following the `UnstakeClaimed` event (or based on the SXT Chain's own tracking of the completed 7-day unbonding), the SXT off-chain system identifies that the user's tokens are unbonded on the SXT Chain.
    *   It prepares the necessary data for `sxtFulfillUnstake`: staker address, amount, the relevant `sxtBlockNumber` (let's call it `SXT_Block_Orphan`) where this unbonded state was observed and attested, the Merkle proof for this state, and the SXT validator signatures (r,s,v) over a message hash containing `SXT_Block_Orphan`'s state root.
    *   Crucially, the attestation for `SXT_Block_Orphan` (which forms the basis of the signatures) occurs once `SXT_Block_Orphan` is GRANDPA-finalized.

2.  **A Deep SXT Chain Reorganization Occurs:**
    *   While very rare and indicative of a severe issue on the SXT Chain itself (e.g., >1/3 validators colluding to stall and then fork, or a major consensus bug), a reorganization happens that does affect GRANDPA-finalized blocks.
    *   As a result, `SXT_Block_Orphan` is no longer part of the canonical, recognized SXT Chain. The state it represented (including the user's specific unbonded balance details) is effectively "gone" or altered in the new canonical chain.

3.  **`sxtFulfillUnstake` Attempt Fails:**
    *   SXT's oracle system is robust enough to detect the re-org, it might not even attempt to submit the transaction with proofs from `SXT_Block_Orphan`. Instead, it would try to generate new proofs based on the new canonical SXT chain.

4.  **User is Stuck in `UnstakeClaimed` State:**
    *   The user's `stakerState[msg.sender]` on the EVM contract remains `StakerState.UnstakeClaimed`.
    *   As per the contract logic:
        *   `stake()`: Blocked by `requireStateNotUnstakeClaimed(msg.sender)`.
        *   `initiateUnstake()`: Blocked by `requireState(msg.sender, StakerState.Staked)`.
        *   `cancelInitiateUnstake()`: Blocked by `requireState(msg.sender, StakerState.UnstakeInitiated)`.
        *   `claimUnstake()`: Blocked by `requireState(msg.sender, StakerState.UnstakeInitiated)`.
    *   There are no other functions in the provided code that allow a user in the `UnstakeClaimed` state to transition to any other state or to retry any part of the unstaking process from the EVM side.

#### **Security Guarantees Broken:**

*   **Liveness of Fund Withdrawal:** The guarantee that a user can eventually withdraw their unstaked tokens is broken if the cross-chain fulfillment step fails due to SXT Chain issues like a deep re-org, and there's no EVM-side recovery.
*   **Availability of Staked Funds:** The funds are not stolen by an attacker in this scenario but become permanently unavailable to the legitimate owner. This is equivalent to a loss of funds from the user's perspective.
*   **Protocol Robustness to Cross-Chain Failures:** The EVM contract does not gracefully handle a scenario where its state (`UnstakeClaimed`) reflects an expectation of an SXT Chain action that can no longer occur or be validated.

This vulnerability highlights the complexities and risks inherent in cross-chain interactions, especially when one chain's state transitions are tightly coupled with proofs derived from another chain that can, however rarely, undergo significant state changes like deep re-orgs.

### **Impact Explanation**

This issue is rated **High Impact** because a single SxT chain re-organization during the claim window will permanently lock a user’s funds in the staking contract. The user loses the ability to withdraw, restake, or cancel their unstake, effectively freezing their tokens with no on-chain remedy.

### **Likelihood Explanation**

The likelihood of this scenario is **Low** because it requires a chain re-organization on the SxT network at precisely the narrow window between `UnstakeClaimed` and proof finalization. Such deep reorganizations are rare under normal network conditions, but even a single occurrence leads to irreversible user fund lock-ups, warranting careful mitigation.

> I choose **Medium** severity because if the protocol experience a re-org all users that claim their unstake will lose their fund and also they will be inaccessible to use the protocol again (by staking ) because their statue is stakeclaimed.

### **Proof of Concept**

#### **Proof-of-Concept Scenario: Stuck Funds due to Chain Re-org**

1.  **User Stakes & Initiates Unstake**
    *   Alice stakes 1,000 SXT tokens via `stake(1000)`.
    *   After some time, she calls `initiateUnstake(1000)`, moving her state to `UnstakeInitiated` and recording `initiateUnstakeRequestsTimestamp = T`.

2.  **Alice Claims Unstake**
    *   At `T + 7` days, Alice calls `claimUnstake()`.
    *   Contract sets her state to `UnstakeClaimed` and emits `UnstakeClaimed(Alice)`.
    *   Off-chain, Indexers detect this event, gather SXT-chain Merkle proofs and signatures for Alice’s unstake, and then call `sxtFulfillUnstake(...)` on Ethereum.

3.  **SxT Chain Re-org Occurs**
    *   Before proof finalization, the SxT network experiences a re-organization that invalidates the block containing Alice’s `UnstakeInitiated` or earlier staking data.
    *   Indexers’ proofs, now based on a forked state, no longer match the on-chain Merkle root in the contract.

4.  **Fulfillment Fails & Alice Is Locked**
    *   When relayers submit `sxtFulfillUnstake`, `_validateSxtFulfillUnstake` reverts with `InvalidProof()`.
    *   The EVM contract has already moved Alice’s state to `UnstakeClaimed`, so she cannot call `cancelInitiateUnstake` (requires `UnstakeInitiated`) nor `stake` (blocked by `requireStateNotUnstakeClaimed`).
    *   No admin function or fallback exists to reset her state. Alice’s 1,000 SXT is now permanently locked.

### **Recommendation**

*   **Admin Reset Function:** Add an `owner`-only `resetUnstakeClaim(address user)` that reverts a stuck `UnstakeClaimed` back to `UnstakeInitiated` or `Staked`, enabling manual recovery in the rare case of a fork.
*   **Event Monitoring:** Emit a special `ProofRejected(queryHash)` event when fulfillment fails so off-chain tooling can detect and trigger the admin reset.
