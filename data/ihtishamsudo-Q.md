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