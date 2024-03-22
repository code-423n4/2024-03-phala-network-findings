## Approach Taken-in Evaluating The Phala

- **Initial Scope and Documentation Review**: Thoroughly went through the Contest Readme File  to understand the protocol's objectives and functionalities.
    
- **High-level Architecture Understanding**: Performed an initial architecture review of the codebase by going through all files without going into function details.
    
- **Test Environment Setup and Analysis**: Set up a test environment and execute all tests. Additionally, use Static Analyzer tools like Slither to identify potential vulnerabilities.
    
- **Comprehensive Code Review**: Conducted a line-by-line code review focusing on understanding code functionalities.
    
- **Report Writing**: Write a Report by compiling all the insights I gained throughout the line by line code review.
    

# Overview

## About Pink Runtime

`pink-runtime` is the `ink!` contract execution engine for Phala Network, built on Substrate's `pallet-contracts` with custom chain extensions, written in Rust.  
It executes smart contracts on the Phala Network. It is compiled to a Linux shared object, `libpink.so`. It is loaded and runs in the Phala Network's off-chain TEE workers.

See the [<ins>README</ins>](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/README.md) for more information.

# System Overview

### Scope

```
- src/runtime.rs
- src/contract.rs
- src/storage/mod.rs
- src/storage/external_backend.rs
- src/runtime/pallet_pink.rs
- src/capi/mod.rs
- src/capi/ecall_impl.rs
- src/capi/ocall_impl.rs
- src/runtime/extension.rs
- src/v1/mod.rs
- src/types.rs
- src/lib.rs
- src/local_cache.rs
```

## Here's a high-level analysis of each contract:

## <ins>pink/runtime/src/runtime.rs</ins>

&nbsp;this contract serves as the foundation for a substrate-based blockchain runtime, providing essential configurations, parameters, and functionalities necessary for deploying decentralized applications and managing smart contracts.

In general, this contract represents the configuration and setup of a substrate runtime environment called `PinkRuntime`.

1.  **Modular Design**:

- The contract is organized into modules and imports, allowing for a structured approach to managing dependencies and functionalities.

2.  **Type Definitions**:

- Type aliases are used to define fundamental types, enhancing code readability and flexibility.

3.  **Runtime Construction**:

- The `construct_runtime!` macro is utilized to construct the runtime, defining core components such as the system, timestamp, balances, randomness, contracts, and a custom module named `Pink`.

4.  **Parameter Configurations**:

- Constants and parameter types are defined to configure various aspects of the runtime, such as block hash count, block weights, existential deposit, etc.

5.  **Pallet Configurations**:

- Configurations for pallets like balances, timestamp, contracts, and randomness are provided, specifying how each pallet interacts within the runtime.

6.  **Schedule Definition**:

- The default schedule for contract execution is defined, setting limits on memory, instruction weights, runtime memory, payload length, etc.

7.  **Contracts Configuration**:

- Configuration for the contracts pallet is established, governing smart contract operation parameters like time, randomness, currency, scheduling, deposits, etc.

8.  **Lifecycle Management**:

- Functions are provided for genesis initialization, runtime upgrades, and idle processing, managing the lifecycle of the runtime effectively.

## <ins>pink/runtime/src/contract.rs</ins>

In simple terms, this contract provides functionalities related to contract instantiation and method invocation within a substrate-based blockchain runtime.. It includes functions for creating contracts, calling contract methods, and managing gas payments and refunds during contract execution.

1.  **Macro and Helper Functions**:

- Macros and helper functions are defined to manipulate and mask numbers for various purposes, such as masking low bits and manipulating gas values.

2.  **Masking Functions**:

- Functions are provided to mask low bits of numbers, such as gas and deposits, to ensure proper handling and security in contract execution.

3.  **Coarse Grained Function**:

- A function is implemented to apply coarse-grained adjustments to gas, storage deposits, and gas requirements based on specified conditions.

4.  **Instantiation and Call Functions**:

- Functions are defined for contract instantiation (`instantiate`) and method invocation (`bare_call`). These functions handle parameters, gas limits, determinism, and other factors related to contract execution.

5.  **Transaction Handling**:

- A function (`contract_tx`) is provided to manage transactions related to gas payment and refund, ensuring proper handling of gas limits and refunds after contract execution.

&nbsp;

## <ins>pink/runtime/src/storage/mod.rs</ins>

This contract defines functionalities related to storage management, execution of runtime code segments, and emission of substrate runtime events within a substrate-based blockchain runtime environment.  It provides functionalities for interacting with the storage backend, executing code securely, and ensuring transparency and accessibility of runtime events.

1.  **CommitTransaction Trait**:

