# Lesson 09 — CPU Subsystem: From Instruction Execution to Silicon Integration

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 09 of 38  
**Estimated study time:** 60 minutes, plus optional RTL and software exercises  
**Level:** Intermediate to advanced CPU, SoC integration, RTL, verification and post-silicon engineering

> **Central idea:** A CPU subsystem is not just an RTL core. It is the execution engine plus the contracts that let it fetch instructions, access memory and peripherals, enforce privilege, take traps, enter debug, boot predictably and communicate with the rest of the SoC. The ISA defines what software observes; the microarchitecture defines how the core executes; the SoC integration defines how that core reaches real silicon resources.

---

## 1. Learning outcomes and one-hour plan

By the end of this lesson, you should be able to:

1. Separate ISA, microarchitecture, CPU core and CPU subsystem responsibilities.
2. Trace an instruction from reset-vector fetch through decode, execution, retirement and memory-system access.
3. Explain pipeline hazards, forwarding, stalls, branch recovery and precise exceptions.
4. Distinguish an MMU, TLB, page-table walker, PMP/MPU and physical-memory attributes.
5. Explain RISC-V privilege modes, CSRs, traps, interrupts and return-from-trap behavior.
6. Trace a virtual address through translation, protection, cache and the SoC interconnect.
7. Understand CPU bus adapters, boot interfaces, debug access and interrupt integration.
8. Build a practical CPU verification and post-silicon bring-up checklist.

### Suggested 60-minute schedule

| Time | Topic |
|---:|---|
| 0–6 min | Historical backstory and CPU subsystem boundaries |
| 6–16 min | Pipeline, execution, hazards and retirement |
| 16–28 min | Address translation, TLB, PMP and memory attributes |
| 28–38 min | Privilege, exceptions, interrupts and CSRs |
| 38–45 min | Bus integration, boot, debug and clock/reset |
| 45–53 min | Performance calculations and silicon-debug case |
| 53–60 min | Verification plan, exercises and interview questions |

---

## 2. What exactly is a CPU subsystem?

Use these terms precisely:

- **ISA (Instruction Set Architecture):** the software-visible contract: instructions, registers, exceptions, privilege behavior and memory-ordering rules.
- **Microarchitecture:** the internal implementation: pipeline stages, queues, branch predictor, register renaming, execution units, caches and scheduling.
- **CPU core:** the hardware that fetches and executes instructions for one or more hardware threads.
- **CPU subsystem:** the core or cores plus the tightly coupled structures and SoC interfaces needed to make them usable: interrupt delivery, debug, reset/boot control, bus adapters, local memories, protection, cache interfaces and sometimes an MMU.

A CPU core can be ISA-compatible while having a very different microarchitecture. Two RISC-V cores can execute the same architectural program but have different pipeline depths, cache sizes, branch predictors, TLB organizations, bus protocols and performance.

### 2.1 CPU subsystem block diagram

~~~mermaid
flowchart TB
  BOOT[Boot ROM / reset vector] --> IF[Instruction fetch]
  subgraph CPU[CPU core]
    IF --> PF[Prefetch / I-cache]
    PF --> DEC[Decode and register read]
    DEC --> EX[Execute / branch]
    EX --> LSU[Load-store unit]
    EX --> CSR[CSR / privilege / trap control]
    LSU --> WB[Writeback / retirement]
    EX --> WB
    CSR --> WB
    WB --> DEC
    BP[Branch prediction] --> IF
    TLB[ITLB / DTLB] <--> IF
    TLB <--> LSU
    PTW[Page-table walker] <--> TLB
    CSR <--> PTW
  end
  IF --> IBUS[Instruction bus adapter]
  LSU --> DBUS[Data bus adapter]
  IBUS --> FAB[AXI / AHB / TileLink / NoC]
  DBUS --> FAB
  FAB --> MEM[SRAM / cache / DDR]
  FAB --> MMIO[Peripheral register space]
  IRQ[Interrupt controller] --> CSR
  TIMER[Timer / software interrupt] --> CSR
  DBG[Debug module / JTAG DTM] <--> CSR
  RESET[Reset and boot control] --> CPU
  CLOCK[Clock / power controller] --> CPU
~~~

The diagram is conceptual. Not every core has an MMU, cache, branch predictor, separate instruction/data ports, page-table walker or external interrupt controller. Those are configuration and platform choices.

### 2.2 The four contracts

| Contract | Main question | Typical owner |
|---|---|---|
| ISA contract | What must software observe? | ISA specification |
| Core microarchitecture | How does the core execute? | CPU RTL/design team |
| Memory-system contract | How are addresses, attributes and ordering handled? | CPU + cache/MMU + fabric |
| Platform contract | How do reset, boot, interrupts, debug and peripherals connect? | SoC integration team |

A recurring integration mistake is to treat a core integration guide as if it were the ISA specification. A reference implementation describes one legal design choice; it does not make that choice mandatory for every CPU.

---

## 3. Historical backstory: why CPU architectures changed

### 3.1 From complex instruction control to RISC ideas

IBM's historical account describes a 1974 research effort led by John Cocke to build a fast telephone-exchange controller. The performance target forced the team to rethink the cost of instruction execution. The work contributed to the IBM 801 and the Reduced Instruction Set Computer approach. The documented design rationale was to avoid hardware complexity that did not pay for itself in frequently used workloads.

