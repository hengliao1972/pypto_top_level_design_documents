# Simpler Distributed Runtime: HostWorker / DistWorker Implementation Design

本文档描述 `simpler` 仓库中分布式运行时（Phase 2: HostWorker）的实现设计，涵盖 L3 层级的 DistWorker 调度引擎、HostSubWorker fork+shm 并行执行、统一 Worker 接口，以及使用方式。

本文档是对 `simpler/.docs/` 中 `UNIFIED_RUNTIME_PLAN.md`、`PHASE2_HOST_WORKER.md`、`HOSTSUB_FORK_SHM_DESIGN.md`、`PLATFORM_SERVICES_DESIGN.md` 的综合整理，面向上层（pypto / linqu）开发者理解底层分布式执行能力。

---

## 1. 背景与定位

### 1.1 层级定义

| Level | 资源 | 编排 | 实现 |
|-------|------|------|------|
| L1 | 单个 AICore (AIC/AIV) | 无 | kernel binary，由 L2 调度 |
| L2 | 单个 Chip (含 N 个 AICore) | C++ orch，跑在 AICPU | **ChipWorker**（已完成） |
| L3 | 单个 Host (含 M 个 Chip) | Python orch，跑在 host CPU | **HostWorker / DistWorker**（Phase 2） |
| L4+ | 多 Host | 同构扩展 | 未来 |

### 1.2 核心设计原则

- **与 L2 同构**：L3 复用 L2 的全部执行模型（scope + ringbuffer + tensormap + submit），不发明新概念
- **Worker 模型统一**：每层 Worker 对外接口都是 `run(task)`（阻塞），与 ChipWorker 完全同构
- **不修改 simpler**：所有 L3+ 设计在 simpler 的 L0-L2 之上构建，不改动已有 L2 代码

### 1.3 实施进度

| Phase | 状态 | 说明 |
|-------|------|------|
| Phase 0: Pre-built Runtime Binaries | ✅ 完成 | PR #386 |
| Phase 0.5: TaskArgs 类型重构 | ✅ 完成 | PR #391, #392, #395 |
| Phase 1a: Callable 类型 | ✅ 完成 | PR #408, #413 |
| Phase 1b: ChipWorker | ✅ 完成 | feat/chip-worker 分支 |
| **Phase 2: HostWorker (DistWorker)** | **⏳ 未开始** | 本文档重点 |

---

## 2. 统一 Worker 接口

### 2.1 IWorker 接口

所有 Worker 实现统一接口，对上层调用方完全透明：

```cpp
class IWorker {
 public:
    virtual ~IWorker() = default;
    // 在 worker 自己的工作线程中被调用，阻塞直到任务完成
    virtual void run(const WorkerPayload& payload) = 0;
};
```

### 2.2 Worker 层次结构

```
IWorker (接口)
  run(payload)   ← 在 worker 自己的工作线程中阻塞执行

  ├── ChipWorker     → L2 硬件。run() = C API 直接阻塞
  ├── SubWorker      → fork/shm Python 函数。run() = 写 mailbox + 自轮询到 TASK_DONE
  └── DistWorker     → 任意层级节点。run() = execute(host_task) 内部 orch + drain
                       int level_（3=L3, 4=L4, ...）
                       L3 实例：sub_workers = [ChipWorker×N, SubWorker×M]
                       L4 实例：sub_workers = [DistWorker(level=3)×K, SubWorker×M]
```

每个 IWorker 实例有自己的工作线程。Scheduler 仅做路由，不执行 `run()`。

### 2.3 各 Worker 的 run() 实现

```
ChipWorker::run(payload):
    chip_worker_impl.run(callable, args, config)   ← C API 本身阻塞

SubWorker::run(payload):
    write_mailbox(TASK_READY, callable_id)          ← dispatch 给 fork 子进程
    while (read_mailbox() != TASK_DONE) sleep(50us) ← 自轮询，阻塞本工作线程
    write_mailbox(IDLE)

DistWorker::run(payload):                           ← 当作 L4 sub-worker 时
    execute(host_task)                              ← 内部 orch + drain，阻塞
```

---

## 3. Phase 2 架构：DistWorker 调度引擎

### 3.1 组件总览

