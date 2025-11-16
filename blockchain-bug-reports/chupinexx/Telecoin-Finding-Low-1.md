### **Title:** Double-Decrement of Committee Length After Ejection Causes Incorrect Committee Size Validation

### **Summary**

When removing a validator from a committee, the protocol decrements the committee length twice—once by popping the array and again manually in the size check. This double-decrement results in an incorrect committee size calculation, which can trigger false invariant violations and cause critical protocol operations to bypass unnecessarily.

### **Finding Description**

The protocol enforces a critical invariant that the committee size must never exceed the number of eligible (active or pending) validators. This is checked using the following logic in `_checkCommitteeSize`:

```solidity
function _checkCommitteeSize(
    uint256 activeOrPending,
    uint256 committeeSize
) internal pure {
    if (
        activeOrPending == 0 ||
        committeeSize == 0 ||
        committeeSize > activeOrPending
    ) {
        revert InvalidCommitteeSize(activeOrPending, committeeSize);
    }
}
```

When a validator is ejected from a committee, the `_eject` function removes the validator and calls `committee.pop()`, which already decrements the committee’s length by one. However, in the calling function (`_ejectFromCommittees`), the code further decrements the committee length by one when performing the size check:

```solidity
bool ejected = _eject(committee, validatorAddress);
uint256 committeeSize = ejected
    ? committee.length - 1
    : committee.length;
_checkCommitteeSize(numEligible, committeeSize);
```

This double-decrement results in an artificially low `committeeSize` being checked against the number of eligible validators. The security guarantee that the committee size never exceeds the eligible validator count is thus broken—not by a revert, but by a silent bypass of the intended invariant.

### **How the Bug Propagates**

1.  A validator is ejected from the committee, and `committee.pop()` reduces the array length by one.
2.  The code then subtracts one again (`committee.length - 1`) for the size check.
3.  The result is that the committee size appears smaller than it actually is.
4.  The check `committeeSize > activeOrPending` is bypassed, because the double-decremented value is less than or equal to `activeOrPending`, even if the real committee size is actually greater than the eligible validator count.

### **Exploitation Scenario**

Suppose there are 5 eligible validators and the committee should not exceed this number. If, after ejection, the real committee size is 6, but due to the double-decrement, the check is performed with a value of 5, the following happens:

*   The check `committeeSize > activeOrPending` becomes `5 > 5`, which is false, so no revert occurs.
*   In reality, the committee size is 6, which should have triggered a revert, but the bug allows this invalid state to pass as correct.


### **Recommendation**

Update the logic in `_ejectFromCommittees` so that the committee size is checked using the actual, current length of the committee array after any ejection, without further manual decrementing. Specifically, after calling `_eject`, always use `committee.length` directly in the `_checkCommitteeSize` call, regardless of whether an element was removed.

**Example Fix:**
```solidity
bool ejected = _eject(committee, validatorAddress);
// Always use the actual committee length after pop, do not subtract 1
uint256 committeeSize = committee.length;
_checkCommitteeSize(numEligible, committeeSize);
```
