# Lesson 06 — AXI Advanced: Bursts, Ordering, QoS, Atomics and Data Movement

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 06 of 38  
**Suggested duration:** 60 minutes  
**Level:** Advanced protocol, interconnect, RTL, verification and silicon performance  
**Primary focus:** AXI4 memory-mapped advanced behavior, with AXI4-Lite, AXI4-Stream and selected AXI5 extensions

> **Learning objective:** Given a transaction trace, calculate its byte addresses and lane enables, determine which responses may be reordered, identify whether a stall is caused by protocol backpressure or implementation limits, and construct a verification plan for a high-throughput AXI path.

---

## 1. One-hour lesson plan

| Time | Topic | Engineering outcome |
|---:|---|---|
| 0–6 min | AXI4 recap and evolution | Place AXI4/AXI5 features in historical context |
| 6–17 min | Burst addressing and 4KB rule | Calculate beat count, byte count and address sequence |
| 17–26 min | Unaligned transfers, WSTRB and width conversion | Reason about byte lanes and partial writes |
| 26–36 min | IDs, outstanding requests and ordering | Explain response association and permitted reordering |
| 36–43 min | QoS, protection, cache attributes and atomics | Understand system-level transaction attributes |
| 43–49 min | AXI4-Lite, AXI4-Stream and bridges | Select the correct interface for a data path |
| 49–56 min | Performance, RTL and verification | Size queues and find throughput bottlenecks |
| 56–60 min | Silicon debug and exercises | Turn symptoms into measurable hypotheses |

### Prerequisites

- Lesson 02: interconnect foundations.
- Lesson 03: APB register interfaces.
- Lesson 04: AHB pipelining and bursts.
- Lesson 05: AXI five-channel fundamentals.

### Outcomes

By the end, you should be able to:

1. Decode AxLEN, AxSIZE and AxBURST without confusing beats with bytes.
2. Calculate FIXED, INCR and WRAP address sequences.
3. Explain the 4KB burst-boundary restriction.
4. Explain byte strobes, partial writes and unaligned accesses.
5. Describe AXI4 write-data ordering and read-response reordering.
6. Size outstanding transactions using bandwidth-delay product.
7. Distinguish AXI4 exclusive accesses from AXI5 atomic transactions.
8. Select AXI4, AXI4-Lite or AXI4-Stream for a given IP.
9. Build assertions and tests for burst, ordering, backpressure and error cases.

---

## 2. Historical backstory: why AXI kept evolving

### 2.1 AXI3 to AXI4: simplify the common case and improve fabric utilization

AXI was introduced for high-frequency, high-bandwidth on-chip communication. The first generation of AXI provided independent address, data and response channels, transaction IDs, bursts and multiple outstanding transactions. These capabilities allowed a system to overlap work rather than serialize every operation.

AMBA 4, announced in 2010, brought AXI4, AXI4-Lite and AXI4-Stream. Arm's public material identifies three notable AXI4 improvements over AXI3: bursts up to 256 beats, QoS signaling and multiple region interfaces. AXI4-Lite provided a smaller register-oriented subset; AXI4-Stream provided a unidirectional data path without memory addresses or responses.

One particularly useful historical change was removal of write-data interleaving and the WID signal in AXI4. AXI3 could interleave write data from different write transactions using WID. Arm's specification explains that AXI4 requires all data beats for a write transaction to be consecutive on the W channel, in the same order as the associated write addresses. The WID field became unnecessary. Arm's public engineering discussion explains the trade-off: interleaving could use otherwise idle write-data cycles, but it added complexity to generate, track and de-interleave streams, and was not commonly used in many applications.

**Engineering interpretation:** the AXI4 change favors a simpler and more predictable write-data path. A multi-master interconnect still needs to arbitrate and buffer write streams, but it no longer has to merge arbitrary beat-level interleaving for AXI4 endpoints.

### 2.2 AXI4 to AXI5: add capabilities without assuming every endpoint implements all of them

Arm announced AXI5 and ACE5 in January 2018 as part of AMBA 5. The newer generation adds optional capabilities aligned with the broader AMBA 5 ecosystem, including atomic transactions, cache stashing, data protection and poison signaling, QoS acceptance, and persistent cache-maintenance operations. Later specification issues add optional features such as interface parity, unique-ID indication, memory partitioning and monitoring, memory tagging and additional transaction optimizations.

The key architectural lesson is that **AXI5 is capability-qualified**. An interface is not automatically assumed to support every optional feature simply because the product description says AXI5. The integration contract must state which properties are implemented at each endpoint and which are preserved by the interconnect or bridge.

### 2.3 Historical timeline

| Date | Public milestone | Why it matters |
|---|---|---|
| 2003 | AXI appears in AMBA 3 | Independent channels, bursts and multiple outstanding transactions |
| 2010 | AMBA 4 AXI4, AXI4-Lite and AXI4-Stream | Longer bursts, QoS, register-oriented and streaming subsets |
| 2011 | ACE and ACE-Lite | Add cache-coherency extensions for coherent systems |
| 2013–2014 | AMBA 5 CHI introduced | New coherent transport architecture for larger systems |
| 2018-01-29 | Arm announces AXI5 and ACE5 | Adds optional performance, atomic and reliability features |
| 2019-10-15 | Arm announces AXI/ACE Issue G | Clarifies ordering and adds unique-ID, read-data chunking, parity and related options |
| Later Issue H | Additional optional features | Includes MTE-related support and further transaction types/optimizations |

**Source discipline:** dates and named protocol features above are documented in Arm's public specifications and announcements. Explanations of why a trade-off is attractive are engineering interpretations unless explicitly attributed to Arm.

### 2.4 The design-rationale pattern

For each protocol generation, ask four questions:

1. **What was the bottleneck?** Examples: limited burst length, unnecessary signal count, interconnect ordering complexity, protection or RAS requirements.
2. **What changed in the interface?** A field, handshake, ordering rule or optional capability.
3. **Who pays the complexity cost?** Manager, Subordinate, bridge, interconnect, verification environment or software.
4. **What is the compatibility contract?** Can a legacy endpoint connect directly, or is a converter/adapter required?

This method is reusable for PCIe generations, DDR revisions, USB, SoundWire and other IPs.

---

## 3. Burst fundamentals: beats are not bytes

### 3.1 The three primary address attributes

