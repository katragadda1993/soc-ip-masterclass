# Lesson 02 — On-chip Interconnect Foundations

**Track:** SoC Basics → Masterclass  
**Level:** Foundation to RTL microarchitecture and silicon integration  
**Estimated study time:** 60 minutes  
**Prerequisite:** Lesson 01 — SoC Foundations  
**Outcome:** Explain how multiple initiators reach multiple targets; design a first-pass address decoder, arbiter, shared bus, crossbar and bridge; identify ordering, backpressure, CDC, deadlock and performance risks.

> **Scope boundary:** This lesson teaches the interconnect as a system and microarchitecture. It does not replace the normative AXI, AHB, APB, TileLink or CHI specifications. Dedicated lessons will cover their exact signal rules, transaction encodings, timing and corner cases. Generic signal names below are conceptual unless identified as protocol-specific.

---

## 1. One-hour learning plan

| Time | Topic | Expected outcome |
|---:|---|---|
| 0–6 min | Why an interconnect exists | Distinguish endpoints, protocol and topology |
| 6–14 min | Historical backstory | Explain the move from shared buses to richer fabrics and NoC research |
| 14–23 min | Address decode and routing | Trace a request to the correct target |
| 23–33 min | Shared bus, crossbar and hierarchy | Compare concurrency, area, timing and power |
| 33–41 min | Arbitration, backpressure and ordering | Understand grant, acceptance, response routing and fairness |
| 41–48 min | Bridges, CDC and deadlock | Recognize integration hazards |
| 48–55 min | Performance and RTL design | Calculate bandwidth and outstanding work |
| 55–60 min | Verification, silicon debug and quiz | Build a practical validation checklist |

## 2. The interconnect's job

An interconnect is the communication infrastructure between **initiators** and **targets**.

- An **initiator** starts a transaction. Examples: CPU, DSP, GPU, NPU, DMA engine, PCIe controller or debug access port.
- A **target** responds to a transaction. Examples: SRAM controller, ROM, DRAM controller, UART register bank, interrupt controller or peripheral bridge.
- A **protocol** defines the legal interface behavior: how a request is presented, accepted, completed and reported.
- A **topology** defines how endpoints are connected: shared bus, crossbar, tree, ring, mesh or a hierarchy.
- A **fabric implementation** combines routing, arbitration, buffering, protocol adaptation, clock crossing, security checks and error handling.

The terms master/slave are common in older specifications and RTL. This course will use initiator/target or host/device in explanatory text where practical; exact specification terminology remains unchanged when discussing a named protocol.

### 2.1 Generic system view

~~~mermaid
flowchart LR
  CPU[CPU] --> FAB[Interconnect fabric]
  DSP[DSP] --> FAB
  DMA[DMA] --> FAB
  GPU[GPU] --> FAB
  DBG[Debug port] --> FAB
  FAB --> ROM[Boot ROM]
  FAB --> SRAM[SRAM]
  FAB --> MEM[Memory controller]
  FAB --> PERI[Peripheral bridge]
  FAB --> SEC[Security / firewall]
  PERI --> UART[UART]
  PERI --> TIMER[Timer]
  PERI --> GPIO[GPIO]
  PERI --> I2C[I2C / SPI]
  MEM --> PHY[DDR PHY]
  PHY --> DRAM[External DRAM]
~~~

The diagram is intentionally simplified. A real SoC often has separate high-performance and peripheral fabrics, multiple memory ports, coherent and non-coherent paths, security firewalls and multiple clock domains.

### 2.2 Why a CPU bus is not the whole system

Consider a multimedia SoC:

- CPU fetches instructions and accesses control registers.
- DSP streams audio samples from SRAM.
- Camera DMA writes image lines to DRAM.
- Display engine reads frame buffers from DRAM.
- GPU reads textures and writes render targets.
- PCIe DMA moves data to or from an external device.

These agents can be active at the same time. The interconnect must decide who can use each resource, preserve transaction identity, route responses, prevent illegal accesses and deliver acceptable latency to real-time traffic.

**Key insight:** A fabric can be protocol-correct and still fail the product because its arbitration, buffering or physical topology cannot meet worst-case traffic requirements.

## 3. Historical backstory: why interconnects evolved

