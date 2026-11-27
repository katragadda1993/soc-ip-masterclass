# Lesson 08 — Network-on-Chip (NoC): Architecture, Routers, Routing, Flow Control and Silicon Debug

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 08 of 38  
**Estimated study time:** 60 minutes, plus optional RTL/simulation exercises  
**Level:** Intermediate to advanced SoC, interconnect, RTL, verification and post-silicon engineering

> **Central idea:** A Network-on-Chip turns a collection of point-to-point wires into a structured communication fabric. Endpoints inject messages; routers move flits; flow control protects finite buffers; routing chooses paths; arbitration shares links. The NoC is not merely a larger crossbar: it is a distributed microarchitecture whose topology, protocol, buffers, clocking and traffic policy must be designed together.

---

## 1. Learning outcomes and one-hour plan

By the end of this lesson, you should be able to:

1. Explain why buses and centralized crossbars stop scaling gracefully in large SoCs.
2. Distinguish a NoC architecture from a transaction protocol such as AXI or CHI.
3. Trace a transaction from a manager through a network interface, packet, flits and multiple routers to a target.
4. Compare mesh, torus, ring, tree, fat-tree and hybrid topologies.
5. Explain routing, arbitration, store-and-forward, virtual-cut-through, wormhole flow control, credits, virtual channels and virtual networks.
6. Recognize head-of-line blocking, congestion, starvation, deadlock and livelock.
7. Estimate link bandwidth, bisection bandwidth, packet latency and credit-buffer depth.
8. Understand how a coherent fabric transports requests, responses and snoops without confusing transport with cache-coherence semantics.
9. Plan reset, clock-domain, power-domain, security, register and software integration.
10. Build a NoC validation plan spanning simulation, formal checks, emulation and silicon characterization.

Suggested timing:

| Minutes | Topic |
|---:|---|
| 0–7 | Historical backstory and why packet networks appeared |
| 7–15 | NoC system architecture, interfaces and packetization |
| 15–25 | Topology, routing and router pipeline |
| 25–35 | Flow control, credits, virtual channels and deadlock |
| 35–43 | QoS, coherency, performance calculations and worked example |
| 43–53 | Integration, RTL, verification and silicon debug |
| 53–60 | Exercises, interview questions and recap |

---

## 2. Brief recap: where NoC fits after the AXI crossbar

Lesson 07 studied a centralized AXI crossbar. A crossbar can provide multiple simultaneous transfers when they use independent input/output paths. However, the design still has to connect many initiators to many targets, distribute arbitration and route responses. As port count and physical distance grow, wiring, muxing, buffering, timing closure and congestion become increasingly important.

A NoC changes the organization:

- A crossbar creates a relatively centralized set of paths between ports.
- A NoC creates a graph of routers connected by point-to-point links.
- A transaction is adapted into one or more network messages.
- Each router makes a local forwarding decision.
- Finite buffers and flow-control rules coordinate adjacent routers.

A NoC does **not** automatically make every transaction faster. It trades centralized wiring and mux complexity for distributed routers, packet headers, buffering, hop latency, routing and flow-control complexity.

### NoC versus protocol: a critical distinction

| Concept | Defines |
|---|---|
| AXI | Manager/subordinate transaction channels, handshakes, bursts, IDs and responses |
| CHI | Coherent transaction protocol, request/response/snoop/data channels and ordering rules |
| NoC topology | Which routers and links physically connect endpoints |
| NoC routing | How a packet chooses its next hop |
| NoC flow control | How a sender knows that downstream resources can accept data |
| NoC implementation | Flit width, buffer depth, router pipeline, arbitration, link encoding and counters |

There is no single universal NoC packet format or mandatory router microarchitecture. Commercial and research fabrics make different choices. AXI or CHI may be the endpoint protocol while the internal network uses a vendor-specific packet format.

---

## 3. Historical backstory: why engineers moved from wires to networks

### 3.1 Before the NoC label: buses, crossbars and parallel-computer networks

Early SoCs commonly connected a processor, memory controller and peripherals using shared buses or hierarchical bus structures. These were attractive because they were understandable, inexpensive for a modest number of blocks and relatively straightforward to verify.

As systems grew, several pressures appeared:

- More initiators competed for shared bandwidth.
- Wide crossbars required many data muxes and long wires.
- Large centralized fabrics became difficult to place and close timing on.
- Processor clusters, accelerators and memory controllers needed more concurrent traffic.
- Global wires became a first-class physical-design problem rather than a free connection.

The research community did not invent packet switching from nothing. It adapted ideas from parallel-computer interconnection networks—topologies, packet routing, flow control and arbitration—to on-chip communication, where wire density, pin availability, area, power and short physical distances create a different set of trade-offs.

### 3.2 Milestones worth knowing

| Date | Documented milestone | Why it matters |
|---|---|---|
| 1992 | William Dally published *Virtual-Channel Flow Control* | Established the important idea of separating buffer resources into virtual channels that share a physical channel. |
| 2000 | Guerrier and Greiner published a generic architecture for on-chip packet-switched interconnections at DATE | An early documented on-chip packet-network architecture. |
| June 2001 | Dally and Towles presented *Route Packets, Not Wires* at DAC | Made a clear case for structured on-chip packet networks instead of ad-hoc global wiring. |
| January 2002 | Benini and De Micheli published *Networks on Chips: A New SoC Paradigm* in IEEE Computer | Helped establish NoC as a SoC architecture and research direction. |
| 2011–2012 | Broad NoC architecture/design-methodology reviews consolidated topology, flow control, routing and QoS techniques | Shows how the field matured beyond the initial proposal. |
| 2016 | Princeton published the OpenPiton manycore framework | A concrete open research platform with a scalable on-chip network and a taped-out 25-core implementation. |
| Current commercial systems | Coherent mesh products such as Arm CMN connect compute clusters, accelerators and memory-facing nodes using CHI | Demonstrates that NoC concepts now support large coherent systems, not only academic tiled processors. |

The 2001 Dally–Towles paper argued that structured packet networks can improve modularity and give global wires more controlled electrical characteristics. It estimated a modest network area overhead for the example studied; that estimate is a paper-specific result, not a universal NoC area rule.

The 2002 Benini–De Micheli article framed NoC as a new SoC communication paradigm. Later research examined topology, router microarchitecture, flow control, power and reliability in much greater detail.

### 3.3 Why these architectural decisions made sense

**Decision: packetize communication.**  
A packet carries destination and control information with payload. Routers can forward the packet hop by hop without requiring a dedicated global wire for every source/destination pair.

