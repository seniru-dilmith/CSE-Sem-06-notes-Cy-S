---
Domain: Operating Systems and Kernel Engoineering
Focus Area: eBPF
Target: FYP
Date: 2026-05-28
---
## Topic 1: Verifier-Aware LLVM Optimization Pass for eBPF Bytecode

### Concept
Design and implement a custom LLVM Intermediate Representation (IR) optimization pass that modifies eBPF bytecode specifically to satisfy the strict state-tracking boundaries of the in-kernel verifier without sacrificing native execution speed.

### Problem Context & Feasibility
Standard compiler optimization flags (such as Clang `-O2` or `-O3`) focus purely on streamlining native CPU instruction paths. However, this often introduces aggressive register reuse, complex loop unrolling, or highly branching structural logic. Because the kernel verifier tracks all possible paths to guarantee safety, these standard optimizations frequently trigger a state-space explosion, forcing developers to manually refactor valid logic just to get it accepted (Jia et al., 2023).

### Research Novelty
While tools exist to analyze source code or catch verifier bugs defensively, there is currently no active compilation pass that dynamically mutates code optimization strategies *based directly on verifier state limits*. Your project would create a "cooperative pipeline" between the compiler frontend and the kernel constraints.

### Validation & Verification Sources
- **Core Academic Paper:** *Kernel extension verification is untenable* (Jia et al., 2023). This paper breaks down the exact human and machine verification costs that make current static verification approaches problematic.
- **Primary Source Link:** [Jia et al. (2023) Research Paper PDF](https://people.cs.vt.edu/djwillia/papers/hotos23-untenable.pdf)
- **Complementary Bytecode Work:** To understand modern programmatic parsing of compiled eBPF paths, check the *Yaksha-Prashna* verification tracking project [Singh et al. (2026) on arXiv](https://arxiv.org/pdf/2602.11232).

---

## Topic 2: Adaptive Runtime Graph Pruning Under High Load

### Concept
Build an adaptive, closed-loop telemetry framework for eBPF that dynamically shortens or "prunes" execution branches based on real-time hardware execution pressure (CPU cycle limits, cache misses).

### Problem Context & Feasibility
In high-throughput target nodes (such as dense containerized firewalls or security monitors), executing a large, static eBPF program chain on every single incoming packet or system call introduces non-negligible processing penalties. Under sudden stress, like a high-bandwidth distributed denial-of-service spike, the execution overhead of the observation subsystem can exhaust the CPU itself.

### Research Novelty
Once an eBPF program passes verification and is loaded, its structural path execution is fixed. Your approach will introduce the first self-throttling runtime controller. By utilizing eBPF program arrays and tail calls (`BPF_MAP_TYPE_PROG_ARRAY`), your background daemon will monitor localized core metrics (such as L3 cache missing and ring-buffer drop alerts) and dynamically hot-swap complex monitoring chains into lightweight "fast paths" during runtime spikes.

### Validation & Verification Sources
- **Industry Implementations:** Production frameworks like Cilium's Tetragon highlight the ongoing need for microsecond-scale security hook execution without choking line-rate performance.

---

## Topic 3: Lockless, Cache-Aligned NUMA-Sharded eBPF Maps

### Concept
Create an autonomous, multi-socket topology-aware map subsystem within the eBPF infrastructure designed to completely eliminate cross-socket memory bus latency and cache invalidation bottlenecks.

### Problem Context & Feasibility
Modern high-performance infrastructure operates on multi-socket architectures with Non-Uniform Memory Access (NUMA). Standard eBPF map implementations allow manually pinning a map's backing memory to a specific socket configuration (`bpf_map__set_numa_node`). However, if concurrent user space runtimes or scheduling threads scale across different sockets, cross-node cache synchronization overhead and bus lock cycles heavily degrade performance.

### Research Novelty
Instead of relying on a single pinned memory address space, your project will introduce a self-sharding map primitive. It will dynamically partition map states, implement lockless localized cache mirrors across asymmetric core topologies, and leverage synchronized ring buffers to prepare eBPF for next-generation tiered memory pooling frameworks, such as Compute Express Link (CXL) hardware.

### Validation & Verification Sources
- **Kernel Implementation:** Linux Kernel Source documentation regarding core structural constraints for `bpf_map` data structures and multi-socket NUMA affinity allocations.

---

## Topic 4: Predictive Path Speculation for `sched_ext` Runtimes

### Concept
Develop a predictive JIT compiler optimization helper extension for the extensible scheduler framework (`sched_ext`) to pre-fetch task contexts and streamline microsecond-level scheduling paths.

### Problem Context & Feasibility
The integration of `sched_ext` into the mainline kernel has opened up operating system design, enabling developers to bypass the hardcoded scheduler architecture and deploy custom thread-scheduling logic directly via eBPF. While this has shown immense promise for isolating conflicting workloads, any lookup latency or helper execution loop inside a critical scheduling hook (like `select_cpu`) can bottleneck maximum processing throughput.

### Research Novelty
Current research focuses on inventing custom scheduling rules for specific workloads, such as the Unfair Scheduler (UFS) designed to isolate multi-tenant database requests. Your project focuses instead on the underlying execution framework: designing an inline branch speculator that learns historical execution behaviors of the custom scheduler and preemptively updates the running JIT-compiled instruction path to reduce evaluation cycle overhead to nearly zero.

### Validation & Verification Sources
- **Core Academic Paper:** *Unfair by design: eBPF-based scheduling of mixed database workloads* (2026). This paper documents how deploying a custom scheduler via `sched_ext` can double database processing capabilities under heavily oversubscribed environments.
- **Primary Source Link:** [UFS Sched_ext Research Paper PDF (2026)](https://arxiv.org/pdf/2605.02377)

---

## References

Jia, J., Sahu, R., Oswald, A., Williams, D., Le, M. V., & Xu, T. (2023). Kernel extension verification is untenable. *Proceedings of the 19th Workshop on Hot Topics in Operating Systems*, 150-157. https://doi.org/10.1145/3593856.3595892
Cited by: 46