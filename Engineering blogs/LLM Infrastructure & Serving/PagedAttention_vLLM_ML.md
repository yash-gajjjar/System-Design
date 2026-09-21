# 1. Executive Summary & Core ML/AI Challenge

The paper is fundamentally about **LLM inference serving efficiency**, not model training or model quality. The central problem is that autoregressive LLM generation is sequential and therefore becomes **memory-bound**, especially because every active request maintains a dynamically growing **KV cache**. To serve more users efficiently, the system wants to batch many requests, but the KV cache consumes a large and unpredictable amount of GPU memory. Existing serving systems reserve large contiguous memory regions based on maximum sequence lengths, causing **reserved memory, internal fragmentation, and external fragmentation**. The paper reports that existing systems may use only **20.4%-38.2% of KV-cache memory for actual token states**, which directly limits batch size and therefore throughput.

The proposed solution is **PagedAttention + vLLM**. PagedAttention splits each request's KV cache into fixed-size blocks that can live in **non-contiguous GPU memory**, similar to pages in operating-system virtual memory. vLLM then adds a **block-level KV-cache manager, centralized scheduler, preemptive scheduling, memory sharing, and distributed GPU execution** around it. The headline result is **2-4x higher serving throughput at the same latency level** compared with systems such as FasterTransformer and Orca, with the gains becoming more pronounced for longer sequences, larger models, and more complex decoding methods.

### Scale mentioned in the paper

| DimensionWhat the paper reports |                                           |
| ------------------------------- | ----------------------------------------- |
| Example model                   | 13B-parameter model on NVIDIA A100 40GB   |
| Model weight memory             | \~26GB / \~65% of GPU memory              |
| KV cache                        | close to 30% of GPU memory in the example |
| OPT-13B KV per token            | \~800KB                                   |
| OPT-13B KV at 2048 tokens       | up to \~1.6GB/request                     |
| Evaluated model sizes           | OPT-13B, OPT-66B, OPT-175B; LLaMA-13B     |
| GPU configurations              | 1×A100, 4×A100, 8×A100-80GB               |
| Total GPU memory                | 40GB, 160GB, 640GB                        |
| Memory available for KV cache   | 12GB, 21GB, 264GB                         |
| Maximum KV-cache slots          | 15.7K, 9.7K, 60.1K                        |
| Main throughput result          | 2-4x vs. state-of-the-art serving systems |

For OPT-13B, the paper calculates that one token's KV state needs **800KB**, so a 2048-token request can consume about **1.6GB** of KV-cache memory. This explains why GPU memory, rather than raw FLOPS, becomes the key serving constraint.

---

# 2. ML/AI System & Platform Architecture

## 2.1 End-to-end architecture

The architecture is essentially:

```text
Client Request
     |
     v
FastAPI / OpenAI-compatible API
     |
     v
Centralized Scheduler
     |
     +-----------------------------+
     |                             |
     v                             v
KV Cache Manager              Request Scheduling
     |                             |
     +-------------+---------------+
                   |
             Candidate sequences
                   |
                   v
        Logical -> Physical
          KV Block Mapping
             Block Tables
                   |
                   v
      +------------+-------------+
      |            |             |
      v            v             v
   Worker 0     Worker 1      Worker N-1
   Model Shard  Model Shard   Model Shard
      |            |             |
      +------------+-------------+
                   |
               NCCL / AllReduce
                   |
                   v
        PagedAttention CUDA Kernels
                   |
                   v
             Sampled Token
                   |
                   v
              Scheduler
                   |
                   v
              Client
```

The paper's **Figure 4 on page 5** explicitly shows the core architecture: a centralized scheduler, KV-cache manager, CPU/GPU block allocators, block tables, and multiple GPU workers containing model shards, cache, and execution engines.

---

## 2.2 What actually happens during inference?

The paper divides generation into two phases.

### Phase 1: Prompt / Prefill