For a memory-mapped AXI burst:

- AxLEN encodes the number of data beats minus one.
- AxSIZE encodes the maximum number of bytes per beat as a power of two.
- AxBURST selects FIXED, INCR or WRAP addressing.

Define:

\[
N = AxLEN + 1
\]

\[
B = 2^{AxSIZE}
\]

For a full-width, aligned transfer, the nominal payload is:

\[
\text{Payload bytes} = N \times B
\]

Example: AxLEN = 15 and AxSIZE = 3 means 16 beats, 8 bytes per beat and 128 nominal payload bytes.

Do not confuse:
- **Beat:** one data-channel transfer accepted on a clock edge.
- **Burst:** the group of beats associated with one address transaction.
- **Byte lane:** one byte position on the physical data bus.
- **Payload bytes:** the bytes actually enabled by the transfer and strobes.

### 3.2 AXI4 burst types

| AxBURST | Name | Address behavior | Typical use |
|---|---|---|---|
| 00 | FIXED | Address remains constant | Peripheral FIFO or repeated access to one location |
| 01 | INCR | Address advances by transfer size | Normal memory and DMA buffers |
| 10 | WRAP | Address increments and wraps at a power-of-two boundary | Cache-line and aligned circular windows |
| 11 | Reserved | Not a normal AXI4 burst type | Do not generate |

### 3.3 INCR address generation

For an INCR burst, the first transfer uses the programmed start address. Subsequent transfer addresses advance by the transfer size, with the address sequence following the protocol's alignment rules.

For aligned start addresses:

\[
A_i = A_0 + i \times B,\quad i=0,\ldots,N-1
\]

Example:
- Start address = 0x8000
- AxSIZE = 3, so B = 8 bytes
- AxLEN = 3, so N = 4

| Beat | Address |
|---:|---:|
| 0 | 0x8000 |
| 1 | 0x8008 |
| 2 | 0x8010 |
| 3 | 0x8018 |

Nominal payload = 4 × 8 = 32 bytes.

### 3.4 FIXED addressing

A FIXED burst repeatedly accesses the same address. This is useful for a peripheral FIFO register whose address does not increment for each item.

Example: a DMA engine writes four words to a transmit FIFO at 0x4000_1000. The address remains 0x4000_1000 for each beat.

**Integration warning:** a memory controller or bridge must not silently reinterpret a FIXED burst as INCR. Conversely, a DMA descriptor that accidentally programs FIXED instead of INCR can repeatedly overwrite one memory location or repeatedly access one FIFO entry.

### 3.5 WRAP addressing

A WRAP burst increments through a bounded window and wraps to the beginning of that window. The allowed WRAP burst lengths are 2, 4, 8 or 16 transfers, and the start address must satisfy the required transfer-size alignment.

For a valid aligned WRAP burst:

\[
W = N \times B
\]

\[
WrapBase = \left\lfloor\frac{StartAddress}{W}\right\rfloor \times W
\]

The address sequence advances within the W-byte window and wraps to WrapBase.

Example:
- Start = 0x1038
- B = 8 bytes
- N = 4
- W = 32 bytes
- WrapBase = 0x1020

| Beat | Address |
|---:|---:|
| 0 | 0x1038 |
| 1 | 0x1020 |
| 2 | 0x1028 |
| 3 | 0x1030 |

WRAP is useful for cache-line-oriented access patterns. It is not a general substitute for INCR DMA traffic.

### 3.6 AXI4 burst-length limits

For AXI4:
- INCR bursts can contain up to 256 beats.
- FIXED bursts are limited to 16 beats.
- WRAP bursts are limited to 16 beats and use lengths of 2, 4, 8 or 16 beats.

The AxLEN field is eight bits in AXI4, but the legal range depends on burst type. Do not infer that every AxBURST encoding can use every AxLEN value.

### 3.7 The 4KB boundary rule

An AXI burst must not cross a 4KB address boundary. This helps an interconnect or target avoid a single burst spanning two 4KB regions that may decode to different destinations or translation/protection regions.

Aligned example:
- Start = 0x0000_0FE0
- Beat size = 8 bytes
- Length = 4 beats
- Total = 32 bytes
- Byte range = 0x0FE0 through 0x0FFF

This burst stays within the 4KB region.

If the same start address used 8 beats:
- Total = 64 bytes
- End address = 0x101F
- The burst crosses the 0x1000 boundary and is illegal as one burst.

A Manager or DMA descriptor engine must split a larger request into legal bursts. A bridge or interconnect that creates new bursts must preserve the boundary rule.

**RTL review question:** Is the split calculation based on the actual byte address and transfer size, or is it incorrectly based only on the number of beats?

---

## 4. Unaligned accesses, byte lanes and WSTRB

### 4.1 Why byte strobes exist

A data bus transfers several byte lanes in parallel. A write may update only a subset of those bytes. WSTRB identifies which byte lanes contain valid write data for that beat.

For a data bus of D bits:

\[
ByteLanes = D/8
\]

A 128-bit interface has 16 byte lanes and therefore a 16-bit WSTRB.

A strobe bit of 1 enables the corresponding byte lane. A strobe bit of 0 means that byte lane must not update the destination storage for that transfer.

### 4.2 Partial-write example

Assume a 64-bit data bus:
- WDATA has 8 byte lanes.
- WSTRB has 8 bits.
- The Manager wants to update four byte lanes in one aligned bus word.

If lanes 2, 3, 4 and 5 are enabled, WSTRB is:

~~~text
WSTRB[7:0] = 8'b0011_1100
~~~ 

The Subordinate must preserve the disabled byte lanes. A register bank can implement this with per-byte write enables; SRAM wrappers may need byte-write support or a read-modify-write sequence if the macro lacks native byte enables.

### 4.3 Unaligned transfers

An unaligned transfer begins at an address that is not naturally aligned to the transfer size. AXI supports unaligned addressing, but the first transfer's byte-lane use is not necessarily a full set of lanes.

For example, a transfer size of 8 bytes does not imply that every first beat of every burst writes all 8 bytes. The first-beat address, bus width, lane mapping and WSTRB determine which byte locations are active.

Engineering rules:

