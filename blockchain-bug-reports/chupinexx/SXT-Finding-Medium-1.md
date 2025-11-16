
### **Title: `validateMessage` Fails with Unordered Signatures, Leading to Permanent Loss of Access to Unstaked Funds**

### **Summary**

The `sxtFulfillUnstake` function in the EVM staking contract relies on a signature validation logic (`validateMessage`) that implicitly requires incoming SXT validator signatures to be sorted by signer address. However, the SXT Chain does not guarantee this order, leading to potentially valid multi-signature sets being incorrectly rejected. This breaks the unstake fulfillment process, permanently locking user funds in the `UnstakeClaimed` state.

### **Finding Description**

The `Staking.sol` contract on the EVM chain contains a critical vulnerability in its `sxtFulfillUnstake` function, specifically within the logic of the `ISubstrateSignatureValidator.validateMessage` function it calls. This validation function is designed to verify that a given `messageHash` (representing SXT Chain state for an unstake) has been signed by a sufficient threshold of registered SXT attestors. However, the implementation of `validateMessage` implicitly assumes that the provided arrays of signature components (`r`, `s`, `v`) will yield recovered signer addresses in a strictly ascending order, corresponding to the sorted list of attestors stored within the `SubstrateSignatureValidator` contract.

Information regarding the SXT Chain's off-chain attestation processing indicates that while each attestor signs uniquely per SXT block, the collected signatures relayed to the EVM for `sxtFulfillUnstake` are not sorted by the signer's Ethereum address; they maintain the order in which attestations were processed or received on the SXT side.

This mismatch between the EVM contract's expectation of sorted signer data and the SXT Chain's provision of unsorted signer data will cause the `validateMessage` function to incorrectly determine that a valid set of signatures (meeting the threshold) is invalid if those signatures happen to be presented out of ascending signer address order. Consequently, `sxtFulfillUnstake` will revert with an `InvalidSignature()` error, preventing the user from withdrawing their tokens. Since the user's state in the `Staking.sol` contract is `UnstakeClaimed` at this point, and there are no other state transitions out of `UnstakeClaimed` except via a successful `sxtFulfillUnstake`, the user's funds become permanently inaccessible through the contract's intended mechanisms.

#### The Unstake Fulfillment and Validation Process:

1.  A user has successfully called `initiateUnstake()` and then `claimUnstake()`, moving their `stakerState` to `UnstakeClaimed`. They are awaiting the SXT system to call `sxtFulfillUnstake`.
2.  The SXT system (relayer/oracle) calls `sxtFulfillUnstake` on the EVM `Staking.sol` contract, providing parameters including the `staker`, `amount`, `sxtBlockNumber`, `proof`, and the signature components `r[]`, `s[]`, `v[]` from multiple SXT attestors.

    ```solidity
    // In Staking.sol
    function sxtFulfillUnstake(
        address staker,
        uint248 amount,
        uint64 sxtBlockNumber,
        bytes32[] calldata proof,
        bytes32[] calldata r, // Array of r values from SXT attestors
        bytes32[] calldata s, // Array of s values from SXT attestors
        uint8[] calldata v   // Array of v values (27/28) from SXT attestors
    ) external requireState(staker, StakerState.UnstakeClaimed) whenNotPaused {
        // ... (anti-replay check) ...
    
        // The call to the vulnerable validation logic:
        if (!_validateSxtFulfillUnstake(staker, amount, sxtBlockNumber, proof, r, s, v)) {
            revert InvalidSignature(); // This revert is incorrectly triggered
        }
        // ... (state changes and token withdrawal) ...
    }
    ```

3.  The `_validateSxtFulfillUnstake` function reconstructs the `messageHash` that SXT attestors should have signed:

    ```solidity
    // In Staking.sol
    function _validateSxtFulfillUnstake(
        // ... params ...
        bytes32[] calldata r,
        bytes32[] calldata s,
        uint8[] calldata v
    ) internal view returns (bool isValid) {
        // ... (leaf and rootHash calculation) ...
        bytes32 messageHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n40", rootHash, sxtBlockNumber));
    
        // Call to the SubstrateSignatureValidator contract
        isValid =
            ISubstrateSignatureValidator(SUBSTRATE_SIGNATURE_VALIDATOR_ADDRESS)
                .validateMessage(messageHash, r, s, v);
    }
    ```

