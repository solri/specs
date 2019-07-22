# Solri Blockchain Experiment

[![Discord](https://img.shields.io/discord/586902457053872148.svg)](https://discord.gg/DZbg4rZ)
[![Matrix](https://img.shields.io/matrix/solri:matrix.org.svg)](https://riot.im/app/#/room/#solri:matrix.org)

Solri is a blockchain experiment that tries to bring
[Substrate](https://github.com/paritytech/substrate) style on-chain
governance for Proof of Work. In particular, the design goal is to:

* Become a **pure** coin, in most possible aspects that blockchain
  community cares about. This includes having **no premine** and
  establishes **fair launch**.
* Use well-established **Proof of Work** consensus algorithm, and at
  the same time, allows it to change to either become ASIC-friendly or
  ASIC-resistent.
* Most essential functions of the blockchain is coded into
  **WebAssembly** runtime. This makes it so that the blockchain can
  evolve and upgrade features **without hard fork**. In fact, we aim
  to never conduct any hard fork!
* Have clear and simple **specification** to make supporting multiple
  implementations easier.
* Be **stateless**. To fully verify a new block, you will only need
  the parent block together with the parent block's runtime
  output. This reduces the bare minimal storage requirement for full
  node to nearly zero. In practice, clients can selectively choose to
  only store states it cares about.
  
Before we go on, it is important to note that Solri is currently
entirely a hobby project, and it's really early stage.

## Execution Environment

The runtime is only required to expose one functions in the module.

### `execute`

This exposes the method to execute a block for a runtime, assuming
parent block provided and code provided is valid, defined as:

```
fn execute(
  code_ptr: i32, code_len_ptr: i32,
  block_ptr: i32, block_len: i32,
  result_ptr: i32,
) -> i64;
```

After instantiation, the caller should copy executing block binary
into `memory`, and set `block_ptr` and `block_len` respectively. The
caller should also copy the current WebAssembly code being executed
into `memory` where its length (32 bit) should be set in another
location within the `memory`, and set `code_ptr`, `code_len_ptr`
respectively.

The runtime can change `code` being executed for the next block by
modifying memory location of `code_ptr` and `code_len_ptr`, which the
caller is responsible to fetch after the call. The function returns
`-1` if execution is invalid. Otherwise, the result is `0`.

If the execution is successful, the runtime is expected to return the
necessary metadata for the caller to further consider the validity of
the block and be able to store it in database. The structure should be
set at `result_ptr`, using C representation:

```
#[repr(C)]
struct {
  parent_hash: [u8; 32],
  hash: [u8; 32],
  timestamp: i64,
  difficulty: i64,
}
```

If the execution was invalid, the caller should not assume data under
`result_ptr`.

## Block

The full block is up to interpretation of the runtime. The runtime
should return metadata required as defined above.

## Timestamp Validation and Fork Choice

When mining, it is always expected that the miner uses a client whose
native version supports the current WebAssembly runtime. In this case,
the miner should be able to decode `timestamp` from incoming
blocks. The client is then responsible to verify that `timestamp` is
reasonable. The margin is up to each client implementation. When only
validating blocks, one can use the returned metadata to know the
validity of the block, if native version and WebAssembly runtime
mismatches.

Fork choice rule is defined as choosing the block with the highest
total difficulty. Notice that although we don't limit what the PoW
algorithm is, the fork choice rule is indeed limited by total
difficulty, to ensure backward compatibility.

## Runtime

The actual `runtime` (which exposes `execute`) function is expected to
be stateless. Notice we don't provide any imports for the execution
environment. The initial runtime only supports two operations --
transfer, and runtime upgrade.

One method to do this is to require each transaction to provide merkle
proof for all execution it requires. The current design is as follows.

* Define the state as binary merkle trie, where accounts are addressed
  by their indexes (instead of public keys).
* Provide multi merkle proof for all transactions at the end of the
  block.
* The transaction is also signed with the local merkle proof that it
  requires, but this data can be omitted (and reconstructed) during
  transmission.
* If a transaction misses a local merkle proof, that whole transaction
  fails with out-of-gas.
  
