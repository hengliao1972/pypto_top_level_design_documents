# Tensor `tile_shape`: Tile-Contiguous Memory Layout for Efficient TLOAD/TSTORE

## Overview

This document specifies an optional `tile_shape` attribute for GM (Global Memory) tensors in pypto. By declaring a `tile_shape`, the programmer instructs the compiler/PTOAS to lay out the tensor in **tile-contiguous** order rather than the default **row-major** order. When the program later issues `TLOAD` / `TSTORE` operations using exactly that tile shape, every transfer becomes a single contiguous burst, maximizing GM bandwidth, NoC bandwidth, and L2 cache efficiency.

> Ruoyu: This "single contiguous burst" claim needs to be qualified against the current PTOAS/A5 DMA model. PTOAS already lowers `TLOAD`/`TSTORE` through hierarchical DMA operations (`copy_gm_to_ubuf`, `copy_ubuf_to_gm`) with hardware loop registers, `n_burst`, `len_burst`, and GM/UB strides. A tile-contiguous GM layout can make the GM source interval contiguous, but it does not by itself guarantee that the generated PTOAS instruction is one flat DMA burst unless the destination UB layout and lowering rule are also defined that way. The proposal should distinguish "physical GM bytes are contiguous" from "PTOAS emits exactly one DMA transaction."

`tile_shape` is purely a **physical layout hint**. The logical `shape` of the tensor, the type system, and program semantics are unchanged.

> Ruoyu: "Hint" is too weak if external memory bytes are actually reordered. This is a physical layout contract between producer, runtime, PTOAS, and consumer. If host code, another kernel, checkpoint/debug code, or peer transfer assumes row-major bytes, correctness can break even though the logical shape is unchanged.

## 1. Problem Statement

`TLOAD` and `TSTORE` instructions transfer a sub-tensor (a tile / slice) between GM and on-core SRAM (the local memory of an AIV or AIC core). The Ascend memory subsystem — the GM controller, the network-on-chip (NoC), and the cache hierarchy — is optimized around a target burst length and an L2 cache line size:

| Platform | GM/NoC burst length | L2 cache line |
|----------|--------------------|---------------|
| A2 / A3  | 512 B              | 512 B         |
| A5       | 128 B              | 512 B         |

Today, when the programmer defines a GM tensor by giving only its `shape`, the compiler defaults to a **row-major** layout: the bytes along the lowest (innermost) dimension are placed consecutively in memory, then the next lowest, and so on.

This default is fine when programs stream along the innermost dimension. It is **catastrophic** when a kernel accesses small 2-D (or higher-rank) tiles that span multiple rows of a wide tensor.

> Ruoyu: The problem statement should also include the already-existing hierarchical blocking alternative: load a larger macro tile into UB/L0-local storage, then use local layout-transform instructions to extract/insert the subtiles needed by compute. This can avoid repeated GM traffic without changing persistent GM layout. The proposal currently frames the issue as if persistent GM tile layout is the only remedy.

### Concrete Example

Consider a GM tensor of shape `(1024, 1024)` in FP8 (1 byte/element), laid out row-major:

```
row 0:  [ b0,0  b0,1  b0,2 ... b0,1023 ]   (1024 bytes contiguous)
row 1:  [ b1,0  b1,1  b1,2 ... b1,1023 ]   (1024 bytes contiguous)
...
row 1023: ...
```

Suppose a kernel issues a `TLOAD` for a `(16, 16)` FP8 tile starting at `(r0, c0)`. With the row-major layout, the 16 bytes of each tile-row are contiguous, but consecutive tile-rows are **1024 bytes apart**:

```
TLOAD (16, 16) tile from row-major (1024, 1024) FP8 tensor:

  load 0:  16 bytes at  base + (r0+0)*1024 + c0    (BL = 16 B)
  load 1:  16 bytes at  base + (r0+1)*1024 + c0    (BL = 16 B, stride = 1024 B)
  load 2:  16 bytes at  base + (r0+2)*1024 + c0    (BL = 16 B, stride = 1024 B)
  ...
  load 15: 16 bytes at  base + (r0+15)*1024 + c0   (BL = 16 B, stride = 1024 B)

  → 16 short bursts of 16 B each, with a large stride between them.
```

Consequences:

1. **Burst length = 16 B**, far below the optimal 512 B (A2/A3) or 128 B (A5). The GM controller and NoC are forced to issue many short transactions, paying per-transaction overhead on every one.
2. **L2 cache pollution.** Each 16 B fetch pulls in a full 512 B cache line, of which only 16 B is used; the remainder is wasted unless re-touched soon.
3. **NoC channel underutilization.** The wide pipes are starved by the small request size.
4. **Longer end-to-end runtime** of `TLOAD` / `TSTORE`, which often sit on the critical path of compute kernels.

