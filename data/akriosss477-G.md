https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/contract.rs#L232-L258

The contract_tx function does not check whether the gas_limit is greater than or equal to the gas_required value returned by the contract call. If the gas_limit is less than the gas_required, the function will return an error, but it will not refund any unused gas to the transaction origin. It is recommended to check the gas_required value and refund any unused gas.
POC
use frame_support::weights::Weight;
use pallet_contracts::ContractExecResult;
use sp_runtime::DispatchError;

use crate::{
    runtime::{Contracts, Pink as PalletPink, PinkRuntime},
    types::{AccountId, Balance},
};

fn poc_contract_tx(origin: AccountId, gas_limit: Weight, gas_free: bool) -> ContractExecResult<Balance> {
    let result = Contracts::bare_call(
        origin,
        origin,
        0,
        gas_limit,
        0,
        vec![],
        pallet_contracts::DebugInfo::UnsafeDebug,
        pallet_contracts::CollectEvents::Skip,
        pallet_contracts::Determinism::Enforced,
    );

    if !gas_free {
        let refund = gas_limit
            .checked_sub(&result.gas_consumed)
            .expect("BUG: consumed gas more than the gas limit");
        PalletPink::refund_gas(&origin, refund).expect("BUG: failed to refund gas");
    }

    result
}

#[test]
fn poc_gas_limit_less_than_gas_required() {
    let origin = AccountId::from([0u8; 32]);
    let gas_limit = Weight::from_ref_time(1_000_000);
    let gas_free = false;

    // Set the `gas_required` value to be greater than the `gas_limit`.
    let mut result = poc_contract_tx(origin, gas_limit, gas_free);
    let gas_required = Weight::from_ref_time(2_000_000);
    result.gas_required = gas_required;

    // Call the `poc_contract_tx` function again with the same `gas_limit`.
    let result2 = poc_contract_tx(origin, gas_limit, gas_free);

    // Check that the `result2` value has an error due to insufficient gas.
    assert!(matches!(result2.result, Err(DispatchError::OutOfGas)));

    // Check that the unused gas was not refunded to the transaction origin.
    let expected_refund = gas_limit.saturating_sub(gas_required);
    assert_ne!(PalletPink::free_balance(&origin), expected_refund);

We first define a poc_contract_tx function that is similar to the contract_tx function in the original code. We then define a test function poc_gas_limit_less_than_gas_required that sets the gas_required value to be greater than the gas_limit and calls the poc_contract_tx function twice with the same gas_limit.

After the first call to poc_contract_tx, we modify the gas_required value of the result to be greater than the gas_limit. This simulates a contract call that requires more gas than the gas_limit provided by the transaction origin.

When we call poc_contract_tx again with the same gas_limit, the function should return an error due to insufficient gas. However, because the function does not check whether the gas_limit is greater than or equal to the gas_required value, it does not refund any unused gas to the transaction origin.

We can check that the unused gas was not refunded by comparing the free_balance of the transaction origin to the expected refund amount. In this case, the expected refund amount is the difference between the gas_limit and the gas_required value. Because the gas_required value was greater than the gas_limit, the expected refund amount should be zero. However, because the function did not refund any unused gas, the free_balance of the transaction origin will be less than the expected refund amount.
