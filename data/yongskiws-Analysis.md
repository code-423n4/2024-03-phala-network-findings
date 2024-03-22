| Title                              | Description                                                                                                                                                                                                                                  |
|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Phala Network introduction         | Phala Network is a groundbreaking platform in the Web3 space, offering dApp developers a decentralized and trustless off-chain compute infrastructure through its innovative Phat Contracts.                                                 |
| Mission and Vision                 | Phala Network's mission is to revolutionize Web3 by providing developers with the tools they need to build advanced decentralized applications.                                                                                           |
| Off-Chain Computation              | Phat Contracts solve issues with on-chain computation by running off-chain on Phala Network's workers, allowing for complex logic execution without the constraints of on-chain computation.                                                |
| Features and Use Cases             | Phat Contracts offer a range of features, including connectivity to any API, integration with S3 storage platforms, computation over data, and automation of smart contracts.                                                                |
| Trustless Computation              | Phala Network ensures trustless computation through multiple layers of security guarantees.                                                                                                                                                  |
| Cross-Chain Compatibility          | Phat Contracts can connect to any blockchain, including EVM and Substrate blockchains, without the need for bridges.                                                                                                                         |
| Stake-to-Compute Model             | Developers and end-users must stake PHA tokens to use Phala Network, ensuring fair and efficient resource allocation.                                                                                                                        |
| PHA Token Utility                  | PHA is the native token of the Phala blockchain, used for rewarding workers, staking to run Phat Contracts, and participating in the governance of the network.                                                                              |
| Supported Chains                   | Phat Contract can connect to any blockchain for reading and writing operations. Presently, it has extensive support for EVM and Substrate blockchains.                                                                                      |
| EVM blockchains                    | At the Native Phat Contract level, it can interact with any EVM-compatible blockchains through their RPC nodes, including Ethereum, Polygon, Arbitrum, BSC, Optimism, and others.                                                          |
| Substrate blockchains              | Phat Contract fully supports Substrate-based blockchains, including Polkadot, Kusama, Phala Network, Astar, and others.                                                                                                                      |
| Runtime Architecture               | Manages aspects such as system time, account balances, and smart contracts using various modules and configurations.                                                                                                                        |
| Contract Architecture              | Manages gas management, contract execution, and transaction logic for contract execution.                                                                                                                                                   |
| Extension Architecture             | An implementation of a chain extension for smart contracts running within the Pink Network environment, providing additional functions such as making HTTP requests, caching data, and performing cryptographic operations.               |
| Pallet Pink Architecture           | Defines various types, storage items, and functions related to the blockchain, including error handling, configurations, and storage manipulation.                                                                                           |
| ecall_impl Architecture            | Implements the ECalls trait for an ECallImpl structure, providing functions to interact with contracts on the blockchain.                                                                                                                    |
| lib.rs                             | Default implementation of the `PinkExtBackend` trait for interacting with the Pink extension, providing functions for making HTTP requests, signing data, verifying signatures, and managing cache.                                        |
| Local Cache                        | Provides a local KV cache for contracts to perform off-chain computations, with structures and functions for managing cache operations.                                                                                                     |

  
  

