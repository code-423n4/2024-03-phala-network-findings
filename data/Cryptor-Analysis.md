## Overview:

Phala network is an extension of the ink framework that adds additional functionality to ink contracts which include http request, evm cross chain functionality and ecdsa signing 



## Key Concepts:

There are 3 different execution modes in the pink! runtime. The differences are as follows:

| Execution Mode | Return Coarse Gas | Gas Limit | Determinism |
| --- | --- | --- | --- |
| Query | True | x*10 | False |
| Transaction  | False | x/2 | True |
| Estimating | True | x/2 | True |

There are also several Signature types to take note of 

| Signature Type  | Determinism  |
| --- | --- |
| ecdsa  | True |
| ed25519 | True  |
| sr25519 | False  |

Contract Flow:
![Chart](https://cdn.discordapp.com/attachments/1165340854853042200/1220822860357042296/image.png?ex=661056e3&is=65fde1e3&hm=db8252c2d38b743cbbd3009872eee7c43327ea4e1c05131fbdb59547df258526&)

https://cdn.discordapp.com/attachments/1165340854853042200/1220822860357042296/image.png?ex=661056e3&is=65fde1e3&hm=db8252c2d38b743cbbd3009872eee7c43327ea4e1c05131fbdb59547df258526&

## Files Overview:

This is an overview of areas of concern/ weakness throughout the coadebase where there could be vulnerabilities

## **[pink/runtime/src/runtime.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime.rs)**

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime.rs#L114-L133

```jsx
const MAX_CODE_LEN: u32 = 2 * 1024 * 1024;

parameter_types! {
    pub DepositPerStorageByte: Balance = Pink::deposit_per_byte();
    pub DepositPerStorageItem: Balance = Pink::deposit_per_item();
    pub const DefaultDepositLimit: Balance = Balance::max_value();
    pub const MaxCodeLen: u32 = MAX_CODE_LEN;
    pub const MaxStorageKeyLen: u32 = 128;
    pub const MaxDebugBufferLen: u32 = 128 * 1024;
    pub DefaultSchedule: Schedule<PinkRuntime> = {
        let mut schedule = Schedule::<PinkRuntime>::default();
        const MB: u32 = 16;  // 64KiB * 16
        // Each concurrent query would create a VM instance to serve it. We couldn't
        // allocate too much here.
        schedule.limits.memory_pages = 4 * MB;
        schedule.instruction_weights.base = 8000;
        schedule.limits.runtime_memory = 2048 * 1024 * 1024; // For unittests
        schedule.limits.payload_len = 1024 * 1024; // Max size for storage value
        schedule
    };
```

Areas of concern:

Why aren’t the MAX CODE LEN and the schedule payload_len equal to each other?

---

## **[pink/runtime/src/contract.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/contract.rs)**

This file is the contracts call/instantiation API

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/contract.rs#L77-L88

```jsx
fn mask_deposit(deposit: u128, deposit_per_byte: u128) -> u128 {
    const MIN_MASKED_BYTES: u128 = 256;
    let min_masked_value = deposit_per_byte
        .saturating_mul(MIN_MASKED_BYTES)
        .saturating_sub(1);
    let min_mask_bits = 128 - min_masked_value.leading_zeros();
    mask_low_bits128(deposit, min_mask_bits)
}

fn mask_gas(weight: Weight) -> Weight {
    Weight::from_parts(mask_low_bits64(weight.ref_time(), 28), 0)
}
```

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/contract.rs#L251-L256

```jsx
 if !gas_free {
        let refund = gas_limit
            .checked_sub(&result.gas_consumed)
            .expect("BUG: consumed gas more than the gas limit");
        PalletPink::refund_gas(&origin, refund).expect("BUG: failed to refund gas");
    }
```

Key Questions:

Are deposits and gas correctly masked with no possibility of dirty bits?

Is gas correctly calculated throughout all execution modes?

---

---

## **[pink/runtime/src/storage/external_backend.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/storage/external_backend.rs)**

The file is the storage provider for the runtime

Areas of Interest:

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/storage/external_backend.rs#L55-L59

```jsx
   /// Check the existence of a particular ink code in the storage.
    pub fn code_exists(code_hash: &Hash) -> bool {
        let key = code_owner_key(code_hash);
        super::ExternalStorage::instantiate().get(&key).is_some()
    }

    fn code_owner_key(code_hash: &Hash) -> Vec<u8> {
        let mut key = Vec::new();
        key.extend(twox_128("Contracts".as_bytes()));
        key.extend(twox_128("CodeInfoOf".as_bytes()));
        key.extend(&code_hash.encode());
        key
    }
```

Key Questions:

Is there any way that code_exist doesn’t return valid ink code in the storage?

Is there any advantage or disadvantage to using twox_128 hash over over hashes?

---

**[pink/runtime/src/runtime/pallet_pink.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs)**

This pallet is used to store custom configuration of the runtime 

Areas of Interest:

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs#L125-L128

```jsx
    impl<T: Config> Pallet<T> {
        pub fn set_cluster_id(cluster_id: Hash) {
            <ClusterId<T>>::put(cluster_id);
        }
```

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs#L134-L146

```
    pub fn put_sidevm_code(
        owner: T::AccountId,
        code: Vec<u8>,
    ) -> Result<T::Hash, DispatchError> {
        let hash = T::Hashing::hash(&code);
        let bytes = code.len() + hash.as_ref().len();
        let fee = Self::deposit_per_byte()
            .saturating_mul(BalanceOf::<T>::saturated_from(bytes))
            .saturating_add(Self::deposit_per_item());
        Self::pay(&owner, fee)?;
        <SidevmCodes<T>>::insert(hash, WasmCode { owner, code });
        Ok(hash)
    }

```

Key Questions:

Is there any code size limit for the sideevm code?

Can fn put_sidevm_code fee overflow?

Is there or should there be any input validation for the side evm code?

---

## **[pink/runtime/src/capi/mod.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/mod.rs)**

This file shows all of the cross host-lib function calls (ecalls and ocalls)

Areas of Interest:

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/capi/src/v1/mod.rs#L320-L322

```jsx
  /// Gets the nonce of the current contract call (if available).
        #[xcall(id = 17)]
        fn contract_call_nonce(&self) -> Option<Vec<u8>>;
```

Key Questions:

Where is the implementation of the call contract nonce?

---

## **[pink/runtime/src/capi/ecall_impl.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs)**

**The enterward cross boundary call support. This is to be called by the worker** 

Areas of concern:

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs#L59-L62

```jsx
impl ecall::ECalls for ECallImpl {
    fn cluster_id(&self) -> Hash {
        PalletPink::cluster_id()
    }
    fn setup(&mut self, config: ClusterSetupConfig) -> Result<(), String> {
        on_genesis();
        let ClusterSetupConfig {
            cluster_id,
```

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs#L99

```jsx
 let result = crate::contract::instantiate(
            code_hash,
            selector,
            vec![],
            ExecutionMode::Transaction,
            args,
        );
```

Key Questions:

Is cluster_id hash transparent enough to be used here?

Can a contract be instantiated in estimating mode?

---

---

## **[pink/runtime/src/runtime/extension.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs)**

The chain extension implementation of the runtime. Users are expected to use this implemenation to interact with the runtime engine. This is considered to be the most important in scope file of the audit 

Areas of concern:

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L361-L371

```jsx
 fn sign(
        &self,
        sigtype: SigType,
        key: Cow<[u8]>,
        message: Cow<[u8]>,
    ) -> Result<Vec<u8>, Self::Error> {
        if matches!(sigtype, SigType::Sr25519) {
            return Ok(vec![]);
        }
        self.as_in_query.sign(sigtype, key, message)
    }
```

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L295-L314

```jsx
fn import_latest_system_code(
        &self,
        payer: ext::AccountId,
    ) -> Result<Option<Hash>, Self::Error> {
        self.ensure_system()?;
        let system_code = OCallImpl.latest_system_code();
        if system_code.is_empty() {
            return Ok(None);
        }
        let code_hash = sp_core::blake2_256(&system_code);
        if !self.code_exists(code_hash, false)? {
            crate::runtime::Contracts::bare_upload_code(
                payer.convert_to(),
                system_code,
                None,
                pallet_contracts::Determinism::Enforced,
            )?;
        };
        Ok(Some(code_hash))
    }
```

Key Questions:

We have CallinQuery and CallinCommand to represent both Query and Execution modes. But where is the implementation for the estimating mode?

The fn ensure_system is not called when executing any function within **PinkExtBackend block.**

What would happen if the system contract address is upgraded?

Is the determinism for each function (Enforced, Relaxed), correct?

---

---

---

## **[pink/chain-extension/src/lib.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/chain-extension/src/lib.rs)**

This is the library for the chain extension. This is the interface for all of the function that are expected to be called by users utilizing the phat contract 

Areas of concern:

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/lib.rs#L66

```jsx
pub fn batch_http_request(requests: Vec<HttpRequest>, timeout_ms: u64) -> ext::BatchHttpResult {
    const MAX_CONCURRENT_REQUESTS: usize = 5;
    if requests.len() > MAX_CONCURRENT_REQUESTS {
        return Err(ext::HttpRequestError::TooManyRequests);
    }
    block_on(async move {
        let futs = requests
            .into_iter()
            .map(|request| async_http_request(request, timeout_ms));
        tokio::time::timeout(
            Duration::from_millis(timeout_ms + 200),
            futures::future::join_all(futs),
        )
        .await
    })
    .or(Err(ext::HttpRequestError::Timeout))
}
```

```jsx
 fn verify(
        &self,
        sigtype: SigType,
        pubkey: Cow<[u8]>,
        message: Cow<[u8]>,
        signature: Cow<[u8]>,
    ) -> Result<bool, Self::Error> {
        macro_rules! verify_with {
            ($sigtype:ident) => {{
                let pubkey = sp_core::$sigtype::Public::from_slice(&pubkey)
                    .map_err(|_| "Invalid public key")?;
                let signature = sp_core::$sigtype::Signature::from_slice(&signature)
                    .ok_or("Invalid signature")?;
                Ok(sp_core::$sigtype::Pair::verify(
                    &signature, message, &pubkey,
                ))
            }};
        }
        match sigtype {
            SigType::Sr25519 => verify_with!(sr25519),
            SigType::Ed25519 => verify_with!(ed25519),
            SigType::Ecdsa => verify_with!(ecdsa),
        }
    }

```

Key Question:

How can the determinism of execution and query modes impact various function calls?

Is there any situation where signatures cannot be verified?

Can timeouts be prevented?

Why are the timeouts for batch http requests and regular http requests different?

---

## **[pink/chain-extension/src/local_cache.rs](https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/chain-extension/src/local_cache.rs)**

This files manages the local cache of the worker. When data is added to the cache, they have an expiration data. When data is expired or the cache size becomes too big, the deleted data is deleted in favor of new data

Areas of concern: 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/local_cache.rs#L57-L79

```jsx
  /// Remove the items closest to the expiration date until the size can be fit into `max_size`.
    fn fit_size(&mut self) {
        if self.size <= self.max_size {
            return;
        }
        let map = std::mem::take(&mut self.kvs);

        let mut kvs: Vec<_> = map
            .into_iter()
            .map(|(k, v)| (v.expire_at, (k, v)))
            .collect();
        kvs.sort_by_key(|(expire, _)| *expire);
        self.kvs = kvs
            .into_iter()
            .filter_map(|(_, (k, v))| {
                if self.size <= self.max_size {
                    return Some((k, v));
                }
                self.size -= k.len() + v.value.len();
                None
            })
            .collect();
    }
```

Key Questions

Is there a way for users to malicious force cache to expire?

What would happen if the cache becomes too big but data is not expired yet?

---

## **Codebase Quality**

| Codebase Quality Categories | Comments |
| --- | --- |
| Documentation | The documentation for the project overall is great. However, it is important to note that some sections could have been explained better such as how interoperability with the evm works. Also, it would have been better to see a diagram regarding the typical user flow. One the main issues I had when auditing this protocol is the amount of relevant information in files that were out of scope. |
| Code Comments | The code comments in general, was good. Complex functions were well explained. It would have been helpful to get a more detailed but higher level overview of what each file does. Like in the previous section, it would have been helpful to get better context about each function without having to ask the sponsor or look in out of scope files |
| Tests | The tests overall was good. It is important to note that many tests were located in different repos so in the context of the contest repo, there were some integration tests that was lacking. It also would have been helpful to see some fuzzing and invariant testing, especially for some of the more complex functions of the protocol. However, there were plenty of unit tests found which demonstrates a high quality codebase. |
| Organization | Codebase is well organized, with in scope files being very easy to locate. |

## Systemic Risks:

Systemic Risks: At the core of the Phala Network protocol, are the system contracts. Owners are deemed to be trusted, however a malicious owner can do irrepariable damage to users. This is in large part to the fact that there is no timelock on the any admin functions. This means that a malicious admin has unexpectedly change many of the parameters of the runtime engine causing issues such as making users pay too much gas, making users pay high fees to the treasury account, to reverting valid transactions.

There is also an issue regarding the 

## Observations/Learnings:

I noticed that several functions that can be called from different execution modes. but considering that there is a difference in gas limit in both the query and transaction modes, it makes no sense to call certain functions in transaction mode for example, as the gas limit is higher, so there is a lower chance of that function reverting.

Many of the high level contracts are out of scope. This made it tricky to track user flow  contract flow and  decipher valid bugs as there was very little information presented in the in scope files about who can call each function. One example of this is the apply quotas function

```jsx
 pub fn apply_quotas<'a>(&mut self, quotas: impl IntoIterator<Item = (&'a [u8], usize)>) {
        for (contract, max_size) in quotas.into_iter() {
            log::trace!(
                "Applying cache quotas for {} max_size={max_size}",
                hex_fmt::HexFmt(contract)
            ); 
```

which on the surface, is supposed to be called by the worker but it has no access control on first glance. However ,after confirming with the sponsor, the implementation is used in a file that is out of scope 

https://github.com/Phala-Network/phala-blockchain/blob/e6260c6e72daea38796c00fb09728c5725caf5b6/crates/phactory/src/contracts/support/keeper.rs#L67

```jsx
        local_cache::apply_quotas(calc_cache_quotas(&self.contracts));
```

One particular function that caught my eye was the calculations for coarse gas.

 It was very complicated so I decided to try fuzz testing it using proptest with several invariants which is something that I have never done before. As a result, I noticed that those invariants that I had tested, failed in several circumstances. 

## Weak Points:

Compatibility with EVM blockchains: The Phala network expects to be interoperable with evm blockchains where phat contracts can sent transactions to evm blockchains and vice versa. However, with this comes risk. Substrate blockchains are not the same as evm blockchains and thus compatibility should be considered an area of high concern as there could be overflows. Some specific concerns include the parameter type in substrate vs those in evm blockchains e.g. if block.timestamp is a u32 on the evm, is it the same on substrate? 

Calling functions with not enough or too much determinism: Determinism is important for the maintain security and trust on the blockchain. Phala Network has 3 execution modes: estimating, query, and transaction. Estimating and Transaction modes are designed to be deterministic while query mode is not. It is important to make sure that determinism is enforced and relaxed at the right times or else the security and trust in the blockchain could be compromised. 

Accurate gas calculations: Gas calculation is important and an area of high concern. It is important for the runtime to correctly charge the right amount of gas for users. However, since the protocol has a several execution modes, with 2 different gas calculations as well as the introduction of coarse gas to mitigate side channel attacks, there is a possibility that gas can be miscalculated in some scenarios

Access Control: Access control is an area of high concern. If a user is able to manipulate sensitive areas such as cache management or manipulate gas fees, then user will be severely impacted. It is important that only trusted users are able to call certain function and that malicious actors are not able to harm innocent users of the phat contract 

DOS attacks: DOS is an area of high concern, as there shouldn’t be a way for bad actors to DOS other users by manipulating the cache management in the local cache file or manipulate gas management to force any OOG errors 

Worker trustworthyness: The worker is an intergral part of the protocol that is responsible for sending transaction. If they are not trusted actors, then they can cause significant damage to the integrity of the protocol. For example, the worker is trust to manage the cache management in the local cache file. If the worker is not trusted then they can manipulate the cache to remove an honest user’s data. 

## Methodology:

My methodology was to first review the documentation at https://docs.phala.network/developers/build-on-phat-contract/use-pink-extension as well as look at the docs for substrate https://docs.substrate.io/maintain/runtime-upgrades/. This allowed me to set up some systemic invariants and look at the code for any systemic inconsistencies with the docs. After reviewing the codebase and asking the sponsors a few questions, I proceeded to take a look at some of the higher level files, taking note of several invariants laid out in the previous section of the analysis. After that I proceeded to take a look at the tests and then edited a few of them to test certain properties. I also took a look at some of the real world examples found here 

https://github.com/Phala-Network/phat-dashboard/tree/master/phat/contracts

https://github.com/Phala-Network/index-contract/tree/main/contracts

 This step was crucial as it gave me additional context for several functions with the protocol.

Additionally, I engaged with the development team to fill in any knowledge gaps and get their insights on certain discoveries that I found in the code. This collaborative approach helped in understanding the rationale behind specific design choices.

Finally, I compiled my findings on the QA report, Analysis and any M/H findings that I discovered.

## Conclusion:

Overall, I enjoyed my time on this audit, I further enchanced my knowledge regarding rust, substrate and ink. I do however, wonder whether the protocol was underscoped, as many of the high level functionality was outside the audit scope. Still, it was a very rewarding experience for me as I have not audited a runtime engine before, so it I had to think about what kind of bugs to look for.

Time Spent: 100 hours

### Time spent:
60 hours