1. Decode AxSIZE as the maximum transfer size, not a promise that all physical byte lanes are enabled.
2. Derive the byte-lane mapping from the AXI specification and the interface data width.
3. Do not hard-code WSTRB to all ones in a generic Subordinate.
4. Test every legal start-address offset for the supported transfer sizes.
5. Check that width converters preserve byte addresses and strobes.
6. Confirm how the local SRAM or register bank handles partial writes.

### 4.4 Narrow transfers on a wide bus

A 128-bit data bus can carry a 32-bit transfer. AxSIZE describes the transfer size, while WDATA still has the full interface width. Only the relevant lanes are meaningful for that transfer.

This distinction matters in:
- 32-bit control-register accesses on a 128-bit fabric.
- DMA engines moving byte or half-word data.
- Width converters between 32-bit peripheral bridges and 128-bit memory fabrics.
- ECC-protected SRAM where the physical ECC granularity may exceed the requested write width.

### 4.5 Width conversion

A width converter changes the number of byte lanes between two interfaces. Example: a 64-bit Manager talks to a 256-bit memory-side interface.

The converter must preserve:
- byte address and transfer size;
- burst semantics and ordering;
- WSTRB meaning;
- WLAST and RLAST behavior;
- response status;
- transaction IDs or a valid ID-remapping scheme;
- 4KB boundary restrictions;
- security and cache/protection attributes.

A downsizer may split one wide beat into multiple narrow beats. An upsizer may combine several narrow beats into a wide beat. The converter may need buffering to handle backpressure and partial groups.

**Common bug:** a width converter combines data correctly but shifts WSTRB by the wrong lane offset. Full-width tests pass; byte and half-word writes fail.

---

## 5. Transaction IDs, outstanding requests and ordering

### 5.1 Why IDs exist

A Manager may issue multiple requests before earlier requests complete. The response must be associated with the request that caused it.

- AWID tags a write-address transaction.
- BID identifies the write response.
- ARID tags a read-address transaction.
- RID identifies the read response.

The Subordinate and interconnect use these IDs, together with their own state, to route responses and enforce ordering rules.

### 5.2 Outstanding transactions

A transaction is outstanding after its address has been accepted and before its corresponding response has completed.

A simple endpoint might allow one outstanding read and one outstanding write. A high-performance memory controller may allow many requests in flight, subject to its resources and ordering constraints.

A typical implementation contains:
- request acceptance queue;
- ID and destination tracking;
- write-data collection state;
- read-return tracking;
- response buffering;
- reorder or ordering-control logic;
- timeout/error monitoring.

### 5.3 Read response ordering

Different read IDs can allow responses to be returned out of request order when the protocol ordering rules permit it.

Example:

~~~text
AR request 0: ARID = 2, address = A
AR request 1: ARID = 5, address = B

Possible response sequence:
R response for ID 5
R response for ID 2
~~~

The Manager uses RID to identify which request each response belongs to.

However, do not simplify this to “all different IDs may always reorder.” The ordering model includes transaction attributes, destination and interconnect behavior. Same-ID ordering requirements must be preserved where the specification requires them. Bridges that remap IDs must retain the required ordering and response association.

### 5.4 AXI4 write-data ordering

AXI4 does not support write-data interleaving. The W-channel data for write transactions must follow the order in which their AW transactions are accepted, and the beats of each write burst are contiguous on W.

Example:

~~~text
AW transaction A accepted
AW transaction B accepted

Legal W sequence:
A0 A1 A2 B0 B1

Not legal for AXI4:
A0 B0 A1 B1 A2
~~~

This is a key difference from AXI3, which had WID and supported write-data interleaving.

The interconnect must preserve the required association when merging write traffic from multiple Managers to one Subordinate. It cannot simply arbitrate every W beat independently without tracking which accepted AW stream owns the next data burst.

### 5.5 IDs are not a universal global ordering mechanism

An ID identifies a transaction stream for protocol response association and ordering. It does not automatically provide:
- cache coherency;
- a memory fence;
- software synchronization;
- global ordering across every target;
- completion at a physical DRAM device;
- persistence after power loss.

Those properties require the relevant architecture, memory attributes, coherency protocol, barrier semantics or system-specific completion contract.

### 5.6 ID remapping in an interconnect

Two Managers can independently issue the same ID value. If they share a downstream Subordinate, the interconnect must distinguish the sources.

One common strategy is to prepend or encode a Manager-port tag into the downstream ID:

~~~text
Downstream_ID = {Manager_Port, Upstream_ID}
~~~

This is an implementation example, not a mandated bit layout. The interconnect may use another mapping or internal table. The response path must reverse the mapping and route the response to the correct Manager.

### 5.7 Queue depth and throughput

Suppose a Manager can issue only one 64-byte read burst at a time. If the memory path has a 50 ns round-trip latency, the Manager may spend most of its time waiting for the response.

To keep a link busy, enough work must be in flight to cover the bandwidth-delay product:

\[
OutstandingBytes \ge TargetBandwidth \times RoundTripLatency
\]

At 8 GB/s and 50 ns:

\[
OutstandingBytes \ge 8 \times 10^9 \times 50 \times 10^{-9}
= 400\ bytes
\]

With 64-byte bursts:

\[
OutstandingBursts \ge \lceil 400/64 \rceil = 7
\]

This is a simplified lower-bound estimate. Real systems also account for request issue rate, response timing, arbitration, DRAM bank conflicts, refresh, queue limits and burst efficiency.

**Silicon-debug clue:** if a link has long idle gaps despite low backpressure, inspect the number of outstanding requests, ID reuse restrictions, queue occupancy and response latency. The problem may be insufficient concurrency rather than a slow clock.

---

## 6. Response codes and error propagation

### 6.1 AXI response values

| Encoding | Name | Meaning |
|---|---|---|
| 00 | OKAY | Normal successful response |
| 01 | EXOKAY | Exclusive access success indication |
| 10 | SLVERR | Subordinate reports an access error |
| 11 | DECERR | Decode error, typically no valid target for the address |

The exact use of EXOKAY is associated with exclusive accesses. A system should not treat it as a generic “better OKAY.”

### 6.2 Read and write response behavior

- B channel returns one write response per completed write transaction.
- R channel returns one response with each read data beat.
- RLAST marks the final beat of the read burst.
- WLAST marks the final beat of the write burst.
- A read can return an error status with data; software and the Manager must interpret RRESP.
- A write error is reported on BRESP.

### 6.3 Error path design

