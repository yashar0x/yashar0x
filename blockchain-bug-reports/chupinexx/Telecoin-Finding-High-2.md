### **Title:** Incorrect Assignment of Delegator and Validator Fields in Delegation Struct Results in Delegator Losing Reward Claim Rights

### **Finding Description**

The `delegateStake` function in `ConsensusRegistry.sol` is intended to allow a delegator to delegate stake to a validator, with the expectation that the delegator will be able to claim rewards accrued from this delegation. However, due to a misassignment in the `Delegation` struct, the actual delegator is not properly recorded, resulting in the delegator being unable to claim their rewards. This breaks the security guarantee that only the true delegator (the party who provided the stake) can claim the associated rewards.

### **Root Cause**

The root of the issue lies in the construction of the `Delegation` struct within the `delegateStake` function. The struct is defined as follows (from `IStakeManager.sol`):

```solidity
struct Delegation {
    bytes32 blsPubkeyHash;
    address validatorAddress;
    address delegator;
    uint8 validatorVersion;
    uint64 nonce;
}
```

However, in `ConsensusRegistry.sol`, the assignment is:

```solidity
delegations[validatorAddress] = Delegation(
    blsPubkeyHash,
    msg.sender, // Incorrectly assigned to validatorAddress field
    validatorAddress, // Incorrectly assigned to delegator field
    validatorVersion,
    nonce + 1
);
```

Here, `msg.sender` (the actual delegator) is incorrectly assigned to the `validatorAddress` field, and `validatorAddress` is incorrectly assigned to the `delegator` field. The correct assignment should be:

```solidity
delegations[validatorAddress] = Delegation(
    blsPubkeyHash,
    validatorAddress,
    msg.sender,
    validatorVersion,
    nonce + 1
);
```

This bug breaks the guarantee that only the delegator can claim rewards for their delegation. The system uses the `delegator` field to determine the rightful recipient of rewards. With the current misassignment, the validator is set as the delegator, and the actual delegator is ignored in reward-claiming logic and also for unstaking.

### **Impact Explanation**

The impact is assessed as **Medium** because this issue breaks the core invariant that only the delegator should be able to claim their rewards. However, the protocol assumes an off-chain relationship and agreement between validator and delegator, which is why this is seen as a Medium.

### **Likelihood Explanation**

The likelihood is **High** because the bug is present in the code logic and will occur every time delegation is used; it is not dependent on any specific user action or edge case.

### **Recommendation**

Correct the assignment in the `delegateStake` function so that `validatorAddress` is set to the validator and `delegator` is set to `msg.sender` in the `Delegation` struct. This will ensure the actual delegator is properly recorded and able to claim their rewards.
