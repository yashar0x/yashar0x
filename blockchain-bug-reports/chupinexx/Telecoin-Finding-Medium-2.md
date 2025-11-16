### **Title**:Low-Level Call Allows EIP-7702 Wallets to Block Slashing/Burn, Enabling Denial-of-Service Against Protocol Enforcement

### **Summary**

A validator or delegator can exploit the protocol’s use of a low-level `call` for reward and stake distribution by leveraging EIP-7702 smart contract wallets. By dynamically updating their wallet’s `fallback` function to revert during slashing or burn events, they can intentionally block these critical operations, resulting in a denial-of-service and undermining protocol enforcement.

### **Finding Description**

The `distributeStakeReward` function in `Issuance.sol` is responsible for distributing rewards and stake to validators or delegators. It uses a low-level `call` to transfer funds to the recipient:

```solidity
(bool res, ) = recipient.call{ value: totalAmount }("");
if (!res) revert RewardDistributionFailure(recipient);
```

This function is invoked not only during normal reward claims, but also during slashing and burning events (via `consensusBurn`), where a validator or delegator’s stake and rewards are forcibly withdrawn and sent to their address.

#### **Security Guarantee Broken**

The protocol assumes that slashing or burning a validator will always succeed, ensuring that malicious or non-compliant validators can be removed from the system and their funds appropriately handled. However, this guarantee is broken if the recipient can intentionally cause the transfer to fail.

### **Exploitation via EIP-7702**

With the introduction of EIP-7702, users can upgrade their externally owned accounts (EOAs) to smart contract wallets with customizable logic. This allows a validator or delegator to dynamically change their wallet’s `fallback` function.

**Normal Operation:**

When claiming rewards, the user can set their `fallback` function to accept TEL (or leave it empty), allowing the transfer to succeed.

**Attack Scenario:**

When the validator or delegator knows they are about to be slashed to zero or burned (e.g., due to protocol enforcement or governance action), they can update their wallet’s `fallback` function to always revert. For example:

```solidity
fallback() external payable {
    revert("Block slashing");
}
```

When the protocol attempts to send their remaining stake and rewards during slashing or burning (in this case 0 TEL), the low-level call will revert, causing the entire transaction to fail:

```solidity
(bool res, ) = recipient.call{ value: totalAmount }("");
if (!res) revert RewardDistributionFailure(recipient);
```

This prevents the protocol from completing the slashing or burning operation, effectively making the validator or delegator “unslashable” or “unburnable.”

### **Recommendation**

To mitigate this issue, consider sending a wrapped TEL token (an ERC20) instead of native TEL.

If you need a specific mitigation against EIP7702 wallets use this from this [X](https://x.com/apoorveth/status/1923241437593821230) post