### 3.1 Shared communication was the natural starting point

A shared bus is conceptually simple: multiple initiators share a common address/control/data path to multiple targets. An arbiter selects one initiator when more than one wants the bus. Address decode selects the target, and response logic returns data or status.

This model is attractive when there are few initiators, modest bandwidth requirements and limited physical design complexity. It also makes integration relatively straightforward: connect endpoints to a common bus contract and add decode and arbitration.

Its limitations become visible as the system grows:

- Only one transfer, or a limited set of transfers, can use the shared resource at a time.
- Unrelated initiators contend even when they want different targets.
- A slow target can hold up the shared resource unless the architecture isolates it.
- A wide global bus creates long wires, large fanout and difficult timing closure.
- Adding initiators increases arbitration and muxing complexity.

These are architectural trade-offs, not a claim that every historical bus had exactly the same restrictions. Pipelining, split transactions and multiple layers can improve concurrency, but the underlying shared-resource contention remains.

### 3.2 AMBA's evolution illustrates differentiated bus roles

Arm's AMBA family is a useful documented case study:

| Milestone | Documented change | Architectural significance |
|---|---|---|
| AMBA 2 generation | Added AHB alongside APB | A higher-performance pipelined system-bus option while retaining a compact peripheral bus |
| 16 June 2003 | First release of the AXI/ACE specification document | Marks the first release of the AXI specification document |
| 19 March 2004 | AXI specification v1.0 first release | Establishes the first numbered AXI specification release |
| AMBA 3 generation | Introduced AXI and AHB-Lite; extended APB | Supports more concurrency on high-performance paths and simpler peripheral attachment |
| 13 April 2010 | APB specification issue C, first release for APB v2.0 | Adds protection and sparse-write features in the APB family |

Arm describes AXI's five independent channels, flexible relative timing, multiple outstanding transactions and out-of-order data capability as enablers for pipelined, high-bandwidth interconnect. Arm describes APB as a compact, low-power way to isolate low-bandwidth register traffic from high-performance interconnects.

**Design rationale — engineering interpretation:** The family is not a contest where one bus replaces all others. Different traffic has different needs. A CPU or DMA memory path may need high throughput and concurrency; a timer register does not. Using a simpler peripheral path can reduce switching, wiring and endpoint complexity.

### 3.3 Why NoC became a research direction

In January 2002, Luca Benini and Giovanni De Micheli published *Networks on Chips: A New SoC Paradigm* in IEEE Computer. The paper argued for a layered, network-oriented approach to the communication complexity of large SoCs. It emphasized component reuse, reliable communication and the physical limits of global on-chip wires. It discussed energy and timing concerns as technology scaled.

That paper is a dated research milestone, not the single invention date of every NoC architecture. Packet-switched on-chip networks had earlier research antecedents, and commercial implementations evolved differently across vendors.

**The important backstory:** As systems gained more independent agents and larger physical footprints, the interconnect itself became a design problem. The network analogy offered a way to distribute communication through routers and links rather than relying on one global shared resource.

### 3.4 What changed, and what did not

| Architecture | What it improves | What it does not magically solve |
|---|---|---|
| Shared bus | Simplicity, compact control and easy integration for small systems | Contention on the shared resource |
| Pipelined/shared multi-layer bus | Overlap between address and data phases; independent layers in some designs | Physical scaling and shared-layer contention |
| Crossbar | Concurrent transfers between independent input/output pairs | Contention when several initiators need the same target; wiring and mux cost |
| Hierarchical fabric | Localizes traffic and can reduce global wiring | Requires careful partitioning and bridge design |
| NoC | Distributed routing, scalable connectivity and multiple paths | Router latency, buffering cost, congestion, deadlock and QoS complexity |

**Normative vs interpretation:** These are architectural comparisons, not protocol requirements. A specific crossbar or NoC must be evaluated from its actual RTL, configuration, timing and performance model.

## 4. Address decoding: how a request finds its target

A memory-mapped request normally carries an address plus operation information such as read/write, transfer size, byte enables, protection attributes and possibly a transaction ID.

The address decoder maps the address to a target-select signal.

### 4.1 Example address map

This is an illustrative map, not a real chip address map.