A robust interconnect must generate or propagate errors consistently:
- unmapped address → decode error;
- target reports internal failure → subordinate error;
- bridge timeout → implementation-defined error handling mapped into protocol response;
- parity/ECC fault → error or poison behavior according to interface capability and system architecture.

**Do not silently convert DECERR or SLVERR to OKAY.** That hides failures and can turn a bus fault into silent data corruption.

---

## 7. QoS, protection and cache attributes

### 7.1 QoS signals

AXI4 includes AWQOS and ARQOS fields. They allow a Manager to indicate transaction quality-of-service information.

Possible traffic classes:
- real-time audio DMA;
- display scanout;
- camera capture;
- CPU demand reads;
- bulk storage DMA;
- background accelerator traffic.

The QoS value alone does not guarantee a latency. The interconnect's arbitration and resource policy determine how it is used.

Implementation questions:
1. Does the fabric use strict priority, weighted round-robin or another scheme?
2. Can high-priority traffic starve low-priority traffic?
3. Is QoS preserved through every bridge?
4. Are read and write paths scheduled independently?
5. Is the minimum service rate measurable?

### 7.2 Protection attributes

AxPROT typically carries:
- privileged versus unprivileged access;
- secure versus non-secure access;
- instruction versus data access.

A firewall or security controller may use these attributes to allow or reject access. A bridge must not accidentally drop or rewrite security attributes.

### 7.3 Cache and memory attributes

AxCACHE communicates memory attributes such as bufferable, cacheable and allocation behavior. The precise interpretation depends on the AXI generation and system memory-attribute model.

Important distinction:
- **AXI is not itself a cache-coherency protocol.**
- AXI4 memory attributes do not magically snoop CPU caches.
- ACE/ACE-Lite and CHI provide coherent-system mechanisms.
- Non-coherent DMA requires software or hardware cache-maintenance coordination.

### 7.4 Region and user fields

- AxREGION can distinguish regions associated with an interface.
- AxUSER is an implementation-defined sideband field.
- User fields are often used for internal tags, error metadata or platform-specific attributes.

Never assume an AxUSER field has a portable meaning across vendors unless the system integration specification defines it.

---

## 8. Exclusive accesses versus atomic transactions

### 8.1 Exclusive access concept

An exclusive access is a monitor-based mechanism used to implement software synchronization primitives such as compare-and-swap loops or spinlock updates.

Conceptual sequence:

1. Manager performs an exclusive read of a location.
2. The system records an exclusive monitor reservation.
3. Manager computes the update.
4. Manager attempts an exclusive write to the same monitored location.
5. If the reservation is still valid, the write succeeds; otherwise the write fails without performing the update.

The response indicates success or failure according to the exclusive-access rules.

An exclusive access is **not** equivalent to locking the entire bus for the duration of the software sequence. Other transactions can proceed. The monitor tracks whether a conflicting access invalidates the reservation.

### 8.2 Practical cautions

- Read and write address/size attributes must satisfy the exclusive-access requirements.
- The monitor may be implemented at a granularity larger than the exact byte range.
- A failed exclusive write must be retried by software if the algorithm requires it.
- Not every Subordinate or bridge supports exclusive accesses.
- AXI4-Lite does not support exclusive accesses.
- Verify that every path in the system preserves the required exclusive semantics.

### 8.3 AXI5 atomic transactions

AXI5 adds optional atomic transaction capabilities. These are different from a software-managed exclusive read followed by an exclusive write: an atomic transaction carries an operation that the receiving system performs as an atomic update.

Examples of atomic operations include arithmetic or logical updates to a memory location. The exact operations, signals and response behavior depend on the AXI5 specification issue and the interface's declared capability.

**Integration rule:** do not infer AXI5 atomic support from an AXI4-compatible data path. Check the interface properties, bridge support, target support and system memory semantics end to end.

### 8.4 Coherency is a separate question

Even when an atomic operation is supported, the system must define how it interacts with caches and coherent agents. AXI5, ACE5 and CHI have different feature sets. A non-coherent AXI Manager cannot assume its writes automatically invalidate or update a CPU's cached copy.

---

## 9. AXI4-Lite: control-plane interface

AXI4-Lite is designed for simple register-oriented communication.

Characteristics:
- single-beat transactions;
- no burst length greater than one;
- all accesses use the interface data width;
- no exclusive accesses;
- reduced transaction complexity compared with full AXI4.

Typical uses:
- DMA configuration registers;
- interrupt controller registers;
- timer and watchdog control;
- audio codec interface configuration;
- GPIO and pinmux registers;
- accelerator control/status.

### 9.1 AXI4-Lite is still five-channel AXI

It retains independent AW, W, B, AR and R channels with VALID/READY handshakes. A common mistake is to assume AW and W always handshake together because the interface is used for simple registers. A correct AXI4-Lite Subordinate still handles their independent handshakes.

### 9.2 Register programming example

Suppose a DMA block exposes:
- CTRL at 0x00
- SRC_ADDR at 0x04
- DST_ADDR at 0x08
- LENGTH at 0x0C
- STATUS at 0x10

A software driver might:
1. write source address;
2. write destination address;
3. write length;
4. clear stale status;
5. set enable/start;
6. poll or wait for completion interrupt;
7. read status and acknowledge the event.

The exact register semantics are IP-specific, not defined by AXI4-Lite.

---

## 10. AXI4-Stream: data plane without memory addresses

AXI4-Stream is a unidirectional streaming protocol. It removes the memory-mapped address and response channels and focuses on moving data from a source to a sink.

Typical applications:
- audio sample streams;
- camera pixel streams;
- video pipelines;
- DSP chains;
- FPGA accelerator pipelines;
- packet processing.

### 10.1 Main signals

| Signal | Purpose |
|---|---|
| TVALID | Source presents a valid stream beat |
| TREADY | Sink can accept a beat |
| TDATA | Stream payload |
| TKEEP | Byte-valid qualifiers |
| TSTRB | Data-byte versus position-byte qualifiers |
| TLAST | Marks a packet or application-defined boundary |
| TID | Stream identifier |
| TDEST | Routing/destination information |
| TUSER | Application-specific sideband metadata |

Optional signals depend on the interface configuration.

### 10.2 Stream handshake

A beat transfers on a rising edge when TVALID and TREADY are both HIGH. If TVALID is HIGH and TREADY is LOW, the source must hold the offered beat and its associated sideband information stable.

