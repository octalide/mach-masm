# Pools

Per-function data pools store values that exceed the 16-bit instruction field width. Instructions reference pool entries via their `f2` field.

## ConstPool

Stores 64-bit program constants: addresses, large integers, and double-precision floats bit-cast to `u64`. Instructions reference entries by **index**.

```
ConstPool {
    values: *u64;
    count:  i32;
    cap:    i32;
    dedup:  Map[u64, i32];
}
```

**API:**
- `const_init(alloc) ConstPool` — create empty pool
- `const_intern(alloc, pool, value) i32` — add value, returns index (>= 0) or -1
- `const_get(pool, idx) u64` — retrieve by index
- `const_count(pool) i32` — number of interned values
- `const_dnit(alloc, pool)` — free all memory

Deduplicates on intern: identical values share the same index. Grows geometrically from capacity 8.

**Used by:** `mov.imm64` (f2 = const pool index), ALU immediate variants (f2 = const pool index with flag=1).

## LabelTable

Interns label name strings. Branch and jump instructions reference labels by **index** rather than carrying string pointers.

```
LabelTable {
    names: *str;
    count: i32;
    cap:   i32;
    dedup: Map[str, i32];
}
```

**API:**
- `label_init(alloc) LabelTable` — create empty table
- `label_intern(alloc, tbl, name) i32` — add name, returns index (>= 0) or -1
- `label_get(tbl, idx) str` — retrieve by index
- `label_count(tbl) i32` — number of interned labels
- `label_dnit(alloc, tbl)` — free all memory

Deduplicates by string equality. Grows geometrically from capacity 16.

**Used by:** `jmp` (f2), `branch` (f2), `call` direct (f2), `label` (f2), `addr.label` (f2).

## DataPool

Width-agnostic byte pool for instruction metadata: displacements (i32), packed indexed memory operands, and other binary data. Instructions reference entries by **byte offset** into the backing buffer.

```
DataPool {
    data:     *u8;
    offsets:  *i32;       (internal, for dedup)
    sizes:    *i32;       (internal, for dedup)
    count:    i32;
    cap:      i32;
    data_len: i32;
    data_cap: i32;
    dedup:    Map[u64, i32];
}
```

**API:**
- `data_init(alloc) DataPool` — create empty pool
- `data_intern(alloc, pool, ptr, size) i32` — intern bytes, returns byte offset (>= 0) or -1
- `data_intern_i32(alloc, pool, value) i32` — convenience for a single i32
- `data_get(pool, offset) *u8` — retrieve pointer by byte offset
- `data_get_i32(pool, offset) i32` — convenience to read an i32
- `data_length(pool) i32` — used byte count in buffer
- `data_dnit(alloc, pool)` — free all memory

Entries are 8-byte aligned for safe typed pointer reads. Deduplicates via FNV-1a hash with byte-level equality verification. Grows geometrically from 16 entries / 128 bytes.

**Used by:** `mov.imm32` (f2), `load`/`store`/`addr` (f2 for displacement), indexed variants (f2 for packed index+disp), atomics (f2 for packed metadata).

## TypeDescTable

Per-function table of type descriptors for ABI classification. `OP_ARG`, `OP_PARAM`, and `OP_RET` instructions reference entries by **index** in their `f2` field.

```
TypeDescTable {
    descs: Vector[TypeDesc];
}

TypeDesc {
    kind:        u8;     // TYPEDESC_INT, TYPEDESC_FLOAT, TYPEDESC_PTR, TYPEDESC_AGG
    total_size:  u16;
    align:       u8;
    field_count: u8;
    fields:      *TypeDescField;    // nil for scalars
}

TypeDescField {
    offset: u16;
    kind:   u8;
    size:   u8;
}
```

**API:**
- `init(alloc) TypeDescTable` — create empty table
- `intern(tbl, desc) i32` — append descriptor, returns index (>= 0) or -1
- `get(tbl, idx) *TypeDesc` — retrieve by index
- `count(tbl) i32` — number of descriptors
- `dnit(alloc, tbl)` — free all memory (including per-descriptor field arrays)

Convenience constructors: `make_int(size, align)`, `make_float(size, align)`, `make_ptr(ptr_size)`, `make_agg(total_size, align, fields, field_count)`.

**Purpose:** Gives ABI lowering passes enough type information to classify arguments and return values (GP vs FP register, stack passing, aggregate splitting) without encoding ABI-specific metadata into the instruction format.

## Ownership

All pools are owned by their parent `Function`. They share the same allocator, grow independently, and are freed by `func_dnit`.
