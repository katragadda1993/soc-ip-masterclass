# Lesson 07 — AXI Interconnect Implementation: Crossbar Microarchitecture, Routing, Arbitration, Deadlock, Timing Closure and Performance Instrumentation

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 07 of 38  
**Suggested duration:** 60 minutes  
**Level:** Advanced RTL architecture, integration, verification and post-silicon debug  
**Primary focus:** A memory-mapped AXI4 crossbar with multiple initiators and targets

> **Learning objective:** Starting from a system connectivity table, derive an interconnect topology, decode addresses, arbitrate per target, route write data and responses, preserve transaction identity, size buffering, identify deadlock risks and define a practical verification and silicon-observability plan.

---

## 1. One-hour lesson plan

| Time | Topic | Engineering outcome |
|---:|---|---|
| 0–6 min | Why interconnects evolved | Explain the trade-off between shared buses, crossbars and hierarchical fabrics |
| 6–14 min | Crossbar block architecture | Identify ingress, egress, decode, arbitration and response-routing blocks |
| 14–23 min | Address decode and ID management | Build a non-overlapping address map and route responses to the correct initiator |
| 23–33 min | AXI write and read paths | Understand AW/W association, R/B routing and channel independence |
| 33–41 min | Arbitration, QoS and buffering | Select policies and estimate queue depth |
| 41–48 min | Deadlock and timing closure | Recognize cyclic dependencies and long ready/valid paths |
| 48–56 min | RTL and verification architecture | Define datapath control, assertions, scoreboard and stress tests |
| 56–60 min | Silicon debug and exercises | Turn a stalled bus into a measurable debug hypothesis |

### Prerequisites

- Lesson 02: shared bus, crossbar, bridge, arbitration and backpressure.
- Lesson 05: AXI five-channel handshakes and IDs.
- Lesson 06: bursts, outstanding requests, ordering, QoS and performance.

### Outcomes

By the end, you should be able to:

1. Draw a multi-initiator, multi-target AXI crossbar.
2. Explain why address decode and arbitration are per destination.
3. Distinguish an address grant from a data-channel grant.
4. Explain why AXI4 write-data routing is particularly subtle without WID.
5. Extend or remap IDs so responses return to the correct initiator.
6. Design default error handling for unmapped addresses.
7. Compare round-robin, fixed-priority and weighted arbitration.
8. Identify a documented cyclic-dependency deadlock pattern.
9. Estimate interconnect buffering from throughput and latency.
10. Build an observability plan for post-silicon validation.

---

## 2. Historical backstory: why the interconnect became an IP of its own

### 2.1 The original scaling problem

A small microcontroller can connect a CPU, SRAM and peripherals using a simple shared bus. As a system grows, independent requesters appear:

- CPU instruction and data paths
- Display and camera engines
- Audio DSP and codec DMA
- USB, PCIe and Ethernet DMA
- GPU/NPU accelerators
- Debug and security engines

A shared bus serializes traffic at its shared resource. If two independent requesters need two different targets at the same time, a shared bus may still force them to take turns.

A crossbar introduces multiple paths. If Master 0 targets SRAM while Master 1 targets a separate peripheral, both paths may progress concurrently. If both target the same DDR controller, they still contend for that target's acceptance capacity and memory bandwidth.

**Important distinction:** a crossbar does not create additional bandwidth inside a congested target. It creates independent connectivity between non-conflicting source/destination pairs.

### 2.2 Why the crossbar is not a free upgrade

A full N-initiator by M-target crossbar needs routing and selection resources across the connectivity matrix. The rough architectural scaling pressure is proportional to N × M, multiplied by datapath width and the number of independent channels or planes.

This is not an exact gate-count formula. Synthesis sharing, sparse connectivity, pipeline placement, buffering and implementation style all affect actual area and timing.

The trade-off is:

| Architecture | Advantage | Cost or limitation |
|---|---|---|
| Shared bus | Small and easy to reason about | Serializes access to the shared bus |
| Multi-layer / hierarchical bus | More concurrency than one shared bus | Topology and bridge complexity |
| Full crossbar | Concurrent paths to independent targets | Mux, arbitration, wiring and state scale with connectivity |
| Sparse crossbar | Removes impossible source-target paths | Requires accurate connectivity description |
| Hierarchical crossbar | Local traffic stays local; fewer global wires | Some traffic crosses multiple stages |
| Network-on-Chip | Scales using routers, links and flow control | Packetization, routing, virtual channels and network-level deadlock analysis |

### 2.3 Publicly documented implementation milestones

There is no single date on which all SoC crossbars were invented. Crossbar switching is a general switching topology with a much longer history. The relevant SoC lesson is how configurable AMBA interconnect products made these fabrics reusable and generatable.

| Date / release | Public evidence | Engineering significance |
|---|---|---|
| 7 August 2012 | Arm CoreLink NIC-400 TRM first release | Documents a configurable network interconnect built from switches, interface blocks and domain-crossing functions |
| 2012–2013 | NIC-400 revisions r0p0 to r0p2 | Public revision history documents product evolution and configuration refinements |
| 2025-12-19 | AMD AXI Interconnect PG059 version 2.1 release | Documents a modern crossbar plus optional infrastructure blocks such as register slices, FIFOs, clock converters, width converters and protocol converters |
| Current OpenTitan documentation | TL-UL crossbar generator | Demonstrates configuration-driven generation of RTL and DV collateral from a topology description |
| Current PULP AXI project | Public synthesizable SystemVerilog AXI modules | Demonstrates modular composition: muxes, demuxes, crossbars, ID remapping, FIFOs, width conversion and CDC |

Arm's NIC-400 documentation describes configurations ranging from a single bridge to a complex network with multiple switches and interface blocks. It also documents a generation flow that produces RTL, testbench/stimulus and synthesis scripts from configuration. AMD PG059 describes a crossbar surrounded by optional infrastructure/coupler blocks. OpenTitan's tlgen reads a connectivity description and generates crossbar RTL and DV collateral. These are documented product/project facts, not a claim that all commercial SoCs use the same microarchitecture.

### 2.4 The design-rationale lesson

A modern interconnect is not merely a large address mux. It is a configurable subsystem that must solve several independent problems:

1. **Reachability:** which initiator may reach which target?
2. **Selection:** which requester wins when several want the same target?
3. **Association:** how does a response find its original requester?
4. **Ordering:** which transactions must remain ordered?
5. **Flow control:** how much traffic can be accepted and buffered?
6. **Isolation:** how are clock, reset, power and security boundaries handled?
7. **Observability:** how can a silicon engineer see where traffic is blocked?

