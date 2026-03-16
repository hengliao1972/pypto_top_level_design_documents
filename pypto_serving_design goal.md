# pypto-serving 设计目标

构建基于 **Lingqu 分布式运行时** (`pypto_runtime_distributed`) 的超高性能 LLM 推理引擎，具备 vLLM / SGLang 的核心子集能力。

---

## 设计目标

1. **C/C++ 高性能核心**
   性能关键路径（Prefill、Decode、KV Cache 访问、Radix Tree 查找）全部用 C/C++ 实现，不允许 Python 出现在自回归路径、prefill-to-decode 路径或 KV Cache 访问路径中。

2. **基于 Lingqu L3–L7 分布式运行时**
   推理引擎构建在 `pypto_runtime_distributed` 的 `LevelRuntime` 之上（L3 Host 层为单机推理核心，L4–L7 为分布式扩展）。使用 orchestrator/worker/scheduler 线程模型、DAG tensor 依赖调度、ring buffer 内存管理——与 simpler L0–L2 运行时设计保持一致。

3. **PyPTO 编码规则：每层级 orchestrator + workers 分离**
   每个层级严格遵循 PyPTO 编码规范：
   - **一个编排函数**（orchestrator）：构建任务 DAG，提交 worker 和子编排器，等待 future。不直接计算数据。
   - **多个 worker 函数**：纯计算函数，在 worker 线程上并行执行。不提交后续任务。
   
   PyPTO Python DSL（`.py` 文件）作为权威规范，C++ 实现必须与之 1:1 对应。参见 `L2_chip_workers.py`、`L3_prefill_server.py`、`L3_decode_server.py`、`L4_pod_orchestrator.py`。

4. **L2 芯片级 kernel 通过 ChipBackend adapter 调用**
   假设 L2 层已有 `model_prefill` 和 `model_decode` 占位函数（当前为空实现）。L3 Host worker 通过 `ChipBackend` adapter（动态链接 simpler 的 `libhost_runtime.so`）调用 L2 kernel，执行 h2d_copy / d2h_copy 数据搬移。**不修改 simpler**。

5. **初始部署拓扑：PC16 x2（Prefill/Decode 分离）**
   首版面向最小 Lingqu 系统：
   - **L4 Pod**（1 实例）：2 台 PC16 服务器，`LevelRuntime(level=4, sched=1, workers=2)`
   - **L3[0] Prefill PC16**：16 NPU 芯片，`LevelRuntime(level=3, sched=1, workers=16)`
   - **L3[1] Decode PC16**：16 NPU 芯片，`LevelRuntime(level=3, sched=1, workers=16)`
   - **L2 NPU stubs**：32 块芯片（16×2），Phase 0 为 `ChipBackendStub`
   
   两个独立的 L3 LevelRuntime 实例模拟物理服务器分离。L4 orchestrator 将 prefill 路由到 L3[0]，decode 路由到 L3[1]。

6. **Radix Tree KV Cache 管理（SGLang 风格）**
   - **Radix Tree 元数据**：通过 `lingqu_db`（Redis-style K/V 服务）存储，L3–L7 所有层级可直接访问。Phase 0 使用本地 `meta_data.dat` 文件作为 stub。
   - **KV Cache 三层存储**：
     - **L1 (Hot)**：GPU VRAM，由 L2 simpler HeapRing 管理
     - **L2 (Warm)**：Host 内存，通过 `lingqu_shmem` 跨节点共享
     - **L3 (Cold)**：SSD / 分布式文件，通过 `lingqu_block`（异步 DMA）或 `lingqu_dfs`（POSIX 文件 API）持久化
   - **驱逐策略**：LRU，L1→L2→L3 逐级驱逐，按需从 L3→L2→L1 预取。

7. **Autoregressive Loop**
   - **Phase 0–3**：自回归循环在 L3 Host 层通过 LevelRuntime DAG 调度执行，每步 decode 由 16 个 NPU worker 并行执行（tensor parallelism），logits 通过 `tree_reduce` allreduce，再经 `sample_token` worker 采样。
   - **Phase 5（未来优化）**：通过 `autoregressive_loop_wrapper` 将整条 AR 循环下推到 L2 设备侧闭环执行，Host 一次提交、一次取回，无逐 token 往返。

8. **分布式推理扩展（L4–L7）**
   - **L4 (Pod)**：Prefill/Decode disaggregation（首版已实现），Tensor parallelism
   - **L5 (Supernode)**：Prefill/Decode 服务分离，独立批处理
   - **L6 (Cluster)**：QoS 分级，多档 batch size 实例对应不同 TTFT/TPOT 目标
   - **L7 (Global)**：全局请求路由，按 QoS 等级分发到对应服务实例
   - 同档内按序列长度分组以提升 batch 效率

9. **测试直通路径与 golden 校验**
   通过 Test Path 的 C 接口（`inject_request` / `get_response`）注入请求和取回响应。从 simpler 复制 golden.py 到 `pypto-serving/examples/`，修改为通过 Test Path 注入输入，与 `compute_golden` 参考值对比。

10. **Perfetto UI trace 支持**
    复用 `pypto_runtime_distributed` 的 `TraceWriter`，记录每个 prefill / decode / sample worker 的执行时间、每个层级实例的 orchestrator / scheduler / worker 线程活动，支持 `--trace` 命令行生成 trace 文件。Phase 0 已验证 trace 生成。

11. **长期上下文无关**
    引擎不维护用户 / 会话状态，每次请求由客户端携带完整上下文（含历史）。唯一的持久化状态是 Radix Tree（及其关联的 KV Cache 持久池）。

---

## 项目位置与依赖

- **推理引擎**：`pypto_workspace/pypto-serving/`
- **分布式运行时**：`pypto_workspace/pypto_runtime_distributed/`（链接其 `linqu_runtime_lib` 和 `linqu_core`）
- **芯片运行时**：`pypto_workspace/simpler/`（通过 ChipBackend adapter 动态链接，不修改）

## Phase 0 实现状态（已完成）

| 组件 | 状态 |
|------|------|
| CMakeLists.txt（链接 pypto_runtime_distributed） | 已完成 |
| PyPTO DSL 规范（L2/L3/L4 per-level .py） | 已完成 |
| ChipBackend 接口 + 确定性 Stub | 已完成 |
| L3 Worker 函数 (C++) | 已完成 |
| InferenceEngine (L3 orchestration) | 已完成 |
| PodOrchestrator (L4 orchestration) | 已完成 |
| ServingSystem (顶层生命周期 + trace) | 已完成 |
| 冒烟测试（L4→L3 prefill→decode 全链路） | 已通过 |

---

## 参考文档

- `pypto_serving_implementation_plan.md` — 详细实现计划与阶段分解
- `pypto_serving_reference_sglang_vllm.md` — vLLM / SGLang 接口与架构参考
- `linqu_runtime_design.md` — Lingqu L3–L7 分布式运行时设计
- `machine_hierarchy_and_function_hierarchy.md` — 层级模型与 pl.function / pl.at 语法
- `linqu_data_system.md` — Lingqu 四层数据服务（shmem, block, dfs, db）
