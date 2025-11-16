### **Title:** Lack of Per-Transaction Gas Limit Allows Single Transaction to Block Entire Batch in Telcoin Network

### **Finding Description**

The Telcoin Network’s transaction processing pipeline fails to enforce a per-transaction gas limit, allowing any user to submit a transaction with a `gas_limit` up to the batch gas limit (e.g., 30,000,000). This “gas bomb” transaction consumes the entire batch’s gas capacity, preventing other transactions from being included, even if they have higher priority fees or are critical for network operation. The vulnerability affects the transaction pool (`WorkerTxPool`), batch builder, and batch validation stages, as none check whether a transaction’s `gas_limit` is reasonable for its execution needs. By submitting a transaction with a high `gas_limit` and a competitive tip, an attacker can reliably execute a denial-of-service attack, censor transactions, and manipulate transaction ordering, breaking the network’s ordering guarantees. The attack is permissionless, cheap, and sustainable, as the attacker only pays for the gas actually used, not the declared `gas_limit`.

The vulnerability stems from the absence of per-transaction gas limit checks across the transaction processing pipeline. Below is a detailed analysis of the relevant code paths:

#### 1. Transaction Pool Admission (WorkerTxPool)
The transaction pool only validates basic transaction properties, such as sender balance and nonce order, but does not check the `gas_limit` for reasonableness.

**Code Context (implied from description):**
```rust
// In WorkerTxPool::add_transaction
if tx.sender_balance >= tx.cost && tx.nonce == expected_nonce {
    // Accept transaction
} else {
    // Reject transaction
}
```
No check ensures that `tx.gas_limit` is within a reasonable range (e.g., proportional to typical transaction gas usage). A transaction with `gas_limit = 30,000,000` (the batch limit) is accepted if it meets other criteria.

#### 2. Batch Builder Selection
The batch builder selects transactions greedily based on priority (tip/fee) and checks if their cumulative gas limit fits within the batch gas limit:

**Code Context:**
```rust
if current_total_gas + tx.gas_limit <= batch_gas_limit {
    // Include transaction
}
```
The batch builder does not validate whether a transaction’s `gas_limit` is excessive relative to its type or actual gas usage. A transaction with `gas_limit = 30,000,000` and a high tip is prioritized and included, consuming the entire batch’s capacity, even if it uses only a fraction of that gas (e.g., 100,000).

#### 3. Batch Validation and Execution
Validators and the execution engine verify that the sum of transaction gas limits in a batch does not exceed the batch gas limit but do not check individual transaction gas limits.

This means Peers accept batches containing a single transaction with an excessive `gas_limit`, as long as the total is within the batch limit. During execution, only the actual gas used is charged, allowing the transaction to waste batch space without cost to the attacker.

### **Attack Propagation**

The attack is executed as follows:

1.  A malicious user submits a transaction with:
    *   `gas_limit = 30,000,000` (equal to the batch gas limit).
    *   A high priority tip to ensure selection.
    *   Minimal actual gas usage (e.g., 100,000 for a simple transfer).
2.  The transaction pool accepts `Tx` due to the lack of gas limit checks.
3.  The batch builder prioritizes `Tx` due to its high tip and includes it in the batch, as `30,000,000 <= 30,000,000`.
4.  No other transactions can fit, as the remaining gas capacity is zero.
5.  Validators and the engine accept the batch, as it meets the total gas limit rule.
6.  The transaction executes, using only its actual gas (e.g., 100,000), and the attacker pays only for that amount.
7.  The batch is finalized with only `Tx`, wasting the remaining gas capacity (e.g., 29,900,000).

#### **Example Scenario:**

*Batch Gas Limit: 30,000,000.*

**Transaction Pool:**
| Tx | gas_limit | Tip/Priority | Actual Gas Used |
| :--- | :--- | :--- | :--- |
| A | 29,900,000 | High | 100,000 |
| B | 300,000 | Medium | 200,000 |
| C | 300,000 | Low | 250,000 |

**Outcome:**
*   `Tx_A` is included due to its high tip.
*   `Tx_B` and `Tx_C` are skipped, as `30,000,000 - 29,900,000 = 100,000` is insufficient.
*   The batch processes only `Tx_A`, using 100,000 gas, wasting 29,900,000 gas.

#### **Ordering Manipulation**

The lack of gas limit enforcement also enables unfair transaction ordering:

*Batch Gas Limit: 30,000,000.*

**Transaction Pool:**
| Tx | gas_limit | Tip/Priority | Actual Gas Used |
| :--- | :--- | :--- | :--- |
| 1 | 29,900,000 | High | 100,000 |
| 2 | 300,000 | Medium | 100,000 |
| 3 | 100,000 | Low | 100,000 |

**Outcome:**
*   `Tx_1` is included due to its high tip.
*   `Tx_2` is skipped, as `30,000,000 - 29,900,000 = 100,000` is insufficient.
*   `Tx_3` is not skipped and it's included because `30,000,000 - 29,900,000 = 100,000` is sufficient.
*   The batch processes incorrect order by including `Tx1` and `Tx3` and excluding `TX2` that should be the second Tx in the order.

### **Economic and Network Impact**

*   **Throughput Denial of Service:** The attacker reduces the network’s transactions per second (TPS) to near zero by filling batches with single, wasteful transactions.
*   **Transaction Censorship:** Legitimate transactions are excluded, delaying or preventing their processing, which impacts users and applications .
*   **Fee Market Distortion:** Users are incentivized to inflate their `gas_limit` to compete for inclusion, leading to inefficient block space usage and higher fees.
*   **Economic Viability:** The attack is cheap, as the attacker pays only for actual gas used (e.g., 100,000 gas), not the declared `gas_limit`, making it sustainable and repeatable.

### **Impact Explanation**

The impact is **high** because this vulnerability allows any user to cheaply and reliably block all other transactions from being included in a batch, resulting in denial-of-service, transaction censorship, wasted block space, and manipulation of transaction ordering. This directly undermines the network’s throughput, fairness, and reliability.

### **Likelihood Explanation**

The likelihood is **high** because the attack requires no special permissions or privileged access—any user with a wallet can exploit it simply by submitting a transaction with a high gas limit and tip. There are no protocol or implementation defences in place, making this attack trivial to execute and highly probable in practice.

### **Recommendation**

To mitigate this vulnerability, the Telcoin protocol and implementation should enforce a maximum per-transaction gas limit that is significantly lower than the batch/block gas limit (e.g., 1/10 or 1/20 of the batch limit). This check should be applied at multiple layers for defence-in-depth:

#### 1. Transaction Pool Admission:
Reject any transaction with a `gas_limit` exceeding the configured per-transaction maximum when it is submitted to the pool.

```rust
// Pseudocode for pool admission
const MAX_TX_GAS_LIMIT: u64 = 3_000_000; // Example: 1/10th of 30,000,000

if tx.gas_limit > MAX_TX_GAS_LIMIT {
    return Err("Transaction gas limit exceeds allowed maximum");
}
```

#### 2. Batch/Block Construction:
Ensure the batch builder does not include transactions with a `gas_limit` above the maximum, even if they are present in the pool.

#### 3. Batch Validation:
Honest peers and validators should reject any batch containing a transaction with a `gas_limit` above the maximum, providing a second line of defence.
