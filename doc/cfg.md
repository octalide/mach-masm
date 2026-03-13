# Control Flow Graph

`masm.cfg` provides CFG construction, dominance analysis, and backward dataflow liveness analysis over MASM IR functions. These are optional analysis passes — the CFG is built on demand, not maintained automatically.

## CFG Construction

```
CFG {
    blocks:      *BasicBlock;
    block_count: u16;
    entry:       u16;         // always 0
}

BasicBlock {
    block:  *Block;    // pointer to the masm.Block in Function.blocks
    id:     u16;       // block index
    preds:  *u16;      // predecessor IDs (slice into shared edge array)
    npreds: u16;
    succs:  *u16;      // successor IDs (slice into shared edge array)
    nsuccs: u16;
}
```

**API:**
- `build_cfg(alloc, func) CFG` — analyze blocks, build edges
- `cfg_dnit(alloc, cfg)` — free all memory

Construction analyzes each block's terminator instruction to determine successors:
- `jmp` → single successor (target label)
- `branch` → two successors (target label + fallthrough)
- `ret`, `trap`, tail `call` → no successors

Predecessors are computed in reverse from the successor edges. Edge arrays are allocated contiguously; each BasicBlock holds a slice into the shared array.

## Dominance Analysis

Uses the Cooper-Harvey-Kennedy iterative algorithm on the reverse postorder traversal.

```
DomInfo {
    idom:       *u16;     // immediate dominator per block
    rpo:        *u16;     // reverse postorder traversal
    rpo_count:  u16;      // blocks in RPO (excludes unreachable)
    rpo_number: *u16;     // RPO position per block
    block_count: u16;
}
```

**API:**
- `compute_dominance(alloc, cfg) DomInfo`
- `dominates(dom, a, b) bool` — does block `a` dominate block `b`?
- `dom_dnit(alloc, dom)` — free all memory

The entry block dominates itself. Unreachable blocks are excluded from the RPO traversal.

## Liveness Analysis

Backward dataflow analysis computing live-in and live-out register sets per block.

```
RegSet {
    bits:       *u64;     // packed bitset (one bit per register)
    word_count: u16;      // ceil(max_reg / 64)
}

LiveSets {
    live_in:  *RegSet;    // per-block live-in sets
    live_out: *RegSet;    // per-block live-out sets
    count:    u16;
}
```

**API:**
- `compute_liveness(alloc, cfg, func) LiveSets`
- `is_live_in(sets, block, reg) bool`
- `is_live_out(sets, block, reg) bool`
- `live_dnit(alloc, sets)` — free all memory

**Algorithm:**
1. Scan all instructions to find the maximum register ID
2. Compute per-block `def` and `use` sets from instruction operands
3. Iterate in reverse RPO: `live_in = use | (live_out & ~def)`, `live_out = union(live_in of successors)`
4. Repeat until convergence (no live_in set changes)

The liveness analysis correctly handles registers that are live-through a block (present in `live_out` but not `def`/`use` locally).

## Usage Pattern

```
val cfg: CFG = build_cfg(alloc, func);
val dom: DomInfo = compute_dominance(alloc, ?cfg);
val live: LiveSets = compute_liveness(alloc, ?cfg, func);

// use live.is_live_in, live.is_live_out, dom.dominates...

live_dnit(alloc, ?live);
dom_dnit(alloc, ?dom);
cfg_dnit(alloc, ?cfg);
```

The CFG must be rebuilt if blocks are modified after construction.
