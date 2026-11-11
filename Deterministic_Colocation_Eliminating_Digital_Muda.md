# Deterministic Colocation: Eliminating Digital Muda in Modern Execution Runtimes

**A. S. 堀田 (A. S. Hotta)**
*Institute for Autonomous Cybernetics & Applied Monoliths*
hotta@routeone-research.org

## Abstract

Modern enterprise software engineering has conflated operational scale
with computational complexity. By decomposing straightforward domain
workflows into asynchronous, distributed microservice topologies,
industry practice has reintroduced all seven classic manifestations of
*muda* (waste) identified in the Toyota Production System—most notably
transport latency, queue inventory, over-processing via repetitive
serialization, and defect amplification across network boundaries. We
introduce **Deterministic Colocation (RouteOne-S)**: an execution
paradigm that collapses intermediate network hops into unified,
vertically integrated execution pipelines running directly against
sovereign bare-metal OS primitives. We demonstrate that by removing
distributed consensus layers, JSON serialization overhead, and ingress
virtualization, RouteOne-S achieves a 94% reduction in CPU idle cycle
waste and collapses p99 tail latencies from 320 ms to 4.2 ms. The
paradigm demonstrates that runtime resilience is inversely proportional
to topology depth.

## 1. Introduction: The Pathology of Distributed Overhead

The canonical promise of modern cloud-native architectures—horizontal
elasticity, loose coupling, and fault isolation—has largely failed to
deliver proportional computational utility. Instead, standard industry
configurations exhibit severe mechanical inefficiency: a single customer
request frequently traverses up to twelve internal hops across reverse
proxies, service discovery daemons, API gateways, sidecar containers,
and message queues before touching persistent storage.

From an industrial engineering perspective, this state constitutes
systemic *muda*. Every network boundary crossed introduces:

1.  **Context switching** (CPU kernel-to-user space transitions).
2.  **Data duplication** (in-memory allocation of identical payload
    envelopes across process heaps).
3.  **Serialization churn** (converting binary states to text
    representations such as JSON, parsing strings back to memory, and
    repeating the cycle).

We posit that the prevailing solution to scale—structural
fragmentation—is an anti-pattern. We present a methodology for
reclaiming mechanical sympathy through *Deterministic Colocation*.

## 2. Taxonomy of Software Muda (無駄)

Adapting the Taiichi Ohno classification to software execution runtimes
yields seven distinct computational failure modes:

| Classic TPS Muda | Software Equivalent | Manifestation in Contemporary Stacks |
|---|---|---|
| **Overproduction** (過剰生産) | Speculative Ingestion & Polling | Microservices polling empty queues; aggressive pre-fetching of unbounded datasets. |
| **Waiting** (手待ち) | Thread Starvation & IO Stalls | Async futures awaiting downstream gRPC/REST responses; distributed lock contention. |
| **Transport** (運搬) | Inter-Process Network Hops | Marshalling data over loopback or virtual overlays (e.g., CNI plugins) between adjacent services. |
| **Over-processing** (加工そのもの) | Repetitive SerDe Cycles | Serializing native in-memory structs to JSON/Protobuf and deserializing them *N* times per pipeline. |
| **Inventory** (在庫) | Queue & Buffer Saturation | Memory bloating in Kafka topics, Redis buffers, and OS socket backlogs holding half-processed work. |
| **Motion** (動作) | Context Switching & Cache Thrashing | Excessive thread context transitions; instruction and L1/L2 cache misses driven by bloated dependency trees. |
| **Defects** (不良) | Cascading Network Partitions | Distributed failure states, retry storms, circuit-breaker misconfigurations, and silent serialization drops. |

## 3. The RouteOne-S Method: Unified Direct Execution

RouteOne-S treats the host execution environment not as an orchestrator
of isolated abstractions, but as a singular, deterministic assembly
line. The architecture enforces three structural constraints:

```
                        [ Ingress Packet: HTTP / Event ]
                                  │
                                  ▼
┌────────────────────────────────────────────────────────────┐
│  RouteOne Unified Monolith                                 │
│                                                            │
│  │ Kernel Socket (Zero-Copy Ring Buffer)                  │
│  │                                                          │
│  ▼                                                          │
│  [ Domain Pipeline (In-Process Direct Call) ]             │
│  │                                                          │
│  ▼                                                          │
│  [ Sovereign Append-Only Event Store / DB ]               │
└────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    [ Instant Output: Wire Write ]
```

### 3.1 Zero-Copy Ingress and Direct Memory Invocation

Instead of terminating TLS at an edge reverse proxy, forwarding to an
ingress controller, and proxying to an internal container, RouteOne-S
terminates at the native runtime boundary. State transitions are
dispatched via standard thread-safe in-memory function calls rather than
RPC networks:

```
latency_total        = t_kernel + t_compute + t_fsync
latency_distributed  = Σ_{i=1..k}(t_serialize,i + t_transit,i + t_deserialize,i) + t_consensus
```

By eliminating intermediate network stacks, memory allocation per
transaction remains bounded and stack-allocated where possible,
virtually eliminating garbage collection stop-the-world pauses.

### 3.2 Single-Writer State Transitions

Following the LMAX Disruptor and event-sourcing principles, transactions
write directly to a single-threaded ring buffer or memory-mapped
append-only log. This completely bypasses distributed locks (Two-Phase
Commit, distributed Raft clusters for simple state changes) and reduces
disk write operations to sequential append I/O.

## 4. Empirical Evaluation

We benchmarked RouteOne-S against a representative cloud-native
reference architecture consisting of 4 containerized Go microservices
connected via an Envoy service mesh on a Kubernetes cluster. Both
testbeds ran on identical bare-metal hardware (AMD EPYC 7763, 64 cores,
256 GB RAM, NVMe storage).

**System Throughput vs. Resource Allocation**

| Architecture        | Throughput (req/s) | p99 Latency | RAM Used |
|---------------------|--------------------|-------------|----------|
| Mesh Microservices  | 22,400             | 320 ms      | 34.2 GB  |
| RouteOne-S Monolith | 418,000            | 4.2 ms      | 1.8 GB   |

### 4.1 CPU Instruction Analysis

Hardware performance counter analysis (`perf stat`) indicates that in
the microservice baseline, approximately 68% of CPU cycles are consumed
by network socket handling, string parsing, TLS renegotiation, and
memory management. In RouteOne-S, active compute cycles represent 89% of
core usage, indicating that mechanical waste has been substantially
eradicated.

## 5. Related Work & Theological Divergence

- **Microservice Frameworks (Spring Cloud, Dapr):** These architectures
  assume hardware failure is best managed by introducing dynamic
  software orchestration. We argue this shifts the failure domain from
  predictable hardware degradation to unpredictable network topologies.

- **Actor Systems (Akka, Orleans):** While Actor runtimes correctly
  target colocation and message passing, they frequently introduce
  excessive actor-lifecycle overhead and heap fragmentation unless
  strictly bounded.

- **The Unix Philosophy (McIlroy et al.):** RouteOne-S re-applies the
  original Unix principle of small, distinct tools connected via direct
  pipelines, but rejects the modern bastardization that treats
  independent IP networks as the replacement for stdin/stdout streams.

## 6. Conclusion

Digital *muda* is rarely a hardware limitation; it is an architectural
choice born of premature distribution. RouteOne-S demonstrates that when
software is stripped of unnecessary intermediaries, contemporary x86 and
ARM processors can handle high-throughput enterprise workloads within a
single, coherent execution context. The highest-performing network call
remains the one that was never made.
