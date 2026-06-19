# MASM IR

MASM is a portable intermediate representation for the Mach compiler. It defines a compact, target-independent instruction set that sits between AST lowering and ISA-specific instruction selection.

## Documentation

- **[IR Opcodes](ir.md)** — 30-opcode vocabulary with dot-suffix variants
- **[Instruction Format](instruction.md)** — 8-byte encoding, meta byte, field layout
- **[Pools](pools.md)** — ConstPool, DataPool, LabelTable, TypeDescTable
- **[Control Flow Graph](cfg.md)** — CFG construction, dominance, liveness analysis
- **[Codec](codec.md)** — Binary `.mo` and text `.masm` serialization formats
- **[Verifier](verify.md)** — IR well-formedness validation

## Design Principles

- **Target-independent**: No ISA-specific opcodes, registers, or encodings. Backends translate MASM IR to machine instructions via instruction selection.
- **Compact**: 8 bytes per instruction. Metadata that exceeds 16-bit field width lives in per-function pools.
- **Dot-suffix variants**: Related operations share an opcode and encode the variant in the meta byte's `cc` or `flag` fields (e.g., `div.rem`, `cmp.f`, `conv.zext`).
- **ABI-abstract**: `OP_ARG`, `OP_PARAM`, and `OP_RET` carry type descriptors so any calling convention can classify arguments without ISA-specific metadata in the IR.

## Module Structure

```
Module
  functions: []*Function
  data_decls: DataDeclList
  debug_files: []str

Function
  name: str
  blocks: []*Block
  const_pool: ConstPool      (64-bit constants)
  labels: LabelTable          (label name interning)
  data_pool: DataPool         (instruction metadata bytes)
  type_descs: TypeDescTable   (ABI type descriptors)

Block
  label: str
  insts: []Instruction
```

## Source Files

| Module | File | Description |
|--------|------|-------------|
| `masm.op` | `src/op.mach` | Opcode definitions, condition codes, constants |
| `masm.inst` | `src/inst.mach` | Instruction record, constructors, accessors |
| `masm.pool` | `src/pool.mach` | ConstPool, DataPool, LabelTable |
| `masm.typedesc` | `src/typedesc.mach` | Type descriptors for ABI classification |
| `masm.function` | `src/function.mach` | Function and Block records |
| `masm.module` | `src/module.mach` | Module and ModuleBuilder |
| `masm.cfg` | `src/cfg.mach` | CFG, dominance, liveness |
| `masm.verify` | `src/verify.mach` | IR validation pass |
| `masm.codec` | `src/codec.mach` | Encode/decode/print/parse API |
| `masm.symbol` | `src/symbol.mach` | Symbols, sections, relocations |