The entire prompt is known, so the model can process the prompt using highly parallel matrix-matrix operations. During this phase, the KV vectors for the prompt are generated.

### Phase 2: Autoregressive decoding

Only one new token is generated at a time. The model uses the previously generated tokens' KV states from the KV cache and creates the next KV state.

This second phase is particularly expensive for serving because the iterations are sequential and cannot be parallelized across time. The paper says this causes poor GPU utilization and makes generation **memory-bound**.

A simple way to remember it:

```text
Prompt:
[ A B C D E F ]
     ↓
process many tokens together
     ↓
KV cache created

Generation:
[A B C D E F] -> G
[A B C D E F G] -> H
[A B C D E F G H] -> I
...
```

The challenge is that every active request keeps accumulating KV state.

---

## 2.3 Why normal batching is not enough

Traditional batching has two problems:

**Different arrival times:** a request arriving later may have to wait for an entire batch.

**Different sequence lengths:** requests with different prompt/output lengths require padding, wasting memory and compute.

The paper therefore relies on **iteration-level scheduling**, where completed sequences are removed and new ones can join after an iteration rather than waiting for the entire batch.

So the system is really combining two ideas:

```text
Iteration-level scheduling
        +
Paged KV-cache memory management
        =
More requests fitting into GPU memory
        =
Higher throughput
```

---

## 2.4 PagedAttention: the key architectural innovation

Instead of:

```text
Request A -> one huge contiguous KV buffer
```

vLLM does:

```text
Request A
Logical blocks:
[B0][B1][B2][B3]

Physical GPU memory:
      [B2]   [B0] [B3]      [B1]
```

The logical sequence remains contiguous from the request's perspective, but the physical blocks do not need to be adjacent.

A **block table** maps:

```text
Logical Block -> Physical Block
```

and also records how many positions in the block are currently filled.

This is the central abstraction that makes the rest of the system possible.

---

## 2.5 Dynamic allocation instead of maximum reservation

Traditional systems reserve memory for the maximum possible sequence length.

For example:

```text
Maximum sequence = 2048 tokens

Reserve:
[2048 slots]

Actual request:
[150 tokens]

Unused:
1898 slots
```

vLLM instead allocates blocks as tokens are actually generated:

```text
Start:
[B0][B1]

Later:
[B0][B1][B2]

Later:
[B0][B1][B2][B3]
```

Thus, the request only consumes approximately what it actually needs, with waste limited to at most the unused portion of the currently active block.

---

## 2.6 KV-cache sharing

This is one of the most important design consequences.

### Parallel sampling

Suppose one prompt produces four outputs:

```text
Prompt
  |
  +--> Sample A
  +--> Sample B
  +--> Sample C
  +--> Sample D
```

All four sequences have the same prompt KV cache.

Instead of storing:

```text
Prompt KV
Prompt KV
Prompt KV
Prompt KV
```

vLLM can map all four logical prefixes to the same physical blocks.

When one sequence needs to modify a shared block, vLLM uses **copy-on-write**.

This is exactly analogous to how operating systems can share pages until one process modifies them.

### Beam search

Beam search has even more sharing opportunities because different candidates can share blocks for portions of their history.

The sharing structure changes dynamically as beams diverge and later candidates are discarded. vLLM uses reference counts and frees physical blocks when the reference count reaches zero.

### Shared prefixes

The system can also pre-cache common prompt prefixes.

For example:

```text
System Prompt
+ Example 1
+ Example 2
+ Example 3
-------------------
             |
      shared prefix
             |
     +-------+-------+
     |               |
 Task A            Task B
```

The common prefix's KV blocks can be reused rather than recomputed for each request.

---

## 2.7 Mixed decoding

One particularly interesting architectural property is that different requests can use different decoding strategies while being processed together.

The mapping layer hides the complexity:

```text
Request A -> sampling
Request B -> beam search
Request C -> prefix sharing
Request D -> multiple samples

             |
             v

        Logical -> Physical
        block mapping
```

The model/kernel sees physical block IDs rather than having to understand the sharing relationships.