The exact answer is implementation-specific. AXI defines endpoint protocol behavior; it does not prescribe one universal crossbar architecture.

---

## 3. System-level architecture: source ports, target ports and connectivity

### 3.1 Terminology warning

Different vendor documents use different interface names. In this lesson:

- **Initiator / Manager / Master:** the IP that launches an AXI transaction.
- **Target / Subordinate / Slave:** the IP that receives the transaction.
- **Ingress port:** the crossbar port connected to an initiator.
- **Egress port:** the crossbar port connected to a target.

Some AMD IP documentation calls the initiator-facing port an SI (Slave Interface) because the interconnect is acting as a slave to the external master. Its target-facing port is called an MI (Master Interface). Always translate the naming from the perspective of the connected device.

### 3.2 Example SoC

Consider three initiators and four targets:

| Initiator | Function | Targets it can access |
|---|---|---|
| CPU | Instruction/data traffic | SRAM, DDR, APB bridge, crypto |
| DMA | Bulk data movement | SRAM, DDR, APB bridge |
| DSP | Audio/compute traffic | SRAM, DDR, APB bridge, crypto |

Targets:

- T0: on-chip SRAM
- T1: DDR memory controller
- T2: APB peripheral bridge
- T3: crypto engine

Connectivity matrix:

| Initiator → Target | SRAM | DDR | APB bridge | Crypto |
|---|:---:|:---:|:---:|:---:|
| CPU | ✓ | ✓ | ✓ | ✓ |
| DMA | ✓ | ✓ | ✓ | — |
| DSP | ✓ | ✓ | ✓ | ✓ |

The dash means the route is intentionally absent. A sparse connectivity matrix reduces unnecessary muxing and prevents an initiator from reaching a target it should not access.

### 3.3 Conceptual block diagram

~~~mermaid
flowchart LR
  CPU[CPU AXI] --> I0[Ingress 0]
  DMA[DMA AXI] --> I1[Ingress 1]
  DSP[DSP AXI] --> I2[Ingress 2]

  I0 --> DEC[Address decode and route metadata]
  I1 --> DEC
  I2 --> DEC

  DEC --> ARB[Per-target arbitration]
  ARB --> FAB[AXI crossbar datapaths]

  FAB --> SRAM[SRAM target]
  FAB --> DDR[DDR controller]
  FAB --> APB[APB bridge]
  FAB --> CRYPTO[Crypto target]

  SRAM --> RESP[Response routing and ID restore]
  DDR --> RESP
  APB --> RESP
  CRYPTO --> RESP

  RESP --> CPU
  RESP --> DMA
  RESP --> DSP
~~~

This is a teaching-level architecture. A production design normally has separate read-address, write-address, write-data, read-response and write-response control paths, plus buffering and optional converters.

### 3.4 The internal functional blocks

| Block | Main responsibility |
|---|---|
| Address decoder | Convert AWADDR/ARADDR into a target selection |
| Connectivity filter | Reject or remove prohibited routes |
| AW arbiter | Select one initiator for a target's write-address channel |
| AR arbiter | Select one initiator for a target's read-address channel |
| W router | Send write data to the target selected by the corresponding AW transaction |
| R router | Return read data to the original initiator |
| B router | Return write responses to the original initiator |
| ID extender/remapper | Prevent collisions between independent initiators |
| Acceptance/issuing trackers | Enforce configured in-flight limits |
| Default error target | Return DECERR for unmapped accesses |
| Register slice/FIFO | Break timing paths and absorb backpressure |
| CDC/width/protocol converter | Adapt clock, datapath width or protocol where required |

---

## 4. Address decode: how an address selects a target

### 4.1 Address map example

| Target | Base address | Size | End address |
|---|---:|---:|---:|
| SRAM | 0x0000_0000 | 256 KiB | 0x0003_FFFF |
| DDR | 0x8000_0000 | 1 GiB | 0xBFFF_FFFF |
| APB bridge | 0x4000_0000 | 64 KiB | 0x4000_FFFF |
| Crypto | 0x5000_0000 | 64 KiB | 0x5000_FFFF |

For a power-of-two region, a common conceptual decode is:

\[
hit_i = ((address \mathbin{\&} mask_i) = base_i)
\]

where the mask has ones in the address bits that identify the region. For a region of size \(2^k\), the low k address bits are ignored by the region comparison.

Example: a 64 KiB region has 16 low address bits inside the region. A conceptual mask is 0xFFFF_0000 for a 32-bit address map.

### 4.2 Decode requirements

- The address map must be unambiguous.
- Base addresses must satisfy the alignment rules of the chosen region representation.
- No two target windows should overlap unless an explicitly documented priority/alias policy exists.
- The complete original address may still be forwarded to the target.
- A miss must be handled deliberately; it must not select an arbitrary target.
- Access-control checks may be per initiator, not merely global decode.
- Address remapping is a separate function from selecting a target.

AMD PG059 documents power-of-two range sizes, aligned bases and non-overlapping address ranges for its configured AXI Crossbar. It also describes a DECERR path for addresses that match no configured target. This is an implementation example, not a universal rule that every crossbar must use the same configuration format.

### 4.3 Default error response

A robust fabric should define behavior for an unmapped address:

- Read: accept AR, return a protocol-compliant read response with DECERR and the correct number of beats/last indication for the supported implementation contract.
- Write: accept AW and the associated W burst according to the chosen error-target architecture, then return a B response with DECERR.
- Do not silently drop the transaction.
- Do not accidentally forward it to a valid peripheral.

The default target must obey the protocol and must not deadlock when the requester applies backpressure.

### 4.4 Decode RTL sketch

The following is an illustrative combinational decode, not a complete production decoder:

~~~systemverilog
always_comb begin
  target_hit = '0;

  target_hit[0] = (addr[31:18] == 14'h0000); // example SRAM window
  target_hit[1] = (addr[31:30] == 2'b10);    // example DDR window
  target_hit[2] = (addr[31:16] == 16'h4000); // APB bridge
  target_hit[3] = (addr[31:16] == 16'h5000); // crypto

  if ($onehot(target_hit))
    target_valid = 1'b1;
  else
    target_valid = 1'b0;
end
~~~

The constants and comparisons must be derived from the actual address map. The sketch intentionally omits parameterization and default-error handling. A real implementation should check for zero hits and multiple hits, and should generate the selected target index only after the decode is known to be valid.

Useful verification properties:

- No legal address hits more than one target.
- Every documented address range hits exactly one intended target.
- Every unmapped address reaches the error path.
- A target not connected to an initiator never receives its request.
- Secure-only targets reject prohibited non-secure traffic according to the system security policy.

---

## 5. AXI IDs: preserving the source of a transaction

### 5.1 Why IDs become a crossbar problem

Each initiator has its own local ID namespace. Two different initiators can both issue a request with ID 3. If both requests are forwarded to the same target without any source distinction, the target may return a response whose ID is insufficient for the crossbar to determine which initiator should receive it.

The crossbar therefore needs a response-routing strategy.

### 5.2 Strategy A — Prefix the source-port index

For three initiators, two source bits are sufficient to encode the source port:

\[
ID_{fabric} = \{source\_index,\ ID_{local}\}
\]

Example with a 4-bit local ID:

| Initiator | Local ID | Fabric ID |
|---|---:|---:|
| CPU (00) | 3 (0011) | 000011 |
| DMA (01) | 3 (0011) | 010011 |
| DSP (10) | 3 (0011) | 100011 |

The response returns with the extended ID. The crossbar extracts the source field, routes the response and restores the local ID.

Advantages:
- Simple response routing.
- Distinguishes same-valued IDs from different initiators.
- Can avoid a large associative lookup table.

Costs:
- Increases downstream ID width.
- The target and all downstream bridges must preserve the extended ID.
- The source index must not be truncated by a converter.

### 5.3 Strategy B — Allocate and track internal IDs

If downstream ID width cannot grow, the fabric can allocate a free internal ID and store a mapping:

| Internal ID | Source port | Original ID | Target | Direction |
|---:|---|---:|---|---|
| 0x05 | CPU | 3 | DDR | Read |
| 0x06 | DMA | 3 | DDR | Read |
| 0x07 | DSP | 1 | SRAM | Write |

When a response returns, the table restores the original source and ID. The entry is freed only when the response transaction is fully completed according to the implementation's tracking rules.

Costs:
- Table allocation and free logic.
- CAM or indexed lookup, depending on implementation.
- More complex reset and error handling.
- A finite number of remappable outstanding transactions.

### 5.4 ID correctness checklist

- Never allow two active source transactions to become indistinguishable at a target if their responses need different routes.
- Preserve the ID across every pipeline and conversion block, or explicitly remap it.
- Restore the exact original ID at the source-facing interface.
- Respect same-ID ordering rules.
- Track read and write namespaces according to the chosen implementation.
- Do not free an ID-map entry before the complete response is accepted.
- Verify behavior when the ID allocator is full.
- Verify responses arriving out of order across different IDs.
- Verify that reset flushes or safely aborts all outstanding route state according to the system reset contract.

AMD PG059 describes using propagated IDs to identify the original source interface and route responses back to it. Its configuration also includes ID width, source thread-ID width and acceptance/issuing limits. These are concrete examples of the bookkeeping required by a real crossbar.

---

## 6. Write path: AW routing and W-data association

### 6.1 Why the write path is subtle

AXI has independent AW and W channels. AW can be accepted before the write data is available, or W can be presented before the address is accepted, depending on the endpoint's behavior.

AXI4 has no WID signal. Therefore, a crossbar cannot look at WID to decide which outstanding write transaction a beat belongs to.

For each initiator, write-data order must remain associated with that initiator's accepted write-address order. At a target, the crossbar must also preserve the order in which write addresses were issued to that target.

### 6.2 Conceptual write path

1. Initiator asserts AWVALID with AWADDR, AWID and burst attributes.
2. Ingress captures the address and decodes the target.
3. The target's AW arbiter selects one eligible initiator.
4. The target accepts the AW transaction.
5. The crossbar records the source and write-route metadata in the target's write-order tracking structure.
6. The W router selects the source whose data belongs to the oldest forwarded AW transaction for that target.
7. The selected W stream is forwarded until WLAST is accepted.
8. The target eventually returns BVALID/BID/BRESP.
9. The B router uses the extended ID or lookup table to route the response to the original initiator.

### 6.3 Write route FIFO concept

A simple implementation may use a per-target FIFO that records which initiator owns the next write-data burst:

| FIFO position | Source | Original ID | Expected data completion |
|---:|---|---:|---|
| Head | DMA | 2 | Wait for accepted WLAST |
| Next | CPU | 5 | Not yet selected |
| Next | DSP | 1 | Not yet selected |

The W router must not skip the head entry merely because another initiator has WVALID high. It must preserve the target's AW-to-W association.

A simple and conservative design locks the W-data source for the selected burst until the accepted WLAST beat. This prevents accidental beat-level mixing. The exact buffering and scheduling implementation can differ, but the externally visible transaction ordering must remain correct.

### 6.4 Important corner case: W arrives before AW routing is ready

Suppose DMA asserts WVALID before the corresponding AW has won arbitration. The crossbar has several legal design choices:

- Backpressure W by keeping the source-facing WREADY low until the route metadata is available.
- Buffer the W beats in a per-source FIFO until the corresponding AW route is committed.
- Use a combined scheduling structure that reserves the write route before accepting the relevant W data.

The design must ensure that the W stream cannot be assigned to the wrong target. It must also avoid a circular wait in which AW cannot be accepted until W is buffered while W cannot be accepted until AW is accepted.

### 6.5 Write-path timing sketch

~~~text
Cycle             0    1    2    3    4    5    6
CPU AWVALID       1    1    0    0    0    0    0
CPU AWREADY       0    1    0    0    0    0    0
DMA AWVALID       1    1    1    0    0    0    0
DMA AWREADY       0    0    1    0    0    0    0
DMA WVALID        0    1    1    1    1    0    0
DMA WREADY        0    0    1    1    1    0    0
DMA WLAST         0    0    0    0    1    0    0
Target W source   -    -   DMA  DMA  DMA   -    -
~~~

This is only a conceptual schedule. Actual handshakes depend on arbitration, route buffering and target readiness. The important point is that the W source is selected from recorded AW routing state, not by guessing from whichever WVALID happens to be high.

### 6.6 Write-path failure signatures

- AW reaches DDR but W reaches SRAM: route metadata or W mux selection is wrong.
- W beats are duplicated: FIFO pop or handshake accounting is wrong.
- WLAST never reaches target: burst counter or source lock is wrong.
- B response reaches the wrong initiator: ID extension/remapping or response mux is wrong.
- B response never appears: target completion, internal write-done, response FIFO or reset/clock path is stuck.

---

## 7. Read path: address arbitration and response routing

### 7.1 Read request path

1. Initiator presents ARVALID, ARADDR, ARID and attributes.
2. The ingress decoder selects the target.
3. The target's AR arbiter selects one eligible initiator.
4. The crossbar forwards AR with the fabric ID.
5. The target returns RVALID, RID, RDATA, RRESP and RLAST.
6. The crossbar decodes the source from RID or an internal mapping table.
7. The response is routed to that initiator.

### 7.2 Multiple targets responding to one initiator

A single initiator has one R channel. If two targets have read data ready for that initiator at the same time, the crossbar must select one source for the output cycle and apply backpressure to the other source.

For a conservative AXI4 implementation, the crossbar can hold a selected source through the entire response burst until an accepted RLAST. The exact permitted response ordering and interleaving behavior must be checked against the selected AXI revision and the capabilities of connected endpoints.

### 7.3 Read-response routing state

Useful metadata includes:

- Original source port
- Original transaction ID
- Target port
- Expected response ordering/thread state
- Outstanding beat/transaction count where required by the implementation

A response router must not rely only on the current address bus. The address phase may be long gone by the time RVALID returns.

### 7.4 Read-path corner cases

- Target returns a different ID than the one accepted.
- RLAST is early or missing.
- One target holds RVALID while the source has RREADY low.
- Two targets compete for one source R channel.
- Response arrives after a reset or power transition.
- ID map is exhausted.
- Target returns out-of-order responses for different IDs.
- A bridge changes ID width or strips sideband information.

---

## 8. Arbitration: who gets the target?

### 8.1 Arbitration is per contested resource

For a crossbar, a target's AW channel may have several initiators requesting it. A separate target AR channel may also have several requesters. Each target therefore normally needs separate read-address and write-address arbitration state.

The write-data path is not simply another independent address arbiter: it must honor the AW-to-W association described above.

### 8.2 Common arbitration algorithms

| Policy | How it works | Advantages | Risks |
|---|---|---|---|
| Fixed priority | Highest-priority requester wins | Simple, predictable for highest class | Low-priority starvation |
| Round-robin | Rotate priority after grants | Good basic fairness | Does not inherently guarantee latency |
| Weighted round-robin | Allocate service proportional to weights | Supports differentiated bandwidth | Weight and burst-size interactions need analysis |
| Age-based | Prefer the oldest waiting request | Helps bound waiting under assumptions | Requires age state and careful tie-breaking |
| QoS priority | Use transaction QoS class in selection | Supports traffic differentiation | QoS is not itself a deadline guarantee |

### 8.3 Arbitration granularity matters

Ask what the arbiter is granting:

- One address handshake?
- One transaction?
- One burst?
- One data beat?
- A time slot or bandwidth credit?

For a read-address arbiter, a grant may last only through AR acceptance, while the response returns later through a separate router.

For a write-data arbiter, the selected source generally needs to remain associated with the target's current write burst through the accepted WLAST.

If a policy locks a target for a long burst, a short urgent request may wait. If the policy switches every beat without preserving AXI write ordering, it can corrupt transaction association. The scheduling unit must match the protocol's ordering constraints.

### 8.4 Round-robin example

Three requesters continuously request one target. A simple round-robin arbiter can grant:

~~~text
Grant sequence: CPU → DMA → DSP → CPU → DMA → DSP → ...
~~~

This assumes all requesters remain eligible and each grant completes. If a requester is blocked by an acceptance limit, it may be temporarily removed from eligibility.

### 8.5 QoS is a system policy, not a magic wire

AXI has QoS sideband fields such as ARQOS and AWQOS. The fabric may use these fields in arbitration, propagate them to downstream IP or override them according to configuration.

A QoS value does not guarantee that a request meets a deadline. The memory controller, downstream fabric, target queueing and competing traffic must all be considered.

The Arm NIC-400 TRM documents static, programmable or incoming QoS allocation and arbitration using QoS values, with a tie-break policy. The AMD PG059 guide documents configurable source priorities and round-robin behavior for a particular priority configuration. These examples show that arbitration details are implementation-specific.

---

## 9. Acceptance limits, issuing limits and queue sizing

### 9.1 Two different limits

- **Acceptance limit:** how many transactions the fabric accepts from an ingress initiator.
- **Issuing limit:** how many transactions the fabric allows to be active toward an egress target.

These limits prevent unbounded state growth and can protect a target from receiving more traffic than it can manage.

An initiator can have available acceptance capacity while a particular target is full. Conversely, one congested target should not necessarily block traffic to every other target.

### 9.2 Buffer placement

Common buffer locations:

- Per-initiator AW and AR queues
- Per-target arbitration queues
- Per-target write-route metadata FIFO
- Per-channel W and R data FIFOs
- Per-initiator B and R response queues
- ID mapping tables
- Clock-domain crossing FIFOs

Not every design needs every FIFO. Buffer placement is a trade-off between timing, throughput, area, latency and the amount of outstanding state.

### 9.3 Bandwidth-delay product

To keep a path busy, enough data or transactions must be in flight to cover the round-trip latency.

\[
Bytes_{flight} \ge Bandwidth_{target} \times Latency_{roundtrip}
\]

Example assumptions:

- AXI datapath = 256 bits = 32 bytes per beat
- Clock = 800 MHz
- Effective target throughput = 80% of one beat per cycle
- Round-trip latency = 90 ns
- Burst payload = 128 bytes

Peak datapath bandwidth:

\[
32 \times 800 \times 10^6 = 25.6\ GB/s
\]

Target throughput assumption:

\[
25.6 \times 0.8 = 20.48\ GB/s
\]

Bytes required in flight:

\[
20.48\ GB/s \times 90\ ns = 1843.2\ bytes
\]

Approximate 128-byte bursts required:

\[
\lceil 1843.2 / 128 \rceil = 15
\]

A starting design target might allow at least 16 outstanding 128-byte bursts, subject to the target's own acceptance capacity, refresh behavior, arbitration and actual measured latency.

This is a sizing estimate, not an AXI requirement. It also does not mean that every one of those bursts must be stored as a complete data FIFO; some may be represented by address/ID metadata while the target performs the data movement.

### 9.4 Contention example

Two independent targets each have a 128-bit datapath at 500 MHz:

\[
Bandwidth_{one\ target} = 16 \times 500\ MHz = 8\ GB/s
\]

If both targets can operate concurrently, the fabric may carry up to 16 GB/s of aggregate data under ideal assumptions.

If CPU, DMA and DSP all target the same DDR controller, they still share that target's service rate. The crossbar arbitrates; it cannot make the DDR controller deliver three times its physical capacity.

---

## 10. Deadlock: a real interconnect-level failure mode

### 10.1 Deadlock is more than READY staying low

A stalled handshake can be legal backpressure. Deadlock occurs when dependencies prevent all involved transactions from making progress, even though each component is waiting for another event that cannot occur.

The interconnect must be analyzed together with target response ordering, finite queues, IDs and outstanding limits.

### 10.2 A documented two-master, two-target example

AMD PG059 documents a read-response cyclic-dependency scenario:

1. M0 reads S0.
2. M0 then reads S1 using the same ID thread.
3. M1 reads S1.
4. M1 then reads S0 using the same ID thread.
5. S0 responds to M1 first. That response cannot be forwarded to M1 because M1's earlier same-ID response from S1 must be delivered first.
6. S1 responds to M0. That response cannot be forwarded to M0 because M0's earlier same-ID response from S0 must be delivered first.
7. Both responses are blocked by the ordering constraints, producing deadlock.

The key lesson is not that AXI is inherently deadlocking. It is that a legal set of endpoint reorderings combined with a particular interconnect ordering policy and multiple outstanding transactions can create a cyclic dependency.

### 10.3 Mitigation options

| Strategy | Benefit | Trade-off |
|---|---|---|
| Restrict one ID thread to one target at a time | Simple way to avoid cross-target same-ID cycles | Reduces concurrency |
| Allocate distinct IDs for independent target streams | Allows more response independence | Requires ID capacity and correct mapping |
| Add sufficient independent response buffering | Can break some resource dependencies | More area; must prove the buffer policy is sufficient |
| Use deadlock-aware scheduling | Avoids unsafe combinations of outstanding routes | More complex control |
| Partition traffic into independent virtual networks | Reduces head-of-line and cyclic dependencies | Additional resources and routing complexity |

Do not treat a single mitigation as universally sufficient. The correct solution depends on the AXI revision, the exact interconnect ordering model, target capabilities and system traffic constraints.

### 10.4 Deadlock verification

- Construct traffic that targets multiple slaves with repeated IDs.
- Allow targets to return responses in different legal orders.
- Stall BREADY and RREADY independently.
- Fill acceptance and issuing queues.
- Randomize target latency.
- Use watchdogs to detect no-progress windows.
- Capture the outstanding ID/target table on timeout.
- Run formal liveness checks with explicit fairness assumptions.

---

## 11. Timing closure: where the critical path comes from

### 11.1 Typical combinational path

An unpipelined request path may contain:

1. Input address capture/selection
2. Address decode
3. Connectivity qualification
4. Arbitration priority logic
5. Target mux selection
6. READY propagation back to the initiator

A data path may contain:

1. Source data mux
2. Width or format logic
3. Target data mux
4. Target READY feedback
5. Response selection

The exact path depends on the implementation. In a large fabric, the ready path can be as important as the data path.

### 11.2 Register slices and skid buffers

A register slice inserts pipeline state between interfaces. A skid buffer absorbs a transfer when downstream backpressure arrives while an upstream transfer is already in flight.

A correct elastic stage must preserve:

- VALID and payload association
- All relevant sideband fields
- Transaction ordering
- No data loss or duplication
- Correct READY behavior

Do not register only VALID while leaving the payload combinational, or register the payload without preserving the handshake. The entire channel payload and its flow-control state must move together.

### 11.3 Pipeline placement trade-offs

| Placement | Benefit | Cost |
|---|---|---|
| At ingress | Shortens long paths from initiators | Adds latency to all traffic from that ingress |
| Before target | Helps target-side timing and physical placement | May add target-side buffering |
| Between switch stages | Supports hierarchical fabrics | More route state and latency |
| On response path | Helps R/B timing closure | Delays completion and can increase outstanding occupancy |

### 11.4 Clock-domain crossings

If initiators and targets use unrelated clocks, use a proper asynchronous AXI bridge or equivalent CDC architecture. Do not directly synchronize each AXI signal independently.

The CDC block must transfer coherent channel payloads and handshake state. Reset assertion/deassertion, FIFO pointer synchronization, MTBF assumptions and clock stoppage must be reviewed.

### 11.5 Physical-design considerations

- Crossbar datapaths can create long, high-fanout routes.
- A full-connectivity matrix may be physically expensive.
- Cluster local initiators and targets where traffic is local.
- Use sparse connectivity where legal.
- Place pipeline cuts near physical boundaries.
- Check that response paths are not longer or more congested than request paths.
- Review congestion around large mux structures and central arbitration logic.

---

## 12. AXI infrastructure around the crossbar

A production interconnect often includes more than a switch.

| Infrastructure block | Function | Typical reason to insert it |
|---|---|---|
| Register slice | Adds pipeline registers | Timing closure |
| Data FIFO | Buffers data and absorbs bursts/backpressure | Rate mismatch or latency hiding |
| Clock converter | Bridges clock domains | Different clock frequencies/phases |
| Width converter | Upsizes or downsizes data paths | Different endpoint datapath widths |
| Protocol converter | Converts AXI3/AXI4/AXI4-Lite or another supported form | Endpoint compatibility |
| ID remapper | Changes ID width/namespace | Response routing and limited downstream ID width |
| Address remapper/MMU | Translates or restricts address ranges | Address-space adaptation and access control |
| Error responder | Generates error responses | Unmapped or unsupported accesses |

AMD PG059 explicitly documents a crossbar plus optional register slices, data FIFOs, clock converters, width converters, protocol converters and AXI MMU. Its width-conversion documentation explains that upsizing/downsizing may pack, serialize or merge data beats and may require internal transaction tracking. These behaviors are specific to that IP implementation.

### 12.1 Do not confuse an interconnect with a firewall

Address decode answers: **Which target does this address select?**

A firewall answers: **Is this initiator allowed to access that target/range with these security attributes?**

Some products integrate both functions. If separate, ensure the security decision is made before an unauthorized request can reach the target, and preserve the security attributes across every bridge.

### 12.2 Static topology versus runtime configuration

Two common approaches:

- **Elaboration-time topology:** connectivity and address map are fixed in generated RTL.
- **Programmable topology:** selected routes, QoS or address windows can be changed through configuration registers.

A programmable topology requires reset defaults, secure programming access, lock behavior, safe reconfiguration sequencing and rules for outstanding transactions. Never change a route table while traffic is in flight unless the architecture explicitly supports it.

---

## 13. Configuration and register model

There is no universal AXI-crossbar register map defined by the AXI protocol. Some fabrics are mostly static hardware generated from configuration; others expose programmable control and debug registers.

A real interconnect TRM must define:

- Register base address and access permissions
- Reset values
- Address window configuration
- Connectivity enable/disable
- QoS override or arbitration weights
- Outstanding/acceptance limits
- Clock and power controls
- Error and timeout status
- Performance counter control and snapshot semantics

### Illustrative debug register proposal

The following offsets are a teaching example only. They are not an Arm or AMD register map.

| Offset | Register | Example meaning |
|---:|---|---|
| 0x000 | FABRIC_ID | Implementation/version identifier |
| 0x004 | FABRIC_CTRL | Enable, bypass or test mode |
| 0x010 | DECERR_COUNT | Unmapped access count |
| 0x014 | TIMEOUT_COUNT | Timeout event count |
| 0x020 | M0_REQ_COUNT | Accepted requests from initiator 0 |
| 0x024 | M0_STALL_CYCLES | Cycles initiator 0 was backpressured |
| 0x040 | T0_BUSY_CYCLES | Target 0 busy cycles |
| 0x044 | T0_ARB_WAIT_MAX | Maximum arbitration wait |
| 0x080 | SNAPSHOT_CTRL | Freeze/snapshot counters |
| 0x084 | SNAPSHOT_STATUS | Snapshot complete/valid |

A counter's definition must be precise. For example, "request count" could mean address handshakes, completed transactions or accepted beats. Those are different metrics.

---

## 14. RTL microarchitecture: build the crossbar from smaller blocks

### 14.1 Recommended decomposition

Do not begin by writing one monolithic N-by-M RTL module. Build and verify smaller units:

1. Address decoder
2. One-hot route selector
3. Per-target read-address arbiter
4. Per-target write-address arbiter
5. Per-source write-route tracking
6. Per-target write-data scheduler
7. Read response router
8. Write response router
9. ID extender/remapper
10. Default error responder
11. Optional FIFO/register-slice wrappers
12. Top-level connectivity generator

This decomposition improves unit testing and makes failures easier to isolate.

### 14.2 Arbiter pseudocode

~~~text
for each target:
    eligible = requesting_initiators
               AND connectivity_allowed
               AND acceptance_capacity_available
               AND target_issuing_capacity_available

    winner = arbitration_policy(eligible, priority, qos, age)

    if downstream_ready:
        accept winner's address
        update arbitration state
        record source/ID/target metadata
    else:
        retain the pending winner and payload
~~~

A registered arbiter should retain a valid grant while the downstream channel is stalled. The source must keep VALID and payload stable until the handshake.

### 14.3 Example SVA checks

These are illustrative properties. Adapt reset polarity, signal names and assumptions to the actual RTL.

~~~systemverilog
// A selected output must not be granted to two sources at once.
assert property (@(posedge clk) disable iff (!rst_n)
  $onehot0(target_aw_grant));

// A stalled address channel holds VALID and all address metadata.
assert property (@(posedge clk) disable iff (!rst_n)
  m_awvalid && !m_awready
  |=> m_awvalid &&
      $stable({m_awaddr, m_awid, m_awlen, m_awsize,
               m_awburst, m_awprot, m_awcache, m_awqos}));

// A stalled write-data channel holds data, strobes and LAST.
assert property (@(posedge clk) disable iff (!rst_n)
  m_wvalid && !m_wready
  |=> m_wvalid && $stable({m_wdata, m_wstrb, m_wlast}));

// A stalled read response holds its payload.
assert property (@(posedge clk) disable iff (!rst_n)
  s_rvalid && !s_rready
  |=> s_rvalid && $stable({s_rid, s_rdata, s_rresp, s_rlast}));
~~~

Additional properties should check:
- One-hot-or-zero target decode
- No response for an unallocated internal ID
- ID-map allocation and free consistency
- AW route FIFO push/pop balance
- W source remains associated through accepted WLAST
- R response source selection is stable under backpressure
- No queue overflow/underflow
- No transaction is duplicated or lost
- All response metadata is restored correctly

### 14.4 Formal properties and assumptions

Safety properties can often be proven without assuming that a target eventually responds:

- A request is never routed to two targets.
- A response is never routed to two initiators.
- An accepted request is not silently discarded.
- IDs are not aliased while active.
- Payload remains stable during a stall.

Liveness properties need assumptions. For example, "every accepted read eventually completes" requires an assumption that the selected target eventually returns a response and the initiator eventually accepts it. Without fairness assumptions, no interconnect can guarantee progress against an endpoint that holds READY low forever.

---

## 15. Verification strategy

### 15.1 Directed tests

1. One initiator to one target.
2. Every initiator to every permitted target.
3. Every prohibited connectivity path.
4. Two initiators contend for one target.
5. Two initiators access two independent targets simultaneously.
6. Read and write traffic to the same target concurrently.
7. AW accepted before W data.
8. WVALID asserted before AW is routed.
9. Long bursts with W backpressure.
10. Multiple outstanding reads with same and different IDs.
11. Two initiators use the same local ID.
12. Out-of-order responses across different IDs.
13. Unmapped address and DECERR.
14. Overlapping address-map configuration rejection.
15. Target acceptance limit reached.
16. Initiator acceptance limit reached.
17. BREADY and RREADY held low.
18. Target clock slower than initiator clock.
19. Reset during idle and during outstanding traffic.
20. QoS contention between real-time and best-effort traffic.
21. Response FIFO full and ID-table full.
22. Default error target under sustained traffic.

### 15.2 Scoreboard architecture

A useful fabric scoreboard should:

- Observe every ingress AW/AR handshake.
- Calculate the expected target from the reference address map.
- Check that the transaction appears on exactly the expected egress.
- Track source, ID, burst and ordering metadata.
- Observe every egress response.
- Restore the expected source and original ID.
- Compare read data and response status.
- Check W routing against accepted AW order.
- Model per-target latency and legal response reordering.
- Verify no duplication, loss or illegal reordering.

Do not write a scoreboard that assumes all responses return globally in request order. AXI permits concurrency and protocol-controlled reordering. The scoreboard must model the actual ordering contract.

### 15.3 Functional coverage

Cross these dimensions:

- Initiator × target × read/write
- Connectivity allowed/denied
- Target contention count
- Arbitration policy × priority/QoS class
- Burst length × transfer size
- Outstanding depth
- Local ID collision across initiators
- ID-map occupancy
- AW-to-W delay
- W burst length and WLAST timing
- R/B backpressure duration
- Target response latency
- Error type
- Clock ratio
- Reset with outstanding traffic

### 15.4 CDC/RDC and power verification

- Run CDC on every asynchronous boundary.
- Run reset-domain crossing analysis on independently resettable initiators/targets.
- Test clock stop/restart with no traffic and with quiesce asserted.
- Verify reset releases synchronously in each destination domain.
- Verify isolation before a power domain is switched off.
- Verify outstanding transactions are drained, aborted or retained according to the architecture.
- Check that no stale response appears after reset recovery.

---

## 16. Post-silicon debug playbook

### Symptom A — One initiator hangs, but other traffic continues

Capture:
- Initiator AW/AR VALID and READY
- Target selected by decode
- Per-target arbiter request/grant
- Acceptance/issuing counters
- ID map entries
- Queue full/empty flags
- Clock and reset status

Likely causes:
- One target's issuing limit is exhausted.
- The initiator's acceptance queue is full.
- The arbiter is stuck on a stale grant.
- An ID mapping entry was never freed.
- The target is not returning responses.

### Symptom B — Data corruption only under multi-master traffic

Check:
- AW acceptance order at each target
- W source selection and WLAST
- Write-route FIFO contents
- Extended IDs on the target interface
- BID/RID on return
- Response mux selection
- Width-converter packing and byte strobes
- CDC FIFO overflow/underflow

A single-master test may never expose source-ID collisions or write-route arbitration errors.

### Symptom C — DDR bandwidth is much lower than expected

Measure:
- AR/AW accepted transactions per initiator
- Outstanding read/write depth
- Target READY-low cycles
- Average and maximum arbitration wait
- Burst length distribution
- DDR busy cycles and refresh effects
- QoS values at ingress and egress
- Register-slice/FIFO occupancy

Separate the loss into:
1. Initiator under-issuing
2. Interconnect arbitration/buffering
3. Target acceptance limitation
4. Memory-controller scheduling
5. Physical DRAM bandwidth limitation

### Symptom D — Deadlock after a stress test

Freeze or snapshot:
- Outstanding transaction table
- Source/target/ID for every request
- Per-target response queues
- AW route FIFO heads
- Current W owner and WLAST state
- R/B response arbitration state
- All READY/VALID signals
- Queue occupancy and full flags

Draw a wait-for graph. A cycle in the graph is more informative than a large waveform with no ownership information.

### Symptom E — Timing fails after adding more initiators

Inspect:
- Address decode fanout
- Arbitration compare tree
- Grant mux
- READY feedback path
- ID extension/remap logic
- Crossbar data mux width
- Physical wire delay and congestion

Possible fixes:
- Sparse connectivity
- Hierarchical topology
- Registered decode/arbitration stages
- Register slices
- Local buffering
- Target clustering
- Width conversion near endpoints

Do not insert a pipeline stage without rechecking latency, outstanding depth and reset behavior.

---

## 17. Performance instrumentation: counters worth designing before tape-out

| Metric | Definition | Why it matters |
|---|---|---|
| Accepted AW/AR | Address handshakes per port | Measures request generation |
| Completed B/R | Response completion count | Measures end-to-end completion |
| Data beats | Accepted W/R beats | Measures payload throughput |
| READY stall cycles | VALID high while READY low | Identifies backpressure |
| Arbitration wait | Cycles from eligible request to grant | Measures contention |
| Outstanding high-water | Maximum active requests | Shows queue utilization |
| FIFO high-water | Maximum occupancy | Helps tune depth |
| Target busy cycles | Cycles target accepts or services traffic | Measures target utilization |
| DECERR/SLVERR count | Error response totals | Finds map or target failures |
| Timeout count | Requests exceeding a defined timer | Detects hangs |
| QoS class service | Completed transactions/bytes per class | Validates policy |

### Example utilization calculation

If a 128-bit target interface runs at 500 MHz, peak one-beat-per-cycle bandwidth is:

\[
16\ bytes \times 500\ MHz = 8\ GB/s
\]

If 300 million data beats are accepted over 1 second:

\[
300M \times 16 = 4.8\ GB
\]

Average bandwidth:

\[
4.8\ GB/s
\]

Utilization against the 8 GB/s peak:

\[
4.8/8 = 60\%
\]

This metric alone does not tell you why utilization is 60%. Correlate it with source starvation, READY stalls, target busy cycles and arbitration wait.

---

## 18. Hands-on architecture exercise

### Design a 3-initiator, 4-target fabric

Requirements:

- CPU: 64-bit AXI, 4-bit local ID
- DMA: 128-bit AXI, 4-bit local ID
- DSP: 128-bit AXI, 3-bit local ID
- SRAM: 128-bit, 600 MHz
- DDR controller: 256-bit, 800 MHz
- APB bridge: 32-bit, 100 MHz
- Crypto engine: 128-bit, 400 MHz
- All interfaces are initially in separate clock domains except the CPU and SRAM.

Tasks:

1. Draw the topology and connectivity matrix.
2. Define address windows for all four targets.
3. Decide which paths need width converters.
4. Decide which paths need CDC bridges.
5. Calculate the minimum fabric ID width if source-prefix encoding is used.
6. Define read and write acceptance limits for each initiator.
7. Define per-target arbitration policy.
8. Describe how AW and W are associated.
9. Define the DECERR path.
10. Define security restrictions for crypto.
11. Estimate DDR in-flight data for a measured 100 ns round-trip latency and a 128-byte burst.
12. Define counters needed to characterize audio/DSP traffic under CPU and DMA contention.

### Reference solution outline

- Source-prefix ID width: 2 source bits for three initiators plus 4 local ID bits = 6 bits. The DSP's unused local-ID bit is zero/unused by that source, but the shared downstream ID field remains wide enough for the largest source namespace.
- CPU-to-SRAM can be direct if clock and width match.
- DMA/DSP to SRAM need width adaptation only if the selected fabric width differs from endpoint width.
- Every asynchronous clock boundary requires a proper CDC bridge; do not use ordinary register slices as CDC.
- APB bridge is a protocol conversion boundary, not merely a width converter.
- Crypto route must be checked by a security policy using initiator identity and security attributes.
- DDR burst in-flight bytes = target bandwidth × 100 ns. Use the actual target bandwidth assumption, not just the bus peak.
- Measure arbitration wait, request-to-first-data latency, total response latency, accepted bytes and maximum outstanding count.

There is no unique correct topology. The best answer is the one that meets bandwidth, latency, security, area, timing and verification constraints with a clearly documented interface contract.

---

## 19. Interview questions with concise answers

**Q1. Why does a crossbar need per-target arbitration?**  
Because multiple initiators can request the same target simultaneously. Each target has finite acceptance capacity.

**Q2. Can two initiators transfer simultaneously through one crossbar?**  
Yes, if they use independent paths to different targets and the implementation has no shared bottleneck on those paths.

**Q3. Does a full crossbar guarantee non-blocking operation?**  
Only in the architectural sense for conflict-free paths, and only if the implementation's internal resources do not introduce blocking. Contending for one target still requires arbitration.

**Q4. Why are AW and W handled separately?**  
They are independent AXI channels. Address acceptance and write-data availability can occur at different times.

**Q5. Why is W routing harder than AW routing in AXI4?**  
AXI4 has no WID. The fabric must associate W data with write addresses by preserving the defined transaction order and tracking the route.

**Q6. Why extend IDs?**  
Different initiators can use identical local IDs. Extending or remapping IDs allows responses to be routed to the correct source.

**Q7. What is the difference between acceptance and issuing limits?**  
Acceptance limits constrain what the fabric accepts from an initiator; issuing limits constrain what it allows toward a target.

**Q8. Can round-robin guarantee a real-time deadline?**  
No. It can provide a fairness property under assumptions, but deadlines require bounded target latency, bounded burst behavior and system-level analysis.

**Q9. Why can a crossbar deadlock even when every endpoint obeys AXI?**  
Because legal response reordering and same-ID ordering constraints can combine with finite interconnect resources to create cyclic dependencies.

**Q10. What is the first signal group to inspect for a stalled request?**  
The relevant channel's VALID/READY, followed by route selection, arbiter grant, queue occupancy and target clock/reset state.

**Q11. Why insert a register slice?**  
To break a long combinational path and improve timing closure. It adds latency and must preserve the complete channel handshake and payload.

**Q12. When should a crossbar become a NoC?**  
When scale, physical distance, traffic isolation, topology and routing complexity make a centralized or hierarchical crossbar unattractive. There is no universal port-count threshold.

---

## 20. Exercise answer key

### Exercise 1 — Independent paths

CPU reads SRAM while DMA writes DDR. If the paths are independent and both targets can accept the traffic, the crossbar can progress both concurrently. A shared downstream resource or clock bridge may still limit concurrency.

### Exercise 2 — Same target contention

CPU and DSP both request DDR. The DDR target's AR/AW arbitration chooses eligible requesters. The loser waits or is buffered. Aggregate throughput remains bounded by the DDR controller's service capacity.

### Exercise 3 — ID width

Three source ports require:

\[
\lceil \log_2(3) \rceil = 2\ source\ bits
\]

The widest local ID is 4 bits, so the source-prefix design needs:

\[
2 + 4 = 6\ fabric\ ID\ bits
\]

### Exercise 4 — Address map overlap

If two target windows match the same address, decode is ambiguous unless the design explicitly defines an alias/priority rule. The normal solution is to reject the map at generation or elaboration time.

### Exercise 5 — W route

If the target's AW order is DMA then CPU, the target's W stream must carry the DMA burst first and then the CPU burst for that target. Do not select whichever WVALID is currently asserted without consulting the recorded AW-to-W association.

### Exercise 6 — Deadlock

Construct two initiators issuing same-ID reads to two targets in opposite order. Allow the targets to return responses in the order described in Section 10. Capture the per-ID ordering queues and show the cyclic wait.

### Exercise 7 — Instrumentation

At minimum capture accepted AW/AR, completed B/R, data beats, READY stalls, arbitration wait, target busy cycles, outstanding high-water, FIFO high-water and errors.

---

## 21. Reference material

Use the exact specification issue and IP revision selected by the project. The sources below are public reading entry points.

1. **Arm — AMBA AXI Protocol Specification, IHI 0022.** Normative AXI endpoint behavior and signal semantics: https://developer.arm.com/documentation/ihi0022/latest
2. **Arm — CoreLink NIC-400 Network Interconnect TRM, DDI 0475C.** Public product documentation; first release 7 August 2012, with revisions through 9 December 2013: https://documentation-service.arm.com/static/5e8f1ac06db15673561aab14
3. **AMD — AXI Interconnect Product Guide PG059, version 2.1.** Product release date 19 December 2025: https://docs.amd.com/r/en-US/pg059-axi-interconnect/Product-Specification
4. **AMD — AXI Crossbar Core Parameters.** Example configuration of SI/MI counts, ID width, data width, connectivity and arbitration parameters: https://docs.amd.com/r/en-US/pg059-axi-interconnect/AXI-Crossbar-Core-Parameters
5. **AMD — Address Decode.** Example of target selection, unmapped DECERR handling and address-range restrictions: https://docs.amd.com/r/en-US/pg059-axi-interconnect/Address-Decode
6. **AMD — Transaction Arbitration.** Example of priority, eligibility limits and round-robin behavior: https://docs.amd.com/r/en-US/pg059-axi-interconnect/Transaction-Arbitration
7. **AMD — Cyclic Dependency Avoidance and How Deadlock Occurs.** A documented read-response deadlock example: https://docs.amd.com/r/en-US/pg059-axi-interconnect/Cyclic-Dependency-Avoidance and https://docs.amd.com/r/en-US/pg059-axi-interconnect/How-Deadlock-Occurs
8. **AMD — AXI Infrastructure Cores.** Crossbar, width converter, clock converter, protocol converter, FIFO and register-slice roles: https://docs.amd.com/r/en-US/pg059-axi-interconnect/AXI-Infrastructure-Cores
9. **OpenTitan — Crossbar Generation Tool.** Configuration-driven generation of TL-UL crossbar RTL and DV collateral: https://opentitan.org/book/util/tlgen/index.html
10. **PULP Platform — AXI SystemVerilog IP.** Public synthesizable AXI modules and verification infrastructure: https://github.com/pulp-platform/axi

### Source interpretation

- Arm IHI 0022 is the protocol reference for AXI endpoint behavior.
- Arm NIC-400 TRM describes one licensed Arm interconnect implementation, not a universal crossbar specification.
- AMD PG059 describes one configurable AMD interconnect implementation and its documented behavior.
- OpenTitan and PULP are public implementation examples using different protocol and design choices.
- The RTL sketches and exercises in this lesson are original teaching examples, not copied vendor implementation code.

---

## 22. Final engineering takeaways

1. A crossbar creates concurrent paths for independent source-target pairs; it does not increase a target's physical bandwidth.
2. Address decode selects the destination; arbitration selects the requester when a destination is contested.
3. AW and AR arbitration are distinct resources.
4. AXI4 W-data routing must follow accepted AW ordering because there is no WID.
5. IDs must preserve enough information to route responses to the original initiator.
6. Default DECERR behavior is essential for unmapped addresses.
7. Acceptance limits, issuing limits and FIFO depths are system performance parameters.
8. QoS is effective only when every relevant stage honors a compatible policy.
9. Legal endpoint reordering can still create interconnect-level cyclic dependencies.
10. Register slices improve timing but add latency and state.
11. CDC, reset, power isolation and security must be designed into the fabric.
12. A production interconnect needs performance counters and deadlock snapshots designed before silicon.

**Next session:** Lesson 08 — Network-on-Chip (NoC): Routers, Links, Packetization, Virtual Channels, Routing, Flow Control, QoS, Topology and Coherency Transport.