### 10.3 Packet boundaries

TLAST indicates the end of a packet or frame as defined by the application. AXI4-Stream does not define the meaning of a video line, audio frame or Ethernet packet by itself. The endpoint contract defines how TUSER, TLAST, TID and TDEST are interpreted.

For example, a camera pipeline may use:
- TDATA for pixel data;
- TUSER bit for start-of-frame;
- TLAST for end-of-line;
- TKEEP for valid bytes on a partial final beat.

That mapping is an implementation convention, not a universal AXI4-Stream rule.

### 10.4 AXI4 versus AXI4-Stream

| Feature | AXI4 memory-mapped | AXI4-Stream |
|---|---|---|
| Address | Yes | No |
| Read/write response | Yes | No |
| Burst length | AxLEN | Stream continues until application boundary |
| Data direction | Read and write paths | Unidirectional source to sink |
| Typical use | Memory and registers | Audio/video/DSP/packet pipelines |
| Flow control | Per channel | TVALID/TREADY |

---

## 11. Bridges, converters and integration architecture

### 11.1 Common AXI infrastructure blocks

~~~mermaid
flowchart LR
    M[AXI Manager] --> S[Skid / Register Slice]
    S --> I[Interconnect]
    I --> W[Width Converter]
    W --> C[Clock Converter]
    C --> T[AXI Subordinate]
    I --> P[AXI-to-APB Bridge]
    P --> R[APB Register IP]
~~~

### 11.2 Register slice and skid buffer

A register slice breaks a long combinational path and helps timing closure. A skid buffer absorbs a beat when backpressure arrives after the upstream side has already committed to presenting data.

Design checks:
- payload remains stable during stall;
- no beat is duplicated or dropped;
- throughput can reach one beat per cycle when both sides are ready;
- reset clears valid state;
- forward and reverse paths meet timing.

### 11.3 Clock converter

An asynchronous clock converter uses CDC-safe buffering, commonly asynchronous FIFOs or equivalent structures, to transfer requests and responses between unrelated clocks.

Check:
- Gray-coded pointers or approved CDC primitive;
- synchronizer depth and placement;
- reset deassertion in each domain;
- independent clock stoppage;
- FIFO full/empty correctness;
- no combinational crossing of VALID/READY into the other domain.

### 11.4 AXI-to-APB bridge

The bridge accepts AXI transactions, converts them to APB setup/access phases, and returns AXI responses.

It must handle:
- independent AW and W acceptance;
- write data buffering;
- address decode;
- APB wait states via PREADY;
- PSLVERR to AXI response mapping;
- byte strobes and register access width;
- reset and clock-domain boundaries.

A simple APB peripheral may not support bursts. The bridge may need to split a burst into individual APB transfers or reject unsupported access types according to the integration design.

### 11.5 ID and attribute preservation

For every bridge, document which fields are:
- preserved unchanged;
- transformed;
- synthesized;
- dropped because the downstream protocol cannot represent them;
- rejected as unsupported.

This is especially important for ID, AxPROT, AxCACHE, AxQOS, AxREGION, AxUSER, exclusive accesses and AXI5 optional features.

---

## 12. Performance engineering

### 12.1 Peak bandwidth

For a data path that transfers one beat per cycle:

\[
PeakBandwidth = \frac{DataWidth}{8} \times ClockFrequency
\]

Example: 128-bit data bus at 500 MHz:

\[
16\ bytes/cycle \times 500 \times 10^6
= 8\ GB/s
\]

This is the theoretical bandwidth of one direction when every cycle carries a beat. AXI read and write data channels are separate, so simultaneous read and write traffic may use both directions, subject to the fabric and memory controller's capabilities.

### 12.2 Effective bandwidth

\[
EffectiveBandwidth = PeakBandwidth \times Utilization
\]

If utilization is 65%:

\[
8\ GB/s \times 0.65 = 5.2\ GB/s
\]

Utilization is reduced by:
- address and response latency;
- insufficient outstanding requests;
- target backpressure;
- arbitration gaps;
- DRAM row conflicts and refresh;
- burst splitting;
- width conversion;
- clock-domain crossing;
- read/write turnarounds;
- software descriptor gaps.

### 12.3 Outstanding depth calculation

Use the bandwidth-delay product:

\[
RequiredOutstandingBytes \approx Bandwidth \times Latency
\]

At 8 GB/s with 50 ns latency, at least 400 bytes should be in flight to cover the latency. With 64-byte bursts, that is approximately seven outstanding bursts.

If the Manager supports only four outstanding bursts, it may not saturate the link even when the Subordinate could accept more traffic.

### 12.4 QoS and real-time traffic

Suppose:
- audio DMA requires a bounded service interval;
- display DMA fetches scanout data;
- CPU and storage DMA generate best-effort traffic.

A practical architecture may reserve buffering or assign higher arbitration weight to audio/display traffic. But QoS signaling alone does not prove a latency guarantee. Measure worst-case latency under concurrent traffic and refresh/thermal/power conditions.

### 12.5 Useful performance counters

Instrument:
- accepted AW and AR requests;
- W and R beats;
- cycles with VALID and READY;
- cycles stalled by READY low;
- outstanding request histogram;
- response latency histogram;
- queue occupancy high-water mark;
- arbitration wait cycles;
- DECERR/SLVERR counts;
- 4KB split count;
- per-QoS-class bandwidth and latency.

A good silicon debug plan compares counters from Manager, interconnect and Subordinate to identify where the delay is introduced.

---

## 13. RTL architecture and implementation checklist

### 13.1 Write Subordinate state

A robust write path commonly needs:
- AW address/control holding register or FIFO;
- W data/strobe/last holding register or FIFO;
- write association logic;
- local register/SRAM write engine;
- B response holding register.

Do not assume AW and W arrive simultaneously. Do not issue B until the complete write transaction has been accepted and the local operation has reached its defined completion point.

### 13.2 Read Subordinate state

A robust read path commonly needs:
- AR address/control queue;
- burst beat counter;
- address generator;
- memory read pipeline;
- R data/response holding register or FIFO;
- RID and RLAST generation.

RVALID must remain asserted with stable R payload while RREADY is LOW.

### 13.3 Common implementation mistakes

