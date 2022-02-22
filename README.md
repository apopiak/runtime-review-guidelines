# Runtime Review Guidelines
Collection of advice on how to review (and write) Substrate based runtimes.

## General / Misc
- Fail fast (to reduce computation done on-chain).
- Verify any assumptions about the rest of the runtime made in your custom pallets and logic.
- Ensure math is explicit about how to handle overflows. (Using e.g. `checked_add` or `saturating_add` instead of `+` , alternatively adding comments why a bare `+` is safe.)
- Avoid panics (e.g. `unwrap` ) at almost any cost and add "proofs" for `expect()` calls on why they cannot fail. The only reason to panic should be in case a block should not be built when that code path is triggered.
- When reviewing runtime tests ensure that there's at least one `assume_noop!` for each place in the code that returns an error.

## Permissioning / Origins
- Make sure the origin is being checked (either none, root, signed or custom origin) even if the sender is not important.
  - `None` for unsigned transactions.
  - `Root` for privileged operation only executable by "the chain itself".
  - `Signed(account)` for signed transactions by "regular users" of the chain.
  - custom origin when you want to combine the above or have custom origin logic (e.g. for extrinsics dispatched via XCM).

## Runtime Storage
- Pay attention to efficient usage of storage (avoid duplicate keys, be aware of the trade offs of how often you want to read/write something).
- Make sure to keep atomicity in basically all cases. It's incredibly rare to want a non-atomic extrinsic. Ensure writes happen after reads/checks ("verify first, write last"). Alternatively: use `with_transaction` or `#[transactional]` wrapper to roll back unwanted changes.
- Make sure that hashers in storage are appropriate.
    - `blake2_concat` as secure default
    - `twox_concat` for cheaper hashing, keys need to be safe (i.e. not user-controlled; e.g. continuous counter controlled by the chain)
    - `identity` is cheapest, only use for already random (i.e. usually hashed) data (e.g. the hash of a utxo used as a key to store it)

## Migrations & Stability

### Within Pallets
- Watch out for anything that changes the storage layout -- sometimes it is more subtle than simply adding or removing a field to `decl_storage`/`#[pallet::storage]`. Reasons the key or value might change:
  - Key: changing the name. Use [`#[pallet::storage_prefix]` renaming](https://paritytech.github.io/substrate/monthly-2022-01/frame_support/attr.pallet.html#storage-palletstorage-optional) to avoid the underlying key changing.
  - Key: changing the hasher (e.g. `Blake2_128Concat` to `Twox64Concat`)
  - Key/Value: changing a type (that serializes differently) (e.g. `type MyKey = u16` to `type MyKey = u32`)
  - Value: adding a field to a struct (e.g. `MyStruct(u8)` to `MyStruct(u8, u8)`)
- Avoid depending directly on other pallets and adhere to separation of concerns.
- Don't reorder the `Call` enum. If you do, you need to bump the transaction version in the chains using the pallet.

### Within the Runtime
- Set explicit indices in `construct_runtime` and avoid reordering the pallets as it will:
    - change the `Call` indices (in case indices were not set explicitly) necessitating a transaction version bump.
    - change the execution order of hooks (e.g. `on_initialize`) which can lead to subtle errors.

## Offchain Workers

- Triple check every `validate_unsigned`, `pre_dispatch` or custom `SignedExtension` to make sure it doesn't open up a DoS vector.
- If there are any Offchain Workers, make sure the results they generate are validated on-chain (and not assumed to be valid).

## Weights

- Limit the size of dynamic data passed into your dispatchables (e.g. `Vec`s) via (constant) limits or economics.
- Check that `on_finalize` weight is added to `on_initialize` weight (and both are determined correctly). Also keep `on_finalize` as cheap as possible. Consider using `on_idle` for things you don't want to happen at the start of the block.
- Benchmarks used to determine the weight need to measure the worst case. This can mean covering all relevant branches with a benchmark each.