| Target | Base address | Last address | Size |
|---|---:|---:|---:|
| Boot ROM | 0x0000_0000 | 0x0003_FFFF | 256 KiB |
| SRAM | 0x1000_0000 | 0x1007_FFFF | 512 KiB |
| UART0 | 0x4000_0000 | 0x4000_0FFF | 4 KiB |
| Timer | 0x4001_0000 | 0x4001_0FFF | 4 KiB |
| DMA registers | 0x4002_0000 | 0x4002_0FFF | 4 KiB |
| DRAM window | 0x8000_0000 | 0xBFFF_FFFF | 1 GiB |

For a 4 KiB-aligned target window, a simple comparison can use the upper address bits:

- UART select: address[31:12] equals 0x40000.
- Timer select: address[31:12] equals 0x40010.
- DMA select: address[31:12] equals 0x40020.

The exact decoder depends on address width, map granularity, aliases, security regions and implementation style.

### 4.2 Decoder design checks

1. Every legal address maps to exactly one intended target, unless aliases are explicitly designed.
2. No unintended overlap exists between target windows.
3. Unmapped addresses return a defined error or route to a documented default error target.
4. Address attributes (secure/non-secure, privilege, instruction/data, stream ID or requester ID) are checked where required.
5. Decode is stable for the transaction it selects.
6. Decode and target selection are not accidentally based on changing live address signals after a request has been accepted.
7. Boundary addresses, address-width truncation and sign/zero extension are verified.

A dangerous bug is **decode aliasing**: two different software-visible addresses accidentally select the same target because upper bits were ignored or an address was truncated.

### 4.3 Request journey

~~~mermaid
sequenceDiagram
  participant H as Initiator
  participant D as Address decoder
  participant A as Arbiter / route logic
  participant T as Target
  H->>D: Address + command + attributes
  D->>A: Target select
  A->>T: Granted request
  T-->>A: Response + status + transaction identity
  A-->>H: Routed response
~~~

The fabric must associate the response with the correct initiator and request. A simple in-order fabric may use a queue. A fabric supporting multiple outstanding or out-of-order completions generally needs transaction IDs or equivalent tracking.

## 5. Shared bus, crossbar and hierarchical fabric

### 5.1 Shared bus

~~~mermaid
flowchart TB
  H1[CPU] --> ARB[Arbiter]
  H2[DMA] --> ARB
  H3[DSP] --> ARB
  ARB --> BUS[Shared address / control / data path]
  BUS --> D1[SRAM]
  BUS --> D2[DRAM controller]
  BUS --> D3[Peripheral bridge]
~~~

**Operation:**

1. Each initiator requests the shared resource.
2. The arbiter selects one eligible initiator.
3. The selected request is presented to the bus.
4. Address decode selects a target.
5. The target accepts the transfer and eventually responds.
6. The bus releases or advances according to the protocol.

**Advantages:** low conceptual complexity, compact topology for small designs, easy to reason about.

**Disadvantages:** contention, global fanout, long wires and limited concurrency. If one shared resource is occupied, an unrelated request may wait even if its target is idle.

Some bus protocols pipeline address and data or support split transactions. Therefore, do not assume every shared-bus architecture permits only one entire transaction to exist at a time. The exact protocol and implementation determine the overlap.

### 5.2 Crossbar

A crossbar provides selectable paths from multiple initiators to multiple targets.

~~~mermaid
flowchart LR
  H1[CPU] --> X[Crossbar switch]
  H2[DMA] --> X
  H3[DSP] --> X
  X --> M1[SRAM]
  X --> M2[DRAM]
  X --> P[Peripheral bridge]
~~~

If CPU accesses SRAM while DSP accesses DRAM, a crossbar may permit both transfers concurrently, provided the paths and internal resources are independent.

If CPU and DMA both access the same DRAM target, they still contend for that target port. A crossbar removes some global contention; it does not create infinite memory bandwidth.

**Microarchitecture commonly includes:**

- Input-side request capture or buffering.
- Address decode and route selection.
- One or more arbiters for each contended output.
- Request muxes that select the winning initiator.
- Response routing back to the source.
- Transaction-ID tracking or response queues.
- Optional pipeline registers and clock-domain bridges.

