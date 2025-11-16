### **Title:** Incorrect Committee Size Validation During Slashing of PendingExit Validators Causes Unintended Reverts

### **Summary**

When slashing a validator in `PendingExit` status, the protocol incorrectly decrements the count of eligible validators for all committee checks, even if the validator is not present in the next or future committees. This mismatch can cause unnecessary transaction reverts, blocking the protocol from slashing and removing ineligible validators, and undermining the enforcement of penalties.

### **Finding Description**

The protocol’s slashing logic in `ConsensusRegistry.sol` is designed to remove a validator from all relevant committees and enforce penalties when their balance reaches zero. However, when a validator in `PendingExit` status is slashed, the protocol incorrectly decrements the count of eligible validators (`numEligible`) for all committee size checks, regardless of whether the validator is actually present in the next or future committees. This can cause the transaction to revert if the committee size exceeds the decremented `numEligible`, preventing the protocol from slashing and removing the validator as intended.

### **Root Cause**

The issue originates in the `_consensusBurn` function:

```solidity
function _consensusBurn(address validatorAddress) internal {
    balances[validatorAddress] = 0;

    ValidatorInfo storage validator = validators[validatorAddress];
    ValidatorStatus status = validator.currentStatus;
    // reverts if decremented committee size after ejection reaches 0, preventing network halt
    uint256 numEligible = _getValidators(ValidatorStatus.Active).length;
    // if validator being ejected is committee-eligible, ejection will decrement `numEligible`
    if (_eligibleForCommitteeNextEpoch(status)) {
        numEligible = numEligible - 1;
    }
    _ejectFromCommittees(validatorAddress, numEligible);

    // exit, retire, and unstake + burn validator immediately
    _exit(validator, currentEpoch);
    _retire(validator);
    address recipient = _getRecipient(validatorAddress);
    _unstake(validatorAddress, recipient, validator.stakeVersion);
}
```

The function `_eligibleForCommitteeNextEpoch` returns `true` for `PendingExit` status, so `numEligible` is decremented. The decremented value is then passed to `_ejectFromCommittees`, which checks committee sizes for the current, next, and subsequent epochs:

```solidity
function _ejectFromCommittees(address validatorAddress, uint256 numEligible) internal {
    // ... for each committee (current, next, subsequent):
    bool ejected = _eject(committee, validatorAddress);
    uint256 committeeSize = ejected
        ? committee.length - 1
        : committee.length;
    _checkCommitteeSize(numEligible, committeeSize);
}
```

The problem arises when the validator is not present in the next or future committees (as is typical for `PendingExit` validators under normal conditions with enough eligible validators). The code still uses the decremented `numEligible` for all committee size checks, even though the validator was not removed from those committees. If the actual committee size is greater than the decremented `numEligible`, the `_checkCommitteeSize` call will revert, blocking the slashing operation.

#### **Security Guarantee Broken**

This bug breaks the protocol’s guarantee that it can always enforce slashing and remove ineligible or malicious validators. Instead, a validator in `PendingExit` status who is not present in future committees can become unslashable due to a revert, undermining protocol enforcement and liveness.

### **Exploitation Scenario**

1.  **PendingExit Status:** A validator transitions to `PendingExit` and is not selected for the next or future committees (as expected under normal protocol operation when enough eligible validators exist).
2.  **Slashing Event:** The protocol attempts to slash the validator.
3.  **Revert Triggered:** `_consensusBurn` decrements `numEligible` and passes it to `_ejectFromCommittees`. Since the validator is not in the next or future committees, the committee size check uses a value that is too small, causing `_checkCommitteeSize` to revert.
4.  **Result:** The slashing transaction fails, and the validator is not removed or penalized as intended.

### **Recommendation**

Update the logic in `_consensusBurn` and `_ejectFromCommittees` so that the `numEligible` value used for committee size checks is only decremented for committees from which the validator is actually removed. Specifically, before calling `_checkCommitteeSize`, determine if the validator was present and ejected from each committee; only then should the eligible count be decremented for that committee. For committees where the validator was not present, use the original `numEligible` value.

```solidity
function _ejectFromCommittees(address validatorAddress, uint256 numEligible) internal {
    uint32 current = currentEpoch;
    uint8 currentEpochPointer = epochPointer;

    // Current committee
    address[] storage currentCommittee = _getRecentEpochInfo(current, current, currentEpochPointer).committee;
    bool ejected = _eject(currentCommittee, validatorAddress);
    uint256 committeeSize = currentCommittee.length;
    uint256 eligibleForCurrent = ejected ? numEligible : numEligible + 1;
    _checkCommitteeSize(eligibleForCurrent, committeeSize);

    // Next committee
    uint32 nextEpoch = current + 1;
    address[] storage nextCommittee = _getFutureEpochInfo(nextEpoch, current, currentEpochPointer).committee;
    ejected = _eject(nextCommittee, validatorAddress);
    committeeSize = nextCommittee.length;
    uint256 eligibleForNext = ejected ? numEligible : numEligible + 1;
    _checkCommitteeSize(eligibleForNext, committeeSize);

    // Subsequent committee
    uint32 subsequentEpoch = current + 2;
    address[] storage subsequentCommittee = _getFutureEpochInfo(subsequentEpoch, current, currentEpochPointer).committee;
    ejected = _eject(subsequentCommittee, validatorAddress);
    committeeSize = subsequentCommittee.length;
    uint256 eligibleForSubsequent = ejected ? numEligible : numEligible + 1;
    _checkCommitteeSize(eligibleForSubsequent, committeeSize);
}
```
