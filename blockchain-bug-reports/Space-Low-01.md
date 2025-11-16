# Unrestricted nominated events allow low-cost spam on nodes

## Description

Stakers are able to nominate validators using the `Staking.sol::nominate()` function.

This function accepts an array of Ed25519 public keys, performs a simple duplicate check, and emits an event containing all the pubkeys:
```solidity
    function nominate(bytes32[] calldata nodesEd25519PubKeys) external {
        if (nodesEd25519PubKeys.length == 0) revert EmptyNodesList();
        if (nodesEd25519PubKeys[0] == bytes32(0)) revert InvalidNodeEd25519PubKey();

        uint256 nodesEd25519PubKeysLength = nodesEd25519PubKeys.length;
        for (uint256 i = 1; i < nodesEd25519PubKeysLength; ++i) {
            // solhint-disable-next-line gas-strict-inequalities
            if (nodesEd25519PubKeys[i] <= nodesEd25519PubKeys[i - 1]) {
                revert DuplicateNodeEd25519PubKey();
            }
        }

        emit Nominated(nodesEd25519PubKeys, msg.sender);
    }
```

Later, this `Nominated` event will be processed by the `system_tables` pallet, which calls `process_nominating`:
```rust
            SystemRequestType::Staking(StakingSystemRequest::Nominate) => {
                process_nominating::<T>(request)
            }
```

Inside `process_nominating`, the function calls `string_to_address_list` to convert all Ed25519 public keys to an address list by iterating over each key:
```rust
                        let nominations =
                            sxt_core::utils::string_to_address_list::<T>(nodes.clone());
```

```rust
pub fn string_to_address_list<T: frame_system::Config>(
    address_list: String,
) -> Vec<<T::Lookup as StaticLookup>::Source> {
    address_list
        .split(',')
        .filter_map(|s| {
            Some(<T as frame_system::Config>::Lookup::unlookup(
                eth_address_to_substrate_account_id::<T>(s.trim()).ok()?,
            ))
        })
        .collect()
}
```

### The problem
There are two key issues here:
1. `Staking.sol::nominate()` does not check whether the caller is a staker. This means any account can call this function and emit arbitrary `Nominated` events.
2. During processing of these events, the sxt-node uses `string_to_address_list`, which iterates over every pubkey included in the event. This causes unnecessary CPU and memory load on nodes.

As a result, an attacker can flood the chain with `Nominated` events containing a large number of public keys, causing unnecessary resource pressure on nodes.

Even though the actual nomination will later fail (since the caller has no stake), the damage is already done before reaching that point, because the iteration and account translation happen before the `pallet_staking::nominate()` call:
```rust
                        let nominator = hex::encode(nominator);
                        let nominator_id =
                            sxt_core::utils::eth_address_to_substrate_account_id::<T>(&nominator)?;
                        let nominations =
=>                          sxt_core::utils::string_to_address_list::<T>(nodes.clone());
                        let nominator_signer: OriginFor<T> = RawOrigin::Signed(nominator_id).into();
=>                      pallet_staking::Pallet::<T>::nominate(
                            nominator_signer,
                            nominations.clone(),
                        )?;
                        Ok(())
                    }
```

Since there's no upfront stake requirement or filtering on the contract side, this attack is low-cost (only gas fees) and can be executed repeatedly.

## Proof of Concept
To test the scenario, please add the following `use` statement at the top of `/sxt-node/pallets/system_tables/src/tests.rs`:
```diff
use core::str::from_utf8;

use frame_support::{assert_err, assert_ok};
use sp_core::crypto::AccountId32;
use sp_core::U256;
use sp_runtime::DispatchError;
use sxt_core::tables::TableIdentifier;
use sxt_core::utils::{convert_account_id, eth_address_to_substrate_account_id};
+use sxt_core::utils::string_to_address_list;

use crate::mock::*;
use crate::parse::{
    StakingSystemRequest, SystemFieldValue, SystemRequest, SystemRequestType, SystemTableField,
};
```

Also, add the following test case to the same file:
```rust
#[test]
fn nomination_by_non_staker_with_large_node_list_should_process_all_nodes() {
    new_test_ext().execute_with(|| {
        // Fresh ETH address that hasn't bonded
        let unbonded_eth = "1111111111111111111111111111111111111111";

        // Simulate spam: 10,000 fake public keys
        let pubkeys: Vec<String> = (0..10_000)
            .map(|i| format!("{:040x}", i)) // valid 20-byte hex
            .collect();
        let node_list = pubkeys.join(",");

        let fields = vec![
            SystemTableField::with_value(
                "NOMINATOR".to_string(),
                SystemFieldValue::Bytes(hex::decode(unbonded_eth).unwrap()),
            ),
            SystemTableField::with_value(
                "NODES".to_string(),
                SystemFieldValue::Varchar(node_list.clone()),
            ),
        ];

        let request = SystemRequest {
            request_type: SystemRequestType::Staking(StakingSystemRequest::Nominate),
            table_id: TableIdentifier::from_str_unchecked("NOMINATE", "SXT_SYSTEM_STAKING"),
            fields,
        };

        // Process the nomination request
        assert_ok!(crate::process_nominating::<Test>(request));

        // Confirm that nomination was rejected (since nominator is not bonded)
        let nominator_id =
            eth_address_to_substrate_account_id::<Test>(unbonded_eth).expect("valid address");

        let recorded_nominations =
            pallet_staking::Nominators::<Test>::get(&nominator_id);

        assert!(
            recorded_nominations.is_none(),
            "Expected no recorded nominations for unbonded nominator"
        );

        // Check that all pubkeys were processed (node count matches)
        let parsed_node_count = string_to_address_list::<Test>(node_list);
        assert_eq!(
            parsed_node_count.len(),
            pubkeys.len(),
            "Expected all nodes to be parsed and processed"
        );
    });
}
```

Run the test:
```bash
$ cargo test nomination_by_non_staker_with_large_node_list_should_process_all_nodes
```

## Recommendation

- Require the caller to be a valid staker in `Staking.sol::nominate()` before emitting the event.
- Limit the maximum number of public keys accepted per call.