A fully connected M-by-N crossbar has a large number of potential input/output relationships. The wiring, muxing, arbitration and physical timing can become expensive as M and N grow. Real implementations use hierarchical, sparse or generated connectivity to reduce cost.

### 5.3 Hierarchical interconnect

A common system-level pattern is:

- High-performance initiators connect to a main fabric.
- Memory controllers and high-bandwidth accelerators attach to that fabric.
- A bridge leads to a peripheral subsystem.
- Low-bandwidth register targets share a compact peripheral bus.
- Security filters and clock-domain bridges are placed at chosen boundaries.

This makes the physical and functional partition explicit. It also creates integration questions: which transactions may cross the bridge, which attributes are preserved, how errors are translated and whether traffic can be blocked by a slow peripheral.

## 6. Arbitration: who gets the target?

When multiple initiators request the same target or shared resource, arbitration selects one request.

### 6.1 Common policies

| Policy | Behavior | Trade-off |
|---|---|---|
| Fixed priority | Highest-priority requester wins | Low latency for critical client; starvation risk for low priority |
| Round robin | Priority rotates among requesters | Fairer service; may not satisfy hard real-time deadlines |
| Weighted round robin | Requesters receive configured service weights | Balances fairness and bandwidth shares |
| Age-based | Older waiting request gains priority | Can limit starvation; needs age tracking |
| QoS/deadline-aware | Priority depends on traffic class, urgency or deadline | Useful for real-time traffic; more complex policy and verification |

Round robin is not automatically fair in terms of bytes or latency if requesters issue different burst lengths. A requester granted a long burst can occupy the target much longer than one issuing a short transfer. Some designs arbitrate per transaction, per beat, per time quantum or by credits.

### 6.2 Arbitration example

Assume CPU, camera DMA and DSP all request DRAM.

- CPU requests a 16-beat burst.
- Camera DMA requests a 64-beat burst.
- DSP requests a 4-beat burst.

A simple round-robin arbiter grants one transaction at a time. If the arbiter cannot preempt a burst, the DSP may wait behind the camera's long transfer even though it needs only four beats.

For real-time audio, the worst-case wait can matter more than average bandwidth. A QoS design may prioritize audio deadlines, limit maximum burst length, reserve bandwidth or use a separate path.

**Silicon symptom:** Audio underrun while camera capture is enabled may indicate excessive worst-case memory service latency rather than insufficient average DRAM bandwidth.

## 7. Backpressure, acceptance and response routing

Many modern interfaces use a ready/valid-style handshake, but the exact names and rules differ by protocol.

For a generic ready/valid request:

- Producer asserts VALID when request payload is available.
- Consumer asserts READY when it can accept the request.
- Transfer occurs on a clock edge when VALID and READY are both high.
- If VALID is high and READY is low, the producer must hold VALID and the associated payload stable until accepted, subject to the protocol's precise rules.

~~~text
Cycle:       0   1   2   3   4   5
VALID:       1   1   1   1   0   1
READY:       0   0   1   0   1   1
Transfer:    -   -   X   -   -   X
Payload:     A   A   A   B   B   C
~~~

At cycle 2, request A is accepted. Request B is then presented and stalled at cycle 3; it must remain stable until accepted at cycle 4. Request C is accepted at cycle 5.

This is an abstract handshake illustration, not a waveform for a specific AMBA protocol.

### 7.1 Why buffering is used

A FIFO or skid buffer can:

- Absorb short bursts of backpressure.
- Break long combinational paths.
- Decouple producer and consumer timing.
- Support clock-domain crossing when implemented as an asynchronous FIFO.
- Help meet timing by inserting a register stage.

But buffering costs area and latency. It can also hide congestion until the FIFO fills. Once full, backpressure propagates upstream.

### 7.2 Response routing

Suppose CPU and DMA both issue reads to the same memory target. Responses must return to the correct initiator.

Common approaches include:

- Per-initiator response queues.
- Transaction IDs carried through the fabric.
- A route table mapping returned IDs to source ports.
- In simple in-order paths, a FIFO of source identities.

The exact mechanism depends on protocol and implementation. Never assume that responses always return in request order unless the protocol and fabric guarantee it.

## 8. Bridges and adapters

A bridge connects two interfaces that differ in protocol, width, clock, power or security requirements.

