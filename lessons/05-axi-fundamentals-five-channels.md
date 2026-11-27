# Lesson 05 — AMBA AXI Fundamentals: The Five Independent Channels

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 05 of 38  
**Suggested duration:** 60 minutes  
**Level:** Protocol fundamentals through first-level RTL and silicon debug  
**Primary focus:** Memory-mapped AXI4, with AXI4-Lite and AXI4-Stream contrasted at the end

> **Learning objective:** Be able to look at an AXI waveform and explain which transactions have been accepted, which are still outstanding, why a channel is stalled, and whether the Manager or Subordinate is responsible for making progress.

---

## 1. One-hour lesson plan

| Time | Topic | Engineering outcome |
|---:|---|---|
| 0–7 min | Why AXI exists; AMBA history | Understand the performance problem and protocol evolution |
| 7–17 min | Five channels and VALID/READY | Read channel-level handshakes correctly |
| 17–29 min | Write path | Explain independent AW/W acceptance and B response |
| 29–39 min | Read path | Explain AR acceptance, R beats, IDs and RLAST |
| 39–47 min | Bursts, byte lanes and performance | Calculate transfer size, bandwidth and in-flight work |
| 47–55 min | RTL architecture and verification | Design a safe first AXI Subordinate |
| 55–60 min | Silicon debug and exercises | Turn a waveform symptom into a debug hypothesis |

### Prerequisites

- Lesson 02: interconnect fundamentals.
- Lesson 03: APB setup/access and peripheral register interfaces.
- Lesson 04: AHB address/data pipelining and wait states.

### Outcomes

By the end, you should be able to:

1. Identify all five AXI memory-mapped channels and their directions.
2. Determine exactly when a beat or address is transferred.
3. Explain why AW and W can handshake in different cycles.
4. Explain how B and R responses complete transactions.
5. Decode a basic burst using AxLEN, AxSIZE and AxBURST.
6. Distinguish protocol capability from what a particular IP implementation supports.
7. Build a verification plan that stresses independent backpressure.

---

## 2. Historical backstory: why AXI was needed

### 2.1 The system problem

As SoCs grew beyond a processor plus a few peripherals, many independent agents needed to reach memory and other targets:

- CPU instruction and data paths
- DMA engines
- display and camera pipelines
- audio engines
- GPU and DSP accelerators
- USB, PCIe and Ethernet controllers
- SRAM and DRAM controllers

A simple shared bus is attractive for area and simplicity, but it serializes access and can become a bottleneck. A high-performance fabric needs to keep multiple transactions in flight, separate unrelated traffic, support bursts, and avoid forcing every IP block to implement a bespoke interface.

AXI is an **IP interface protocol**. It defines how two connected interface endpoints exchange requests, data and responses. It does not, by itself, define the complete topology or microarchitecture of the interconnect. A crossbar, network interconnect or bridge is a separate system component. Arm explicitly describes AXI as a point-to-point interface specification for IP blocks rather than the interconnect itself.

### 2.2 Documented evolution

The following timeline is supported by Arm's published historical material:

| Date / generation | Milestone | Engineering context |
|---|---|---|
| 1995–1996 | AMBA 1: ASB and APB | Establish reusable on-chip interfaces for controllers and peripherals |
| 1999 | AMBA 2: AHB | Higher-performance pipelined bus for processor/memory traffic |
| 2003 | AMBA 3: AXI / AXI3 | Higher clock-rate, high-performance interface with independent channels and outstanding transactions |
| 2010 | AMBA 4: AXI4, AXI4-Lite, AXI4-Stream | Longer INCR bursts; simpler memory-mapped control option; address-free streaming |
| 2011 | ACE and ACE-Lite | Add system-level cache coherency capabilities to the AMBA ecosystem |
| 2013 | AMBA 5 CHI announced | New coherent transport architecture for larger, higher-performance systems |
| 2018 | AXI5 / ACE5 announced | Additional capabilities aligned with AMBA 5 and CHI, including selected atomic, RAS and QoS-related enhancements |
| 2019 | AXI/ACE Issue G announced | Clarifies ordering and adds features such as unique-ID indication, read data chunking and interface parity protection |

**Why the architecture changed:** the move from AHB-style shared/pipelined bus transactions to AXI's five independent channels permits address, data and response activity to be decoupled. That creates opportunities for concurrency and buffering, but also moves more responsibility into the endpoints and interconnect: channel alignment, transaction tracking, ordering, arbitration and backpressure must all be handled correctly.

**Documented facts versus interpretation:** the dates and named features above are documented by Arm. The explanation that independent channels enable concurrency while increasing implementation complexity is an engineering interpretation of those architectural choices, not a claim about any unpublished internal design meeting.

### 2.3 Why five channels instead of one bus handshake?

A memory-mapped AXI interface splits a read/write transaction into logically independent paths:

- address/control for writes
- write payload
- write completion
- address/control for reads
- read payload and completion

