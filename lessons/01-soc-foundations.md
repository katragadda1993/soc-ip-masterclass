# Lesson 01 — SoC Foundations: From IP Blocks to a Working Silicon System

**Track:** SoC Basics → Masterclass  
**Level:** Foundation, with silicon-integration depth  
**Estimated study time:** 60 minutes  
**Status:** Session 01  
**Prerequisites:** Basic digital logic, binary/hexadecimal numbers, CPU and memory concepts  
**Outcome:** Read a top-level SoC block diagram, trace a transaction end-to-end, explain why the major IP categories exist, and identify the integration contracts that make the chip boot and operate.

> **Scope note:** This is an architectural foundation lesson, not a replacement for normative protocol specifications. Protocol-specific signal rules, timing, corner cases and register programming sequences will be covered in dedicated lessons. Examples are illustrative unless explicitly identified as a documented reference design.

---

## 1. One-hour learning plan

| Time | Topic | What you should be able to explain |
|---:|---|---|
| 0–5 min | SoC mental model | Why a chip is a system, not just a CPU |
| 5–12 min | Historical backstory | Why integration and reusable IP created a need for standard interfaces |
| 12–22 min | SoC anatomy | Compute, memory, interconnect, I/O, control and security |
| 22–32 min | Transaction journeys | CPU fetch, peripheral access and DMA transfer |
| 32–40 min | Addressing and performance | Decode, bandwidth, latency, contention and QoS |
| 40–48 min | Clock, reset, power and boot | Why sequencing is a system-level function |
| 48–54 min | Interrupts, DMA and coherency | How asynchronous events and bulk data move |
| 54–60 min | Integration/debug exercise | Build a first-pass architecture and validation checklist |

## 2. What is a System-on-Chip?

A **System-on-Chip (SoC)** integrates many functions that might previously have been separate chips or board-level components into one silicon device. A typical SoC may contain:

- One or more CPU cores and their local subsystems.
- SRAM, ROM, cache controllers and external-memory controllers.
- Interconnect fabrics that move commands, addresses and data.
- DMA engines and accelerators.
- Peripheral controllers such as UART, SPI, I2C, I3C, USB, PCIe and Ethernet.
- Audio, display, camera and sensor interfaces.
- Clock, reset, power, interrupt and pin-multiplexing infrastructure.
- Security, debug, trace, test and lifecycle-control logic.
- Physical interfaces (PHYs), analog blocks and pad-ring circuitry.

The important idea is that **the system is defined by the contracts between blocks**. An IP block is not integrated merely because its RTL compiles. It must agree with the rest of the chip on interfaces, address ownership, clocking, reset behavior, power state, interrupts, error handling, security permissions, test access and software-visible behavior.

### 2.1 A generic SoC view

~~~mermaid
flowchart TB
  subgraph COMPUTE["Compute and acceleration"]
    CPU["CPU cluster"]
    DSP["DSP / GPU / NPU"]
    DMA["DMA engines"]
  end

  subgraph FABRIC["On-chip communication"]
    XBAR["Crossbar / NoC / bus fabric"]
    SMMU["IOMMU / protection / firewall"]
  end

  subgraph MEMORY["Memory system"]
    L1["CPU caches"]
    SRAM["On-chip SRAM"]
    ROM["Boot ROM"]
    MEMC["DDR / LPDDR controller"]
    PHY["Memory PHY"]
    DRAM["External DRAM"]
  end

  subgraph PERIPH["Peripheral and media IP"]
    APB["Low-speed register bus"]
    UART["UART / SPI / I2C / I3C"]
    PCIE["PCIe / USB / Ethernet"]
    MEDIA["Audio / Display / Camera"]
  end

  subgraph CONTROL["Platform control"]
    CLK["Clock manager"]
    RST["Reset manager"]
    PWR["Power manager"]
    IRQ["Interrupt controller"]
    BOOT["Boot ROM / security"]
  end

  CPU <--> L1
  CPU <--> XBAR
  DSP <--> XBAR
  DMA <--> XBAR
  XBAR <--> SMMU
  XBAR <--> SRAM
  XBAR <--> ROM
  XBAR <--> MEMC
  MEMC <--> PHY
  PHY <--> DRAM
  XBAR <--> APB
  APB <--> UART
  XBAR <--> PCIE
  XBAR <--> MEDIA
  CLK -. controls .-> COMPUTE
  CLK -. controls .-> FABRIC
  RST -. sequences .-> COMPUTE
  RST -. sequences .-> PERIPH
  PWR -. sequences .-> CONTROL
  IRQ -. events .-> CPU
  BOOT -. initializes .-> CONTROL
