# QA Report

## Table of Contents

| Issue ID                                                                             | Description                                                           |
| ------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| [QA-01](#qa-01-unnecessary-gas-refund-in-pallet-pinks-contract_tx-function)          | Unnecessary Gas Refund in Pallet Pink's `contract_tx` function        |
| [QA-02](#qa-02-reconsider-hardcoded-constants-configuration-values-from-the-runtime) | Reconsider hardcoded constants configuration values from the runtime  |
| [QA-03](#qa-03-update-code-and-remove-testing-variables)                             | Update code and remove testing variables                              |
| [QA-04](#qa-04-limited-error-handling-in-externaldbs-get-function)                   | Limited Error Handling in ExternalDB's `get` Function                 |
| [QA-05](#qa-05-fix-typos)                                                            | Fix typos                                                             |
| [QA-06](#qa-06-the-public-transactionarguments-struct-should-be-clearly-documented)  | The public `TransactionArguments` struct should be clearly documented |

## QA-01 Unnecessary Gas Refund in Pallet Pink's `contract_tx` function

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/contract.rs#L232-L258

```rust
fn contract_tx<T>(
    origin: AccountId,
    gas_limit: Weight,
    gas_free: bool,
    tx_fn: impl FnOnce() -> ContractResult<T>,
) -> ContractResult<T> {
    if !gas_free {
        if let Err(err) = PalletPink::pay_for_gas(&origin, gas_limit) {
            return ContractResult {
                gas_consumed: Weight::zero(),
                gas_required: Weight::zero(),
                storage_deposit: Default::default(),
                debug_message: Default::default(),
                result: Err(err),
                events: None,
            };
        }
    }
    let result = tx_fn();
    //@audit
    if !gas_free {
        let refund = gas_limit
            .checked_sub(&result.gas_consumed)
            .expect("BUG: consumed gas more than the gas limit");
        PalletPink::refund_gas(&origin, refund).expect("BUG: failed to refund gas");
    }
    result
}
```

In this code, if `gas_limit` is equal to `result.gas_consumed`, the `checked_sub` operation will return `Some(Weight::zero())`. However, the `expect` method will still succeed, and the code will attempt to refund zero gas using `PalletPink::refund_gas`.

### Impact

The current implementation of `contract_tx` attempts to refund gas even when the gas consumed by the contract is exactly equal to the gas limit. This can lead to unnecessary calls to the `PalletPink::refund_gas` function, potentially impacting performance and code clarity.

### Recommended Mitigation Steps

**Check for zero refund before calling `PalletPink::refund_gas`:**

```rust
if !gas_free {
  let refund = gas_limit.checked_sub(&result.gas_consumed)
      .expect("BUG: consumed gas more than the gas limit");
  if refund != Weight::zero() {
    PalletPink::refund_gas(&origin, refund).expect("BUG: failed to refund gas");
  }
}
```

This approach explicitly checks if the refund amount is zero before calling `PalletPink::refund_gas`. This avoids unnecessary function calls and improves code clarity.

## QA-02 Reconsider hardcoded constants configuration values from the runtime

### Proof of Concept

Substrate runtimes, including the provided `PinkRuntime`, often use hardcoded constants to define various operational parameters. These include `MAX_CODE_LEN`, `MaxLocks`, `MaxReserves`, and `MaxHolds`, which play critical roles in limiting resource usage, ensuring network security, and preventing abuse. However, the static nature of these values can pose challenges to network adaptability and scalability.

Consider the following snippet from the runtime configuration: https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime.rs#L114

```rust
const MAX_CODE_LEN: u32 = 2 * 1024 * 1024;
parameter_types! {
    pub const MaxLocks: u32 = 50;
    pub const MaxReserves: u32 = 50;
    pub const MaxHolds: u32 = 10;
}
```

These parameters directly influence the runtime's behavior, with `MAX_CODE_LEN` limiting the maximum allowable size of smart contract code, and `MaxLocks`, `MaxReserves`, and `MaxHolds` dictating the limits on user account operations.

### Impact

Hardcoding these values means that any adjustments, whether for optimization, security, or feature enhancements, necessitate a runtime upgrade. Runtime upgrades, while supported by Substrate, are non-trivial operations requiring on-chain governance decisions and coordination among network participants. This can lead to situations where necessary adjustments to these parameters are delayed or avoided, potentially impacting network performance, security, and the ability to respond to emerging threats or opportunities.

For example, if the network were to experience a surge in smart contract deployment, the hardcoded `MAX_CODE_LEN` might either become a bottleneck for legitimate use cases or, conversely, allow oversized contracts that could strain network resources.

### Recommended Mitigation Steps

Implement mechanisms that allow these parameters to be adjusted via on-chain governance. This approach enables network stakeholders to propose, vote on, and enact changes to critical parameters without requiring a full runtime upgrade.

```rust
#[pallet::storage]
pub(super) type MaxCodeLen<T: Config> = StorageValue<_, u32, ValueQuery>;
```

## QA-03 Update code and remove testing variables

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime.rs#L123-L133

```rust
    pub DefaultSchedule: Schedule<PinkRuntime> = {
        let mut schedule = Schedule::<PinkRuntime>::default();
        const MB: u32 = 16;  // 64KiB * 16
        // Each concurrent query would create a VM instance to serve it. We couldn't
        // allocate too much here.
        schedule.limits.memory_pages = 4 * MB;
        schedule.instruction_weights.base = 8000;
        schedule.limits.runtime_memory = 2048 * 1024 * 1024; // For unittests @audit
        schedule.limits.payload_len = 1024 * 1024; // Max size for storage value
        schedule
    };
```

As hinted by the "@audit" sign, there are currently limits thatwere placed due to them being for testing purposes, but project is pending final release and could be released with them.

### Impact

Bad code structure, deployment with unnecessary variables

### Recommended Mitigation Steps

Remove unneccessary/unused section of codes before finalizing project.

## QA-04 Limited Error Handling in ExternalDB's `get` Function

### Proof of Concept

Take a look at this from the `ExternalDB` struct:

```rust
impl TrieBackendStorage<Hashing> for ExternalDB {
  fn get(&self, key: &Hash, _prefix: Prefix) -> Result<Option<DBValue>, DefaultError> {
    Ok(OCallImpl.storage_get(key.as_ref().to_vec()))
  }
}
```

The `get` function attempts to retrieve data using the `OCallImpl.storage_get` function and wraps the result in a `Result`. However, it uses the generic `DefaultError` type, which doesn't provide specific details about the failure.

### Impact

The current implementation of the `get` function in the `ExternalDB` struct returns a `Result<Option<DBValue>, DefaultError>`. This approach provides limited information about potential errors during data retrieval using OCalls.

When an error occurs, debuggers or monitoring systems only see a generic "DefaultError". This makes it difficult to pinpoint the exact cause of the issue (e.g., OCall failure, network issues, errors on the host side).

### Recommended Mitigation Steps

Define a custom error type that captures specific reasons for failures. This could include:

- `OCallError`: Indicates a general OCall failure.
- `NetworkError`: Indicates a network issue during the OCall.
- `HostError`: Indicates an error reported by the host system.

Here's an example with a custom error type:

```rust
enum StorageError {
  OCallError,
  NetworkError,
  HostError(String),
}

impl From<sp_state_machine::DefaultError> for StorageError {
  fn from(_: sp_state_machine::DefaultError) -> Self {
    StorageError::OCallError
  }
}

impl TrieBackendStorage<Hashing> for ExternalDB {
  fn get(&self, key: &Hash, _prefix: Prefix) -> Result<Option<DBValue>, StorageError> {
    OCallImpl.storage_get(key.as_ref().to_vec())
      .map_err(|err| match err {
        // Handle specific OCall errors here (e.g., network issues)
        _ => StorageError::OCallError,
      })
  }
}
```

This approach provides more context about the error, aiding in debugging and troubleshooting.

## QA-05 Fix typos

### Proof of Concept

Take a look at [https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs#L74-L78

```rust
    /// The priviate key of the cluster
    #[pallet::storage]
    #[pallet::getter(fn key)]
    pub(crate) type Key<T: Config> = StorageValue<_, Sr25519SecretKey>;

```

### Impact

Unreadable/Non-understandable code

### Recommended Mitigation Steps

Change `   /// The priviate key of the cluster` to:
`   /// The private key of the cluster`

## QA-06 The public `TransactionArguments` struct should be clearly documented

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/capi/src/v1/mod.rs#L50-L65

```rust
    pub struct TransactionArguments {
        /// The sender's AccountId.
        pub origin: AccountId,
        /// The amount of Balance transferred to the contract.
        pub transfer: Balance,
        /// The maximum gas amount for this transaction.
        pub gas_limit: Weight,
        //@audit
        /// Indicates whether the transaction requires gas. If `true`, no tokens are reserved for gas.
        pub gas_free: bool,
        /// The storage limit for this transaction. `None` indicates no limit.
        pub storage_deposit_limit: Option<Balance>,
        /// The balance to deposit to the caller address, used when the caller's balance is insufficient
        /// for a gas estimation.
        pub deposit: Balance,
    }

```

This struct has 5 implementations and contains the parameters for a contract call or instantiation, case with this is that the dpcumentation around the `gas_free` bool is faulty, it currently states that this bool value "Indicates whether the transaction **requires gas**..." , whereas as the name suggests it should instead be "Indicates whether the transaction **does not require gas**..." since if the value is `true`, no tokens are reserved for gas.

### Impact

Low, confused docs, make sit harder for users and developers to interact with code

### Recommended Mitigation Steps

Make changes as suggested in _Proof Of Concept_.