This allows a Subordinate to accept a write address while it is still receiving data for an earlier write, or to return read data while another request is being accepted. The exact amount of concurrency depends on the implementation's buffers, outstanding limits and interconnect resources.

**Trade-off:** more concurrency and independent flow control require more state, queues, ID tracking and verification. AXI is not automatically faster merely because it has five channels; a poorly buffered Subordinate can still accept only one request at a time.

---

## 3. AXI memory-mapped architecture

### 3.1 Manager and Subordinate terminology

The modern terms are **Manager** and **Subordinate**:

- A Manager initiates a transaction, for example a CPU or DMA engine.
- A Subordinate responds to transactions, for example SRAM, a register block or a memory controller.
- An Interconnect routes requests and responses between multiple Managers and Subordinates.

Older documents and many RTL signal names still use master/slave. The signal prefixes commonly remain **M_AXI** and **S_AXI**; do not confuse the historical signal naming with the protocol roles.

### 3.2 System-level view

~~~mermaid
flowchart LR
    CPU[CPU Manager]
    DMA[DMA Manager]
    GPU[GPU Manager]
    FAB[AXI Interconnect<br/>decode / arbitration / routing]
    SRAM[AXI SRAM Subordinate]
    DDR[DDR Controller Subordinate]
    REG[AXI-Lite Register Subordinate]

    CPU --> FAB
    DMA --> FAB
    GPU --> FAB
    FAB --> SRAM
    FAB --> DDR
    FAB --> REG
~~~

The fabric may contain address decoders, arbiters, request/response queues, width converters, clock converters, security filters and register slices. Those are implementation structures, not five additional AXI channels.

### 3.3 The five channels

| Channel | Direction | Main purpose | Typical payload |
|---|---|---|---|
| AW — Write Address | Manager → Subordinate | Describe a write transaction | AWADDR, AWLEN, AWSIZE, AWBURST, AWID |
| W — Write Data | Manager → Subordinate | Carry write beats | WDATA, WSTRB, WLAST |
| B — Write Response | Subordinate → Manager | Complete a write transaction | BID, BRESP |
| AR — Read Address | Manager → Subordinate | Describe a read transaction | ARADDR, ARLEN, ARSIZE, ARBURST, ARID |
| R — Read Data | Subordinate → Manager | Return read beats and final status | RID, RDATA, RRESP, RLAST |

Each channel has its own VALID/READY handshake. There is no single global AXI READY signal.

### 3.4 Conceptual interface

~~~mermaid
flowchart TB
    subgraph M[Manager]
      AW[AW source]
      W[W source]
      B[B sink]
      AR[AR source]
      R[R sink]
    end
    subgraph S[Subordinate]
      AWR[AW sink]
      WR[W sink]
      BR[B source]
      ARR[AR sink]
      RR[R source]
    end
    AW -->|AWVALID / AWREADY| AWR
    W -->|WVALID / WREADY| WR
    BR -->|BVALID / BREADY| B
    AR -->|ARVALID / ARREADY| ARR
    RR -->|RVALID / RREADY| R
~~~

The diagram shows logical directions. Physical implementations may insert skid buffers or register slices between the endpoints.

---

## 4. The fundamental rule: VALID and READY

### 4.1 Transfer definition

For any AXI channel, a transfer occurs on a rising edge of ACLK when both signals are HIGH:

\[
\text{handshake} = VALID \land READY
\]

If VALID=1 and READY=0, the source is presenting a valid item but the destination has not accepted it. The item is **pending**, not transferred.

If VALID=0, no item is being offered, even if READY is HIGH.

### 4.2 Source and destination responsibilities

**Source:**

1. Assert VALID when a payload item is available.
2. Keep VALID asserted until a rising edge where READY is also HIGH.
3. Keep the associated payload stable while VALID is HIGH and READY is LOW.
4. Do not withdraw or change the offered item just because the destination is taking several cycles.

**Destination:**

1. Assert READY when it can accept a transfer.
2. May assert READY before VALID arrives.
3. May deassert READY to apply backpressure.
4. Must ensure its internal capacity and state are consistent with the READY it advertises.

The source must not make progress depend on READY being asserted first. Otherwise two endpoints can deadlock if each waits for the other. A destination is permitted to wait for VALID before asserting READY, but a design should be reviewed for deadlock and throughput consequences.

### 4.3 Timing example

| Cycle | VALID | READY | Payload | Meaning |
|---:|---:|---:|---|---|
| C0 | 0 | 1 | — | No transfer offered |
| C1 | 1 | 0 | A | A is pending; not accepted |
| C2 | 1 | 0 | A | A must remain stable |
| C3 | 1 | 1 | A | A transfers on this edge |
| C4 | 1 | 1 | B | B transfers on this edge |

A common RTL bug is changing the payload from A to B in C2 while VALID remains HIGH. That violates the channel's stability requirement.

~~~text
ACLK       _/‾\_/‾\_/‾\_/‾\_/‾\_
VALID      ____/‾‾‾‾‾‾‾‾‾‾‾‾‾‾
READY      ‾‾‾‾________/‾‾‾‾‾‾‾
PAYLOAD    ---- A ------ A -- B
HANDSHAKE  _____________^_____^
                         A     B
