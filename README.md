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

* [Execution Environment and Runtime](./runtime.md)
* [Solri-flavored WebAssembly](./webassembly.md)