**Decision: use regular topologies such as a 2D mesh.**  
A mesh has predictable local connectivity, short neighboring links and a layout that can map naturally onto a planar die. The trade-off is that distant endpoints require multiple hops and edge/corner routers have fewer neighbors.

**Decision: use small flits and wormhole-style flow control.**  
A complete packet need not be stored at every router. Flits can advance as buffers become available. This reduces per-router storage, but a blocked packet can hold resources across several routers.

**Decision: add virtual channels.**  
Multiple logical queues can share one physical link. A packet waiting for one downstream resource need not occupy the only queue associated with that physical channel. Virtual channels can improve utilization and can be part of a deadlock-avoidance construction—but only when routing and resource dependencies are designed correctly.

### 3.4 What the backstory does not prove

The historical sources document research milestones and published architectures. They do not reveal the private decision process inside every semiconductor company. When this lesson says “engineers chose a mesh because…”, it describes a documented or widely understood engineering trade-off, not an undocumented internal story from a specific chip team.

---

## 4. NoC system architecture

A practical NoC has four major elements:

1. **Initiator/target endpoint:** CPU, DSP, GPU, NPU, ISP, DMA, memory controller, PCIe controller or peripheral subsystem.
2. **Network Interface (NI):** Converts the endpoint's transaction protocol into the network's internal message format and converts returned messages back.
3. **Router/switch:** Buffers incoming flits, computes a route, arbitrates for output resources and forwards flits.
4. **Link:** Carries flits and flow-control information between adjacent routers.

~~~mermaid
flowchart LR
  CPU["CPU / AXI manager"] --> NI0["Network interface"]
  DMA["DMA / AXI manager"] --> NI1["Network interface"]
  NI0 --> R0["Router R0"]
  NI1 --> R0
  R0 <--> R1["Router R1"]
  R1 <--> R2["Router R2"]
  R1 <--> RM["Memory-side router"]
  RM --> NIM["Memory NI"]
  NIM --> DDR["DDR controller"]
  R2 --> NIA["Accelerator NI"]
  NIA --> ACC["GPU / NPU / DSP"]
~~~

### 4.1 Network Interface responsibilities

The NI is often where protocol complexity becomes visible. It can perform:

- AXI/CHI request acceptance and response generation.
- Address-to-destination mapping.
- Packet construction and packet parsing.
- Transaction-ID preservation or remapping.
- Burst splitting and reassembly.
- Byte-enable and protection-attribute transport.
- Ordering-domain and virtual-network selection.
- Error translation.
- Clock-domain crossing between the endpoint and network.
- Flow-control adaptation between endpoint READY/VALID and network credits.

Do not assume every NI performs all of these functions. The exact behavior depends on the fabric and protocol adapter.

### 4.2 Router responsibilities

A router generally performs:

- Input buffering.
- Head-flit detection and packet-state tracking.
- Route computation.
- Virtual-channel allocation, if VCs are implemented.
- Switch arbitration.
- Crossbar traversal.
- Credit or ready/valid flow control.
- Tail-flit detection and resource release.
- Optional error detection, performance counting and power management.

A router is a local switch. It normally does not need to understand the full semantic meaning of an AXI read or a cache-coherence transaction. It needs enough header/control information to forward the packet according to the network contract.

### 4.3 Typical mesh router ports

A common 2D mesh router has five ports:

- North
- East
- South
- West
- Local endpoint

A router on an edge or corner has fewer physical neighbors. The logical router design may still expose the same port set, with unused directions disabled or tied off.

A 5-port router is an example, not a NoC requirement. Concentrated meshes, multi-port endpoint routers, hierarchical fabrics and irregular topologies use other degrees.

---

## 5. Packets, flits and message framing

### 5.1 Packet versus flit

- **Message:** A protocol-level operation, such as a cache-line read request or a write response.
- **Packet:** The unit transported through the network. A message may map to one or multiple packets.
- **Flit:** Flow-control digit; the unit stored, forwarded or credited by the router.
- **Phit:** Physical transfer unit on a link in some literature; a flit may span one or multiple physical transfers.

A common packet contains a header flit, zero or more body flits and a tail flit. A one-flit packet may use a HEAD_TAIL encoding.

~~~mermaid
flowchart LR
  H["HEAD flit<br/>destination + type"] --> B1["BODY flit"]
  B1 --> B2["BODY flit"]
  B2 --> T["TAIL flit<br/>end of packet"]
~~~

### 5.2 Generic header fields

The following table is conceptual. It is **not** a standardized NoC header layout.

| Field | Purpose |
|---|---|
| Destination node or coordinates | Identifies the target endpoint/router |
| Source node | Supports return routing, diagnostics or response matching |
| Message/opcode | Read, write, snoop, response, interrupt, message or implementation-specific operation |
| Length | Packet/flit count or payload length |
| Transaction ID | Associates a response with a request |
| Virtual network / traffic class | Separates protocol classes or services |
| Virtual-channel identifier | Selects a logical buffer/flow-control resource |
| QoS/priority | Influences arbitration or service policy |
| Protection/security attributes | Carries privilege, security or access metadata if required |
| Integrity field | Optional parity, CRC or other error-detection information |

The network designer must define field widths, legal values, packet length limits, endianness/bit ordering, reset behavior and error handling. A mismatch between the NI and router header definitions can cause a packet to reach the wrong endpoint while all links appear electrically healthy.

### 5.3 Example: AXI read transported over a packet network

A simplified sequence:

1. CPU issues an AXI read address to its NI.
2. NI captures address, ID, length, size, protection and other required attributes.
3. Address decode selects a destination memory-side node.
4. NI creates a request packet.
5. Router headers select a path through the mesh.
6. Memory-side NI reconstructs the request and presents it to the memory controller or its AXI adapter.
7. Returned data is packetized into one or more response packets.
8. Response reaches the source NI.
9. Source NI restores the AXI ID and emits R-channel beats with correct response and last-beat behavior.

The NoC must preserve the transaction properties required by the endpoint protocol. It may internally reorder independent messages, but it cannot silently violate the ordering and ID rules promised at the AXI/CHI interface.

---

## 6. Topology: how routers and endpoints are connected

Topology is the static connectivity graph. It strongly influences hop count, wire length, bisection bandwidth, router count, physical placement and fault paths.