~~~

The exact edge convention is important: evaluate VALID and READY at the active clock edge. A waveform viewer may display a signal changing just after the edge due to simulation scheduling; use sampled values when debugging.

---

## 5. Write transaction: AW, W and B

### 5.1 Write path sequence

A write transaction has three channel-level parts:

1. **AW:** Manager provides address and transaction attributes.
2. **W:** Manager provides one or more data beats.
3. **B:** Subordinate returns one write response after the write has been accepted and processed according to the interface's completion rules.

AW and W are independent. The address and first data beat do not have to handshake in the same cycle. A Subordinate must not assume that they arrive together.

~~~mermaid
sequenceDiagram
    participant M as Manager
    participant S as Subordinate
    M->>S: AWVALID + AW payload
    Note over M,S: AW handshake when AWREADY is HIGH
    M->>S: WVALID + WDATA + WSTRB + WLAST
    Note over M,S: W handshake for each beat
    S->>M: BVALID + BRESP + BID
    Note over M,S: B handshake when BREADY is HIGH
~~~

This is a conceptual sequence, not a requirement that AW must precede W in time. Either channel can be accepted first; multiple W beats may be transferred before or after the AW handshake, subject to the protocol and the Subordinate's buffering/acceptance behavior.

### 5.2 Write address channel signals

| Signal | Meaning |
|---|---|
| AWVALID | Manager presents a write address/control item |
| AWREADY | Subordinate accepts the write address/control item |
| AWADDR | Byte address |
| AWID | Transaction ID |
| AWLEN | Number of data beats minus one |
| AWSIZE | Bytes per beat encoded as log2(bytes) |
| AWBURST | FIXED, INCR or WRAP |
| AWLOCK | Exclusive-access attribute in applicable AXI revisions |
| AWCACHE | Memory/cache attribute information |
| AWPROT | Protection attributes |
| AWQOS | QoS indication |
| AWREGION | Region identifier where implemented |
| AWUSER | Optional implementation-defined user sideband |

Optional signals and widths depend on the selected AXI revision and interface configuration. Do not assume every port exposes every optional field.

### 5.3 Write data channel signals

| Signal | Meaning |
|---|---|
| WVALID | Manager presents a write data beat |
| WREADY | Subordinate accepts the beat |
| WDATA | Write data payload |
| WSTRB | Byte-lane write enables |
| WLAST | Marks the final beat of the write burst |
| WUSER | Optional user sideband |

For a 64-bit data bus, there are eight byte lanes, so WSTRB is typically eight bits wide. A strobe bit of 1 indicates that the corresponding byte lane is valid for the write.

Example:

- WDATA = **0x1122334455667788**
- WSTRB = **8'b0000_1111**

Only the four enabled byte lanes are written. Which byte address maps to each WSTRB bit depends on the interface's byte-lane convention and the transfer address; verify the implementation and endian mapping rather than guessing from a hex dump.

### 5.4 Write response channel

| Signal | Meaning |
|---|---|
| BVALID | Subordinate presents a write response |
| BREADY | Manager accepts the response |
| BRESP | Write response status |
| BID | ID associated with the completed write |
| BUSER | Optional user sideband |

Common response encodings:

| BRESP | Meaning |
|---|---|
| OKAY | Normal access success |
| EXOKAY | Exclusive access succeeded |
| SLVERR | Subordinate reports an access error |
| DECERR | Decode error, typically generated when no target matches |

The exact response semantics and exclusive-access use are specified by the applicable AXI revision. For a basic non-exclusive transfer, OKAY is the normal successful response.

**Important:** a write response is not a response to AW alone or to an individual W beat. It completes the write transaction. A Subordinate must not issue BVALID before the required write address and data have been accepted and the write completion condition has been met.

### 5.5 Why AXI4 removed WID

AXI3 included a write-data ID (WID) and allowed write data interleaving between write transactions. AXI4 removed WID and write-data interleaving. This simplified write-data association and interconnect logic, at the cost of constraining how write data from different write transactions can be interleaved on the W channel.

Do not apply AXI3 assumptions to AXI4. When integrating older IP, check whether an AXI3-to-AXI4 conversion is required and how the bridge serializes or tracks write data.

---

## 6. Read transaction: AR and R

### 6.1 Read path sequence

A read transaction has:

1. **AR:** Manager presents the read address and attributes.
2. **R:** Subordinate returns one or more read data beats.
3. **RLAST:** Marks the final beat of that read burst.

~~~mermaid
sequenceDiagram
    participant M as Manager
    participant S as Subordinate
    M->>S: ARVALID + AR payload
    Note over M,S: AR handshake when ARREADY is HIGH
    loop Each read beat
      S->>M: RVALID + RID + RDATA + RRESP
      M->>S: RREADY
      Note over M,S: Beat transfers when RVALID and RREADY are HIGH
    end
    Note over M,S: RLAST is HIGH on the final beat