### 8.1 Protocol bridge: high-performance fabric to APB

A typical AXI-to-APB bridge conceptually performs:

1. Accept an upstream request.
2. Capture address, operation, data and byte enables.
3. Decode the peripheral select.
4. Drive APB Setup phase.
5. Drive APB Access phase.
6. Wait while PREADY is low.
7. Capture read data and PSLVERR on completion.
8. Return the upstream response.
9. Release the bridge for the next transfer.

APB uses a Setup phase followed by an Access phase. PREADY can extend the Access phase; the address and control signals remain stable during the wait. The exact bridge implementation may queue requests, permit pipelining on its upstream side or support multiple APB ports, but the APB transfer itself follows the APB specification.

**Design trade-off:** A peripheral bridge keeps simple register devices off the high-performance fabric, but a single bridge can become a bottleneck if software or hardware frequently accesses many peripherals.

### 8.2 Other bridge types

| Bridge / adapter | Why it is needed | Key risk |
|---|---|---|
| Width converter | Connects 32-bit and 128-bit interfaces | Beat splitting/merging, alignment and byte strobes |
| Clock-domain bridge | Connects unrelated clocks | Metastability, reset coordination and FIFO overflow/underflow |
| Protocol converter | Connects AXI, AHB, APB, TileLink or proprietary interfaces | Semantic mismatch, error mapping and ordering |
| Register slice | Adds pipeline stages for timing closure | Added latency and changed timing budget |
| Security firewall | Checks requester and transaction attributes | Incorrect policy or lost sideband attributes |
| Address remapper | Translates address windows | Aliasing, overflow and inconsistent software map |

A bridge must preserve every attribute that matters: address, size, byte enables, operation type, security/protection attributes, transaction identity, ordering requirements and error status.

## 9. Clock-domain crossings and reset boundaries

A fabric may connect CPU, audio, USB, DRAM and always-on domains with different clocks.

A multi-bit request bus must not be synchronized by placing an independent two-flop synchronizer on every bit. Different bits could be captured in different cycles and form a corrupted address or data word.

Typical approaches:

- Asynchronous FIFO for request and response payloads.
- Handshake-based transfer for low-rate control bundles.
- Gray-coded FIFO pointers with synchronizers.
- Reset synchronization and explicit FIFO flush/recovery behavior.

An asynchronous bridge needs both source and destination clock/reset inputs. Verification must include independent clock ratios, phase drift, clock pause/restart, reset assertion during traffic and reset release ordering.

**Important:** A clock-domain bridge is not just a timing-cleanup element. It changes buffering, latency, outstanding capacity and reset behavior.

## 10. Ordering, outstanding transactions and deadlock

### 10.1 Ordering

Ordering defines which transactions must be observed before others.

Examples:

- A CPU writes a peripheral configuration register and then rings a DMA doorbell.
- A DMA engine writes data and then raises an interrupt.
- A device performs a read after a write to the same address.
- Two initiators access shared memory under a coherency model.

The interconnect must preserve the ordering rules required by the protocol and system memory model. It must not invent stronger or weaker ordering than the architecture permits.

A bridge can accidentally break ordering if it buffers writes but allows a dependent read or doorbell to pass without the required fence or completion rule.

### 10.2 Outstanding transactions

Outstanding means a requester has issued a transaction that has not yet received its completion response.

Multiple outstanding transactions can hide target latency. However, they require storage for request metadata, response association and flow-control state.

A useful first-order relation is Little's Law:

$$N \approx R \times T$$

where:

- N is average work in flight.
- R is sustained transaction or byte rate.
- T is average latency.

Example: to sustain 6 GB/s with 100 ns of round-trip latency, the fabric must have roughly:

$$6 \times 10^9 \times 100 \times 10^{-9} = 600\text{ bytes}$$

in flight. If each transaction transfers 64 bytes, approximately 10 such transfers must be in flight on average. This is a simplified estimate; protocol overhead, burst size, memory scheduling and queueing affect the real requirement.

### 10.3 Deadlock

A deadlock occurs when progress is impossible because resources are held while waiting for other resources in a cycle.

Illustrative pattern:

1. Path A holds a request buffer and waits for response buffer B.
2. Path B holds response buffer B and waits for request buffer A.
3. Neither side can release its resource.