| 组件 | 运行位置 | 语言 | 职责 |
|------|---------|------|------|
| **Orch** | 主进程，主线程 | Python 调 C++ | submit（tensormap + fanin wiring）、scope |
| **Scheduler** | 主进程，独立 C++ 线程 | C++ | 从 ready_queue 取任务 → 推入 worker task_queue；从 completion_queue 收通知 → fanout release |
| **ChipWorker 线程 ×N** | 主进程，C++ 线程 per device | C++ | loop: pop task_queue → `run()` 阻塞 → push completion_queue |
| **SubWorker 线程 ×M** | 主进程，C++ 线程 per SubWorker | C++ | loop: pop task_queue → `run()` 写 mailbox + 自轮询 TASK_DONE → push completion_queue |
| **HostSubWorker 进程 ×M** | fork 出的子进程 | Python | loop: 轮询 mailbox TASK_READY → 执行 callable → 写 TASK_DONE |

### 3.2 通信模型

```
Orch ──ready_queue.push──→ Scheduler ──task_queue.push──→ Worker 工作线程
                                                                │
                                      completion_queue.push ←──┘ (run() 完成后)
                                      cv.notify()
     ←──ring.release────── Scheduler ←──completion_queue.pop──
                            fanout release → ready_queue.push
```

Scheduler 等在 CV 上，ready_queue 和 completion_queue 任一有内容时唤醒。

### 3.3 架构图

```
主进程
├── 主线程: Orch (Python → C++)
│     submit():
│       1. alloc slot + output (ring 满时 CV 阻塞)
│       2. tensormap lookup (找 producer → fanin)
│       3. tensormap insert (注册 output)
│       4. write task slot (fanin_count, payload)
│       5. finalize fanin (lock fanout_mu, 挂 consumer 链表)
│       6. if fanin_count==0 → state=READY, push ready_queue + cv.notify
│     scope_begin/scope_end
│
├── Scheduler 线程 (C++)
│     loop: wait on cv
│       drain completion_queue → on_task_complete → fanout release
│       drain ready_queue → pick idle worker → task_queue.push
│     全部完成 → 通知 execute() 返回
│
├── ChipWorker 工作线程 ×N (per device)
│     loop: task_queue.pop() → run(payload) → completion_queue.push + cv.notify
│
├── SubWorker 工作线程 ×M
│     loop: task_queue.pop()
│           → write_mailbox(TASK_READY)
│           → spin-poll mailbox until TASK_DONE
│           → write_mailbox(IDLE)
│           → completion_queue.push + cv.notify
│
HostSubWorker 进程 ×M (fork 出的子进程，各有独立 GIL)
      ← fork 在 HostWorker.__init__() 中完成，早于所有 C++ 线程启动
      loop: spin-poll mailbox.state == TASK_READY
            → callable_registry[callable_id]()
            → mailbox.state = TASK_DONE
```

### 3.4 与 L2 的对应关系

| L2 (AICPU, C++) | L3 (Host) |
|---|---|
| Orch 线程 → `pto2_submit_mixed_task()` | 主线程 → `submit()` |
| aicpu_executor 线程 | **Scheduler C++ 线程** |
| 轮询 AICore COND register（硬件无线程，只能轮询）| **completion_queue + CV 通知**（有线程，无需轮询）|
| AICore 硬件执行 | **Worker 工作线程**（每 Worker 一个线程）|
| dispatch entry + COND register | worker task_queue + completion_queue |

### 3.5 并行关系

| 场景 | 真并行 | 机制 |
|------|--------|------|
| Orch (submit) ↔ Scheduler | ✅ | 独立 C++ 线程，通过 ready_queue + CV 通信 |
| ChipWorker[0] ↔ ChipWorker[1] | ✅ | 各自工作线程独立执行 `run()` |
| ChipWorker ↔ SubWorker | ✅ | 工作线程各自阻塞执行，互不干扰 |
| SubWorker 线程 ↔ fork 子进程 | ✅ | 线程 spin-poll mailbox；子进程独立 GIL |
| Task chain 流水线 | ✅ | completion → fanout → ready_queue，Orch 不参与 |

---

## 4. HostSubWorker：Fork + Shared Memory

### 4.1 方案选择

| 方案 | GIL 并行 | tensor 零拷贝 | callable 约束 |
|------|---------|--------------|--------------|
| C++ 线程 + `gil_scoped_acquire` | ❌ Python 串行 | ✅ | 无 |
| spawn 新进程 | ✅ | ❌ 需序列化 | 必须 picklable |
| **fork + shm（采用）** | ✅ | ✅ | 无（fork 前已在内存） |

### 4.2 Fork 时机（关键约束）

`fork()` 在多线程环境下 unsafe。**必须在 `HostWorker.__init__()` 中完成所有 fork，早于任何 C++ 线程启动**：