~~~

The Subordinate may take multiple cycles between AR acceptance and the first R beat. It may also stall between R beats. The Manager can apply backpressure by deasserting RREADY.

### 6.2 Read address channel

| Signal | Meaning |
|---|---|
| ARVALID / ARREADY | Read address handshake |
| ARADDR | Start byte address |
| ARID | Read transaction ID |
| ARLEN | Number of beats minus one |
| ARSIZE | Bytes per beat, encoded as log2(bytes) |
| ARBURST | FIXED, INCR or WRAP |
| ARLOCK | Exclusive-access attribute where applicable |
| ARCACHE | Memory/cache attributes |
| ARPROT | Protection attributes |
| ARQOS | QoS indication |
| ARREGION | Region identifier where implemented |
| ARUSER | Optional user sideband |

### 6.3 Read data channel

| Signal | Meaning |
|---|---|
| RVALID / RREADY | Read beat handshake |
| RID | ID of the read transaction |
| RDATA | Read data |
| RRESP | Response status for the beat |
| RLAST | Final beat marker |
| RUSER | Optional user sideband |

A read burst returns multiple R beats. RLAST must identify the final beat correctly. A Manager must accept or otherwise correctly handle the complete burst; simply ignoring a beat does not cancel the rest of the transaction.

### 6.4 Response handling

RRESP is associated with each read data beat. A Subordinate can report an error on a beat using SLVERR or DECERR as appropriate. The Manager must not assume every beat in a burst has the same response unless the system's implementation or higher-level contract guarantees that.

---

## 7. Burst fundamentals

### 7.1 Three burst types

| AxBURST | Name | Address behavior | Typical use |
|---|---|---|---|
| 00 | FIXED | Address remains fixed | FIFO/data register access |
| 01 | INCR | Address advances by transfer size | Normal memory reads/writes |
| 10 | WRAP | Address wraps within an aligned region | Cache-line-oriented accesses |
| 11 | Reserved | Not a normal burst type | Do not generate |

For incrementing bursts, the address advances by the transfer size. For unaligned first addresses, the first transfer can have a smaller aligned byte-lane window; later beats follow the protocol's address calculation rules. Use a protocol-aware reference model rather than assuming next address = current address + bus bytes for every possible transfer.

### 7.2 AxLEN and AxSIZE

- **AxLEN** = number of beats minus one.
- **AxSIZE** = log2 of the number of bytes per beat.

Examples:

| AxLEN | Beats | AxSIZE | Bytes per beat | Total bytes |
|---:|---:|---:|---:|---:|
| 0 | 1 | 2 | 4 | 4 |
| 3 | 4 | 3 | 8 | 32 |
| 15 | 16 | 4 | 16 | 256 |
| 255 | 256 | 4 | 16 | 4096 |

For AXI4, INCR bursts can contain up to 256 transfers. FIXED bursts are limited to 16 transfers. WRAP bursts have legal lengths of 2, 4, 8 or 16 transfers. AXI4 bursts must not cross a 4KB address boundary. Early termination of a burst is not supported.

These rules are protocol-level requirements; an individual IP may support a smaller maximum burst or narrower data width.

### 7.3 4KB boundary example

Suppose:

- Start address = **0x0000_0FF0**
- Data width = 128 bits = 16 bytes per beat
- INCR burst length = 4 beats

Total transfer size = 4 × 16 = 64 bytes. The address range crosses the 4KB boundary at **0x1000**, so this burst is illegal as a single AXI4 burst.

A DMA engine or upstream adapter must split it into legal bursts, for example:

- Burst 1: 1 beat at **0x0FF0**
- Burst 2: 3 beats beginning at **0x1000**

This is a classic source of failures in DMA engines, width converters and bridges.

---

## 8. IDs, outstanding transactions and ordering — first principles

### 8.1 Why IDs exist

Without IDs, a Manager would have difficulty associating returned responses with multiple requests that are in flight at the same time. AXI IDs allow a response to identify the transaction to which it belongs.

- AWID is returned as BID.
- ARID is returned as RID.
- Different IDs can enable more concurrency and response reordering where the ordering rules permit it.
- Same-ID ordering constraints still matter.
- IDs are not physical addresses and do not replace address decoding.

The interconnect may need to track which Manager originated a transaction. It may extend or transform IDs internally, then restore the expected ID on the return path. The precise implementation is fabric-specific.

### 8.2 Outstanding transaction

An outstanding transaction is one that has been accepted into the system but has not yet completed with its response.

A design's effective outstanding capacity is limited by several resources:

- Manager request queues
- Interconnect acceptance limits
- Per-target queues
- Subordinate buffers
- ID space and ID allocation policy
- Return-path buffering
- Memory-controller scheduling resources

Increasing outstanding depth can improve throughput on a high-latency path, but it also consumes area and can increase contention or worst-case latency for other traffic.

### 8.3 Ordering is not a single global FIFO

Do not assume that every AXI response returns in the order in which all requests were issued across the entire SoC. Ordering is defined by the protocol's ID and transaction ordering rules and can also be constrained by the interconnect, target and system memory attributes.