- Defines a trait `CommitTransaction` for committing transactions to the storage backend.

2.  **Storage Execution Functions**:

- Provides functions (`execute_with` and `execute_mut`) for executing runtime code segments within a storage context. It manages transactional state and changes to the storage backend.

3.  **Backend Transaction Functions**:

- Defines functions for converting overlayed changes into backend transactions (`changes_transaction`) and committing storage changes to the backend (`commit_changes`).

4.  **Emission of Substrate Runtime Events**:

- Defines a function (`maybe_emit_system_event_block`) responsible for emitting substrate runtime events. It prepares and sends out runtime events for external accessibility and verification purposes.

5.  **Submodules**:

- Contains submodules (`external_backend` and `in_memory_backend`) for different storage backend implementations.

## <ins>pink/runtime/src/storage/external\_backend.rs</ins>

this contract facilitates interaction between the substrate runtime and an external key-value store by implementing the necessary database interfaces and providing functions for reading and writing data to the external store via ocalls. Additionally, it includes a helper function to check the existence of specific ink code in the storage.

1.  **Purpose and Design**:

- The purpose of this database type is to act as an interface between the substrate runtime and an external key-value store.
- It delegates key-value reads and writes to the host environment via ocalls (out-of-call interop).

2.  **ExternalDB Struct**:

- `ExternalDB` struct represents the database type that implements the `TrieBackendStorage` trait.
- It contains methods for reading (`get`) and committing transactions (`commit_transaction`) to the external key-value store.

3.  **ExternalBackend and ExternalStorage Type Aliases**:

- `ExternalBackend` is a type alias for `TrieBackend` with `ExternalDB` and `Hashing`.
- `ExternalStorage` is a type alias for `Storage` using `ExternalBackend`.

4.  **Implementation of TrieBackendStorage**:

- The `get` method implementation retrieves values from the external storage using ocalls.
- It takes a key and a prefix, and returns the value associated with the key.

5.  **Implementation of CommitTransaction Trait**:

- The `commit_transaction` method implementation commits transactions to the external storage.
- It takes a root hash and a backend transaction, converts the transaction into changes, and commits them to the external storage using ocalls.

6.  **Instantiation of ExternalStorage**:

- The `instantiate` function creates an instance of `ExternalStorage`.
- It retrieves the root hash from the external storage or initializes an empty trie root if not found, then builds the backend and creates the storage.

7.  **Helper Module**:

- Contains a helper function `code_exists` to check the existence of a particular ink code in the storage.
- It generates the storage key based on the code hash and checks if the key exists in the storage.

## <ins>pink/runtime/src/runtime/pallet\_pink.rs</ins>

This code defines a pallet (module) containing various configurations, storage items, and functions for managing a blockchain runtime, including gas management, contract deployment, system settings, and event handling.

1.  **Pallet Module**:

- This module is annotated with the `frame_support::pallet` attribute, indicating that it's a substrate pallet.
- It contains the pallet definition, including its configurations and implementation.

2.  **Dependencies**:

- It imports several dependencies from various crates, including `frame_support`, `pallet_contracts`, `phala_crypto`, `scale`, `scale_info`, `sp_core`, and `sp_runtime`.
- These dependencies provide necessary traits, types, and functionalities for building the pallet.

3.  **Configuration**:

- It defines a trait `Config` that extends `frame_system::Config` and specifies an associated type `Currency`.
- This trait is used to parameterize the pallet with specific runtime configurations.

4.  **Storage Items**:

- It declares several storage items using the `#[pallet::storage]` attribute.
- These storage items store data persistently on the blockchain and include `ClusterId`, `GasPrice`, `DepositPerByte`, `DepositPerItem`, `TreasuryAccount`, `Key`, `SidevmCodes`, `SystemContract`, `NextEventBlockNumber`, and `LastEventBlockHash`.

5.  **Pallet Implementation**:

- It implements various functions to interact with the pallet's storage items and perform specific actions.
- Functions include setters and getters for storage items, functions to manage gas payments, sidevm code operations, system contract settings, event block management, and more.
- The implementation includes error handling and ensures proper balance transfers using the `Currency` trait.

6.  **Traits Implementation**:

- It implements the `AddressGenerator` trait for the pallet, which provides a method to generate contract addresses based on specific parameters.

7.  **Conversion Implementation**:

- It implements the `Convert` trait for converting between `Weight` and `BalanceOf` types used in gas calculations.

## <ins>pink/runtime/src/capi/mod.rs</ins>

this contract initializes the runtime environment, sets up OCall functions, handles ECalls, and manages logging. It's designed for use in an enclave-based execution environment.

