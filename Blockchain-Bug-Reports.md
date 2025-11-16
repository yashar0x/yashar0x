# L1/L2 Bug Reports

Here I’ve included my L1/L2 bug reports.

## About

I’m a Web3 security researcher focused on deep protocol analysis and smart-contract security across a wide range of languages and ecosystems, including Rust, Go, Clarity, Vyper and Solidity.

I’m also the co-founder of [**Shred Security**](https://shredsec.xyz), where I work on securing DeFi protocols, L1/L2 infrastructure, and complex on-chain systems. My work emphasizes adversarial thinking, high-signal findings, and practical guidance that teams can actually use to harden their code.

## Contact

- **Twitter:** [yashar0x](https://x.com/yashar0x)
- **Discord:** [yashar0x](https://discordapp.com/users/1116969574009688094)

## Bug Reports

| Project | Title | Severity | Language | Platform | Report |
| --- | --- | --- | --- | --- | --- |
| Movement Labs | Attackers can drain the sequencer’s wallet and DoS network by submitting transactions from unfunded accounts | **Critical** | Rust | Immunefi |     |
| Movement Labs | Premature transaction acceptance to mempool/DA without signature validation | **High** | Rust | Immunefi |     |
| Stacks | Unvalidated withdrawal events allow data manipulation and denial of service in Emily | **Low** | Rust | Immunefi |     |
| Andromeda | Stakers Funds Will Be Permanently Locked Within the Contract if a Validator is Tombstoned | **Medium** | Rust | Sherlock |     |
| Space and Time | Low Existential Deposit allows mass account creation for near-zero cost | **Medium** | Rust | Cantina |     |
| Space and Time | initiateUnstake will unstake the full staked amount | Medium | Rust | Cantina |     |
| Space and Time | Malicious indexer can cause storage bloat | Medium | Rust | Cantina |     |
| Space and Time | Unrestricted nominated events allow low-cost spam on nodes | Low | Rust | Cantina |     |
| Optimism Java | Lack of Idle connection handling may DoS the entire consensus layer | Medium | Java | Cantina |     |
| Optimism Java | Race condition in RpcServer connectionHandler allows exceeding maxActiveConnections | Low | Java | Cantina |