**For this introductory lesson:** remember that IDs associate requests and responses, and that same-ID ordering matters. The full ordering model, barriers, exclusives and advanced memory attributes are reserved for Session 06.

Arm's 2019 Issue G announcement is especially useful historically: it explains that the original ID-based ordering model could require substantial interconnect tracking, and describes the Unique ID indicator and read data chunking additions in that issue.

---

## 9. Performance engineering

### 9.1 Peak bandwidth

For a single data channel operating at one accepted beat per clock:

\[
BW = \text{bytes per beat} \times f_{ACLK}
\]

Example:

- Data width = 128 bits = 16 bytes
- ACLK = 500 MHz
- One accepted beat every cycle

\[
BW = 16 \times 500 \times 10^6
= 8 \times 10^9 \text{ bytes/s}
= 8 \text{ GB/s}
\]

This is a theoretical peak for that direction and path. Real throughput is reduced by stalls, arbitration, target latency, refresh, turnaround, burst gaps, width conversion and other system effects. Read and write channels are logically independent, but they may share physical resources downstream.

### 9.2 Efficiency

If only 70% of clock cycles transfer a beat:

\[
BW_{effective} = 8 \text{ GB/s} \times 0.70 = 5.6 \text{ GB/s}
\]

Measure accepted beats, not merely VALID cycles. Count VALID && READY handshakes.

### 9.3 Outstanding depth and latency hiding

Little's Law gives a useful first-order estimate:

\[
N = \lambda \times L
\]

where N is the average number of in-flight units, λ is throughput in units per second, and L is average latency in seconds.

Suppose the path needs 8 GB/s and the round-trip latency is 100 ns:

\[
8 \times 10^9 \times 100 \times 10^{-9} = 800 \text{ bytes}
\]

With 16-byte beats, about 50 beats must be in flight to sustain that idealized rate. If the implementation uses 16-beat bursts (256 bytes), at least four such bursts in flight represent 1024 bytes of outstanding payload capacity.

This is a capacity estimate, not a guarantee: the Manager, interconnect and target must all accept enough work, and the DRAM scheduler must sustain the required service rate.

### 9.4 What to measure

For each channel, measure:

- VALID cycles
- READY cycles
- Handshake count
- Stall cycles (VALID && !READY)
- Idle cycles (!VALID)
- Burst length distribution
- Average and maximum response latency
- Outstanding transaction high-water mark
- Bytes transferred per unit time
- Per-ID latency and ordering events

Do not use one aggregate bandwidth number to diagnose every bottleneck. Separate AW, W, B, AR and R behavior.

---

## 10. AXI4, AXI4-Lite and AXI4-Stream

| Feature | AXI4 memory-mapped | AXI4-Lite | AXI4-Stream |
|---|---|---|---|
| Address | Yes | Yes | No |
| Write/read split | Yes | Yes | Not in the memory-mapped sense |
| Burst | Yes | No; single-beat accesses | Stream beats, packet boundaries via sideband |
| IDs | Supported | Not present | Optional stream ID |
| Typical use | DDR, SRAM, DMA data path | Control/status registers | Audio/video/packet datapath |
| Main payload | Address + data + response | Address + data + response | TDATA and stream sidebands |

### AXI4-Lite

AXI4-Lite is intended for simpler memory-mapped control interfaces. It retains the five channel concept but supports single-beat transactions and removes many advanced features such as bursts and transaction IDs. It is commonly used for control/status register banks.

A common SoC pattern is:
- CPU uses AXI4-Lite to program DMA registers.
- DMA uses full AXI4 to move payload data.
- DMA may expose AXI4-Stream for a streaming datapath.

### AXI4-Stream

AXI4-Stream transfers data without a memory address. Its core handshake is TVALID/TREADY. TDATA carries payload; optional signals such as TKEEP, TSTRB, TLAST, TID, TDEST and TUSER provide byte qualification, packet boundary, stream routing and application metadata.

For a video pipeline, TLAST may represent an end-of-line or packet convention depending on the system contract; do not assume its semantic meaning without reading the IP integration guide. AXI4-Stream is covered more deeply in Session 06.

---

## 11. RTL implementation: how to build a first Subordinate

### 11.1 Recommended microarchitecture

A robust memory-mapped Subordinate should separate channel acceptance from the internal operation:

~~~mermaid
flowchart TD
    AW[AW channel] --> AWQ[AW skid buffer / queue]
    W[W channel] --> WQ[W skid buffer / beat queue]
    AR[AR channel] --> ARQ[AR queue]
    AWQ --> JOIN[Write transaction assembler]
    WQ --> JOIN
    JOIN --> WR[Register / SRAM write engine]
    WR --> BQ[B response queue]
    BQ --> B[B channel]
    ARQ --> RR[Read engine]
    RR --> RQ[Read response queue]
    RQ --> R[R channel]
~~~

The architecture can be simplified for a low-throughput register bank, but the protocol obligations remain.