```
HostWorker.__init__():
    1. 用户注册 callable（hw.register(name, fn) → callable_id）
    2. 分配 shm mailbox（SharedMemory，/dev/shm）
    3. fork M 个 HostSubWorker 进程            ← 单线程，safe
    4. 创建 C++ HostWorkerEngine（nanobind）     ← C++ 线程在此之后启动
    5. 创建 ChipWorker ×N
```

### 4.3 Mailbox 通信

每个 HostSubWorker 有独立的 256 字节 mailbox（cache-line aligned）：

```
Mailbox layout:
  offset 0:   int32  state        # IDLE=0, TASK_READY=1, TASK_DONE=2, SHUTDOWN=3
  offset 4:   int32  callable_id  # 预注册的 callable 编号
  offset 8:   int64  args_shm_fd  # tensor 数据所在 shm 的 fd
  offset 16:  int64  args_offset  # TaskArgs 在 shm 中的偏移
  offset 24:  int64  result_addr  # 结果 tensor 写回地址
  offset 32:  int32  error_code   # 0=ok, nonzero=failed
  offset 64:  char[192] error_msg # 错误信息
  total: 256 bytes (4 cache lines)
```

状态机（与 L2 AICore dispatch 同构）：

```
       Scheduler writes                Worker reads
IDLE ──→ TASK_READY ──→ Worker 轮询 ──→ 执行 callable ──→ TASK_DONE
  ↑                                                          │
  └──────────────── Scheduler 轮询 ←─────────────────────────┘
```

### 4.4 Tensor 零拷贝

- **torch tensor**: `tensor.share_memory_()` → storage 移到 `/dev/shm` → fork 后子进程直接访问同一物理页
- **Mailbox**: `SharedMemory`（`/dev/shm` 命名文件，`MAP_SHARED`）→ 父子进程共享物理页

### 4.5 Callable 注册

```python
hw = HostWorker(device_ids=[0, 1], ...)
cid = hw.register("postprocess", postprocess_fn)   # fork 前注册
# fork 后子进程 COW 继承 callable_registry，无需 pickle，lambda/闭包均可

def my_orch(hw, args):
    hw.submit(HOST_SUB, cid, args_c)  # 传 callable_id，不传函数对象
```

### 4.6 POC 验证结果

| 用例 | 结论 |
|------|------|
| SharedMemory fork 后双向可见 | ✅ |
| `share_memory_()` tensor 零拷贝 | ✅ |
| lambda/闭包 fork 后无需 pickle 直接调用 | ✅ |
| IDLE→TASK_READY→TASK_DONE 多轮循环 | ✅ |
| 3 worker 并行 wall time < serial × 0.7 | ✅ |
| fork 后启动 Python/C++ 线程无死锁 | ✅ |

---

## 5. 数据归属与同步

### 5.1 数据归属（与 L2 一致）

| 数据结构 | 归属 | 谁访问 | 保护方式 |
|----------|------|--------|---------|
| **TensorMap** | **Orch 独占** | 只有 submit() | **不需要锁** |
| **Scope stack** | **Orch 独占** | submit() / scope_begin/end() | **不需要锁** |
| Task slot state | 共享 | Orch + Scheduler | per-task mutex + atomic |
| Ready queue | 共享 | Orch push + Scheduler pop | mutex + CV |
| Completion queue | 共享 | Worker push + Scheduler pop | mutex + CV（同一 CV）|
| Ring flow control | 共享 | Orch alloc + Scheduler release | atomic + CV |
| Worker task_queue | Scheduler → Worker | Scheduler push + Worker pop | per-worker mutex |
| SubWorker mailbox | 线程 ↔ fork 子进程 | MAP_SHARED | acquire/release store |
| Tensor 数据 | 共享内存 | Orch + Worker | `share_memory_()` |

### 5.2 依赖推断与调度

与 L2 完全一致，基于 tensormap 的自动依赖推断 + eager dispatch：

1. submit 时，对每个 INPUT tensor 查询 tensormap 找到 producer → 建立 fanin 依赖
2. submit 时，对每个 OUTPUT tensor 写入 tensormap 注册为 producer
3. task 提交后立刻检查 fanin_refcount，全部满足就 dispatch
4. task 完成后遍历 fanout list，release consumer → 新的 ready task 立刻 dispatch

L3 不使用独立的 DAG 数据结构。依赖关系完全由 tensormap 管理。

---

## 6. Scope 管理

### 6.1 L2 Scope（已有，不改）

