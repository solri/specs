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

## Value Types

Solri-flavored WebAssembly supports the following value types:

* `i32`: 32-bit integer
* `i64`: 64-bit integer

That is, WebAssembly MVP value types with floating points
disabled. Practically, this means that execution environments should
check that all values in function signatures does not contain floating
point types.

## Instructions

The following instructions are supported in Solri-flavored
WebAssembly:

* Unreachable
* Nop
* Block(BlockType)
* Loop(BlockType)
* If(BlockType)
* Else
* End
* Br(u32)
* BrIf(u32)
* BrTable(Box<BrTableData>)
* Return
* Call(u32)
* CallIndirect(u32 u8)
* Drop
* Select
* GetLocal(u32)
* SetLocal(u32)
* TeeLocal(u32)
* GetGlobal(u32)
* SetGlobal(u32)
* I32Load(u32 u32)
* I64Load(u32 u32)
* I32Load8S(u32 u32)
* I32Load8U(u32 u32)
* I32Load16S(u32 u32)
* I32Load16U(u32 u32)
* I64Load8S(u32 u32)
* I64Load8U(u32 u32)
* I64Load16S(u32 u32)
* I64Load16U(u32 u32)
* I64Load32S(u32 u32)
* I64Load32U(u32 u32)
* I32Store(u32 u32)
* I64Store(u32 u32)
* I32Store8(u32 u32)
* I32Store16(u32 u32)
* I64Store8(u32 u32)
* I64Store16(u32 u32)
* I64Store32(u32 u32)
* CurrentMemory(u8)
* GrowMemory(u8)
* I32Const(i32)
* I64Const(i64)
* I32Eqz
* I32Eq
* I32Ne
* I32LtS
* I32LtU
* I32GtS
* I32GtU
* I32LeS
* I32LeU
* I32GeS
* I32GeU
* I64Eqz
* I64Eq
* I64Ne
* I64LtS
* I64LtU
* I64GtS
* I64GtU
* I64LeS
* I64LeU
* I64GeS
* I64GeU
* I32Clz
* I32Ctz
* I32Popcnt
* I32Add
* I32Sub
* I32Mul
* I32DivS
* I32DivU
* I32RemS
* I32RemU
* I32And
* I32Or
* I32Xor
* I32Shl
* I32ShrS
* I32ShrU
* I32Rotl
* I32Rotr
* I64Clz
* I64Ctz
* I64Popcnt
* I64Add
* I64Sub
* I64Mul
* I64DivS
* I64DivU
* I64RemS
* I64RemU
* I64And
* I64Or
* I64Xor
* I64Shl
* I64ShrS
* I64ShrU
* I64Rotl
* I64Rotr
* I32WrapI64
* I64ExtendSI32
* I64ExtendUI32

That is, all MVP instructions with floating point instructions
disabled. Practically, this means that the execution environment
should check that none of the floating point instructions exist.

## Module Sections

The following sections are allowed in Solri-flavoured WebAssembly:

* **import**: Module imports. Linear memory, global variable and
  functions are supported.
* **export**: Module exports. Linear memory, global variable and
  functions are supported.
* **start**: Module start function.
* **global**: Global section.
* **memory**: Linear memory.
* **data**: Data.
* **function** and **code**: Function and code.

That is, all MVP sections with table support disabled.