### 11.2 Write path state

At minimum, track:

- Whether an AW item has been accepted.
- The captured address, ID and write attributes.
- Whether W beats have arrived.
- The expected number of W beats.
- The current beat index.
- WLAST consistency.
- Whether the internal write has completed.
- Whether a B response is pending and its ID/status.

**Never assume AWVALID and WVALID arrive together.** The Manager is allowed to present them independently. A one-outstanding Subordinate may store AW and W information separately, then execute the write when enough information is available.

### 11.3 Read path state

Track:

- Accepted AR address and attributes.
- Read ID.
- Expected beat count.
- Current beat index.
- Read data source and latency.
- R response status.
- RLAST generation.
- Pending R response when RREADY is LOW.

The read engine must hold RVALID and the associated R payload stable while the Manager applies backpressure.

### 11.4 First RTL design choices

Before coding, define:

1. AXI revision and supported subset.
2. Address width and data width.
3. ID width and number of outstanding transactions.
4. Burst types and maximum burst length.
5. Whether unaligned accesses are supported.
6. Whether the target is memory, FIFO or register space.
7. Error response behavior.
8. Reset behavior.
9. Clock-domain relationship.
10. Whether optional protection, cache, QoS and user fields are consumed or ignored.

A design that supports only one outstanding read and one outstanding write can be perfectly valid if its advertised interface behavior is correct. Do not claim full protocol capability if the implementation silently rejects legal traffic.

### 11.5 Register bank versus memory target

A register bank may:
- accept only single-beat accesses,
- decode a small address map,
- use WSTRB for byte writes,
- return a response after a fixed or variable number of cycles.

A memory target may:
- accept bursts,
- pipeline multiple requests,
- return read data over several cycles,
- require queues and ID tracking.

A FIFO target is special: FIXED bursts can be meaningful because repeated transfers access the same data register address. Accidentally converting a FIFO transfer to INCR can break the device semantics.

---

## 12. Verification strategy

### 12.1 Directed tests

1. Single-beat read and write.
2. Back-to-back reads.
3. Back-to-back writes.
4. AW accepted before W.
5. W accepted before AW.
6. AW and W accepted on the same edge.
7. Multi-beat INCR burst.
8. FIXED burst to a FIFO-like target.
9. Legal WRAP burst.
10. WSTRB patterns: all bytes, one byte, alternating bytes, no bytes.
11. Read and write response backpressure.
12. SLVERR and DECERR paths.
13. Different IDs with permitted reordering.
14. 4KB boundary legal and illegal cases.
15. Reset during idle and during traffic, according to system reset policy.

### 12.2 Randomized stress

Randomize each channel independently:

- AWVALID timing
- WVALID timing and gaps
- BREADY stalls
- ARVALID timing
- RREADY stalls
- subordinate internal latency
- burst size and type
- IDs
- address regions
- byte strobes

Avoid a testbench that always asserts AWVALID and WVALID together or keeps BREADY and RREADY permanently HIGH. Such a testbench can miss important protocol bugs.

### 12.3 Assertions and protocol checks

Useful properties include:

- VALID remains asserted until handshake.
- Payload remains stable while VALID is HIGH and READY is LOW.
- No response is generated for a transaction that was never accepted.
- WLAST occurs on the expected final beat.
- RLAST occurs on the expected final beat.
- BID/RID match the accepted transaction.
- No response queue overflow or underflow.
- No illegal burst crosses a 4KB boundary.
- Reset drives protocol outputs to the specified inactive state.

Example conceptual SVA for payload stability:

~~~systemverilog
property p_aw_stable_while_stalled;
  @(posedge ACLK) disable iff (!ARESETn)
    AWVALID && !AWREADY |=> AWVALID &&
      $stable({AWADDR, AWID, AWLEN, AWSIZE, AWBURST});
endproperty

assert property (p_aw_stable_while_stalled);
~~~

This is an illustrative assertion pattern. Adapt the payload list to the actual interface, optional fields and reset convention. The same principle applies independently to W, B, AR and R.

### 12.4 Scoreboard model

The scoreboard should:
- record accepted AW and AR transactions,
- associate accepted W beats with write transactions in the required order,
- update a reference memory using WSTRB,
- compare accepted R beats against expected data,
- validate response IDs and response status,
- check final-beat markers,
- account for legal response reordering across IDs.

A scoreboard that only checks data values but ignores IDs, response order and WLAST/RLAST can miss serious integration failures.

---

## 13. Common RTL and integration bugs

