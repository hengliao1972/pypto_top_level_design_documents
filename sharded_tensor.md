# `sharded_tensor`: Symmetric, Rank-Partitioned Storage for pypto Tensors

## Status

This document describes an **experimental** design direction for a new value type, `sharded_tensor`, in pypto. It is written for **design review** by the pypto team. After the team approves the model and any adjustments to the details below, we will track implementation work to promote it to a **supported, full** feature in the type system, compiler, runtime, and tools.

## Overview

`sharded_tensor` is a **derived type** that **inherits** the semantic and metadata surface of the ordinary `tensor` type, and at the same time **inherits the symmetric shared-memory model** from the **open share memory** specification. Logically, a `sharded_tensor` represents a single global tensor. Physically, its **backing storage** is not monolithic on one node: it is **evenly split** across `rank_num` participants so that each rank holds **1 / rank_num** of the total byte size of the tensor, under the symmetric shared-memory contract.

Inherits:

| From | What carries over |
|------|-------------------|
| `tensor` | Logical shape, `dtype`, optional `tile_shape` and other tensor metadata, indexing semantics, and the programmer-visible tensor API (subject to rank-aware restrictions defined later). |
| `open_share_memory` (symmetric) | **Symmetric** share memory: every participating process sees the same **addressing contract** and naming of ranks; collective creation and access rules align with the open share memory spec. |

`sharded_tensor` is **not** a replacement for local GM tensors. It is an additional type for programs that are compiled and executed in a **multi-rank, symmetric shared-memory** environment.

## Design philosophy: sharded memory as pypto's cross-node programming paradigm

`sharded_tensor` plays **two** complementary roles in pypto:

- A **programming paradigm for cross-node data placement** — declaring, in one place, how a logically global tensor is distributed across the participating ranks, with `shape`, `tile_shape`, and `rank_shape` carrying the partitioning through the type system. One declaration replaces the conventional pattern of every rank manually allocating its own slice and hand-agreeing on offsets, strides, and synchronization points.
- A **memory management mechanism** — collective allocation in the scope ring buffer (§6.1), an `open_share_memory` join with the all-to-all global barrier at construction (§6.2), lifetime tracking, and (default) collective retirement (§6.3). Bugs of the form *"rank 3 freed early while rank 5 was still reading"* become **type-level** errors rather than runtime races.

Building on these two roles, **the direction this design recommends** is:

> **Use the memory management mechanism of `sharded_tensor` as the surface API for cross-node collective communication, instead of the traditional buffer-and-parameter style of NCCL-equivalent calls.**

Concretely, an `all_reduce` collective is **defined on top of `sharded_tensor`** and takes the **whole `sharded_tensor` as its argument**, rather than threading raw buffers, element counts, dtypes, communicator groups, and streams through every call site:

```python
# Sketch (illustrative; final API names are TBD)
ST.all_reduce(op=pl.ReduceOp.SUM)            # collective expressed against the typed object
ST.all_gather(target=YT)                     # YT is itself a sharded_tensor with the gathered partitioning
ST.reduce_scatter(op=pl.ReduceOp.SUM)
ST.all_to_all(target=YT)
ST.send(rank_id=k);  ST.recv(rank_id=k)      # PP-style point-to-point
```

The reader of `ST.all_reduce(op=pl.ReduceOp.SUM)` sees, **from the type alone**, what is being reduced, across whom (the world over which `ST` is sharded), and with what partitioning. Threading per-rank buffers, counts, communicator handles, and streams through every call site is replaced by a **single typed argument**. This materially improves the **clarity of program semantics** at every cross-node call site.

### The surface API and the implementation are independent concerns

The recommendation above is about the **shape of the surface API**, not about how the collective is **implemented** on the wire. The collective library underneath remains free to use **any** transport — ring or tree algorithms, hardware-collective engines, RDMA-style send / recv, or the direct-access path of §5.3 / §5.3.1 — whichever is fastest on the target platform. The typed surface and the high-performance implementation are independent concerns; one does not bind the other.

| Concern | Provided by |
|---------|-------------|
| **Definition and data-structure layer** — type, metadata (`shape` / `tile_shape` / `rank_shape`), per-rank allocation, lifetime barriers, addressing model, **typed surface for collectives** | `sharded_tensor` (this document) |
| **Implementation layer for cross-rank operations** — collectives, P2P, fused exchange-and-reduce, …; transport, algorithm, scheduling | A collective communication library (or libraries), free to pick **any** transport per operation, including but **not limited to** the direct-access path of §5.3 / §5.3.1 |

### Positioning relative to existing frameworks

pypto is **not** aiming to adapt to or mirror the cross-node API of existing open-source frameworks such as **vLLM**, **SGLang**, **Megatron-LM**, or **DeepSpeed**. Those frameworks express cross-rank traffic in a buffer-and-collective-call style for historical and practical reasons (NCCL semantics, PyTorch tensor model, accumulated kernel libraries). pypto's design goal in the **hierarchical runtime** is different: a **simpler, cleaner multi-node cluster programming paradigm** in which the programmer expresses intent more freely and more clearly at the source level, while the runtime and the collective library are free to deliver high execution efficiency at the implementation layer.

`sharded_tensor` is the **central piece** of that direction at the cross-node layer. The remaining uncertainty discussed in the next section is therefore about **implementation-level details** and **target-workload validation** — not about whether the direction itself is the right one for pypto.

## Why this is still experimental: implementation-level open questions