# Phala Network introduction

  ![images (2)](https://i.imgur.com/FCYcHMg.png)

Phala Network is a groundbreaking platform in the Web3 space, offering dApp developers a decentralized and trustless off-chain compute infrastructure through its innovative Phat Contracts. These contracts enable developers to enhance their dApps with features like seamless cross-chain integrations, internet connectivity, and heavy computation, all while maintaining the principles of Web3.

  

## Mission and Vision

  

Phala Network's mission is to revolutionize Web3 by providing developers with the tools they need to build advanced decentralized applications. Through Phat Contracts, developers can make their smart contracts even smarter, unlocking new possibilities for their dApps.

  

## Off-Chain Computation

  

On-chain computation has limitations, such as high costs and difficulty in interacting with off-chain data sources. Phat Contracts solve these issues by running off-chain on Phala Network's workers, allowing for complex logic execution without the constraints of on-chain computation.

  

## Features and Use Cases

  

Phat Contracts offer a range of features, including connectivity to any API, integration with S3 storage platforms, computation over data, and automation of smart contracts. These capabilities enable developers to create powerful and versatile dApps for various use cases.

  

## Trustless Computation

  

Phala Network ensures trustless computation through multiple layers of security guarantees. Workers, who contribute computing power to the network, are incentivized to carry out computation faithfully and securely, ensuring verifiable results.

  

## Cross-Chain Compatibility

  

Phat Contracts can connect to any blockchain, including EVM and Substrate blockchains, without the need for bridges. This universal compatibility expands the capabilities of smart contracts and dApps, enhancing their functionality and interoperability.

  

## Stake-to-Compute Model

  

To use Phala Network, developers and end-users must stake PHA tokens. This stake is used to allocate computing resources among contract clusters and their end-users, ensuring fair and efficient resource allocation.

  

## PHA Token Utility

  

PHA is the native token of the Phala blockchain, used for rewarding workers, staking to run Phat Contracts, and participating in the governance of the network. It can also be staked or delegated to earn rewards and support network security.

  

## Supported Chains

  

Phat Contract can connect to any blockchain for reading and writing operations. Presently, it has extensive support for EVM and Substrate blockchains.

  

## EVM blockchains

  

At the Native Phat Contract level, it can interact with any EVM-compatible blockchains through their RPC nodes, including Ethereum, Polygon, Arbitrum, BSC, Optimism, and others.

  

## Substrate blockchains

  

Phat Contract fully supports Substrate-based blockchains, including Polkadot, Kusama, Phala Network, Astar, and others.

  

Phat Contract's cross-chain capabilities allow for seamless integration with various blockchains, and supporting additional blockchains does not require changes to Phala Network's infrastructure. Developers can implement their RPC client, serialization, and signing libraries and share them with the community.

  

In conclusion, Phala Network's Phat Contracts offer a powerful solution for developers looking to build advanced dApps in the Web3 ecosystem. By providing a decentralized and trustless off-chain compute infrastructure, Phala Network is paving the way for the next generation of decentralized applications.

  

# Runtime Architecture

    

![images (2)](https://i.imgur.com/uZMQ81U.png)

*To configure the media runtime, this code uses various modules and configurations to manage aspects such as system time, account balances, and smart contracts.*

  

## Code Base Characteristics

-  **Modularity:** The code is well divided into different modules such as System, Timestamp, Balances, Randomness, Contracts, and Pink, which aids in maintenance and further development.

-  **Flexible Configuration:** The use of `parameter_types` provides the ability to easily configure important parameters.

-  **Unit Tests:** Well-defined unit tests can help maintain code quality.

-  **Documentation:** The code has fairly clear and detailed documentation.

  

## Informational

-  **External Usage:** The code uses several external modules such as `frame_support`, `log`, and `sp_runtime`, indicating integration with existing frameworks and libraries.

-  **Key Concepts:** Concepts like block weight settings, deposit limits, and smart contract configurations have been implemented effectively.

  
  

# Contract Architecture

  

Functions related to gas management (transaction costs) and contract execution, such as masking gas and deposit values, as well as managing transaction logic for contract execution.

  

**Admin Abuse Risk:**

If the `origin` parameter in the `contract_tx` function is not properly controlled or authenticated, there is a risk of misuse. The `origin` parameter is used to pay for gas and return gas, so if unauthorized users can manipulate this parameter, it could allow them to abuse the system.

  

**Systemic Risk:**

There is a potential systemic risk related to gas calculation and management. Gas calculation logic is quite complex, and any bugs or miscalculations could lead to unexpected behavior or vulnerabilities.

  

**Technical Risk:**

There is a technical risk related to the correctness and efficiency of the code. The use of macros and bitwise operations, while efficient, can be prone to errors if not implemented and tested carefully.

  

**Integration Risk:**

The `mask_gas`, `mask_deposit`, and `coarse_grained` functions change gas and storage values, which can affect the behavior of other contracts or systems interacting with this contract.

  

**Software Engineering Considerations:**

- Well-structured and uses common Rust idioms and patterns.

- The use of macros for bit manipulation (`define_mask_fn`) is a good way to abstract and reuse code but can make the code difficult to understand for those unfamiliar with the macro implementation.

  
  

# Extension Architecture

is an implementation of a chain extension for smart contracts running within the Pink Network environment. This chain extension provides additional functions that can be called from within smart contracts, such as making HTTP requests, caching data, performing signing and verification for cryptographic operations, and so on.

  

This implementation leverages data structures from `pallet_contracts` and `pink` to interact with contracts and events on the Pink Network blockchain. Additionally, it also utilizes various functions from `phala_crypto` for cryptographic operations like signing and verification using SR25519 keys, as well as `sp_runtime` and `frame_support`.

  

## Software Engineering and Architecture Considerations

- The architecture follows good design patterns with a clear separation of responsibilities between various components.

- The use of `ChainExtension` and separation of Query and Command modes is a good approach to manage chain extensions securely.

- Separating code for processing HTTP requests in Query and Command modes is also a good approach to maintain data consistency.

  
  

# Pallet Pink Architecture

  ![images (2)](https://i.imgur.com/WrK5fhJ.png)

The code is part of the pallet module that defines various types, storage items, and functions related to the blockchain.

  

-  `pub use pallet::*;`: Export all items from the pallet module.

-  `pub mod pallet { ... }`: Define the pallet module containing the pallet definition and its implementation.

-  `#[pallet::error] pub enum Error<T> { ... }`: Error enum used to indicate the types of errors that can occur within the pallet.

-  `#[pallet::config] pub trait Config: frame_system::Config { ... }`: Config trait required for pallet configuration.

- Definition of several alias types, such as `BalanceOf<T>`, `AccountIdOf<T>`, and `HashOf<T>`, used in the pallet.

- Several storage items (`ClusterId`, `GasPrice`, `DepositPerByte`, etc.) that store information on various configurations and important data within the blockchain.

- Implementation of several functions for manipulating storage items, such as `set_cluster_id`, `put_sidevm_code`, `pay_for_gas`, etc.

- Utility functions like `pay`, `refund`, and `convert` used for asset transfer, refund, and conversion of transaction weight to the required amount.

-  `impl<T: Config + pallet_contracts::Config> AddressGenerator<T> for Pallet<T> { ... }`: Implementation of the function to generate contract addresses based on user address, contract hash code, and some other parameters.

  

**Technical Risks:** Technical risks include issues in contract logic and data manipulation. This code has several areas that require extra attention regarding secure data processing and input validation.

  

**Software Engineering Considerations and Architecture Assessment:**

-  **Optimization:** It needs to be evaluated whether memory usage and other resources are well optimized.

-  **Scalability:** Evaluate the system's scalability with this pallet and the possible changes needed to support system growth.

-  **Monitoring and Troubleshooting:** Consider how this system will be monitored and how issues will be addressed if they occur.

  
  

# ecall_impl Architecture 

  ![images (2)](https://i.imgur.com/5s5g1Pp.png)

Implementation of the ECalls trait for an ECallImpl structure. The ECalls trait provides various functions that can be called to interact with contracts on the blockchain. This implementation uses functions from the Substrate pallet to perform various operations, such as creating contracts, sending transactions, and more.

  

One example of using this implementation is in the setup function, which is used to configure the cluster. Inside this function, there are calls to functions from the PalletPink pallet to configure the cluster, including setting the cluster ID, gas price, deposit, and more.

  

This implementation also contains several utility functions, such as the sanitize_args function to limit the gas used, and the handle_deposit function to handle the deposit that needs to be paid.

  

## Systemic Risks:

-  **Cluster Setup Reliability:** The cluster setup process must be thoroughly tested to ensure the reliability and consistency of the system.

-  **Dependency on External Modules:** Dependency on external modules such as `pallet_contracts` and `phala_crypto` can pose risks if these modules do not adhere to the expected specifications.

  

## Technical Risks:

-  **Scalability:** With gas limitations, it is necessary to ensure that the system can handle increasing transaction loads effectively.

  

## Software Engineering Considerations

  

## Architecture and Design:

-  **Modular Code Design:** Modular design will facilitate maintenance and further development.

  

## Testing

-  **Security Testing:** Conduct comprehensive security testing to ensure that the system is resilient to attacks such as reentrancy and overflow attacks.

  
  

# Lib Architecture 

  ![images (2)](https://i.imgur.com/HJ8jyb6.png)

Default implementation of the `PinkExtBackend` trait, which provides functions for interacting with the Pink extension (`substrate-chain-extension`). The Pink extension is used to communicate with external entities, such as making HTTP requests, managing cache, and more. This implementation includes basic functions like performing HTTP requests, signing data, verifying signatures, and so on.

  

Some key points of this implementation are:

- Functions `http_request` and `batch_http_request` for performing HTTP requests, both single and batch.

- Functions for signing data and verifying signatures with various cryptographic algorithms like Sr25519, Ed25519, and ECDSA.

- Functions for managing cache, logging, and other functions that support blockchain operations.

- This implementation uses `tokio` to run async operations in a non-blocking manner.

  

**Admin Abuse Risks**: in the `cache_set`, `cache_set_expiration`, and `cache_remove` methods that do not have additional security mechanisms such as access control. This can lead to abuse of access to cache data.

  

**Systemic Risks**: may occur if the system is unable to handle a large number of HTTP requests in the `batch_http_request` method. Although there is a maximum request limit, there is no mechanism to manage a larger request queue.

  

**Technical Risks**: such as the use of `try_current` in `block_on` which will not provide proper handling if there is no active `tokio` runtime. Additionally, error handling in HTTP client creation (`FailedToCreateClient`) is also quite common and can be fixed with more specific handling.

  

**Integration Risks**: may have integration risks if dependent on externals such as dependencies on `reqwest` and `getrandom`. Changes or updates to these dependencies can affect overall functionality.

  

**Performance**: Continuously check if the implementation of HTTP requests, caching, and other functions has been optimized for good performance and does not burden the system.

  

**Security**: Ensure that adequate security mechanisms have been implemented, especially in terms of cache management and network communication.

  
  

# Local Cache

  

Implementation of `LocalCache` providing a local KV cache for contracts to perform off-chain computations. Data stored in this cache varies across machines of the same contract, and may be lost on pruntime restarts or due to expiration mechanisms.

  

## Structures

  

### Storage

- Stores cache keys, values, and expiration times.

- Includes logic to remove expired items to maintain desired size.

  

### LocalCache

- Manages storages and provides functions to access, add, or remove data.

- Includes logic to periodically clean the cache.

  

## Functions

  

-  `apply_cache_op`: Apply cache operations (add, remove, set expiration).

-  `set`, `get`, `set_expiration`, `remove`: Perform specific cache operations.

-  `apply_quotas`: Apply cache size limits for contracts.

  

## Risks

  

### Admin Abuse

- Uses `AtomicBool` and mutex to control cache access.

- Lacks clear authentication or authorization mechanism, posing a risk for tighter access control.

  

### Systemic

- Thread safety concerns despite efforts (Mutex, thread_local).

- Potential deadlock or race conditions in multi-threaded environments.

  

### Technical

- Unsafe operations in `set`, `get`, `set_expire`, and `remove`.

-  `get` without protection could lead to panics.

-  `fit_size` and `clear_expired` complexity may lead to bugs or vulnerabilities.

### Time spent:
53 hours