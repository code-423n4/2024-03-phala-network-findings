## [L-01]  Unused Function `on_genesis()`


```rust

190 pub fn on_genesis() {
    <AllPalletsWithSystem as frame_support::traits::OnGenesis>::on_genesis();
}

```
https://github.com/code-423n4/2024-03-phala-network/blob/main/phala-blockchain/crates/pink/runtime/src/runtime.rs#L190C1-L192C2

 ###  Impact
 
The `on_genesis()` function is defined but appears to be unused within the PinkRuntime codebase. While this does not pose an immediate security risk, it indicates potential dead code or unused functionality, which may lead to confusion for developers maintaining or extending the codebase. Additionally, it may contribute to increased binary size if included in the final build without serving any purpose.

### Recommended

**Remove Unused Functionality**: If `on_genesis()` is no longer needed or intended for future use, consider removing it from the codebase to reduce complexity and improve code maintainability.

**Documentation**: Document the rationale behind the removal of `on_genesis()` or any other unused functionality to ensure clarity for future developers.