| Topology | Strengths | Weaknesses / cautions | Common use |
|---|---|---|---|
| 2D mesh | Regular, local links, easy floorplan mapping, scalable by adding rows/columns | Manhattan distance grows with die dimensions; central links can be hot spots | Manycore and coherent mesh fabrics |
| Torus | Wrap-around links reduce diameter and increase path choices | Long wrap links, physical routing complexity and routing/deadlock constraints | Research and specialized systems |
| Ring | Simple, low router degree, straightforward wiring | Hop count and contention grow with node count; shared ring bandwidth | Small clusters and control fabrics |
| Tree | Natural hierarchy and aggregation | Upper levels can become bottlenecks or single congestion points | Hierarchical fabrics |
| Fat-tree / Clos | Multiple paths and high bisection capacity when provisioned | More switches/wires and complex routing | High-bandwidth clustered systems |
| Crossbar | Direct paths and low hop count for modest port counts | Wiring and switch area grow rapidly with port count | Small local fabrics |
| Hybrid / irregular | Can match placement and traffic locality | More complex mapping, routing, verification and physical design | Real heterogeneous SoCs |

### 6.1 Mesh coordinates and Manhattan distance

For a 2D mesh, let source coordinate be (xs, ys) and destination be (xd, yd).

For dimension-order routing, the minimum hop count is:

**H = |xd − xs| + |yd − ys|**

Example: source (0,1), destination (3,2):

- X movement = 3 hops.
- Y movement = 1 hop.
- Total = 4 router-to-router hops.

### 6.2 XY dimension-order routing

XY routing completes X movement before Y movement. The reverse convention (Y then X) is also used; the chosen convention must match the routing logic and deadlock proof.

~~~mermaid
flowchart TD
  S["Source (0,1)"] --> X1["Move East"]
  X1 --> X2["Move East"]
  X2 --> X3["Move East"]
  X3 --> Y1["Move South"]
  Y1 --> D["Destination (3,2)"]
~~~

For a conventional mesh without wrap-around links, deterministic dimension-order routing can be constructed to avoid routing-channel dependency cycles. Do not assume the same proof automatically applies to a torus, irregular mesh, adaptive routing or a fabric with additional request/response dependencies.

### 6.3 Physical topology is not logical address mapping

A memory address map may select a target node, but the target node's physical coordinates may be assigned separately. A system address map can map regions to memory controllers or home nodes; a route function then moves packets through routers.

The mapping may be:

- Fixed at elaboration time.
- Generated from a configuration database.
- Loaded into routing/address registers.
- Partially programmable for partitioning or multi-chip systems.

The configuration flow must keep address decode, endpoint IDs, routing tables and software-visible memory maps consistent.

---

## 7. Routing and arbitration are different jobs

### 7.1 Routing

Routing determines **which output direction(s) are legal** for a packet.

Examples:

- Deterministic XY/YX dimension-order routing.
- Table-based routing.
- Source routing, where the packet carries path information.
- Adaptive routing, where congestion or availability influences path choice.
- Escape-path routing, where an adaptive packet can enter a deterministic deadlock-free subnetwork.

Routing must ensure that packets make progress toward a valid destination and that the resource-dependency design is deadlock-safe.

### 7.2 Arbitration

Arbitration determines **which requester wins a shared output** when multiple input ports request it in the same cycle.

Common policies:

- Fixed priority: simple, but low-priority traffic can starve.
- Round robin: improves fairness, but does not guarantee equal latency under all traffic conditions.
- Weighted round robin: allocates relative service weights.
- Age-based arbitration: favors older requests.
- QoS-aware arbitration: considers traffic class, deadline or service level.

A packet may have a legal route but still wait for arbitration. Conversely, arbitration cannot fix an illegal route.

### 7.3 Example contention

Three input ports request the East output in the same cycle. The output can transmit only one flit per cycle if the crossbar/link has one-flit-per-cycle capacity.

The arbiter grants one request. The others remain buffered and retry. The grant policy determines fairness and latency.

---

## 8. Router microarchitecture

A common input-buffered virtual-channel router uses a pipeline similar to:

1. Buffer write (BW)
2. Route computation (RC)
3. Virtual-channel allocation (VA)
4. Switch allocation (SA)
5. Switch traversal (ST)
6. Link traversal (LT)

Some implementations merge stages, bypass buffers, pipeline the crossbar, or use different allocation structures. A three-stage wormhole router and a five-stage virtual-channel router are both valid implementation styles.

~~~mermaid
flowchart LR
  IN["Input link"] --> BUF["Input VC buffers"]
  BUF --> RC["Route compute"]
  RC --> VA["VC allocation"]
  VA --> SA["Switch allocation"]
  SA --> XBAR["Crossbar traversal"]
  XBAR --> OUT["Output link"]
  OUT -. "downstream credits" .-> BUF
~~~

### 8.1 Input buffers

An input buffer stores flits that cannot immediately traverse the router. A virtual-channel implementation has multiple logical queues per physical input.

Important implementation questions:

- Is each VC a separate SRAM/FIFO or a partition of shared storage?
- What is the depth in flits?
- Is the buffer single- or multi-pumped?
- How are simultaneous enqueue/dequeue operations handled?
- What happens on full/empty boundary conditions?
- Is head/tail metadata stored with the flit or separately?

### 8.2 Route computation (RC)

The head flit normally contains enough destination information to compute the route. The router stores the selected output port in packet/VC state so body and tail flits follow the same path.

For deterministic routing, RC can be small combinational logic. Table-based or adaptive routing can require more state, table access or congestion inputs.

### 8.3 Virtual-channel allocation (VA)

If the packet uses virtual channels, the head flit must acquire a free downstream VC. The allocator must avoid assigning a VC already owned by another packet and must release it at the correct packet boundary.

A classic bug is freeing the VC on a local tail-flit observation before the tail has actually left the resource whose ownership is being released.

### 8.4 Switch allocation (SA)

The allocator chooses one or more legal input-to-output transfers. Separable allocators commonly use input-side and output-side arbitration. A grant must be checked against:

- A valid flit at the input VC.
- A legal route.
- A free/available downstream VC for a head flit.
- A downstream credit for a body/tail flit.
- No conflicting use of the same input or output crossbar resource.

### 8.5 Switch traversal (ST)

The selected flit crosses the router's internal crossbar and is driven onto the output link. A 5-port, 128-bit router has a conceptual 5×5 data crossbar, although actual implementations may prune unused paths or use different internal structures.

### 8.6 Link traversal (LT)

The flit propagates to the next router. Long physical links may need repeaters, pipeline registers or retiming. A “one-hop” logical link can therefore have more than one cycle of physical latency.

---

## 9. Flow control: protecting finite buffers

A sender must not transmit into a receiver buffer that has no space. NoC flow control defines how this constraint is enforced.

### 9.1 Store-and-forward

The router receives the complete packet before forwarding it.

- Simple packet ownership model.
- Requires enough storage for a whole packet at each hop.
- Adds packet-level forwarding latency.
- Useful in some buffered packet networks, but can be expensive for long packets.

