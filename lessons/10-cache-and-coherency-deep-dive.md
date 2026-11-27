# Lesson 10 — Cache and Coherency: From SRAM Tags to Multi-Agent Consistency

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 10 of 38  
**Suggested duration:** 60 minutes  
**Level:** Intermediate to advanced  
**Focus:** Cache microarchitecture, cache-line state, coherence protocols, CPU/DMA interaction, ACE/CHI concepts, verification and post-silicon diagnosis.

> **Scope note:** Cache geometry, replacement policy, queue depths, register maps and exact protocol encodings are implementation-specific unless a cited architecture specification says otherwise. This lesson explains the common engineering model and labels examples as examples. It is not a substitute for the relevant Arm, RISC-V, interconnect or processor implementation manuals.

---

## 1. Learning objectives and 60-minute plan

By the end of this session, you should be able to:

1. Decode an address into cache tag, set index and byte offset.
2. Explain a cache hit, miss, refill, writeback and eviction at the controller level.
3. Distinguish write-through/write-back and write-allocate/no-write-allocate.
4. Explain why multiple private caches need a coherence protocol.
5. Walk through MSI/MESI-style read sharing and ownership acquisition.
6. Compare broadcast snooping with directory-based coherence.
7. Explain why a DMA engine can read stale data or overwrite newer CPU data on a non-coherent system.
8. Select the correct ownership-transfer and cache-maintenance sequence for common DMA directions.
9. Identify the integration and verification requirements for a coherent multi-core SoC.

| Time | Topic |
|---:|---|
| 0–5 min | Why caches exist; history and locality |
| 5–15 min | Cache arrays, address fields, mapping and replacement |
| 15–25 min | Read/write misses, write policies, MSHRs and hierarchy |
| 25–38 min | Coherence, MESI, snooping and directories |
| 38–47 min | DMA coherency, barriers, ACE/CHI and software contract |
| 47–55 min | Performance calculations, verification and silicon debug |
| 55–60 min | Exercises and interview questions |

## 2. Historical backstory: why cache memory appeared

### 2.1 The original problem was a growing speed gap

A processor can execute internal operations much faster than a large, inexpensive main memory can deliver arbitrary data. Increasing main-memory capacity does not automatically reduce its access latency. A system therefore needs a small, fast store close to the processor while retaining a larger, slower backing store.

A particularly well-documented milestone is the IBM System/360 Model 85. J. S. Liptay's 1968 IBM Systems Journal paper describes a storage hierarchy in which a small semiconductor cache was integrated into the processor to bridge the gap between the processor cycle and much slower main storage. The cache was not part of the programmer-visible address space; hardware moved the currently useful portions of main storage into it automatically. The paper also describes the simulation and design studies used to choose the cache organization and parameters.

This is a documented example, not a claim that no earlier machine used any form of buffering or high-speed storage. The useful historical lesson is that cache design emerged from measured system-level performance trade-offs, not simply from a desire to add another memory block.