| Symptom | Likely cause | Debug direction |
|---|---|---|
| Write hangs forever | AW or W never accepted; B response not generated | Count AW and W handshakes independently |
| Write data corrupted | W beat ordering or WSTRB bug | Capture each accepted W beat and byte lane |
| BVALID appears too early | Response issued before write completion | Trace accepted AW, W beat count and internal commit |
| Read returns wrong ID | RID tracking lost in queue or interconnect | Follow ARID through every pipeline stage |
| RLAST one beat early/late | Off-by-one AxLEN counter | Check AxLEN + 1 and beat index |
| RDATA changes while stalled | Payload not held under RREADY backpressure | Probe RVALID, RREADY and RDATA together |
| Low throughput | Shallow outstanding queue, bubbles, target stalls | Measure accepted beats and outstanding high-water mark |
| Works in simple test, fails in SoC | AW/W assumed simultaneous or no independent stalls | Randomize channel timing independently |
| DMA fails near 4KB | Burst crosses boundary | Split burst before issuing |
| Only some byte writes fail | WSTRB lane mapping or alignment error | Test one-hot strobes at every byte offset |
| Rare multi-master corruption | ID aliasing or response misrouting | Track source-port identity plus transaction ID |
| Deadlock under stress | Cyclic resource dependency or poorly designed buffering | Build wait-for/resource dependency analysis |

### Clock and reset integration

- All five channels on a basic AXI interface share ACLK.
- A Subordinate must obey the interface reset requirements; AXI uses active-low ARESETn.
- Reset assertion and deassertion must be handled consistently with the applicable specification and SoC reset architecture.
- If endpoints use different clocks, use a proper AXI clock converter or asynchronous bridge; do not independently synchronize every payload bit.
- Clock gating must not stop a clock while a channel is expected to complete a handshake unless the system has a defined quiesce/retention protocol.

---

## 14. Post-silicon validation and debug

### 14.1 First bring-up checklist

1. Confirm clock frequency and reset release.
2. Perform a single-beat register write/read.
3. Check address decode and response status.
4. Test each byte lane with WSTRB.
5. Test a short INCR burst.
6. Increase burst length.
7. Introduce multiple outstanding requests if supported.
8. Stress read and write traffic concurrently.
9. Exercise multiple Managers through the interconnect.
10. Measure bandwidth, latency and stall counters.

### 14.2 What to capture in a trace

For writes:
- AWVALID/AWREADY and AW payload
- WVALID/WREADY and W payload
- BVALID/BREADY and B payload
- transaction ID and internal queue occupancy

For reads:
- ARVALID/ARREADY and AR payload
- RVALID/RREADY and R payload
- RID, RRESP and RLAST
- read queue occupancy and target latency

Add internal debug counters where possible:
- accepted AW count
- accepted W beat count
- accepted AR count
- completed B count
- accepted R beat count
- outstanding transaction count
- maximum stall length
- timeout count
- decode-error count

### 14.3 Example debug reasoning

**Symptom:** DMA starts a 4KB copy but hangs at a page boundary.

A disciplined debug sequence:

1. Read the DMA descriptor and confirm source, destination and length.
2. Capture the final AXI address before the hang.
3. Decode AxLEN, AxSIZE and AxBURST.
4. Calculate the last byte address.
5. Check whether the burst crosses the 4KB boundary.
6. Check whether the DMA splits the transfer correctly.
7. Check if the Subordinate accepted AW/AR and whether all data beats completed.
8. Check if the response is stuck behind BREADY/RREADY backpressure or a response-routing issue.

Do not immediately blame the DRAM controller. The bug could be in descriptor splitting, the AXI Manager, interconnect or bridge.

---

## 15. Design-rationale recap

| Design choice | Benefit | Cost / risk |
|---|---|---|
| Five independent channels | Decouples address, data and responses | More queues and state |
| VALID/READY | Explicit backpressure without a fixed-latency assumption | Source must hold payload stable |
| Burst transfers | Amortizes address/control overhead | Burst counters, boundary and alignment checks |
| Transaction IDs | Multiple requests can be in flight and responses associated | ID tracking and ordering complexity |
| WSTRB | Byte-granular writes | Byte-lane and alignment corner cases |
| Separate read/write paths | Concurrent read and write activity | More routing and buffering resources |
| AXI4-Lite | Simpler register/control path | No burst or ID-based concurrency |
| AXI4-Stream | Efficient address-free datapath | Requires a separate system contract for stream semantics |

The architecture is a set of trade-offs, not a universal performance guarantee. A small MCU may use APB/AHB for area and power. A high-bandwidth SoC may use AXI for CPU, DMA and accelerator traffic, then a coherent protocol such as ACE or CHI where cache-coherent sharing is required.

---

## 16. Hands-on exercises

### Exercise A — Decode a burst

A Manager issues:

- ARADDR = **0x8000_1000**
- ARLEN = 7
- ARSIZE = 3
- ARBURST = INCR
- Data width = 64 bits

Answer:
1. How many beats?
2. How many bytes per beat?
3. What is the total payload size?
4. What is the address of the final beat?
5. On which beat should RLAST be asserted?

### Exercise B — Throughput

An AXI4 interface has:
- 64-bit data width
- 250 MHz ACLK
- 80% accepted-beat efficiency

Calculate:
1. Peak one-direction bandwidth.
2. Effective bandwidth.
3. Time to transfer 1 MiB at that effective rate, ignoring software and setup overhead.

### Exercise C — Independent channels