This increases batching opportunities across heterogeneous requests.

---

## 2.8 Scheduling and preemption

When GPU blocks are exhausted, vLLM uses **FCFS scheduling**.

The paper specifically says this provides fairness and prevents starvation: earlier arrivals are served first, while later arrivals are preempted first.

When memory becomes insufficient, vLLM supports two recovery mechanisms.

### Option A: Swap

```text
GPU KV blocks
     |
     v
CPU RAM
```

The KV blocks are moved from GPU memory to CPU memory.

A CPU block allocator manages the swapped blocks.

### Option B: Recomputation

Instead of storing the KV cache, vLLM can recompute it later.

The interesting observation is that generated tokens can be concatenated with the original prompt and processed together as a new prompt, allowing the KV cache to be regenerated through the more parallel prompt phase.

---

## 2.9 Distributed serving

Large models may not fit on one GPU, so the paper uses **Megatron-LM-style tensor model parallelism**.

For example:

```text
175B Model
     |
     +--------------------+
     |                    |
 GPU 0                 GPU 1 ... GPU 7
 Shard 0               Shard 1 ... Shard 7
```

The model's linear layers are partitioned, and attention heads are split across workers.

The GPU workers synchronize intermediate computations through **all-reduce**.

An important design decision is that vLLM maintains **one centralized KV-cache manager** even though execution is distributed. All workers share the logical-to-physical block mapping, while each worker stores only the KV portion corresponding to its attention heads.

This is a very good ML-system-design pattern:

> **Centralize global memory-management decisions, but keep GPU computation distributed.**

---

## 2.10 Serving stack / technology choices

The paper explicitly describes:

| LayerTechnology / design |                                      |
| ------------------------ | ------------------------------------ |
| API                      | FastAPI                              |
| API interface            | OpenAI API-compatible interface      |
| Control plane            | Python                               |
| Scheduler                | Python                               |
| Block manager            | Python                               |
| Inference engine         | GPU                                  |
| Model implementation     | PyTorch + Transformers               |
| Custom acceleration      | C++ / CUDA                           |
| Communication            | NCCL                                 |
| Parallelism              | Megatron-LM-style tensor parallelism |
| Main kernel              | Custom PagedAttention CUDA kernel    |
| GPU memory               | GPU block allocator                  |
| CPU offload              | CPU block allocator                  |

The implementation is approximately **8.5K lines of Python and 2K lines of C++/CUDA**. Control-related components are implemented in Python, while custom CUDA kernels are used for performance-sensitive operations such as PagedAttention.

The paper also explicitly shows the motivation for custom kernels: PagedAttention introduces new memory-access patterns that existing systems do not efficiently support, so the authors fuse operations and optimize memory access themselves.

---

## 2.11 Kernel-level optimization

Three important kernel optimizations are described:

**1. Fused reshape + block write**

KV states are reshaped into the appropriate block layout and written in one kernel to reduce kernel-launch overhead.

**2. Fused block read + attention**

The attention kernel directly reads KV blocks according to the block table and computes attention.

**3. Fused block copy**

Copy-on-write may otherwise trigger many small GPU copies, so vLLM batches those copies into one kernel launch.

This is an important architectural lesson: the paper does not stop at a clever memory abstraction; it modifies the GPU execution layer to prevent the abstraction itself from becoming too expensive.

---

## 2.12 What is *not* in this architecture?

The document is specifically focused on **serving**, so the following are not described:

- Feature store
- Feature engineering pipeline
- Training pipeline
- Training orchestration
- Data lake / warehouse
- Vector database
- RAG retrieval
- Agent orchestration
- Online model-quality monitoring
- Model registry/version rollout system

So for an interview, do **not** invent these components when explaining this paper. The model is assumed to already be trained and deployed for inference.

---

# 3. Critical ML Engineering Trade-offs & Design Choices

## 3.1 Memory efficiency vs. memory-access overhead

This is probably the most important trade-off.

