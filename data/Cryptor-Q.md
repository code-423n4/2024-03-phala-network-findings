[L01] Ensure_System should be checked after uploading bare code 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L299

Current implementation holds little value as a bad actor can still upload malicious system code since the check is made before any new code is added, posing significant risk to users 



[L02] No point in using regular http request over batchhttperequest 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs#L299-L300

Since the len in batchhttprequest is not restricted to being > 1, a user can just use batchhttperequest over the regular httprequest as the former is less likely to revert due to the timeout bound being longer 



[L03] CallinCommand is misleading and should be renamed callinTransaction for clarity 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L336




[L04] is_running_in_command function is missing or misnamed in the code 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L439


the function is_running_in_command (from the documentation) is missing or misnamed in the code to be 
is_it_in_transaction.



[L05] Functions that are not supported in transaction mode should revert not send empty arrays

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L436

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L473

Sending empty arrays could be misleading to users. It is recommended to revert and send a error that these functions are not support in transaction mode


[L06] Twox64Concat has a higher risk of a hash collision than blake128Concat 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs#L83

It is recommended to use blake128Concat hash for better security and a lower chance of a collision


[L07] Ensure_System should be called for sign and verify

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L174-L181

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L183-L191



If the system contract is maliciously upgraded, then calling sign or verify can lead to unexpected results for users 


[L08] Gas price limited as u128 could cause interoperability issues with evm blockchains 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/contract.rs#L201




[L09] Masking deposit calculation can lead to unexpected results in certain circumstances 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/contract.rs#L122-L135

The function coarse gas relies on complex bit math to mask deposits in order to mitigate side channel attacks. Therefore we can consider that masked deposits must be at least higher or equal to the original deposit. However, this invariant can be broken.

Consider the fuzz test below 

```
#[test]
fn test_mask_deposit_properties(deposit in 1u128..=u128::MAX, deposit_per_byte in any::<u128>()) {
    let masked_deposit = mask_deposit(deposit, deposit_per_byte);

    // Property 1: Masked deposit should be greater than or equal to the original deposit.
    prop_assert!(masked_deposit >= deposit);
}

```

Here the invariant will be broken if deposit_per_byte is = 664613997892457936451903530140172289. This can be problematic if the cluster creator (the actor who sets the deposit per byte) is acting in a malicious way as this opens the door for the side channel attack that the function is supposed to prevent. 





[L10] Not checking the bytecode for system contract can be dangerous as it can be upgraded 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L145-L151

The function ensure system doesn't check for correct bytecode, only the correct address. This can be dangerous as the contract can be maliciously upgraded



[L11] No deadline for signature function 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L174-L181

It is a common security to add a deadline for signatures to prevent stale transactions or prevent them from maliciously executed by workers


[L12] is_in_transaction return value can be misleading 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L439-L441

If a call is in estimating mode, then is_in_transaction will return true since estimating mode is treated the same as transaction, when in reality,
it should be false. There should be a function to return whether the call is in estimating mode instead


[L13] MAX_CODE_LEN should equal the schedule 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime.rs#L114-L133


It makes little sense that the MAX_CODE_LEN (equal to 2 *1024 *1024) is not equal to the schedule payload_len (the max size for storage value equal to 1024 *1024). This means that a code size is free to be bigger than schedule max size but it won't be deployed. 






