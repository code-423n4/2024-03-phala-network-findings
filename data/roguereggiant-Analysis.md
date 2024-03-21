
| File Name               | File Description                                        |
|-------------------------|---------------------------------------------------------|
| runtime.rs              |  This code is a configuration for a blockchain runtime using Substrate, a framework for building blockchains. It defines modules (pallets) such as system, timestamp, balances, and contracts, sets up runtime parameters, and includes tests for checking metadata and executing runtime upgrades.                                   |
| contract.rs             | This code snippet is part of a blockchain module focused on contract interactions within a Substrate-based runtime, offering functionalities like instantiation and calling of contracts, including gas and storage deposit calculations. It features utility functions for handling results and adjusting gas and deposit values based on specified conditions.                                    |
| mod.rs                  | This code facilitates execution of runtime code segments with storage backends in a Substrate-based blockchain environment, enabling execution within specified contexts, managing transactions, and committing changes to storage. It also includes functionality to emit system events in a privacy-conscious manner, ensuring external accessibility without exposing sensitive RPCs.                                     |
| external_backend.rs     | This code introduces a database abstraction (`ExternalDB`) that complies with the `TrieBackendStorage` interface for Substrate's `TrieBackend`, specifically designed for environments where storage operations are delegated externally via ocalls. It facilitates reading and writing to a key-value store outside the immediate blockchain execution environment, particularly useful for systems that interact with external data sources or storage mechanisms.                                     |
| pallet_pink.rs          | This Substrate pallet provides functionalities for managing side virtual machines (SideVMs), including storage of WASM code, handling gas pricing, deposits for storage use, and managing a treasury account. It uses the FRAME support library to define storage items, errors, and configuration traits essential for operation within a blockchain's runtime module.                                     |
| mod.rs                  | This codebase is part of a blockchain runtime module that initializes the runtime environment and facilitates communication between the host and the runtime through a set of predefined external calls (ecalls) and output functions. It establishes the runtime's entry point, error handling for initialization, version reporting, and the dispatch mechanism for routing function calls based on identifiers, ensuring safe interaction with raw pointers and external data.                                     |
| ecall_impl.rs           | This codebase integrates a blockchain runtime with external functions, allowing for the setup and interaction with a contract execution environment. It includes mechanisms for initializing the runtime, deploying contracts, handling transactions, and managing storage through external calls (ocalls) and internal executions (ecalls), facilitating a bridge between the runtime logic and external inputs or configurations.                                    |
| ocall_impl.rs           | This codebase establishes a framework for handling out-of-contract calls (ocalls) in a blockchain context, defaulting to a panic if no ocall function is provided. It sets up a mechanism for the runtime environment to safely call external functions, potentially involving memory allocation and deallocation. This setup allows for dynamic interaction between the blockchain runtime and its environment, including features for memory management when the "allocator" feature is enabled.                                     |
| extension.rs            | This code integrates advanced functionalities into a blockchain runtime, enabling it to interact with external services and manage internal operations such as HTTP requests, cryptographic operations, and storage management. It also supports the deployment and execution of smart contracts, including handling of events and extending the runtime capabilities through a chain extension mechanism, allowing for custom operations and interactions within the blockchain ecosystem.                                     |
| mod.rs                  | This codebase sets up a comprehensive framework for cross-call capabilities within a blockchain runtime, allowing both immutable and mutable function executions. It defines traits and implementations for executing external calls (ecalls and ocalls), handling transactions, managing system and contract states, as well as facilitating HTTP requests, cryptographic operations, and storage interactions. Additionally, it includes mechanisms for logging, caching, and querying contract-related information, alongside runtime and cluster setup configurations.                                     |
| lib.rs                  | This codebase facilitates HTTP requests and blockchain chain extensions, integrating with external web services and performing cryptographic operations. It provides asynchronous support for handling HTTP requests, including batch requests, with timeout handling and error management, alongside utility functions for caching, logging, and generating random data within a blockchain runtime environment.                                     |
| local_cache.rs          | This codebase implements a local key-value (KV) cache system specifically designed for blockchain contracts, facilitating offchain computation and data storage with the caveat that data may vary across different instances and could be lost due to restarts or expiration mechanisms. It supports testing modes, size limits, and expiration for cache entries to manage resources efficiently within a decentralized environment.                                     |



### Architecture Overview

The system consists of a blockchain runtime environment that extends its functionality through the implementation of external calls (ecalls and ocalls), chain extensions, and a local caching mechanism. The runtime environment interacts with external web services, supports cryptographic operations, and performs off-chain computations. Key components include:

- **Runtime Environment**: Manages the blockchain's state, executes smart contracts, and handles transactions.
- **Chain Extension**: Extends the runtime's capabilities, allowing contracts to interact with external services and perform additional computations.
- **HTTP Requests**: Facilitates communication with external web services, including both individual and batch requests.
- **Caching Mechanism**: Provides a local cache for off-chain data storage and computation, with mechanisms for data expiration and storage quota management.
- **Cryptographic Operations**: Supports various cryptographic functions, including signing, verification, and key derivation.
- **External Calls (ecalls/ocalls)**: Allows the runtime to interact with the external environment and the host system, including logging, storage operations, and accessing execution context.