The Berkeley RISC project began in 1980 and published RISC-I results in 1981. It explored a simpler instruction set and a VLSI-friendly implementation. These were research projects, not a single universal design recipe. Their lasting influence was the argument that a carefully selected instruction set, compiler support, register organization and pipeline could outperform a more elaborate control implementation for important workloads.

RISC-V began at UC Berkeley in May 2010. RISC-V International describes it as an open standard ISA with a modular extension model; the RISC-V Foundation was formed in 2015. The design rationale for an open, modular ISA is different from the original RISC performance motivation: it also addresses reuse, extensibility, ecosystem collaboration and implementation freedom.

### 3.2 Why the modern CPU subsystem has so many blocks

| Historical/technology pressure | Architectural response | Cost or trade-off |
|---|---|---|
| Main memory became much slower than CPU execution | Caches, prefetch, TLBs and out-of-order memory overlap | Area, coherence complexity, timing and verification |
| Workloads grew beyond one protected program | Privilege modes, MMU and process address spaces | Page-table walks, TLB shootdowns, OS complexity |
| Branches disrupt pipelines | Branch prediction and speculative execution | Misprediction recovery and side-channel concerns |
| Multiple cores share memory | Cache coherence and interconnect ordering | Snoop traffic, directories and protocol complexity |
| Software needs field debug and manufacturing test | Debug modules, triggers and trace | Security lifecycle and observability trade-offs |
| SoCs combine many IP vendors | Standard ISA and standardized bus adapters | Integration still requires platform-specific contracts |

**Engineering interpretation:** The CPU subsystem grew because the gap between fast instruction execution and the rest of the system widened. Every optimization moves complexity somewhere else: caches move it into coherence, virtual memory moves it into translation and TLB maintenance, speculation moves it into recovery and security, and debug moves it into access-control policy.

### 3.3 Normative history versus implementation history

- IBM's RISC history and RISC-V International's history pages document the dated historical claims above.
- The RISC-V privileged specification defines architectural behavior such as privilege modes, CSRs, traps, PMP and virtual memory.
- Ibex and CVA6 documentation describe particular microarchitectures. Their pipeline depths, TLB organizations and interface details are implementation examples, not universal ISA requirements.

---

## 4. Instruction execution: from fetch to retirement

### 4.1 A representative five-stage pipeline

A classic teaching model uses:

1. IF — instruction fetch
2. ID — instruction decode and register read
3. EX — ALU, branch decision or address generation
4. MEM — data-memory access
5. WB — architectural register writeback

This is a model, not a requirement. Real cores may combine stages, add stages, use decoupled queues or execute instructions out of order.

~~~mermaid
flowchart LR
  A[IF] --> B[ID]
  B --> C[EX]
  C --> D[MEM]
  D --> E[WB]
  E --> F[Architectural state]
~~~

A scalar in-order pipeline ideally completes one instruction per cycle after filling. Its latency for an individual instruction can be several cycles even when throughput is one instruction per cycle.

### 4.2 Pipeline hazards

**Structural hazard:** two operations need the same hardware resource in the same cycle. Example: instruction fetch and load data contend for a single-ported memory.

**Data hazard:** an instruction needs a value that an older instruction has not produced yet.

~~~text
ADD x5, x1, x2
SUB x6, x5, x3
~~~

The SUB depends on x5. A bypass/forwarding path may send the ALU result directly to the consumer. If the producer is a load and the data returns later, a load-use stall may still be required.

**Control hazard:** a branch or jump changes the next PC. Instructions fetched from the fall-through path may be wrong and must be flushed.

### 4.3 Forwarding, stalling and flushing

- **Forwarding:** use a result from an internal pipeline stage rather than waiting for register-file writeback.
- **Stall:** hold one or more stages while a dependency or resource is unavailable.
- **Flush:** invalidate younger instructions after a redirect, exception, interrupt or debug entry.
- **Replay:** reissue an operation when a speculative or resource-dependent attempt cannot complete.

A correct pipeline must preserve architectural ordering even when internal operations overlap.

### 4.4 In-order versus out-of-order execution

| Feature | In-order core | Out-of-order core |
|---|---|---|
| Issue | Usually follows program order | Independent instructions may issue when operands/resources are ready |
| Register dependencies | Forwarding and stalls | Register renaming removes false dependencies |
| Completion | Often in program order | Execution may complete out of order |
| Retirement | In order | In order, commonly through a reorder buffer |
| Main complexity | Pipeline control and stalls | Rename, issue queues, ROB, load/store queue, replay and recovery |
| Typical goal | Small area, predictable control | Higher instruction-level parallelism |

A modern out-of-order core generally retires instructions in program order so that architectural state and synchronous exceptions appear precise. A faulting instruction must not leave younger instructions architecturally committed.

### 4.5 Retirement is the architectural boundary

The execute stage can calculate a result before it is safe to expose that result architecturally. Retirement/commit is the point where the implementation makes the instruction's effects permanent according to the architecture.

This distinction matters for:

- Precise exceptions.
- Interrupt boundaries.
- Debug single-step.
- Speculative loads.
- Branch misprediction recovery.
- Performance counters such as retired-instruction counters.

---

## 5. Core interfaces and transaction paths

A CPU may expose native request/grant interfaces, AXI, TileLink, AHB or proprietary internal ports. The core is not automatically an AXI master just because the SoC fabric uses AXI.

### 5.1 Instruction-fetch path

A typical decoupled instruction interface has:

