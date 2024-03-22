| Serial No. | Topic                                                                                     |
| ---------- | ----------------------------------------------------------------------------------------- |
| 01         | [Overview](#1-overview)                                                                   |
| 02         | [Architecture Overview](#2-architecture-overview)                                         |
| 03         | [Approach Taken in Evaluating Pink Runtime](#3-approach-taken-in-evaluating-pink-runtime) |
| 04         | [Runtime Components Analysis](#4-runtime-components-analysis)                             |
| 05         | [Codebase Quality](#5-codebase-quality)                                                   |
| 06         | [Systematic Risks](#6-systematic-risks)                                                   |

## **1. Overview**

The Pink Runtime is a substrate execution engine for executing smart contracts in the Phala Network. It's generally designed to provide a trustless environment for executing smart contracts - hence confidentiality is primarily built-in. The Phala Protocol through its use of the Rust language developed the runtime building on top of Substrate's `pallet-contracts`. Pink's computation is done offchain in a trustless and decentralized environment.

At its core, the protocol employs data interoperability in the sense that Phat contracts deployed on the Phala Network can communicate or be interfaced with smart contracts on any other chain.

## **2. Architecture Overview**

During the course of this analysis for the Pink Runtime, we engaged the protocol with a function-trace and deep dive overview of the protocol to dissect each function, logic and core implementation. These points are talked about in the Runtime Components Analysis section of this report.

Flaws of functions we covered in the Runtime Component Analysis section is available as individual components for such functions in the Systemic risk sections along with insights to make the logic better.

## **3. Approach Taken in Evaluating Pink Runtime**

Before we delve in, it is crucial to understand what a runtime is in the substrate term. A runtime has all the business logic for executing transactions, transitioning & saving state as well as interacting with other nodes. In essence, it's like the EVM but for executing transactions in another network being Phala. To understand substrate and what the Phala Network is doing, we had to spend some time building a runtime from scratch ourselves because that is how we can truly understand substrate's working under the hood. We spent 7 days looking into the protocol in total with 2-3 hours allocated each day.

We focused mainly in the runtime's initialization, runtime configurations, runtime calls, smart contract instantiation and execution and finally storage within the pink runtime. With the limited time we had to look into this codebase, we spent the most of our time covering the above mentioned logic of the runtime.

**D1-2:**

- Getting context on Phala from the provided docs: [Docs](https://docs.phala.network/)
- Getting an understanding of Substrate runtimes

**D2-4:**

- Brainstorm some attack vectors relevant to execution engines
- Set out building a runtime from scratch
- Further reviews of the substrate runtime topic

**D4-7:**

- Delving deeper into the codebase
- Draft & submission of report

## **4. Runtime Components Analysis**

### 1. Initialization:

```rs
#[no_mangle]
pub unsafe extern "C" fn __pink_runtime_init(
    config: *const config_t,
    ecalls: *mut ecalls_t,
) -> ::core::ffi::c_int
```

`(__pink_runtime_init)` function: This is the entry point for the runtime initialization. It takes two parameters: config (a pointer to configuration data) and ecalls (a pointer to the table of external calls). Firstly, it sets the `OCall` function using `ocall_impl::set_ocall_fn`. This function handles the initialization of `OCall` functions.

Afterwards, it checks if the `ecalls` pointer is `null`, logs an error if it is, and returns `-1`.

It sets the `ecall` and `get_version` functions in the `ecalls` table and if the `config.is_dylib` is `true`, it initializes the `logger`. When all that is done, the initialization function finally returns `0` to indicate a successful initialization.

### 2. Runtime versioning:

```rs
unsafe extern "C" fn get_version(major: *mut u32, minor: *mut u32) {
```

`(get_version)` function: This function sets the major and minor version numbers using the `crate::version()` function.

### 3. ECall Dispatchs

```rs
unsafe extern "C" fn ecall(
    call_id: u32,
    data: *const u8,
    len: usize,
    ctx: *mut ::core::ffi::c_void,
    output_fn: output_fn_t,
) {
```

`(ecall)` function: This function serves as a central hub for all `ECall` functions.

It takes parameters such as a `call_id` (which is a unique identifier for the function call), `data` ( apointer to the input data), `len` (the size of input data), `ctx` (the context pointer), and `output_fn` (a function pointer to receive the output).

It uses `ecall::dispatch` to route function calls to `ECallImpl`, which implements the `ECalls` trait.

Finally, it invokes the output function to send the results back.

### 4. ECall Implementations

```rs
macro_rules! instrument_context {
    ($context: expr) => {
```

`(instrument_context!)` macro definition: This macro takes a context parameter and is used for instrumentation, such as for tracing or logging. It creates spans for `prpc` (for remote procedure calls) and `pink` (referring to the Pink Runtime) with additional attributes like `blk` for block number and `req_id` for request ID.

It utilizes the tracing crate for span creation and its management.

```rs
impl Executing for crate::storage::ExternalStorage {
```

Executing Trait Implementations: The Executing trait is implemented for `ExternalStorage`, which is used for executing functions within the context of the Pink Runtime.

```rs
fn execute<T>(&self, f: impl FnOnce() -> T) -> T {
```

The `execute` function defined within the `Executing` implementation takes a closure and executes it within the context of the Pink Runtime.

It then retrieves the execution context using `OCallImpl.exec_context()` and instruments it using the `instrument_context!` macro.

```rs
fn execute_mut<T>(&mut self, f: impl FnOnce() -> T) -> T
```

The `execute_mut` function defined in the same `Executing` implementation behaves similarly to the `execute` function, but in this context, it takes a `mutable` reference to self i.e Pink Runtime and additionally emits side effects using `OCallImpl.emit_side_effects()`.

```rs
impl ecall::ECalls for ECallImpl {
```

This implementation provides various functions to interact with the Pink Runtime such as:

- `fn cluster_id(&self) -> Hash {`: For retrieval of cluster IDs from the Pink Pallet.
- `fn setup(&mut self, config: ClusterSetupConfig) -> Result<(), String> {`: Setting up the Pink cluster with provided configuration.
- `fn deposit(&mut self, who: AccountId, value: Balance) {`: Depositing funds into an account using the Balances Pallet
- `fn set_key(&mut self, key: Sr25519SecretKey) {`: Sets the secret key for the Pink Pallet
- `fn get_key(&self) -> Option<Sr25519SecretKey> {` : Retrieves the secret key for the Pink Pallet
- `fn upload_code(
    &mut self,
    account: AccountId,
    code: Vec<u8>,
    deterministic: bool,
) -> Result<Hash, String> {`: Uploads wasm code to the Pink Pallet.
- `fn upload_sidevm_code(&mut self, account: AccountId, code: Vec<u8>) -> Result<Hash, String> {`: Uploads wasm code to the Pink Pallet.
  `fn contract_instantiate(
      &mut self,
      code_hash: Hash,
      input_data: Vec<u8>,
      salt: Vec<u8>,
      mode: ExecutionMode,
      tx_args: TransactionArguments,
  ) -> Vec<u8> {`: Executing contract instantiation on the Pink Pallet.
- `fn contract_call(
    &mut self,
    address: AccountId,
    input_data: Vec<u8>,
    mode: ExecutionMode,
    tx_args: TransactionArguments,
) -> Vec<u8> {`: Executing contract calls on the Pink Pallet.

```rs
fn sanitize_args(mut args: TransactionArguments, mode: ExecutionMode) -> TransactionArguments {
    const GAS_PER_SECOND: u64 = WEIGHT_REF_TIME_PER_SECOND * 5;
    let gas_limit = match mode {
        ExecutionMode::Transaction | ExecutionMode::Estimating => GAS_PER_SECOND / 2,
        ExecutionMode::Query => GAS_PER_SECOND * 10,
    };
    args.gas_limit = args.gas_limit.min(gas_limit);
    args
}
```

- The `sanitize_args()` function adjusts the gas limit based on the execution mode (Transaction, Estimating, or Query).
- It calculates the gas limit based on a constant `GAS_PER_SECOND`, which represents the gas consumed per second.
- For transactions and estimation, it sets the gas limit to half a second's worth of gas. For queries, it sets the gas limit to 10 seconds' worth of gas.
- The calculated gas limit is then clipped to ensure it does not exceed the maximum allowed gas limit specified in the transaction arguments.
- Lastly, the sanitized transaction arguments are returned.

```rs
fn handle_deposit(args: &TransactionArguments) {
    if args.deposit > 0 {
        let _ = PalletBalances::deposit_creating(&args.origin, args.deposit);
    }
}
```

The `handle_deposit()` function handles `deposits` specified in the transaction arguments.
If the deposit amount `(args.deposit)` is greater than `0`, it deposits the specified amount of funds into the origin account using the `deposit_creating` function from the `Balances` Pallet.

#### Error Handling:

Errors are handled using `Result` types with `String` error messages, providing detailed error information in case of failure.

Error messages are logged using the log crate and also sent to the server through `OCallImpl.log_to_server`.

### 5. OCall Implementations -> Outgoing Calls

```rs
static mut OCALL: InnerType<cross_call_fn_t> = _default_ocall;
```

- The line of code above defines a static mutable variable `OCALL` of type `InnerType<cross_call_fn_t>`. This variable is used to store the function pointer for handling `OCall` operations.
- It is initially set to `_default_ocall`, which is a function that panics with a message indicating that no `OCall` function is provided.

```rs
pub(super) fn set_ocall_fn(ocalls: ocalls_t) -> Result<(), &'static str> {
    let Some(ocall) = ocalls.ocall else {
        return Err("No ocall function provided");
    };
    unsafe {
        OCALL = ocall;
        #[cfg(feature = "allocator")]
        if let Some(alloc) = ocalls.alloc {
            allocator::ALLOC_FUNC = alloc;
        }
        #[cfg(feature = "allocator")]
        if let Some(dealloc) = ocalls.dealloc {
            allocator::DEALLOC_FUNC = dealloc;
        }
    }
    Ok(())
}
```

- The `set_ocall_fn()` function sets the `OCall` functions by accepting an `ocalls_t` structure containing `OCall` function pointers.
- It then extracts the `OCall` function from the `ocalls` parameter and sets it to the `OCALL` variable.
- If the allocator feature is enabled, it also sets the allocator and deallocator functions.

```rs
pub(crate) struct OCallImpl;
impl CrossCallMut for OCallImpl {
    fn cross_call_mut(&mut self, call_id: u32, data: &[u8]) -> Vec<u8> {
        self.cross_call(call_id, data)
    }
}

impl CrossCall for OCallImpl {
    fn cross_call(&self, id: u32, data: &[u8]) -> Vec<u8> {
        unsafe extern "C" fn output_fn(ctx: *mut ::core::ffi::c_void, data: *const u8, len: usize) {
            let output = &mut *(ctx as *mut Vec<u8>);
            output.extend_from_slice(std::slice::from_raw_parts(data, len));
        }
        unsafe {
            let mut output = Vec::new();
            let ctx = &mut output as *mut _ as *mut ::core::ffi::c_void;
            OCALL(id, data.as_ptr(), data.len(), ctx, Some(output_fn));
            output
        }
    }
}
impl OCall for OCallImpl {}
```

- The `OCallImpl` struct is defined to handle `OCall` operations.
- It implements the `CrossCallMut`, `CrossCall`, and `OCall` traits.
- `CrossCall` and `CrossCallMut` traits define methods for making cross calls (calls from the enclave to outside world) with immutable and mutable references.
- The `cross_call` and `cross_call_mut` methods use the `OCALL` function pointer to execute `OCall` operations.

### 6. Runtime configurations

The configuration for the Pink runtime is defined using the `construct_runtime!` macro, which constructs the runtime by combining various pallets and system modules.

Some of these modules include:

- `frame_system`: The system module for basic system functionalities.
- `pallet_timestamp`: A timestamp module for managing block timestamps.
- `pallet_balances`: A Balances module for handling token balances and transfers.
- `pallet_insecure_randomness_collective_flip`: Randomness module for generating random numbers.
- `pallet_contracts`: Contracts module for smart contract functionality.
- `pallet_pink`: A custom pallet specific to the Pink Network.
- The runtime struct declaration as `PinkRuntime`.

Constants include:

- `NORMAL_DISPATCH_RATIO`: Specifies the ratio of normal dispatch to operational dispatch. It is set to 75%.
- `BlockHashCount`: Which specifies the number of block hashes to keep in storage.
- `RuntimeBlockWeights`: This defines the block weights configuration, including base weight and normal dispatch ratio.
- `ExistentialDeposit`: Specifies the existential deposit required for an account to exist.
- `MaxLocks`, `MaxReserves`, `MaxHolds`: These define maximum limits for locks, reserves, and holds, respectively.

```rs
impl pallet_pink::Config for PinkRuntime {
    type Currency = Balances;
}
```

This implementation:

- Specifies the configuration for the custom pallet - `pallet_pink`.
- Defines the associated type Currency as `Balances`, indicating that the balances module will be used for currency functionality.

```rs
impl pallet_balances::Config for PinkRuntime {
```

The `pallet_balances` implementation:

- Configures the `pallet_balances` module, which manages token balances and transfers.

Defines various associated types and constants such as the:

- `Balance`: The balance type used in the pink runtime.
- `DustRemoval`: Placeholder type indicating no dust removal functionality.
- `RuntimeEvent`: Type alias for runtime events.
- `ExistentialDeposit`: Specifies the existential deposit required for an account to exist.
- `AccountStore`: Specifies the storage pallet used to store accounts.
- `WeightInfo`: Specifies the weight information provider.
- `MaxLocks`, `MaxReserves`, `MaxHolds`: The maximum limits for locks, reserves, and holds.
- `ReserveIdentifier`, `FreezeIdentifier`: Such identifiers for reserves and freezes.
- `RuntimeHoldReason`, `RuntimeFreezeReason`: Reasons for holding and freezing accounts.
- As an added functionality, it specifies `MaxFreezes` as `()` indicating no maximum freezes.

```rs
impl frame_system::Config for PinkRuntime {
```

The `frame_system` implementation provides runtime system functionalities and also defines various types such as:

- `BaseCallFilter`: Specifies the base call filter.
- `BlockWeights`: Specifies the block weights configuration.
- `Nonce`, `Hash`, `AccountId`: Types for nonce, hash, and account identifiers.
- `Block`, `BlockHashCount`: Types for blocks and block hash count.
- `RuntimeEvent`: Type alias for runtime events.
- `PalletInfo`, `AccountData`: Information about pallets and account data.

```rs
impl pallet_insecure_randomness_collective_flip::Config for PinkRuntime {}
```

This implementation configures the `pallet_insecure_randomness_collective_flip` module, which provides insecure randomness generation.

```rs
parameter_types! {
    pub const MinimumPeriod: u64 = 1;
}
```

Definition of a parameter `uint64` datatype `MinimumPeriod` with a value of 1.

```rs
impl pallet_timestamp::Config for PinkRuntime {
    type Moment = u64;
    type OnTimestampSet = ();
    type MinimumPeriod = MinimumPeriod;
    type WeightInfo = ();
}
```

This implementation configures the `pallet_timestamp` module, which provides timestamp functionalities with some associated type definitions such as:

- `Moment`: Type for representing moments in time, set to `u64`.
- `OnTimestampSet`: Placeholder type indicating no action taken on timestamp set.
- `MinimumPeriod`: Specifies the minimum period between timestamp updates.
- `WeightInfo`: Placeholder type for weight information.

```rs
impl pallet_contracts::Config for PinkRuntime {
```

This implementation configures the `pallet_contracts` module, which enables smart contract functionalities. It also has the associated types below which we will dissect briefly:

- `Time`: Specifies the timestamp type used.
- `Randomness`: Specifies the randomness type used.
- `Currency`: Specifies the currency type used.
- `RuntimeEvent`: Type alias for runtime events.
- `RuntimeCall`: Type alias for runtime calls.
- `CallFilter`: Specifies the call filter.
- `CallStack`: Array type representing the call stack.
- `WeightPrice`: Specifies the weight price.
- `WeightInfo`: Specifies the weight information provider.
- `ChainExtension`: Specifies the chain extension used.
- `Schedule`: Specifies the contract execution schedule.
- `DepositPerByte`, `DepositPerItem`, `DefaultDepositLimit`: Deposit-related parameters.
- `AddressGenerator`: Specifies the address generator.
- `MaxCodeLen`, `MaxStorageKeyLen`, `MaxDebugBufferLen`: Parameters related to code and storage limits.
- `UnsafeUnstableInterface`: Specifies whether unsafe unstable interface is enabled.
- `Migrations`: Specifies the migration types.
- `CodeHashLockupDepositPercent`: Specifies the deposit percent for code hash lockup.
- `MaxDelegateDependencies`: Specifies the maximum delegate dependencies.
- `RuntimeHoldReason`: Specifies the runtime hold reason.
- `Debug`, `Environment`, `Xcm`: Placeholder types for debugging, environment, and cross-chain messaging.

### 7. Contracts instantiation and executions

Let's look into some important type aliases in the `contract.rs` file:

- `EventRecord`: This is an alias for `frame_system::EventRecord` specialized for the PinkRuntime configuration. It represents an event record containing information about runtime events relating to contracts.
- `ContractExecResult`: An alias for the result of executing a contract, containing a balance and event records.
- `ContractInstantiateResult`: An alias for the result of instantiating a contract, containing an account ID, balance, and event records.
- `ContractResult<T>`: Lastly, an alias for a contract result, containing either a result of type `T` wrapped in Result or a dispatch error, along with a balance and event records for the call.

```rs
macro_rules! define_mask_fn {
    ($name: ident, $bits: expr, $typ: ty) => {
```

The `define_mask_fn`, generates a function for masking the lowest bits of a given number. To further understand the logic of this macro, let's break down its components:

- `macro_rules! define_mask_fn { ... }`: This code block defines a macro named `define_mask_fn` using the `macro_rules!` syntax, which allows for pattern matching and generation of code based on whatever pattern is provided.
- `($name: ident, $bits: expr, $typ: ty) => { ... }`: This pattern expects three parameters such as:
- `$name`: The name of the function to be called.
- `$bits`: The number of bits in the type `$typ`.
- `$typ`: The data-type of the input number (e.g `u8`, `u16`, `u32`, `u64`, etc.).

The macro function is defined with the provided name (`$name`) and type signature. Inside the function body:

- The number of leading zeros of the input value (`v`) is calculated to determine the position of the most significant bit.
- Then, the position is adjusted to start from zero.
- Position is then clamped to ensure it's within the specified range `(min_mask_bits to $bits - 1)`.
- A cursor value is calculated by shifting 1 left by the adjusted position.
- Then, a mask is generated by subtracting 1 from the cursor value.
- The input value `v` is masked to set the lowest bits to 1.
- The resulting masked value is then returned.

So, in simple terms:

- Developers can use this `macro` to generate masking functions for different types (`$typ`) and specify the number of bits to mask (`$bits`) along with the minimum number of bits to mask (`min_mask_bits`).

Moving on, the `define_mask_fn` macro is used to create the functions below:

- `mask_low_bits64`: This function handles masking the lowest bits of a `u64` value.
- `mask_low_bits128`: While this other function masks the lowest bits of a `u128` value.

- `mask_deposit`: This calculates the minimum masked value based on the deposit per byte (`deposit_per_byte`) and a constant minimum masked bytes (`MIN_MASKED_BYTES`). The minimum masked value is then used to determine the minimum number of bits to mask. The `mask_low_bits128` function is then called with the deposit and the calculated minimum mask bits, and the result is lastly returned.

- `mask_gas`: This function extracts the ref time component from the given weight and passes it to `mask_low_bits64` to mask the lowest bits. The resulting masked ref time is then used to create a new weight object with zero gas fees, and the new weight is returned.

#### Contract Instantiations

```rs
pub fn instantiate(
    code_hash: Hash,
    input_data: Vec<u8>,
    salt: Vec<u8>,
    mode: ExecutionMode,
    args: TransactionArguments,
) -> ContractInstantiateResult
```

The `instantiate()` function is crucial. It is used to instantiate a contract with the given code hash, input data, salt, execution mode, and transaction arguments. Here's a our breakdown of its implementation:

Parameters:
- `code_hash`: The hash of the code that the contract will execute.
- `input_data`: The input data for the contract.
- `salt`: The salt value used for contract address derivation.
- `mode`: The execution mode, which determines how the contract will be executed.
- `args`: Transaction arguments containing information such as the tx origin, transfer amount, gas limit, storage deposit limit, gas-free flag, and deposit (which is not used in this function).

Execution:
- The `instantiate()` function extracts relevant fields from the `args` parameter as we have covered, such as the origin, transfer amount, gas limit, storage deposit limit, and gas-free flag.
- It then constructs a `Weight` object using the provided gas limit and sets the proof size to `u64::MAX` which is the max value of a `uint64` datatype.
- The contract transaction is then executed using the `contract_tx` function, passing the origin, gas limit, gas-free flag, and a closure that performs the contract instantiation.
- Inside the closure, the `Contracts::bare_instantiate` function is called to perform the actual contract instantiation. It uses the various parameters, including the origin, transfer amount, gas limit, storage deposit limit, code (specified by the code hash), input data, salt, debug information, and event collection strategy.
- The result of the contract instantiation is logged using the `log` crate.
- If the execution mode indicates that coarse-grained gas accounting should be used, the `coarse_grained` function is called to adjust the result before returning it. Otherwise, the result is returned as is - which is a `ContractInstantiateResult` that contains information about the result of the contract instantiation.

```rs
pub fn bare_call(
    address: AccountId,
    input_data: Vec<u8>,
    mode: ExecutionMode,
    tx_args: TransactionArguments,
) -> ContractExecResult {
```

The `bare_call()` function above takes charge for executing a contract call without deploying a new contract instance. It takes the target contract address to execute, input data for the call, execution mode, and transaction arguments as input parameters. Here's a further breakdown of its call trace:

- First, extract relevant transaction arguments like the transaction `origin`, `transfer`, `gas_limit`, `gas_free`, and `storage_deposit_limit` from the provided `tx_args`.
- Convert the provided `gas_limit` into a Weight instance and set the proof size to `u64::MAX`.
- Then, figure out the level of determinism required based on the execution mode (mode). If deterministic execution is required, it sets `Determinism::Enforced`, otherwise `Determinism::Relaxed`.
- Execute the contract transaction by invoking the `contract_tx`'s function, passing the origin, gas limit, gas free flag, and a closure that represents the contract call.
- In the case that the execution is not gas-free, it handles the gas costs by paying for the gas upfront, executing the transaction function, and then refunding any unused gas to the origin account -> caller.
- Lastly, it returns the result of the contract execution.

```rs
fn contract_tx<T>(
    origin: AccountId,
    gas_limit: Weight,
    gas_free: bool,
    tx_fn: impl FnOnce() -> ContractResult<T>,
) -> ContractResult<T> {
```

The `contract_tx` function on the other hand, is an internal helper function used to handle the execution of contract transactions. It takes the origin account, gas limit, gas-free flag, and a closure representing the transaction function. Here's a further breakdown of its call trace:

- In the case the transaction is not gas-free, attempt to pay for the gas upfront using the `PalletPink::pay_for_gas` function.
- Execute the provided transaction function (`tx_fn`).
- If the transaction is not gas-free, calculate the gas consumed and refund any unused gas to the origin account using the `PalletPink::refund_gas` function.
- Return the result of the transaction, including gas consumption, storage deposits, debug messages, and events.

In essence, we can see that the `bare_call()` function uses the `contract_tx` internal function for executing it's logic during a smart contract call as we can see in the code snippet below:

```rs
let result = contract_tx(origin.clone(), gas_limit, gas_free, move || {
    Contracts::bare_call(
```

### 8. Storage

```rs
impl<Backend> Storage<Backend> {
    pub fn new(backend: Backend) -> Self {
        Self { backend }
    }
}
```

The above code snippet implements a constructor for the runtime `Storage` struct. It allows for easy instantiation of the `Storage` struct backend storage for the pink runtime.

```rs
impl<Backend> Storage<Backend>
where
    Backend: StorageBackend<Hashing> + AsTrieBackend<Hashing>,
{
    pub fn execute_with<R>(
        &self,
        exec_context: &ExecContext,
        f: impl FnOnce() -> R,
    ) -> (R, ExecSideEffects, OverlayedChanges<Hashing>) {
```

- Inside the above implementation, the backend storage is retrieved using `as_trie_backend`.
- An `OverlayedChanges` struct is created to track changes made to storage.
- The transaction is started using `overlay.start_transaction()`.
- An `Ext` struct is created with the overlay changes, backend, and no trie root.
- The closure `f` is executed within the context of the externalities using `sp_externalities::set_and_run_with_externalities`.

Hence, within the closure execution:
- The timestamp and block number are set based on the provided execution context.
- System events are reset.
- Then closure `f` is executed, and its result is obtained.
- System events and execution effects are then retrieved using `crate::runtime::get_side_effects`.
- System events are emitted using `maybe_emit_system_event_block`.
- After the closure execution, the transaction is committed using `overlay.commit_transaction()`


```rs
pub fn execute_mut<R>(
        &mut self,
        context: &ExecContext,
        f: impl FnOnce() -> R,
    ) -> (R, ExecSideEffects) {
        let (rv, effects, overlay) = self.execute_with(context, f);
        self.commit_changes(overlay);
        (rv, effects)
    }
```
`execute_mut` function:
This function is similar to `execute_with` but commits the storage changes to the backend. It takes two parameters:
- `context`: A reference to an `ExecContext` object, providing the execution context.
- `f`: A closure that takes no arguments and returns a value of type `R` -> Result.
- It returns a tuple containing the return value of the closure `f` and the execution side effects.
- Inside the method, `execute_with` is called to execute the closure and obtain the result, effects, and overlay changes.
- The overlay changes are then committed to the backend using `commit_changes`.

```rs
pub fn changes_transaction(
        &self,
        changes: OverlayedChanges<Hashing>,
    ) -> (Hash, BackendTransaction<Hashing>) {
        let delta = changes
            .changes()
            .map(|(k, v)| (&k[..], v.value().map(|v| &v[..])));
        let child_delta = changes.children().map(|(changes, info)| {
            (
                info,
                changes.map(|(k, v)| (&k[..], v.value().map(|v| &v[..]))),
            )
        });

        self.backend
            .full_storage_root(delta, child_delta, sp_core::storage::StateVersion::V0)
    }
```

- This function converts overlay changes into a backend transaction.
- It then takes the overlay changes as a parameter.
- Returns a tuple containing the root hash and the backend transaction.
- Inside the method, it maps the overlay changes into a format suitable for the backend transaction.
- Finally, it calls the backend's `full_storage_root` method to obtain the root hash and transaction.

```rs
pub fn commit_changes(&mut self, changes: OverlayedChanges<Hashing>) {
        let (root, transaction) = self.changes_transaction(changes);
        self.backend.commit_transaction(root, transaction)
    }
```

- This function commits the storage changes to the backend.
- First, it takes the overlay changes as a parameter.
- Then, inside the method, it calls `changes_transaction` to obtain the root hash and transaction.
- Lastly, it calls the backend's `commit_transaction` method to commit the changes.

```rs
pub fn get(&self, key: &[u8]) -> Option<Vec<u8>> {
    self.backend
        .as_trie_backend()
        .storage(key)
        .expect("Failed to get storage key")
}
```

- The above function retrieves the storage value of a specified key.
- Then, it takes a key slice as a parameter.
- Afterwards, it returns an `Option<Vec<u8>>`, representing the stored value associated with the `key`, if it is present.
- Inside the function, it delegates the storage retrieval operation to the backend's storage method. If successful, it returns the storage value; otherwise, it returns None.

```rs
pub fn maybe_emit_system_event_block(events: SystemEvents) {
    use pink_capi::v1::ocall::OCalls;
    if events.is_empty() {
        return;
    }
    if !OCallImpl.exec_context().mode.is_transaction() {
        return;
    }
    let body = EventsBlockBody {
        phala_block_number: System::block_number(),
        contract_call_nonce: OCallImpl.contract_call_nonce(),
        entry_contract: OCallImpl.entry_contract(),
        origin: OCallImpl.origin(),
        events,
    };
    let number = PalletPink::take_next_event_block_number();
    let header = EventsBlockHeader {
        parent_hash: PalletPink::last_event_block_hash(),
        number,
        runtime_version: crate::version(),
        body_hash: body.using_encoded(sp_core::hashing::blake2_256).into(),
    };
    let header_hash = sp_core::hashing::blake2_256(&header.encode()).into();
    PalletPink::set_last_event_block_hash(header_hash);
    let block = EventsBlock { header, body };
    OCallImpl.emit_system_event_block(number, block.encode());
}
```

- The above `maybe_emit_system_event_block()` function first checks if the provided events is empty. If it is, the function returns early, indicating that there are no events to emit.
- Next, it checks if the execution mode, obtained from the `OCallImpl`, is a transaction mode. If it's not, the function returns early, as it only emits events during transactions.
- If both conditions are met (i.e., there are non-empty events and the execution mode is a transaction), the function proceeds to create an `EventsBlockBody` instance.

The `EventsBlockBody` contains such various fields:

- `phala_block_number`: The block number obtained from the System pallet.
- `contract_call_nonce`: The contract call nonce obtained from the `OCallImpl`.
- `entry_contract`: The entry contract obtained from the `OCallImpl`.
- `origin`: The origin of the transaction obtained from the `OCallImpl`.
- events: The events that need to be emitted.

The function then generates a new event block number by calling `PalletPink::take_next_event_block_number`.

It constructs an `EventsBlockHeader` instance, containing:

- `parent_hash`: The hash of the last event block obtained from `PalletPink::last_event_block_hash`.
- `number`: The newly generated event block number.
- `runtime_version`: The version of the runtime obtained from `crate::version()`.
- `body_hash`: The hash of the encoded EventsBlockBody.

The function sets the hash of the last event block to the hash of the current header.

Finally, it constructs an `EventsBlock` instance with the header and body, and emits this block using the `OCallImpl.emit_system_event_block` function.


## **5. Codebase Quality**

Here are some of our observations of the `Pink Runtime` codebase's quality.

### Readability:
The runtime is well-formatted and follows consistent naming conventions, which contributes to readability. There are some few comments and documentation are present at various levels of the code, aiding in understanding the purpose and behavior of functions, logic and macros.

The use of descriptive variable and function names makes the code easier to understand.

### Maintainability:
The runtime is organized into modules, structs, and traits, which helps in maintaining a clear structure and separation of concerns.
Macros are used effectively to reduce code duplication and improve maintainability.

### Performance:
The team have tried to adhere to best practices and use efficient algorithmic calculations where applicable. The use of `macros` for repetitive tasks improves the runtime performance by reducing function call overhead.

### Robustness:
Error handling is implemented using Rust's `Result` type, which provides robustness by explicitly handling potential errors. Unsafe code blocks are used judiciously, and precautions are taken to ensure memory safety and prevent undefined behaviors.

## **6. Systematic Risks**

This section of the analysis report covers error handling, slight flaws and major systemic risks of the covered components in the Pink Runtime.

### Entry point: `/capi/mod.rs`

- Null Pointer De-reference: Although, this part of the runtime code checks if `ecalls` is null before usage, there may be scenarios where null pointers are not handled properly elsewhere, leading to potential null pointer de-reference issues.
- How the pink runtime handles erros in `__pink_runtime_init`: While errors are logged, returning a fixed error code `(-1)` might not provide enough context to the caller about the nature of the error. This can be improved by returning different error codes based on the type of error encountered.
- Limited Error Propagation in `Ocall`: Errors encountered during `OCall` setup `(set_ocall_fn)` are logged but not propagated further. Depending on how critical these errors are, it would be beneficial to propagate them up the call stack for higher-level error handling.

### Ocall Implementations: `/capi/ocall_impl.rs`

- The reliance on a default `OCall` function that panics may not be suitable for all scenarios. A more graceful error handling mechanism should be considered.

## Insights

### Entry point: `/capi/mod.rs`

- The initialization routine enforces a proper setup of `OCall` functions and sets up the `ECall` dispatcher mechanism.
- Usage of raw pointers and unsafe blocks is direct interaction with memory, typical of the flexibility low-level systems programming exposes to us.
- Overall, the initialization code is concise handles potential errors gracefully by logging and returning appropriate error codes.

### ECall Implementations: `/capi/ecall_impl.rs`

- The `instrument_context!` macro is used for adding instrumentation to track the execution context, particularly for tracing or logging purposes. It provides visibility into the execution flow within the Pink Runtime.
- The `Executing` trait implementation for `ExternalStorage` enables this struct to be involved in executing functions within the runtime related to storage operations.
- The use of closures `(FnOnce() -> T)` allows for flexible execution of code within the context of the Pink Runtime.
- The implementation of the `ECalls` trait provides a modular interface for interacting with the Pink Runtime, covering various aspects such as cluster setup, contract deployments, key management, and lifecycle events.
- Error handling is quite robust, providing informative error messages and logging capabilities to aid in debugging.
- Lifecycle events such as `on_genesis`, `on_runtime_upgrade`, and `on_idle` ensure proper initialization and maintenance of the Pink Runtime.
- The `sanitize_args` function ensures that the gas limit is within reasonable bounds based on the execution mode. This prevents excessive gas consumption and potential DoS attacks.
- On the other hand, the `handle_deposit` function ensures that deposits specified in transaction arguments are properly handled, ensuring that funds are deposited into the origin account if required and gatekeeps against zero value deposits.

### Ocall Implementations: `/capi/ocall_impl.rs`

- The ocall implementation code provides a mechanism for setting `OCall` functions dynamically, allowing flexibility in handling outgoing enclave calls.
- Optionally, it supports a custom allocator, which can be enabled via the allocator feature. This enables better integration with external memory management systems.
- Test cases ensure correctness and robustness, validating critical functionalities such as OCall handling and memory allocation.

### Runtime configurations: `runtime.rs`

- The Pink Runtime is built using the FRAME framework, utilizing custom pallets and system modules specific to the Pink Network.
- Configuration parameters such as block weights, block hash count, and maximum limits are defined as constants to customize the behavior of the runtime.
- Configurations are done for various pallets used in the Pink Runtime, including custom pallets `(pallet_pink)` and standard `FRAME` pallets `(pallet_balances, frame_system, pallet_insecure_randomness_collective_flip)`.

Overall, the Pink Runtime codebase review was a great one to delve into even though this is our first attempt at a Rust codebase after having studied Rust for a couple days.




### Time spent:
21 hours