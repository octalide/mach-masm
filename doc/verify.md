# Verifier

`masm.verify` validates that MASM IR is well-formed. It catches malformed instructions, out-of-bounds pool references, and structural block violations before they propagate to codegen.

## API

```
verify_function(func: *Function) Result[bool, str]
verify_module(m: *Module) Result[bool, str]
```

Returns `ok(true)` if valid. On the first violation, returns `err` with a description.

## CLI

```
masm verify <input.mo|input.masm>
```

Loads the module (tries binary first, falls back to text), runs `verify_module`, prints `OK` or `FAIL: <message>`.

## Checks

### Per-Instruction

| Check | Description |
|-------|-------------|
| Opcode range | `op < OP_COUNT` (0-29) |
| Condition code | `is_valid_cc(op, cc)` — validates cc for the specific opcode |
| Flag bit | 0 or 1 |

### Per-Opcode Shape

| Opcode | Constraints |
|--------|-------------|
| `mov` (reg) | cc=0, flag=0 |
| `mov.imm32` | cc=1, flag=0, f2 < data pool length |
| `mov.imm64` | cc=0, flag=1, f2 < const pool count |
| `load`/`store` | f2 < data pool length |
| `addr` | f2 < data pool length |
| `addr.label` | cc=1, flag=0, f2 < label count |
| ALU (flag=1) | f2 < const pool count (immediate) |
| `jmp` | f0=0, f1=0, f2 < label count |
| `branch` | f2 < label count |
| `call` (direct) | flag=0, f2 < label count |
| `call` (indirect) | flag=1, f2=0 |
| `ret` (void) | f0=0, f1=0, f2=0 |
| `ret` (value) | f2 < type desc count |
| `label` | f0=0, f1=0, f2 < label count |
| `syscall` | f1=0, f2=0 |
| `arg` | flag=0: f2 < type desc count |
| `param` | f2 < type desc count |
| `atomic.*` | f2 < data pool length |
| `fence` | f0=0, f1=0, f2=0 |
| `trap` | all fields and meta = 0 |
| `hint` | f0=0, f1=0, f2=0 |

### Pool Bounds

All pool references are checked against the owning function's pool sizes:
- Label index: `0 <= idx < label_count()`
- Const pool index: `0 <= idx < const_count()`
- Data pool offset: `0 <= offset < data_length()`
- Type desc index: `0 <= idx < count()`

Bounds checks are skipped when the pool is empty (count = 0), since instructions may use f2=0 as a valid "no value" sentinel.

### Block-Level

| Check | Description |
|-------|-------------|
| Non-empty | Every block has at least one instruction |
| Terminator | Last instruction is `jmp`, `branch`, `ret`, `trap`, or tail `call` |
| No mid-block terminator | No terminator appears before the last instruction |
| Label position | `OP_LABEL` only appears as the first instruction |

### Function-Level

| Check | Description |
|-------|-------------|
| Has blocks | At least one block exists |