> Ruoyu: The example should compare three cases, not two: row-major direct subtile load, tile-contiguous GM layout, and hierarchical load/store where a larger region, for example a 64 KB macro tile, is brought into UB/L0 once and reused via local transforms. If many nearby subtiles are consumed, the hierarchical path can dominate because it amortizes GM traffic while preserving the existing external tensor layout.

## 2. Solution: Tile-Contiguous Layout via `tile_shape`

We extend the GM tensor definition (typically the output of a `submit` task, or any user-declared GM tensor) with an **optional** `tile_shape` argument:

```python
T = pl.gm_tensor(
        shape      = (M, N),       # logical shape (unchanged)
        dtype      = pl.fp8,
        tile_shape = (TM, TN),     # NEW: optional physical tile shape
)
```

> Ruoyu: This should be presented as one possible layout mechanism, not the whole solution. The design needs a second, orthogonal concept for execution blocking, such as `macro_tile_shape` or compiler-selected staging: GM -> UB/L0 macro tile -> local subtile extraction/insertion -> compute. Persistent `tile_shape` and hierarchical staging solve different problems and should be designed together.

### Rules

1. `tile_shape` is **optional**. If omitted, it defaults to `shape`, which reproduces today's row-major behavior — fully backward compatible.
2. `tile_shape` must have the **same number of dimensions** as `shape`.
3. Each dimension of `shape` must be a **multiple** of the corresponding dimension of `tile_shape`:

   ```
   ∀ i:  shape[i] % tile_shape[i] == 0
   ```

   The compiler shall reject definitions that violate this constraint.

> Ruoyu: Tail handling is missing. PTOAS tile buffers already have valid shape/padding concepts, so requiring exact divisibility may be unnecessarily restrictive. The proposal should decide whether boundary tiles are padded, represented with valid-shape metadata, handled by a separate tail path, or rejected. This matters for both direct `TLOAD` and hierarchical macro-tile staging.

### Memory Layout Definition

The tensor is partitioned into a regular grid of tiles of shape `tile_shape`. Tiles are enumerated in row-major order over the *tile grid* (lowest tile-grid dimension varies fastest). Within each tile, bytes are laid out contiguously in row-major order over the *tile interior*. The full physical layout is the concatenation of all tiles in tile-grid order:

```
physical layout =  [ tile(0,0) bytes ][ tile(0,1) bytes ] ... [ tile(0, N/TN - 1) bytes ]
                   [ tile(1,0) bytes ][ tile(1,1) bytes ] ... [ tile(1, N/TN - 1) bytes ]
                   ...
                   [ tile(M/TM - 1, 0) bytes ] ... [ tile(M/TM - 1, N/TN - 1) bytes ]
```

For the running example (`shape = (1024, 1024)`, FP8, `tile_shape = (16, 16)`), each tile is `16*16 = 256` consecutive bytes in GM:

```
GM byte offsets for tile_shape = (16, 16):

  offset      0 .. 255    : tile (0, 0)         (16x16 contiguous)
  offset    256 .. 511    : tile (0, 1)
  offset    512 .. 767    : tile (0, 2)
  ...
  offset 16128 .. 16383   : tile (0, 63)
  offset 16384 .. 16639   : tile (1, 0)
  ...
```

A `TLOAD` of a `(16, 16)` tile at tile-grid coordinate `(ti, tj)` is now a **single contiguous 256-byte burst** at offset `(ti * (N/TN) + tj) * 256`, instead of 16 strided 16-byte bursts.

> Ruoyu: The 256-byte example is contiguous in GM, but A5 DMA is still modeled with `n_burst`, `len_burst`, and source/destination strides. If the UB destination tile remains a 2-D row-major tile buffer, PTOAS may still naturally express this as multiple rows unless it intentionally flattens the UB destination. The proposal should specify whether matching `tile_shape` lowers to `(n_burst = 1, len_burst = tile_bytes)` or to a 2-D DMA where GM stride equals row width. The performance claim depends on this lowering rule.

### Visual Comparison

