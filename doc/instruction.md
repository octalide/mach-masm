# Instruction Format

Every MASM instruction is exactly 8 bytes:

```
Instruction {
    op:   u8;    // opcode (0-29)
    meta: u8;    // packed: cc:4 | size_log2:3 | flag:1
    f0:   u16;   // destination register, parameter index, or result register
    f1:   u16;   // source register, base register, or byte size
    f2:   u16;   // source register, pool offset, label index, or literal
}
```

## Meta Byte

The meta byte packs three fields into 8 bits:

```
bit  7  6  5  4  3  2  1  0
    [  cc (4)  ][sz_log2][f]
```

- **cc** (bits 7-4): Condition code, variant selector, or memory ordering. Interpretation depends on the opcode.
- **size_log2** (bits 3-1): Log2 of operand size in bytes. `1 << size_log2` gives the byte count.
- **flag** (bit 0): Context-dependent. Common uses: indexed addressing (load/store), immediate source (ALU), indirect call, float comparison, syscall argument.

### Size Encoding

| Encoding | Bytes |
|----------|-------|
| 0 | 1 |
| 1 | 2 |
| 2 | 4 |
| 3 | 8 |
| 4 | 16 |
| 5 | 32 |
| 6 | 64 |
| 7 | 128 |

Valid sizes are powers of two from 1 to 128.

## Virtual Register Space

| Range | Meaning |
|-------|---------|
| 0-29 | Physical registers |
| 30 | `VREG_SP` — stack pointer |
| 31 | `VREG_FP` — frame pointer |
| 32+ | `VREG_START` — user-allocated virtual registers |

## Field Interpretation by Opcode

### Data Movement

| Opcode | cc | flag | f0 | f1 | f2 |
|--------|----|------|----|----|----|
| `mov` (reg) | 0 | 0 | dst | src | 0 |
| `mov.imm32` | 1 | 0 | dst | 0 | data pool offset |
| `mov.imm64` | 0 | 1 | dst | 0 | const pool index |
| `load` | 0 | 0 | dst | base | data pool offset (disp) |
| `load.indexed` | scale | 1 | dst | base | data pool offset (packed) |
| `store` | 0 | 0 | src | base | data pool offset (disp) |
| `store.indexed` | scale | 1 | src | base | data pool offset (packed) |
| `addr` | 0 | 0 | dst | base | data pool offset (disp) |
| `addr.indexed` | scale | 1 | dst | base | data pool offset (packed) |
| `addr.label` | 1 | 0 | dst | 0 | label index |

### ALU Operations

| Opcode | cc | flag | f0 | f1 | f2 |
|--------|----|------|----|----|----|
| `add`/`sub`/`mul`/`and`/`or`/`xor`/`shl` | 0 | 0 | dst | src1 | src2 |
| `add.f`/`sub.f`/`mul.f` | ALU_FLOAT | 0 | dst | src1 | src2 |
| ALU with immediate | 0 | 1 | dst | src1 | const pool index |
| `div` | DIV_* | 0 | dst | src1 | src2 |
| `shr` | 0 or SHR_ARITHMETIC | 0 | dst | src1 | src2 |
| `cmp` | CC_* | 0 | dst | src1 | src2 |
| `cmp.f` | CC_* | 1 | dst | src1 | src2 |

### Control Flow

| Opcode | cc | flag | f0 | f1 | f2 |
|--------|----|------|----|----|----|
| `jmp` | 0 | 0 | 0 | 0 | label index |
| `branch` | CC_* | 0 | cond reg | 0 | label index |
| `call` (direct) | 0 or CALL_TAIL | 0 | result reg | 0 | label index |
| `call` (indirect) | 0 or CALL_TAIL | 1 | result reg | target reg | 0 |
| `ret` (void) | 0 | 0 | 0 | 0 | 0 |
| `ret` (value) | 0 | 0 | value reg | 0 | type desc index |

