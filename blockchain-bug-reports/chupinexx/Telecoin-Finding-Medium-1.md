### **Title:** Lack of Mandatory Base Fee Enforcement Enables Fee-Free Attacks

### **Finding Description**

The Telcoin Network’s batch validation logic, designed to enforce EIP-1559’s dynamic fee market, contains a critical flaw that allows validators to submit batches with no `base_fee_per_gas`. EIP-1559 requires a minimum `base_fee_per_gas` to prevent spam and ensure fair resource allocation, but the `validate_basefee` function permits batches with `base_fee_per_gas = None`, effectively setting the base fee to zero. This allows malicious validators to include unlimited transactions without incurring fees, bypassing the transaction pool’s minimum gas price checks by directly crafting batches. The flaw propagates through the `Primary` actor and `Engine`, which do not enforce base fee requirements, enabling fee-free transactions to be executed and gossiped across the network. This vulnerability undermines the fee market, exposes the network to spam and DoS attacks, and threatens overall stability.

The vulnerability originates in the batch validation logic, specifically in the `validate_basefee` function, and propagates through the lack of base fee checks in other components. Below is a detailed analysis of the problematic code paths:

#### 1. Batch Validation Logic
The `validate_basefee` function checks the `base_fee_per_gas` field in a batch:

```rust
fn validate_basefee(&self, base_fee: Option<u64>) -> BatchValidationResult<()> {
    if let Some(base_fee) = base_fee {
        let expected_base_fee = self.base_fee.base_fee();
        if base_fee != expected_base_fee {
            return Err(BatchValidationError::InvalidBaseFee { expected_base_fee, base_fee });
        }
    }
    Ok(())
}
```
If `base_fee` is `Some(value)`, it is validated against the expected EIP-1559 base fee. However, if `base_fee` is `None`, the function returns `Ok(())`, accepting the batch without a base fee. This allows validators to submit batches with `base_fee_per_gas = None`, effectively setting the base fee to zero.

#### 2. Primary Actor Validation
The `Primary` actor, responsible for validating and voting on batches, does not check the `base_fee_per_gas` field:
```rust
self.state_sync.sync_header_batches(&header, false, 0).await?;```
The `Primary` actor verifies certificate signatures and parent links but does not inspect the batch’s `base_fee_per_gas`. It relies on the `validate_basefee` function, which permits `None`, allowing fee-free batches to pass validation and be gossiped to other nodes.

#### 3. Engine Execution
When the `Engine` executes the batch, it interprets `base_fee_per_gas = None` as a base fee of zero. Transactions in the batch are processed without a minimum fee, allowing validators to include their own transactions (with minimal tips) for free. This bypasses the economic protections of EIP-1559.

#### 4. Transaction Pool Irrelevance
The transaction pool (`TxPool`) enforces a minimum gas price for admitted transactions:
```rust
fn get_pending_base_fee(&self) -> u64 {
    MIN_PROTOCOL_BASE_FEE
}
```
This check is irrelevant to the attack, as malicious validators can bypass the `TxPool` by directly crafting batches with their own transactions, which are not subject to pool validation.

### **Attack Propagation Path**

The attack unfolds as follows:

1.  **Batch Creation:** A malicious validator creates a batch with `base_fee_per_gas = None` and fills it with their own transactions (e.g., with zero or minimal tips).
2.  **Batch Validation:** The `validate_basefee` function accepts the batch, as `base_fee = None` returns `Ok(())`.
3.  **Primary Actor Processing:** The `Primary` actor validates signatures and parent links but does not check the base fee, allowing the batch to be gossiped to other nodes.
4.  **Network Propagation:** Honest nodes accept the batch, as it passes all validation checks.
5.  **Engine Execution:** The `Engine` processes the batch with a base fee of zero, executing all transactions without charging the minimum fee.

**Result:** The attacker floods the network with fee-free transactions, causing congestion, state bloat, and displacement of honest users’ transactions.

### **Impact Explanation**

The impact of this vulnerability is **critical** because it allows any validator to submit batches of transactions with no base fee, effectively making all included transactions free. This undermines the EIP-1559 fee market, enabling unlimited spam, state bloat, and network congestion. As a result, honest users may experience severe transaction delays, increased resource consumption, and potential denial-of-service, threatening the stability and economic security of the entire Telcoin network.

### **Likelihood Explanation**

The likelihood of this issue being exploited is **high** because the flaw is present in the core batch validation logic and can be triggered by any validator without special privileges. There are no protocol or implementation safeguards to prevent a validator from submitting a batch with `base_fee_per_gas = None`. Since validator misbehavior is a realistic threat in permissioned and permissionless networks alike, this vulnerability can be exploited at any time.

> The protocol team assume that validators are Trusted but any misbehavior from validator that lead to a problem to the network should be reported and in scope.
>
> So Even i see this as High x High , i will choose Medium severity and its up to the judge to choose the final severity.

### **Recommendation**

To prevent zero-fee spam and enforce the integrity of the fee market, the Telcoin protocol and implementation should require every batch to include a valid, non-zero `base_fee_per_gas` that matches the expected EIP-1559 base fee for the current block.

**Recommended steps:**

1.  **Enforce Mandatory Base Fee in Validation:**
    Update the `validate_basefee` function to reject any batch where `base_fee_per_gas` is `None` or does not match the expected value.

    **Example fix:**
    ```rust
    fn validate_basefee(&self, base_fee: Option<u64>) -> BatchValidationResult<()> {
        let expected_base_fee = self.base_fee.base_fee();
        match base_fee {
            Some(fee) if fee == expected_base_fee => Ok(()),
            Some(fee) => Err(BatchValidationError::InvalidBaseFee { expected_base_fee, base_fee: fee }),
            None => Err(BatchValidationError::MissingBaseFee),
        }
    }
    ```

2.  **Add Protocol-Level Checks:**
    Ensure that all actors (including the `Primary` and `Engine`) verify that `base_fee_per_gas` is present and correct before accepting or executing a batch.