PagedAttention gains memory efficiency through indirection:

```text
logical block
      |
      v
block table
      |
      v
physical block
```

But indirection itself costs computation.

The paper measures this directly: the PagedAttention kernels have **20%-26% higher attention-kernel latency** than FasterTransformer because of block-table access, extra branches, and variable sequence handling.

Yet this local slowdown is outweighed by the improvement in end-to-end serving throughput.

This is a classic systems principle:

> **A small hot-path overhead can be worthwhile when it unlocks much better global resource utilization.**

---

## 3.2 Block size: memory efficiency vs. GPU utilization

Block size introduces another explicit trade-off.

### Too small

Pros:

- Less fragmentation
- Better sharing granularity

Cons:

- Worse GPU utilization

### Too large

Pros:

- Better parallel processing of KV blocks

Cons:

- More internal fragmentation
- Lower sharing opportunities

The paper found that **16-128** worked well for ShareGPT, while larger blocks degraded performance on Alpaca because the sequences were shorter. The authors therefore selected **block size 16 as the default**.

This is an excellent interview example of a tunable systems parameter whose optimum depends on workload characteristics.

---

## 3.3 Swapping vs. recomputation

There is no single universally optimal recovery strategy.

### Swapping

```text
GPU -> CPU -> GPU
```

Problem: many small blocks can result in many small PCIe transfers.

### Recomputation

```text
Discard KV
   |
   v
Run prompt again
   |
   v
Recreate KV
```

The paper finds:

- Swapping suffers significantly with small block sizes.
- Recomputation cost is relatively stable across block sizes.
- Recomputation is more efficient for small blocks.
- Swapping becomes more attractive for larger blocks.
- For block sizes 16-64, the two approaches show comparable end-to-end performance.
- Recomputation overhead is never higher than 20% of swapping latency in their experiments.

So the broader principle is:

> **When memory pressure occurs, choose between data movement and recomputation based on the relative cost of bandwidth vs. compute.**

---

## 3.4 Memory utilization vs. compute utilization

An especially important experimental observation is that PagedAttention's benefit is workload-dependent.

For the OPT-175B + Alpaca case, there was already enough GPU memory for a large number of short sequences. Consequently, the system became **compute-bound rather than memory-bound**, reducing vLLM's advantage.

This tells you exactly when the technique matters:

```text
Memory-bound workload
        -> memory optimization matters greatly

Compute-bound workload
        -> memory optimization may provide limited benefit
```

The paper explicitly warns that applying this technique to workloads that do not have these characteristics can actually hurt performance because of the additional indirection and non-contiguous memory access.

---

## 3.5 Preallocation vs. dynamic allocation

The older design essentially says:

```text
"I don't know the future,
so reserve the maximum."
```

vLLM says:

```text
"I don't know the future,
so allocate incrementally."
```

This is a much better match for autoregressive generation because future output length is unknown.

The paper identifies three major waste sources in the older approach:

```text
1. Reserved memory
2. Internal fragmentation
3. External fragmentation
```

and shows that effective memory utilization could fall as low as **20.4%**.

---

## 3.6 Copying vs. sharing

Instead of copying KV states whenever multiple sequences share context, vLLM uses:

```text
Reference counting
        +
Copy-on-write
```

This is particularly valuable for beam search.

The paper reports memory savings of approximately:

| Decoding methodAlpacaShareGPT |             |             |
| ----------------------------- | ----------- | ----------- |
| Parallel sampling             | 6.1%-9.8%   | 16.2%-30.5% |
| Beam search                   | 37.6%-55.2% | 44.3%-66.3% |

The paper's evaluation shows that sharing becomes increasingly important as decoding becomes more complex.

---

## 3.7 Quality and reliability enforcement

This paper is **not primarily a model-quality paper**.

There is no discussion of:

- model accuracy dashboards,
- drift monitoring,
- model-version governance,
- canary rollout,
- model rollback,
- quality SLAs,
- production alerting.