1.  **Entry Point**:

- The function `__pink_runtime_init` serves as the entry point for the runtime.
- It takes a configuration pointer (`config`) and an ECalls pointer (`ecalls`) as arguments.
- Within this function, the OCall function is set based on the provided configuration.
- It initializes the ECalls function pointers, logging errors if initialization fails.
- If the runtime is configured as a dynamic library (`is_dylib != 0`), it initializes the logger.

2.  **`ecall` Function**:

- This function serves as a central hub for all ECalls (enclave calls) in the runtime.
- It takes parameters related to the ECalls function, including `call_id`, `data`, `len`, `ctx`, and `output_fn`.
- It dispatches the ECalls using the `ecall::dispatch` function, passing input data and retrieving output data.
- If an `output_fn` is provided, it calls the function to handle the output data.

3.  **Submodules**:

- The module `ecall_impl` likely contains the implementation details for ECalls handling.
- The module `ocall_impl` likely contains the implementation details for OCalls handling.

## <ins>pink/runtime/src/capi/ecall\_impl.rs</ins>

this contract provides a comprehensive implementation for handling ECalls in the runtime environment, including setting up the cluster, managing accounts and balances, uploading and calling contracts, and handling various runtime events.

1.  **Implementation of `Executing` trait for `ExternalStorage`**:

- Two methods `execute` and `execute_mut` are implemented for `ExternalStorage`.
- These methods execute the provided closure within the context of an execution environment and handle side effects.

2.  **Implementation of `ECalls` trait for `ECallImpl`**:

- Several methods are implemented to handle different aspects of ECalls:
- `cluster_id`: Retrieves the cluster ID from the `PalletPink`.
- `setup`: Sets up the cluster based on the provided configuration.
- `deposit`: Deposits a certain amount of balance to an account.
- `set_key` and `get_key`: Set and retrieve the secret key respectively.
- `upload_code` and `upload_sidevm_code`: Uploads wasm code or sidevm code respectively.
- `system_contract`: Retrieves the system contract.
- `free_balance` and `total_balance`: Retrieves the free and total balance of an account respectively.
- `contract_instantiate` and `contract_call`: Instantiates or calls a contract respectively.
- `git_revision`: Retrieves the Git revision.
- `on_genesis`, `on_runtime_upgrade`, and `on_idle`: Handles events related to genesis, runtime upgrade, and idle respectively.

3.  **Utility Functions**:

- `sanitize_args`: Adjusts gas limit based on the execution mode.
- `handle_deposit`: Handles deposits to an account.

## <ins>pink/runtime/src/capi/ocall\_impl.rs</ins>

this contract provides a flexible interface for setting and handling OCall functions in an enclave environment, with optional support for a custom memory allocator.

1.  **Static Mutable Variable and Default OCall Function**:

- It declares a static mutable variable `OCALL` of type `InnerType<cross_call_fn_t>`, which represents the OCall function pointer.
- The default OCall function `_default_ocall` is defined, which simply panics with a message indicating that no OCall function is provided.

2.  **`set_ocall_fn` Function**:

- This function sets the OCall function pointer to the provided function.
- It also optionally sets the allocator functions if the `allocator` feature is enabled.
- The OCall function pointer is assigned to the `OCALL` static mutable variable.
- If the `allocator` feature is enabled, it assigns the allocator and deallocator functions to the respective static mutable variables.

3.  **`OCallImpl` Struct and Implementations**:

- The `OCallImpl` struct is defined, representing the implementation for OCall functions.
- Implementations are provided for `CrossCallMut`, `CrossCall`, and `OCall` traits.
- The `cross_call_mut` and `cross_call` methods are implemented to handle cross-call operations. They use the OCall function pointer stored in the `OCALL` variable to make external function calls.
- The `cross_call` method sets up an output function to capture the output data from the OCall.

4.  **Allocator Module (Conditional Compilation)**:

- This module provides an implementation for a custom memory allocator when the `allocator` feature is enabled.
- It defines static mutable variables for allocator and deallocator functions (`ALLOC_FUNC` and `DEALLOC_FUNC`).
- The `PinkAllocator` struct represents the custom allocator, implementing the `GlobalAlloc` trait.
- The `alloc` and `dealloc` methods of `PinkAllocator` use the allocator and deallocator functions respectively.
- The `system_alloc` and `system_dealloc` functions are defined as wrappers around the system allocator functions.
- A test function `can_set_ocalls` is provided to test setting OCall functions and allocation using the custom allocator.

## <ins>pink/runtime/src/runtime/extension.rs</ins>

