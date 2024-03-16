[L01] set_expire has no access control 
A bad actor can maliciously force data to be expired earlier than it should 



[L02] No point in using regular http request over batchhttperequest 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs#L299-L300

Since the len in batchhttprequest is not restricted to being > 1, a user can just use batchhttperequest over the regular httprequest as the former is less likely to revert due to the timeout bound being longer



[L03] CallinCommand is misleading and should be renamed callinTransaction for clarity 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L336




[L04] is_running_in_command function is missing or misnamed in the code 

https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/extension.rs#L439


the function is_running_in_command is missing or misnamed in the code to be 
is_it_in_transaction.



[L05] Functions that are not supported in transaction mode should revert not send empty arrays


[L06] Estimating mode returning coarse gas can make it not determinstic 


[L07] Ensure_System should be called on every function for 


[L08] Gas price limited as u128 could cause interoperability issues with evm blockchains 