### 9.2 Virtual cut-through

The router can begin forwarding once the downstream router has enough capacity for the packet or can otherwise guarantee acceptance.

- Can reduce latency compared with store-and-forward.
- Still relies on packet-level buffer/resource reservation.
- Can require substantial downstream storage.

### 9.3 Wormhole flow control

The packet is split into flits. Flits can advance as soon as the next buffer has room.

- Small per-router buffers are possible.
- A packet can occupy resources across multiple routers.
- A blocked head flit can stop the body/tail behind it.
- Resource dependencies can create deadlock if the routing/VC construction is not safe.

### 9.4 Ready/valid versus credit-based flow control

| Scheme | Sender rule | Receiver/resource meaning |
|---|---|---|
| Ready/valid | Transfer occurs when VALID and READY are both asserted at the sampling edge | READY indicates acceptance capacity under that interface's contract |
| Credit-based | Sender transmits only when it has a credit for the downstream VC/buffer | A credit represents one available buffer slot or defined storage unit |

These are not interchangeable without an adapter. Credit-based flow control is common in NoCs because it can account for a finite buffer at the far end of a link, including link latency.

### 9.5 Credit example

Suppose a downstream virtual-channel FIFO has depth 4 flits.

- At initialization, upstream receives 4 credits.
- Each transmitted flit consumes one credit.
- When downstream removes a flit from the credited buffer, it returns one credit.
- Upstream may send again when the credit is available.

If the upstream sends 4 flits before any credit returns, it must stop. It must never allow the counter to wrap or become negative.

A credit is not simply a pulse meaning “I am ready now.” It is accounting for a specific buffer resource.

### 9.6 Credit conservation invariant

For a fixed downstream VC, a useful conceptual invariant is:

**Available credits + flits currently occupying the credited buffer = configured buffer capacity**

The exact accounting point depends on whether the implementation includes in-flight link flits in the credit model. Define this boundary precisely. An off-by-one error at initialization or credit return can cause overflow, underutilization or a permanent stall.

---

## 10. Virtual channels versus virtual networks

These terms are often confused.

### 10.1 Virtual channel (VC)

A VC is a logical buffer/flow-control resource sharing a physical channel. Each VC has independent queue state and usually independent credit state. VCs allow multiple packets to share a physical link without sharing a single queue.

### 10.2 Virtual network (VN)

A VN is a logical traffic network or protocol class. It may have a distinct set of VCs, buffers, routing constraints or physical resources.

Example conceptual separation:

- VN0: requests.
- VN1: responses.
- VN2: snoops.
- VN3: data.

This is an illustrative partition only. Real coherent fabrics define their own channel and VN architecture.

### 10.3 Why VNs matter for protocol deadlock

Consider a requester that holds a request resource while waiting for a response. If the response needs the same blocked resources, a cycle can arise. Separating request and response traffic into independently progressing resources can break that dependency.

But separate VNs are not a universal magic switch. The complete protocol dependency graph must be analyzed, including:

- Request-to-response dependencies.
- Snoop-to-data dependencies.
- Retry/credit dependencies.
- Endpoint queue dependencies.
- Bridge and home-node resource dependencies.

---

## 11. Deadlock, livelock, starvation and head-of-line blocking

### 11.1 Deadlock

A set of packets/resources forms a circular wait. Every packet waits for a resource held by another packet in the cycle, so no packet can advance.

A standard analysis uses a **channel dependency graph (CDG)**:

- Vertices represent channels or channel/VC resources.
- A directed edge A → B means a packet can hold A while requesting B.
- A cycle in the dependency graph indicates a potential deadlock under the model's assumptions.

A cycle is a warning requiring analysis; whether it is reachable depends on the routing and allocation rules. Deadlock freedom is established by proving an acyclic resource dependency or by constructing an escape mechanism with a valid proof.

### 11.2 Livelock

Packets continue moving or changing paths but do not reach their destination. Adaptive routing needs a progress condition, not merely legal next hops.

### 11.3 Starvation

A packet waits indefinitely because arbitration continually favors other requests. Round-robin or age-based fairness can help, but the full system must include backpressure and traffic-class priority.

### 11.4 Head-of-line (HoL) blocking

In a single FIFO, the packet at the head may be blocked for one output while a packet behind it could use a different output. The blocked head prevents the second packet from advancing.

Virtual channels or virtual-output queues can reduce this interference. They consume additional buffer and allocator resources.

### 11.5 XY routing and torus warning

For a conventional 2D mesh, deterministic dimension-order routing can be proven deadlock-free when the allowed turns and resource model match the proof. A torus introduces wrap-around channel cycles; the same simple proof does not automatically hold. Additional virtual channels, routing restrictions or escape paths may be required.

---

## 12. Quality of Service (QoS)

A NoC may carry traffic with very different requirements:

- CPU cache misses: latency sensitive.
- Display/camera: sustained bandwidth and bounded service gaps.
- Audio: modest bandwidth but strict periodic deadlines.
- DMA: high throughput, often bursty.
- Debug/management: low bandwidth but must remain serviceable.
- Best-effort background traffic: should use spare capacity without starving critical traffic.

Possible QoS mechanisms:

1. Separate VNs/VCs for traffic classes.
2. Weighted or priority arbitration.
3. Token-bucket/rate shaping at injection.
4. Time-division multiplexing or reserved slots.
5. Admission control for guaranteed bandwidth.
6. Per-flow virtual circuits or resource reservation.
7. Congestion-aware routing, when supported.
8. Per-class buffering and counters.

**Important:** A priority bit alone does not guarantee a latency bound. To claim a deadline, the design needs assumptions about offered load, topology, arbitration, buffer capacity, path length, clocking and fault behavior. Admission control and worst-case analysis may be necessary.

---

## 13. Coherent NoC: transport is not coherence

A coherent fabric transports messages required by a cache-coherence protocol. The network itself does not automatically make caches coherent.

A typical coherent system may contain:

- Request nodes (CPU clusters, GPUs, coherent accelerators).
- Home/directory nodes that serialize or track coherence state.
- Subordinate/slave nodes such as memory controllers.
- Snoop-capable agents.
- System-level cache.
- Request, response, snoop and data message paths.

Arm's AMBA CHI is a coherent transaction protocol. Arm CMN products provide mesh interconnect implementations that support CHI. The protocol defines transaction semantics; the mesh defines how the messages physically travel.