~~~

This is a conceptual diagram. Real products may merge blocks, use several fabrics, place some peripherals behind bridges, or have multiple clock and power islands.

## 3. Historical backstory: why SoC integration needed standards

### 3.1 From board-level systems to reusable IP

Early electronic systems commonly assembled processors, memories, controllers and peripheral functions as separate components. As integration density increased, designers moved more of those functions onto a single die. A 1998 IBM paper, for example, described a system-on-chip integrating DRAM, digital logic, SRAM, PLLs and video DAC functionality.

The next problem was **reuse**. If every IP supplier used a different interface, timing convention, reset behavior and deliverable format, a block could not be moved between projects cheaply. A 2002 SoC design paper described how early reuse was impeded by multiple interface protocols and a lack of standard deliverables; parameterization and standard interfaces became important parts of the methodology.

**Engineering interpretation:** Standard interfaces do not make all IP identical. They make the boundary predictable enough that teams can integrate different implementations without redesigning every connection from scratch.

### 3.2 AMBA evolution: a documented example of the problem being solved

Arm's *Introduction to AMBA AXI4* guide presents this protocol timeline:

| Year / generation | Milestone | Design motivation / significance |
|---|---|---|
| 1996 — AMBA 1 | ASB and APB | Separate system-level communication from simple peripheral register access |
| 1999 — AMBA 2 | AHB and APB 2 | Higher-performance pipelined system bus while retaining a simpler peripheral bus |
| 2003 — AMBA 3 | AXI, AHB-Lite, ATB | Higher-bandwidth and more flexible transactions; trace transport |
| 2010 — AMBA 4 | AXI4, AXI4-Lite, AXI4-Stream | Distinguish memory-mapped, lightweight control and streaming use cases |
| 2011 — AMBA 4 | ACE / ACE-Lite, APB 4 | Add cache-coherency extensions and evolve peripheral signaling |
| 2014 — AMBA 5 | CHI | A coherent interface for more scalable high-performance systems |
| 2016 — AMBA 5 | AHB5 | Extend the AHB family for newer embedded requirements |

**Why did the families diverge?**

- **APB:** simple, low-power, low-bandwidth register transactions. It is appropriate for control/status registers, not usually bulk data movement.
- **AHB:** a pipelined shared-bus style suited to embedded systems that need more throughput and multiple masters.
- **AXI:** independent read and write address/data/response channels allow more concurrency and decoupling.
- **ACE / CHI:** ordinary data transport is not enough when several cached agents must maintain a coherent view of memory.
- **NoC:** as the number of communicating agents and the physical size of the design grow, a routed, packet-based fabric can scale better than one global shared bus in suitable systems.

The first AMBA specification's 1999 architecture diagram already illustrates a high-performance system bus for the CPU, memory and DMA, with an APB bridge for UART, timer and parallel I/O. The hierarchy was a deliberate way to avoid making every low-speed peripheral load the high-performance bus.

### 3.3 NoC backstory

A notable 2002 paper by Kumar and colleagues presented a packet-switched Network-on-Chip architecture based on a two-dimensional mesh of switches and resources. It described layered communication and a methodology for mapping resources onto the network.

The broader research motivation was that many-core and heterogeneous SoCs create communication demands that can become difficult to satisfy with a single shared bus. NoC is not automatically better for every chip: a small microcontroller may be better served by a compact bus or crossbar. The topology must match the number of agents, traffic patterns, physical floorplan, latency targets, power budget and verification cost.

**Design lesson:** Protocol and topology are separate decisions. AXI defines an interface protocol; a crossbar or NoC is an interconnect implementation that connects many such interfaces.