Possible design controls include:

- Avoid cyclic channel dependencies.
- Separate request and response resources where appropriate.
- Ensure response capacity exists for accepted requests.
- Use credits or reserved response slots.
- Define timeout and error-recovery policy.
- Prove progress assumptions with formal or targeted stress tests.

A timeout can detect a stuck transaction, but it does not prove the architecture is deadlock-free.

## 11. Performance: bandwidth, latency and physical cost

### 11.1 Peak bandwidth

For a bus that transfers one beat per cycle:

$$B_{peak} = \frac{W}{8} \times f$$

For a 128-bit data path at 500 MHz:

- Width = 16 bytes per beat.
- Clock = 500 million beats/s.
- Peak = 8 GB/s.

If measured efficiency is 75%:

$$B_{effective} \approx 8 \times 0.75 = 6\text{ GB/s}$$

The 75% is an illustrative assumption. Actual efficiency depends on arbitration, burst length, target stalls, clock crossings, read/write turnaround and memory-controller scheduling.

### 11.2 Latency decomposition

A request's observed latency may be modeled as:

$$T_{total}=T_{queue}+T_{arb}+T_{decode}+T_{pipe}+T_{CDC}+T_{target}+T_{return}$$

This helps debug where time is spent. If target latency is fixed but arbitration delay increases when camera DMA starts, the bottleneck is likely contention or QoS rather than the peripheral itself.

### 11.3 Physical design cost

Interconnect cost is not only gate count.

- Wide buses consume routing tracks.
- Global control signals create fanout and buffering.
- Large muxes can become timing-critical.
- Crossbar paths can span a large physical region.
- Added pipeline stages increase latency and may change throughput behavior.
- Clocked buffers consume dynamic power.
- Congestion can make post-route timing worse than synthesis estimates.

A physically good architecture localizes traffic where possible and avoids unnecessary global connectivity.

## 12. RTL microarchitecture: blocks you should recognize

A simple crossbar implementation may contain:

1. Address decode logic per initiator.
2. Request eligibility and target selection.
3. Arbitration per target output.
4. Request mux per target.
5. Optional request FIFO or register slice.
6. Response demultiplexing by source or transaction ID.
7. Optional response FIFO.
8. Error response path for unmapped targets.
9. Performance counters and debug visibility.

### 12.1 Generic stalled-request assertion

For a generic ready/valid interface, a useful assertion concept is:

~~~systemverilog
property hold_payload_while_stalled;
  @(posedge clk) disable iff (!rst_n)
    req_valid && !req_ready
    |=> req_valid && $stable({req_addr, req_write, req_wdata, req_id});
endproperty

assert property (hold_payload_while_stalled);
~~~

Adapt this to the actual protocol, reset semantics and payload fields. Do not paste it into an AXI/AHB/APB checker without reviewing the relevant specification and signal behavior.

### 12.2 Common RTL bugs

- Decoder creates two target selects for one request.
- Unmapped access receives no response and hangs the initiator.
- Arbiter changes grant while a request is stalled.
- Response returns to the wrong initiator.
- Transaction ID is truncated or reused too early.
- FIFO full/empty logic drops or duplicates a transfer.
- Read and write paths accidentally share state.
- Reset clears request state but not corresponding response tracking.
- Width conversion mishandles byte lanes or unaligned transfers.
- CDC reset release causes stale pointers or phantom transactions.
- Priority policy starves low-priority traffic.
- A bridge returns completion before the downstream operation is actually complete.

## 13. Pre-silicon verification plan

### 13.1 Functional tests

- Every initiator reads and writes every legal target.
- Unmapped, protected and boundary addresses return defined responses.
- Simultaneous requests to the same target exercise arbitration.
- Simultaneous requests to independent targets test concurrency.
- Long bursts and short latency-sensitive requests contend.
- Backpressure is asserted for one cycle and for long random intervals.
- Responses are delayed, reordered where legal and returned under congestion.
- Reset is asserted during idle, request stall and outstanding response.
- Clock ratios and phases vary across CDC bridges.
- Error responses are propagated through every bridge.
- Security attributes survive all adapters.
- Power gating occurs only under the defined drain/idle conditions.

### 13.2 Assertions and coverage

Track:

- No request accepted twice.
- No accepted request disappears.
- No response without a matching accepted request.
- No response routed to the wrong source.
- No illegal multi-target select.
- No FIFO overflow/underflow.
- No protocol payload changes while stalled where prohibited.
- Fairness or bounded service properties for the selected arbiter policy.
- Address-space coverage and target connectivity coverage.
- Maximum outstanding count and queue occupancy.
- Latency histograms and throughput under contention.

Use simulation for scenarios, formal for invariants and progress assumptions, CDC/RDC tools for structural crossings, and emulation/FPGA when system-level traffic volume is too large for RTL simulation.

## 14. Post-silicon bring-up and debug

When a CPU or DMA request hangs, capture evidence at several points:

1. Did the initiator actually issue the request?
2. Was the request accepted at its source interface?
3. What address and attributes were observed?
4. Which target did the decoder select?
5. Did the arbiter grant the request?
6. Did the target accept it?
7. Did the target produce a response?
8. Did the fabric route the response to the right source?
9. Was the response blocked by backpressure?
10. Did a timeout, firewall or power-state transition intervene?

Useful observability features include:

- Per-target request and grant counters.
- Per-initiator stall counters.
- FIFO occupancy and high-water marks.
- Outstanding transaction counters.
- Latency histograms.
- Timeout and decode-error registers.
- Trace events for request accepted, grant, target response and completion.
- QoS class and requester ID visibility.

### Failure signatures

| Symptom | Likely areas to investigate |
|---|---|
| One peripheral address hangs | Decode, target select, bridge state machine, missing response |
| Only one initiator hangs | Arbitration, source-ID tracking, response demux |
| Failure under simultaneous DMA and CPU traffic | Arbitration, FIFO sizing, ordering, response routing |
| Failure only across clock domains | Async FIFO, CDC constraints, reset release, clock stop/restart |
| Audio underrun during camera capture | Worst-case DRAM latency, QoS, burst length, FIFO threshold |
| Works at low clock but fails at high clock | Combinational path, fanout, mux depth, routing congestion |
| Failure after warm reset | Outstanding-state flush, reset ordering, stale IDs or FIFO pointers |

**Debug discipline:** Distinguish protocol failure, functional routing failure, performance starvation and physical timing failure. They can produce similar software symptoms but require different fixes.

## 15. Practical lab: design an interconnect for a multimedia SoC

Assume these initiators:

- CPU instruction/data interface.
- DSP data interface.
- Camera DMA.
- Display DMA.
- Debug access port.

Assume these targets:

- Boot ROM.
- SRAM.
- DRAM controller.
- Peripheral bridge.
- Security/OTP target.

### Exercise A — connectivity matrix

Mark which initiators may access which targets. Do not automatically allow every master to access every target. Add security restrictions and debug policy.

### Exercise B — choose a topology

Propose:
- A main fabric for CPU, DSP and DMA.
- A path to DRAM and SRAM.
- A peripheral bridge.
- CDC boundaries for audio and always-on domains.

Explain why each path is shared or independent.

### Exercise C — arbitration

Camera and display DMA both request DRAM. CPU also issues cache misses. Define a policy that protects real-time display deadlines without starving CPU traffic. State whether you arbitrate per beat, burst or time window.

### Exercise D — performance

A 128-bit fabric at 500 MHz has 8 GB/s theoretical peak. If the target is 6 GB/s sustained, what minimum average efficiency is needed?

Answer: 6 / 8 = 75%, before considering any separate downstream bottleneck.

### Exercise E — fault injection

Create tests for:
- Unmapped address.
- Target never asserts ready.
- Response ID corruption.
- FIFO full.
- Clock stops during traffic.
- Reset during outstanding transaction.

For each, define expected detection and recovery behavior.

## 16. Knowledge check

1. What is the difference between a bus protocol and a crossbar topology?
2. Why can a crossbar allow CPU-to-SRAM and DSP-to-DRAM traffic simultaneously?
3. Why can two initiators still contend in a crossbar?
4. What is the difference between arbitration fairness and real-time QoS?
5. Why must a stalled ready/valid request hold its payload stable?
6. Why does a bridge need to preserve transaction identity and security attributes?
7. Why is an asynchronous FIFO generally used for a multi-bit CDC payload?
8. What can cause a response to return to the wrong initiator?
9. How does Little's Law help estimate the number of outstanding transactions?
10. Why does a timeout not prove the interconnect is deadlock-free?