~~~mermaid
flowchart LR
  RN0["CPU cluster<br/>Request Node"] --> MESH["Coherent mesh / routers"]
  RN1["GPU / accelerator<br/>Request Node"] --> MESH
  MESH --> HN["Home / directory node"]
  HN --> SLC["System-level cache"]
  HN --> MEM["Memory-side node"]
  HN -. "snoop messages" .-> RN0
  HN -. "snoop messages" .-> RN1
~~~

### 13.1 Coherence-related integration questions

- Which nodes are coherent requesters?
- Which node is the home for each physical-address region?
- Where is the system address map programmed?
- Are snoops multicast, unicast or directory-directed?
- Are data and control messages separated?
- Which transactions can be retried?
- What ordering is guaranteed between request, response and data?
- How are non-coherent DMA agents made coherent—software cache maintenance, I/O-coherent ports or explicit cache operations?
- How are security and access permissions propagated?

### 13.2 Ordering

The network can have multiple paths and independent queues. A fabric must preserve the ordering guarantees promised by its protocol and configuration. Do not infer global ordering from packet arrival order on one link.

For AXI-facing NIs, consider IDs, ordering domains, response association and whether the NI can reorder transactions internally. For coherent protocols, follow the CHI specification and the configured home-node/ordering rules.

---

## 14. Performance engineering and calculations

NoC performance is more than peak link bandwidth. Measure injection/ejection rate, hop count, router service rate, link utilization, buffer occupancy, tail latency, bisection bandwidth and congestion distribution.

### 14.1 Raw link bandwidth

For link width W bits and link clock F Hz:

**Raw one-direction bandwidth = (W / 8) × F bytes/s**

Example: 128-bit link at 1 GHz:

- Width = 16 bytes per flit.
- Frequency = 1,000,000,000 flits/s at one flit per cycle.
- Raw one-direction bandwidth = 16 GB/s.

This assumes one flit can be transferred every cycle. It excludes packet headers, idle cycles, retries, credit stalls and protocol overhead.

If sustained utilization is 75%, approximate payload bandwidth is:

**16 GB/s × 0.75 = 12 GB/s**

If 1 of every 8 flits is header/control overhead, useful payload bandwidth is lower still. State whether utilization refers to flits, payload bytes or link cycles.

### 14.2 4×4 mesh bisection example

A 4×4 mesh has four links crossing a vertical bisection cut. With 128-bit, 1 GHz full-duplex links:

- One link direction: 16 GB/s.
- Four parallel links in one direction: 64 GB/s raw bisection bandwidth.
- Reverse direction has its own full-duplex capacity.

This is a cut capacity, not a guarantee that any traffic pattern can achieve 64 GB/s. Endpoint injection, destination ejection, router arbitration, memory controllers and traffic distribution can bottleneck first.

### 14.3 Idealized packet latency

Assume:

- 4 router-to-router hops.
- Router pipeline = 2 cycles per hop.
- Link traversal = 1 cycle per hop.
- Packet length = 16 flits.
- One flit injected per cycle.
- No contention, no credit stall, no extra NI latency.

Head-flit traversal estimate = 4 × (2 + 1) = 12 cycles.

Tail serialization after head = 15 cycles.

Approximate tail arrival = 27 cycles after the first flit begins injection, plus any endpoint NI/ejection latency not included.

At 1 GHz, 27 cycles = 27 ns.

This is a simplified timing model, not a protocol guarantee. Real pipelines may overlap stages and links differently; always count the exact cycle boundaries in the RTL or simulator.

### 14.4 Credit-buffer sizing

A first-order rule for sustaining a target rate across a credit round trip is:

**Required outstanding flit slots ≈ flits/cycle × credit round-trip cycles**

If a link transmits 1 flit/cycle and a credit takes 5 cycles to return, at least 5 credits are needed to keep the pipeline continuously supplied in the idealized model. Add margin for pipeline placement, arbitration and burstiness.

This is not a universal FIFO-depth formula. Model where the credit is consumed, where it is returned, how many flits are in flight, and whether the credit path is registered.

### 14.5 Router buffer area estimate

Illustrative router:

- 5 physical input ports.
- 2 VCs per input.
- 4 flits per VC.
- 128 bits per flit.

Storage bits = 5 × 2 × 4 × 128 = 5,120 bits.

Storage bytes = 640 bytes, excluding parity/ECC, valid bits, head/tail metadata, pointers, allocator state and physical macro overhead.

This example shows why buffer depth and VC count strongly influence router area and dynamic power.

---

## 15. Worked architecture: heterogeneous 4×4 SoC mesh

Imagine a SoC containing:

- CPU cluster.
- DSP/audio subsystem.
- GPU.
- NPU.
- ISP/camera subsystem.
- Display engine.
- Two DDR controllers.
- PCIe and Ethernet I/O.
- DMA engines and management peripherals.

A possible floorplan-oriented mapping places high-bandwidth clients near memory controllers and latency-sensitive clients near their dominant targets. This is a placement hypothesis, not a universal mapping.

### 15.1 Design questions

1. Which traffic is latency-critical versus bandwidth-critical?
2. Do CPU and GPU need coherent access to shared memory?
3. Should camera/display have reserved bandwidth or a QoS class?
4. Do audio and real-time control require bounded service gaps?
5. Are DDR controllers distributed to reduce average path length?
6. Does the PCIe DMA path need IOMMU/security checks at the NI?
7. Which traffic classes require independent VNs?
8. What is the maximum packet size?
9. What is the worst-case congestion pattern?
10. What counters are required to diagnose an underflow?

### 15.2 Example data path: camera to memory to display

1. Camera CSI receiver/ISP produces image lines.
2. ISP DMA issues writes through its NI.
3. NI maps the address to a memory-side node.
4. Routers forward write packets to the selected DDR controller.
5. Later, display DMA issues reads for the frame buffer.
6. Read packets traverse the NoC to memory.
7. Returned data packets reach the display NI.
8. Display fetch FIFO feeds the pixel pipeline.

If display fetch is starved, investigate the complete path: source injection, route, link utilization, router VC occupancy, arbitration, memory-controller queueing, response network and display FIFO threshold. A display underflow is not automatically a display-IP bug.

---

## 16. Address map, configuration registers and software interaction

Unlike APB or AXI, a NoC does not have one universal programmer's register map. Register layouts are implementation-specific.

Typical configuration/monitoring registers may include:

| Register group | Typical purpose |
|---|---|
| Node ID / endpoint ID | Identify an endpoint |
| Address-region map | Map address windows to home/memory nodes |
| Route table | Select next hop or destination path |
| VC/VN configuration | Enable traffic classes and resource mapping |
| QoS weights | Configure arbitration service policy |
| Credit/buffer status | Observe available capacity and stalls |
| Performance counters | Count flits, packets, cycles, stalls and congestion |
| Error status | Parity/CRC, timeout, malformed packet or protocol error |
| Interrupt status/mask | Report fatal or threshold conditions |
| Power/clock control | Gate idle links/routers or request wakeup |

