# Analysis Report

![AltText](https://private-user-images.githubusercontent.com/107410002/313245558-d47d7b11-b730-47a8-b753-78dc851369dd.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTA1MTY2MTUsIm5iZiI6MTcxMDUxNjMxNSwicGF0aCI6Ii8xMDc0MTAwMDIvMzEzMjQ1NTU4LWQ0N2Q3YjExLWI3MzAtNDdhOC1iNzUzLTc4ZGM4NTEzNjlkZC5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjQwMzE1JTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI0MDMxNVQxNTI1MTVaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT0wNGVmNGU3ZWY2MDE2YWNjN2I3NjQzZGYxMjIzNTUxNWI0NTAzZTc0YTI2NzYwMjBmZmUxNjEyMjkyOTYzYWE5JlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCZhY3Rvcl9pZD0wJmtleV9pZD0wJnJlcG9faWQ9MCJ9.ppXfc9DYnbRCe7jsI3uNqrOUgaq_NPPGQY9mmEsEHQQ)

## Table of Contents

- [Approach](#approach)
- [Brief Overview](#brief-overview)
- [Scope and Architecture Overview](#scope-and-architecture-overview)
  - [Runtime Configuration and Contract API](#runtime-configuration-and-contract-api)
    - [pink/runtime/src/runtime.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime.rs)
    - [pink/runtime/src/contract.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/contract.rs)
    - [pink/runtime/src/runtime/pallet_pink.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs)
    - [pink/runtime/src/runtime/extension.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs)
  - [Storage Implementation](#storage-implementation)
    - [pink/runtime/src/storage/mod.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/storage/mod.rs)
    - [pink/runtime/src/storage/external_backend.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/storage/external_backend.rs)
  - [Cross-Library Call Interface (C-API)](#cross-library-call-interface-c-api)
    - [pink/runtime/src/capi/mod.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/mod.rs)
    - [pink/runtime/src/capi/ecall_impl.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs)
    - [pink/runtime/src/capi/ocall_impl.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/ocall_impl.rs)
    - [pink/capi/src/v1/mod.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/capi/src/v1/mod.rs)
    - [pink/capi/src/types.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/capi/src/types.rs)
  - [Chain Extension and Local Cache](#chain-extension-and-local-cache)
    - [pink/chain-extension/src/lib.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/chain-extension/src/lib.rs)
    - [pink/chain-extension/src/local_cache.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/chain-extension/src/local_cache.rs)
- [Systemic Risks](#systemic-risks)
- [Recommendations](#recommendations)
- [Security Researcher Logistics](#security-researcher-logistics)
- [Conclusion](#conclusion)
- [Resources](#resources)

## Approach

Started with a comprehensive walk-through of the provided documentation to understand protocol functionality (heavily based on the pink runtime) and key points, ambiguities were discussed with the sponsors(deva), and possible risk areas were outlined.

Followed by a manual review of each Rust contract in scope, testing function behavior, protocol logic against expectations, and working out potential attack vectors. Comparisons with similar protocols was also performed to identify recurring issues and evaluate fix effectiveness.

Finally, identified issues from the security review were compiled into a comprehensive audit report.

## Brief Overview

The Pink Runtime serves as the execution engine for `ink!` smart contracts within the Phala Network, leveraging the Substrate framework's `pallet-contracts` module along with Phala's specialized chain extensions. As a core component of the Phala ecosystem, it facilitates the running of smart contracts on the blockchain. Designed to operate within the secure environment of off-chain TEE (Trusted Execution Environment) workers, the Pink Runtime is instrumental in processing both on-chain transactions and off-chain RPC queries directed towards `ink!` contracts. This high-level functionality is encapsulated within a dynamically linked library, `libpink.so`, which is compiled for Linux systems and dynamically loaded by the Phala Network's workers to execute smart contracts securely and efficiently.

The architecture of Pink Runtime is distinguished by its dylib interface and an intricate system of cross-calls facilitated through the ecalls/ocalls layer, enabling seamless interactions between the worker environment and the runtime. At its core, the Substrate runtime, complemented by Phala-specific chain extensions, defines the operational foundation of Pink Runtime, ensuring the smooth execution of smart contracts along with the integration of Phala's unique features. This sophisticated structure not only enhances the capability and flexibility of contract execution on Phala Network but also maintains a secure, isolated runtime environment for contracts, thereby safeguarding both contract data and execution logic against external threats and ensuring integrity and confidentiality.

## Scope and Architecture Overview

## Runtime Configuration and Contract API

### [runtime.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime.rs) - 209 SLOC

This file is designed to configure and initialize the custom Substrate-based runtime named `PinkRuntime`, which includes a set of pallets for various blockchain functionalities such as system operations, timestamp management, balance handling, contract execution, and more.

#### Key Components and Functionalities

##### Runtime Configuration

- The module starts by defining types and importing configurations necessary for runtime operations, including types for account IDs, balances, block numbers, hashes, and weight calculations.
- `PinkRuntime` is then constructed using Substrate's `construct_runtime!` macro, incorporating essential pallets like `frame_system`, `pallet_timestamp`, `pallet_balances`, `pallet_contracts`, and a custom `pallet_pink`.

##### Parameter Definitions

- `parameter_types!` macro is used to define various runtime parameters like `BlockHashCount`, `RuntimeBlockWeights`, `ExistentialDeposit`, `MaxCodeLen`, and others that are essential for blockchain operations.

##### Pallet Configuration

- Implements configuration traits for included pallets, setting parameters for system operations, contract executions, balances management, and timestamp functionalities. This includes setting up the gas price for contract execution, deposit requirements for storage, and defining the contract execution schedule.

##### Runtime Initialization Functions

- The module includes functions like `on_genesis`, `on_runtime_upgrade`, and `on_idle`, which are hooks for initializing the runtime, performing migrations or upgrades, and executing tasks when the blockchain is idle.

##### Utility Functions and Macros

- Defines utility functions and macros for operations like masking deposit values and gas weights, which are used to ensure consistency and potentially enhance privacy in transactions.
- Implements functionality for executing contract instantiations and calls, including handling gas payment and refunds based on the execution mode (transaction or query).

#### Short Code Snippets Dive

##### Constructing the Runtime

```rust
construct_runtime! {
    pub struct PinkRuntime {
        System: frame_system,
        // other pallets...
    }
}
```

##### Setting Runtime Parameters

```rust
parameter_types! {
    pub const BlockHashCount: u32 = 250;
    // other parameters...
}
```

##### Implementing Config for Pallets

```rust
impl pallet_balances::Config for PinkRuntime {
    type Balance = Balance;
    // other configurations...
}
```

##### Handling Genesis and Runtime Upgrades

```rust
pub fn on_genesis() {
    <AllPalletsWithSystem as frame_support::traits::OnGenesis>::on_genesis();
}

pub fn on_runtime_upgrade() {
    // Migration logic...
}
```

##### Metadata Check for Testing

```rust
#[test]
fn check_metadata() {
    let (major, minor, _) = this_crate::version_tuple!();
    let filename = format!("assets/metadata-{major}.{minor}.bin");
    check_metadata_with_path(&filename).expect("metadata changed");
}
```

#### Summary

This file sets up the comprehensive runtime environment for the Substrate-based blockchain, detailing configurations for essential blockchain functionalities. It outlines the process for integrating and configuring pallets, setting runtime parameters, and providing hooks for initializing the runtime and handling upgrades or idle state operations.

### [contract.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/contract.rs) - 199 SLOC

This provides the utility functions for instantiating and interacting with the smart contracts in the Substrate-based blockchain environment, specifically within a custom pallet named Pink. It leverages Substrate's `pallet_contracts` for contract management and execution.

#### Key Functionalities

##### Masking Functions

- `mask_low_bits64` and `mask_low_bits128` are macros used to mask the lower bits of a number, ensuring only a specific number of bits are used for calculations. This is particularly useful for normalizing gas and deposit amounts.
- `mask_deposit` and `mask_gas` apply these masking operations to contract deposits and gas weights, respectively, to standardize their values and potentially reduce the granularity of transaction costs.

```rust
fn mask_deposit(deposit: u128, deposit_per_byte: u128) -> u128 { /* implementation */ }
fn mask_gas(weight: Weight) -> Weight { /* implementation */ }
```

##### Contract Interaction

- Defines types `ContractExecResult`, `ContractInstantiateResult`, and `ContractResult` to encapsulate the results of contract executions, instantiations, and general contract calls. These are specialized versions of `pallet_contracts` result types tailored for the Pink pallet.

##### Coarse Graining

- `coarse_grained` function takes a `ContractResult` and applies masking to its gas and deposit values based on the execution mode. This is done to ensure consistency in how transaction costs are reported and to potentially enhance privacy.

```rust
fn coarse_grained<T>(result: ContractResult<T>, deposit_per_byte: u128) -> ContractResult<T> { /* implementation */ }
```

##### Contract Instantiation and Calls

- `check_instantiate_result` checks the result of a contract instantiation, ensuring it was successful and did not revert.
- `instantiate` attempts to instantiate a new contract with the given code hash, input data, and execution arguments. It applies coarse graining to the result if required by the execution mode.
- `bare_call` performs a raw call to a contract method, similarly applying coarse graining to the transaction result as needed.

```rust
pub fn check_instantiate_result(result: &ContractInstantiateResult) -> Result<AccountId> { /* implementation */ }
pub fn instantiate(code_hash: Hash, input_data: Vec<u8>, salt: Vec<u8>, mode: ExecutionMode, args: TransactionArguments) -> ContractInstantiateResult { /* implementation */ }
pub fn bare_call(address: AccountId, input_data: Vec<u8>, mode: ExecutionMode, tx_args: TransactionArguments) -> ContractExecResult { /* implementation */ }
```

##### Internal Contract Transaction Execution

- `contract_tx` is a helper function that wraps the execution of a contract transaction, handling gas payment and refunding based on whether the transaction specifies free gas usage. It encapsulates the logic for deducting gas costs and refunding unused gas.

```rust
fn contract_tx<T>(origin: AccountId, gas_limit: Weight, gas_free: bool, tx_fn: impl FnOnce() -> ContractResult<T>) -> ContractResult<T> { /* implementation */ }
```

#### Summary

This file is designed to facilitate interactions with smart contracts for the Pink pallet. It includes utilities for normalizing transaction costs through bit masking, handling the results of contract interactions, and executing contract instantiations and calls with consideration for gas management and execution modes.

### [pallet_pink.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs) - 168 SLOC

This defines a **pallet**, reusable in the framework, providing functionalities for the execution, gas pricing, treasury management, and contract deployment tracking within a blockchain ecosystem. Below is a detailed explanation of its components:

#### Key Functionalities

##### Pallet Configuration (`Config` Trait)

- **`Config` Trait**: Defines configuration parameters for the pallet, including dependencies on external components such as `frame_system` and currency handling through the `Currency` trait.

```rust
pub trait Config: frame_system::Config {
    type Currency: Currency<Self::AccountId>;
}
```

##### Pallet Storage Items

- **Storage Items**: Various storage items store the pallet's operational parameters and state, such as gas pricing, deposit requirements, the treasury account, and contract-related data.
  - **`ClusterId`**: Stores a unique identifier for the contract cluster.
  - **`GasPrice`, `DepositPerByte`, `DepositPerItem`**: Manage the economic parameters of contract execution.
  - **`TreasuryAccount`**: Holds the account ID of the treasury to which fees are paid.
  - **`Key`**: Stores the private key associated with the contract cluster.
  - **`SidevmCodes`**: A mapping from hash to WASM code for SideVMs, storing contract code.
  - **`SystemContract`**: Stores the account ID of the system contract.
  - **Event Tracking**: `NextEventBlockNumber` and `LastEventBlockHash` track event sequencing and integrity.

##### Pallet Errors (`Error` Enum)

- **`Error` Enum**: Defines potential errors that can occur during the pallet's operation, such as unrecognized chain extension IDs or functions, buffer overflows, and cryptographic issues.

```rust
pub enum Error<T> { ... }
```

##### Pallet Implementation (`Pallet` Struct)

- **`Pallet` Struct**: Acts as the main entry point for the pallet's functionalities, with implementations extending the capabilities of the blockchain's contract layer.

```rust
pub struct Pallet<T>(_);
```

##### Address Generation

- It implements the `AddressGenerator` trait to deterministically generate contract addresses based on inputs such as the deploying address, code hash, and a nonce (salt).

```rust
impl<T: Config + pallet_contracts::Config> AddressGenerator<T> for Pallet<T> { ... }
```

##### Utility Functions

- Functions such as `set_cluster_id`, `set_key`, `put_sidevm_code`, and `set_system_contract` enable configuration and management of contract execution parameters.
- **Economic Management**: Functions to manage gas payment and refunds, adjusting economic parameters related to contract execution.
- **Event Tracking**: Functions to manage the sequencing and tracking of contract-related events, ensuring the integrity and order of events within the blockchain.

##### Type Definitions

- Type aliases (`BalanceOf`, `AccountIdOf`, `HashOf`) improve code readability and maintenance by abstracting complex type definitions.

#### Summary

This pallet facilitates contract deployment, execution monitoring, and economic adjustments, ensuring a flexible and efficient contract ecosystem. Through its storage items, configuration traits, and utility functions, it provides a comprehensive suite of tools for blockchain developers to manage and extend their smart contract capabilities.

### [extension.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs) - 440 SLOC

This Rust code defines a contract that extends the functionalities of pink contracts with additional capabilities, focusing on a chain extension for Pink Runtime. It involves complex interactions with blockchain state, events, cryptography, and contract-specific functionalities.

The contract is designed to provide extended functionalities to pink contracts within the Pink Runtime, a substrate-based blockchain environment. Key components include handling HTTP requests, cryptographic operations, cache management, event logging, and more, specifically tailored for contract execution contexts (query and command).

#### Key Components

##### PinkExtension

Defines a chain extension that enables pink contracts to access extended functionalities not directly available in the substrate pallet.

```rust
#[derive(Default)]
pub struct PinkExtension;
```

Implements the `ChainExtension` trait for Pink Runtime, providing a mechanism to call extended functions using environment variables, handling errors, and ensuring only known extension IDs and function IDs are processed.

```rust
impl ChainExtension<PinkRuntime> for PinkExtension { ... }
```

##### CallInQuery

A structure that encapsulates an account ID representing a contract's address and implements the `PinkRuntimeEnv` and `PinkExtBackend` traits.

```rust
struct CallInQuery { address: AccountId, }
```

It provides functionalities such as HTTP request handling, cryptographic operations, and cache management, leveraging `OCallImpl` for off-chain operations and `DefaultPinkExtension` for some default implementations.

```rust
impl PinkRuntimeEnv for CallInQuery { ... }
impl PinkExtBackend for CallInQuery { ... }
```

##### CallInCommand

Similar to `CallInQuery`, but used in command contexts. It ensures deterministic behavior, which is crucial for transactional operations.

```rust
struct CallInCommand { as_in_query: CallInQuery, }
```

Overrides methods to provide deterministic outputs or simulate the effects of operations that would be nondeterministic or unsupported in transactions, such as HTTP requests or JavaScript evaluations.

```rust
impl PinkExtBackend for CallInCommand { ... }
```

#### Key Functionalities

- **HTTP Requests**: Provides simulated responses for HTTP requests in command contexts, indicating that such requests are unsupported during transactions for determinism.
- **Cryptographic Operations**: Offers cryptographic functionalities like signing and verifying, utilizing the underlying `CallInQuery` implementation or providing default, deterministic responses.
- **Cache Management**: Manages contract-specific cache data, simulating cache operations in command contexts by emitting pink events without actual data modification to maintain determinism.
- **Logging and Random Number Generation**: Supports logging operations and provides deterministic responses for random number generation requests.
- **Querying Contract and System Information**: Facilitates querying various contract and system-related information, such as system contract IDs, account balances, and runtime versions, adapting based on the execution context (query or command).

#### Conclusion

The contract extends pink contracts' capabilities within the Pink Runtime, providing a structured way to interact with blockchain functionalities while adhering to the determinism required in blockchain environments. Through `CallInQuery` and `CallInCommand`, it offers a flexible approach to handle different execution contexts, ensuring that contracts can perform a wide range of operations safely and deterministically.

## Storage Implementation

### [mod.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/storage/mod.rs) - 119 SLOC

This Rust module provides a framework for executing and committing transactions within the storage system, leveraging the external storage and Substrate's state machine (`sp_state_machine`) functionalities.

#### Key Components

##### External and In-Memory Backend

- The module includes references to external and in-memory backends which abstract the specifics of how blockchain state is stored and retrieved, allowing for flexibility in storage implementation.

##### Storage Structure

- **`Storage`** encapsulates a generic backend and provides functionalities to execute code within a specified runtime context, manipulate overlay changes (temporary storage changes before they're committed), and commit these changes to the underlying storage backend.

```rust
pub struct Storage<Backend> {
    backend: Backend,
}
```

#### Main Functionalities

##### Executing with Context

- **`execute_with`** function allows executing a closure with access to the blockchain state, capturing any side effects and overlay changes. This function sets the environment based on the execution context, including the current block number and timestamp, and handles the start and end of transactions.

```rust
pub fn execute_with<R>(&self, exec_context: &ExecContext, f: impl FnOnce() -> R) -> (R, ExecSideEffects, OverlayedChanges<Hashing>)
```

##### Committing Changes

- **`execute_mut`** is similar to `execute_with` but additionally commits the changes captured in the overlay to the backend storage, making them persistent.

```rust
pub fn execute_mut<R>(&mut self, context: &ExecContext, f: impl FnOnce() -> R) -> (R, ExecSideEffects)
```

- **`commit_changes`** directly commits overlay changes to the storage backend, transforming the ephemeral state changes into persistent storage modifications.

```rust
pub fn commit_changes(&mut self, changes: OverlayedChanges<Hashing>)
```

##### Managing Storage Transactions

- The **`CommitTransaction`** trait is defined for storage backends capable of committing a series of changes as a transaction. Implementations of this trait must define how to commit these transactions to their specific storage medium.

```rust
pub trait CommitTransaction: StorageBackend<Hashing> {
    fn commit_transaction(&mut self, root: Hash, transaction: BackendTransaction<Hashing>);
}
```

#### System Event Blocks

- **`maybe_emit_system_event_block`** function allows for the emission of system event blocks, packaging events generated by contract executions into blocks that can be externally verified for integrity. This functionality supports privacy while enabling external accessibility and verification of contract events.

```rust
pub fn maybe_emit_system_event_block(events: SystemEvents)
```

#### Summary

This module establishes the comprehensive system for executing the blockchain logic within a customizable storage context, efficiently handling storage transactions, and facilitating the emission of verifiable system events. It abstracts the complexities of interacting with the blockchain storage layer, providing a flexible and robust foundation for blockchain and smart contract development.

### [external_backend.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/storage/external_backend.rs) - 18 SLOC

This module outlines an **`ExternalDB`** struct, a custom implementation of the **`TrieBackendStorage`** trait from the Substrate framework, designed to interface with an external key-value storage system. This setup allows the blockchain runtime to interact with storage not managed directly by the blockchain node, but rather, through "ocalls" (outward calls to the host environment or operating system).

##### External Database (`ExternalDB`)

- **`ExternalDB`**: Acts as a bridge for the blockchain trie storage backend to interact with an external storage system. It does not manage storage directly but forwards storage operations to an external handler via ocalls.

```rust
pub struct ExternalDB;
```

##### External Backend and Storage Types

- **`ExternalBackend` & `ExternalStorage`**: These types are specialized versions of the `TrieBackend` and `Storage` constructs, respectively, using `ExternalDB` for their storage operations. This allows the trie operations to be conducted with data that resides outside the blockchain node's direct control.

```rust
pub type ExternalBackend = TrieBackend<ExternalDB, Hashing>;
pub type ExternalStorage = Storage<ExternalBackend>;
```

#### Key Functionalities

##### Storage Reads and Writes

- The implementation of **`TrieBackendStorage`** for `ExternalDB` overrides the `get` method to fetch data from the external storage. This is done by making an ocall to `OCallImpl.storage_get`, passing the requested key and receiving the value, if any, from the external storage system.

```rust
impl TrieBackendStorage<Hashing> for ExternalDB { ... }
```

##### Committing Transactions

- **`CommitTransaction`** for `ExternalBackend` enables the backend to commit a set of changes to the external storage. Changes are sent via `OCallImpl.storage_commit`, encapsulating the new root hash and the list of key-value changes to be applied.

```rust
impl CommitTransaction for ExternalBackend { ... }
```

##### Instantiating External Storage

- **`instantiate`** function for `ExternalStorage` constructs a new instance of external storage. It starts by fetching the current storage root via an ocall (`OCallImpl.storage_root()`), then builds a trie backend around `ExternalDB` with this root, and finally wraps this backend in a `Storage` instance.

```rust
impl ExternalStorage { ... }
```

#### Helper Functions

##### Code Existence Check

- A utility function **`code_exists`** checks for the presence of contract code in the external storage. It generates a storage key specific to the code hash and queries the external storage to verify its existence.

```rust
pub mod helper { ... }
```

#### Summary

This module showcases how storage operations can be extended to utilize external storage systems. By delegating storage operations through `ocalls`, it facilitates interaction with storage mechanisms not inherently managed by the blockchain node, enabling more flexible and decentralized storage solutions. This setup is particularly useful in scenarios where off-chain data needs to be securely accessed and manipulated by on-chain logic, bridging the gap between the blockchain and external systems.

## Cross-Library Call Interface (C-API)

### [capi/mod.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/mod.rs) - 53 SLOC

This defines the entry point and function dispatching for a runtime environment specifically designed for pink contracts. It involves interacting with a C interface, managing initialization parameters, and routing external calls (ecalls) to their respective implementations.

#### Key Components

##### Imports and Setup

Imports necessary modules and types from `pink_capi::v1` and other crates. `logger` is used for logging purposes, and `OCallImpl` is a module that presumably handles outgoing calls from the contract.

```rust
use pink_capi::v1;
use v1::*;

use phala_sanitized_logger as logger;

pub(crate) use ocall_impl::OCallImpl;
```

##### Runtime Initialization

Defines a constant to ensure the `__pink_runtime_init` function matches the `init_t` type. This function initializes the runtime environment, setting up necessary configurations and ecall functions.

```rust
const _: init_t = Some(__pink_runtime_init);
```

- `__pink_runtime_init()`

Marks the beginning of the runtime execution. It initializes the runtime with provided configuration and populates the ecall table with supported ecall functions.

- **Safety**: It's marked `unsafe` due to direct pointer manipulations.
- **Arguments**:
  - `config`: A pointer to the runtime configuration.
  - `ecalls`: A mutable pointer to the ecalls table that will be populated by this function.

```rust
#[no_mangle]
pub unsafe extern "C" fn __pink_runtime_init(config: *const config_t, ecalls: *mut ecalls_t) -> ::core::ffi::c_int { ... }
```

##### Version Reporting

Defines a function to report the runtime's version. This is likely used by external callers to determine the capabilities or version of the runtime they are interacting with.

```rust
unsafe extern "C" fn get_version(major: *mut u32, minor: *mut u32) { ... }
```

##### Ecall Dispatching

Serves as a central dispatcher for all ecall functions. It decodes input data and routes the call to the appropriate function within the runtime based on the `call_id`.

- **Safety**: Marked `unsafe` due to raw pointer interactions.
- **Arguments**:
  - `call_id`: Identifies the function to be called.
  - `data`, `len`: Pointers to the input data buffer and its length.
  - `ctx`: A context passed to the `output_fn`.
  - `output_fn`: A function pointer to handle the output from the ecall.

```rust
unsafe extern "C" fn ecall(call_id: u32, data: *const u8, len: usize, ctx: *mut ::core::ffi::c_void, output_fn: output_fn_t) { ... }
```

#### Summary

The contract is essentially a bridge between the pink contracts runtime and external (possibly non-Rust) environments, facilitating initialization, configuration, and function dispatching. It demonstrates how Rust can interact with C-style interfaces, using unsafe code for pointer manipulation and FFI interactions. The contract provides a structured way to initialize the runtime, handle versioning, and route external calls to their respective implementations, showcasing a practical example of extending Rust-based blockchain runtime with external functionalities.

### [ecall_impl.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs) - 273 SLOC

Comprehensive implementation of the ECall (Enclave Call) interface for a blockchain or enclave runtime, facilitating the execution of smart contracts and interaction with the blockchain state. It integrates closely with the external environment, managing contract instantiation, execution, and communication between the contract and the blockchain.

#### Key Components

##### ECallImpl Structure

- **`ECallImpl`**: Serves as the primary implementation for handling ECall operations. It doesn't contain state by itself but defines methods to interact with the blockchain state and external calls.

```rust
pub struct ECallImpl;
```

##### Storage Access

- **`storage` Function**: Instantiates an external storage interface, allowing contracts to interact with off-chain storage.

```rust
pub(crate) fn storage() -> crate::storage::ExternalStorage { ... }
```

##### Context Instrumentation

- **`instrument_context` Macro**: Instruments the execution context with additional tracing and logging for diagnostics. It logs the block number and request ID, aiding in debugging and monitoring.

```rust
macro_rules! instrument_context { ... }
```

##### Execution Interface

- **Implementing `Executing` for `ExternalStorage`**: Allows executing functions within a managed execution context, automatically handling logging, side effects, and state changes.

```rust
impl Executing for crate::storage::ExternalStorage { ... }
```

##### Key Functionalities

- **`ECalls` Implementation**: Defines a series of operations related to smart contracts, including their setup, instantiation, and execution. Key functionalities include:
  - **`cluster_id`**: Retrieves the unique identifier of the contract cluster.
  - **`setup`**: Initializes the contract cluster with specific configuration parameters.
  - **`deposit`**: Handles the deposit of funds to a specific account.
  - **`set_key` and `get_key`**: Manages the cryptographic key associated with the contract.
  - **`upload_code`**: Uploads the smart contract code to the blockchain.
  - **`upload_sidevm_code` and `get_sidevm_code`**: Manages SideVM (possibly a virtual machine or execution environment) code uploading and retrieval.
  - **`system_contract`**: Retrieves the system contract's account identifier.
  - **`free_balance` and `total_balance`**: Queries the balance information of a specific account.
  - **`code_hash`**: Fetches the hash of the contract code associated with an account.
  - **`contract_instantiate` and `contract_call`**: Handles the instantiation and execution of contracts.
  - **`git_revision`**: Returns the git revision of the codebase, aiding in version tracking.
  - Lifecycle hooks like **`on_genesis`**, **`on_runtime_upgrade`**, and **`on_idle`** for contract lifecycle management.

```rust
impl ecall::ECalls for ECallImpl { ... }
```

##### Utility Functions

- **`sanitize_args`**: Adjusts the transaction arguments based on the execution mode to ensure reasonable gas limits and prevent abuse.
- **`handle_deposit`**: Facilitates deposit transactions as part of contract call preparations.

#### Summary

This module is a crucial part of the contract execution framework, bridging the gap between the sandboxed execution environment and the blockchain. It ensures that smart contracts can be instantiated, executed, and interact with the blockchain securely and efficiently. Through the implementation of the `ECalls` interface, it provides a rich set of functionalities for contract management, execution, and interaction with the blockchain state, all while ensuring security and determinism in contract execution.

### [ocall_impl.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/ocall_impl.rs) - 92 SLOC

This module provides a comprehensive setup for handling outwards calls (Ocalls) from a sandboxed environment, such as a blockchain smart contract or a secure enclave, to the outside world. It's part of a broader framework likely aimed at enhancing blockchain or enclave applications with external functionalities through a cross-call interface.

The core of this module revolves around the ability to make calls from a sandboxed environment to the host environment's functions. This is crucial for extending the functionalities of smart contracts or secure enclaves by leveraging external services or resources. The setup involves:

- **Initialization**: Ensuring that a valid OCall function is set up at runtime initialization. If not, the default panic behavior enforces the necessity of this setup.
- **Cross-Call Execution**: Providing a standardized way (`cross_call` and `cross_call_mut` methods) to execute these calls, with safety ensured through raw pointers and external function pointers.

In environments with custom memory management needs, such as secure enclaves or smart contracts, it's essential to have a way to manage memory allocations that align with the host's expectations. The conditional inclusion of a custom allocator allows this module to seamlessly integrate with the host environment's memory management schemes, ensuring efficient and accurate memory usage tracking.

The module includes tests to ensure that setting the OCall function works as expected and that the default OCall behavior is correctly panic-inducing. These tests, alongside the careful use of `unsafe` blocks, underscore the importance of safety and correctness in cross-boundary function calls.

#### Key Components

##### OCall Setup and Default Behavior

- **`OCALL` Static Variable**: Holds the actual OCall function that's provided by the host environment. Initialized with a default function (`_default_ocall`) that panics, indicating no function has been provided yet.

```rust
static mut OCALL: InnerType<cross_call_fn_t> = _default_ocall;
```

- **`_default_ocall` Function**: A placeholder OCall function that panics. This is used until a real OCall function is set via `set_ocall_fn`.

```rust
unsafe extern "C" fn _default_ocall(...) { panic!("No ocall function provided"); }
```

##### Setting the OCall Function

- **`set_ocall_fn` Function**: Allows setting the actual OCall function provided by the external environment. This function also sets up custom allocator and deallocator functions if available and the `allocator` feature is enabled.

```rust
pub(super) fn set_ocall_fn(ocalls: ocalls_t) -> Result<(), &'static str> { ... }
```

##### OCall Implementation

- **`OCallImpl` Struct**: Represents the implementation of the cross-call interface for outgoing calls.

```rust
pub(crate) struct OCallImpl;
```

- **Cross-call Interfaces**: `OCallImpl` implements both `CrossCall` and `CrossCallMut` traits, providing mechanisms for making synchronous calls to external functions.

```rust
impl CrossCall for OCallImpl { ... }
impl CrossCallMut for OCallImpl { ... }
```

##### Allocator Setup

The module conditionally includes a custom memory allocator setup, redirecting allocation and deallocation calls to the external environment, ensuring memory usage statistics remain accurate across dynamic runtime boundaries.

```rust
#[cfg(feature = "allocator")]
mod allocator { ... }
```

#### Summary

This module lays the foundation for a flexible and secure mechanism to extend the capabilities of sandboxed environments like blockchain smart contracts or secure enclaves. It carefully manages the intricacies of cross-call setup, execution, and memory management, providing a critical bridge between the sandboxed and host environments.

### [v1/mod.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/capi/src/v1/mod.rs) - 225 SLOC

This works on scenarios where contracts need to communicate with external systems or require data from outside their isolated execution environment.

#### Module: Types

Imports custom data types from a separate module, preparing them for use in defining interfaces and data structures throughout the current context.

#### Traits: CrossCall and CrossCallMut

Define the foundational interface for making cross-boundary calls, allowing contracts to request operations outside their execution scope. The `CrossCall` trait facilitates read-only operations, while `CrossCallMut` supports mutations.

#### Trait: Executing

Specifies methods for executing provided closures either immutably or mutably, enabling flexible execution patterns based on the current execution context.

#### Struct: IdentExecute

An implementation of the `Executing` trait that directly executes provided functions without modification, serving as a straightforward execution path.

#### Traits: ECall and OCall

Marker traits that don't introduce any methods but signify different types of cross-call interactions. `ECall` is used for inward calls into the contract environment, while `OCall` represents outward calls from the contract.

#### Module: ecall

Focused on inward-bound calls (Ecalls), allowing contracts to perform various operations like managing cluster setup, handling balances, and interacting with contract code. Each operation is uniquely identified and designed to facilitate essential contract lifecycle and interaction tasks.

##### Structs: TransactionArguments and ClusterSetupConfig

Define data structures for specifying the arguments to transactions and the configuration for cluster setup, respectively. These structs include fields for managing balances, gas limits, and contract deployment specifics.

##### Trait: ECalls

Defines a comprehensive set of operations available to contracts, including cluster management, balance adjustments, contract code handling, and more. Each method is associated with a unique cross-call ID, enabling precise and controlled execution of contract requests.

#### Module: ocall

Targets outward-bound calls (Ocalls), defining the means through which contracts can interact with external systems and perform operations like storage management, logging, caching, and HTTP requests.

##### Struct: ExecContext

Provides detailed context about the current execution environment, including mode, block number, and timestamp, enabling contracts to make informed decisions based on their operational context.

##### Trait: OCalls

Outlines a broad set of functionalities for contracts to interact with their environment, including storage operations, logging, cache management, HTTP requests, and handling system events. This trait ensures contracts have access to necessary external data and services, supporting complex and dynamic contract behaviors.

#### Test Coverage

Includes test setups and coverage for ctypes, demonstrating instantiation and cloning of test structures to ensure the reliability and functionality of defined types and operations.

#### Summary

The provided Rust code constructs a comprehensive framework for smart contract execution, enabling complex interactions within a blockchain environment. By defining clear interfaces for cross-boundary calls, execution patterns, and external operations, it facilitates a wide range of contract behaviors, from simple value transfers to complex interactions with external data sources and systems.

### [types.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/capi/src/types.rs) - 86 SLOC

This contract uses several basic blockchain types, an execution mode enumeration to distinguish between different runtime contexts, and a structure to handle events and their side effects resulting from contract execution. Here's a closer look at the key functionalities:

#### Basic Types Definitions

The contract establishes fundamental blockchain types such as `Hash`, `AccountId`, `Balance`, and `BlockNumber`, employing `BlakeTwo256` as the default hashing algorithm. These types are essential for creating, identifying, and managing accounts, transactions, and other blockchain entities.

#### Execution Mode Enumeration

`ExecutionMode` is an enum that identifies the current context in which the runtime is executing: `Query`, `Estimating`, or `Transaction`. Each mode is designed for specific operations:

- **Query**: Used for RPC queries where state changes are discarded post-execution, and indeterministic operations like HTTP requests are permitted.
- **Estimating**: Simulates transactions to estimate gas usage without committing state changes.
- **Transaction**: Real transaction execution where state changes are permanent, and indeterministic operations are restricted.

Utility methods associated with `ExecutionMode` facilitate checks on the current mode and determine operational behaviors like gas estimation and the requirement for deterministic execution.

#### ExecSideEffects Enum

`ExecSideEffects` captures the potential side effects of contract execution. In its current version (`V1`), it contains vectors for events emitted by Pink and ink! contracts and a list of instantiated contracts. A noteworthy feature is its ability to filter events permissible in query contexts, allowing for a controlled execution environment that respects the constraints of different runtime modes.

#### Practical Implications

This contract structure provides a robust framework for managing the execution context and handling the side effects of contract operations. By distinguishing between querying, estimating, and transaction modes, it ensures that contracts can operate effectively in various scenarios, from gas estimation to actual transaction execution.

The explicit handling of execution side effects and events also adds a layer of transparency and control, allowing contract developers and runtime environments to manage how contract actions influence the broader system and interact with other contracts.

In summary, the contract's design promotes secure, efficient, and flexible blockchain operations within the Pink Protocol environment, catering to both the deterministic requirements of blockchain transactions and the need for flexibility in queries and gas estimation.

## Chain Extension and Local Cache

### [lib.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/chain-extension/src/lib.rs) - 458 SLOC

This contract showcases the integration of `HTTP` requests, cryptographic operations, caching, and logging within a blockchain runtime environment. Below is a summary of key functionalities, broken down for clarity:

##### Core Structs and Traits

- `PinkRuntimeEnv`: Defines a trait for accessing the address of the current environment, ensuring compatibility with various blockchain account formats.
- `DefaultPinkExtension`: A struct that encapsulates the environment and provides default implementations for the PinkExtBackend trait, facilitating interactions with external HTTP services, cryptographic operations, and more.

#### HTTP Requests

- `http_request` and `batch_http_request`: Functions to perform synchronous and asynchronous HTTP requests, respectively, with support for handling request errors, setting request timeouts, and processing batch requests efficiently.
- `async_http_request`: An asynchronous function to execute an HTTP request, handling URL parsing, client creation, method determination, header processing, and response handling, including error scenarios.

#### Cryptographic Operations

- Implements signing and verification functions for different signature types (`sr25519`, `ed25519`, `ecdsa`), allowing for the generation and verification of cryptographic signatures using various key types.
- Provides functionality to derive keys and extract public keys from given seeds, supporting secure cryptographic operations within the contract.

#### Caching and Logging

- Caching operations (`cache_set`, `cache_get`, `cache_remove`, `cache_set_expiration`) allow storing and retrieving data efficiently, with the capability to set expiration times for cached entries.
- The logging function enables outputting messages at different severity levels (error, warn, info, debug, trace) to the console, facilitating debugging and monitoring of contract execution.

#### Utility Functions and Types

- `LimitedWriter`: A custom writer that imposes a size limit on the amount of data that can be written, preventing excessive resource usage.
- Various helper functions and types facilitate the conversion between different data representations, error handling, and execution of futures.

The code includes comprehensive unit tests to verify the functionality of HTTP request processing, cryptographic operations, caching, logging, and other features provided by the contract. These tests ensure the reliability and security of the contract's operations within a blockchain environment.

This contract demonstrates a sophisticated integration of network communication, cryptographic security, data management, and debugging facilities within a blockchain runtime, aiming to provide a robust foundation for developing decentralized applications on the Pink network.

### [local_cache.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/chain-extension/src/local_cache.rs) - 371 SLOC

The LocalCache module is designed to provide off-chain, local key-value (KV) storage for smart contracts. This cache allows for data storage that is unique to each instance of the contract running on different nodes or machines, with the understanding that stored data may be lost upon restarts or due to expiration mechanisms. Here's a closer look at key functionalities within this module:

#### Key Components and Functionalities

##### LocalCache and Storage Structs

- `LocalCache`: Manages multiple `Storage` instances, each identified by a contract ID, providing a mechanism to store, retrieve, and manage cached data with considerations for garbage collection (GC), expiration, and quota limits.
- `Storage`: Represents the storage for a single contract, containing key-value pairs (`kvs`), tracking the total size of stored data, and enforcing a maximum storage size to adhere to quotas.

##### Cache Operations

- `set`, `get`, `remove`, `set_expiration`: Core operations to manipulate the cache, including setting and getting values, removing entries, and setting custom expiration times for entries.
- `apply_cache_op`: Applies higher-level cache operations (`CacheOp`) like setting values, setting expiration times, and removing entries, abstracted for ease of use within smart contracts.

##### Garbage Collection and Expiration

- `clear_expired`: Removes expired entries based on the current time to free up space and ensure data freshness.
- `fit_size`: Ensures the storage does not exceed its quota by removing the least recently used or soonest to expire entries until the total size is within limits.
- `maybe_clear_expired`: Periodically triggers `clear_expired` based on a predefined interval (`gc_interval`) to maintain cache hygiene without overly frequent checks.

##### Test Mode and Quota Management

- `enable_test_mode`, `apply_quotas`: Special functions for testing and development, allowing the simulation of different cache environments and managing storage quotas for each contract.

##### Thread Safety and Test Isolation

- Implements thread-local storage for test mode, ensuring that tests run in a multi-threaded environment do not interfere with each other by providing isolated cache instances per test case.

#### Summary

The LocalCache module offers a robust, flexible caching solution for off-chain computations in smart contracts. It intelligently manages data with expiration, garbage collection, and storage quotas, ensuring efficient use of resources. The module's design caters to both operational needs and development/testing environments, showcasing a well-thought-out approach to local caching within blockchain applications.

## Systemic Risks

- The runtime configuration heavily relies on external pallets (pallet_contracts, pallet_balances, etc.) and traits from frame_support and sp_runtime. Any breaking changes or bugs in these dependencies could directly impact the PinkRuntime. Regular dependency audits and adherence to semantic versioning can mitigate this risk.
- There are multiple shprt hands in testing cases, for example for the Pink Runtime contract.rs, the presence of a single test function (`mask_low_bits_works`) shows the limited unit testing coverage. Comprehensive testing, including edge cases for gas masking and contract execution scenarios, is essential for ensuring the reliability and security of these utilities.
- Looking at the Ocall implementation, while some parts of the code are self-explanatory, adding detailed comments and documentation, especially for the public API and unsafe blocks, would greatly enhance maintainability and understandability.
- Protocol demonstrates potential security risks due to extensive unsafe code and lack of proper error handling and input validation.
- The design assumes that all cached values have the same default lifetime (`default_value_lifetime`). This might not be flexible enough for all use cases. Allowing custom lifetimes per item could provide more granular control over cache expiration.
- External dependencies from, inherited contracts/struct, outsorced calls etc.

## Recommendations

- Enhance documentation quality.
- Re-enhance event monitoring, error logging cyrre implementation subtly suggests that event tracking isn't maximized. And there exist multiple instances where the errors beig logged are same for different cases.
- Improve protocol's testability, one thing to note about tests is that _there's always room for further refinement and improvement._, a few out of the box ideas need to also be attached to test cases, currently most files include very little test cases.
- To support the above, even more background tes cases could be employed, that's to say the `local_cache.with_global_cache()` function uses a Mutex in non-test mode and RefCell in test mode. While Mutex provides thread safety, RefCell allows borrowing for mutation, but only one mutable borrow can exist at a time. This can lead to issues if the cache is accessed from multiple threads concurrently in non-test mode leading to an inequality in execution results from test and non-test modes.
- There seems to be a need to improve the naming conventions in some areas of protocol to ease user/developer experience while trying to use/understand protocol.
- Onboard more developers, cause multiple eyes checking out a protocol can be invaluable. More contributors can significantly reduce potential risks and oversights.

## Security Researcher Logistics

My attempt on reviewing the Codebase spanned around 60 hours distributed over 9 days:

- 4 hours dedicated to writing this analysis.
- 6 hours (first day) spent exploring the whole system to grasp the foundations before diving into it line by line
- 2 hours were allocated for discussions with sponsors on the private discord group regarding potential vulnerabilities.
- The remaining time was spent on finding issues and writing the report for each of them.

## Conclusion

The codebase was a very great learning experience, though it was a pretty hard nut to crack being purely based on the Rust language, nonetheless during my review, I uncovered a few issues within the protocol and they need to be fixed. Recommended steps should be taken to protect the protocol from potential attacks. Timely audits and codebase cleanup for mitigations should also be conducted to keep the codebase fresh and up to date with evolving security times.

## Resources

- **Documentation:** https://docs.phala.network/developers/phat-contract

- **Website:** https://phala.network/

- **Audit Page:** https://code4rena.com/audits/2024-03-phat-contract-runtime#top

- **Documentation for the Phala Blockchain:** https://github.com/Phala-Network/phala-blockchain/tree/master/docs


### Time spent:
060 hours