```
Default row-major layout, 16x16 TLOAD:

   row 0  ┌─[##]─────────────────────────────────────────┐
   row 1  ├─[##]─────────────────────────────────────────┤
   row 2  ├─[##]─────────────────────────────────────────┤   16 short bursts
   ...    │   ...                                          │   stride = 1024 B
   row 15 ├─[##]─────────────────────────────────────────┤   BL     = 16 B
          └────────────────────────────────────────────────┘

Tile-contiguous layout (tile_shape = (16,16)), same 16x16 TLOAD:

          ┌────────────────────────────────────────────────┐
          │ [################################]              │   1 burst
          │  ^ 256 contiguous bytes for the tile             │   BL = 256 B
          └────────────────────────────────────────────────┘
```

## 3. When and Why This Works

The key invariant is:

> If, during the lifetime of the GM tensor, the program **only** accesses it via `TLOAD` / `TSTORE` whose access shape equals the declared `tile_shape` (and is aligned to the tile grid), then **every** transfer becomes a contiguous burst whose length equals `prod(tile_shape) * sizeof(dtype)`.

> Ruoyu: This invariant is too strong for the full system. It should say the GM address interval for the tile is contiguous. Whether the hardware transfer is one burst depends on PTOAS lowering, DMA limits, UB layout, alignment, and whether local layout transforms are inserted. Also, if the tensor is reused at multiple granularities, hierarchical macro-tile staging may be better than forcing the persistent GM layout to the smallest compute tile.

When this invariant holds, tile-contiguous layout delivers:

- **Maximal GM burst length** — the burst length naturally equals the tile byte size, and the programmer chooses `tile_shape` so this matches (or is a multiple of) the platform's preferred burst length (512 B on A2/A3, 128 B on A5).
- **Maximal NoC efficiency** — wide channels carry full-size transactions instead of being starved by short, strided requests.
- **L2 cache line utilization → 100 %** — every fetched cache line is fully consumed by the tile.
- **Fewer outstanding transactions** — reduces controller queue pressure and tail latency.

> Ruoyu: The L2 statement needs alignment and size conditions. For example, a 16x16 FP8 tile is 256 B, while the installed CANN simulator config reports a 512 B cache line for 910B and 950. A single 256 B tile cannot by itself guarantee 100% L2 cache-line utilization unless adjacent data in the same line is also consumed and the tile is cache-line aligned.

### Alternative / Complement: Hierarchical DMA and Local Layout Transforms

> Ruoyu: This proposal should explicitly account for PTOAS's hierarchical memory and layout-transform model. A typical optimized path can be:
>
> 1. Use `TLOAD`/DMA to move a larger GM macro tile into UB or another local buffer.
> 2. Use local transform operations such as `tmov`, `textract`, and `tinsert` to produce the exact compute tile layout needed by each layer.
> 3. Compute on the local tile layout.
> 4. Use `tinsert`/`tmov` and `TSTORE`/DMA to assemble and write the result.
>
> In this model, every layer can have its own local layout transform design. Persistent GM layout should not be the only place where layout is optimized. The proposal should describe when to prefer persistent `tile_shape` and when to prefer local macro-tile staging.
>
> `tmov` is a local tile move or layout conversion between tile domains or memory scopes. `textract` extracts a sub-tile/window from a larger local tile buffer into another tile buffer. `tinsert` is the inverse operation: it inserts a source tile/window into a destination tile buffer at a specified position, so several smaller computed tiles can be assembled into a larger local output tile before store.

### Caveats: When *not* to specify `tile_shape`

The optimization is only beneficial when the access pattern is congruent with the declared `tile_shape`. If the program also accesses the tensor with a *different* tile shape — for example a transposed or reshaped view, a strip along a single row, or a sub-tile smaller than `tile_shape` — the access reverts to short, strided bursts (often *worse* than the row-major default, because the addressing also becomes non-trivial).

Therefore:

- `tile_shape` shall only be specified when the dominant lifetime access pattern of the GM tensor matches it.
- The programmer must specify it **with discretion**, treating it as a performance contract.
- When in doubt, omit it — the compiler will fall back to row-major, and correctness is unaffected.

The compiler is **not** required to dynamically rewrite layouts based on observed access patterns. `tile_shape` is a programmer-driven hint, not an autotuner.

> Ruoyu: Even if the compiler is not an autotuner, it should still recognize and preserve opportunities for hierarchical DMA reuse. If the same GM neighborhood feeds multiple subtiles, a compiler pass or PTOAS lowering should be allowed to stage a macro tile and use local transforms, rather than requiring the programmer to encode every optimization as persistent GM layout.

## 4. Impact on the pypto Compiler, PTOAS, and Simpler Runtime

### 4.1 pypto Compiler

#### Tensor metadata

The pypto IR's GM tensor structure must carry an additional optional field:

| Field        | Type             | Meaning                                               |
|--------------|------------------|-------------------------------------------------------|
| `shape`      | tuple of int/Expr | Logical shape (unchanged).                            |
| `dtype`      | type             | Element type (unchanged).                             |
| `tile_shape` | tuple of int/Expr or `None` | Physical tile shape; `None` ⇒ row-major default. |

The field must be **preserved** through every IR pass and **propagated** to the back end so that PTOAS can see it.

> Ruoyu: The metadata should probably be a more explicit layout descriptor, not only `tile_shape`. At minimum it should distinguish `layout = row_major` from `layout = tile_contiguous(tile_shape=...)`. If hierarchical staging is part of the design, there may also be execution-local metadata such as macro tile shape, local layout, or transform plan. These should not be conflated with persistent GM byte layout.

#### Validation

When a tensor is declared with `tile_shape`, the compiler must verify:

1. `len(tile_shape) == len(shape)`.
2. `shape[i] % tile_shape[i] == 0` for every dimension `i`.
3. `prod(tile_shape) * sizeof(dtype)` is a sensible burst size (not below platform minimum, not above tile SRAM budget). This may be a soft check producing a warning rather than an error.

Violations of (1) or (2) are hard errors.

#### Layout-affecting operations: warn the programmer

Operations that re-interpret or reshape a tensor's logical structure can silently break the tile-contiguity assumption. When a tensor with a non-default `tile_shape` is the input to any of the following, the compiler shall emit a **warning**:

- `tensor.slice` / `pl.slice` — when the slice does not align to the tile grid, or when the slice extents are not multiples of `tile_shape`, the slice cannot inherit `tile_shape` and falls back to a sub-region with non-trivial stride.
- `tensor.shallow_reshape` — a metadata-only reshape that assumes a contiguous row-major layout. With a non-default `tile_shape`, shallow reshape is generally **invalid** and must be rejected (or downgraded to a warning + propagated `tile_shape = None`).
- `tensor.deep_reshape` — a copying reshape. The compiler should warn that the destination tensor's layout will be the default row-major unless the user explicitly supplies a new `tile_shape` for the destination.
- Any other tensor layout / view operation (`transpose`, `permute`, `view`, `expand`, ...).

Default behavior on warning: produce the result tensor with `tile_shape = None` (row-major) so that subsequent codegen is correct, even if no longer optimal. The warning message must clearly identify the source location and suggest either dropping `tile_shape`, aligning the slice, or rematerializing with a fresh `tile_shape`.

> Ruoyu: Falling back to `tile_shape = None` on views may hide a physical-layout mismatch. If the underlying bytes remain tile-contiguous, simply marking a view as row-major is not correct unless a real copy/rematerialization occurs. The compiler needs a clear distinction between "view with non-trivial physical layout" and "new row-major materialized tensor."

#### Codegen

When generating `TLOAD` / `TSTORE` for a tensor with non-default `tile_shape`, the compiler computes the GM offset of the requested tile from the **tile-grid coordinates**, not the default row-major offset formula. This information is forwarded to PTOAS via the tensor's metadata.

> Ruoyu: Codegen should also say how local layout transforms are selected. A GM offset formula is not enough: PTOAS must decide the DMA shape, UB destination layout, whether a `tmov`/`textract` is required after load, and whether multiple local tiles are assembled with `tinsert` before store.

### 4.2 PTOAS

PTOAS is responsible for emitting the physical instruction binary for `TLOAD`, `TSTORE`, and `TASSEMBLE`. Its requirements:

> Ruoyu: `TASSEMBLE` should probably not be introduced as a new required primitive. In current PTOAS terms, assembling a larger local tile from smaller pieces can be represented as multiple `tinsert` instructions. A `tinsert` writes a smaller source tile/window into a destination tile buffer at a selected offset or subregion. Therefore "TASSEMBLE" is just a higher-level pattern:
>
> ```text
> dst_macro_tile = empty/local output tile
> tinsert(subtile_0, dst_macro_tile, offset_0)
> tinsert(subtile_1, dst_macro_tile, offset_1)
> ...
> TSTORE(dst_macro_tile, GM)
> ```
>
> The design should remove `TASSEMBLE` or define it only as syntactic sugar/lowering shorthand for a sequence of `tinsert` operations.

1. **Accept and honor the `tile_shape` metadata** for every GM tensor argument.
2. **Be cognizant of the layout** when emitting addressing logic. For a non-default `tile_shape`, the byte address of element `(i0, i1, ...)` is:

   ```
   tile_idx = (i0 / tile_shape[0], i1 / tile_shape[1], ...)
   intra   = (i0 % tile_shape[0], i1 % tile_shape[1], ...)

   addr    = base
           + linearize_row_major(tile_idx, shape / tile_shape) * prod(tile_shape) * sizeof(dtype)
           + linearize_row_major(intra,    tile_shape)         * sizeof(dtype)
   ```

   For row-major (`tile_shape == shape`), this collapses to today's address formula.