| Bug | Symptom | Likely root cause |
|---|---|---|
| AW and W assumed simultaneous | Random write hangs | Independent channels not handled |
| VALID withdrawn during stall | Lost or corrupted transfer | Source stability violation |
| WLAST one beat early/late | Burst corruption or hang | Incorrect beat counter |
| RLAST incorrect | Manager waits forever or truncates data | Read length tracking error |
| WSTRB lane shift | Byte writes corrupt adjacent registers | Incorrect byte-lane mapping |
| ID mismatch | Response routed to wrong requester | ID extension/remap defect |
| 4KB crossing | Interconnect error or illegal burst | Burst splitter defect |
| FIXED treated as INCR | FIFO data loss | Burst type ignored |
| Response dropped under BREADY/RREADY low | Deadlock or lost completion | Response not buffered |
| QoS discarded | Real-time underrun | Attribute not propagated |
| AxPROT rewritten | Security violation | Bridge/firewall integration error |
| CDC reset mismatch | Rare boot-time deadlock | Reset release or FIFO pointer issue |

### 13.4 RTL review checklist

- Is every channel independently handshaken?
- Are payloads stable under backpressure?
- Are burst counters based on AxLEN + 1?
- Is the correct address sequence generated for FIXED, INCR and WRAP?
- Are illegal burst encodings handled safely?
- Is the 4KB boundary rule enforced by the Manager/bridge?
- Are partial writes and all supported transfer sizes tested?
- Are read and write response IDs correct?
- Are outstanding limits explicit and documented?
- Are same-ID ordering rules preserved?
- Are response FIFOs deep enough for the advertised acceptance rate?
- Are error responses generated for unmapped or unsupported accesses?
- Are reset, CDC and low-power transitions covered?
- Are optional AXI5 features enabled only when all connected components support them?

---

## 14. Verification strategy

### 14.1 Directed tests

1. Single-beat read and write.
2. INCR bursts of 2, 4, 16 and maximum supported length.
3. FIXED burst to a FIFO-like target.
4. WRAP bursts of 2, 4, 8 and 16 beats.
5. All supported AxSIZE values.
6. Every legal byte offset for unaligned transfers.
7. Every WSTRB pattern supported by the IP.
8. Bursts ending exactly at a 4KB boundary.
9. Requests that must be split at a 4KB boundary.
10. AW accepted before W and W accepted before AW where supported by the endpoint.
11. BREADY low for multiple cycles.
12. RREADY low on first, middle and final beats.
13. Multiple outstanding reads with different IDs.
14. Multiple writes with different IDs and ordered W streams.
15. DECERR and SLVERR propagation.
16. Reset while idle and reset during pending traffic, according to the system reset contract.
17. QoS contention with real-time and best-effort traffic.
18. Width conversion with partial strobes.
19. Clock converter with unrelated clock ratios and pauses.
20. AXI-to-APB wait-state and error scenarios.

### 14.2 Assertions

Illustrative SystemVerilog Assertions (adapt signal names and reset polarity to the design):

~~~systemverilog
// Address payload must remain stable while AW is stalled.
property p_aw_stable_when_stalled;
  @(posedge ACLK) disable iff (!ARESETn)
    AWVALID && !AWREADY
    |=> AWVALID && $stable({
      AWADDR, AWID, AWLEN, AWSIZE, AWBURST,
      AWLOCK, AWCACHE, AWPROT, AWQOS
    });
endproperty
assert property (p_aw_stable_when_stalled);

// Write data and byte strobes must remain stable while W is stalled.
property p_w_stable_when_stalled;
  @(posedge ACLK) disable iff (!ARESETn)
    WVALID && !WREADY
    |=> WVALID && $stable({WDATA, WSTRB, WLAST});
endproperty
assert property (p_w_stable_when_stalled);

// Read response payload must remain stable while R is stalled.
property p_r_stable_when_stalled;
  @(posedge ACLK) disable iff (!ARESETn)
    RVALID && !RREADY
    |=> RVALID && $stable({RID, RDATA, RRESP, RLAST});
endproperty
assert property (p_r_stable_when_stalled);
~~~

These are protocol-oriented examples. A complete checker also verifies burst length, legal burst type/size combinations, response association, ordering, 4KB boundaries and reset behavior. If a design supports optional sideband fields, include them in the relevant stability checks.

### 14.3 Scoreboard design

A useful AXI scoreboard should:
- record every accepted address handshake;
- track expected beat count and byte addresses;
- collect W data in accepted AW order;
- apply WSTRB to a byte-addressable reference memory;
- generate expected read data;
- compare RID/BID and response codes;
- allow only protocol-permitted response reordering;
- check RLAST/WLAST positions;
- model errors and exclusive monitor behavior if supported.

Do not build a scoreboard that assumes all responses return in request order. That can hide an ordering bug or falsely flag a legal implementation.

### 14.4 Coverage

Cover:
- burst type × burst length × transfer size;
- start-address alignment × strobe pattern;
- 4KB boundary distance;
- outstanding depth;
- response reordering across IDs;
- same-ID sequences;
- READY stall length;
- response backpressure;
- QoS class contention;
- error response type;
- bridge and clock-ratio combinations.

---

## 15. Post-silicon debug playbook

### Symptom A: DMA transfers only part of a buffer

Check:
1. programmed length and descriptor count;
2. AxLEN and WLAST/RLAST alignment;
3. 4KB burst splitting;
4. WSTRB on the first and final beat;
5. outstanding limit and response completion;
6. memory protection/firewall response;
7. address alignment and cache maintenance.

### Symptom B: Read data is correct in a single test but corrupted under load

Check:
1. RID preservation and ID remapping;
2. same-ID ordering;
3. response FIFO overflow;
4. RREADY backpressure;
5. width-converter beat packing;
6. CDC FIFO pointers;
7. memory-controller return-tag handling.

### Symptom C: Audio or display underrun during CPU traffic

Check:
1. ARQOS/AWQOS values at the Manager;
2. whether the interconnect preserves QoS;
3. arbitration weights and starvation;
4. read queue depth;
5. outstanding transactions;
6. memory-controller scheduling;
7. worst-case latency, not just average bandwidth.

### Symptom D: Bus hangs with VALID high

Capture:
- which channel has VALID high;
- whether the matching READY is low;
- whether payload is stable;
- queue-full indicators;
- target clock and reset state;
- outstanding transaction table;
- response path readiness.

