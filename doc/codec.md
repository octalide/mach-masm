# Codec

MASM modules can be serialized in two formats: a compact binary format (`.mo`) and a human-readable text format (`.masm`). Both round-trip through `masm.codec`.

## Binary Format (.mo)

Version 2. Compact, designed for fast load/store.

### File Layout

```
[File Header]
  magic:       "MASM" (4 bytes)
  version:     u32 (2)
  func_count:  u32
  data_count:  u32
  file_count:  u32
  strtab_size: u32
  symtab_size: u32

[String Table]
  Packed null-terminated strings, deduplicated.
  Referenced by byte offset from other sections.

[Symbol Table]
  Flat array of symbol entries with name offsets into the string table.

[Debug File Table]
  File path offsets into the string table.

[Functions] (repeated func_count times)
  Function header:
    name offset, param_count, flags,
    pool counts (const, label, data, typedesc),
    block count

  Per-function pools:
    Const pool values (u64 array)
    Label names (string offsets)
    Data pool bytes
    Type descriptors

  Blocks (repeated block_count times):
    Label offset, instruction count
    Instructions (8 bytes each)
    Debug locations (optional, if FUNC_HAS_DEBUG)

[Data Declarations]
  Bytes or zero-fill with relocations.
```

## Text Format (.masm)

Human-readable assembly syntax. Parsed by a recursive descent parser (~1300 lines).

### Example

```masm
function main {
  .entry:
    param p0, 8, td0
    mov.8 r32, r33
    add.8 r34, r32, r33
    cmp.eq.8 r35, r34, r32
    branch.ne r35, .done
    jmp .entry
  .done:
    ret
}
```

### Syntax Elements

- `function <name> { ... }` — function definition
- `.<label>:` — block label
- `<mnemonic>.<suffix>.<size> <operands>` — instruction
- `r<N>` — register (physical 0-29, virtual 32+)
- `SP`, `FP` — stack/frame pointer aliases
- `#<int>` — immediate constant
- `[r<N> + <disp>]` — memory operand

## API

```
// Binary
codec.encode_file(alloc, module, path) Result[bool, str]
codec.decode_file(alloc, path) Result[Module, str]

// Text
codec.print_file(alloc, module, path) Result[bool, str]
codec.parse_file(alloc, path) Result[Module, str]

// In-memory
codec.encode(alloc, module, buffer) Result[bool, str]
codec.decode(alloc, data, len) Result[Module, str]
codec.print(alloc, module, writer) Result[bool, str]
codec.parse(alloc, src, len) Result[Module, str]
```

## Round-Trip

Binary → text → binary produces identical IR. The text format preserves all semantic information. Label pseudo-instructions may appear in binary but are implicit in text block headers.
