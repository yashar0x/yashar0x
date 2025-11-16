# L1/L2 Bug Reports

Here I’ve included L1/L2 bugs found by Shred Security Researchers

## Bug Reports

| Project | Title | Severity | Language | Platform | Report |
| --- | --- | --- | --- | --- | --- |
| Movement Labs | Attackers can drain the sequencer’s wallet and DoS network by submitting transactions from unfunded accounts | Critical | Rust | Immunefi |  [LINK](./blockchain-bug-reports/yashar/Move-Crit-01.md)   |
| Movement Labs | Premature transaction acceptance to mempool/DA without signature validation | High | Rust | Immunefi |  [LINK](./blockchain-bug-reports/yashar/Move-High-01.md)   |
| Telecoin Network | Lack of Per-Transaction Gas Limit Allows Single Transaction to Block Entire Batch in Telcoin Network | High | Rust | Cantina |  [LINK](./blockchain-bug-reports/chupinexx/Telecoin-Finding-High-1.md)   |
| Telecoin Network | Incorrect Assignment of Delegator and Validator Fields in Delegation Struct Results in Delegator Losing Reward Claim Rights | High | Rust | Cantina |  [LINK](./blockchain-bug-reports/chupinexx/L1-L2-Findings/blob/main/Telecoin-Finding-High-2.md)   |
| Telecoin Network | Lack of Mandatory Base Fee Enforcement Enables Fee-Free Attacks | Medium | Rust | Cantina |  [LINK](./blockchain-bug-reports/chupinexx/Telecoin-Finding-Medium-1.md)   |
| Telecoin Network | Low-Level Call Allows EIP-7702 Wallets to Block Slashing/Burn, Enabling Denial-of-Service Against Protocol Enforcement | Medium | Rust | Cantina |  [LINK](./blockchain-bug-reports/chupinexx/Telecoin-Finding-Medium-2.md)   |
| Telecoin Network | Double-Decrement of Committee Length After Ejection Causes Incorrect Committee Size Validation | Low| Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/Telecoin-Finding-Low-1.md)  |
| Telecoin Network | Unsafe Non-Blocking Shutdown Lead To Database Corruption and Extended Validator Downtime | Info | Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/Telecoin-Finding-Info-1.md)  |
| Telecoin Network | Incorrect Committee Size Validation During Slashing of PendingExit Validators Causes Unintended Reverts | Info | Rust | Cantina |  [LINK](./blockchain-bug-reports/chupinexx/Telecoin-Finding-info-2.md)   |
| Stacks | Unvalidated withdrawal events allow data manipulation and denial of service in Emily | Low | Rust | Immunefi |  [LINK](./blockchain-bug-reports/yashar/Stacks-Low-01.md)   |
| Andromeda | Stakers Funds Will Be Permanently Locked Within the Contract if a Validator is Tombstoned | Medium | Rust | Sherlock |  [LINK](./blockchain-bug-reports/yashar/Andromeda-Medium-01.md)   |
| Space and Time | Critical Replay and State Confusion in Staking Contract Allows Double Withdrawal of Unstaked Funds | High | Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/SXT-Finding-High-1.md)  |
| Space and Time | validateMessage Fails with Unordered Signatures, Leading to Permanent Loss of Access to Unstaked Funds | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/SXT-Finding-Medium-1.md)  |
| Space and Time | Hardcoded 10-Day EVM Timelock Desynchronizes with SXT Chain's 7-Day Unbonding, Impairing Unstake Cancellation Utility | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/SXT-Finding-Medium-2.md)  |
| Space and Time | Users Trapped in UnstakeClaimed State with No Recovery Path After SXT Chain Reorganization | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/SXT-Finding-Medium-3.md])  |
| Space and Time | Protocol Performs Unfunded Work and Pays Gas for Queries Submitted with Minimal Deposits | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/SXT-Finding-Medium-4.md)  |
| Space and Time | Malicious customLogicContractAddress Allows Diversion of Protocol Fees and Gas Reimbursements| Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/chupinexx/SXT-Finding-Medium-5.md)  |
| Space and Time | Low Existential Deposit allows mass account creation for near-zero cost | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/yashar/Space-Medium-01.md)  |
| Space and Time | initiateUnstake will unstake the full staked amount | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/yashar/Space-Medium-02.md)  |
| Space and Time | Malicious indexer can cause storage bloat | Medium | Rust | Cantina |  [LINK](./blockchain-bug-reports/yashar/Space-Medium-03.md)   |
| Space and Time | Unrestricted nominated events allow low-cost spam on nodes | Low | Rust | Cantina |   [LINK](./blockchain-bug-reports/yashar/Space-Low-01.md)  |
| Optimism Java | Lack of Idle connection handling may DoS the entire consensus layer | Medium | Java | Cantina |   [LINK](./blockchain-bug-reports/yashar/OP-Medium-01.md)  |
| Optimism Java | Race condition in RpcServer connectionHandler allows exceeding maxActiveConnections | Low | Java | Cantina |   [LINK](./blockchain-bug-reports/yashar/OP-Low-01.md)  |