### Architecture Diagram

[![arc-drawio.png](https://i.postimg.cc/5NMht7y8/arc-drawio.png)](https://postimg.cc/9zJ88BGf) 

This architecture supports a modular and extendable blockchain runtime environment, allowing for robust interactions within the blockchain ecosystem and with external systems. It enables the runtime to perform a wide range of operations, from smart contract execution and transaction management to off-chain computation and interaction with web services, all while maintaining the security and integrity of the blockchain.

### Overall Sequence Diagram

This sequence diagram outlines the flow of operations starting from a smart contract making a request to external services, performing cryptographic operations, and utilizing local caching.

[![seq-drawio-2.png](https://i.postimg.cc/7655fPBt/seq-drawio-2.png)](https://postimg.cc/62NWPKxC) 

This diagram starts with the smart contract invoking the chain extension to perform an operation, such as an HTTP request to external web services. The response from the web service is cached locally via the local cache component. The smart contract can also request cryptographic operations like signing, which are performed by the crypto operations component. Additionally, the smart contract interacts directly with the local cache for setting or retrieving data. Finally, transactions, potentially including operations performed and data fetched or modified, are submitted to the blockchain node to be included in the blockchain.

### Runtime.rs
[![f1-uml-drawio-3.png](https://i.postimg.cc/4ygXS1zz/f1-uml-drawio-3.png)](https://postimg.cc/DJxVWLpZ) 
### Overview of Smart Contract Functionalities

This smart contract setup involves various configurations and functionalities that enhance the blockchain's ability to process transactions, interact with external data, and perform specific operations efficiently. Below are the key functionalities described:

#### Runtime Configuration

- **Construct Runtime**: Establishes the `PinkRuntime` structure incorporating several pallets such as System, Timestamp, Balances, Randomness, Contracts, and Pink, facilitating comprehensive blockchain operations.
- **Parameter Types**: Defines constants and parameters like `BlockHashCount`, `RuntimeBlockWeights`, and `ExistentialDeposit`, essential for blockchain operation specifics such as gas calculation and account management.
- **Pallet Configurations**: Each pallet is configured with specific parameters and types, tailored to the runtime's needs. For instance, the `pallet_balances` is configured with the currency type, account data structure, and existential deposit value.

#### Chain Extensions and Migrations

- **Chain Extension**: Integrates custom logic via the `PinkExtension`, allowing smart contracts to interact beyond the predefined pallet functionalities, like making HTTP requests or accessing custom cryptographic operations.
- **Migration Logic**: Implements a series of migrations (v11 to v15 and a NoopMigration for v10) to ensure the runtime adapts to new data structures or logic changes without compromising data integrity or functionality.

#### Utility Functions

- **Genesis and Runtime Upgrade Hooks**: Functions like `on_genesis` and `on_runtime_upgrade` are essential for initializing pallets upon blockchain genesis and applying necessary migrations or updates during runtime upgrades.
- **Idle Functionality**: The `on_idle` function is called when the blockchain node is not processing transactions, allowing for cleanup tasks or resource optimization processes to run.

#### Metadata and Testing

- **Metadata Management**: Functions to manage and validate runtime metadata, ensuring that the contract's interface remains consistent and up-to-date with the underlying blockchain structure.
- **Testing and Coverage**: Includes utilities for testing the runtime's functionality, ensuring that changes to parameters or logic do not break existing contract interactions or blockchain stability.

Overall, this smart contract setup provides a robust framework for blockchain operation, enabling enhanced functionality through extensions, efficient management of resources through parameter configurations, and ensuring the system's integrity through migrations and metadata management.
[![f1-seq-drawio-2.png](https://i.postimg.cc/PJTC0DZ8/f1-seq-drawio-2.png)](https://postimg.cc/CBrMqzcF) 
### Contract.rs
[![f2-uml-drawio-3.png](https://i.postimg.cc/ncYq0HyW/f2-uml-drawio-3.png)](https://postimg.cc/GHtBp1hP) 
### Smart Contract Functions Overview

This smart contract framework offers a suite of functionalities centered around contract execution, instantiation, and result handling within a blockchain runtime environment. Here's a breakdown of each function and its role in the contract lifecycle.

#### Contract Execution and Instantiation

- **`instantiate`**: This function creates a new instance of a contract on the blockchain using a specified code hash, input data, and initialization arguments. It configures the contract's execution environment, including gas limits and storage deposits, and logs the outcome.

- **`bare_call`**: Executes a method on an already instantiated contract, identified by the `address` parameter. It sends encoded input data and transaction arguments, managing gas and storage requirements for the call.

#### Result Handling

- **`ContractExecResult` and `ContractInstantiateResult`**: These types represent the outcomes of contract execution and instantiation, respectively, encapsulating details such as the balance changes, emitted events, and execution errors if any.

- **`ContractResult<T>`**: A generic result type that wraps the outcome of contract calls, including potential runtime errors and associated balance and event information.

#### Gas and Storage Management

- **`mask_deposit` and `mask_gas`**: Functions that adjust the deposit and gas values to conform with certain constraints, such as minimum masked bytes or gas usage, enhancing transaction safety and predictability.

- **`coarse_grained`**: Modifies a contract's execution result to round gas consumption and storage deposits to coarse-grained values, simplifying the contract's interaction with blockchain resources.

#### Utility Functions

- **`check_instantiate_result`**: Checks the result of a contract instantiation, verifying whether the operation was successful or reverted, to ensure the correct handling of contract deployment outcomes.

- **`contract_tx`**: A lower-level function that executes a transaction with the specified parameters, handling gas payment and refunds. It acts as a transaction wrapper, facilitating contract calls and instantiation within the blockchain's execution environment.

#### Macro Definitions

- **`define_mask_fn`**: A macro that defines functions for masking numeric values. It's used to create `mask_low_bits64` and `mask_low_bits128`, which mask the lower bits of 64-bit and 128-bit numbers, respectively, according to predefined rules. These masking functions play a critical role in adjusting numeric values for gas and deposit calculations.

Each function and macro within this framework plays a pivotal role in facilitating the secure, efficient, and predictable execution of smart contracts on the blockchain, providing foundational mechanisms for contract interaction, resource management, and execution outcome processing.
[![f2-seq-drawio-1.png](https://i.postimg.cc/JzF0Qvzg/f2-seq-drawio-1.png)](https://postimg.cc/CRjwhvDH) 
### Mod.rs
[![f3-uml-drawio-2.png](https://i.postimg.cc/q75jGkPK/f3-uml-drawio-2.png)](https://postimg.cc/wtDXqdTx) 
### Smart Contract Storage and Execution Framework Overview

This smart contract framework encompasses mechanisms for executing contract code, managing storage, and emitting events. It integrates with the blockchain's state machine and utilizes external backend storage systems. Here's a detailed look at its components and functionalities.

#### Storage Management

- **Storage Structure**: Holds a reference to the underlying storage backend, facilitating interactions with blockchain storage.

- **`new` Function**: Initializes a new instance of the storage structure with a specified backend.

#### Execution Environment

- **`execute_with` Function**: Executes a given closure within a blockchain runtime environment, leveraging the current storage backend. It captures the closure's return value, execution side effects, and any changes made to the storage, packaging them into a cohesive result.

- **`execute_mut` Function**: Similar to `execute_with`, but with the ability to commit changes to the storage backend. It's useful for operations that modify the blockchain's state.

#### Transaction and Change Management

- **`changes_transaction` Function**: Converts overlay changes recorded during execution into a transaction format, preparing them for commitment to the storage backend.

- **`commit_changes` Function**: Commits storage changes to the backend, effectively updating the blockchain state based on the results of contract execution.

- **`get` Function**: Retrieves the value associated with a specific key from the storage, enabling contracts to access stored data.

#### Event Emission

- **`maybe_emit_system_event_block` Function**: Handles the emission of blockchain events generated by contract execution. It packages events into blocks, maintaining their sequence and integrity, and utilizes external calls (ocalls) to emit them outside the runtime environment. This function ensures that contract-generated events are accessible externally, supporting contract transparency and traceability without compromising on privacy.

#### External and In-Memory Backends

- **External and In-Memory Storage Backends**: The framework includes provisions for both external storage backends and in-memory storage solutions, offering flexibility in how contract data is stored and accessed. These backends are integral to the framework's ability to adapt to different runtime environments and storage requirements.

#### Commit Transaction Trait

- **`CommitTransaction` Trait**: Defines the essential function required for committing transactions to the storage backend. Implementations of this trait must provide mechanisms for applying changes to the blockchain state, as dictated by the outcomes of contract execution.

This framework provides a robust foundation for executing smart contracts within a blockchain environment, handling storage interactions, and managing the lifecycle of contract-generated events. Its components work together to ensure that contracts can be executed efficiently, their effects can be reliably committed to the blockchain, and the events they generate can be made accessible for external consumption.
[![f3-seq-drawio-2.png](https://i.postimg.cc/fL7SRCfb/f3-seq-drawio-2.png)](https://postimg.cc/YG9CX1Vc) 
### External_backend.rs 
[![f4-uml-drawio-2.png](https://i.postimg.cc/C5J1bw7q/f4-uml-drawio-2.png)](https://postimg.cc/D4JnTktv) 
### Overview of the External Storage Framework for Smart Contracts

This smart contract framework introduces a mechanism for interfacing with external storage systems, emphasizing the separation of storage management from the contract execution environment. It facilitates direct communication with the host's storage system through external calls, enabling smart contracts to interact with storage without directly managing it.

#### External Database Interface

- **`ExternalDB` Structure**: Acts as an intermediary that does not manage any key-value backend directly. Instead, it delegates storage operations to the host system through external calls (ocalls), allowing smart contracts to read from and write to external storage systems efficiently.

#### Storage Integration

- **`ExternalBackend` Type**: Defines a specialized `TrieBackend` that utilizes `ExternalDB` for storage operations. This setup allows the use of trie data structures for organizing contract storage, leveraging external systems for actual data persistence.

- **`ExternalStorage` Type**: Encapsulates the `ExternalBackend`, providing a higher-level interface for smart contracts to interact with storage. It simplifies the process of storing and retrieving contract data.

#### Storage Operations

- **TrieBackendStorage Implementation**: Implements the `TrieBackendStorage` trait for `ExternalDB`, enabling basic storage operations such as fetching a value for a given key. This implementation directly utilizes external calls to interact with the host's storage system.

- **Commit Transaction Functionality**: Through the `CommitTransaction` trait implementation for `ExternalBackend`, the framework provides functionality for committing transactions to the external storage. It processes and commits the changes collected during contract execution, ensuring that the state modifications are reflected in the external storage system.

#### Instantiation and Code Existence Verification

- **Storage Instantiation**: Facilitates the creation of an `ExternalStorage` instance, initializing it with a storage root obtained from the host system. This function sets up the necessary environment for storage interactions within the smart contract execution context.

- **Code Existence Verification**: Utilizes helper functions to verify the existence of specific contract code within the storage. It constructs the appropriate storage key for the code and checks if it exists, enabling smart contracts to query the presence of other contracts or code segments in the system.

### Helper Module

The helper module provides utility functions to support common storage-related operations, such as verifying the existence of contract code. It illustrates how to construct storage keys for specific queries, leveraging the external storage system to check for the presence of code. This approach simplifies the process of interacting with complex storage structures and enhances the smart contract's capabilities to make informed decisions based on the stored data.

Overall, this framework extends the smart contract's ability to interact with external storage systems efficiently, providing a flexible and powerful interface for managing storage operations outside the immediate blockchain environment. It enables the seamless integration of smart contracts with external storage solutions, paving the way for more complex and dynamic contract behaviors.
[![f4-seq-drawio-2.png](https://i.postimg.cc/Hxd8MvmF/f4-seq-drawio-2.png)](https://postimg.cc/rdZpXJK9) 
### Pallet_pink.rs
[![f5-uml-drawio-2.png](https://i.postimg.cc/vHJB92LX/f5-uml-drawio-2.png)](https://postimg.cc/nC1pfkWQ) 
### Pallet Overview

This smart contract introduces a flexible framework designed to facilitate various operations related to contract execution, storage management, and system configuration within a blockchain environment. It leverages Rust's `frame_support` and `pallet_contracts` for extended functionality.

### Configurable Parameters and Storage

- **Config Trait**: Defines the configuration interface for the pallet, including the type of currency used for transactions.

- **Storage Items**: Various storage items, such as `ClusterId`, `GasPrice`, `DepositPerByte`, `DepositPerItem`, `TreasuryAccount`, `Key`, `SidevmCodes`, `SystemContract`, `NextEventBlockNumber`, and `LastEventBlockHash`, are declared to store configurations and operational data essential for contract execution and management.

### Error Handling

- **Error Enum**: Enumerates potential errors that might occur during pallet operations, such as unknown chain extension IDs or functions, buffer overflows, missing key seeds, derivation failures, and missing system contracts.

### Data Structures

- **WasmCode Struct**: Represents WebAssembly code associated with an account, storing both the owner's account ID and the bytecode.

### Essential Functions and Traits

- **AddressGenerator Implementation**: Customizes the mechanism for generating contract addresses based on deploying addresses, code hashes, and additional parameters.

- **Pallet Functionality**: Provides a comprehensive set of functions to manipulate the storage items and conduct system operations, including setting cluster IDs, managing keys, handling sidevm codes, adjusting gas prices, managing deposits, and updating treasury account information.

- **Payment and Refund Mechanisms**: Implements functions to handle payment for gas usage and refunds, ensuring a flexible management of contract execution costs.

- **Utility Functions**: Includes various utility functions, such as `convert`, to facilitate operations like converting weights to balances according to the current gas price.

### Contract Execution and Management

This framework is tailored for building and managing smart contracts on the blockchain, providing developers with tools to deploy, execute, and interact with contracts efficiently. It emphasizes on ease of use, security, and flexibility, offering a robust foundation for developing decentralized applications.
[![f5-seq-drawio-2.png](https://i.postimg.cc/Gt3MXjF5/f5-seq-drawio-2.png)](https://postimg.cc/8J3B5MnR)
### Mod.rs 
[![f6-uml-drawio-3.png](https://i.postimg.cc/661SMDPV/f6-uml-drawio-3.png)](https://postimg.cc/bdk3JFpd)
### Initialization and Runtime Configuration

- **Runtime Initialization (`__pink_runtime_init`)**: Acts as the entry point to initialize the runtime, setting up o-call functions and filling the e-call table. It ensures the provided pointers (`config` and `ecalls`) are valid and initializes logging if specified.

- **Version Getter (`get_version`)**: Extracts and returns the major and minor version numbers of the runtime, helping to identify the runtime version externally.

### e-call Dispatcher

- **e-call Dispatcher (`ecall`)**: Serves as the central function to dispatch e-calls based on the provided call ID. It decodes input data for the specified call, executes the corresponding function in the `ECallImpl` structure, and uses the output function pointer to return the result.

### Module Organization

- **Module Definitions**: Separates implementation details into modules, specifically `ecall_impl` for handling e-call implementations and `ocall_impl` for managing o-call functions. This organization helps maintain clear separation between different parts of the runtime logic.

### Safety and Error Handling

- **Safety Considerations**: Both the initialization function and e-call dispatcher are marked as `unsafe` due to direct pointer manipulations and requirements for external contracts to ensure the validity of provided pointers.

- **Error Reporting**: Utilizes logging to report initialization failures or issues with setting up o-call functions, aiding in troubleshooting potential problems during runtime setup.

### Interaction Flow

- **Function Dispatch Mechanism**: Implements a mechanism to dynamically route calls to their respective handlers based on call IDs, streamlining the process of extending runtime functionalities with new e-calls.

- **Version Reporting**: Facilitates external inquiries about the runtime version, allowing for compatibility checks and ensuring that external components can verify they are interacting with the correct version of the runtime.
[![f6-seq-drawio-2.png](https://i.postimg.cc/Zn7CNTYM/f6-seq-drawio-2.png)](https://postimg.cc/jnfx0Y7H) 
### Ecall_impl.rs
[![f7-uml-drawio-1.png](https://i.postimg.cc/DwJKQ1Pg/f7-uml-drawio-1.png)](https://postimg.cc/LJp72Y7Y) 
### ECall Implementation Overview

The `ECallImpl` struct and related functions provide the implementation for executing specific operations, interacting with the contract system, and managing the storage. This implementation spans across initializing clusters, handling code uploads, managing account balances, and interacting with contracts directly.

### Cluster Setup and Initialization

- **Cluster Setup (`setup`)**: Initializes a cluster with specific configurations including the cluster ID, owner, deposit requirements, and the system contract. It sets up various parameters for the runtime environment and deploys the system contract.

- **Genesis and Runtime Upgrade Handlers (`on_genesis`, `on_runtime_upgrade`)**: Functions that are called during the genesis block creation and runtime upgrades, ensuring that the pallets and the runtime environment are correctly initialized or upgraded.

### Account and Balance Management

- **Deposit Handling (`deposit`)**: Allows depositing a specified balance into an account, facilitating balance transfers within the contract environment.

- **Key Management (`set_key`, `get_key`)**: Manages the secret key for the cluster, providing functionality to set and retrieve the key as needed.

### Contract Interaction

- **Contract Instantiation and Calling (`instantiate`, `contract_call`)**: Facilitates the instantiation of contracts from given code hashes and the execution of contract calls, providing the mechanism to interact with deployed contracts directly.

- **Utility Functions (`sanitize_args`, `handle_deposit`)**: Helpers used to adjust transaction arguments based on the execution mode and handle deposits for transactions, ensuring correct gas limit and deposit handling.

### Code and Storage Management

- **Code Upload (`upload_code`, `upload_sidevm_code`)**: Supports uploading contract code to the storage, allowing the deployment of contracts and side VMs. It includes checks for determinism and code size.

- **Storage and Effect Tracking (`execute`, `execute_mut`)**: Executes given functions with or without mutating the state, respectively, capturing the execution side effects and managing the storage overlay.

- **Utility and Information Retrieval (`git_revision`, `system_contract`, `free_balance`, `total_balance`, `code_hash`)**: Provides various utilities and accessors for retrieving system information, including the current git revision, contract balances, and code hashes.

### Environment and Execution Context

- **Execution Context Instrumentation (`instrument_context`)**: A macro to set up the execution context, including request ID and block number, for tracing and logging purposes.

These functionalities collectively enable a wide range of operations within the contract ecosystem, from basic account and balance management to complex contract deployment and interaction mechanisms.
[![f7-seq-drawio-1.png](https://i.postimg.cc/4xT3bBtW/f7-seq-drawio-1.png)](https://postimg.cc/VdD1z9Kt) 
### Ocall_impl.rs
[![f8-uml-drawio-3.png](https://i.postimg.cc/gJkdk8KY/f8-uml-drawio-3.png)](https://postimg.cc/KKVC0KVV) 
### Cross-Call Mechanism and Memory Allocation

This component implements the infrastructure for cross-calling and memory management within a dynamic runtime environment, particularly focusing on external call (ocall) functionalities and custom memory allocation strategies.

### OCall Functionality

- **OCall Initialization**: Establishes the mechanism for performing ocalls by setting up a default panic-triggering function for scenarios where no ocall function is provided, ensuring runtime safety.

- **Set OCall Function (`set_ocall_fn`)**: Configures the actual function to handle ocalls, replacing the default one. It integrates optional allocation and deallocation functions for custom memory management, enabling flexibility in resource handling.

### Cross-Call Implementations

- **OCall Implementation (`OCallImpl`)**: Provides the implementation for the `CrossCall` and `CrossCallMut` traits, facilitating the execution of cross-calls. It leverages an output function to collect execution results into a vector, ensuring data integrity and retrievability.

### Custom Allocator

- **Allocator Setup**: Introduces a custom allocator by designating specific functions for memory allocation and deallocation, aiming to use a shared allocator across the runtime and the main executable for consistent memory metrics and statistics.

- **Global Allocator (`PinkAllocator`)**: Defines a global allocator for the runtime, utilizing the specified allocation and deallocation functions. This custom allocator ensures that memory management conforms to the runtime's requirements, enhancing performance and monitoring capabilities.

### Safety and Testing

- **Safety Measures**: Incorporates safety checks, particularly in the context of memory layout validations, to prevent undefined behaviors during dynamic memory operations.

- **Testing Framework**: Includes test cases to validate the functionality of setting ocall functions and the behavior of the default ocall function, ensuring the reliability and robustness of the ocall mechanism and custom allocator.

This component essentially enhances the runtime's interoperability with the host environment through ocalls, alongside optimizing memory management via a tailored allocator. It ensures seamless external interactions and efficient resource utilization, crucial for dynamic runtime operations and performance optimization.
[![f8-seq-drawio-2.png](https://i.postimg.cc/fy3qTRt2/f8-seq-drawio-2.png)](https://postimg.cc/xN236YPM) 
### Extension.rs 
[![f9-uml-drawio.png](https://i.postimg.cc/dQnPn5Kn/f9-uml-drawio.png)](https://postimg.cc/DmWMwd0b) 
### Enhanced Smart Contract Capabilities with Pink Chain Extension

The Pink Chain Extension introduces advanced capabilities to smart contracts running on the Pink protocol, enriching them with external interactions, cryptographic operations, and custom event management.

### External HTTP Requests

- **HTTP Request Handling**: Enables contracts to initiate HTTP requests directly, allowing interaction with external web services and APIs. This is crucial for contracts requiring access to off-chain data or services.

### Cryptographic Operations

- **Signature and Verification**: Provides functionality for signing messages and verifying signatures within contracts, leveraging different signature schemes. This enhances contract security and authenticity checks.

- **Key Derivation**: Supports the derivation of new keys from a base key, enabling more sophisticated cryptographic schemes and privacy-enhancing techniques within contracts.

### Cache Operations and Logging

- **Cache Management**: Facilitates the temporary storage of data in a cache, supporting operations like setting, getting, removing, and managing the expiration of cache entries. This is vital for optimizing data access and storage within contracts.

- **Logging**: Offers a logging mechanism for contracts, allowing them to emit logs at various levels (e.g., error, info, debug). This aids in debugging and monitoring contract execution.

### Randomness and Contract Queries

- **Random Number Generation**: Allows contracts to request random numbers, useful in applications requiring unpredictability, such as gaming or lottery contracts.

- **Query Execution Mode**: Determines the execution context (e.g., query or transaction), affecting the availability and behavior of certain operations, ensuring appropriate use of resources and functionalities.

### System Contract Interaction and Balance Queries

- **System Contract Queries**: Permits contracts to query the identity of the system contract and interact with it, centralizing system-level operations and configurations.

- **Balance Inquiries**: Enables contracts to query the balance of any account, supporting operations that depend on financial conditions or require balance checks.

### JavaScript Evaluation and SGX Quote Retrieval

- **JavaScript Evaluation**: Provides the capability to evaluate JavaScript code from within contracts, opening up possibilities for dynamic script execution and complex computation.

- **SGX Quote Retrieval**: Allows contracts to retrieve the SGX quote of the executing environment, essential for applications needing to verify the integrity and authenticity of the execution environment.

### Event Emission and Versioning

- **Custom Event Emission**: Supports the emission of custom events by contracts, facilitating communication and interaction patterns between contracts and external observers.

- **Runtime Versioning**: Helps contracts ascertain the runtime version they are operating in, enabling compatibility checks and feature toggling based on version differences.

This chain extension significantly expands the functionalities available to smart contracts on the Pink platform, enabling more complex, interactive, and secure applications.
[![f9-seq-drawio.png](https://i.postimg.cc/m2P0yMN5/f9-seq-drawio.png)](https://postimg.cc/4mTLJY06) 
### Mod.rs
[![f10-uml-drawio-1.png](https://i.postimg.cc/sxPxx86t/f10-uml-drawio-1.png)](https://postimg.cc/PvxTF6k4) 
### Cross-Call and Execution Interface for Pink Contracts

The design introduces a flexible framework for Pink contracts, allowing them to interact with the host environment and other contracts through a set of well-defined interfaces and execution modes. 

### Cross-Call Traits

- **CrossCall**: Provides a standard interface for contracts to invoke other contracts or the host environment, identified by unique IDs, passing data back and forth.
- **CrossCallMut**: An extension of `CrossCall` allowing mutable operations, enhancing contracts' ability to interact dynamically with their ecosystem.
- **Executing**: Abstracts execution logic, offering methods to execute functions both immutably and mutably, catering to different operational contexts within contracts.

### Execution Utilities

- **IdentExecute**: A basic implementation of the `Executing` trait, directly passing through function invocations without additional logic, serving as a simple execution model.

### Enhanced Contract Functionality

- **ECall and OCall**: Traits extending `CrossCall`, specializing in internal (ECall) and external (OCall) operations, further structuring the interaction model of contracts.

### ECall: Enhanced Contract Operations

Detailed methods under `ECalls` trait encompass a wide range of functionalities:

- **Cluster Management**: Setup and identification of contract clusters for organized execution environments.
- **Balance Management**: Facilitates deposit operations and queries related to balances, improving financial interactions within the contract ecosystem.
- **Code Management**: Upload and management of contract code, ensuring flexibility in contract deployment and operation.
- **Contract Interaction**: Advanced methods for contract instantiation and calls, broadening the scope of contract interactivity.
- **System Utilities**: Includes versioning and system contract interactions, providing essential tools for contract development and execution.

### OCall: Outward Bound Calls from Contracts

Methods under `OCalls` address various needs:

- **Storage Operations**: Direct interactions with the storage layer, including commits and queries, ensuring contracts have essential data management capabilities.
- **Logging and Event Management**: Supports logging and event emission, enhancing the observability of contract operations.
- **Execution Context and Identity**: Offers insights into the execution context and contract identities, essential for context-aware operations.
- **Networking and Computation**: Includes HTTP requests and JavaScript evaluation capabilities, extending contracts' ability to communicate externally and perform complex computations.
- **Security and Identity**: Addresses security concerns and identity verification through SGX quote retrieval, contributing to the integrity and trustworthiness of contract operations.

### Testing and Utility Functions

- **Coverage Tests**: Ensures comprehensive testing coverage, validating the functionality and reliability of the interfaces and their implementations.
[![f10-seq-drawio-1.png](https://i.postimg.cc/28N5szcT/f10-seq-drawio-1.png)](https://postimg.cc/MXtxR88j) 
### Lib.rs
[![f12-uml-drawio-2.png](https://i.postimg.cc/wM2DG08Y/f12-uml-drawio-2.png)](https://postimg.cc/4HKKKvxW) 
### Enhanced Networking and Crypto Utilities for Pink Contracts

This module extends the functionality of Pink contracts, offering sophisticated networking, cryptographic operations, and state management through a set of utilities and traits.

### Networking Utilities

- **HTTP Requests**: Allows contracts to perform HTTP(S) requests, supporting both single and batch requests with customizable timeouts and handling for various HTTP methods and headers.
- **Batch HTTP Requests**: Efficiently handles multiple HTTP requests concurrently, imposing limits to prevent overload, and gracefully handling timeouts.

### Cryptographic Operations

- **Signature and Verification**: Facilitates signing and verification operations across different signature types (Sr25519, Ed25519, Ecdsa), including key derivation and public key retrieval.
- **ECDSA Operations**: Specifically tailored functions for ECDSA, including prehashed signing and verification, leveraging the contracts' ability to interact with cryptographic elements securely.

### State Management and Logging

- **Local Cache Management**: Introduces a local caching mechanism for contracts, enabling them to store and retrieve data efficiently, with support for expiration settings.
- **Logging**: Provides a mechanism for contracts to log messages at different levels (Error, Warn, Info, Debug, Trace), enhancing the debugging and monitoring capabilities of contract execution.

### Environment and Backend Extensions

- **Runtime Environment Interface**: Defines a contract's runtime environment, encapsulating the address and providing a foundation for more complex behaviors.
- **Pink Runtime Backend**: Offers a default implementation of the Pink extension backend, implementing the core functionalities required by Pink contracts, including HTTP requests and cryptographic operations.

### Utilities and Test Modules

- **Utility Functions**: Includes utility functions for asynchronous HTTP request processing, timeout handling, and response size limiting, ensuring robust and efficient network communication.
- **Mock Extensions and Tests**: Provides a mock extension for testing and demonstration purposes, alongside comprehensive tests covering various scenarios, such as handling large batch HTTP requests, network errors, and timeout conditions.

### Key Highlights

- The module significantly enhances the Pink contract ecosystem's capabilities, offering advanced networking, cryptographic operations, and efficient state management.
- It lays the foundation for building complex decentralized applications on Pink, enabling secure, reliable, and efficient interactions with the external world and within the blockchain environment.
[![f12-seq-drawio-1.png](https://i.postimg.cc/2jgxgXZr/f12-seq-drawio-1.png)](https://postimg.cc/fkjX9CCr) 
### Local_cache.rs
[![f13-uml-drawio-1.png](https://i.postimg.cc/05XTZK48/f13-uml-drawio-1.png)](https://postimg.cc/nM7ShrZ5) 
### Local Cache for Pink Contracts

The LocalCache module provides a mechanism for Pink contracts to utilize a local, key-value cache for off-chain computations, facilitating temporary data storage with customizable expiration and quota management.

#### Key Features:

- **Local and Transient Storage**: Enables contracts to store data locally and transiently, with the cache contents being machine-specific and potentially volatile across restarts or due to expiration mechanisms.
- **Test Mode Support**: Allows enabling a test mode for unit testing, ensuring thread-local cache instances for isolated testing environments.
- **Global Cache Access**: Facilitates access to a global, mutex-protected cache instance for managing shared state across different parts of a contract.
- **Storage Management**: Implements a storage structure that tracks the cumulative size of keys and values, enforces maximum storage quotas, and supports garbage collection to maintain cache size within predefined limits.
- **Expiration and Cleanup**: Offers functions to clear expired cache entries based on timestamps, with customizable expiration periods for stored values.
- **Cache Operations**: Provides utility functions for cache operations such as setting, getting, removing values, and applying expiration times, along with higher-level functions for applying quotas and handling predefined cache operations (e.g., CacheOp::Set, CacheOp::Remove).

#### Cache Operation Implementation:

- **Set Operation**: Stores a key-value pair in the cache, optionally with a specified lifetime. Ensures the total cache size does not exceed the maximum quota, performing garbage collection if necessary.
- **Get Operation**: Retrieves the value associated with a given key from the cache, considering the entry's expiration.
- **Remove Operation**: Deletes a key-value pair from the cache, adjusting the total cache size accordingly.
- **Expiration Management**: Allows setting an expiration time for a specific cache entry, automatically removing expired entries during garbage collection.
- **Quota Management**: Enables applying storage quotas to specific contracts or cache segments, ensuring cache usage stays within allocated limits and facilitating cache size fitting when necessary.

#### Usage Scenarios:

- **Off-Chain Computations**: Temporarily storing data required for off-chain computations, reducing on-chain storage needs and enabling more complex contract logic.
- **Caching External Data**: Caching data retrieved from external sources (e.g., via HTTP requests) for short-term reuse within the contract, optimizing network usage and computation times.
- **State Management for Contract Logic**: Managing transient state within contracts for scenarios where persistence across contract calls is unnecessary or undesirable.

This module enhances the flexibility and efficiency of Pink contracts by providing robust mechanisms for transient data storage and management, supporting a wide range of decentralized application use cases.
[![f13-seq-drawio-1.png](https://i.postimg.cc/W18CFD0L/f13-seq-drawio-1.png)](https://postimg.cc/cvKXPLDX) 
### Risk Assessment of the Pink Contracts Local Cache System

#### Centralization Risk

- **Single Point of Failure**: Utilizing a local cache introduces a risk where data essential for contract execution becomes inaccessible if the node or system hosting the cache encounters issues. This risk is heightened in scenarios where the cached data is not replicated or backed up externally, leading to potential data loss or downtime.
- **Inconsistent State Across Nodes**: Since the cache is local and transient, there's a risk of creating inconsistencies across different nodes executing the same contract. This could lead to divergent contract behaviors or outcomes, complicating consensus and potentially undermining the trust in the contract's execution integrity.

#### Systematic Risk

- **Dependency on External Factors**: The reliance on system time for cache expiration and garbage collection introduces a risk where system clock inaccuracies or manipulations could affect cache behavior. Furthermore, external environmental factors, such as system restarts or network issues, can unpredictably impact the availability and reliability of the cached data.
- **Impact of Runtime Environment Changes**: Updates or changes to the contract runtime environment, including upgrades to the Pink contracts framework or underlying blockchain platform, could introduce backward compatibility issues or unexpected behaviors in cache management and access patterns.

#### Architecture Risks

- **Garbage Collection Mechanism**: The implemented garbage collection strategy, while necessary for managing cache size, introduces performance variability. Intensive garbage collection processes can lead to increased execution times for contract calls, potentially impacting the overall performance of the contract under high load or when nearing storage quota limits.
- **Cache Quota Management**: The mechanism for applying storage quotas and managing cache size introduces a risk of inadvertently restricting contract functionality. Misconfiguration or overly aggressive quota limits could prevent essential data from being cached, leading to degraded performance or the inability to execute certain contract operations.

#### Complexity Risks

- **Increased Contract Complexity**: Incorporating local cache logic into contract development introduces additional complexity, requiring developers to manage cache states, handle expiration logic, and ensure consistency across executions. This complexity can increase the likelihood of bugs or oversights that compromise contract functionality or security.
- **Testing and Verification Challenges**: The transient and local nature of the cache, combined with the variability introduced by garbage collection and expiration mechanisms, complicates testing and verification. Ensuring comprehensive coverage across different scenarios and runtime environments becomes more challenging, increasing the risk of unanticipated issues in production.

## Conclusion

Phala Network embodies a groundbreaking shift towards decentralized and trustless computation, marking a significant advancement for the Web3 ecosystem. By melding the power of Phat Contracts with smart contracts, it opens new horizons for dApp development, ensuring scalability, security, and efficiency. The network not only empowers developers with advanced computational tools but also invites community participation through a comprehensive reward system. Phala Network stands as a testament to the potential of decentralized technologies in shaping a more secure and interoperable digital future.





### Time spent:
17 hours