Device 侧 ring buffer 中间 buffer 生命周期管理。Scope 深度映射到不同 ring layer（最多 4 层），内层 scope 独立回收。

### 6.2 L3 Scope（新增，与 L2 同构）

管理 host 侧中间 tensor 生命周期：
- scope_begin → 记录 scope_tasks 位置，scope_stack_top++
- scope 内 submit() 的 task 获得 scope 引用（fanout_count +1）
- scope_end → 遍历 scope 内 task，release_scope_ref（fanout_refcount--）
- 当 fanout_refcount 达到 fanout_count → task 转为 CONSUMED，heap 可回收
- scope depth → ring layer 映射（与 L2 一致，不同 ring 独立回收）

---

## 7. 使用方式

### 7.1 Worker 工厂

所有层级统一通过 `Worker(level, ...)` 工厂创建，接口完全对称：

```python
worker = Worker(level=x, ...)   # 工厂：根据 level 构造对应 Worker
worker.register(fn) -> int      # 注册 SubWorker callable（init 前调用）
worker.init()                   # 分配资源、fork 子进程、启动 C++ 线程
worker.run(task)                # 阻塞执行一个任务
worker.close()                  # 释放资源
```

### 7.2 L2 单任务使用

```python
w2 = Worker(level=2, device_id=0, platform="a2a3sim", runtime="tmr")
w2.init()

for batch in batches:
    w2.run(WorkerPayload(callable=chip_callable, args=batch_args))

w2.close()
```

### 7.3 L3 多 Chip + SubWorker 使用

```python
w3 = Worker(level=3,
            chip_workers=[Worker(level=2, device_id=i, platform="a2a3sim", runtime="tmr")
                          for i in range(4)],
            num_sub_workers=4)

# SubWorker callable 在 fork 前注册
sub_cids = [w3.register(fn_i) for fn_i in postprocess_fns]
w3.init()

def my_orch(w, args):
    chip_results = []
    for i in range(4):
        r = w.submit(WorkerType.CHIP, chip_payload(i),
                     inputs=[args.input_ptrs[i]], outputs=[4*1024*1024])
        chip_results.append(r)

    for i in range(4):
        w.submit(WorkerType.SUB,
                 sub_payload(sub_cids[i]),
                 inputs=[chip_results[i].outputs[0].ptr])  # 自动推断依赖

# run() 与 L2 ChipWorker.run() 完全同构：接收一个任务，完成后返回
w3.run(Task(orch=my_orch, args=my_args))

# 多轮
for batch in batches:
    w3.run(Task(orch=my_orch, args=batch))

w3.close()
```

### 7.4 L4 递归组合

```python
w4 = Worker(level=4,
            dist_workers=[
                Worker(level=3, chip_workers=[...], num_sub_workers=2),
                Worker(level=3, chip_workers=[...], num_sub_workers=2),
            ])
w4.init()

def l4_orch(w, args):
    # 每个 submit 调度到一个 L3 节点，L3 节点内部自行展开
    w.submit(WorkerType.DIST, l3_task_payload_a, inputs=[args.part_a])
    w.submit(WorkerType.DIST, l3_task_payload_b, inputs=[args.part_b])

w4.run(Task(orch=l4_orch, args=big_args))
w4.close()
```

### 7.5 层级对称性

```
w = Worker(level=N, ...)
w.run(task)
```

从外部看，任意层级的 `Worker` 都是一样的：接收一个任务，阻塞，完成返回。
调用方不需要知道 task 是由 AICore、ChipWorker 还是 DistWorker 执行的。

---

## 8. 关键类型

### 8.1 类型名称映射

| 文档原名 | 实际类型名 | 定义位置 |
|---------|-----------|---------|
| `TensorArg` | `ContinuousTensor` | `src/common/task_interface/tensor_arg.h` |
| `SeparatedArgs` | `TaskArgs` | `src/common/task_interface/task_args.h` |
| `OrchArgs` | `DynamicTaskArgs` | `TaskArgs<ContinuousTensor, uint64_t, 0, 0>` |
| `OrchArgStorage` | `ChipStorageTaskArgs` | `TaskArgs<ContinuousTensor, uint64_t, 16, 128>` |

### 8.2 ChipCallable

ChipCallable 替代了 ChipTask 中的二进制载荷，包含 orch 函数和 kernel binaries。
通过 `init_runtime(RuntimeHandle, ChipCallable*, ChipStorageTaskArgs*)` 三参数调用。

---

## 9. Platform Services 分层

### 9.1 每个平台能力域独立 .so