- request-valid,
- request address,
- grant/accept,
- response-valid,
- returned instruction data,
- error indication.

The request must remain asserted until accepted if that is the interface contract. The response may arrive later. The frontend associates returned data with the correct fetch address, handles redirects and discards stale responses after a branch or trap.

Ibex documents an instruction prefetch buffer and fetch FIFO. Its instruction-side request/grant/response interface is a concrete example of a simple decoupled interface.

### 5.2 Load-store path

The load-store unit (LSU) translates an instruction into one or more memory transactions:

- effective address calculation,
- access size,
- read or write,
- byte enables,
- sign or zero extension,
- alignment handling,
- translation and protection checks,
- bus request and response,
- exception reporting.

A simplified native interface may contain:

| Signal | Meaning |
|---|---|
| req | Request is being presented |
| addr | Effective or physical address, depending on interface |
| we | Write enable |
| be | Byte enables |
| wdata | Write payload |
| gnt | Request accepted |
| rvalid | Response data is valid |
| rdata | Read payload |
| err | Access error |

**Important:** the exact meaning and timing of these signals are implementation-specific. Read the core interface specification before connecting a wrapper.

### 5.3 One load instruction, end to end

Suppose software executes a 32-bit load from virtual address VA:

1. Decode identifies a load and its width/sign-extension behavior.
2. The register file supplies the base operand.
3. The ALU computes the effective virtual address.
4. The DTLB is queried if translation is enabled.
5. On a TLB miss, a page-table walker reads page-table entries.
6. The final physical address is checked against protection and memory attributes.
7. The cache or data bus receives the request.
8. The fabric routes the transaction to cache, SRAM or DDR.
9. The response returns with data or an error.
10. The LSU formats the value and reports completion or an exception.
11. The destination register is updated at the architectural commit point.

~~~mermaid
sequenceDiagram
  participant DEC as Decode/Execute
  participant TLB as DTLB
  participant PTW as Page Walker
  participant PROT as PMP/PMA
  participant MEM as Cache/Fabric/Memory
  participant WB as Retirement
  DEC->>TLB: Lookup virtual address
  alt TLB hit
    TLB-->>DEC: Physical address + attributes
  else TLB miss
    TLB->>PTW: Request translation
    PTW->>MEM: Read page-table entries
    MEM-->>PTW: PTE data
    PTW-->>TLB: Fill translation or fault
    TLB-->>DEC: Physical address or page fault
  end
  DEC->>PROT: Check physical access
  PROT-->>DEC: Permit or access fault
  DEC->>MEM: Load request
  MEM-->>DEC: Data / error
  DEC->>WB: Result or precise exception
~~~

This sequence is conceptual. A cache may sit before or after a particular translation/protection point depending on the architecture; the normative architecture and implementation's cache/MMU design determine the exact ordering.

---

## 6. Address translation: MMU, TLB and page-table walker

### 6.1 Virtual address versus physical address

A virtual address (VA) is the address used by software in a virtual-memory system. A physical address (PA) identifies a location in the platform's physical address map.

The MMU translates VA to PA and checks access permissions. The translation can depend on:

- current privilege mode,
- read/write/execute access type,
- page-table permissions,
- address-space identifier (ASID),
- page size,
- global mapping state,
- optional virtualization stage.

An MMU is not the same as an MPU. An MPU typically checks regions and permissions without general virtual-to-physical page translation. PMP is a RISC-V physical-memory protection mechanism; it is not a replacement for a full virtual-memory MMU.

### 6.2 TLB

A translation lookaside buffer caches recently used translations. Common structures include:

- ITLB for instruction fetch,
- DTLB for data access,
- shared second-level TLB,
- separate instruction/data page-walk caches.

A TLB entry conceptually contains a virtual page tag, physical page number, page size, ASID/global information and permission/attribute bits.

**TLB hit:** translation is returned from the cached entry.

**TLB miss:** the page-table walker reads translation entries from memory. A miss is not necessarily a page fault. The page-table walk can find a valid mapping and refill the TLB.

**Page fault:** the page-table entry or permission state makes the access architecturally invalid. The OS may allocate/map a page, update the page table and retry.

**Access fault:** a physical access or platform protection check fails. This is distinct from a page fault.

### 6.3 RISC-V Sv39 example

For a 64-bit RISC-V supervisor environment supporting Sv39:

- virtual addresses are 39-bit canonical addresses,
- page size is 4 KiB for a base page,
- the page table has three levels,
- each page-table entry is 8 bytes,
- a leaf entry at a higher level can map a larger page when alignment and permission rules are satisfied.

For a 4 KiB page, the lower 12 VA bits are the page offset. The remaining VPN is split into three 9-bit indexes.

~~~text
VA[38:30]  VPN[2]  9 bits
VA[29:21]  VPN[1]  9 bits
VA[20:12]  VPN[0]  9 bits
VA[11:0]   Offset 12 bits
~~~

A simplified walk is:

1. Read the root page-table pointer from the translation-control CSR.
2. Index the root page using VPN[2].
3. If the PTE is non-leaf, use its physical page number as the next-level table.
4. Repeat for VPN[1] and VPN[0].
5. Validate the leaf entry, permissions and page-size alignment.
6. Form the PA using the PPN plus the page offset.
7. Cache the result in the TLB.

A three-level walk can require up to three dependent PTE reads for a base-page mapping, before accounting for page-walk caches, caches, memory latency and implementation optimizations.

