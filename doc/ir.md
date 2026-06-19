# IR Opcodes

MASM defines 30 opcodes (0-29). Variants are encoded in the meta byte's `cc` and `flag` fields, represented textually as dot suffixes (e.g., `div.rem`, `cmp.f.eq`, `conv.zext`).

## Opcode Table

| # | Mnemonic | Variants | Description |
|---|----------|----------|-------------|
| 0 | `mov` | `.imm32` (cc=1), `.imm64` (flag=1) | Register move or immediate load |
| 1 | `load` | `.indexed` (flag=1) | Load from memory |
| 2 | `store` | `.indexed` (flag=1) | Store to memory |
| 3 | `addr` | `.indexed` (flag=1), `.label` (cc=1) | Effective address computation |
| 4 | `add` | `.f` (cc=1) | Addition |
| 5 | `sub` | `.f` (cc=1) | Subtraction |
| 6 | `mul` | `.f` (cc=1) | Multiplication |
| 7 | `div` | `.rem`, `.u`, `.remu`, `.f` | Division (cc selects mode) |
| 8 | `and` | | Bitwise AND |
| 9 | `or` | | Bitwise OR |
| 10 | `xor` | | Bitwise XOR |
| 11 | `shl` | | Shift left |
| 12 | `shr` | `.a` (cc=1) | Shift right (logical or arithmetic) |
| 13 | `cmp` | `.f` (flag=1) | Comparison (cc=condition code) |
| 14 | `jmp` | | Unconditional jump |
| 15 | `branch` | | Conditional branch (cc=condition code) |
| 16 | `call` | `.tail` (cc=1), `.indirect` (flag=1) | Function call |
| 17 | `ret` | `.value` (f0 nonzero) | Return from function |
| 18 | `conv` | `.zext`, `.sext`, `.trunc`, `.itof`, `.ftoi`, `.fext`, `.ftrunc` | Type conversion (cc selects kind) |
| 19 | `label` | | Label definition (pseudo-instruction) |
| 20 | `syscall` | | System call |
| 21 | `arg` | `.sc` (flag=1) | Place call/syscall argument |
| 22 | `param` | | Receive function parameter |
| 23 | `atomic.load` | | Atomic load (cc=memory ordering) |
| 24 | `atomic.store` | | Atomic store (cc=memory ordering) |
| 25 | `atomic.cas` | | Atomic compare-and-swap |
| 26 | `atomic.rmw` | | Atomic read-modify-write |
| 27 | `fence` | | Memory barrier (cc=memory ordering) |
| 28 | `trap` | | Unconditional halt |
| 29 | `hint` | `.pause` (cc=0) | CPU hint |

The canonical range boundary is `OP_CANONICAL_MAX = 29`. Compiler-internal extensions must use opcode values strictly above this.

## Condition Codes

Used by `cmp`, `cmp.f`, and `branch` in the meta byte's cc field:

| Code | Value | Description |
|------|-------|-------------|
| `CC_NONE` | 0 | No condition |
| `CC_EQ` | 1 | Equal |
| `CC_NE` | 2 | Not equal |
| `CC_LT` | 3 | Signed less than |
| `CC_LE` | 4 | Signed less or equal |
| `CC_GT` | 5 | Signed greater than |
| `CC_GE` | 6 | Signed greater or equal |
| `CC_ULT` | 7 | Unsigned less than |
| `CC_ULE` | 8 | Unsigned less or equal |
| `CC_UGT` | 9 | Unsigned greater than |
| `CC_UGE` | 10 | Unsigned greater or equal |

Comparisons produce 1 (true) or 0 (false) in the destination register. No implicit flags.

## Division Modes

Used by `div` in the cc field:

| Mode | Value | Result |
|------|-------|--------|
| `DIV_QUOT` | 0 | Signed quotient |
| `DIV_REM` | 1 | Signed remainder |
| `DIV_UQUOT` | 2 | Unsigned quotient |
| `DIV_UREM` | 3 | Unsigned remainder |
| `DIV_FLOAT` | 4 | Float division |

## Conversion Kinds

Used by `conv` in the cc field:

| Kind | Value | Description |
|------|-------|-------------|
| `CONV_ZEXT` | 0 | Zero extend |
| `CONV_SEXT` | 1 | Sign extend |
| `CONV_TRUNC` | 2 | Truncate |
| `CONV_ITOF` | 3 | Integer to float |
| `CONV_FTOI` | 4 | Float to integer |
| `CONV_FEXT` | 5 | Float extend (f32 to f64) |
| `CONV_FTRUNC` | 6 | Float truncate (f64 to f32) |

## Memory Orderings

Used by atomics and `fence` in the cc field:

| Ordering | Value |
|----------|-------|
| `MO_RELAXED` | 0 |
| `MO_ACQUIRE` | 1 |
| `MO_RELEASE` | 2 |
| `MO_ACQ_REL` | 3 |
| `MO_SEQ_CST` | 4 |

## Atomic RMW Operations

Used by `atomic.rmw` packed in the data pool entry:

| Op | Value |
|----|-------|
| `RMW_ADD` | 0 |
| `RMW_SUB` | 1 |
| `RMW_AND` | 2 |
| `RMW_OR` | 3 |
| `RMW_XOR` | 4 |
| `RMW_XCHG` | 5 |

## Float ALU

`add`, `sub`, and `mul` support a float variant via `cc = ALU_FLOAT` (1). Textually: `add.f`, `sub.f`, `mul.f`. Float division uses `div.f` (`DIV_FLOAT`).

## Terminators

These opcodes terminate a basic block: `jmp`, `branch`, `ret`, `trap`, and `call.tail`.