### 16.1 Typical initialization sequence

The actual sequence is implementation-specific, but a safe conceptual order is:

1. Assert reset to the NoC and endpoint interfaces.
2. Bring up reference clocks and required power domains.
3. Release reset in the documented domain order.
4. Initialize router/endpoint IDs and address maps.
5. Initialize route tables or generated routing configuration.
6. Initialize VC/VN allocation and credit state.
7. Program QoS and traffic-class policy.
8. Clear error and performance-counter state.
9. Enable links and endpoint injection.
10. Run a low-risk memory/peripheral transaction test.
11. Enable high-bandwidth traffic and monitor counters.

Never enable injection while downstream credit state, routing state or endpoint identity is unknown.

### 16.2 Security

Address routing is not an access-control policy by itself. A secure SoC must enforce access permissions at appropriate NI, firewall, home-node or target boundaries. Security attributes must be preserved across packetization and bridges. Verify that a malformed or untrusted packet cannot choose a privileged destination by modifying its header.

---

## 17. Clock, reset, power and physical integration

### 17.1 Clock architecture

NoCs may be:

- One synchronous clock domain.
- Multiple synchronous islands connected by asynchronous FIFOs.
- Mesochronous or source-synchronous across selected links.
- Hierarchical fabrics with separate local and global clocks.

For each link, document clock source, frequency, phase relationship, reset domain, link pipeline depth and credit-return timing.

A CDC adapter must preserve packet ordering and flow-control accounting. A simple two-flop synchronizer is not a safe substitute for an asynchronous FIFO on a multi-bit flit bus.

### 17.2 Reset architecture

Reset must initialize:

- Buffer valid bits and pointers.
- VC allocation state.
- Credits.
- Packet ownership/route state.
- Link-valid state.
- Route tables or their validity.
- Error counters/status.
- NI outstanding-transaction tables.

A frequent integration failure is one router leaving reset before its neighbor has initialized credits. The first packet can be dropped, or the sender can believe buffer capacity exists when it does not.

Use a defined link initialization handshake or a system reset sequence that guarantees consistent initial state.

### 17.3 Power management

Possible techniques:

- Clock gating idle router pipelines.
- Link clock gating when no traffic is present.
- Power-gating inactive clusters or network regions.
- Retention of route/configuration state.
- Wakeup signaling.
- Dynamic voltage/frequency scaling.
- Traffic-aware power control.

Before gating a router or link, ensure no packet is in flight, no VC is allocated, no credit is outstanding and no endpoint transaction is waiting for a response. The exact quiescence protocol must be specified.

### 17.4 Physical design

NoC RTL quality is not enough. Physical implementation must consider:

- Router placement relative to endpoints.
- Link wire length and repeater insertion.
- Congestion around central routers.
- Buffer macro placement and access timing.
- Clock-tree insertion and skew.
- Crossbar fanout and mux depth.
- IR drop and switching activity.
- Thermal concentration at traffic hotspots.
- Floorplan changes that alter route length and hop latency.

A logically symmetric mesh may become physically asymmetric after placement and routing. Use extracted timing and real floorplan-aware traffic analysis.

---

## 18. RTL design checklist: what to implement and what to prove

A router RTL project is easier to review when split into blocks:

~~~text
noc_router
  ├── input_buffer[port][vc]
  ├── packet_state[port][vc]
  ├── route_compute
  ├── vc_allocator
  ├── switch_allocator
  ├── crossbar
  ├── credit_manager
  ├── link_tx_rx
  ├── error_monitor
  └── performance_counters
~~~

### 18.1 Representative interface contract

A conceptual credit-based input/output link may contain:

- Clock and reset.
- Flit data bus.
- Flit-valid.
- VC identifier.
- Credit-return indication and VC identifier.
- Optional parity/CRC or error indication.
- Optional link-up/initialization handshake.

A ready/valid implementation may instead use VALID/READY per channel. Do not combine both models casually; write down exactly what a transfer and a credit mean.

### 18.2 Example assertions to adapt

The following are property ideas, not a drop-in assertion library. Signal names and reset semantics must be adapted to the RTL.

~~~systemverilog
// A valid output flit must be associated with a legal allocated route.
assert property (@(posedge clk) disable iff (!rst_n)
  out_valid |-> route_valid);

// A credit counter must never exceed the configured VC depth.
assert property (@(posedge clk) disable iff (!rst_n)
  credit_count[vc] <= VC_DEPTH);

// Do not transmit a flit when no downstream credit exists.
assert property (@(posedge clk) disable iff (!rst_n)
  tx_fire |-> (credit_count[tx_vc] != 0));

// A stalled ready/valid output must keep its payload stable.
assert property (@(posedge clk) disable iff (!rst_n)
  out_valid && !out_ready |=> out_valid && $stable(out_flit));

// A packet tail must release packet ownership only at the defined release point.
assert property (@(posedge clk) disable iff (!rst_n)
  release_vc |-> tail_flit_departed);
~~~

These properties need careful sampling and same-cycle credit-return/send handling. A credit counter may legitimately receive and consume a credit in the same cycle. The RTL and assertions must agree on whether the counter is modeled before or after the transfer edge.

### 18.3 Verification invariants

Prove or test:

- No flit is duplicated.
- No flit is dropped.
- No packet is routed to the wrong destination.
- Packet order is preserved where required.
- Every accepted request eventually completes under stated fairness/environment assumptions.
- Credits are conserved.
- No VC is allocated to two packets simultaneously.
- No packet releases a VC before the tail leaves.
- No illegal output grant occurs.
- Arbitration meets its fairness contract.
- Reset leaves all queues and ownership state empty.
- Error injection is detected and handled as specified.
- QoS guarantees hold under a bounded traffic model.

### 18.4 Test strategy

**Directed tests**

- Single packet, one hop.
- Single packet, multiple hops.
- Head/body/tail boundaries.
- Single-flit packet.
- Every legal destination.
- Every legal turn.
- Backpressure on each output.
- Buffer full and empty boundaries.
- Simultaneous credit return and flit send.
- Reset while idle and reset during traffic, if supported.

**Constrained-random tests**

- Multiple sources to one destination.
- Uniform random traffic.
- Transpose, bit-complement and hotspot patterns.
- Long packets and mixed packet lengths.
- Mixed VNs/VCs.
- Random link stalls and credit delays.
- Clock-ratio variation for CDC links.
- Error injection and recovery.

