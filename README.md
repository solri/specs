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
`-1` if the block is deemed invalid. Otherwise, the result is `0`.

If the runtime cannot execute on the current WebAssembly executor, it
should trap (for example, by using the `unreachable` opcode).

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
type StateProof = bm::CompactValue<bm_le::Intermediate, bm_le::End>;

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