### 6.4 TLB invalidation and page-table ordering

Changing a page table does not automatically mean every cached translation has disappeared. Software and hardware need a defined synchronization mechanism.

In RISC-V, SFENCE.VMA is used to order relevant page-table updates and invalidate address-translation caches according to its operands. The exact scope depends on the virtual address and ASID operands. It is not a generic flush of all data caches.

**Common bug:** software updates a PTE and immediately reuses the virtual address without the required translation synchronization. The CPU can continue using a stale TLB entry.

### 6.5 PMP and physical memory attributes

PMP (Physical Memory Protection) checks access permissions against physical address regions. Depending on the implementation, region matching can use TOR, NA4 where supported, or NAPOT encodings. PMP is especially important for limiting lower-privilege accesses and protecting firmware regions.

PMA (Physical Memory Attributes) describe physical-region behavior such as supported access types, cacheability, coherence and idempotency. PMA is not a page table and is not simply a software permission bit.

A robust access path conceptually performs:

1. Translation (if enabled).
2. Physical protection checks.
3. Physical-region attribute checks.
4. Cache/fabric access under the applicable memory-ordering rules.

PMP checks also apply to implicit page-table accesses in virtual-memory implementations. See the privileged specification for exact rules and synchronization requirements.

---

## 7. Privilege architecture, exceptions and interrupts

This lesson uses RISC-V terminology for concrete examples. Arm A-profile uses a different architectural model, including exception levels EL0–EL3 and translation regimes. Do not mix CSR names from RISC-V with system-register names from Arm.

### 7.1 RISC-V privilege levels

| Mode | Typical role |
|---|---|
| M-mode | Machine firmware and highest architectural privilege |
| S-mode | Supervisor / operating system |
| U-mode | Application software |
| Debug mode | Separate debug execution state defined by the debug architecture |
| Hypervisor extension | Adds virtualization support; not present in every core |

M-mode is mandatory in the RISC-V privileged architecture. S-mode and U-mode are optional, and a small embedded core may implement fewer privilege modes than an application-class Linux-capable core.

Privilege is not merely a software convention. It affects instruction legality, CSR access, memory permissions, trap routing and physical protection.

### 7.2 Important machine CSRs

| CSR | Role |
|---|---|
| mstatus | Machine status, privilege/interrupt state |
| misa | ISA extension availability, when implemented as specified |
| mtvec | Trap-vector base and mode |
| mepc | Exception/interrupt return PC |
| mcause | Trap cause |
| mtval | Additional trap value, where defined |
| mie | Interrupt enables |
| mip | Interrupt-pending state |
| mscratch | Machine trap-handler scratch |
| satp | Supervisor address-translation control |
| pmpcfg / pmpaddr | PMP configuration and address regions |
| mhartid | Hardware-thread identifier |

CSR availability and WARL behavior are defined by the implemented ISA profile and extensions. Software must not assume every optional CSR exists or that every field supports every value.

### 7.3 Exception versus interrupt

- **Exception:** synchronous with the instruction or access that caused it, such as illegal instruction, breakpoint, page fault or access fault.
- **Interrupt:** asynchronous event, such as timer, software or external interrupt.

A trap is the architectural control transfer to a handler. Trap entry records state and selects the destination according to the privilege architecture.

### 7.4 Conceptual machine-mode trap flow

For a machine-mode trap in RISC-V, the architectural state includes:

- the interrupted or faulting PC captured in mepc,
- the cause captured in mcause,
- optional trap-specific information in mtval,
- previous privilege recorded in MPP,
- previous interrupt-enable state saved in MPIE,
- current privilege changed to M-mode,
- trap target selected through mtvec.

MRET restores the prior privilege and interrupt-enable state according to the specification and resumes from mepc.

~~~mermaid
stateDiagram-v2
  [*] --> Running
  Running --> TrapEntry: Exception or enabled interrupt
  TrapEntry --> SaveState: Capture PC, cause and status
  SaveState --> Handler: Set privilege and trap PC
  Handler --> Restore: Handler completes
  Restore --> Running: MRET / return
~~~

A real implementation must define precise priority among simultaneous events. The priority is architecture- and implementation-specific within the permitted specification behavior.

### 7.5 Interrupt path

An external interrupt usually travels through multiple blocks:

1. Peripheral asserts an interrupt source.
2. Interrupt controller records pending state and priority.
3. Controller presents an interrupt request to the CPU.
4. CPU checks global and per-source enables and privilege delegation.
5. CPU takes the trap at an architecturally permitted boundary.
6. Handler reads the interrupt-controller state or claim register.
7. Handler services the source and performs the required completion/EOI operation.
8. Pending state clears or remains asserted according to edge/level semantics.

Do not confuse a CPU interrupt-pending CSR with the interrupt controller's peripheral-facing pending register. The CPU may only see a summarized external interrupt pin.

---

## 8. Boot, reset and first instruction

A CPU's first instruction is a system integration problem.

### 8.1 Typical boot sequence

1. Power and clock infrastructure reaches the required state.
2. Reset controller releases the CPU reset in the documented sequence.
3. CPU samples or is configured with a reset/boot address.
4. Instruction fetch reaches ROM or another boot source.
5. Boot ROM establishes stack and minimal machine state.
6. Firmware configures memory protection and required clock/power/peripheral state.
7. Optional DRAM initialization and training complete.
8. Firmware loads or authenticates the next image.
9. Firmware sets trap vectors and interrupt state.
10. If an OS is used, firmware establishes page tables and the required privilege transition.
11. Control transfers to the next stage.