**Formal**

- Local safety properties.
- Credit conservation.
- VC exclusivity.
- Route legality.
- Deadlock freedom for the configured routing/resource graph.
- Liveness under explicit fairness and endpoint assumptions.

---

## 19. Post-silicon bring-up and debug

### 19.1 First bring-up checklist

1. Confirm all NoC clocks and resets.
2. Read router/endpoint identity registers.
3. Check link-up and initialization status.
4. Verify initial credits per VC.
5. Read route/address-map configuration.
6. Perform one short source-to-target transaction.
7. Check destination receive count and source response count.
8. Increase to two sources and two destinations.
9. Test sustained traffic.
10. Enable QoS classes and verify service.
11. Exercise power gating and wakeup.
12. Run error-detection tests if supported.

### 19.2 Counters that matter

| Counter / trace | Diagnostic value |
|---|---|
| Flits/packets injected and ejected | Detect drops or accounting mismatch |
| Per-link utilization | Find saturated links |
| Per-input/VC occupancy high-water mark | Identify queue pressure |
| Cycles blocked for no credit | Detect downstream backpressure |
| Cycles waiting for switch grant | Detect arbitration contention |
| Average and max packet latency | Separate average performance from tail behavior |
| Per-destination traffic | Find hotspots |
| VC allocation failures | Find VC starvation or insufficient resources |
| Parity/CRC/error count | Find link/data integrity issues |
| Timeout/deadlock watchdog | Detect forward-progress failures |

### 19.3 Failure signatures

| Symptom | Likely investigation |
|---|---|
| One destination never receives packets | Address-to-node map, route table, disabled link, bad destination ID |
| All traffic stops after a few flits | Credit initialization/return, buffer full, link handshake |
| Only long packets fail | Tail detection, packet-length field, buffer boundary, credit accounting |
| Failures only with multiple sources | Arbitration, crossbar conflict, VC allocator, ordering |
| CPU traffic works but camera underflows | QoS, sustained bandwidth, hotspot, memory service gap |
| Works at low frequency, fails at high frequency | Link timing, CDC, credit path, crossbar timing, SI |
| Failure after power wakeup | Lost credits/configuration, reset order, stale packet state |
| Correct data but wrong response ID | NI transaction table or response-routing metadata |

### 19.4 Worked debug scenario: camera/display contention

**Observed:** Display underflow occurs only when camera capture and GPU traffic are active.

A disciplined investigation:

1. Reproduce with fixed resolution, frame rate and GPU workload.
2. Capture display FIFO underflow count and threshold.
3. Read per-link utilization and router VC occupancy.
4. Check whether camera writes and GPU reads share the same bisection link.
5. Compare memory-controller service latency under the same workload.
6. Check display traffic's arbitration weight and VN/VC allocation.
7. Measure maximum no-service gap, not only average bandwidth.
8. Increase display reservation or change placement only after identifying the bottleneck.
9. Re-run worst-case traffic and verify CPU/audio/camera QoS was not regressed.

**Engineering lesson:** A workload can meet average bandwidth and still violate a real-time deadline because of bursty service gaps.

---

## 20. Public reference implementations and tools

Use these to connect the architecture to actual models or RTL. They are examples, not universal NoC standards.

### 20.1 BookSim 2.0

BookSim is a cycle-level interconnection-network simulator. Its configuration separates topology, routing, flow control, traffic pattern, VC count, buffer depth and router organization. It supports credit-based virtual-channel flow control and is useful for controlled architecture experiments.

Suggested experiments:

- Mesh versus torus.
- XY versus an adaptive routing policy.
- One VC versus multiple VCs.
- Buffer depth 2, 4, 8 and 16 flits.
- Uniform versus hotspot traffic.
- Router pipeline latency and credit delay.

Repository/manual: https://github.com/booksim/booksim2

### 20.2 gem5 Garnet 2.0

Garnet is a detailed on-chip network model integrated with gem5's Ruby memory system. Its documented router includes buffer write, route computation, switch allocation and crossbar traversal; its model also represents virtual-channel credits and configurable link/router delays.

Documentation: https://www.gem5.org/documentation/general_docs/ruby/garnet-2/

### 20.3 OpenPiton

OpenPiton is a public manycore research framework published at ASPLOS 2016. The project paper documents a scalable uncore, verification infrastructure, ASIC/FPGA implementation scripts and a taped-out 25-core implementation. It is useful for seeing how a NoC fits into a real manycore research system.

Publication: https://doi.org/10.1145/2872362.2872414  
Princeton research page: https://collaborate.princeton.edu/en/publications/openpiton-an-open-source-manycore-research-framework/

### 20.4 Arm CMN

Arm CMN is a commercial coherent mesh family built around AMBA CHI. Public product documentation describes mesh sizing, crosspoints, protocol nodes, address mapping, QoS and system integration. Product feature counts and limits vary by generation and configuration.

- CMN-700: https://www.arm.com/products/silicon-ip-system/corelink-interconnect/cmn-700
- CMN S3(AE) documentation: https://support.arm.com/documentation/107858/0205/About-CMN-S3-AE-

---

## 21. Hands-on exercises

### Exercise A — Route computation

For a 5×4 mesh, calculate XY route and hop count for:

1. (0,0) → (4,3)
2. (3,2) → (1,0)
3. (4,1) → (0,3)

Write a route function and test all source/destination pairs. Confirm that every hop reduces the remaining Manhattan distance by one.

### Exercise B — Link and bisection bandwidth

A mesh has 128-bit links at 800 MHz and one flit/cycle.

1. Calculate raw one-direction link bandwidth.
2. Calculate 4-link bisection bandwidth.
3. Estimate useful payload bandwidth at 80% utilization and 90% payload efficiency.
4. Explain why a DDR controller can still become the bottleneck.

### Exercise C — Credit accounting

A VC has 6 buffer entries. Credit return latency is 4 cycles. Link rate is 1 flit/cycle.

1. Draw the credit counter across 10 cycles.
2. Include simultaneous transmit and credit return.
3. Prove that no more than 6 flits occupy the downstream buffer.
4. Add reset during an idle link and define initial-credit behavior.

### Exercise D — Router microarchitecture

Design a 5-port router with:

- 64-bit flits.
- 2 VCs per input.
- 4 flits per VC.
- Deterministic XY routing.
- Round-robin switch arbitration.
- Credit-based flow control.

Create a block diagram, RTL module interface, packet-state table and 10 assertions.

### Exercise E — Deadlock analysis

Construct a channel dependency graph for your routing design. Then add a second traffic class that uses a different resource order. Determine whether a cycle is possible. If so, propose a VN/VC separation or escape path and explain the proof obligation.