**Source:** [Liptay, “Structural aspects of the System/360 Model 85, II: The cache” (1968)](https://www.ece.ucdavis.edu/~vojin/CLASSES/EEC272/S2005/Papers/IBM360-Liptay_68.pdf).

### 2.2 From one processor to multiprocessors

A single processor can keep a private cached copy of memory without another processor independently modifying that same location. Once several agents have private caches, they can hold multiple copies of one physical cache line. A write by one agent can make the other copies stale.

The problem changed from “how do we make memory appear faster?” to two related questions:

- **Coherence:** For one memory location, how do agents avoid using conflicting stale copies?
- **Consistency / memory ordering:** In what order may agents observe operations to different locations?

These are different properties. A coherent system can still permit relaxed ordering between independent addresses. Software synchronization, architectural memory models and barriers remain important.

In 1983, James R. Goodman published “Using cache memory to reduce processor-memory traffic,” an early influential treatment of snooping cache protocols. The later DASH work, published in 1990, described a scalable shared-memory multiprocessor using a distributed directory rather than relying on a single broadcast point. It illustrates the scaling pressure that encouraged point-to-point coherence messages.

**Sources:** [Goodman paper reference](https://doi.org/10.1145/800046.801647); [Lenoski et al., DASH directory-based coherence protocol (1990)](https://doi.org/10.1145/325096.325132).

### 2.3 Why modern SoCs use coherent fabrics

Modern SoCs combine CPU clusters, GPUs, DSPs, NPUs, display engines, storage controllers and DMA-capable peripherals. These agents share memory but have different latency, bandwidth and cache requirements.

Arm announced AMBA 5 CHI on 3 June 2013 to support high-frequency, non-blocking coherent communication among multiple processors and scalable interconnects. Arm describes CHI as separating protocol and transport layers so implementations can use different topologies and make power/performance/area trade-offs.

**Sources:** [Arm CHI announcement, 3 June 2013](https://www.arm.com/company/news/2013/06/arm-announces-amba-5-chi-specification-to-enable-high-performance-highly-scalable-system-on-chip); [Arm AMBA 5 overview](https://www.arm.com/architecture/system-architectures/amba/amba-5).

**Engineering interpretation:** As core counts and heterogeneous agents grow, a simple shared snoop bus becomes harder to scale in bandwidth, wire loading and power. Directories, snoop filters, distributed home nodes and packetized coherent fabrics are ways to avoid broadcasting every coherence request to every agent. The exact implementation is vendor- and product-specific.

---

## 3. Cache fundamentals: what is stored and why

### 3.1 Locality

Caches exploit two common workload properties:

- **Temporal locality:** Recently used instructions or data are likely to be used again.
- **Spatial locality:** Nearby addresses are likely to be used soon.

A cache transfers a block (also called a line) rather than only the requested byte or word. Fetching a line amortizes the cost of a memory transaction across nearby accesses.

### 3.2 Typical cache datapath

A conventional set-associative cache contains:

- **Data array:** The actual cache-line bytes.
- **Tag array:** Address tag associated with each line.
- **Valid bit:** Indicates whether the entry contains a usable line.
- **Dirty bit:** Indicates that a write-back line differs from its backing copy.
- **Coherence state:** For example, I/S/E/M in a MESI-style design.
- **Replacement metadata:** LRU approximation, pseudo-LRU, random or another policy.
- **Controller:** Handles lookup, miss allocation, refill, eviction, snoops and maintenance.
- **Queues:** MSHRs, write buffers, refill buffers, victim buffers and snoop queues as required by the implementation.

A cache may be split into instruction and data caches (Harvard-style L1) and unified at a lower level. A unified cache can use capacity dynamically for instructions or data, while split caches provide separate access paths and avoid some instruction/data port conflicts.

### 3.3 Address decomposition

For a conventional physically indexed, physically tagged cache:

- Byte offset selects a byte within a line.
- Set index selects a set.
- Tag identifies which memory block occupies a way in that set.

**Worked example — illustrative geometry**

Assume:

- Capacity = 32 KiB
- Associativity = 4 ways
- Line size = 64 bytes
- Physical address width = 40 bits

Calculations:

- Number of lines = 32,768 / 64 = 512
- Number of sets = 512 / 4 = 128
- Offset bits = log2(64) = 6
- Index bits = log2(128) = 7
- Tag bits = 40 − 7 − 6 = 27

Address format:

    [39:13] TAG (27 bits) | [12:6] SET (7 bits) | [5:0] OFFSET (6 bits)

This is a teaching example. Real caches may use virtual indexing, physical tags, hashing, bank selection, skewing or implementation-specific address transformations. Page size and alias rules matter for virtually indexed caches.

### 3.4 Direct-mapped, set-associative and fully associative

| Organization | Placement rule | Advantage | Cost / risk |
|---|---|---|---|
| Direct-mapped | One possible line per index | Simple, low hit latency and area | Conflict misses can be high |
| N-way set-associative | Any of N ways in selected set | Reduces many conflict misses | Tag comparisons and way selection cost |
| Fully associative | Any line can hold any block | Flexible placement | Expensive lookup and replacement logic |

A replacement algorithm chooses a victim only when all candidate ways in the selected set are occupied. Exact replacement behavior is not dictated by a universal cache protocol.

### 3.5 Cache miss categories

- **Compulsory:** First reference to a block.
- **Capacity:** Working set exceeds available cache capacity.
- **Conflict:** Multiple active blocks map to the same set.
- **Coherence:** A line was invalidated or downgraded because another agent accessed it.

The coherence category is particularly important in multi-core and CPU-plus-DMA workloads.

---

## 4. Read path, write path and miss handling

### 4.1 Read hit

1. CPU sends address and access attributes.
2. Index selects a set.
3. Tag array compares the requested tag against valid ways.
4. A matching way produces a hit.
5. Offset and byte enables select the requested data.
6. Data returns to the load pipeline.

The tag and data arrays may be accessed in parallel or in stages. Some implementations use way prediction, banking or pipelining. Do not assume a universal one-cycle hit latency.

### 4.2 Read miss and refill

A simplified miss sequence:

1. Tag lookup finds no valid matching line.
2. Controller checks whether a miss entry is available.
3. Allocate an MSHR (miss-status holding register) or equivalent transaction record.
4. Choose a victim if the target set is full.
5. If the victim is dirty, schedule writeback.
6. Issue a line refill to the next level.
7. Accept refill beats into a refill buffer or data array.
8. Install tag, valid and initial coherence state.
9. Wake the waiting load and merge any compatible secondary misses.

A non-blocking cache can continue servicing independent hits and can track multiple misses. A blocking cache may stop accepting new requests during a miss. MSHR count, fill-buffer depth, bank conflicts and downstream credits all influence sustained performance.

### 4.3 Write policies are two independent choices

**Write-through versus write-back**

- Write-through: A store updates the cache and sends the write toward the next level according to the implementation's ordering/buffering rules.
- Write-back: A store updates the cache and marks the line dirty. The modified line is written to the next level when required, often on eviction or explicit maintenance.

**Write-allocate versus no-write-allocate**

- Write-allocate: A store miss first obtains the line, then modifies it.
- No-write-allocate: A store miss may bypass allocation and send the write downstream.

These choices are orthogonal. A common combination is write-back plus write-allocate, but it is not universal.

### 4.4 Dirty eviction

A dirty victim cannot simply be discarded: its newest bytes may exist only in that cache. The controller must preserve them by writing them back or transferring ownership/data according to the coherence protocol.

A common RTL bug is freeing the victim entry before the writeback has safely captured its tag, address, data and byte-valid information. Another is allowing a refill to overwrite a line while a pending writeback still references the same physical data-array way.

### 4.5 Stores, byte enables and partial writes

A store may update only part of a cache line. The cache must preserve untouched bytes. The implementation may use byte write masks, merge buffers or read-modify-write sequencing. Verification must cover stores crossing word boundaries, cache-line boundaries and page boundaries.

---

## 5. Cache hierarchy and organization trade-offs

A common CPU hierarchy is:

    CPU core
      ├── L1 instruction cache (private)
      └── L1 data cache (private)
                 |
             L2 cache (private or cluster-shared)
                 |
             L3 / system-level cache (optional, often shared)
                 |
             Memory controller → DRAM

Key design choices:

- **Inclusive:** Lower-level cache guarantees that lines present in upper levels are also represented below. This can simplify some snoop filtering, but eviction at the lower level may require invalidating upper copies.
- **Exclusive:** A line is intended to reside in only one level at a time. This can increase effective aggregate capacity but complicates movement and lookup.
- **Non-inclusive / non-exclusive:** No strict inclusion relationship is guaranteed. Directory or snoop-filter structures may track upper-level copies separately.

These labels describe hierarchy relationships, not coherence protocols by themselves.

A shared last-level cache may reduce DRAM traffic and provide a common point for directory metadata. It can also become a contention point between CPU, GPU, DSP and display traffic. Bank count, interleave, arbitration and QoS are system-level design decisions.

---

## 6. Cache coherence: the central multi-agent problem

### 6.1 Coherence invariant

For a cache line, a correct coherence system must prevent two agents from independently using incompatible writable copies. At a high level:

- If one agent has permission to modify a line, other agents must not retain permission to read or write a stale copy.
- If several agents have shared read-only copies, a writer must obtain the required exclusive/modified permission before changing the line.
- The latest data must be supplied by the correct owner or memory point when another agent requests it.

The detailed invariants, transient states, acknowledgements, ordering and retry rules depend on the selected protocol.

### 6.2 MSI and MESI stable states

| State | Meaning | Other copies allowed? |
|---|---|---|
| M — Modified | This cache owns the newest, dirty copy | No other valid copy in basic MESI |
| E — Exclusive | Clean copy with exclusive permission | No other valid copy |
| S — Shared | Clean readable copy | Yes, multiple readers |
| I — Invalid | No usable copy | Not applicable |

MESI adds the E state to MSI. If a line is read and no other cache has it, the requester may receive E rather than S. A later local write can transition E → M without a separate invalidation transaction.

**Important:** MESI is a teaching protocol model. Real implementations include transient states such as IS, IM, SM, eviction states, retry states and acknowledgement tracking. Never design a real controller from only the four stable-state labels.

### 6.3 Example: two cores read, then one writes

Assume memory initially contains X = 0.

1. Core 0 reads X. Its cache misses and receives the line in E if the coherence system establishes that no other cache holds it.
2. Core 1 reads X. Its request causes the system to share the line. Core 0 may downgrade E → S and Core 1 receives S.
3. Core 0 stores X = 1. It must obtain exclusive write permission. The coherence system invalidates Core 1's copy and collects required acknowledgements.
4. Core 0 updates the data and owns the line in M.
5. If Core 1 later reads X, the protocol obtains the latest value from the owner or another architecturally valid source and updates states.

The exact message names and response ordering vary by protocol.

### 6.4 MESI versus MOESI

MOESI adds an O (Owned) state. It permits one cache to retain responsibility for a dirty shared line while other caches hold shared copies. This can avoid forcing an immediate writeback to memory on every transition from a modified owner to a shared state.

More states can reduce traffic or latency in some workloads, but increase transient-state complexity, verification effort and corner cases.

### 6.5 False sharing

Two cores may modify different words that happen to occupy the same cache line. There is no logical data race between those words, but the coherence protocol operates at line granularity. Ownership repeatedly moves between cores, causing invalidations and performance collapse.

Example:

- Core 0 updates counter A at offset 0.
- Core 1 updates counter B at offset 8.
- Both counters are in one 64-byte line.
- The line repeatedly migrates even though the counters are independent.

Mitigations include padding/alignment, per-core counters followed by aggregation, sharding and reducing write frequency.

---

## 7. How coherence is implemented: snoop versus directory

### 7.1 Broadcast snooping

A requester broadcasts a coherence request. Other caches inspect their tags and respond if they have a matching line.

Advantages:
- Conceptually simple.
- No large sharer directory is required.
- Natural fit for small systems with a shared snoop domain.

Limitations:
- Broadcast traffic grows with agent count.
- Every snooping cache spends power checking requests.
- Shared wires and response collection can become timing bottlenecks.

### 7.2 Directory-based coherence

A directory tracks the state and possible sharers/owner for each line. It sends point-to-point messages to the relevant agents instead of broadcasting every request to all caches.

A simplified directory entry may contain:

- State / owner information.
- Sharer vector or compressed sharer representation.
- Pending transaction or transient state.
- Dirty/ownership metadata.

Advantages:
- Better scaling for larger systems.
- Targeted invalidations and forwards.
- Fits naturally with distributed home nodes and NoC fabrics.

Costs:
- Directory storage and lookup latency.
- More complex transient states and races.
- Need to handle stale or compressed sharer information safely.
- Directory/home-node congestion and backpressure.

The 1990 DASH paper is a classic public example of distributed directory-based coherence. Its key contribution was to avoid a single broadcast point by using point-to-point messages.

### 7.3 Snoop filters and system-level caches

A snoop filter remembers which agents may hold a line. It can reduce unnecessary snoops. A system-level cache may combine data storage and directory metadata, but not every design does so.

A false positive in a conservative snoop filter may cause an unnecessary snoop; a false negative that loses a real sharer can violate correctness. Therefore, filter maintenance and replacement rules are correctness-critical.

---

## 8. Coherent interconnects: ACE and CHI concepts

### 8.1 Why AXI alone is not the whole coherence solution

Ordinary AXI transactions move reads and writes. A coherent interface also needs a way to request permission, snoop other caches, transfer dirty data and communicate state/ownership changes.

Arm ACE extends AXI with additional coherency channels and attributes. A common ACE channel view includes:

- AC — snoop address channel, sent to a coherent master.
- CR — snoop response channel.
- CD — snoop data channel.
- Existing AXI read/write channels carry ordinary data traffic and other required responses.

The precise signals, transaction encodings, barriers and ordering rules are specified by the relevant AMBA ACE issue. Do not infer a complete ACE implementation from this summary.

### 8.2 CHI: a scalable coherent protocol family

Arm describes CHI as a layered coherent interface, separating protocol from transport so different topologies can be implemented. It supports high-frequency non-blocking traffic and scalable interconnects such as crossbars and meshes. The official Arm overview also describes requester identifiers, transaction identifiers, QoS and optional features including cache stashing and far atomics.

At a conceptual level, a CHI-based system contains:

- Request nodes (for example, CPU clusters or coherent accelerators).
- Home nodes that serialize and manage coherence for address regions.
- Memory nodes or memory controllers.
- Network transport carrying request, response, data and snoop-related traffic.

**Do not confuse this conceptual node taxonomy with exact CHI channel names or encodings.** Use the official CHI specification issue and the actual IP integration manual for protocol-level implementation.

### 8.3 Coherent versus non-coherent DMA

A coherent DMA-capable agent participates in the system's coherence mechanism. Its accesses are coordinated with CPU caches according to the fabric's coherency rules.

A non-coherent DMA engine reads/writes memory without automatically snooping or invalidating CPU cache lines. Software, an IOMMU/cache-coherent bridge, or explicit hardware maintenance must establish visibility.

Coherency is not the same as ordering. Even coherent descriptor memory may still require barriers before a device doorbell is written.

---

## 9. DMA coherency: the most common real-system failure

This section is essential for audio, camera, display, networking and storage IPs.

### 9.1 CPU produces data; DMA consumes it

Failure scenario:

1. CPU writes a buffer.
2. The cache uses write-back, so the newest bytes remain dirty in L1/L2.
3. CPU programs a DMA source address and starts the engine.
4. Non-coherent DMA reads DRAM.
5. DRAM still contains old bytes.

The DMA engine receives stale data because the CPU's dirty cache line was never made visible at the DMA observation point.

**Required concept:** Before handing a non-coherent DMA_TO_DEVICE buffer to the device, the platform's DMA mapping/synchronization API must make CPU writes visible to the device. On a non-coherent architecture this normally includes the required clean/writeback operation.

### 9.2 DMA produces data; CPU consumes it

Failure scenario:

1. CPU previously read buffer B, so a clean copy is cached.
2. DMA writes new data to DRAM.
3. CPU reads B again.
4. CPU hits its old cached line and sees stale data.

The required ownership transition must ensure the CPU does not use a stale cache copy. For a non-coherent DMA_FROM_DEVICE mapping, the platform API performs the required synchronization/invalidation at the appropriate point.

### 9.3 Bidirectional buffers

If both CPU and device may read and write the same region, ownership transitions are more complex. The driver must synchronize before and after device access as required by the platform DMA API. Avoid simultaneous unsynchronized CPU/device writes to the same cache line.

### 9.4 Linux DMA API: use the platform abstraction

Linux distinguishes coherent DMA mappings and streaming DMA mappings.

- Coherent DMA allocations are designed so CPU and device observe updates without explicit cache flushing for coherency. They still require correct ordering/barriers when publishing descriptors or ringing a doorbell.
- Streaming mappings use a direction such as DMA_TO_DEVICE, DMA_FROM_DEVICE or DMA_BIDIRECTIONAL.
- When a streaming mapping is reused and ownership changes, use the appropriate dma_sync_*_for_cpu() and dma_sync_*_for_device() operations.
- Use the DMA address returned by the DMA API; do not assume it equals a CPU virtual or physical address. An IOMMU or bus translation may intervene.

The Linux documentation explicitly warns that coherent DMA does not eliminate the need for memory barriers. It also gives the required sync pattern for reused streaming buffers.

**References:** [Linux DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html); [Linux DMA API](https://docs.kernel.org/core-api/dma-api.html).

### 9.5 Direction-specific ownership table

| Transfer | Before device starts | After device completes, before CPU reads/writes |
|---|---|---|
| CPU → device (DMA_TO_DEVICE) | Make CPU-produced data visible to device | Usually no post-sync needed if CPU does not reuse/modify until ownership returns; follow API contract |
| Device → CPU (DMA_FROM_DEVICE) | Ensure old dirty CPU data cannot later overwrite device output; follow mapping API | Synchronize for CPU so stale cache lines are not used |
| Both directions | Synchronize for device before handoff | Synchronize for CPU after completion; repeat on each ownership transfer |

This is a conceptual guide, not a replacement for the exact operating-system DMA API or architecture rules. Do not call a raw cache invalidate on an arbitrary range without considering dirty lines, alignment, cache-line sharing and other CPUs.

### 9.6 Cache-line sharing hazard in DMA buffers

Suppose cache line size is 64 bytes and a DMA buffer occupies bytes 16–47 of that line. CPU metadata occupies bytes 0–15 and 48–63.

A range invalidate may operate on the whole 64-byte line. If the CPU has dirty metadata in the same line, invalidating it can discard unrelated CPU updates. Conversely, cleaning a partially shared line can write stale neighboring bytes over data just written by DMA.

**Design/driver rule:** Align and isolate DMA buffers to cache-line boundaries when the platform requires it. Keep CPU-owned metadata out of cache lines that device DMA owns.

### 9.7 Memory barriers are not cache maintenance

A memory barrier orders memory operations; it does not necessarily write dirty cache lines to DRAM or invalidate stale lines.

A common descriptor sequence is:

1. CPU fills descriptor fields.
2. CPU performs the required DMA sync/cache maintenance if memory is non-coherent.
3. CPU executes the appropriate write barrier.
4. CPU writes the device doorbell register.
5. Device fetches descriptor and payload.

The exact barrier and MMIO accessor depend on the OS, architecture and device contract. Linux memory-barrier documentation explains that MMIO can bypass cache and that DMA coherency is a separate concern.

**Source:** [Linux kernel memory barriers — cache coherency vs DMA](https://docs.kernel.org/core-api/wrappers/memory-barriers.html).

---

## 10. Cache maintenance: clean, invalidate and clean+invalidate

Terminology varies between architecture manuals and operating systems. A useful conceptual distinction:

- **Clean:** Write dirty data toward the required point of visibility; retain a valid cache copy if the architecture operation specifies that behavior.
- **Invalidate:** Mark a cache copy unusable. It does not inherently preserve dirty data.
- **Clean + invalidate:** Preserve modified data to the required point, then remove the local copy.

On a write-back cache, invalidating a dirty line without first preserving it can lose data. Arm's programmer guide explicitly distinguishes cleaning (write dirty data out) from invalidating (clear valid state) and warns that simple invalidation of dirty data is generally incorrect.

Arm documentation also uses:
- **Point of Coherency (PoC):** A point where agents that can access memory are guaranteed to observe the same copy.
- **Point of Unification (PoU):** A point where instruction and data views are unified for the relevant processor system.

The exact points and available operations depend on the architecture and implementation.

**Sources:** [Arm Cortex-A Series Programmer's Guide: invalidating and cleaning cache memory](https://support.arm.com/documentation/den0013/0400/Caches/Invalidating-and-cleaning-cache-memory); [Cortex-A5 cache operations and PoC/PoU definitions](https://support.arm.com/documentation/ddi0434/b/System-Control/About-system-control/Cache-Operations-Registers).

### 10.1 RISC-V cache-block management

RISC-V's cache-block management extension defines operations such as:

- CBO.INVAL
- CBO.CLEAN
- CBO.FLUSH

Related extensions define CBO.ZERO and prefetch operations. The existence and availability of these instructions depend on the implemented extensions, privilege configuration and platform contract. Software must not assume every RISC-V processor implements every cache-block operation.

**Source:** [RISC-V Unprivileged ISA specification — Cache-Block Management Operations](https://docs.riscv.org/reference/isa/v20240411/_attachments/riscv-unprivileged.pdf).

---

## 11. Address attributes, translation and security

Cacheability and shareability are not just local cache settings. They are system-wide contracts.

Important attributes can include:

- Cacheable versus non-cacheable.
- Write-back versus write-through policy.
- Shareable domain.
- Device versus normal memory type.
- Execute-never / access permissions.
- Secure/non-secure or privilege attributes.
- Inner/outer cacheability and ordering attributes where supported.

The CPU MMU, page tables, IOMMU, interconnect and memory system must agree on how a physical region is accessed. Mapping the same physical memory with incompatible cacheability attributes through different virtual aliases can produce architecturally unsafe behavior.

Security considerations:
- Cache timing can leak information through side channels.
- Speculative accesses may leave microarchitectural traces even if architecturally squashed.
- Cache partitioning, way lockdown, non-cacheable mappings and constant-time software can be mitigations in specific threat models.
- Cache maintenance and debug access may be privilege/security controlled.

This is a system security topic, not simply a cache RTL feature.

---

## 12. Performance engineering: calculate rather than guess

### 12.1 Average memory access time

For a simple two-level hierarchy:

AMAT = L1 hit time + L1 miss rate × (L2 hit time + L2 miss rate × memory penalty)

Illustrative assumptions:

- L1 hit time = 1 cycle.
- L1 miss rate = 5%.
- L2 hit time = 8 cycles.
- L2 local miss rate = 10%.
- DRAM penalty after L2 miss = 80 cycles.

AMAT = 1 + 0.05 × (8 + 0.10 × 80)  
AMAT = 1 + 0.05 × 16  
AMAT = **1.8 cycles**

This simplified formula ignores overlap, writebacks, prefetching, queueing, bank conflicts and out-of-order execution. It is useful for first-order reasoning only.

### 12.2 Miss concurrency and bandwidth-delay product

Assume a 128-bit cache/memory datapath at 1 GHz:

- Bytes per cycle = 128 / 8 = 16 bytes.
- Peak datapath bandwidth = 16 × 1 GHz = 16 GB/s (decimal).
- Memory round-trip latency = 100 ns.
- Line size = 64 bytes.

Bandwidth-delay product = 16 GB/s × 100 ns = 1,600 bytes.  
Outstanding line-equivalents needed to cover that latency = 1,600 / 64 = **25 lines**.

This does not mean the cache must literally have 25 MSHRs: queueing, burst size, return bandwidth, memory scheduling, hit-under-miss behavior and target utilization affect the actual requirement. It demonstrates why a design with only one or two outstanding misses cannot approach a wide datapath's peak bandwidth at high latency.

### 12.3 Coherence traffic can dominate useful data traffic

A line that ping-pongs between cores may consume interconnect bandwidth for invalidations, acknowledgements, forwards and data responses. A benchmark with a small shared counter can therefore scale poorly even if the DRAM bandwidth is mostly idle.

Useful counters include:
- L1/L2 hit and miss counts.
- Refill and writeback beats.
- MSHR occupancy and full cycles.
- Snoop requests, snoop hits/misses and retries.
- Invalidations and ownership transfers.
- Directory lookup conflicts.
- Dirty evictions.
- Memory-controller queue occupancy.
- Per-agent bandwidth and QoS stall cycles.

Counter definitions are implementation-specific; check whether they count requests, lines, beats, retries or cycles.

---

## 13. SoC integration checklist

Before integrating a cache or coherent agent, answer these questions.

### Addressing and attributes
- Is the cache physically or virtually indexed/tagged?
- What is the line size at every cache level?
- Are CPU and DMA addresses translated by an IOMMU?
- Which regions are cacheable, shareable or device memory?
- Are aliases with conflicting attributes possible?

### Coherence
- Which agents participate in hardware coherence?
- Is DMA coherent, non-coherent or selectively coherent?
- What is the coherency point and which home node owns each address range?
- Are snoop filters/directories present?
- What happens on reset, power collapse, hot-plug or agent disable?
- Can a coherent request be blocked by power-gated or clock-gated agents?

### Clocks, reset and power
- Are cache tags/data arrays reset, invalidated or initialized by firmware?
- Are ECC/parity bits initialized before cache use?
- Does a power-domain transition require clean/invalidate?
- Can snoops reach a sleeping cluster?
- Is retention supported for tag and data arrays?
- Is wake-up sequencing sufficient to restore coherence participation?

### Software and debug
- Does firmware enable caches before or after memory training?
- Are DMA buffers allocated with the correct API and attributes?
- Are descriptor ownership and doorbell ordering correct?
- Are cache maintenance operations performed to the correct point?
- Can debug firmware inspect cache state without changing it?

---

## 14. Verification strategy: prove the invariants, not just the hits

### 14.1 Directed tests

1. Cold read miss followed by repeated read hits.
2. Store hit in a clean line.
3. Store miss with write allocate.
4. Dirty eviction and refill to the same set.
5. Conflict thrash across all ways.
6. Multiple misses to the same line (secondary miss merging).
7. Misses to different banks and sets.
8. Backpressure during refill and writeback.
9. ECC/parity error injection.
10. Cache maintenance while requests are outstanding.

### 14.2 Coherence tests

- Two cores read the same line.
- One core writes while another holds S.
- Read from a line owned in M by another core.
- Two writers contend for ownership.
- Eviction while a snoop is pending.
- Simultaneous read miss and invalidation.
- Directory replacement and snoop-filter eviction.
- Retry, NACK and replay paths.
- Power-down/wake-up with lines held by other agents.
- Coherent DMA read/write against dirty CPU lines.

### 14.3 Assertions and formal properties

Useful properties include:

- A line in M state must not coexist with another valid writable copy of the same line.
- A line in E state must not have another valid copy in the coherence domain.
- A response must match an outstanding transaction and requester.
- A dirty victim's data must not be discarded before its writeback/ownership transfer is safely accepted.
- A line cannot be allocated valid before its required data and metadata are correct.
- Every accepted request eventually completes or takes a defined error/retry path under stated fairness assumptions.
- A snoop response must correspond to the correct address and transaction.
- Reset must leave no stale transaction capable of corrupting a newly initialized cache state.

These are conceptual properties. Translate them into assertions using the actual RTL state, transaction IDs and protocol rules. Liveness proofs require explicit assumptions about downstream readiness and fairness.

### 14.4 Coverage points

- Every stable-state transition.
- Every important transient-state exit.
- All pairwise agent interactions.
- Clean and dirty eviction.
- Snoop hit/miss and forwarded data.
- Same-line and different-line concurrency.
- Backpressure at each channel.
- Queue full/empty boundaries.
- Reset during idle and outstanding activity.
- Cache-line boundary and page boundary accesses.

---

## 15. Post-silicon debug: symptom to likely root cause

| Symptom | Likely causes | First debug action |
|---|---|---|
| CPU sees old DMA output | Missing/incorrect sync, wrong DMA direction, stale alias | Inspect mapping attributes and DMA ownership sequence |
| DMA reads old CPU input | Dirty CPU line not cleaned to device-visible point | Check DMA_TO_DEVICE mapping/sync |
| DMA output later reverts | Dirty CPU line written back after device wrote memory | Check pre-DMA ownership handoff and cache-line sharing |
| Works with caches disabled | Coherency/maintenance/ordering/alias issue | Compare cacheable and non-cacheable mappings |
| Only fails on multi-core | Missing snoop, directory race, false sharing, barrier issue | Trace line address, owner and sharer transitions |
| Random data corruption under load | Refill/writeback collision, MSHR/tag race, ECC or CDC | Correlate transaction IDs and line addresses |
| Performance collapses with core count | False sharing, invalidation storms, directory contention | Count ownership transfers and snoop traffic |
| Hangs only during low-power entry | Pending snoop or dirty line lost across power transition | Trace quiesce, clean, retention and wake protocol |
| Self-modifying code executes old instructions | Data clean / instruction invalidate / synchronization sequence incomplete | Check PoU operations and instruction synchronization |

### 15.1 Practical post-silicon trace recipe

For one failing physical cache line, capture:

1. Physical line address and request attributes.
2. Requesting agent and transaction ID.
3. Current owner and sharer set from directory/snoop-filter trace if available.
4. Cache state before and after each request.
5. Snoop address, response and data events.
6. Refill/writeback request and response.
7. Memory-controller write/read ordering.
8. DMA descriptor address, buffer address, length and direction.
9. CPU cache-maintenance and barrier sequence.
10. Power/clock/reset state of every coherence participant.

Do not start by dumping the entire cache. First isolate one line and reconstruct its ownership history.

---

## 16. Worked SoC example: audio DSP playback buffer

Assume a CPU prepares an audio buffer in DRAM and a DSP DMA engine reads it over an SoC interconnect.

### Non-coherent path

1. CPU writes PCM samples into a cacheable buffer.
2. Some cache lines remain dirty in CPU L1/L2.
3. CPU programs the DSP DMA descriptor.
4. Firmware performs the platform-required DMA_TO_DEVICE mapping/synchronization.
5. Firmware orders descriptor writes before the DMA doorbell.
6. DSP DMA fetches descriptor and samples.
7. DSP consumes the correct samples.

If step 4 is missing, the DSP may read stale DRAM data even though the CPU debugger shows the correct samples in its cache.

### Coherent path

If the DSP DMA interface is a coherent agent and the address range is configured for the correct coherent domain, the fabric coordinates CPU cache state with the DSP's read. Software still must obey descriptor publication ordering and the platform's memory model.

### Debug experiment

Run the same deterministic audio pattern under three configurations:

- Cache enabled, correct DMA mapping/synchronization.
- Cache enabled, intentionally omit synchronization in a controlled test.
- Non-cacheable buffer mapping.

Compare DMA-captured samples, CPU buffer contents, memory dump and coherence/performance counters. The intentionally broken case is a lab demonstration only; never ship it as a workaround.

---

## 17. Reference implementations and study material

| Resource | What to inspect |
|---|---|
| [gem5 MESI Two-Level](https://www.gem5.org/documentation/general_docs/ruby/MESI_Two_Level/) | L1 private / L2 shared directory model, stable and transient states, DMA controller |
| [gem5 classic cache coherence](https://www.gem5.org/documentation/general_docs/memory_system/classic-coherence-protocol/) | MOESI-style snooping model and hierarchy trade-offs |
| [OpenPiton](https://github.com/PrincetonUniversity/openpiton) | Open manycore system with distributed coherence and scalable architecture |
| [Arm AMBA 5 / CHI](https://www.arm.com/architecture/system-architectures/amba/amba-5) | Official coherent interconnect architecture overview |
| [Linux DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html) | Real driver-facing ownership and synchronization contract |
| [Linux memory barriers](https://docs.kernel.org/core-api/wrappers/memory-barriers.html) | Ordering versus cache coherency |
| [RISC-V ISA manual](https://docs.riscv.org/reference/isa/v20240411/_attachments/riscv-unprivileged.pdf) | Cache-block management instruction family |

Reference models are not normative specifications. For example, gem5's MESI Two-Level model is a particular directory protocol model, not a statement that every commercial processor uses that exact organization.

---

## 18. Hands-on exercises

### Exercise A — Decode a cache address

A 64 KiB, 8-way cache uses 64-byte lines and a 48-bit physical address.

Calculate:
1. Number of lines.
2. Number of sets.
3. Offset bits.
4. Index bits.
5. Tag bits.

**Check:** 1,024 lines; 128 sets; 6 offset bits; 7 index bits; 35 tag bits.

### Exercise B — Trace MESI

Two cores start with no copy of address X.

1. Core 0 reads X.
2. Core 1 reads X.
3. Core 0 writes X.
4. Core 1 reads X.

Draw the stable-state changes and list the messages/acknowledgements your chosen protocol model requires. Include transient states in your design, even if the first diagram only shows stable states.

### Exercise C — Diagnose DMA stale data

A camera DMA writes a frame into DRAM. CPU reads the old frame after completion.

Create a checklist covering:
- DMA direction and mapping API.
- Cacheability and shareability.
- IOMMU address translation.
- Cache-line alignment.
- Completion interrupt ordering.
- CPU-side synchronization.
- Whether the camera port is coherent.

### Exercise D — Estimate MSHRs

A system has a 64-byte line, 12 GB/s target bandwidth and 120 ns memory latency.

Calculate the line-equivalent concurrency required to cover the bandwidth-delay product. Explain why the actual MSHR count may need to be higher or lower.

### Exercise E — RTL verification

Write assertions or pseudocode checks for:
- No M/E line has another writable copy.
- Dirty eviction data is preserved.
- A refill cannot install stale data after a newer invalidation.
- Every snoop response is associated with the correct line and request.
- Reset safely cancels or drains outstanding transactions.

---

## 19. Interview questions

1. Why does a cache transfer a line instead of a single word?
2. Explain tag, index and offset with a numerical example.
3. What is the difference between a conflict miss and a coherence miss?
4. Explain write-back versus write-through.
5. Why are write-allocate and write-back not the same choice?
6. What problem does an MSHR solve?
7. What is the difference between coherence and consistency?
8. Explain the E state in MESI.
9. Why does MESI need transient states in real RTL?
10. What is false sharing and how does it affect performance?
11. Compare snooping and directory-based coherence.
12. What is a snoop filter and why must it be conservative?
13. Explain why non-coherent DMA can read stale data.
14. Why can invalidating a dirty cache line lose data?
15. Why are cache maintenance and memory barriers different?
16. What is the purpose of a coherent interconnect such as ACE or CHI?
17. What happens if a cache-owning power domain is shut down without a correct coherence transition?
18. How would you debug a failure that disappears when the cache is disabled?

---

## 20. Key takeaways

- A cache is a performance structure whose correctness depends on preserving the relationship between cached copies and backing memory.
- A cache line has data and metadata; tag/state transitions are as important as the datapath.
- Coherence maintains a valid ownership/sharing relationship for each line. Memory consistency controls the order in which operations are observed.
- MESI/MOESI stable states are only the beginning; real controllers need transient states, queues, retries and careful race handling.
- Snooping is attractive for small domains; directories and coherent fabrics help scale to many agents.
- Coherent DMA and non-coherent DMA require different system/software contracts.
- Cache clean/invalidate operations do not replace barriers, and barriers do not replace cache maintenance.
- DMA buffer alignment, cache-line sharing, IOMMU translation, power sequencing and descriptor ordering are common sources of silicon failures.
- Performance must be measured using miss, refill, snoop, queue and bandwidth counters—not inferred from cache capacity alone.

## 21. References and source discipline

1. J. S. Liptay, “Structural aspects of the System/360 Model 85, II: The cache,” IBM Systems Journal, 1968. [Paper PDF](https://www.ece.ucdavis.edu/~vojin/CLASSES/EEC272/S2005/Papers/IBM360-Liptay_68.pdf).
2. J. R. Goodman, “Using cache memory to reduce processor-memory traffic,” ISCA, 1983. [DOI](https://doi.org/10.1145/800046.801647).
3. D. Lenoski et al., “The directory-based cache coherence protocol for the DASH multiprocessor,” 1990. [DOI](https://doi.org/10.1145/325096.325132).
4. Arm, “AMBA 5 CHI specification announcement,” 3 June 2013. [Announcement](https://www.arm.com/company/news/2013/06/arm-announces-amba-5-chi-specification-to-enable-high-performance-highly-scalable-system-on-chip).
5. Arm, [AMBA 5 / CHI overview](https://www.arm.com/architecture/system-architectures/amba/amba-5).
6. Arm, [Cortex-A Series Programmer's Guide: cache cleaning and invalidation](https://support.arm.com/documentation/den0013/0400/Caches/Invalidating-and-cleaning-cache-memory).
7. Linux kernel, [DMA API HOWTO](https://docs.kernel.org/core-api/dma-api-howto.html).
8. Linux kernel, [Memory barriers](https://docs.kernel.org/core-api/wrappers/memory-barriers.html).
9. RISC-V International, [Unprivileged ISA specification — cache-block operations](https://docs.riscv.org/reference/isa/v20240411/_attachments/riscv-unprivileged.pdf).
10. gem5, [MESI Two-Level protocol](https://www.gem5.org/documentation/general_docs/ruby/MESI_Two_Level/).
11. Princeton University, [OpenPiton repository](https://github.com/PrincetonUniversity/openpiton).

**Interpretation policy:** Historical dates and architecture capabilities above are tied to cited public sources. Cache geometry, the numerical examples, the simplified state walkthroughs and the audio-DMA scenario are educational examples. Actual behavior must be checked against the applicable architecture specification, IP integration manual, SoC memory map and operating-system DMA contract.

---

**Next session — Lesson 11: SRAM, ROM and OTP**
- SRAM bitcell and macro organization.
- 6T cell read/write operation and sense amplifiers.
- Banking, ports, byte write masks and ECC.
- ROM boot structures and patch mechanisms.
- OTP/eFuse programming, sensing and lifecycle use.
- MBIST, repair, retention, scrambling and silicon characterization.