A stable VALID with READY low is not itself a protocol violation. It is a pending transfer. The debug task is to determine why the receiver cannot accept it and whether another dependency prevents forward progress.

### Recommended trace fields

Record at least:
- timestamp/cycle;
- channel;
- VALID and READY;
- address, ID, LEN, SIZE and BURST;
- data and strobes;
- LAST;
- response code;
- QoS and protection attributes;
- queue occupancy;
- target select;
- timeout/error flags.

---

## 16. Worked design example: 128-bit DMA to DDR

### Requirements

- AXI data width: 128 bits.
- Clock: 500 MHz.
- DMA descriptor length: 4096 bytes.
- Memory-side latency: 50 ns.
- Desired utilization: 80%.
- Burst size target: 64 bytes.

### Calculations

Data width in bytes:

\[
128/8 = 16\ bytes
\]

Peak bandwidth:

\[
16 \times 500 \times 10^6 = 8\ GB/s
\]

Target effective bandwidth:

\[
8\ GB/s \times 0.8 = 6.4\ GB/s
\]

Beats per 64-byte burst:

\[
64/16 = 4\ beats
\]

Number of 64-byte bursts in 4096 bytes:

\[
4096/64 = 64\ bursts
\]

Bandwidth-delay product at peak bandwidth:

\[
8\ GB/s \times 50\ ns = 400\ bytes
\]

Outstanding 64-byte bursts to cover this latency:

\[
\lceil 400/64 \rceil = 7
\]

A practical initial design target might therefore allow at least 8 outstanding read bursts, then be tuned using measured DDR latency, arbitration and refresh behavior. This is an engineering starting point, not a protocol requirement.

### Example burst fields

For a 128-bit interface and a 64-byte aligned INCR burst:
- AxSIZE = 4, because 2^4 = 16 bytes per beat.
- AxLEN = 3, because 3 + 1 = 4 beats.
- AxBURST = INCR.
- Start address must be aligned for this example.
- The burst must not cross a 4KB boundary.

~~~text
ARADDR = 0x8000_1000
ARLEN  = 8'h03
ARSIZE = 3'b100
ARBURST= 2'b01
~~~

### Design review questions

- Does the DMA split at every 4KB boundary?
- Does the DDR controller accept eight or more outstanding reads?
- Is RID preserved across the memory fabric?
- Is the return-data FIFO sized for burst backpressure?
- Can another Manager consume the same DDR bandwidth?
- Is QoS sufficient to protect real-time audio/display traffic?
- Is DMA memory coherent, or must software clean/invalidate caches?

---

## 17. Hands-on exercises

### Exercise 1 — Burst decode

A Manager issues:
- ARADDR = 0x4000_2000
- ARLEN = 7
- ARSIZE = 2
- ARBURST = INCR

Find:
1. Number of beats.
2. Bytes per beat.
3. Nominal payload bytes.
4. Address of the last beat.

### Exercise 2 — 4KB split

A DMA engine wants to transfer 256 bytes beginning at 0x0000_0F80 using 16-byte beats.

Find:
1. Whether the entire request fits in one burst.
2. How many bytes remain before the 0x1000 boundary.
3. How the request should be split.

### Exercise 3 — Outstanding depth

A 256-bit interface runs at 400 MHz. Round-trip latency is 80 ns. Each burst is 128 bytes.

Estimate:
1. Peak bandwidth.
2. Bytes needed in flight.
3. Minimum number of outstanding bursts to cover the latency.

### Exercise 4 — WSTRB

A 128-bit interface has 16 byte lanes. A partial write enables lanes 4 through 9.

Find the 16-bit WSTRB value, assuming lane 0 is the least-significant byte lane.

### Exercise 5 — Ordering

A Manager issues two reads with IDs 1 and 7. The response for ID 7 returns first. Is this necessarily illegal? Explain the conditions that matter.

### Exercise 6 — Debug

A DMA write hangs. AWVALID handshakes, all W beats are accepted and WLAST is observed, but BREADY remains HIGH and BVALID never appears.

List at least five likely causes and the signals/counters you would inspect.

### Exercise 7 — Interface selection

Choose AXI4, AXI4-Lite or AXI4-Stream for:
1. CPU access to a DMA configuration register.
2. Camera pixel pipeline between ISP stages.
3. DMA reading a large DDR buffer.
4. DSP output samples sent to a hardware filter.
5. Interrupt controller register access.

---

## 18. Interview questions with concise answers

**Q1. Why are AxLEN and AxSIZE separate?**  
AxLEN specifies the number of beats minus one; AxSIZE specifies bytes per beat. Separating them supports different burst lengths and transfer widths.

**Q2. Why must a burst not cross 4KB?**  
The restriction simplifies interconnect decoding and avoids a burst spanning a 4KB boundary that can represent a different region or translation/protection boundary.

**Q3. Why did AXI4 remove WID?**  
AXI4 removed write-data interleaving, so W data follows AW order and WID is no longer needed. This simplifies the write path.

**Q4. Can AW and W handshake in different cycles?**  
Yes. They are independent channels. A Subordinate must not assume simultaneous acceptance.

**Q5. Can read responses return out of order?**  
Different IDs can permit reordering under the protocol ordering rules. Same-ID and other ordering constraints still apply.

**Q6. Does a larger ID width automatically improve performance?**  
No. It can increase the number of distinguishable in-flight transactions, but buffers, target concurrency, memory scheduling and response paths must also support the traffic.

**Q7. Is AXI4 coherent?**  
No. AXI4 is a memory-mapped interface protocol, not a general cache-coherency protocol. ACE/ACE-Lite and CHI address coherent-system communication.

**Q8. Does AWQOS guarantee a deadline?**  
No. QoS is an input to the implementation's scheduling policy. A hard latency guarantee requires system-level design and measurement.

**Q9. What is the difference between an exclusive access and an atomic transaction?**  
An exclusive sequence uses a monitor to determine whether a later write may succeed. An atomic transaction carries an operation that is performed as an atomic update by a capable system.

**Q10. Why can a 128-bit AXI bus still deliver poor bandwidth?**  
The link may be underutilized because of insufficient outstanding transactions, backpressure, arbitration, short bursts, DRAM timing, conversion overhead or software gaps.

---

## 19. Exercise answer key

### Exercise 1