### Exercise F — QoS experiment

Use BookSim or Garnet to compare:

- Uniform random traffic.
- A single hotspot destination.
- Two high-priority streams and one best-effort stream.

Plot average latency and 95th/99th percentile latency against injection rate. Explain why average latency alone is insufficient for camera/display/audio.

---

## 22. Interview questions with concise answers

**1. What problem does a NoC solve?**  
It structures on-chip communication using routers and links so that large systems can scale more modularly than ad-hoc global wiring or a single shared bus.

**2. Is AXI a NoC?**  
No. AXI is a transaction protocol. A NoC is a network architecture that may transport AXI-derived or CHI transactions.

**3. What is the difference between a packet and a flit?**  
A packet is a network message; a flit is the unit used for flow control and buffering.

**4. Why use a 2D mesh?**  
Regular local connectivity maps well to planar floorplans and supports scalable tiled systems, though hop count and bisection limits remain.

**5. What does a router do?**  
Buffers incoming flits, computes routes, arbitrates for outputs and forwards flits while respecting downstream capacity.

**6. Routing versus arbitration?**  
Routing determines legal output directions. Arbitration chooses a winner when multiple inputs request the same output.

**7. What is wormhole flow control?**  
A packet is divided into flits that advance hop by hop, allowing small buffers but potentially holding resources across multiple routers.

**8. Why use virtual channels?**  
They provide separate logical queues on a shared physical link, reducing some head-of-line blocking and enabling deadlock-avoidance constructions.

**9. Do virtual channels automatically eliminate deadlock?**  
No. Routing, VC allocation and all resource dependencies must be proven deadlock-safe.

**10. What is a credit?**  
A token representing available capacity in a specific downstream buffer resource under the link's credit contract.

**11. What is bisection bandwidth?**  
The aggregate link capacity crossing a cut that divides the network into two parts. It helps bound traffic between those regions.

**12. How does a coherent NoC differ from a non-coherent NoC?**  
The coherent system transports protocol messages for cache ownership and data consistency. The network transports them; the coherence protocol and home/directory logic define semantics.

**13. Why can a display underflow occur when average bandwidth is sufficient?**  
Because worst-case service gaps or bursty contention can empty the display FIFO even when long-term average bandwidth is adequate.

**14. What is the first counter to inspect in a suspected NoC stall?**  
There is no universal first counter. Correlate injection/ejection counts, per-VC occupancy, no-credit stalls, switch-grant stalls and link utilization to locate the blocked resource.

**15. What must be checked after NoC power gating?**  
Quiescence, in-flight packets, VC ownership, credits, configuration retention, reset order and link reinitialization.

---

## 23. Key takeaways

- NoC is an architectural communication fabric, not one universal protocol.
- Packetization and distributed routing trade centralized crossbar complexity for hop latency, buffers, flow control and routing complexity.
- Topology, routing, arbitration and flow control must be co-designed.
- Flits are the flow-control unit; packets are the message unit.
- Virtual channels share a physical link but have separate logical buffer state.
- Credits represent downstream buffer capacity and must obey conservation.
- Deadlock freedom is a proof about resource dependencies, not a checkbox.
- Coherent NoCs transport coherence messages; CHI or another coherence protocol defines their semantics.
- QoS must be evaluated against worst-case service gaps, not only average bandwidth.
- Post-silicon debug needs visibility into injection, ejection, link use, VC occupancy, credit stalls, arbitration stalls and latency.

---

## 24. References and source discipline

The following sources support the historical timeline, architecture concepts and public implementation examples. Protocol/product details must be checked against the exact revision and configuration used in a real design.

1. William J. Dally, **“Virtual-Channel Flow Control,”** IEEE Transactions on Parallel and Distributed Systems, Vol. 3, No. 2, March 1992.  
   https://people.eecs.berkeley.edu/~kubitron/cs252/handouts/papers/dally-ieee92.pdf

2. P. Guerrier and A. Greiner, **“A Generic Architecture for On-Chip Packet-Switched Interconnections,”** DATE 2000.  
   https://doi.org/10.1109/DATE.2000.840047

3. William J. Dally and Brian Towles, **“Route Packets, Not Wires: On-Chip Interconnection Networks,”** DAC 2001, pp. 684–689.  
   https://doi.org/10.1145/378239.379048

4. Luca Benini and Giovanni De Micheli, **“Networks on Chips: A New SoC Paradigm,”** IEEE Computer, Vol. 35, No. 1, January 2002, pp. 70–78.  
   https://doi.org/10.1109/2.976921

5. Wen-Chung Tsai et al., **“Networks on Chips: Structure and Design Methodologies,”** Journal of Electrical and Computer Engineering, first published 10 November 2011; Volume 2012.  
   https://doi.org/10.1155/2012/509465

6. Natalie Enright Jerger, **“20 Years of NoCs,”** ACM SIGARCH Computer Architecture Today, 2 June 2021.  
   https://www.sigarch.org/20-years-of-nocs/

7. Princeton University, **OpenPiton: An Open Source Manycore Research Framework**, ASPLOS 2016.  
   https://doi.org/10.1145/2872362.2872414

8. gem5 documentation, **Garnet 2.0: An On-Chip Network Model for Heterogeneous SoCs**.  
   https://www.gem5.org/documentation/general_docs/ruby/garnet-2/

9. BookSim 2.0, **Interconnection Network Simulator User Guide and Source**.  
   https://github.com/booksim/booksim2

10. Arm, **Neoverse CMN-700 Coherent Mesh Network** product overview.  
    https://www.arm.com/products/silicon-ip-system/corelink-interconnect/cmn-700

11. Arm, **CMN S3(AE) Technical Reference Manual landing page**.  
    https://support.arm.com/documentation/107858/0205/About-CMN-S3-AE-

### Normative versus illustrative

- The historical publications and review papers document research results and proposed architectures; they are not protocol specifications.
- BookSim, Garnet and OpenPiton are concrete models/reference implementations. Their buffer depths, pipeline stages, VC policies and packet formats are implementation choices.
- Arm CMN documentation describes a specific commercial coherent mesh family and its supported configurations; do not generalize a CMN feature into a universal NoC rule.
- Equations in this lesson are first-order engineering estimates. Validate final throughput, latency, area, power and deadlock properties using the actual RTL, protocol specification, floorplan and traffic model.

---

**End of Lesson 08. Next: Lesson 09 — CPU Subsystem: Pipeline, Privilege, MMU/MPU, TLB, Exceptions, Debug, Boot and Bus Adapters.**