### ABI

| Opcode | cc | flag | f0 | f1 | f2 |
|--------|----|------|----|----|----|
| `arg` | 0 | 0 | arg index | value reg | type desc index |
| `arg.sc` | 0 | 1 | arg index | value reg | 0 |
| `param` | 0 | 0 | param index | byte size | type desc index |

### Conversions

| Opcode | cc | flag | f0 | f1 | f2 |
|--------|----|------|----|----|----|
| `conv.*` | CONV_* | 0 | dst | src | 0 |
| `conv.ftoi` | CONV_FTOI | 0 | dst | src | src float size |

### Atomics

| Opcode | cc | flag | f0 | f1 | f2 |
|--------|----|------|----|----|----|
| `atomic.load` | MO_* | 0 | dst | base | data pool offset (disp) |
| `atomic.store` | MO_* | 0 | src | base | data pool offset (disp) |
| `atomic.cas` | MO_* | 0 | result | base | data pool offset (packed) |
| `atomic.rmw` | MO_* | 0 | result | base | data pool offset (packed) |
| `fence` | MO_* | 0 | 0 | 0 | 0 |

### Misc

| Opcode | cc | flag | f0 | f1 | f2 |
|--------|----|------|----|----|----|
| `label` | 0 | 0 | 0 | 0 | label index |
| `syscall` | 0 | 0 | result reg | 0 | 0 |
| `trap` | 0 | 0 | 0 | 0 | 0 |
| `hint` | HINT_* | 0 | 0 | 0 | 0 |

## Indexed Memory Operands

Indexed addressing packs an index register and 16-bit displacement into a 32-bit value stored in the data pool:

```
packed = (index_reg << 16) | (disp16 & 0xFFFF)
```

The `cc` field encodes the scale factor as a 2-bit log2 value: 0=1, 1=2, 2=4, 3=8.

## Constructors

All instruction constructors are in `inst.mach`. Multi-opcode constructors take an opcode parameter; single-opcode constructors hardcode it:

- `make_alu(o, dst, src1, src2, size)` — OP_ADD through OP_SHL
- `make_alu_imm(o, dst, src1, const_idx, size)` — with immediate
- `make_alu_float(o, dst, src1, src2, size)` — float variant
- `make_div(dst, src1, src2, mode, size)`
- `make_shr(dst, src1, src2, mode, size)`
- `make_cmp(dst, src1, src2, cc, size)` / `make_cmp_float(...)`
- `make_conv(dst, src, kind, size)` / `make_conv_ftoi(dst, src, dst_size, src_float_size)`
- `make_mov_reg(dst, src, size)` / `make_mov_imm32(dst, pool_off, size)` / `make_mov_imm64(dst, pool_off, size)`
- `make_load(dst, base, disp_off, size)` / `make_load_indexed(dst, base, packed_off, scale, size)`
- `make_store(src, base, disp_off, size)` / `make_store_indexed(...)`
- `make_addr(dst, base, disp_off, size)` / `make_addr_indexed(...)` / `make_addr_label(dst, lbl_idx, size)`
- `make_jmp(lbl_idx)` / `make_branch(cond_reg, lbl_idx, cc)`
- `make_call_direct(result_reg, lbl_idx, tail, size)` / `make_call_indirect(result_reg, target_reg, tail, size)`
- `make_ret()` / `make_ret_value(value_reg, type_desc_idx, size)`
- `make_label(lbl_idx)` / `make_syscall(result_reg)`
- `make_param(param_idx, byte_size, type_desc_idx)` / `make_arg(arg_idx, value_reg, type_desc_idx, size)` / `make_scarg(arg_idx, value_reg, size)`
- `make_atomic_load(...)` / `make_atomic_store(...)` / `make_atomic_cas(...)` / `make_atomic_rmw(...)`
- `make_fence(ordering)` / `make_trap()` / `make_hint(kind)`