this contractdefines a comprehensive chain extension mechanism for interacting with Pink contracts within the runtime environment. It handles various types of calls and provides backend implementations for different functionalities required by Pink contracts.

1.  **Chain Extension Implementation**:

- The `call` function in the `PinkExtension` struct handles incoming calls to the chain extension. It dispatches calls based on the function ID and mode (query or command), then processes the inputs and outputs accordingly.
- The `enabled` function determines whether the chain extension is enabled.

2.  **Chain Extension Backend Implementation**:

- The implementations of the `PinkExtBackend` trait provide the backend logic for handling specific functions within the chain extension. These functions include HTTP requests, signing and verification, cache operations, logging, and more.
- Depending on the context (query or command), different implementations are used to handle these functions.

3.  **Utility Functions**:

- Utility functions like `deposit_pink_event` and `get_side_effects` are used for depositing events and retrieving side effects respectively.

## <ins>pink/capi/src/v1/mod.rs</ins>

The contract is structured to define interfaces and traits necessary for interacting with the Pink runtime environment, both for making calls within contracts (`ecall`) and for interacting with external systems (`ocall`). These traits are implemented by the Pink runtime itself to provide the required functionality for smart contract execution.

### Traits:

1.  **`CrossCall` and `CrossCallMut`**:

- These traits define methods for making cross-calls between different parts of the system, either immutably or mutably.

2.  **`Executing`**:

- This trait is used for context setup before invoking the runtime implementations of ecall functions. It provides methods for executing functions either immutably or mutably.

3.  **`ECall` and `OCall`**:

- These traits are markers for types that are used in cross-call interfaces. They don't define any methods themselves.

### Modules:

1.  **`ecall`**:

- This module contains traits and associated types related to external calls (`ecall` functions) made by contracts.
- It defines the `ECalls` trait, which is implemented by the Pink runtime to provide functions like contract setup, depositing balance, uploading code, etc.
- Additionally, it contains test cases for coverage.

2.  **`ocall`**:

- This module contains traits and associated types related to outgoing calls (`ocall` functions) made by contracts.
- It defines the `OCalls` trait, which is implemented by the Pink runtime to provide functions like fetching storage, sending logs, performing HTTP requests, etc.
- It also includes types for HTTP requests and responses, as well as a test case for coverage.

### Test Functions:

- The `coverage` functions within each module are test cases meant to ensure that the defined types and traits are covered properly.

## <ins>pink/capi/src/types.rs</ins>

This contracts provides essential types and enums for managing contract execution modes and handling side effects emitted by contracts during execution in the Pink runtime. It ensures proper encapsulation and provides utility methods for working with these types and enums.

### Types:

1.  **`Hash`**: Alias for the output type of the Blake2-256 hasher.
2.  **`Hashing`**: Type alias for Blake2-256 hasher.
3.  **`AccountId`**: Type alias for a 32-byte account identifier.
4.  **`Balance`**: Type alias for an unsigned 128-bit integer representing balance.
5.  **`BlockNumber`**: Type alias for an unsigned 32-bit integer representing block number.
6.  **`Index`**: Type alias for an unsigned 64-bit integer.
7.  **`Address`**: Type alias for an `AccountId32`, representing an address.
8.  **`Weight`**: Type alias for an unsigned 64-bit integer representing weight.

### Enums:

1.  **`ExecutionMode`**:

- Enum representing the mode in which the runtime is executing: Query, Estimating, or Transaction.
- It provides methods to check the current mode and properties of each mode, like whether it's querying, estimating, or requires deterministic execution.

2.  **`ExecSideEffects`**:

- Enum representing the side effects emitted by contracts during execution.
- It contains various types of events emitted by contracts, such as Pink events, Ink events, and instantiated contracts.
- Provides methods to filter side effects and check if the side effects are empty.

### Traits:

- The `Decode` and `Encode` traits are imported from the `scale` crate, which are used for serialization and deserialization of data structures.

## <ins>pink/chain-extension/src/lib.rs</ins>

This module provides a comprehensive set of utilities and default implementations for interacting with external systems within the Pink runtime environment, ensuring compatibility and error handling for different scenarios. It abstracts away the complexity of making HTTP requests, caching data, logging, and other common tasks, allowing developers to focus on writing contracts without worrying about low-level system interactions.

### External Dependencies:

- **reqwest**: Used for making HTTP requests.
- **sp\_core**: Provides cryptographic primitives like key pairs and signatures.
- **pink**: Contains types and traits specific to the Pink runtime environment.

### Types and Traits:

1.  **`PinkRuntimeEnv` trait**: Defines an interface for accessing runtime environment details like the account identifier.
2.  **`DefaultPinkExtension` struct**: Implements the `PinkExtBackend` trait, providing default implementations for various extension functions like HTTP requests, caching, logging, etc.
3.  **`LimitedWriter` struct**: A custom writer that limits the number of bytes written to it.

### Functions:

1.  **`block_on`**: Utility function to run asynchronous futures synchronously.
2.  **`batch_http_request`**: Sends multiple HTTP requests concurrently, enforcing a maximum limit on concurrent requests.
3.  **`http_request`**: Sends a single HTTP request, handling errors and timeouts.
4.  **`async_http_request`**: Sends an HTTP request asynchronously, handling errors and timeouts.
5.  Various methods implementing the `PinkExtBackend` trait for handling different kinds of interactions such as HTTP requests, signing, caching, logging, etc.

### Tests:

- Tests covering various scenarios like default implementation correctness, batch HTTP request handling, and HTTP request error handling.

## <ins>pink/chain-extension/src/local\_cache.rs</ins>

This code implements a local key-value cache (`LocalCache`) designed for use in contracts to perform off-chain computation.

1.  **Purpose**: The `LocalCache` is intended to provide a local key-value cache for contracts. It stores data locally, which means that the cached data can differ across machines running the same contract. Additionally, data in the cache might be lost due to the cache expiring or when the pruntime (presumably a parallel runtime) restarts.
    
2.  **Components**:
    

- `Storage`: Represents the storage for a specific key-value pair. It includes functionality for managing the size of stored items, garbage collection, setting values with expiration times, and removing items.
- `LocalCache`: Manages multiple `Storage` instances, applies quotas, sets and retrieves values, and handles cache operations like setting expiration and removal.
- `now()`: Provides the current time in seconds since the program started running.
- Various helper functions (`apply_cache_op`, `set`, `get`, `set_expiration`, `remove`, `apply_quotas`) for interacting with the cache.

3.  **Concurrency**: The code handles concurrency by using thread-local storage (`thread_local`) for unit tests and a global mutex (`Mutex`) otherwise.
    
4.  **Testing**: The code includes a comprehensive set of tests (`mod test`) that cover different aspects of the cache functionality, including default expiration, setting expiration, size limits, garbage collection, and cache operations.
    
5.  **Dependencies**:
    

- `once_cell`: Used for lazy initialization of the reference time.
- `pink`: Imports `CacheOp` and `StorageQuotaExceeded` types. It's not clear from the provided code what `pink` is, but it's assumed to be an external crate providing these types.

6.  **Error Handling**: Errors are handled using `Result` and `Option` types. For example, `Result<(), StorageQuotaExceeded>` is returned when setting a value, indicating whether the storage quota was exceeded.
    
7.  **Performance**: The code implements garbage collection to manage the cache size and expiration. Garbage collection is triggered based on a specified interval of set operations (`gc_interval`). Additionally, it removes expired items to free up space.
    

## 7\. Test analysis

The audit scope of the contracts to be reviewed is 90%, with the aim of reaching 100% to increase the safety.

## Codebase Quality

Overall, I consider the quality of the Phala codebase to be Good. The code appears to be mature and well-developed. I have noticed the implementation of various standards adhere to appropriately. Details are explained below:

| Codebase Quality Categories | Comments |
| --- | --- |
| **Code Maintainability and Reliability** | The Phala ptotocol demonstrates good maintainability through modular structure, consistent naming. It also prioritizes reliability by handling errors, conducting security checks. However, for enhanced reliability, consider professional security audits and documentation improvements. Regular updates are crucial in the evolving space. |
| **Code Comments** | The code is well commented with NatSpec comments, which are clear and informative. |
| **Documentation** | The documentation provided for the [Phala protocol](https://docs.phala.network/) is quite comprehensive and detailed in terms of explaining its functionality, parameter usage, purpose, and overall architecture. |
| **Code Structure and Formatting** | The codebase contracts are well-structured and formatted, which are clear and informative. This could be improved by adding supplementary external documentation with diagrams outlining the workings of the core functionality. |
| **Imports** | In Phala protocol, the contract's interface should be imported first, followed by each of the interfaces it uses, followed by all other files. |

## Conclusion

In general, the Phala project exhibits an interesting and well-developed architecture we believe the team has done a good job regarding the code, but the identified risks need to be addressed, and measures should be implemented to protect the protocol from potential malicious use cases. Additionally, it is recommended to improve the documentation and comments in the code to enhance understanding and collaboration among developers. It is also highly recommended that the team continues to invest in security measures such as mitigation reviews, audits, and bug bounty programs to maintain the security and reliability of the project.

## Time Spend

32 hours

### Time spent:
32 hours