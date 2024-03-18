# Low

## [L-01] Redundant branching in `pink/runtime/contract.rs#coarse_grained`

In

[**contract, function coarse_grained**](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/contract.rs#L127C1-L132C10)

```rs
fn coarse_grained<T>(mut result: ContractResult<T>, deposit_per_byte: u128) -> ContractResult<T> {
    result.gas_consumed = mask_gas(result.gas_consumed);
    result.gas_required = mask_gas(result.gas_required);

    match &mut result.storage_deposit {
        StorageDeposit::Charge(v) => {
            *v = mask_deposit(*v, deposit_per_byte);
        }
        StorageDeposit::Refund(v) => {
            *v = mask_deposit(*v, deposit_per_byte); // @audit the same as above
        }
    }
    result
}
```

the branching being made is the same for both cases, where the storage deposit is a `Charge` option and a `Refund` one. It seems it should not be the case as a deposit means a deduction of the funds of the caller in favor of the callee, where a refund is the inverse situation. Consider reviewing this section of the project, as the intention seems to be wrongly implemented.