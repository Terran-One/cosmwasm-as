# CosmWasm smart contracts in AssemblyScript

<div align="center">

![image](./banner.png)

</div>

<!-- TOC -->
* [CosmWasm in AssemblyScript](#cosmwasm-in-assemblyscript)
  * [Quickstart](#quickstart)
  * [Project Structure](#project-structure)
  * [Architecture](#architecture)
    * [Wasm Imports](#wasm-imports)
    * [Wasm Exports](#wasm-exports)
        * [Required](#required)
      * [Optional](#optional)
  * [Copyright](#copyright)
<!-- TOC -->

**NOTE: This is purely for study and experimentation. Confio has expressed doubt regarding AssemblyScript's viability as a serious CosmWasm language due to concerns about security.**

This repository contains a sample implementation of several CosmWasm smart contracts written in AssemblyScript. We test the behavior of our contracts against [`cosmwasm-vm-js`](https://github.com/terran-one/cosmwasm-vm-js), a JavaScript-based runtime for CosmWasm that is convenient to instrument and run locally.

### Uploaded Contracts

The AssemblyScript contract shown in this repository are uploaded to the following addresses:

- Terra Testnet - `pisco-1`: *(pending upload)*
- Juno Testnet - `uni-5`: `juno13qfr40ewq0ng63ukgxm4xxue6uv4u5d65xpqes3srpq39jjux4hqeqg484`


## Quickstart

1. First, clone the repository.

```bash
$ git clone https://github.com/terran-one/cosmwasm-as
$ cd cosmwasm-as
```

2. Install the dependencies. We used `yarn`, but you can use `npm` as well.
```bash
$ yarn
```

3. Run `asbuild` to build the AssemblyScript Wasm binaries.

**NOTE:** This compiles using AssemblyScript, then rewrites it using `util/rewrite-wasm.js`, which uses Binaryen.

```bash
$ yarn build
```

4. Run the tests.

```bash
$ yarn test
```

## Project Structure

This project was created with the [`asbuild`](https://github.com/AssemblyScript/asbuild) tool and follows a directory organization similar to other AssemblyScript projects.

```text
cosmwasm-as
    ├── assembly/ -- AssemblyScript source root
    │   ├── cosmwasm/ -- CosmWasm APIs
    │   │   ├── cw-storage-plus.ts -- `cw-storage-plus` analog
    │   │   ├── exports.ts -- defines expected CosmWasm exported members
    │   │   ├── imports.ts -- defines expected CosmWasm imports supplied by VM
    │   │   ├── types.ts -- CosmWasm definitions
    │   ├── src/ -- Contract implementation
    │   │   ├── contract.ts -- `contract.rs` analog
    │   │   ├── msg.ts -- `msg.rs` analog
    │   │   └── state.ts -- `state.rs` analog
    │   └── index.ts -- directs compiler on assembling Wasm module -- `lib.rs` analog
    │── util/ -- Various build utilities
    │   └── rewrite-wasm.js -- Tool for rewriting Wasm binaries to work with CosmWasm
    ├── build/
    │   ├── debug.wasm -- Wasm binary: with debug symbols
    │   └── release.wasm -- Wasm binary: production-optimized
    └── tests/
        └── works.test.js -- Simple acceptance test
```

## Architecture

A *CosmWasm contract* is a Wasm module that adheres to the following structure[^1].

[^1]: Discussed in further detail on the [CosmWasm official repository README](https://github.com/CosmWasm/cosmwasm/blob/007fd626c67945fc548a99b6ba06aefcd0bb4195/README.md)

### Wasm Imports

<details><summary>Imports provided by CosmWasm VM</summary>


```rust
extern "C" {
    #[cfg(feature = "abort")]
    fn abort(source_ptr: u32);

    fn db_read(key: u32) -> u32;
    fn db_write(key: u32, value: u32);
    fn db_remove(key: u32);

    #[cfg(feature = "iterator")]
    fn db_scan(start_ptr: u32, end_ptr: u32, order: i32) -> u32;
    #[cfg(feature = "iterator")]
    fn db_next(iterator_id: u32) -> u32;

    fn addr_validate(source_ptr: u32) -> u32;
    fn addr_canonicalize(source_ptr: u32, destination_ptr: u32) -> u32;
    fn addr_humanize(source_ptr: u32, destination_ptr: u32) -> u32;

    fn secp256k1_verify(message_hash_ptr: u32, signature_ptr: u32, public_key_ptr: u32) -> u32;
    fn secp256k1_recover_pubkey(
        message_hash_ptr: u32,
        signature_ptr: u32,
        recovery_param: u32,
    ) -> u64;

    fn ed25519_verify(message_ptr: u32, signature_ptr: u32, public_key_ptr: u32) -> u32;
    fn ed25519_batch_verify(messages_ptr: u32, signatures_ptr: u32, public_keys_ptr: u32) -> u32;

    fn debug(source_ptr: u32);

    fn query_chain(request: u32) -> u32;
}
```

</details>

We declare them inside `assembly/cosmwasm/imports` and use where needed inside our library code.
Note that since these are quite low level, the end-user consuming the `cosmwasm-as` API probably won't need to import them directly.

### Wasm Exports


<details><summary>Exports expected by CosmWasm VM</summary>


##### Required

```rust
extern "C" {
	fn allocate(size: usize) -> u32;
	fn deallocate(pointer: u32);
	fn instantiate(env_ptr: u32, info_ptr: u32, msg_ptr: u32) -> u32;
	fn interface_version_8() -> ();
}
```


#### Optional

```rust
extern "C" {
	fn execute(env_ptr: u32, info_ptr: u32, msg_ptr: u32) -> u32;
	fn query(env_ptr: u32, msg_ptr: u32) -> u32;

	// TODO: the following have yet to be implemented
	fn migrate(env_ptr: u32, msg_ptr: u32) -> u32;
	fn reply(env_ptr: u32, msg_ptr: u32) -> u32;
	fn sudo(env_ptr: u32, msg_ptr: u32) -> u32;
	fn ibc_channel_open(env_ptr: u32, msg_ptr: u32) -> u32;
	fn ibc_channel_connect(env_ptr: u32, msg_ptr: u32) -> u32;
	fn ibc_channel_close(env_ptr: u32, msg_ptr: u32) -> u32;
	fn ibc_packet_receive(env_ptr: u32, msg_ptr: u32) -> u32;
	fn ibc_packet_ack(env_ptr: u32, msg_ptr: u32) -> u32;
	fn ibc_packet_timeout(env_ptr: u32, msg_ptr: u32) -> u32;
}
```


</details>

This is the main "meat" that is relevant to our implementation, which must get explicitly exported by `assembly/index.ts` to get picked up by the AssemblyScript compiler.
Their implementation resides in `assembly/cosmwasm/exports.ts` -- we simply re-export them in our `assembly/index.ts`:


```ts
// assembly/index.ts

// Required Wasm exports
export {
  interface_version_8,
  instantiate,
  allocate,
  deallocate
} from './cosmwasm/exports';

// Optional Wasm exports
export {
  execute,
  query,
  ...
} from './cosmwasm/exports';
```

### Build Changes

We altered the build script slightly to make it work with CosmWasm.

#### Step 1: Compile AssemblyScript to Wasm

```diff
asc assembly/index.ts
	--target debug
	--sourceMap
	--debug
+	--disable bulk-memory
+	--use abort=assembly/index/logAndCrash
+	--runtime stub
+	--exportStart
```

A couple changes here:

##### 1. `--disable bulk-memory`

If not set, Wasmer will complain about missing 0xFC opcode.

##### 2. `--use abort=assembly/index/logAndCrash`

AssemblyScript usually requires the host environment to supply `env`.`abort`.
However, CosmWasm supplies a different function with the same name, so we rewire it according to [Simon Warta's example](https://github.com/CosmWasm/cosmwasm/blob/1a356a249c7f0fc655c9070776775a765ab7da2f/contracts/assemblyscript-poc/contract/src/cosmwasm-std/cosmwasm.ts#L106-L126).

##### 3. `--runtime stub`

To disable garbage collection (GC).

##### 4. `--exportStart`

Export the `(start ...)` instruction rather than have it being implicity called.

#### Step 2: Rewrite binary

We authored a tool using [`Binaryen`](https://github.com/bytecode-alliance/binaryen) to remove the `(start ...)`  instruction and replace it by prepending it to each entrypoint's function body.

```bash
node util/rewrite-wasm [--optimize=1] build/debug.wasm
```



## Copyright

Copyright &copy; 2022 Terran One LLC