```
host_runtime.so   → L2 runtime 执行       (已有)
host_comm.so      → 芯片间通信             (L3 用)
host_network.so   → 跨节点网络             (L4+ 用，未来)
```

每层按需 dlopen 自己需要的 .so，不侵入其他层。

### 9.2 Build 产物

```
build/lib/{arch}/{variant}/
  {runtime}/                         # runtime-dependent
    host_runtime.so                  # L2
    aicpu.so
    aicore.o
  comm/                              # runtime-independent
    host_comm.so                     # L3
```

---

## 10. 文件布局

### 10.1 C++ 层

```
src/common/distributed/                  # 调度引擎，L3/L4/L5/L6 通用
  dist_types.h                           # IWorker 接口 + 公共类型
  dist_tensormap.{h,cpp}                 # TensorMap
  dist_ring.{h,cpp}                      # TaskAllocator + RingSet + DepListPool
  dist_scope.{h,cpp}                     # ScopeManager
  dist_orchestrator.{h,cpp}              # submit 7步流程
  dist_scheduler.{h,cpp}                 # Scheduler 线程
  dist_sub_worker.{h,cpp}               # SubWorker：fork/shm
  dist_worker.{h,cpp}                    # DistWorker：暴露给 Python

src/common/worker/
  chip_worker.{h,cpp}                    # 已有，仅 L3 使用
```

### 10.2 Python 层

```
python/bindings/
  task_interface.cpp                     # 已有（数据类型 + ChipWorker）
  dist_worker_bind.cpp                   # 新增：DistWorker nanobind 绑定

python/
  task_interface.py                      # 追加 DistWorker re-export
  host_worker/
    __init__.py
    host_worker.py                       # 薄 wrapper，委托 DistWorker(level=3)
    host_task.py                         # HostTask dataclass
```

---

## 11. PR 拆分计划

### PR 2-1: C++ DistWorker 核心 + nanobind 绑定

新增 `src/common/distributed/` 全部文件 + nanobind 绑定 + C++ UT。

### PR 2-2: Python HostWorker wrapper + HostTask + 集成测试

HostWorker Python 薄 wrapper + HostTask + mock/sim 端到端测试。

### PR 2-3: DeviceRunner thread_local + 多 device

DeviceRunner singleton 改为 thread_local，支持多 device 并行。

### PR 2-4: Platform comm .so

独立通信库（由其他人负责）。

---

## 12. 与 Linqu Distributed Runtime 的关系

本设计（simpler Phase 2）对应 [linqu_runtime_design.md](linqu_runtime_design.md) 中 §3.1 描述的 `simpler` 已有能力的 L3 层扩展：

| 关系 | 说明 |
|------|------|
| **simpler L0-L2** | 不修改，ChipWorker 通过 dlsym C API 调用 |
| **simpler L3 (本设计)** | DistWorker + HostSubWorker，在 simpler 仓库内实现 |
| **Linqu L3-L6** | pypto_runtime_distributed 仓库，使用 simpler 的 ChipWorker/DistWorker 作为执行后端 |
| **接口对接** | Linqu 的 `ChipBackend` adapter 通过 dlopen/dlsym 调用 simpler 的 C API |

### 递归关系

```
Linqu L4+ Orchestrator
    ↓ CALL_TASK (RPC)
Linqu L3 NodeDaemon
    ↓ 调用 simpler Worker API
simpler DistWorker (L3)
    ↓ submit(CHIP, ...) / submit(HOST_SUB, ...)
simpler ChipWorker (L2) / HostSubWorker
    ↓ C API
simpler L0-L2 Runtime (device 上执行)
```

---

## 13. 设计要点总结

1. **IWorker 统一接口** — ChipWorker / SubWorker / DistWorker 均实现 `run(payload)` 阻塞接口
2. **Fork before threading** — HostSubWorker 在 `__init__()` 中 fork，早于 C++ 线程启动
3. **Tensor 零拷贝** — 通过 `share_memory_()` + `/dev/shm` 实现 fork 后零拷贝
4. **无 callable pickle 约束** — fork 前注册，子进程 COW 继承，传 ID 即可
5. **Scheduler 不轮询 Worker** — 与 L2 的关键差异：L3 用 completion_queue + CV 替代 COND 寄存器轮询
6. **TensorMap / Scope / Ring 与 L2 同构** — 不发明新概念，L3 是 L2 的 Python/C++ 版本
7. **每层 Worker 可递归组合** — L4 的 DistWorker 包含多个 L3 DistWorker，自然形成树形层次