- Beats = 7 + 1 = 8.
- Bytes per beat = 2^2 = 4.
- Nominal payload = 8 × 4 = 32 bytes.
- Aligned INCR last beat address = 0x4000_201C.

### Exercise 2

- Bytes before the 4KB boundary = 0x1000 − 0x0F80 = 128 bytes.
- The 256-byte request crosses the boundary.
- Split into a 128-byte burst ending at 0x0FFF and a 128-byte burst beginning at 0x1000.
- With 16-byte beats, each part contains eight beats.

### Exercise 3

- Data width = 32 bytes.
- Peak bandwidth = 32 × 400 MHz = 12.8 GB/s.
- Bytes in flight = 12.8 GB/s × 80 ns = 1024 bytes.
- Outstanding bursts = 1024 / 128 = 8 bursts.

### Exercise 4

Bits 4 through 9 are enabled:

~~~text
WSTRB[15:0] = 16'b0000_0011_1111_0000
~~~

### Exercise 5

Not necessarily. Different IDs can permit responses to return in a different order. Confirm the ordering rules for the particular transactions, target and interconnect path. Do not assume that all different-ID responses are unconstrained.

### Exercise 6

Possible causes:
- Subordinate never generates BVALID after local write completion.
- Write engine waits for an internal condition that never occurs.
- WLAST counter mismatch leaves the burst marked incomplete.
- AW metadata was lost or not associated with the W burst.
- Response FIFO is full or incorrectly reports full.
- Reset/clock-domain issue blocks the response path.
- Error path is stuck before response generation.
- BVALID is generated but blocked by a mux or response-routing bug.

Inspect AW/W handshake counters, WLAST, internal write-done, B FIFO occupancy, BVALID before/after interconnect, reset, clocks and response ID.

### Exercise 7

1. AXI4-Lite.
2. AXI4-Stream.
3. AXI4.
4. AXI4-Stream.
5. AXI4-Lite.

---

## 20. Integration checklist for a real SoC

Before signing off an AXI path, capture the following in the interface-control document:

| Category | Required decision |
|---|---|
| Protocol | AXI3, AXI4, AXI4-Lite, AXI4-Stream or AXI5 |
| Width | Address, data, ID and strobe widths |
| Burst | Supported burst types and maximum lengths |
| Alignment | Supported transfer sizes and unaligned behavior |
| Outstanding | Maximum read/write requests |
| Ordering | Same-ID and cross-ID guarantees |
| Responses | Error generation and propagation |
| Attributes | CACHE, PROT, QOS, REGION, USER |
| Atomics | Exclusive/atomic capability and path support |
| Clocks | Frequency, clock domain and CDC boundary |
| Reset | Assertion/deassertion and in-flight transaction policy |
| Power | Interface quiescence, wakeup and isolation |
| Security | Firewall policy and attribute preservation |
| Performance | Bandwidth, latency, QoS and queue depth |
| Verification | Protocol checker, scoreboard, formal and stress plan |
| Silicon | Trace points, counters, timeout and error visibility |

A protocol-compliant endpoint can still be a poor system component if the integration contract is incomplete. The address map, cacheability, DMA coherency, security attributes, reset sequencing and performance budget must be designed together.

---

## 21. Reference material

Prefer the current official specification issue for normative implementation decisions. The links below are reading entry points; use the exact issue and optional feature set selected by the project.

1. **Arm — AMBA AXI Protocol Specification, IHI 0022.** Current specification landing page and issue selection: https://developer.arm.com/documentation/ihi0022/latest
2. **Arm — Introduction to AMBA AXI4**, architecture learning guide: https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf
3. **Arm — AMBA 4 overview**, AXI4, AXI4-Lite, AXI4-Stream and ACE context: https://www.arm.com/architecture/system-architectures/amba/amba-4
4. **Arm — AMBA 5 overview**, AXI5, ACE5, CHI and optional capabilities: https://www.arm.com/architecture/system-architectures/amba/amba-5
5. **Arm — Introducing the next generation of AXI and ACE protocols**, 29 January 2018: https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/introducing-the-next-generation-of-axi-and-ace-protocols
6. **Arm — Updates to AMBA AXI and CHI specifications**, 15 October 2019: https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/updates-to-amba-axi-and-chi-specifications
7. **AMD — AXI interface overview**, implementation-oriented summary of AXI4, AXI4-Lite and AXI4-Stream: https://www.amd.com/en/products/adaptive-socs-and-fpgas/intellectual-property/axi.html
8. **AMD — AXI4-Stream Infrastructure guide PG085**, implementation examples and stream infrastructure: https://docs.amd.com/r/en-US/pg085-axi4stream-infrastructure/Overview
9. **Arm — CoreLink NI-700 technical reference manual**, an example of a real interconnect that supports selected AXI5 capabilities: https://documentation-service.arm.com/static/64c2dc0438511951cb7a1956

### Reading discipline

- The Arm IHI 0022 specification is normative for the selected AXI issue and interface properties.
- Arm announcements explain the public history and intent of selected feature additions.
- AMD and Arm IP technical reference manuals describe particular implementations, not universal AXI requirements.
- This lesson uses original explanatory text and examples; it does not reproduce proprietary specification tables or figures.

---

## 22. Final engineering takeaways

1. AxLEN counts beats minus one; AxSIZE defines bytes per beat.
2. Burst type controls address progression; FIXED is important for FIFO-style targets.
3. No AXI burst may cross a 4KB boundary.
4. WSTRB is essential for byte-accurate writes, especially with width conversion.
5. AXI4 write data is not interleaved; W follows AW transaction order.
6. IDs permit response association and protocol-controlled reordering, not automatic coherency.
7. Outstanding depth must cover the bandwidth-delay product if the system is to sustain high throughput.
8. QoS signaling is only useful when the fabric and targets implement a measurable scheduling policy.
9. Exclusive accesses and AXI5 atomic transactions are distinct mechanisms.
10. AXI4-Lite is for control; AXI4-Stream is for streaming data; AXI4 is for memory-mapped high-throughput traffic.
11. A bridge must preserve or explicitly transform every relevant attribute.
12. Protocol compliance, integration correctness and system performance are three separate sign-off activities.

**Next session:** Lesson 07 — AXI Interconnect Implementation: Crossbar Microarchitecture, Arbitration, Routing, Buffering, Deadlock, Timing Closure and Performance Instrumentation.
