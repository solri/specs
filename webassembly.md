# Solri-flavored WebAssembly

This defines the WebAssembly execution environment for
Solri. Solri-flavored WebAssemly is a subset of WebAssembly that is
deterministic.

## Design Principles

We still aim to maintain WebAssembly's goal of open web platform --
Solri-flavored WebAssembly maintains to be versionless,
feature-tested, and backward compatible. The WebAssembly execution
environment for Solri is aimed both as the blockchain's specification,
and an efficient implementation.

We do not define a frozen specification here. Instead, by versionless,
we allow new features to be added onto Solri-flavored WebAssembly, as
long as it remains backward compatible. A feature being added in the
execution environment does not mean it will be used by any runtime. On
the contrary, runtimes are recommended to wait until the majority of
the network upgraded their execution environments, before adopting the
feature.

In the case when a runtime detects that it requires a feature that the
current execution environment does not support, it must correctly
**trap**. This does make it so that older client might not be able to
execute some runtimes, but it does not result in the classic hard fork
situation where chains can split -- older clients will not be able to
execute any possible blocks on the current runtime, nor build any
blocks.