## 4. SoC IP taxonomy: know the role of each block

| Category | Example IP | Primary responsibility | Typical integration questions |
|---|---|---|---|
| Compute | CPU, DSP, GPU, NPU | Execute instructions or algorithms | ISA, cache, debug, interrupts, reset vector |
| Memory | SRAM, ROM, cache, DDR controller | Store code and data | Address map, ECC, refresh, timing, retention |
| Interconnect | AXI/AHB/APB, TileLink, NoC | Move transactions | Decode, arbitration, ordering, CDC, QoS |
| Data movement | DMA, copy engine | Move data without CPU copying each word | Descriptor format, alignment, cache coherency |
| Control | Clock, reset, power manager | Bring blocks up/down safely | Sequencing, handshakes, isolation, wakeup |
| Event | Interrupt controller, timer, watchdog | Notify software and schedule actions | Edge/level, priority, masking, completion |
| Low-speed I/O | GPIO, UART, I2C, SPI, I3C | Control and sensor communication | Pinmux, clock divider, FIFO, interrupts |
| High-speed I/O | USB, PCIe, Ethernet | External high-throughput communication | PHY, link training, DMA, power states |
| Media | I2S/TDM, SoundWire, HDA, display, camera | Real-time media movement | Clocking, framing, buffering, bandwidth |
| Security | ROM, OTP/eFuse, AES, SHA, key manager | Establish trust and protect assets | Lifecycle, keys, access control, fault handling |
| Debug/test | JTAG, trace, MBIST, scan | Diagnose and test the device | Access policy, observability, test modes |
| Physical/analog | PLL, SerDes, DDR PHY, USB PHY | Bridge digital logic and electrical interfaces | Calibration, termination, jitter, training |

A single IP may belong to more than one category. A PCIe controller, for example, has a protocol controller, DMA-facing interfaces, interrupt logic, configuration registers and a PHY-facing interface.

## 5. Follow three transactions through the chip

### 5.1 CPU instruction fetch

1. The CPU presents a program counter (PC).
2. The instruction-side cache checks whether the requested cache line is present.
3. On a cache hit, the instruction is supplied locally.
4. On a miss, the core or cache controller creates a memory transaction.
5. The interconnect decodes the address and routes the request to SRAM, ROM or a memory controller.
6. If the target is external DRAM, the memory controller schedules commands and the PHY performs electrical signaling.
7. Returned data travels back through the fabric and fills the cache line.
8. The CPU receives the instruction and continues execution.

**Debug question:** If the CPU hangs before executing the first instruction, do not immediately blame the core. Check reset-vector mapping, ROM response, interconnect decode, memory initialization, clock availability and reset release.

### 5.2 CPU writes a peripheral register

1. Software writes a memory-mapped address.
2. The CPU emits a bus transaction.
3. The fabric decodes the address and routes it to the peripheral's register interface.
4. The peripheral captures address, write data, byte enables and control.
5. The register logic updates a control bit or command field.
6. The peripheral may start an operation and later raise an interrupt.

The peripheral's **control plane** is often register-mapped. Its **data plane** may use a FIFO, streaming interface or DMA. Do not assume that writing a control register transfers the entire payload.

### 5.3 DMA transfer

1. CPU software allocates a source buffer and destination buffer.
2. Software prepares a descriptor (source, destination, length, mode and ownership/status fields).
3. CPU makes descriptor/data visible to the DMA according to the platform's cache-coherency rules.
4. CPU programs DMA registers or a descriptor-ring doorbell.
5. DMA becomes an independent bus master.
6. DMA reads source data, buffers it as needed and writes destination data.
7. DMA reports completion or error, commonly through status and an interrupt.
8. CPU handles completion and makes the destination data safe to consume.

**Common bug:** CPU sees stale data because the DMA wrote DRAM while the CPU still has an old cache line. The remedy depends on whether the platform is coherent. A non-coherent system typically needs the correct cache clean/invalidate operations and memory barriers; a coherent system still needs correct ordering and ownership rules.

