# Solri's Runtime

In this section, we define Solri's execution environment and initial
runtime.

## Execution Environment

The following section defines how the execution environment for Solri
works.

### Memory Management

A WebAssembly instance for Solri is expected to handle its own memory
management. The instance is expected to be long-standing, and might
handle executing multiple blocks.

First, the guest is expected to export its linear memory. This is to
make it possible that host can read slice of data (blocks, code, and
metadata) from guest. The host will not attempt to allocate any
memory, nor handle any other memory management functionality for the
guest.

The guest should also exports the following functions:

* `fn write_code(len: i32) -> i32`: indicates that the host wants to
  write the `code` parameter prior of calling `execute` function. The
  return value should be the address in guest where host can write the
  data slice.
* `fn write_block(len: i32) -> i32`: indicates that the host wants to
  write the `block` parameter prior of calling `execute` function. The
  return value should be the address in guest where host can write the
  data slice.
* `fn read_metadata() -> i32`: available after `execute` is
  called. This indicates that the host can read the metadata result of
  the block at the given address.
* `fn free()`: frees wrote code, block parameters, and metadata
  result, if any.

### Block Execution

This section defines the `execute` export, which exposes the method to
execute a block for a runtime, assuming parent block provided and code
provided is valid, defined as:

```
fn execute() -> i32;
```

Prior of calling this function, the host should first call
`write_code` and `write_block` to feed in the code and block data
slice. If the values are not available, it is an invalid call and the
guest should trap.

The runtime can change `code` being executed for the next block by
modifying memory location and pass the result values in metadata. The
function returns `-1` if the block is deemed invalid. Otherwise, the
result is `0`.

If the runtime cannot execute on the current WebAssembly executor, it
should trap (for example, by using the `unreachable` opcode).

If the execution is successful, the runtime is expected to return the
necessary metadata for the host to further consider the validity of
the block and be able to store it in database. The structure should be
set at `result_ptr`, using C representation:

```
#[repr(C)]
struct Metadata {
  timestamp: i64,
  difficulty: i64,
  
  parent_hash_ptr: i32,
  parent_hash_len: i32,
  
  hash_ptr: i32,
  hash_len: i32,
  
  code_ptr: i32,
  code_len: i32,
}
```

The full block is up to interpretation of the runtime. The runtime
should return metadata required as defined above. The host can fetch
the metadata using `read_metadata` function. If the execution was
invalid, the host should not assume data under `result_ptr`.

`execute` should always be used together with memory management
functions. A typical cycle looks like below:

* Call `write_code` for guest to allocate memory for code data
  slice. Then copy code parameter to guest memory.
* Call `write_block` for guest to allocate memory for block data
  slice. Then copy block parameter to guest memory.
* Call `execute`.
* Call `read_metadata` if execution was successful, and copy metadata
  result from guest back to host.
* Call `free` to deallocate parameters and results.

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

## Initial Runtime and Block Structure

The actual `runtime` (which exposes `execute`) function is expected to
be stateless. Notice we don't provide any imports for the execution
environment. The initial runtime only supports two operations --
transfer, and runtime upgrade.

All below structures are stored in binary merkle tree (`bm` is the
reference implementation), and encoded via SCALE (`parity-codec` is
the reference implementation).

### Types and Structures

We define `State`, which consists of the following structures:

```
type AccountId = u64;
type UpgradeId = u64;
type Balance = u128;
type Nonce = u64;
type Public = H256;

struct Account {
  balance: Balance,
  nonce: Nonce,
  public: Public
}

struct Upgrade {
  votes: VecArray<bool, U4096>,
  code: Vec<u8>,
}

struct State {
  accounts: Vec<Account>,
  upgrades: Vec<Upgrade>,
}
```

The block structures are defined as follows:

```
type Difficulty = u128;
type Timestamp = u64;
type Signature = H512;
type StateProof = bm::CompactValue<bm_le::Value>;

enum TransferId {
  Coinbase,
  Existing(AccountId),
  New(Public),
}

enum CoinbaseId {
  Existing(AccountId),
  New(Public),
}

enum UnsealedTransaction {
  UpgradeProposal {
    from: AccountId,
    code: Vec<u8>,
  },
  Transfer {
    from: AccountId,
    to: Vec<(TransferId, Balance)>,
  },
}

struct Transaction {
  unsealed: UnsealedTransaction,
  signature: Signature,
}

struct UnsealedBlock {
  parent_id: Option<H256>,
  coinbase: CoinbaseId,
  timestamp: Timestamp,
  difficulty: Difficulty,
  state_proof: StateProof,
  upgrade_vote: Option<UpgradeId>,
  transactions: Vec<Transaction>,
}

struct Block {
  unsealed: UnsealedBlock,
  pow_proof: Vec<u8>,
}
```

### Block Execution

* **Validity of Proof of Work Proof**: Upon receiving a new block
  structure, the executor should first decode `difficulty` and
  `timestamp`, and check whether `pow_proof` is valid under the given
  difficulty and timestamp. We're still deciding on the actual proof
  of work algorithm and difficulty adjustment algorithm. The block
  time is tentatively set to one minute.
* **Validity of State Proof**: Given a transaction or a coinbase id,
  it is possible to know all the state (defined as generalized merkle
  index) it is going to touch. Check all values exist in block's given
  state proof.
* **Validity of Transaction Signatures**: Check that
  `transaction.from` exists in `state.accounts`, and check that the
  `signature` is valid against `state.accounts[i].public`. Increase
  `state.accounts[i].nonce` by one.
* **Execution of Transfer Transaction**: Check that `from` account has
  balance greater than all `to` amount combined. Transfer value of
  `from` into all `to` account, with balance specified as the second
  item in the tuple. If `to` is coinbase, transfer to coinbase
  account. If `to` is new account, create a new account with all
  fields set to `0`, and transfer to the new account. Note that we
  don't have the concept of transaction fees -- it is fulfilled by
  `coinbase` destination.
* **Execution of Upgrade Proposal Transaction**: Deduct
  `PROPOSAL_COST` from `from` account. After that, push `code` as a
  new upgrade proposal in `state.upgrades`, and set its `votes` to
  `false`.
* **Evaluation of Current Upgrade Proposals**: Iterate over all
  `state.upgrades`, shift all `votes` to the right. Push `true` if
  `block.upgrade_vote` equals to the proposal index. Otherwise, push
  `false`. If a given proposal's `votes` has more than 3072 `true`
  (more than 75% of blocks voted for the proposal in the past 4096
  blocks), then set the runtime code as in the upgrade proposal. At
  this moment, the initial runtime continue its execution, and reaches
  its end-of-life after the current block finishes.
* **Block Rewards**: Increase `block.coinbase`'s balance by
  `BLOCK_REWARD`. If coinbase points to a new account, create it with
  all fields set to `0`.