This is a generic flow, not a universal boot sequence. Some platforms boot from flash, mask ROM, SRAM, external memory or a secure controller. Some initialize DRAM before the main CPU is released.

### 8.2 Reset design questions

- Is the reset vector fixed, parameterized or supplied by a boot controller?
- Is reset synchronous or asynchronous at the core boundary?
- Are reset deassertion synchronizers present in each clock domain?
- Are instruction and data interfaces held inactive during reset?
- Are outstanding bus responses drained, discarded or reset with the fabric?
- Is debug-from-reset supported?
- What happens if an interrupt arrives while the CPU is held in reset?
- Are cache, TLB, predictor and debug state reset to architecturally legal values?

### 8.3 Boot failure signatures

| Symptom | Likely investigation |
|---|---|
| PC remains at reset address | Reset release, clock, fetch grant, ROM decode |
| PC jumps to an unexpected address | Boot address wiring, reset vector, instruction corruption |
| First load faults | Address map, PMP, PMA, bus error, alignment |
| Firmware reaches trap handler repeatedly | mtvec, mepc, mcause, mtval and faulting instruction |
| Works in simulation but not silicon | Reset timing, ROM timing, clock startup, memory integrity, CDC |
| CPU halts before first instruction | Debug reset-halt request, lifecycle/debug policy |

---

## 9. Bus and NoC integration

A CPU subsystem may expose separate instruction and data request ports, a unified master port, cache refill/writeback ports, page-table-walker traffic and debug system-bus access.

### 9.1 Native core interface versus SoC fabric

~~~mermaid
flowchart LR
  CORE[CPU core] --> IAD[Instruction adapter]
  CORE --> DAD[Data adapter]
  IAD --> XBAR[AXI / AHB / TileLink bridge or NoC]
  DAD --> XBAR
  PTW[Page-table walker] --> XBAR
  XBAR --> ROM[Boot ROM]
  XBAR --> SRAM[SRAM]
  XBAR --> CACHE[Last-level cache]
  CACHE --> DDR[DDR controller]
  XBAR --> PERI[MMIO peripherals]
~~~

Questions for integration:

- Are instruction and data ports independent?
- Can the core issue multiple outstanding transactions?
- How are responses associated with requests?
- Does the adapter preserve ordering required by the core?
- Are MMIO accesses marked as device/non-cacheable?
- Can page-table walks reach coherent memory?
- Does the CPU share a cache with DMA or other cores?
- How are bus errors translated into instruction/data access faults?
- Can debug system-bus access bypass the CPU cache, and if so, how is coherency maintained?

### 9.2 MMIO versus normal memory

MMIO accesses can have side effects. A read may clear a status bit; a write may launch a DMA. They must not be treated like ordinary cacheable RAM.

The platform must define memory attributes and ordering so that software-visible device operations behave correctly. A compiler barrier, CPU memory fence and interconnect ordering rule solve different problems; they are not interchangeable.

### 9.3 Address map example

| Region | Example base | Attribute |
|---|---:|---|
| Boot ROM | 0x0000_0000 | Read/execute, implementation-specific |
| SRAM | 0x1000_0000 | Normal memory |
| UART | 0x4000_0000 | Device/MMIO |
| Interrupt controller | 0x4800_0000 | Device/MMIO |
| DDR | 0x8000_0000 | Normal cacheable memory |

These are illustrative addresses only. Never copy them into a real SoC without its memory map.

---

## 10. Debug architecture

A CPU can fail before a console or operating system is available. Debug must therefore be designed into the subsystem, not added as an afterthought.

### 10.1 RISC-V debug path

~~~mermaid
flowchart LR
  HOST[Debugger / GDB] --> TRANS[OpenOCD or debug translator]
  TRANS --> DTM[Debug Transport Module]
  DTM --> DMI[Debug Module Interface]
  DMI --> DM[Debug Module]
  DM --> HART[Hart halt / resume / register access]
  DM --> SBA[Optional system-bus access]
  HART --> TRIG[Trigger module]
~~~

The RISC-V Debug Specification describes a Debug Transport Module, Debug Module Interface and Debug Module. Depending on the implementation, the debug module can halt/resume harts, access registers, use a program buffer and optionally access system memory through a system-bus-access block.

### 10.2 Debug integration checklist

- Can the debugger connect before normal boot?
- Can it halt a hart immediately after reset?
- Are all harts independently selectable?
- Are PC, GPRs and required control registers accessible?
- Are trigger resources implemented and advertised accurately?
- Does system-bus access use physical addresses?
- Is debugger memory access coherent with CPU caches?
- Can lifecycle or fuse policy disable or restrict debug?
- Are debug clocks and resets available when the CPU clock is gated?

**Security note:** Debug is a privileged attack surface. Production lifecycle policy must control authentication, unlock, trace and permanent disable behavior without breaking authorized manufacturing and failure analysis.

---

## 11. Performance engineering

### 11.1 CPI, IPC and throughput

For a single-issue core:

- CPI = cycles / retired instructions
- IPC = retired instructions / cycles
- IPC = 1 / CPI

Assume a 1 GHz single-issue core with:

- ideal base CPI = 1.00,
- branch frequency = 8%,
- branch misprediction rate = 20%,
- branch misprediction penalty = 4 cycles,
- load-use frequency = 12%,
- load-use penalty = 1 cycle.

Mispredicted branch fraction = 0.08 × 0.20 = 0.016.

Branch penalty contribution = 0.016 × 4 = 0.064 CPI.

Load-use penalty contribution = 0.12 × 1 = 0.12 CPI.

Estimated CPI = 1.00 + 0.064 + 0.12 = 1.184.

Estimated IPC = 1 / 1.184 = 0.845.

At 1 GHz, estimated retirement rate = 845 million instructions per second.

This is a simplified model. It assumes penalties do not overlap and ignores cache misses, TLB misses, memory-level parallelism, interrupts and front-end bubbles.

### 11.2 TLB reach

For a 64-entry fully associative TLB containing 4 KiB pages:

64 × 4 KiB = 256 KiB of translation reach.

For 32 entries containing 2 MiB pages:

32 × 2 MiB = 64 MiB of translation reach.

This is only a capacity calculation. Real effective reach depends on associativity, mixed page sizes, replacement, ASIDs, global entries and workload locality.

### 11.3 Why page walks can hurt

A TLB miss can trigger several dependent page-table reads. If the page-table entries miss in cache and travel to external DRAM, the access latency can become much larger than a normal cache hit.

Mitigations include:
- larger or better-organized TLBs,
- second-level TLBs,
- page-walk caches,
- larger pages where appropriate,
- locality-aware OS allocation,
- correct ASID use to reduce unnecessary flushes.

These are architectural and software trade-offs, not a free performance gain.

---

## 12. RTL and microarchitecture implementation checklist

The following is a review checklist, not a complete CPU RTL implementation.

### 12.1 Fetch and redirect

- Track the PC associated with every outstanding fetch.
- Flush stale prefetched instructions after a redirect.
- Handle instruction access faults at the correct PC.
- Prevent a late response from a killed fetch from entering the decode stream.
- Preserve instruction alignment and compressed-instruction rules where supported.

### 12.2 Load-store unit

- Generate byte enables from access size and address offset.
- Implement the documented misaligned-access policy.
- Sign-extend and zero-extend loads correctly.
- Preserve request/response association under stalls.
- Do not retire a load before its required error status is known.
- Prevent stores from becoming externally visible before the architecture permits them.

### 12.3 Trap and flush control

- Define priority for synchronous exceptions, interrupts, debug requests and redirects.
- Capture the correct faulting PC.
- Prevent younger instructions from committing after a precise exception.
- Restore privilege and interrupt state correctly on return.
- Avoid double-fault loops if trap-vector fetch itself fails.

### 12.4 Example assertions to adapt

~~~systemverilog
// A request must remain stable until accepted.
// Adapt reset polarity and handshake semantics to the actual interface.
property req_stable_until_grant;
  @(posedge clk) disable iff (!rst_n)
    req && !gnt |=> req && $stable({addr, we, be, wdata});
endproperty
assert property (req_stable_until_grant);

// A valid response must correspond to an outstanding request.
// This is a conceptual property; use a counter/queue for multiple outstanding requests.
~~~

For a pipelined or multi-outstanding interface, a single outstanding bit is insufficient. Use a request tracker, ID scoreboard or queue that matches the actual protocol.

---

## 13. Verification strategy

CPU verification must cover both architectural correctness and the implementation-specific microarchitecture.

### 13.1 Architectural verification

- ISA compliance tests.
- Privileged architecture tests.
- CSR read/write and WARL behavior.
- Exception and interrupt behavior.
- PMP/PMA access checks.
- Virtual-memory translation and page faults.
- Debug specification compliance.
- Atomic and memory-ordering tests when those extensions are implemented.

### 13.2 Microarchitectural verification

- Pipeline forwarding and stalls.
- Branch redirect and flush.
- Fetch response arriving after redirect.
- Load/store request backpressure.
- Misaligned accesses.
- Simultaneous interrupt, exception and debug request.
- TLB hit, miss, refill and invalidation.
- Page-walk bus errors.
- Cache refill/writeback interactions.
- Reset while requests are pending, according to the integration contract.

### 13.3 Assertions and formal properties

Examples of properties to prove or assert:

1. No instruction retires twice.
2. A killed instruction never commits architectural state.
3. A load result is written only after a successful response.
4. A faulting instruction records the architecturally correct PC and cause.
5. A privilege transition cannot grant permissions beyond the architecture.
6. A PMP-denied access never reaches a side-effecting target.
7. A stale fetch response cannot be decoded after its stream is invalidated.
8. An accepted request receives exactly one response, unless reset semantics explicitly cancel it.
9. Debug halt/resume cannot corrupt architectural state.
10. Reset returns the core to a documented state.

### 13.4 Coverage plan

Cross the following dimensions:

- instruction class × pipeline stall type,
- branch outcome × redirect timing,
- privilege mode × access type,
- PMP region × permission × access result,
- TLB hit/miss × page size × ASID,
- interrupt pending × global enable × current privilege,
- exception × simultaneous debug request,
- reset timing × outstanding transaction count,
- bus error × instruction/data access type.

OpenTitan's Ibex verification documentation is a useful public example of crossing exceptions, interrupts, debug, stalls and PMP permission combinations.

---

## 14. Post-silicon bring-up and debug playbook

### 14.1 First 30 minutes of CPU bring-up