## 6. Address map: how software finds hardware

Memory-mapped I/O gives each memory or peripheral target an address range. The interconnect or address decoder uses the address to select the destination.

The following is a **toy example only**, not a real chip map:

| Region | Example base | Example size | Purpose |
|---|---:|---:|---|
| Boot ROM | 0x0000_0000 | 256 KiB | Immutable startup code |
| SRAM | 0x1000_0000 | 512 KiB | Stack, heap, runtime data |
| UART0 | 0x4000_0000 | 4 KiB | Serial control/status |
| Timer | 0x4001_0000 | 4 KiB | Counter and compare registers |
| DMA0 | 0x4002_0000 | 4 KiB | DMA control and status |
| External DRAM | 0x8000_0000 | 1 GiB | Main memory |

For every region, ask:

- Is the base address aligned to the region size or the decoder's granularity?
- Do any regions overlap?
- Is the range reachable by CPU, DMA and debug masters?
- Are read/write/execute permissions correct?
- Are secure and non-secure accesses distinguished?
- Does the interconnect return a defined error for unmapped addresses?
- Are aliases, remaps and boot-time address windows intentional?
- Does the software header match the hardware decode?

**Integration rule:** The address map is a hardware/software contract. A mismatch between RTL decode, generated headers, firmware linker scripts and documentation can make a correct IP appear broken.

## 7. Performance: calculate before guessing

### 7.1 Peak bandwidth

For a simple one-beat-per-cycle interface:

[
B_{peak} = \frac{W}{8} \times f
]

where:

- (W) is data width in bits.
- (f) is transfer clock frequency in transfers per second.
- (B_{peak}) is bytes per second.

**Example:** 128-bit data path at 500 MHz:

[
B_{peak} = 128/8 \times 500\times10^6 = 8\text{ GB/s}
]

If measured efficiency is 80%, a first-order estimate is:

[
B_{effective} \approx 8 \times 0.8 = 6.4\text{ GB/s}
]

The 80% is an illustrative assumption, not a protocol guarantee. Real efficiency depends on burst length, arbitration, read/write turnaround, target latency, refresh, outstanding transactions, clock crossings and backpressure.

A 16-beat burst on a 128-bit interface carries:

[
16 \times 16\text{ bytes} = 256\text{ bytes}
]

At one beat per 500 MHz cycle, the data phase takes 32 ns, giving the same ideal 8 GB/s data rate before overhead.

### 7.2 Latency is not bandwidth

A transaction's end-to-end latency can include:

[
T_{total}=T_{queue}+T_{arb}+T_{decode}+T_{CDC}+T_{target}+T_{return}
]

A design can have high peak bandwidth and still have poor latency for small random accesses. Conversely, a low-latency register bus may be unsuitable for bulk transfers.

### 7.3 Contention and QoS

If CPU, GPU, display, camera and DMA all access one memory controller, they compete for finite bandwidth. Integration must define:

- Arbitration policy (round-robin, fixed priority, weighted, deadline-aware, etc.).
- Starvation prevention and fairness.
- QoS or traffic-class mapping.
- Buffer depth and backpressure behavior.
- Whether real-time clients receive bounded service.
- Whether one stalled target can block unrelated traffic.

**Silicon symptom:** Audio underruns or display underflows may occur even when average bandwidth appears sufficient. The problem may be worst-case service latency, burstiness or arbitration—not raw average bandwidth.

## 8. Clock, reset and power are first-class IP interfaces

A block's integration checklist must include more than its data bus.

### 8.1 Clock domains

A modern SoC may have separate system, peripheral, USB, audio, memory, always-on and high-speed PHY clocks. If two blocks use unrelated clocks, a clock-domain crossing (CDC) mechanism is required.

Typical CDC mechanisms include:

- Two-flop synchronizers for single-bit control signals (subject to the design's assumptions).
- Handshake synchronizers for multi-bit control transfers.
- Asynchronous FIFOs for multi-bit data streams.
- Gray-coded pointers for asynchronous FIFO read/write positions.
- Carefully constrained reset-domain crossings (RDC).

Never synchronize a multi-bit bus by independently passing each bit through two flops and assuming the word remains coherent.

### 8.2 Reset domains

Reset design must answer:

- Which reset is power-on reset (POR), warm reset, watchdog reset or software reset?
- Which blocks must remain alive during deep sleep?
- Which reset deassertions must be synchronized to a destination clock?
- What is the required order between clocks becoming stable and logic leaving reset?
- What happens to in-flight bus transactions during reset?
- Are reset values of registers and FIFOs architecturally documented?

### 8.3 Power domains

Power gating introduces isolation, retention and sequencing requirements. A power controller must coordinate the domain's clocks, reset, isolation and power-good indications. A block cannot safely be switched off while it owns an unfinished transaction unless the architecture defines how that transaction is drained, aborted or retained.

### 8.4 Real design example: OpenTitan

The OpenTitan Earl Grey design documentation describes two power domains: **Main**, which is switched off during deep sleep, and **Aon**, which remains powered. It documents separate system, I/O, USB and always-on clocks. This is a concrete example of why power, clock and reset architecture must be designed together; it is not a universal SoC template.

The OpenTitan clock-manager documentation also explains why some infrastructure clocks cannot simply be gated: if the fabric or memory clock is disabled, software may lose the path needed to turn the clock back on. Transactional blocks can use an **Idle** indication so clocks are not stopped while work is in flight.

## 9. Boot sequence: the first end-to-end system test

A generic boot sequence is:

~~~mermaid
flowchart TD
  A["Power rails become valid"] --> B["Always-on clock and POR logic start"]
  B --> C["Clock/reset managers establish safe clocks and resets"]
  C --> D["Boot ROM becomes executable"]
  D --> E["ROM checks lifecycle and boot policy"]
  E --> F["Initialize required memory / security / clocks"]
  F --> G["Authenticate or validate next-stage firmware"]
  G --> H["Transfer control to next stage"]
  H --> I["Initialize drivers and operating system / application"]
~~~

The exact order varies by architecture and product. Some devices boot from mask ROM, external flash, eMMC/UFS, a host interface or a secure download path. Some memory controllers must be initialized before external DRAM is usable.

A documented OpenTitan example releases the processor into ROM, which validates the next software stage stored in non-volatile memory before transferring control. Its boot and power-management documentation also shows that the power manager coordinates startup checks and reset/clock sequencing.

**Bring-up rule:** A boot failure is an end-to-end failure until proven otherwise. Validate power, clock, reset, boot straps, ROM mapping, memory controller readiness, interconnect response and security policy in order.

## 10. Interrupts: converting hardware events into software work

A typical interrupt journey is:

~~~mermaid
sequenceDiagram
  participant P as Peripheral
  participant G as Interrupt gateway
  participant C as Interrupt controller
  participant CPU as CPU
  participant SW as Interrupt handler
  P->>G: Event / level assertion
  G->>C: Pending source ID
  C->>CPU: IRQ notification
  CPU->>SW: Enter exception handler
  SW->>P: Read status and service cause
  SW->>P: Clear peripheral source
  SW->>C: Complete / acknowledge source
  C-->>CPU: Deassert when no eligible pending source
~~~

An interrupt controller commonly handles source IDs, pending state, enable masks, priority and target routing. Edge-triggered and level-triggered inputs have different corner cases. For example, the documented OpenTitan RV_PLIC gateway holds a pending source until it is claimed and completed; an edge arriving while the source is already pending may be lost in that implementation. This is an implementation-specific behavior to understand and verify, not a statement that all interrupt controllers behave identically.

## 11. IP integration contract: what every block must declare

| Contract | Questions |
|---|---|
| Functional | What does the block do? What are its legal operating modes? |
| Bus | Host or device? Protocol/version? Width? Outstanding depth? Error response? |
| Address | Base, size, alignment, aliases, register access policy? |
| Clock | Clock source, frequency range, gating, CDC crossings? |
| Reset | Reset type, polarity, synchrony, values, sequencing? |
| Power | Domain, isolation, retention, wakeup and idle handshake? |
| Interrupt | Number, trigger type, polarity, priority and clear semantics? |
| DMA | Master ports, address width, alignment, coherency and security? |
| Pins/PHY | Pinmux, pad voltage, timing, electrical and calibration requirements? |
| Security | Privilege, secure state, lifecycle restrictions, firewall/IOMMU path? |
| Test/debug | Scan, MBIST, JTAG, trace, error injection and observability? |
| Software | Driver model, register definitions, reset defaults, firmware dependencies? |

This contract is the start of the integration checklist and the basis for the IP's specification, verification plan and bring-up plan.

## 12. Pre-silicon verification and post-silicon debug

### 12.1 Pre-silicon checks

- Protocol assertions: legal handshakes, no dropped or duplicated transactions, response correctness.
- Address decode: mapped, unmapped, aliased and boundary addresses.
- Concurrency: multiple masters, arbitration, backpressure and starvation.
- Reset: reset during idle, reset during traffic, reset release order.
- CDC/RDC: structural analysis plus targeted functional tests.
- Power: clock gating, isolation, retention, wakeup and aborted transactions.
- Security: unauthorized access, privilege checks, debug lifecycle and DMA protection.
- Memory: ECC/parity, error injection, alignment, boundary and refresh interactions.
- Interrupt: edge/level, masking, priority, claim/complete and clear ordering.
- Performance: peak throughput, latency distribution, worst-case contention and QoS.
- Software: register reset values, programming sequences and driver/RTL agreement.

### 12.2 Post-silicon bring-up order

1. Confirm supplies, power-good, reference clocks and reset pins.
2. Read chip ID, revision and lifecycle/strap status using the approved debug path.
3. Verify boot ROM visibility and basic memory reads.
4. Check clock-manager and reset-manager status.
5. Validate a simple bus read/write to a known peripheral.
6. Verify interrupt assertion, routing, handler entry and clear/complete sequence.
7. Run SRAM/DRAM tests and ECC/error-injection tests where supported.
8. Validate DMA with known patterns, alignment boundaries and cache-maintenance rules.
9. Exercise peripheral loopbacks and high-speed link training.
10. Characterize throughput, latency, power, temperature and voltage corners.

Capture evidence: register snapshots, bus traces, clock/reset status, error logs, firmware build ID, test conditions and reproducible steps.

## 13. Practical lab — draw and budget a small SoC

Assume a small multimedia SoC with:

- 2 CPU cores.
- 1 DSP.
- 1 DMA engine.
- 512 KiB SRAM.
- External LPDDR.
- UART, SPI, I2C and GPIO.
- I2S/TDM audio interface.
- A camera input.
- One shared interconnect with a low-speed peripheral bridge.

### Exercise A — architecture

Draw the masters, slaves, fabric, bridge, memory controller and control infrastructure. Mark every clock and reset domain. Mark interrupt lines and DMA master ports.

### Exercise B — bandwidth

The DSP has a 128-bit data port at 500 MHz. Calculate the ideal one-beat-per-cycle bandwidth. If the measured fabric efficiency is 70%, estimate effective bandwidth.

**Answer:** Ideal = 8 GB/s. Estimated effective = 5.6 GB/s. This estimate assumes the 70% efficiency applies to the same traffic and that no downstream target is a tighter bottleneck.

### Exercise C — address map

Create a non-overlapping map for ROM, SRAM, LPDDR, UART, DMA and audio registers. Include base, size, access permissions and reset defaults.

### Exercise D — failure analysis

The DSP occasionally reports audio underrun when the camera is active. List at least five possible causes before changing the RTL. Consider memory arbitration, QoS, burst size, FIFO thresholds, clock-domain crossing, DMA descriptor ownership and interrupt latency.

## 14. Knowledge check

1. Why is an IP protocol different from an interconnect topology?
2. Why is APB usually a better fit for control/status registers than a bulk data path?
3. What architectural problem did AXI's independent channels help address?
4. Why might a crossbar become physically expensive as the number of masters and slaves increases?
5. What is the difference between bandwidth and latency?
6. Why can disabling an infrastructure clock cause a deadlock?
7. Why does a non-coherent DMA transfer need cache maintenance?
8. What information must be included in an IP integration contract?
9. Why should an unmapped address return a defined error?
10. A chip boots in simulation but not on silicon. Which power, clock, reset, boot and memory dependencies would you check first?

### Suggested answers

1. A protocol defines transaction/interface behavior; topology defines how endpoints are physically/logically connected.
2. APB's simpler low-bandwidth control model usually costs less area and power than a high-performance data fabric.
3. It allows address, data and response phases to be decoupled, enabling concurrency and better utilization.
4. Port count, wiring, muxing, arbitration and timing complexity can grow rapidly; a hierarchical or network fabric may scale better.
5. Bandwidth is data per unit time; latency is time for a transaction or response.
6. The system may lose the clock needed to access the register that would re-enable the clock.
7. CPU cache lines may not reflect DMA writes, or DMA may read stale memory if dirty CPU lines were not cleaned.
8. Function, bus, address, clocks, resets, power, interrupts, DMA, security, pins, test/debug and software contracts.
9. It exposes programming mistakes and prevents unintended routing or silent data corruption.
10. Start with rails/power-good, root clocks, POR/reset release, reset-vector mapping, ROM access, memory initialization and interconnect decode.

## 15. Next lesson

**Lesson 02 — On-chip interconnect fundamentals:** masters/hosts and slaves/devices, address decoding, arbitration, muxed shared buses, crossbars, bridges, ordering, outstanding transactions, backpressure, CDC boundaries and the evolution from shared bus to NoC. AXI/AHB/APB protocol-level details will then be developed in dedicated lessons.

---

## References and further reading

1. Arm, *AMBA — latest specifications and architecture overview*: https://www.arm.com/architecture/system-architectures/amba
2. Arm, *Introduction to AMBA AXI4*, document 102202, Issue 01 (2020). Includes the AMBA protocol timeline and rationale for APB/AHB/AXI families: https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf
3. Arm, *What is AMBA?* (1 December 2014), historical context for AMBA's 1996 introduction: https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/what-is-amba
4. Arm, *AMBA AXI Protocol Specification*, IHI 0022 (check the current issue on the official documentation page): https://developer.arm.com/documentation/ihi0022/latest
5. Arm, *AMBA AHB Protocol Specification*: https://developer.arm.com/documentation/ihi0033/latest
6. Arm, *AMBA APB Protocol Specification*: https://developer.arm.com/documentation/ihi0024/latest
7. Kumar et al., *A Network on Chip Architecture and Design Methodology*, ISVLSI 2002, DOI: https://doi.org/10.1109/ISVLSI.2002.1016885
8. Benini and De Micheli, *Networks on chips: A new SoC paradigm*, IEEE Computer, 2002, DOI: https://doi.org/10.1109/MC.2002.994091
9. IBM Research, *Designing a testable System on a Chip* (1998): https://research.ibm.com/publications/designing-a-testable-system-on-a-chip
10. OpenTitan, *Earl Grey chip design and memory map*: https://opentitan.org/book/hw/top_earlgrey/doc/design/index.html
11. OpenTitan, *Power manager theory of operation*: https://opentitan.org/book/hw/ip/pwrmgr/doc/theory_of_operation.html
12. OpenTitan, *Clock manager theory of operation*: https://opentitan.org/book/hw/top_darjeeling/ip_autogen/clkmgr/doc/theory_of_operation.html
13. OpenTitan, *RV_PLIC theory of operation*: https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rv_plic/doc/theory_of_operation.html

### Source discipline

- **Normative:** The relevant protocol specification is the authority for legal signal behavior, timing, transaction ordering and required features.
- **Documented implementation:** OpenTitan is used as a public, inspectable example of one real SoC architecture. Its exact clocks, domains, bus latencies and PLIC behavior are implementation-specific.
- **Historical evidence:** Dated Arm and academic publications support the evolution narrative. Where a design motive is an engineering interpretation rather than an explicit statement from the designers, it is labeled as interpretation.
- Do not reproduce proprietary specification text or vendor diagrams. Create original explanations and diagrams and link to the official source.