### Suggested answers

1. Protocol defines legal transaction behavior; topology defines the connection structure.
2. The paths can be independent when the crossbar has separate resources for the two target outputs.
3. They contend when they request the same target, internal link, buffer or downstream resource.
4. Fairness shares service; QoS allocates service according to importance, latency or deadline requirements.
5. The receiver must see one coherent request until it accepts the request.
6. The upstream requester must receive the correct completion, and security policy must not be bypassed.
7. It transfers the entire multi-bit word coherently across unrelated clocks.
8. Lost/truncated IDs, incorrect response queues or faulty demultiplexing.
9. It relates average in-flight work to throughput and latency.
10. A timeout detects a lack of progress; it does not prove every possible resource dependency is acyclic.

## 17. Historical and normative source notes

- **AMBA history:** Arm's AMBA overview and AMBA generation page document the roles of AXI, AHB and APB and their evolution. The AXI/ACE specification change history records the first document release on 16 June 2003 and the first AXI v1.0 release on 19 March 2004.
- **APB transfer behavior:** Arm IHI 0024C describes Setup and Access phases, PREADY wait-state extension and the requirement to hold transfer controls stable during waits.
- **NoC history:** Benini and De Micheli's January 2002 IEEE Computer paper is a documented milestone in the network-on-chip design paradigm. Its motivation includes component reuse, communication complexity, physical wire delay and energy.
- **Public implementation example:** OpenTitan's TL-UL documentation describes host/device interfaces, a generated crossbar, request/response buffering, asynchronous FIFO support and round-robin arbitration in its M:1 socket. Those details describe that implementation, not every crossbar.
- **Protocol authority:** Exact legal behavior must be checked against the relevant protocol revision. This lesson's generic handshake and topology diagrams are explanatory models, not substitutes for AXI/AHB/APB/TileLink specifications.

## 18. References

1. Arm, [AMBA overview and current specification family](https://www.arm.com/architecture/system-architectures/amba).
2. Arm, [AMBA 4 / AMBA 3 / AMBA 2 generation overview](https://www.arm.com/architecture/system-architectures/amba/amba-4).
3. Arm, [AMBA AXI and ACE Protocol Specification, IHI 0022H](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/IHI0022H_amba_axi_protocol_spec.pdf). Consult the current official page for later revisions.
4. Arm, [AMBA AHB Protocol Specification, IHI 0033](https://developer.arm.com/documentation/ihi0033/latest).
5. Arm, [AMBA APB Protocol Specification, IHI 0024C](https://documentation-service.arm.com/static/60d5b617677cf7536a55c273).
6. Benini and De Micheli, [Networks on chips: a new SoC paradigm](https://doi.org/10.1109/2.976921), IEEE Computer, Volume 35, Issue 1, January 2002, pp. 70–78.
7. OpenTitan, [TL-UL Bus Specification](https://opentitan.org/book/hw/ip/tlul/).
8. OpenTitan, [Crossbar Generation Tool (tlgen)](https://opentitan.org/book/util/tlgen/index.html).
9. OpenTitan, [TL-UL protocol checker](https://opentitan.org/book/hw/ip/tlul/doc/TlulProtocolChecker.html).

### Source and licensing discipline

- Protocol signal behavior must be verified against the relevant normative specification revision.
- Public OpenTitan details are labeled as implementation examples, not universal rules.
- Historical design rationale is stated only where supported by dated public material; broader comparisons are labeled engineering interpretation.
- Do not copy proprietary specification text, figures or tables into this repository. The lesson uses original explanations and diagrams with links to the source documents.
- The repository currently has a root-level MIT LICENSE; preserve it and retain any required third-party notices.

---

## Next lesson

**Lesson 03 — APB protocol deep dive:** historical rationale, APB2/APB3/APB4/APB5 evolution, Setup and Access state machine, PSEL/PENABLE/PREADY/PSLVERR/PPROT/PSTRB, read/write timing, wait states, error handling, register interface design, APB bridge RTL, assertions, testbench scenarios and silicon debug.