What it does establish is that the throughput improvements occur **without affecting model accuracy**, because PagedAttention changes memory organization rather than the model's mathematical behavior.

For serving reliability, the paper focuses much more on **fair scheduling and memory-pressure handling**:

```text
FCFS
  |
  +--> fairness
  +--> avoid starvation
  |
  +--> preemption
          |
          +--> swap
          |
          +--> recompute
```

---

## 3.8 Why compaction was not selected

The paper discusses memory compaction as a possible solution to fragmentation but argues that compacting massive KV caches in a performance-sensitive online serving environment is impractical.

Even if compaction were possible, it would not solve the deeper problem of **sharing KV cache between decoding sequences**.

So they don't merely patch the old allocator.

They change the representation itself:

```text
Old:
Contiguous memory + compaction

New:
Non-contiguous paged memory
```

That is a much deeper architectural change.

---

# 4. High-Impact Interview Takeaways

## 4.1 The core design pattern to remember

The strongest interview-level abstraction from this paper is:

> **When a stateful inference workload has dynamically growing memory requirements, don't allocate one large contiguous region per request. Introduce a logical-to-physical indirection layer, allocate fixed-size blocks on demand, and use sharing + copy-on-write + preemption to maximize accelerator utilization.**

The entire architecture can be compressed into this:

```text
Dynamic request state
        |
        v
Fixed-size blocks
        |
        v
Logical -> Physical mapping
        |
        +--> dynamic allocation
        +--> sharing
        +--> copy-on-write
        +--> eviction
        +--> recomputation/swap
        |
        v
More requests fit in GPU memory
        |
        v
Larger effective batch
        |
        v
Higher throughput
```

---

## 4.2 Interview talking point #1

> **"For autoregressive LLM serving, I would first identify whether the workload is memory-bound or compute-bound. In this design, the KV cache grows dynamically with every active request, so contiguous preallocation causes fragmentation and limits batch size. I would use a paged KV-cache design with logical-to-physical block mapping so memory can be allocated incrementally and reused across requests."**

This captures the **problem -> diagnosis -> architecture** chain.

---

## 4.3 Interview talking point #2

> **"I would separate the logical sequence representation from physical GPU memory. The request sees logical KV blocks, while the memory manager maps them to physical blocks. That abstraction lets me dynamically allocate memory, share common prefixes, use reference counting and copy-on-write for branching decoding, and reclaim blocks immediately when sequences finish."**

This is probably the best way to explain PagedAttention without getting lost in the attention equations.

---

## 4.4 Interview talking point #3

> **"At high load, memory pressure becomes a scheduling problem as well as a storage problem. I would therefore co-design the scheduler and memory manager: use iteration-level scheduling for fine-grained batching, FCFS for fairness, and preempt requests when GPU blocks are exhausted. Recovery can be either swapping to CPU memory or recomputation, with the choice depending on the relative cost of GPU compute and CPU-GPU bandwidth."**

This connects **ML serving + scheduling + memory management**, which is exactly what makes this paper valuable for an ML System Design interview.

---

# The one-paragraph mental model

Think of vLLM like a **smart hotel for LLM requests**. The GPU is a hotel with limited rooms, and each request needs a growing amount of room because its KV cache grows token by token. Old systems reserve a huge room for every guest up front, even when they may use only a small part of it, so the hotel quickly runs out of usable space. PagedAttention breaks each guest's requirement into small fixed-size blocks, lets those blocks live anywhere in the hotel, gives the scheduler a map of where each block is located, shares common blocks between guests when possible, copies a block only when someone modifies shared data, and moves or recomputes cached data when the hotel becomes full. That means **more active requests fit into the same GPU memory**, which increases the effective batch size and ultimately produces the paper's reported **2-4x serving-throughput improvement**. The key insight is not "paging is always better"; it is that paging is especially useful when the workload has **dynamic memory requirements and is memory-bound**, which the authors explicitly identify as the situation for autoregressive LLM serving.