4.  The `SubstrateSignatureValidator.validateMessage` function attempts to verify the signatures:

    ```solidity
    // In SubstrateSignatureValidator.sol
    function validateMessage(bytes32 message, bytes32[] calldata r, bytes32[] calldata s, uint8[] calldata v)
        external
        view
        returns (bool result)
    {
        if (r.length != s.length || s.length != v.length || r.length == 0) return false;
    
        uint256 attestorsLength = _attestors.length; // _attestors is sorted ascending on-chain
        uint256 signaturesLength = r.length;
        uint256 validSignaturesCount = 0;
        uint256 attestorIndex = 0; // Pointer for the sorted _attestors list
    
        // Loop through the provided signatures (order determined by SXT relay)
        for (uint256 i = 0; i < signaturesLength; ++i) {
            address recoveredAddress = ecrecover(message, v[i], r[i], s[i]);
    
            // This inner loop advances attestorIndex through the sorted _attestors list
            // to find or pass the recoveredAddress.
            // It ONLY moves attestorIndex FORWARD.
            while (attestorIndex < attestorsLength && _attestors[attestorIndex] < recoveredAddress) {
                ++attestorIndex;
            }
    
            // This check requires that the current _attestors[attestorIndex] IS the recoveredAddress.
            // If recoveredAddress was smaller than a previously processed signature's recovered address,
            // attestorIndex might have already moved past this recoveredAddress's correct spot.
            if (attestorIndex < attestorsLength && _attestors[attestorIndex] == recoveredAddress) {
                ++validSignaturesCount;
                ++attestorIndex; // Move to expect the NEXT attestor in sorted order
            }
    
            if (validSignaturesCount == _threshold) return true;
        }
        return false; // Returns false if threshold not met
    }
    ```

#### The Flaw in `validateMessage`'s Logic:

The `validateMessage` function iterates through the provided signatures one by one. For each signature, it recovers the signer's address (`recoveredAddress`). It then tries to find this `recoveredAddress` in its internally stored `_attestors` list, which is guaranteed to be sorted in ascending order (due to checks in `_updateAttestors`).

The crucial part is how it searches: it uses a single pointer, `attestorIndex`, that only ever increases.

1.  When processing the first signature, `recoveredAddress_0`, it advances `attestorIndex` until `_attestors[attestorIndex]` is greater than or equal to `recoveredAddress_0`. If they match, the signature is counted, and `attestorIndex` is incremented again.
2.  When processing the second signature, `recoveredAddress_1`, it starts searching in `_attestors` from the current (potentially advanced) `attestorIndex`.

If the SXT Chain relays signatures where `recoveredAddress_1 < recoveredAddress_0` (i.e., not sorted by address):

*   When `recoveredAddress_0` (the larger address) is processed, `attestorIndex` will be advanced significantly, possibly past the position where `recoveredAddress_1` (the smaller address) resides in the `_attestors` array.
*   When `recoveredAddress_1` is then processed, the `while` loop (`_attestors[attestorIndex] < recoveredAddress_1`) will not run because `_attestors[attestorIndex]` is likely already greater than `recoveredAddress_1`. The subsequent `if` check (`_attestors[attestorIndex] == recoveredAddress_1`) will fail.
*   Thus, the valid signature from `recoveredAddress_1` is not counted.

If enough valid signatures are missed due to this ordering issue, `validSignaturesCount` will not reach `_threshold`, and `validateMessage` will incorrectly return `false`. This causes `sxtFulfillUnstake` to revert with `InvalidSignature()`.

#### Example of Failure:

*   `_attestors` on EVM = `[AddrA, AddrB, AddrC, AddrD, AddrE]` (sorted)
*   `_threshold` = 3
*   SXT relays valid signatures from `AddrD`, then `AddrB`, then `AddrC`.

1.  **Process sig from `AddrD`**: `recoveredAddress` = `AddrD`. `attestorIndex` moves from 0 to 3 (`_attestors[3] == AddrD`). Match found. `validSignaturesCount` = 1. `attestorIndex` becomes 4.
2.  **Process sig from `AddrB`**: `recoveredAddress` = `AddrB`. `attestorIndex` is 4 (`_attestors[4] == AddrE`). The `while` loop (`_attestors[attestorIndex] < AddrB`) is false because `AddrE < AddrB` is false. The `if` condition (`_attestors[attestorIndex] == AddrB`) is false because `AddrE != AddrB`. Signature from `AddrB` is missed. `validSignaturesCount` remains 1.
3.  **Process sig from `AddrC`**: `recoveredAddress` = `AddrC`. `attestorIndex` is 4. `while` (`_attestors[attestorIndex] < AddrC`) is false. `if` (`_attestors[attestorIndex] == AddrC`) is false. Signature from `AddrC` is missed. `validSignaturesCount` remains 1.