1. Verify CPU supply, clock and reset at the pins or internal observability points.
2. Confirm reset deassertion and boot-address value.
3. Connect JTAG/debug transport and identify the hart.
4. Halt the CPU or use reset-halt if supported.
5. Read PC and essential architectural state.
6. Confirm instruction-bus request and response activity.
7. Confirm boot ROM decode and instruction integrity.
8. Single-step through the first instructions.
9. Check data-bus activity on the first load/store.
10. Read trap CSRs if execution diverts to a handler.
11. Confirm interrupt sources are quiet or masked until handlers are ready.
12. Save a trace/counter snapshot before changing configuration.

### 14.2 Failure signature matrix

| Observation | Suspect blocks | Evidence to collect |
|---|---|---|
| No instruction request | Reset, clock, fetch enable, debug halt | Reset state, clock status, debug status, fetch request |
| Request asserted, no grant | Fabric decode, target power/clock, adapter | Address, grant, target select, timeout |
| Grant but no response | Memory controller, bridge, response path | Outstanding request ID, response-valid, target trace |
| Wrong instruction | ROM image, address alignment, ECC, stale fetch | PC, returned data, integrity status, redirect history |
| Repeated page fault | satp, PTE, permissions, ASID, SFENCE.VMA | VA, mepc, mcause, mtval, page tables, TLB state |
| Access fault only on MMIO | PMP/PMA, address decode, security firewall | PA, privilege, access type, protection registers |
| CPU stalls after load | LSU response, interconnect, DDR, cache | Request age, outstanding count, response path |
| Interrupt never taken | Controller pending/enable, mie/mip, global enable | Peripheral IRQ, controller state, CPU CSR snapshot |
| Debug cannot halt | DTM/DMI, lifecycle policy, clock/reset | JTAG ID, DMI response, debug authentication |
| Rare wrong result under load | hazard, replay, cache, CDC, ordering | Retirement trace, bus trace, syndrome and counters |

### 14.3 Evidence hierarchy

When debugging, collect evidence from the outside inward:

1. Board-level power/clock/reset.
2. Debug transport connectivity.
3. CPU architectural state (PC, cause, privilege).
4. Instruction and data interface waveforms.
5. Fabric request/response trace.
6. Cache/TLB/PMP status.
7. DDR/ROM/target-side status.
8. Performance counters and retirement trace.

Avoid changing several settings at once. Preserve the first failing state, then isolate one layer at a time.

---

## 15. Reference implementation study

### A. lowRISC Ibex — compact embedded core

Study:
- two-stage pipeline,
- instruction prefetch and fetch FIFO,
- load-store interface,
- PMP and CSR behavior,
- debug and exception control.

The Ibex documentation describes a two-stage IF and ID/EX organization. The fetch stage uses buffering to improve throughput and timing closure. This is a valuable example of a small core whose microarchitecture is intentionally different from an application-class out-of-order CPU.

### B. OpenHW CVA6 — application-class core

Study:
- six-stage application-class pipeline family,
- configurable 32/64-bit variants,
- optional MMU, PMP, FPU and caches,
- instruction and data TLBs,
- page-table walker,
- AXI-facing SoC integration examples.

The CVA6 MMU documentation describes ITLB, DTLB, optional shared TLB and a page-table walker. It is a useful contrast with the compact Ibex design.

### C. OpenTitan CPU wrapper — subsystem integration

Study:
- Ibex core wrapper,
- instruction and data TileLink host ports,
- boot address and hart ID,
- software, timer and external interrupt inputs,
- debug request,
- watchdog NMI,
- crash-dump collection,
- security hardening and lockstep options.

The wrapper illustrates the key lesson: the CPU core is only one part of the CPU subsystem. The platform adds reset, boot, interrupt, debug, security and bus integration around it.

---

## 16. Hands-on exercises

### Exercise 1 — Trace the first instruction

Use a waveform or a simple testbench. Hold reset, release it, and record:
- first PC,
- first fetch request,
- grant cycle,
- response cycle,
- first decoded instruction.

Explain the difference between request acceptance and response completion.

### Exercise 2 — Build a dependency test

Create:
~~~text
ADD x5, x1, x2
SUB x6, x5, x3
~~~

Observe whether the core forwards or stalls. Add a load followed by a dependent ALU instruction and compare.

### Exercise 3 — Force a branch redirect

Create a taken branch with sequential instructions behind it. Verify that wrong-path instructions do not retire.

### Exercise 4 — Create a PMP access denial

Configure a region to deny a lower-privilege load. Confirm that:
- the access does not reach the target,
- the correct exception is taken,
- the faulting PC and cause are captured.

Use only a documented PMP configuration for the selected core.

### Exercise 5 — Trigger a TLB miss

On a system with virtual memory, access a mapped address, then inspect the translation behavior. Repeat with a new mapping and study the effect of SFENCE.VMA.

### Exercise 6 — Debug an interrupt

Assert a timer or external interrupt. Observe the path from source to controller to CPU trap. Record entry latency in cycles.

### Exercise 7 — Integrate a CPU with a fabric

Connect a simple CPU or bus-functional master to a small address decoder. Map ROM, SRAM and one MMIO register. Verify fetch, load, store and access-fault behavior.

### Exercise 8 — Create a silicon debug worksheet

For a simulated boot failure, record reset, PC, request/grant/response, trap CSRs, bus target and debug state. Write a hypothesis before changing the design.

---

## 17. Interview questions

