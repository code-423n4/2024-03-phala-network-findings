# Summary
* L-1: `js_eval` is not implemented for queries
* L-2: Contract address derivation does not include `input_data`
* R-1: Flush `LimitedWriter` after use

## L-1: `js_eval` is not implemented for queries

According to the documentation ([online](https://docs.rs/pink/latest/pink/chain_extension/trait.PinkExtBackend.html?search=execute#tymethod.js_eval) and in-line [#1](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/capi/src/v1/mod.rs#L328-L331),[#2](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/pink/src/chain_extension.rs#L649-L704)), the [js_eval(...)](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/lib.rs#L376-L378) method should be available in queries (only in queries because it's non-deterministic). However, it's actually not implemented in the `chain-extension`.

```rust
fn js_eval(&self, _codes: Vec<JsCode>, _args: Vec<String>) -> Result<JsValue, Self::Error> {
    Ok(JsValue::Exception("No Js Runtime".into()))
}
```

## L-2: Contract address derivation does not include `input_data`

Pink's [contact address derivation](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/runtime/src/runtime/pallet_pink.rs#L108-L122) overrides the one of `pallet-contracts` as follows.

```rust
fn contract_address(
    deploying_address: &AccountIdOf<T>,
    code_hash: &HashOf<T>,
    _input_data: &[u8],   // @audit _input_data ignored
    salt: &[u8],
) -> AccountIdOf<T> {
    let cluster_id = <ClusterId<T>>::get();
    let buf = phala_types::contract::contract_id_preimage(
        deploying_address.as_ref(),
        code_hash.as_ref(),
        cluster_id.as_ref(),
        salt,
    );
    UncheckedFrom::unchecked_from(<T as frame_system::Config>::Hashing::hash(&buf))
}
```

Although compatibility with [pallet contracts v9](https://docs.rs/pallet-contracts/9.0.0/pallet_contracts/struct.DefaultAddressGenerator.html) is ensured, it's not in line with the [latest](https://docs.rs/pallet-contracts/latest/pallet_contracts/struct.DefaultAddressGenerator.html) address derivation.  

**Consequence:** Multiple instances of the *same* contract with *different* arguments/`input_data` cannot be created as expected failing with a `DuplicateContract` error. It requires to explicitly alter the salt in order to obtain distinct addresses in such a case.


## R-1: Flush `LimitedWriter` after use

The [async_http_request(...)](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/lib.rs#L100-L179) method uses the [LimitedWriter](https://github.com/code-423n4/2024-03-phala-network/blob/a01ffbe992560d8d0f17deadfb9b9a2bed38377e/phala-blockchain/crates/pink/chain-extension/src/lib.rs#L385-L417) to write the `body` chunk by chunk from the HTTP response.

```rust
...

let mut writer = LimitedWriter::new(&mut body, MAX_BODY_SIZE);

while let Some(chunk) = response
    .chunk()
    .await
    .or(Err(HttpRequestError::NetworkError))?
{
    writer
        .write_all(&chunk)
        .or(Err(HttpRequestError::ResponseTooLarge))?;
}

...
```

However, the `LimitedWriter` is never flushed.  

**Recommendation:** Add `writer.flush().or(Err(HttpRequestError::ResponseTooLarge))?;` below the above snipppet to ensure that the writer is correctly flushed.