The strategic direction in the previous section (sharded memory as pypto's cross-node paradigm; collectives as typed methods on `sharded_tensor`) is the **working assumption** for this document. What is **still experimental** is the **implementation level**: the **exact surface API** for collectives over `sharded_tensor`, the **right concrete workload** to use as the validation target, and the **performance characteristics** of the direct-access lowering (§5.3 / §5.3.1) versus alternatives. The remainder of this section captures the relevant context — the existing cross-rank communication landscape (which pypto is **not** trying to mirror) and the candidate workloads where the direct-access path specifically pays off — so that the design review can decide on a concrete implementation plan.

### The dominant cross-rank pattern in mainstream frameworks is collective communication, not direct remote access

For context (not as a goal pypto is matching): the common parallelization schemes used in AI model training and serving — **DP** (data parallel), **TP** (tensor parallel), **PP** (pipeline parallel), **EP** (expert parallel for MoE), and **SP** (sequence parallel) — express cross-rank data movement as **explicit collective communication operations**, **not** as direct remote memory load / store or `TLOAD` / `TSTORE` against another rank's bytes:

| Scheme | What is split | Dominant cross-rank operation | Implemented as |
|--------|--------------|------------------------------|----------------|
| DP     | The mini-batch | gradient sync: **all-reduce** (or **reduce-scatter + all-gather** in ZeRO-1/2/3) | NCCL / equivalent collective library |
| TP     | Weight matrices (Megatron column-/row-parallel linears, attention heads) | **all-reduce** in forward / backward, **all-gather** for some layouts | NCCL / equivalent collective library |
| PP     | The layer stack | activations across stage boundary: **point-to-point send / recv** (1F1B, interleaved 1F1B) | NCCL P2P / equivalent |
| EP     | The set of experts in MoE | dispatch / combine: **all-to-all** | NCCL all-to-all / equivalent |
| SP     | The sequence dimension (combined with TP in Megatron-SP) | **all-gather** / **reduce-scatter** | NCCL / equivalent collective library |

In each case the cross-rank data movement is **collective by construction**: the schedule, the staging buffers, and the bandwidth model all assume a **single fused operation across (a subset of) ranks**, executed by an optimized library implementation. The compute kernels themselves do **not** dereference remote bytes — they read from and write to **local** buffers that the collective has already populated.

This pattern shapes pypto's **implementation freedom**: the typed `sharded_tensor` surface (Design philosophy section) does not preclude going through such a collective library, and on platforms / workloads where the library is the fastest path, the runtime should use it. pypto's **surface API** is deliberately different (typed methods on `sharded_tensor`, not raw buffer parameters), but its **implementation** is free to look like — or even literally call into — an existing collective library beneath the typed surface.

### When the direct-access path of §5.3 / §5.3.1 specifically helps

The typed surface of `sharded_tensor` (Design philosophy section) is admissible regardless of how the collective is implemented; in particular, an implementation that uses a hand-tuned collective library is fine. The **direct-access path** of §5.3 / §5.3.1 — CPU load / store or MTE `TLOAD` / `TSTORE` through a `ubmem import`-mapped base address, with a fallback to explicit `open_share_memory` API calls — is a **separate, narrower** capability that earns its weight only on workloads where collective formulation is awkward or insufficient. Plausible candidates that are **worth evaluating** but **not yet confirmed** include:

- **Irregular cross-rank lookups** that do not fit the all-to-all dispatch / combine pattern — e.g. routing schemes where the destination set is data-dependent at fine granularity.
- **Prefix-shared KV caches** in serving, when prefix bytes are physically resident on a rank that is not the consuming rank.
- **MoE expert-weight access** or other "fetch a small piece from a known rank" patterns where building a full all-to-all is wasteful.
- **Custom fused kernels** that interleave compute and remote-byte access in a way that no off-the-shelf collective expresses cleanly.

If one or more of these turns out to be a real, measurable pypto requirement, the direct-access path of §5.3 / §5.3.1 pays for itself in v1. If none do, that path can be **deprioritized** in v1 without invalidating the rest of `sharded_tensor`: the typed surface (Design philosophy section) remains valuable on its own, served by a conventional collective library underneath.

### Implication for the design and the implementation

Because the implementation-level details — the exact set of typed collectives, their precise semantics on partitioning transformations, and whether the v1 implementation includes the direct-access path — are still open, this document is intentionally **descriptive** of the **shape** of the feature (type system, lifecycle, access semantics, lowering, simpler-runtime support) rather than prescriptive about a rollout schedule. **Implementation work should not begin solely on the strength of this design**; a **concrete target workload** — narrow (direct cross-rank access) or broad (typed-abstraction ergonomics) — must accompany any decision to promote `sharded_tensor` from **experimental** to a supported, on-by-default feature.

## 1. Open share memory: ranks

In an **open share memory** system, every node that takes part in symmetric sharing is identified by:

| Name | Meaning |
|------|--------|
| `rank_id` | The identifier of this participating node, in `0 .. rank_num - 1` (exact inclusive range is defined by the open share memory spec; this document assumes zero-based contiguous ranks unless otherwise specified). |
| `rank_num` | The **total** number of sharing nodes in the partition. All sharded tensors in a given program region agree on the same `rank_num` (or a compatible subset model if the spec allows subgroups; **subgroups are out of scope** unless the team extends this document). |

Symmetric shared memory **coordinates** (allocation, mapping, and visibility) are defined by the open share memory layer. `sharded_tensor` **does not redefine** the low-level mechanics; it **binds** tensor metadata to that layer so the compiler and runtime can reason about **which rank owns which slice** of storage and how the **logical whole tensor** is reassembled for semantics and (where applicable) debugging or verification.

## 2. `sharded_tensor` vs ordinary `tensor`

A **host / ordinary `tensor`** (in the single-node or monolithic-storage sense) has a single contiguous (or strided) storage blob whose size is determined by `shape`, `dtype`, and layout (e.g. row-major or `tile_shape`-driven layout as in [Tensor `tile_shape`](tensor_layout.md)).

A **`sharded_tensor`** has the **same logical metadata** as a tensor: `shape`, `dtype`, optional `tile_shape` for GM tile layout where applicable, strides/device placement metadata as defined by the implementation, and so on. The **difference** is **where the bytes live**:

- **Total element count** (and total byte size for a given layout) is still determined by the **global** `shape` and `dtype` (and layout).
- **Physical storage** is **evenly** distributed: each of the `rank_num` ranks provides storage for **exactly** **1 / rank_num** of the **total** tensor size (in bytes), under the symmetric allocation policy.

**Uniform split assumption (design default).** The intended default is an **even byte-wise** partition. If a future need arises for **non-uniform** rank slices, that would be a separate design (not implied by this version of `sharded_tensor`).

**Relationship between global shape and per-rank footprint.** For a global shape `S = (s0, s1, ..., s_{d-1})` and element size `E`, the total size is `prod(S) * E` bytes. Each rank holds `prod(S) * E / rank_num` bytes. The **concrete mapping** from global indices to `(rank_id, local offset)` is a **separate, rank-ordering policy** (e.g. linear partition along the last dimension, block-cyclic, etc.); this document only **requires** that the partition be **symmetric in responsibility** (each rank **1 / rank_num**) and **documented** in a follow-on section once the team picks a default algorithm.

## 3. The `rank_shape` attribute

In addition to the standard tensor fields **`shape`** and optional **`tile_shape`**, a `sharded_tensor` carries **`rank_shape`**.

| Field | Role |
|-------|------|
| `shape` | The **global** logical shape of the entire sharded tensor. The union of all ranks' contributions **logically** covers this shape. |
| `tile_shape` | Optional, same meaning as for GM tensors: physical tile shape for tile-contiguous layout, when the backend uses it. See [Tensor `tile_shape`](tensor_layout.md). |
| `rank_shape` | The **per-rank logical shape** (or shape **fraction**) whose **product** × `sizeof(dtype)` × layout overhead matches the **1 / rank_num** storage fraction on that rank, **consistent** across ranks up to a **reordering of dimensions** that the spec may allow. In the simplest case, `rank_shape` is literally `shape` with one or more dimensions divided by `rank_num` (or a factor that yields an integer), so that `prod(rank_shape) * rank_num == prod(shape)` when a simple replication-free partition along one axis is used. |

**Intended meaning.** `rank_shape` answers: *"What is the **tensor-shaped footprint** of **my** piece on this `rank_id` in the same dimension order as `shape`?"* It is **not** a second global shape; it is a **per-rank slice description** in **shape** form, such that the **combination** of all ranks' stored pieces (under the chosen partition rule) **covers** the full `shape` without overlap.

**Example (illustrative, not normative for index mapping).** Global `shape = (1024, 1024)` and `rank_num = 4` might use `rank_shape = (256, 1024)` if the partition is along the first dimension, so that each rank stores a contiguous 256×1024 slab. Another policy might use `rank_shape = (1024, 256)` for a partition along the second dimension. The team must **fix one default** in the full specification to avoid silent divergence between compiler and runtime.

**Consistency rules (draft for review).**

1. `len(rank_shape) == len(shape)`.
2. For every dimension `i`, `shape[i] % rank_shape[i] == 0` **in the default uniform partition** (or the spec states how remainder ranks are handled if `shape` is not divisible; **remainder policy is open** until the team decides).
3. The **sum over ranks of the per-rank element count** must equal the **global** element count `prod(shape)`.

## 4. Inheritance and API surface (design sketch)

- **Type relationship:** `sharded_tensor` is a **subtype** (derivative class) of `tensor` in the type system, with **additional** constraints and **additional** metadata (`rank_id` context, `rank_num`, `rank_shape`, and a reference to symmetric shared-memory handle as required by `open_share_memory`).

- **Behavior:** Operations that only need **local** data (the rank's slice) can be defined naturally. Operations that need a **global** view may require **collective** or **runtime-assisted** materialization, or are **rejected** at compile time. The **exact** split of allowed ops is for the pypto design review to decide.

- **Interoperability with `tensor`:** Converting `tensor` ↔ `sharded_tensor` may require **copy** or **collective** placement; a **view** of one rank’s slice as a local `tensor` is a common pattern if the open share memory model exposes a local base pointer and length for each rank.

## 5. Access semantics: subtensor extraction is keyed by `rank_id`

A `sharded_tensor` is **always accessed with respect to a specific `rank_id`**. The global tensor is logically one object, but its **bytes are partitioned across ranks**, and any operation that **selects, reshapes, or otherwise extracts a sub-region** must say **whose share** it is talking about — with one shorthand: when a rank accesses **its own share**, `rank_id` may be omitted (it defaults to `self_rank`), in which case the slice is treated as a **normal `tensor`** (see §5.1).

### 5.1 Subtensor extraction takes an explicit `rank_id` argument

Every layout / extraction op on a `sharded_tensor` carries an **additional `rank_id`** argument that is **not** present on the corresponding op for an ordinary `tensor`. This applies to (at minimum):

- `view`
- `reshape` (both shallow and deep variants — see [Tensor `tile_shape`](tensor_layout.md))
- `slice`, indexing, and any other subtensor extraction op
- `transpose`, `permute`, `expand`, and other layout/view operations the surface API exposes
- conversion to a local `tensor` view of the slice

The `rank_id` argument **selects which rank's 1 / rank_num share is being operated on**. The op produces a result whose semantics are defined **only with respect to that rank's slice**; the operation does **not** implicitly fan out across all ranks.

```python
# Sketch (illustrative; final API names are TBD)
ST = pl.sharded_tensor(shape=(M, N), dtype=pl.fp16, rank_shape=(M / rank_num, N))

# Extract a (16, 16) tile from rank 3's slice — explicit rank_id is required.
local_tile = ST.view(rank_id=3, region=((r0, r0+16), (c0, c0+16)))

# Reshape rank 0's slice — again, rank_id is part of the call.
local_view = ST.reshape(rank_id=0, new_shape=(...))
```

**Local-share shorthand: `rank_id` may be omitted when a rank accesses its own share.** When a rank accesses **its own slice** of the `sharded_tensor` (i.e. `rank_id == self_rank`), the `rank_id` argument **may be omitted**; equivalently, when omitted, `rank_id` is **defaulted to `self_rank`**. Under this shorthand, the result is, from the compiler's and the kernel's perspective, **just a normal `tensor`** view of the local share: the standard tensor APIs (`view`, `reshape`, `slice`, `TLOAD` / `TSTORE`, ...) apply with their **ordinary** signatures, the same layout / `tile_shape` rules apply, and **no remote transport or symmetric-memory machinery is involved**. This is what makes consuming one's own share cheap inside an InCore kernel: the compiler treats the local slice indistinguishably from a normal local tensor allocated at this rank.

```python
# Local-share shorthand — rank_id omitted; result is a normal tensor view.
local = ST.view(region=((r0, r0+16), (c0, c0+16)))           # implicitly rank_id=self_rank

# Equivalent explicit form.
local = ST.view(rank_id=self_rank, region=((r0, r0+16), (c0, c0+16)))
```

### 5.2 One rank at a time

The `rank_id`-keyed model (with the own-share shorthand of §5.1) is the **access discipline** of `sharded_tensor`: a single call site **operates on exactly one rank's slice** — either the local rank (when `rank_id` is omitted or equals `self_rank`) or one specific remote rank. There is **no fused multi-rank extraction primitive** in this design; if a kernel needs data from several ranks, it issues several extractions, **each** with its own `rank_id` (or each implicitly local).

This restriction is deliberate:

- It keeps the **addressing model explicit**: the programmer (and the compiler) always know **which** physical bytes are being read or written.
- It avoids implicit collective traffic: a one-line `view` cannot accidentally generate an all-ranks broadcast or gather.
- It maps **one-to-one** onto the underlying open-share-memory access primitives (described next), which are themselves **point-to-point** between the issuing rank and one remote rank.

### 5.3 Lowering: local stays as a normal tensor, remote goes through `open_share_memory` APIs

The lowering of an extraction op depends on whether `rank_id` resolves to the **local** rank or to a **remote** rank.

**Local case (`rank_id == self_rank`, including the omitted-`rank_id` shorthand).** No transport is involved. The compiler treats the result as a **normal `tensor`** view of the local share, lowers it via the **standard** tensor codegen path (load/store, `TLOAD` / `TSTORE`, and any tile-shape-driven addressing per [Tensor `tile_shape`](tensor_layout.md)), and applies the same layout / aliasing rules as for any other local tensor. There is **no `open_share_memory` call** in the emitted code for this path.

**Remote case (`rank_id != self_rank`).** The pypto compiler **resolves** the op against the `sharded_tensor`'s metadata (`shape`, `tile_shape`, `rank_shape`, partition rule) to determine **which bytes on rank `rank_id`** are being addressed, and **lowers** the extraction to a sequence of low-level **`open_share_memory` API calls**. These come in two flavors:

- **Synchronous** API calls block the issuing rank until the requested bytes are available locally (read) or until the remote write has been observably delivered (write).
- **Asynchronous** API calls return a handle / event that can be polled or awaited later, allowing the issuing rank to overlap remote transfer with local computation. The simpler runtime is responsible for scheduling and completion-tracking these handles, the same way it does for in-flight DMAs today.

The choice between the synchronous and asynchronous flavor is made by the compiler based on the surrounding schedule (or by the user, if the surface API exposes the choice). Either flavor uses byte addresses derived from `shape`, `tile_shape`, `rank_shape`, and `rank_id`, and either flavor is **legal only on program paths where the construction barrier (§6.2) has completed**, so every dereferenced byte is guaranteed to be published in the open-share-memory address space.

Operations that do **not** move data — pure metadata reshapes that do not change byte layout, for example — may be lowered to **no transfer at all**, just a metadata update on rank `rank_id`'s slice. Whether a given reshape is metadata-only on a `sharded_tensor` follows the same rules as for an ordinary GM tensor (see the warnings in [Tensor `tile_shape`](tensor_layout.md) §4.1 about layout-affecting operations), with the additional constraint that the metadata change is scoped to **rank `rank_id`'s** slice only.

#### 5.3.1 Preferred embodiment: UB `ubmem export` / `ubmem import` of remote shares at setup time

The Unified Bus (UB) supports **remote memory export** and **remote memory import**: a process can **export** a region of its address space, and another process can **import** that exported region into its own address space, getting back a **local virtual base address** that, when accessed, transparently routes the traffic to the exporter's bytes over UB.

In the **preferred embodiment** of `sharded_tensor` on UB-enabled platforms, the simpler runtime exploits this **at setup time** — i.e. as part of the open-share-memory join in §6.2:

1. **Each rank exports its local share** of the `sharded_tensor` to UB (`ubmem export`).
2. **Each rank imports** every other rank's exported share (`ubmem import`), receiving **one mapped local base address per remote rank**.
3. After the construction barrier returns, every rank holds a **per-rank local pointer** into every rank's slice: its own share by direct local allocation, every remote share by UB import mapping.

With these mappings in hand, **remote access does not need to go through an explicit `open_share_memory` API call at every site**: it can be performed directly using the **mapped local base address** for the target rank. Concretely:

- **Direct CPU load / store.** A CPU on the issuing rank can dereference the mapped pointer to read or write rank `k`'s share like ordinary local memory; UB transparently carries the traffic.
- **MTE access (`TLOAD` / `TSTORE`).** The MTE / DMA engine can be programmed with `TLOAD` / `TSTORE` against the **mapped local base address** for rank `k` (plus the offset computed from `shape`, `tile_shape`, `rank_shape`, and the in-slice index) and bulk-transfer remote bytes the same way it bulk-transfers local GM bytes.

The compiler's lowering for the **remote case** therefore **prefers** to emit **direct addressed access** through the imported mapping when the platform offers it, and **falls back** to explicit `open_share_memory` API calls (sync or async, as above) when a UB-style import mapping is not available. The **user-facing semantics** of the extraction op are **identical** in either case; only the generated code differs.

This embodiment is the basis on which §8.1 (simpler runtime over Unified bus) is built; it also gives a uniform low-level model:

| `rank_id` | What the compiler emits |
|-----------|--------------------------|
| omitted / `self_rank` | normal tensor codegen against the local allocation |
| remote rank, UB available | normal load/store or `TLOAD` / `TSTORE` against the **`ubmem import`-mapped** local base address for that rank |
| remote rank, UB not available | explicit synchronous or asynchronous `open_share_memory` API calls |

### 5.4 What this implies for the user

- For **own-share** access, the user **may omit** `rank_id` (or pass `rank_id = self_rank` explicitly); the compiler treats the result as a **normal `tensor`** and emits ordinary local-tensor code (no remote transport, no symmetric-memory machinery).
- For **remote** access, the user **must** supply `rank_id` of the remote rank. Forgetting `rank_id` while addressing a remote slice is a **type / API error**, caught by the compiler. A single call still operates on **exactly one rank's slice** (§5.2); to fetch from multiple ranks, issue multiple extractions.
- The user can **read or write any rank's slice** (subject to the program's higher-level synchronization and memory-model rules); the choice between **synchronous** and **asynchronous** lowering for remote access can be left to the compiler / runtime or, where the surface API exposes it, expressed by the user.
- The **cost** of the extraction depends on locality: own-share is free (local memory); remote is a UB-mediated access (direct load/store or `TLOAD` / `TSTORE` through the `ubmem import` mapping in the preferred embodiment, or an explicit sync/async `open_share_memory` API call when UB import/export is not available).

## 6. Allocation, scope, and lifecycle (collective semantics)

### 6.1 Where it is allocated: same as a normal `tensor`

A normal `tensor` (other than externally-referenced tensor memory, which is brought in by reference and **not** allocated by pypto) is **defined and allocated as a local variable** of an **orchestrator** function or an **incore** function. These tensors are placed in the **ring hierarchy** according to the **grammatical scope** at which they are introduced — see [`machine_hierarchy_and_function_hierarchy.md`](machine_hierarchy_and_function_hierarchy.md) (scope depth `d`, `task_ring[d]` / `buffer_ring[d]`, scope-token reclamation), and the runtime details in the runtime design documents in this directory.

`sharded_tensor` is allocated and defined in the **same fashion**: it is introduced as a local variable inside the relevant orchestrator / incore scope, and its **storage slot** is taken from the **scope's ring buffer** at that scope depth, just like a normal `tensor`. The **only** difference at allocation time is **what** the slot holds: the sharded tensor's storage is the rank's **1 / rank_num** local share, and the metadata also carries the `open_share_memory` symmetric-region handle, `rank_id`, `rank_num`, and `rank_shape`.

This means:

- The same scope rules govern when the variable is **in scope** (visible) and when its slot is **eligible** for reclamation (scope exit, `pl.free`, end of `ref_count` etc., as in the ring-buffer model).
- The tensormap / dependency analysis treats the sharded tensor like any other tensor for **local** producer / consumer reasoning at this rank — the **collective** aspects below add an extra synchronization on top, but **do not** replace the scope/ring lifetime.

### 6.2 Construction: open_share_memory join + global all-to-all sync (blocking)

After a `sharded_tensor` is **defined** at this rank (its local slice has been carved out of the scope ring buffer and its symmetric-memory metadata has been initialized), it must **invoke the `open_share_memory` API** to **join** all ranks together for this tensor. This step:

1. Registers / publishes the rank's local share of the symmetric region with the open-share-memory layer so that **every other rank** can address it using the symmetric addressing contract. **In the preferred UB embodiment (§5.3.1), this step is implemented as a `ubmem export` of the local share and a corresponding `ubmem import` of every other rank's exported share**, so that — once the barrier in step 2 returns — each rank holds a **mapped local base address per remote rank** for direct CPU load/store and MTE `TLOAD` / `TSTORE` access. On platforms without UB import/export, this step degrades to a plain registration with the open-share-memory layer; access is then through explicit sync/async `open_share_memory` API calls.
2. Performs a **global synchronization** — modeled as an **all-to-all** style barrier across **all `rank_num` ranks** — in which every rank announces "**I have finished setting up (and, where applicable, importing the remote shares of) this `sharded_tensor`**" and waits until **every** other rank has done the same.
3. Is **blocking**: the constructing program path on each rank does **not** proceed past the construction point until the synchronization is complete. Only after the sync returns does the `sharded_tensor` become **fully ready**: from that moment on, **any rank** is permitted to issue accesses (local or remote-via-symmetric-memory) to **any** part of the tensor under the established addressing contract.

**Why a global, blocking sync is required.** Until every rank has finished its setup, a remote access from rank A into rank B's slice can land on uninitialized / unmapped / not-yet-published memory. A weaker (e.g. pairwise, or non-blocking) synchronization would expose a race window in which the symmetric addressing contract holds at the language level but does not hold at the physical level. The all-to-all barrier closes that window; the **blocking** semantics make the readiness invariant **lexically observable** in the program — once construction returns, the tensor is usable globally, period.

**Where the sync sits relative to the ring buffer.** The local **slot** is allocated **before** the sync (so each rank has something to publish). The slot's **lifetime** in the ring is governed by the ordinary scope/ring rules; the sync only governs **when the global tensor object is observably ready**. Equivalently: the ring has the storage, and the sync converts it from "private byte region at rank i" to "byte region i of a globally addressable sharded tensor".

### 6.3 Retirement: collective vs unilateral (this is genuinely worth deciding now)

**The simplest model** is symmetric: just as construction is global, **retirement is also a global synchronization** — every rank reaches the end-of-life point for the `sharded_tensor` (scope exit, explicit `pl.free`, or equivalent), participates in a barrier, and **all ranks free their share at the same logical instant**. After the barrier, no rank may legally reference the tensor any longer. This mirrors the construction step and gives the cleanest invariant:

> **A `sharded_tensor` is either fully alive at every rank, or fully dead at every rank.**

**However, the question of whether retirement truly needs a global sync deserves explicit review.** A blocking global retirement is not free, and adding it indiscriminately can complicate error handling. We list both directions for the team to choose.

**Arguments for keeping retirement collective (default in this design):**

- **Symmetry with construction.** Construction is collective and blocking; retirement being collective makes the lifetime model uniform and easy to teach.
- **No "use-after-free across ranks" race.** If rank A keeps the tensor alive for one more access while rank B has already torn down its share, A's remote access through the symmetric-memory contract would target dead memory. A collective barrier removes this hazard by construction.
- **Simple reclamation of the symmetric region.** The open-share-memory handle and the per-rank slot can be reclaimed as a single coordinated event, without requiring a per-rank "is anyone else still using my share?" protocol.
- **Easier reasoning for the runtime / tensormap.** Lifetime is a single global event, not a per-rank distributed garbage collection.

**Arguments against / cases where it is overkill (worth thinking through):**

- **Cost on the critical path.** A blocking global barrier at every retirement, when many `sharded_tensor`s are short-lived and locally-only consumed, is potentially expensive.
- **Error handling complexity.** If one rank fails between construction and retirement (e.g. throws, aborts, or hits a device error), a collective retirement barrier becomes a **distributed cleanup** problem: surviving ranks may block forever waiting for a barrier the failed rank will never enter. Solving this **correctly** typically requires timeouts, fault-tolerant barriers, fencing, or an explicit "kill the whole symmetric region on any rank's failure" policy. Avoiding the collective retirement also avoids needing to specify all of that.
- **Locally-only access patterns.** If a `sharded_tensor` is **provably** never accessed remotely after a certain program point (e.g. a write-once, locally-read tensor), retirement could in principle be local — but proving "no rank will reach across" in the general case is hard.

**Recommended position for review (not a final decision).** Adopt **collective, blocking retirement** as the default — its symmetry with construction is the largest source of correctness simplification — but **explicitly track error handling** in the design (Section 9) so the team knows the cost. If profiling later shows the global retirement is hot, a follow-up extension can introduce an opt-in "local retirement" mode for tensors with statically-proved no remote access.

### 6.4 Mechanism summary

| Step | What happens locally (per rank) | Global synchronization |
|------|----------------------------------|------------------------|
| Allocate | Carve the **1 / rank_num** local share from the scope's ring buffer at the current scope depth `d`; build local metadata (`shape`, `tile_shape`, `rank_shape`, `rank_id`, `rank_num`, open-share-memory handle). | None yet. |
| Open share memory join | Publish the local share via the `open_share_memory` API. | **All-to-all global barrier**: every rank reports "my share is ready" and waits. **Blocking**; on return, the sharded tensor is **fully ready globally**. |
| Use | Own-share access uses **normal tensor codegen** (no `rank_id` needed in the surface API). Remote access (`rank_id != self_rank`) uses, in the **preferred UB embodiment**, **direct load/store or `TLOAD` / `TSTORE` against the `ubmem import`-mapped base address** for that rank; otherwise, **synchronous or asynchronous `open_share_memory` API calls**. | None per access (any cross-rank ordering is a regular memory-model concern, not part of construction). |
| Retire | Drop scope token / `ref_count` reaches the retirement condition; release the local slot back to the ring. | **Collective barrier (default)**: every rank reaches retirement; after the barrier, the symmetric region's local share is reclaimed on every rank in lockstep. |

## 7. Coexistence with normal and tile-consecutive tensors (programming paradigm)

`sharded_tensor` is **additive**. It does **not** replace, deprecate, or restrict the existing tensor types. The pypto programming paradigm **must continue to support** the definition and allocation of:

- **Normal `tensor`** — the ordinary local tensor (row-major default layout).
- **Tile-consecutive tensors** — tensors with an explicit `tile_shape` for tile-contiguous physical layout, as defined in [Tensor `tile_shape`](tensor_layout.md).

…**at every layer of the pypto runtime hierarchy**. Concretely (using the level taxonomy in [`machine_hierarchy_and_function_hierarchy.md`](machine_hierarchy_and_function_hierarchy.md)):

| Layer | Normal `tensor` and tile-consecutive `tensor` allowed? |
|-------|--------------------------------------------------------|
| Host / orchestrator scope | Yes |
| Cluster-level scope | Yes |
| InCore-level scope (and AIC / AIV functions derived from it) | Yes |
| Any other current or future hierarchy layer | Yes |

**Locality of these tensors.** Normal `tensor`s and tile-consecutive `tensor`s are **allocated locally** to the layer / process they are defined in. They have **no ability to be shared across nodes**: their bytes live on a single node, their addresses are not published into the open-share-memory address space, and they are **not** subject to the construction / retirement barriers in §6.

**Relationship to `sharded_tensor`.** The cross-node sharing capability is **exclusive to `sharded_tensor`**. If a program needs cross-node addressable storage, it must use `sharded_tensor` and pay the collective construction / retirement cost; otherwise, the existing local tensor types remain the default and are fully available without any change in semantics or cost.

This separation keeps the local tensor pipeline (allocation, reclamation via the ring buffer at scope depth `d`, codegen for `TLOAD` / `TSTORE`) **untouched** by the introduction of `sharded_tensor`. The compiler tells them apart by **type**, not by inferring intent from access patterns.

## 8. Impact (placeholder for follow-on work)

After the design is accepted, the following areas will need coordinated updates (mirroring the structure in [Tensor `tile_shape`](tensor_layout.md)):

- **Type system and IR** — new tensor variant with `rank_shape` and symmetric-memory linkage; `rank_id` argument added to extraction / layout ops on `sharded_tensor` (§5).
- **Compiler** — legality of ops on `sharded_tensor`, propagation of `rank_shape` and global `shape`, codegen for addressing within the local slice, and **lowering of `rank_id`-keyed extraction ops to open-share-memory API calls** (see below).
- **Runtime (simpler runtime)** — see dedicated subsection below.
- **PTOAS / device code** — as needed, if `sharded_tensor` is visible in kernel arguments or lowered device buffers.

### 8.1 Simpler runtime: open shared-memory over Unified bus

The simpler runtime is the natural home for `sharded_tensor`'s underlying mechanics:

- **Storage and addressability.** Implement `sharded_tensor`'s rank-partitioned storage **using the open shared-memory layer over the Unified bus**. Each rank's local share is registered with the open-share-memory layer at construction time; remote ranks address it through the open-share-memory address space.
- **Preferred UB embodiment: `ubmem` export / import at setup (§5.3.1).** During construction (§6.2), each rank `ubmem export`s its local share and `ubmem import`s every other rank's exported share, ending up with **one mapped local base address per remote rank**. Remote access then uses **direct CPU load/store** or **MTE `TLOAD` / `TSTORE`** against the mapped base — **no per-access `open_share_memory` API call is required**. This embodiment is preferred whenever UB export/import is available; the runtime must publish the per-remote-rank mapped base addresses to the compiler / kernel side so codegen can use them.
- **Construction and retirement collectives.** Provide the **all-to-all blocking barrier** at construction (§6.2) and the **collective retirement barrier** at the retirement event (§6.3, default), riding the same transport.
- **Extraction lowering target.** Expose the **low-level `open_share_memory` API** in **synchronous and asynchronous** flavors (point-to-point copy from / to a remote rank's published region) for platforms where the UB import-mapping path is not available, plus local-pointer access when `rank_id == self_rank`. The pypto compiler picks between (a) direct load/store or `TLOAD` / `TSTORE` against the imported mapping (preferred embodiment) and (b) explicit `open_share_memory` API calls (fallback) when lowering `view` / `reshape` / `slice` / etc. on a `sharded_tensor` (§5.3). Byte addresses are derived from `shape`, `tile_shape`, `rank_shape`, and `rank_id` regardless of which path is taken.
- **Tensormap interaction.** Continue to use the **logical** tensor extents (and per-rank slice extents from `rank_shape`) for overlap and dependency reasoning at this rank; the **collective** events (construction / retirement barriers) appear as additional happens-before edges in the schedule but do not change overlap analysis on the local tensor side.
- **Coexistence with local tensors.** The simpler runtime **must** preserve today's allocation / reclamation paths for normal `tensor`s and tile-consecutive `tensor`s at every hierarchy layer (§7). The shared-memory machinery is invoked **only** for `sharded_tensor` instances.

## 9. Open questions (for design review)

1. **Default partition** — one-dimensional split vs multi-dimensional, and divisibility when `prod(shape) % rank_num != 0`.
2. **`tile_shape` interaction** — can `rank_shape` and `tile_shape` be specified together with a **single** clear layout, or do we require `rank_shape` to be tile-aligned in all dimensions when `tile_shape` is set?
3. **Subgroup / partial participation** — whether `sharded_tensor` is allowed only in **world**-wide `rank_num` or in MPI-style subgroups.
4. **Naming** — current proposal is `sharded_tensor`; alternatives previously considered include `symmetric_tensor`, `shared_tensor`, and `distributed_tensor`. Final naming should align with the **open share memory** document’s terminology.
5. **Is a global sync at retirement truly necessary?** See Section 6.3. The default proposal is **yes** (symmetry with construction, no cross-rank use-after-free), but the cost on the critical path and the **distributed error-handling** implications (what happens to a collective retirement barrier when one rank has already failed?) are non-trivial. Possible alternatives the team should rule on:
   - **(A)** Collective, blocking retirement (default proposal).
   - **(B)** Collective retirement, but with a defined timeout / fault model (requires specifying a fault-tolerant barrier and a failure policy for the symmetric region).
   - **(C)** Local retirement permitted only for tensors statically proven to have **no remote access** after the retirement point; collective retirement otherwise.
6. **Failure semantics** — independently of (5), what happens if any rank fails **between** construction sync and retirement? Options span "abort the whole job", "tear down the symmetric region globally", or "fence the failed rank and continue at remaining ranks (advanced; almost certainly out of scope for the experimental version)".
7. **Sync primitive** — whether to mandate an actual **all-to-all** at construction or accept any **all-ranks barrier with publication** that the open-share-memory layer offers, as long as it provides the same readiness guarantee. The user-visible contract is the same; the implementation cost is not.
8. **Concrete target workload (the experimental gate).** See **"Why this is still experimental"** near the top of this document. Before promoting `sharded_tensor` past the experimental stage, the team should identify **at least one concrete pypto workload** for which a typed, lifetime-managed `sharded_tensor` abstraction yields a measurable improvement — either:
   - **(narrow case)** a workload that benefits from the **direct cross-rank access** path of §5.3 / §5.3.1, where expressing the access as a collective is awkward or insufficient; **or**
   - **(broad case)** a workload that benefits from the **typed definitional / lifetime-management** layer of `sharded_tensor` (per the *Design philosophy* section), even when its collectives are still implemented by a hand-tuned library that does **not** use direct remote load / store or `TLOAD` / `TSTORE`.

   Without such a workload, the recommendation is to **keep this feature experimental** (or defer it).
9. **Exact surface API for collectives on `sharded_tensor`.** The *Design philosophy* section settles the **direction** — collectives are exposed as typed methods or functions taking `sharded_tensor` arguments. What remains for the design review:
   - **The v1 collective set.** Which of `all_reduce`, `all_gather`, `reduce_scatter`, `all_to_all`, `broadcast`, `scatter`, `gather`, P2P `send` / `recv`, and any pypto-specific composites (e.g., fused exchange-and-reduce) ship in v1?
   - **Method names and signatures.** `ST.all_reduce(op=...)` style methods on `sharded_tensor`, free functions taking `sharded_tensor` arguments, or both? Naming should be consistent with the rest of pypto's typed-API surface.
   - **Typing of partitioning transformations.** When a collective changes the partitioning (e.g., `all_gather` produces a fully replicated or differently sharded result; `reduce_scatter` from a replicated input produces a sharded result), what is the **return type**? Should the result be another `sharded_tensor` with a different `rank_shape`, a normal `tensor` (when fully replicated and locally addressable), or both depending on call form?
   - **Interaction with the construction / retirement barriers of §6.** Whether each collective is itself a synchronization point, and whether multiple collectives over the same `sharded_tensor` need any extra fencing beyond what the construction barrier already provides.
   - **Implementation backing.** Whether v1 implements collectives over the §5.3 / §5.3.1 direct-access path, over a conventional collective library, or both with a runtime-selectable backend.

## Summary

| Concept | Description |
|--------|-------------|
| `sharded_tensor` | Derived from `tensor`, with symmetric open-share-memory **partitioned** backing storage. |
| `rank_id` / `rank_num` | Open share memory: which node, how many nodes. |
| Storage per rank | **1 / rank_num** of total tensor bytes (even split by default). |
| `shape` / `tile_shape` / `rank_shape` | Global logical shape, optional physical tile shape, and **per-rank** shape of this rank’s stored fraction (together covering the full tensor). |
| Allocation | Same as a normal `tensor`: local variable of an orchestrator / incore function; placed in the **scope ring buffer** at the relevant scope depth. |
| Construction sync | After local setup, invoke `open_share_memory` API; **global all-to-all blocking barrier** across all `rank_num` ranks; tensor becomes **fully ready globally** only after the barrier returns. |
| Retirement sync | Default: **collective, blocking** global barrier so every rank releases its share at the same logical instant — under explicit review (Section 6.3 / Q5) for cost and error-handling complexity. |
| Access semantics | Extraction / layout ops (`view`, `reshape`, `slice`, …) take a **`rank_id`** argument and operate on **one rank's slice at a time**. **`rank_id` may be omitted for own-share access** — the local slice is then handled as a **normal `tensor`**. |
| Lowering | Local case: normal tensor codegen. Remote case: in the **preferred UB embodiment**, direct CPU load/store or MTE `TLOAD` / `TSTORE` against the **`ubmem import`-mapped** local base address of the target rank (set up at construction time); fallback on non-UB platforms is **synchronous or asynchronous `open_share_memory` API calls**. |
| Coexistence | Normal `tensor`s and **tile-consecutive** `tensor`s remain definable / allocatable at **every** layer of the pypto runtime hierarchy; they are local-only and cannot be shared across nodes. |
| Applicability | **Direction set; implementation experimental.** `sharded_tensor` is positioned as pypto's **cross-node programming paradigm** at the hierarchical-runtime layer: a typed declaration of cross-node placement plus collective memory management, *and* the **typed surface for cross-node collective communication** (e.g., `ST.all_reduce(op=...)`). pypto deliberately does **not** mirror the buffer-and-collective-call API of vLLM / SGLang / Megatron-LM / DeepSpeed; the implementation underneath, however, is free to use any transport (collective library, hardware engine, RDMA, or the direct-access path of §5.3 / §5.3.1). Promotion past experimental requires settling the v1 collective surface (Q9) and a concrete target workload — narrow case (direct cross-rank access) or broad case (typed-abstraction ergonomics). See **Design philosophy**, **Why this is still experimental**, and Open questions Q8 / Q9. |
| **Status** | **Experimental** design; **not** a committed product feature until the pypto team approves, a concrete target workload is identified, and implementation planning completes. |
