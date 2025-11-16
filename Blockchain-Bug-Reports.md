# L1/L2 Bug Reports

Here I’ve included L1/L2 bugs found by Shred Security Researchers

## Bug Reports

| Project | Title | Severity | Language | Platform | Report |
| --- | --- | --- | --- | --- | --- |
| Movement Labs | Attackers can drain the sequencer’s wallet and DoS network by submitting transactions from unfunded accounts | Critical | Rust | Immunefi |  [LINK](./blockchain-bug-reports/Move-Crit-01.md)   |
| Movement Labs | Premature transaction acceptance to mempool/DA without signature validation | High | Rust | Immunefi |  [LINK](./blockchain-bug-reports/yashar/Move-High-01.md)   |
| Stacks | Unvalidated withdrawal events allow data manipulation and denial of service in Emily | Low | Rust | Immunefi |  [LINK](./blockchain-bug-reports/yashar/Stacks-Low-01.md)   |
| Andromeda | Stakers Funds Will Be Permanently Locked Within the Contract if a Validator is Tombstoned | Medium | Rust | Sherlock |  [LINK](./blockchain-bug-reports/yashar/Andromeda-Medium-01.md)   |
| Space and Time | Low Existential Deposit allows mass account creation for near-zero cost | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/yashar/Space-Medium-01.md)  |
| Space and Time | initiateUnstake will unstake the full staked amount | Medium | Rust | Cantina |   [LINK](./blockchain-bug-reports/yashar/Space-Medium-02.md)  |
| Space and Time | Malicious indexer can cause storage bloat | Medium | Rust | Cantina |  [LINK](./blockchain-bug-reports/yashar/Space-Medium-03.md)   |
| Space and Time | Unrestricted nominated events allow low-cost spam on nodes | Low | Rust | Cantina |   [LINK](./blockchain-bug-reports/yashar/Space-Low-01.md)  |
| Optimism Java | Lack of Idle connection handling may DoS the entire consensus layer | Medium | Java | Cantina |   [LINK](./blockchain-bug-reports/yashar/OP-Medium-01.md)  |
| Optimism Java | Race condition in RpcServer connectionHandler allows exceeding maxActiveConnections | Low | Java | Cantina |   [LINK](./blockchain-bug-reports/yashar/OP-Low-01.md)  |