Construct three legal write schedules:
1. AW handshakes before any W beat.
2. W data begins before AW handshakes.
3. AW and W first beat handshake on the same cycle.

For each, show when the Subordinate is allowed to produce BVALID.

### Exercise D — RTL review

Write a one-outstanding AXI4 Subordinate for a small SRAM:
- independently capture AW and W,
- accept a 4-beat INCR write,
- apply WSTRB,
- return one B response,
- accept a read request,
- return four R beats,
- hold R payload stable while RREADY is LOW.

Add assertions for VALID/payload stability and correct last-beat generation.

### Exercise E — Debug

A 128-bit AXI interface runs at 400 MHz. A performance counter reports 3.2 GB/s read bandwidth. What accepted-beat efficiency does this represent relative to the theoretical one-direction peak? List three possible causes for the lost throughput.

---

## 17. Interview questions

1. Why does AXI have five independent channels?
2. What exactly constitutes a VALID/READY handshake?
3. Can AW and W be accepted in different cycles?
4. Why must payload remain stable while VALID is HIGH and READY is LOW?
5. What is the difference between AxLEN and AxSIZE?
6. Why is AxLEN encoded as beats minus one?
7. What does WLAST or RLAST indicate?
8. Can a burst cross a 4KB boundary?
9. Why were WID and write-data interleaving removed in AXI4?
10. How do IDs support multiple outstanding transactions?
11. Can different IDs complete out of order?
12. What is the difference between SLVERR and DECERR?
13. How does WSTRB support partial writes?
14. Why can a Subordinate be protocol-correct but still have poor throughput?
15. How would you debug a write transaction that never returns BVALID?
16. What is the difference between AXI4, AXI4-Lite and AXI4-Stream?
17. Why is AXI not itself a crossbar or NoC?
18. What changes when a Manager is connected through an AXI clock converter?
19. Why is a FIFO often accessed with FIXED bursts?
20. How would you estimate the outstanding depth needed to hide memory latency?

---

## 18. Reference material

### Normative and primary Arm material

1. [Arm — AMBA AXI Protocol Specification, IHI 0022 (latest revision landing page)](https://developer.arm.com/documentation/ihi0022/latest) — use the applicable issue for the design being implemented; do not mix requirements across revisions.
2. [Arm — Introduction to AMBA AXI4, Learn the Architecture](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf) — introductory explanation of AXI4 bursts and AMBA generations.
3. [Arm — Introduction to AXI Protocol: Understanding the AXI Interface, 24 October 2016](https://developer.arm.com/community/arm-community-blogs/b/soc-design-and-simulation-blog/posts/introduction-to-axi-protocol-understanding-the-axi-interface) — accessible overview of the five channels.
4. [Arm — Introducing the next generation of AXI and ACE protocols, 29 January 2018](https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/introducing-the-next-generation-of-axi-and-ace-protocols) — AXI5/ACE5 context.
5. [Arm — AMBA moves forward with major revisions to AXI and CHI specifications, 15 October 2019](https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/updates-to-amba-axi-and-chi-specifications) — Issue G ordering, read data chunking and parity context.
6. [Arm — Evolution of AMBA Specifications, 2011–2013 white paper](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/CacheCoherencyWhitepaper_6June2011.pdf) — generation timeline and AXI4 evolution.

### Implementation and integration references

7. [AMD/Xilinx — Vivado Design Suite: AXI Reference Guide, UG1037](https://docs.amd.com/v/u/en-US/ug1037-vivado-axi-reference-guide) — practical AXI integration and optimization guidance.
8. [Public SystemVerilog AXI4 crossbar reference implementation](https://github.com/dpretet/axi-crossbar) — a concrete example of address decode, response routing, buffering, arbitration and configurable outstanding transactions. This is an implementation example, not the normative specification.

### Source discipline

- Protocol rules must be checked against the exact Arm specification issue selected by the project.
- AMD/Xilinx documentation describes vendor implementation guidance and must not be mistaken for a universal protocol requirement.
- Open-source RTL illustrates one implementation choice; it does not define AXI.
- No proprietary design history is inferred. Historical claims are limited to published sources.

---

## 19. Final mental model

Think of AXI as five independently flowing conversations:

- AW: “Here is the write I want to perform.”
- W: “Here are the bytes for that write.”
- B: “That write has completed; here is its status.”
- AR: “Here is the read I want.”
- R: “Here are the requested data beats and their status.”

At every rising clock edge, ask:

1. Which channels have VALID and READY together?
2. Which payloads were therefore accepted?
3. Which accepted requests are still outstanding?
4. Which IDs must be preserved?
5. Which channel is applying backpressure?
6. Is the design maintaining payload stability and transaction ordering?

If you can answer those six questions from a waveform, you have the foundation needed to debug AXI traffic in a real SoC.

**Next session — Lesson 06: AXI Advanced.** We will go deeper into burst address generation, unaligned accesses, byte strobes, ordering, exclusive accesses, QoS, AXI4-Lite, AXI4-Stream, bridges, width conversion and performance bottlenecks.