**Result**: `validateMessage` returns `false` even though three valid signatures (`AddrD`, `AddrB`, `AddrC`) were provided.

#### Security Guarantees Broken:

*   **Liveness of Fund Withdrawal:** The primary guarantee that valid attestations from a sufficient number of SXT validators will allow a user to withdraw their funds is broken. The success of withdrawal becomes dependent on the unpredictable order in which SXT relays signatures.
*   **Availability of Staked Funds:** For users whose unstake fulfillment transaction happens to receive signatures in an "unlucky" (for the EVM contract's logic) order, their funds become permanently inaccessible via the `sxtFulfillUnstake` mechanism. Since there is no other way to transition out of the `UnstakeClaimed` state, this is equivalent to a loss of funds.
*   **Reliability of Cross-Chain Verification:** The signature verification mechanism is unreliable due to its sensitivity to input order, an order which is not guaranteed by the counterparty system (SXT Chain).

This is not an issue caused by a malicious user providing faulty inputs to `sxtFulfillUnstake` (as that function is typically called by an SXT relayer). Instead, it's a flaw in the EVM contract's validation logic that fails to correctly handle legitimate, but potentially unsorted, data from its trusted SXT oracle/relayer system. The "malicious input" is effectively the unsorted nature of the signature list itself, which the EVM contract is not robust against.

### **Impact Explanation**

Because valid validator signatures can be dropped simply due to ordering, users’ unstaked tokens become irretrievably locked. They cannot complete withdrawal, cannot re-stake or cancel, and their funds remain stuck in the contract. This breaks liveness and causes direct financial harm.

We rate this **High impact**.

### **Likelihood Explanation**

Validator signatures on SxT are collected asynchronously and in arbitrary order. Since there is no sorting step before forwarding them on-chain, this mismatch will occur in a significant fraction of real-world fulfillments. Not every batch will happen to match the L1 order, so we assess **Medium likelihood**.

### **Proof of Concept**

Create a new file `sxt-node-op-contracts/test/SubstrateSignatureValidatorOrderTest.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {SubstrateSignatureValidator} from "../src/SubstrateSignatureValidator.sol";

contract SubstrateSignatureValidatorOrderTest is Test {
    SubstrateSignatureValidator private validator;
    address[] private attestors;
    bytes32 private message;
    uint256[] private attestorsPrivateKeys;

    function setUp() public {
        // Create a message that will be signed
        message = keccak256("test message");

        // Create 3 attestors with known private keys
        attestorsPrivateKeys = new uint256[](3);
        attestorsPrivateKeys[0] = 0x1; // Smallest address
        attestorsPrivateKeys[1] = 0x2; // Middle address
        attestorsPrivateKeys[2] = 0x3; // Largest address

        // Generate addresses from private keys
        attestors = new address[](3);
        for (uint256 i = 0; i < 3; ++i) {
            attestors[i] = vm.addr(attestorsPrivateKeys[i]);
        }

        // Sort attestors to ensure they're in ascending order
        // This simulates the on-chain sorted attestors list
        for (uint256 i = 0; i < attestors.length; i++) {
            for (uint256 j = i + 1; j < attestors.length; j++) {
                if (attestors[i] > attestors[j]) {
                    address temp = attestors[i];
                    uint256 tempKey = attestorsPrivateKeys[i];
                    attestors[i] = attestors[j];
                    attestorsPrivateKeys[i] = attestorsPrivateKeys[j];
                    attestors[j] = temp;
                    attestorsPrivateKeys[j] = tempKey;
                }
            }
        }

        // Create validator with threshold of 2 (requires 2 valid signatures)
        validator = new SubstrateSignatureValidator(attestors, 2);
    }

    function testSignatureOrderVulnerability() public {
        // Create signature arrays
        bytes32[] memory r = new bytes32[](3);
        bytes32[] memory s = new bytes32[](3);
        uint8[] memory v = new uint8[](3);

        // Sign the message with all three attestors
        for (uint256 i = 0; i < 3; ++i) {
            (v[i], r[i], s[i]) = vm.sign(attestorsPrivateKeys[i], message);
        }

        // Test 1: Signatures in correct order (should pass)
        // Order: Smallest address first, then middle, then largest
        bytes32[] memory rCorrect = new bytes32[](2);
        bytes32[] memory sCorrect = new bytes32[](2);
        uint8[] memory vCorrect = new uint8[](2);

        // Use signatures from first two attestors (smallest addresses)
        rCorrect[0] = r[0];
        sCorrect[0] = s[0];
        vCorrect[0] = v[0];
        rCorrect[1] = r[1];
        sCorrect[1] = s[1];
        vCorrect[1] = v[1];

        // This should pass as signatures are in correct order
        bool resultCorrect = validator.validateMessage(
            message,
            rCorrect,
            sCorrect,
            vCorrect
        );
        assertTrue(resultCorrect, "Correct order validation should pass");

        // Test 2: Signatures in wrong order (should fail)
        // Order: Largest address first, then smallest
        bytes32[] memory rWrong = new bytes32[](2);
        bytes32[] memory sWrong = new bytes32[](2);
        uint8[] memory vWrong = new uint8[](2);

        // Use signature from largest address first, then smallest
        rWrong[0] = r[2]; // Largest address signature
        sWrong[0] = s[2];
        vWrong[0] = v[2];
        rWrong[1] = r[0]; // Smallest address signature
        sWrong[1] = s[0];
        vWrong[1] = v[0];

        // This should fail due to the order mismatch
        bool resultWrong = validator.validateMessage(
            message,
            rWrong,
            sWrong,
            vWrong
        );
        assertFalse(resultWrong, "Wrong order validation should fail");

        // Test 3: Same signatures in different order (should fail)
        // Order: Middle address first, then smallest
        bytes32[] memory rWrong2 = new bytes32[](2);
        bytes32[] memory sWrong2 = new bytes32[](2);
        uint8[] memory vWrong2 = new uint8[](2);

        rWrong2[0] = r[1]; // Middle address signature
        sWrong2[0] = s[1];
        vWrong2[0] = v[1];
        rWrong2[1] = r[0]; // Smallest address signature
        sWrong2[1] = s[0];
        vWrong2[1] = v[0];

        // This should also fail due to the order mismatch
        bool resultWrong2 = validator.validateMessage(
            message,
            rWrong2,
            sWrong2,
            vWrong2
        );
        assertFalse(resultWrong2, "Second wrong order validation should fail");

        // Print addresses for verification
        console.log("Attestor addresses (sorted):");
        for (uint256 i = 0; i < attestors.length; i++) {
            console.log("Attestor", i, ":", attestors[i]);
        }
    }
}
```

**Run the POC:**
`forge test --match-test testSignatureOrderVulnerability -vv`

**Results:**
```
Ran 1 test for test/SubstrateSignatureValidatorOrderTest.t.sol:SubstrateSignatureValidatorOrderTest
[PASS] testSignatureOrderVulnerability() (gas: 88145)
Logs:
  Attestor addresses (sorted):
  Attestor 0 : 0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
  Attestor 1 : 0x6813Eb9362372EEF6200f3b1dbC3f819671cBA69
  Attestor 2 : 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.04ms (6.08ms CPU time)

Ran 1 test suite in 76.76ms (17.04ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### **Recommendation**

Introduce an explicit signature ordering (or order-independent validation) before calling `validateMessage`. There are two complementary approaches:

1.  **Sort Signatures by Signer Address**
    Recover each signature’s address off-chain (or in a pre-validation step) and sort the `(v, r, s)` triples by the recovered address in ascending order. Forward the sorted arrays into `validateMessage` so that the on-chain routine’s linear scan matches its internal `_attestors` ordering.

2.  **Use Order-Independent Validation**
    Change `validateMessage` so it does not assume matched ordering. Instead, build a temporary on-chain set or bit-mask of “seen” attestors and count valid signatures regardless of position.

    For example:
    ```solidity
    mapping(address => bool) seen;
    uint validCount;
    for (i in signatures) {
      address signer = ecrecover(...);
      if (!seen[signer] && attestorSet.contains(signer)) {
        seen[signer] = true;
        validCount++;
      }
      if (validCount == threshold) return true;
    }
    return false;
    ```
    This guarantees that any order of signatures will be accepted as long as the threshold is met.

By adopting one (or both) of these changes, you eliminate ordering fragility and ensure liveness: valid attestations will always pass, and users can reliably complete unstake fulfillments.
