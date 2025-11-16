# initiateUnstake will unstake the full staked amount

## Description

Stakers are able to unstake a portion of their stake by calling `Staking.sol::initiateUnstake()` and providing the desired `amount`:
```solidity
    function initiateUnstake(uint248 amount) external requireState(msg.sender, StakerState.Staked) whenNotPaused {
        initiateUnstakeRequestsTimestamp[msg.sender] = uint64(block.timestamp);
        stakerState[msg.sender] = StakerState.UnstakeInitiated;
        emit UnstakeInitiated(msg.sender, amount);
    }
```

Withdrawing only a portion of the stake is also the expected behavior, as described in the comments in `CollaborativeStaking.sol`:
```solidity
/// 4. The staker initiates an unstake of 500 tokens, which moves 500 tokens back to this contract.
```

However, in practice, when the `sxt-node` processes the `UnstakeInitiated` event, it does not take into account the actual `amount` requested by the user. Instead, it unbonds the entire staked amount:
```rust
    /// Parse a system request to initiate unstaking
    pub fn process_unstake_initiated<T: Config>(request: SystemRequest) -> DispatchResult {
        request
            .rows()
            .map(|row| -> DispatchResult {
                match row.get("STAKER") {
                    Some(SystemFieldValue::Bytes(staker)) => {
                        let staker = hex::encode(staker);
                        let staker_id =
                            sxt_core::utils::eth_address_to_substrate_account_id::<T>(&staker)?;
                        let staker_signer: OriginFor<T> =
                            RawOrigin::Signed(staker_id.clone()).into();

                        let raw_balance: u128 =
                            pallet_balances::Pallet::<T>::free_balance(staker_id)
                                .unique_saturated_into();
                        let staking_balance: T::CurrencyBalance = T::CurrencyBalance::from(
                            UniqueSaturatedInto::<u64>::unique_saturated_into(raw_balance),
                        );
                        pallet_staking::Pallet::<T>::unbond(staker_signer, staking_balance)
                            .map_err(|e| e.error)?;
                        Ok(())
                    }
                    _ => Err(Error::<T>::MissingExpectedField.into()),
                }
            })
            .for_each(emit_for_error::<T>);

        Ok(())
    }
```

As shown above, the `amount` field is not fetched from the event. Instead, the code uses `free_balance` of the user:
```rust
                        let raw_balance: u128 =
                            pallet_balances::Pallet::<T>::free_balance(staker_id)
                                .unique_saturated_into();
                        let staking_balance: T::CurrencyBalance = T::CurrencyBalance::from(
                            UniqueSaturatedInto::<u64>::unique_saturated_into(raw_balance),
                        );
```

This causes all available bonded balance to be unbonded, regardless of what the user requested.

## Proof of Concept

To demonstrate the issue, add the following test case to `/sxt-node/pallets/system_tables/src/tests.rs`:
```rust
#[test]
fn unstake_ignores_event_amount_and_unbonds_full_bonded() {
    new_test_ext().execute_with(|| {
        let staker_eth = ETH_TEST_WALLET;
        let bonded_amount = 500;
        let claimed_amount = 100;

        let staker_id = eth_address_to_substrate_account_id::<Test>(staker_eth).unwrap();

        let stake_msg = get_staked_message(staker_eth, bonded_amount.into());
        assert_ok!(crate::process_staking::<Test>(stake_msg));

        let ledger = pallet_staking::Pallet::<Test>::ledger(sp_staking::StakingAccount::Stash(staker_id.clone())).unwrap();
        assert_eq!(ledger.active, bonded_amount);

        let unstake_msg = SystemRequest {
            request_type: SystemRequestType::Staking(StakingSystemRequest::UnstakeInitiated),
            table_id: TableIdentifier::from_str_unchecked("UNSTAKE", "SXT_SYSTEM_STAKING"),
            fields: vec![
                SystemTableField::with_value(
                    "STAKER".to_string(),
                    SystemFieldValue::Bytes(hex::decode(staker_eth).unwrap()),
                ),
                SystemTableField::with_value(
                    "AMOUNT".to_string(),
                    SystemFieldValue::Decimal(U256::from(claimed_amount)),
                ),
            ],
        };

        assert_ok!(crate::process_unstake_initiated::<Test>(unstake_msg));

        let ledger_after = pallet_staking::Pallet::<Test>::ledger(sp_staking::StakingAccount::Stash(staker_id)).unwrap();
        assert_eq!(ledger_after.active, 0, "Active bonded should be zero");
        assert_eq!(ledger_after.total, bonded_amount, "Total should still reflect original bonded amount");
    });
}
```

Run the test:

```bash
$ cd /sxt-node/pallets/system_tables/src/
$ cargo test unstake_ignores_event_amount_and_unbonds_full_bonded
```

## Recommendation

The system should respect the `amount` specified by the user in the `UnstakeInitiated` event.