1. What is the difference between an ISA and a microarchitecture?
2. Why can two cores implement the same ISA and have very different performance?
3. What is the difference between pipeline latency and throughput?
4. Explain structural, data and control hazards.
5. What is forwarding, and why does a load-use hazard often still stall?
6. Why do out-of-order processors need in-order retirement?
7. What is the difference between a TLB miss and a page fault?
8. What is the difference between an MMU, MPU and PMP?
9. Why is SFENCE.VMA required after certain page-table updates?
10. What is the difference between a page fault and a physical access fault?
11. What state must be captured on a precise exception?
12. How does a peripheral interrupt reach a CPU trap handler?
13. Why should MMIO not be treated as ordinary cacheable memory?
14. Why might a debugger's system-bus access be incoherent with CPU caches?
15. What evidence would you collect if the CPU fetch request is asserted but no grant returns?

---

## 18. Design-review checklist

Before signing off a CPU subsystem, review:

- [ ] ISA profile and optional extensions are frozen and documented.
- [ ] Pipeline, retirement and exception behavior are specified.
- [ ] Reset PC and boot-address selection are verified.
- [ ] Instruction/data interface protocols and outstanding limits are documented.
- [ ] Address map and memory attributes are reviewed.
- [ ] MMU/TLB/page-walker behavior is verified if present.
- [ ] PMP/MPU and security firewalls are tested.
- [ ] Interrupt priorities, delegation and reset behavior are verified.
- [ ] Debug transport, halt/resume and lifecycle restrictions are tested.
- [ ] Cache and DMA coherency assumptions are documented.
- [ ] Clock/reset/CDC/RDC checks are clean.
- [ ] Performance counters and trace points are sufficient for silicon debug.
- [ ] Architectural compliance and microarchitectural verification are both complete.

---

## 19. Key takeaways

1. A CPU subsystem is a system of contracts, not just a processor RTL block.
2. ISA behavior is normative; pipeline depth and TLB structure are implementation choices.
3. The load/store path crosses execution, translation, protection, cache and interconnect boundaries.
4. Precise exceptions and in-order retirement are essential architectural concepts even in out-of-order designs.
5. Boot, interrupt and debug integration must work before an operating system can help.
6. CPU performance depends on memory hierarchy and SoC traffic, not just clock frequency or pipeline depth.
7. Post-silicon debug needs architectural state, bus traces and reset/clock observability.

---

## 20. Authoritative references and source discipline

### Normative architecture specifications

1. **RISC-V Privileged Architecture:** https://docs.riscv.org/reference/isa/priv/  
   Use for privilege modes, traps, CSRs, machine/supervisor architecture, PMP, virtual memory and memory-management fences.

2. **RISC-V Unprivileged ISA:** https://docs.riscv.org/reference/isa/unpriv/  
   Use for instruction encodings, base instructions, extensions, loads/stores, atomics and architectural instruction behavior.

3. **RISC-V Debug Specification:** https://docs.riscv.org/reference/debug/  
   Use for debug transport, debug module, abstract commands, program buffer, triggers and system-bus access.

4. **Arm Architecture Reference Manual (A-profile):** https://developer.arm.com/documentation/ddi0487/latest  
   Use for AArch64/AArch32 exception levels, translation, system registers, memory attributes and exception behavior. Arm and RISC-V privilege models are not interchangeable.

### Public implementation references

5. **Ibex documentation:** https://ibex-core.readthedocs.io/  
   Pipeline, fetch, LSU, CSRs, PMP, exceptions and debug.

6. **Ibex source:** https://github.com/lowRISC/ibex  
   Public SystemVerilog reference implementation.

7. **CVA6 documentation:** https://docs.openhwgroup.org/projects/cva6-user-manual/  
   Application-class core, pipeline, MMU, TLB and integration.

8. **OpenTitan CPU wrapper:** https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rv_core_ibex/index.html  
   Example of a CPU wrapper integrated with boot, reset, interrupt, debug and security signals.

9. **OpenTitan CPU wrapper verification:** https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rv_core_ibex/dv/index.html  
   Example of verification goals for the core and wrapper.

### Historical sources

10. **IBM — RISC architecture history:** https://www.ibm.com/history/risc  
    Documents the 1974 research motivation and IBM's account of the 801/RISC work.

11. **RISC-V International — About and history:** https://riscv.org/about/  
    Documents the May 2010 Berkeley origin and 2015 Foundation milestone.

**Source discipline:** ISA requirements above are architectural claims. Ibex, CVA6 and OpenTitan details are explicitly implementation examples. Numerical examples and the illustrative address map are teaching assumptions, not vendor specifications.

---

## Next lesson — Session 10

**Cache and Coherency — From L1/L2 to DMA and Multi-core Silicon**

- SRAM arrays, tags, lines, sets and ways.
- Direct-mapped, set-associative and fully associative caches.
- Write-through, write-back, write-allocate and no-write-allocate.
- Cache miss handling, refill and eviction.
- Write buffers, MSHRs and miss-status tracking.
- Inclusive, exclusive and non-inclusive hierarchy.
- Coherence fundamentals: MSI/MESI and snoop/directory approaches.
- ACE/CHI concepts and coherent interconnect behavior.
- CPU versus DMA coherency.
- Cache maintenance, barriers and ownership transfer.
- RTL structures, verification and silicon performance/debug.

**Progress after this lesson: 9 of 38 sessions complete.**
