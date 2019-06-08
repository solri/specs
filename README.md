# Solri Blockchain Experiment

[![Discord](https://img.shields.io/discord/586902457053872148.svg)](https://discord.gg/DZbg4rZ)

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

The runtime exposes three functions in the module.

### `timestamp`

This exposes the method to fetch timestamp from a block, defined as

```
fn timestamp(block_ptr: i32, block_len: i32) -> i64;
```

After instantiation, the caller should copy raw block binary into
`memory`, of `block_ptr` and `block_len` respectively. The result
should be `-1` if the block is invalid, and a positive value
otherwise.

### `difficulty`

This exposes the method to fetch difficulty from a block, defined as

```
fn difficulty(block_ptr: i32, block_len: i32) -> i64;
```

After instantiation, the caller should copy raw block binary into
`memory`, of `block_ptr` and `block_len` respectively. The result
should be `-1` if the block is invalid, and a positive value
otherwise.

### `execute`

This exposes the method to execute a block for a runtime, assuming
parent block provided and code provided is valid, defined as:

```
fn execute(
  parent_block_ptr: i32, parent_block_len: i32,
  code_ptr: i32, code_len_ptr: i32,
  block_ptr: i32, block_len: i32
) -> i32;
```

After instantiation, the caller should copy raw parent block binary,
and executing block binary into `memory`, and set `parent_block_ptr`,
`parent_block_len`, `block_ptr` and `block_len` respectively. The
caller should also copy the current WebAssembly code being executed
into `memory` where its length should be set in another location
within the `memory`, and set `code_ptr`, `code_len_ptr` respectively.

The runtime can change `code` being executed for the next block by
modifying memory location of `code_ptr` and `code_len_ptr`, which the
caller is responsible to fetch after the call. The result should be
`-1` if execution is invalid, and `0` otherwise.

## Pre-validation and Fork Choice

Before calling runtime's `execute` function for a block. The client is
responsible to verify that `timestamp` (by using the `timestamp`
function in the runtime) is reasonble. The margin is up to each client
implementation.

Fork choice rule is defined as choosing the block with the highest
total difficulty. Notice that although we don't limit what the PoW
algorithm is, the fork choice rule is indeed limited by total
difficulty, to ensure backward compatibility.

## Runtime

The actual `runtime` (which exposes `execute`) function is expected to
be stateless. Notice we don't provide any imports for the execution
environment.

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
  
