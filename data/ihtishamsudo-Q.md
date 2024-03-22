## L-01 No Error Handling in `overlay.commit_transaction()`

[code link](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/storage/mod.rs#L48-L71)

`overlay.commit_transaction()` is expected to always succeed, as indicated by the .expect("BUG: mis-paired transaction"). If it fails, the program will panic. It might be better to handle this error gracefully

- Recommendation

To handle the error, you can return a Result from the function. This allows the caller to decide what to do in case of an error. Mitigated code : 

```diff
pub fn execute_with<R>(
        &self,
        exec_context: &ExecContext,
        f: impl FnOnce() -> R,
-    ) -> (R, ExecSideEffects, OverlayedChanges<Hashing>) {
+    ) -> Result<(R, ExecSideEffects, OverlayedChanges<Hashing>), &'static str> {
        let backend = self.backend.as_trie_backend();

        let mut overlay = OverlayedChanges::default();
        overlay.start_transaction();
        let mut ext = Ext::new(&mut overlay, backend, None);
        let (rv, effects) = sp_externalities::set_and_run_with_externalities(&mut ext, move || {
            Timestamp::set_timestamp(exec_context.now_ms);
            System::set_block_number(exec_context.block_number);
            System::reset_events();
            let result = f();
            let (system_events, effects) = crate::runtime::get_side_effects();
            maybe_emit_system_event_block(system_events);
            (result, effects)
        });
        overlay
            .commit_transaction()
-            .expect("BUG: mis-paired transaction");
-        (rv, effects, overlay)
+        .map_err(|_| "BUG: mis-paired transaction")?;
+         Ok((rv, effects, overlay))
    }
}
```

## L-02 `check_metadata_with_path` silently returns an empty vector if read operation fails 

In the check_metadata_with_path function, the result of `std::fs::read(path)` is unwrapped with unwrap_or_default(). incase of file read operation fails, this will silently return an empty vector. it might be better to handle this error explicitly instead of failing silently.

- Recommendation 

you can use a match statement to handle the Result returned by std::fs::read(path). If the operation is successful, you can proceed as normal. If it fails, you can handle the error in a way provided 

```diff
fn check_metadata_with_path(path: &str) -> Result<(), String> {
    let storage = crate::storage::in_memory_backend::InMemoryStorage::default();
    let context = pink_capi::v1::ocall::ExecContext {
        block_number: 1,
        ..Default::default()
    };
    let metadata: Vec<u8> = storage
        .execute_with(&context, || PinkRuntime::metadata().into())
        .0;
-    let old_metadata = std::fs::read(path).unwrap_or_default();
+    let old_metadata = match std::fs::read(path) {
+        Ok(data) => data,
+        Err(e) => {
+            return Err(format!("Failed to read file: {}", e));
+        },
+    };
    if metadata != old_metadata {
        let new_metadata_path = std::env::current_dir().unwrap().join(format!("{path}.new"));
        std::fs::write(&new_metadata_path, metadata).unwrap();
        return Err(format!(
            "Pink runtime metadata changed. The new metadata is stored at \n {:?}",
            new_metadata_path.display()
        ));
    }
    Ok(())
}
```
## L-03 Use of panic! should be avoided

The [panic!](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/capi/ocall_impl.rs#L15) macro is used to stop execution when a condition is not met. This is useful for testing and prototyping, but should be avoided in production code.

```solidity
unsafe extern "C" fn _default_ocall(
    _call_id: u32,
    _data: *const u8,
    _len: usize,
    _ctx: *mut ::core::ffi::c_void,
    _output: output_fn_t,
) {
    panic!("No ocall function provided");
}
```

- Recommendation 

Using `Result` as return type for functions that can fail is the idiomatic way to handle errors in Rust. The Result type is an enum that can be either `Ok` or `Err`. The Err variant can contain an error message. The ? operator can be used to propagate the error message to the caller.

- [Reference](https://github.com/CoinFabrik/web3-grant/tree/main/vulnerabilities/examples/panic-error)

## L-04 Unsafe Usage of `expect` 

usage of [expect](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/lib.rs#L51) method when creating the Tokio runtime. If the creation of the runtime fails, it will panic with a provided error message, causing the program to crash. This behavior is undesirable, especially in production environments or critical systems.

```solidity
fn block_on<F: core::future::Future>(f: F) -> F::Output {
    match tokio::runtime::Handle::try_current() {
        Ok(handle) => handle.block_on(f),
        Err(_) => tokio::runtime::Runtime::new()
@>          .expect("Failed to create tokio runtime")
            .block_on(f),
    }
}
```

- Recommendation

it's recommended to handle the error case more gracefully, rather than panicking. Instead of using expect, a safer method for error handling should be employed, such as returning a `default value` or propagating the error to the caller for appropriate handling.

try to mitigate all the instances in codebase where expect is used.

## L-05 Use Custom Error Handling Instead of `unwrap`

The function [with_global_cache](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/local_cache.rs#L24) employs the [unwrap](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/local_cache.rs#L35) method when accessing the mutex lock in non-test mode. Using unwrap can result in a panic if the mutex is poisoned, leading to unexpected program termination.

```solidity 
fn with_global_cache<T>(f: impl FnOnce(&mut LocalCache) -> T) -> T {
    if TEST_MODE.load(Ordering::Relaxed) {
        // Unittests are running in multi-threaded env. Let's give per test case a cache instance.
        use std::cell::RefCell;
        thread_local! {
            pub static GLOBAL_CACHE: RefCell<LocalCache> = RefCell::new(LocalCache::new());
        }
        GLOBAL_CACHE.with(move |cache| f(&mut cache.borrow_mut()))
    } else {
        use std::sync::Mutex;
        pub static GLOBAL_CACHE: Mutex<LocalCache> = Mutex::new(LocalCache::new());
@>      f(&mut GLOBAL_CACHE.lock().unwrap())
    }
}
```
- Recommendation

it is advised to handle potential errors by using `custom error` handling instead of unwrap

## L-05 Type Mismatch in `ecall_impl.rs`

The code in [ecall_impl.rs](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/capi/ecall_impl.rs#L46) contains a type mismatch vulnerability, specifically an error of type E0308 in Rust, due to inconsistent types between the expected return type and the actual return value.
indicates that there is a mismatch between the expected return type, which is an enum `std::result::Result`, and the actual return value, which is a tuple `(_, _, _)`

- Recommendation 

it is recommended to modify the pattern matching to correctly match the expected return type.

Try wrapping the pattern in `Ok`

`let Ok((rv, _effects, _)) = self.execute_with(&context, f);`

[Another Instance](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/storage/mod.rs#L84)



## Info-01 The format! macro should not be used 

The problem arises from the use of the format! macro. This is used to format a string with the given arguments. Returning a custom error is desirable.

```solidity
let mut response = match result {
        Ok(response) => response,
        Err(err) => {
            // If there is somthing wrong with the network, we can not inspect the reason too
            // much here. Let it return a non-standard 523 here.
            return Ok(HttpResponse {
                status_code: 523,
                reason_phrase: "Unreachable".into(),
@>              body: format!("{err:?}").into_bytes(),
                headers: vec![],
            });
        }
    };
```
Consider mitgating all the instances in the codebase where the format! macro is used.
- Recommendation

It is recommended to use the `custom error` handling instead of the format! macro. This will allow the caller to handle the error in a more appropriate way.

## Info-02 Use latest version of ink!

Latest version of `ink!` is `5.0.0` and [project](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/pink/Cargo.toml#L10) is using version `4.2` 

- Recommendation

Consider Upgrading ink! version to latest 