3. **Generate physical instructions whose access pattern matches the layout.** When the `TLOAD` / `TSTORE` access shape equals `tile_shape` and is tile-aligned, PTOAS shall emit a single contiguous-burst instruction. When it does not match, PTOAS may emit either a strided sequence (correct but slow) or refuse and report an error, per the platform-specific code generator's policy.
4. **`TASSEMBLE`** (which composes a tensor in SRAM from multiple sub-tile transfers) must use the same layout-aware addressing so that the in-SRAM image is reconstructed correctly regardless of GM layout.

> Ruoyu: Requirement (3) is underspecified for existing hierarchical DMA. PTOAS already has a multi-level DMA model and local tile transforms. The lowering policy should include:
>
> - direct matched tile load/store from persistent tile-contiguous GM;
> - row-major GM macro-tile load into UB followed by `textract` into compute tiles;
> - local `tmov` between tile domains/layouts where the next layer expects a different tile layout;
> - local `tinsert` assembly before one larger `TSTORE`;
> - explicit cost/legality rules for when to choose each path.
>
> Requirement (4) should be replaced by a `tinsert`-based assembly rule. The layout-aware addressing belongs to the GM-facing load/store and macro-tile selection; once data is in local buffers, `textract`, `tmov`, and `tinsert` should describe the layer-by-layer layout transforms.

PTOAS is the lowest level that has to *act* on `tile_shape`. The pypto compiler decides the layout; PTOAS executes it.

> Ruoyu: PTOAS should not only "execute" a layout chosen by pypto. It also owns low-level choices about DMA loop shape, local buffer layout, and transform sequence. The proposal should give PTOAS enough freedom to use hierarchical DMA and local transforms even when persistent GM layout is row-major.

### 4.3 Simpler Runtime

The simpler distributed runtime is responsible for tensor lifetime tracking, dependency analysis (the tensormap), and argument marshalling. With respect to `tile_shape` it has minimal duties:

- **Tensormap / overlap analysis.** Continue to use only the logical `shape` (and offset/stride) to detect overlap and dependency between tensor regions. `tile_shape` does **not** alter logical extents and therefore does not affect overlap reasoning.
- **Data structure.** Carry `tile_shape` as an opaque field on the tensor descriptor and propagate it through the argument list to PTOAS-generated kernels and to debug/trace outputs.
- **Debug output.** When dumping a tensor descriptor (logs, traces, error messages), include `tile_shape` so that layout-related performance issues are inspectable.

At the time of this writing there is no other runtime use of `tile_shape`. Should a future pass need it (e.g., layout-aware scheduling, peer-to-peer transfer chunking), the field is already plumbed through.

> Ruoyu: Runtime cannot always treat `tile_shape` as opaque. For internal tensors this may be acceptable, but for host-visible tensors, distributed transfers, debug dumps, checkpointing, and ABI boundaries, the runtime must know whether bytes are row-major or tile-contiguous. Otherwise external producers/consumers may interpret the same GM allocation differently.

## Summary

> Ruoyu: Summary should be softened. `tile_shape` can improve matching tile accesses, but the full design should combine persistent GM layout with hierarchical DMA staging and local layout transforms. In particular, macro-tile `TLOAD` plus `textract`/`tmov`/`tinsert` may be the better path when multiple subtiles are reused from the same GM region.

| Component        | Action required                                                                                        |
|------------------|--------------------------------------------------------------------------------------------------------|
| Programmer       | Optionally declare `tile_shape` on a GM tensor when the dominant `TLOAD`/`TSTORE` access pattern is known. |
| pypto compiler   | Carry `tile_shape` in the tensor IR; validate divisibility; warn on layout-breaking ops; forward to PTOAS. |
| PTOAS            | Use `tile_shape` to compute GM addresses; emit single-burst instructions when access matches the tile.   |
| Simpler runtime  | Propagate `tile_shape` through descriptors and argument lists; ignore it for overlap/tensormap reasoning; surface it in debug output. |

When used correctly, `tile_shape` turns small-tile `TLOAD` / `TSTORE` traffic from many short, strided bursts into one long contiguous burst, restoring full GM bandwidth, NoC throughput, and L2 cache utilization — directly shortening kernel runtime on A2, A3, and A5.
