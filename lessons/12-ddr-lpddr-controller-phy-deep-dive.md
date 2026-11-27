# Lesson 12 — DDR / LPDDR: DRAM Architecture, Controller Scheduling, PHY Training and Silicon Debug

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 12 of 38 · **Duration:** 60 minutes · **Level:** Intermediate to advanced

> **Scope and evidence discipline:** This lesson explains the shared DRAM concepts and then separates DDR, LPDDR and HBM. Exact command encodings, timing limits, initialization order, mode-register fields, training algorithms, voltage rails and pin assignments are generation-, density-, speed-bin-, package- and vendor-specific. Use the exact JEDEC revision and the selected DRAM/PHY/controller data sheets for implementation. Numerical examples marked *illustrative* are not universal limits. Vendor guides are implementation examples, not substitutes for the normative JEDEC standard.

## 1. Learning objectives and 60-minute plan

By the end, you should be able to:

1. Explain why external DRAM uses a 1-transistor/1-capacitor (1T1C) cell and why it must be refreshed.
2. Navigate the hierarchy of channel, subchannel, rank, die, bank group, bank, row and column.
3. Walk through ACTIVATE, READ, WRITE, PRECHARGE, REFRESH and mode-register operations.
4. Interpret the important timing constraints: tRCD, CL/tCL, tRP, tRAS, tRC, tRRD, tFAW, tCCD, tWR, tWTR, tRFC and tREFI.
5. Explain the controller, PHY and DRAM-device responsibilities and the AXI-to-DRAM transaction path.
6. Calculate peak bandwidth, approximate transaction latency and refresh overhead.
7. Explain row-hit scheduling, bank interleaving, address mapping, QoS and fairness.
8. Describe initialization, training, low-power states, ECC and post-silicon characterization.
9. Distinguish DDR, LPDDR, GDDR and HBM by their system-level design goals.
10. Build a verification and bring-up plan for a real SoC memory subsystem.

| Time | Topic |
|---:|---|
| 0–6 min | Why DRAM exists; history and memory families |
| 6–14 min | 1T1C cell, hierarchy and command model |
| 14–23 min | Timing parameters and bank state machines |
| 23–32 min | Controller architecture, scheduling and address mapping |
| 32–39 min | Bandwidth, latency and refresh calculations |
| 39–47 min | DDR PHY, training, LPDDR and power states |
| 47–54 min | Integration, ECC, verification and silicon debug |
| 54–60 min | Worked design review, interview questions and exercises |

## 2. Why external DRAM exists

An SoC normally uses several memory technologies because no single memory provides the best combination of density, latency, bandwidth, retention, cost and power.

- **Register files and SRAM:** fast, low-latency, no refresh, but relatively expensive in die area per bit.
- **ROM / OTP:** stores immutable or one-time-programmed information such as boot code, fuses and device configuration.
- **External DRAM:** high capacity and good cost per bit; requires refresh and a high-speed interface.
- **Flash storage:** non-volatile and high capacity, but much slower and with erase/program constraints.

A conventional DRAM bit is a capacitor storing charge and an access transistor connecting it to a bitline. The stored charge leaks, so the memory must be refreshed. Reading a DRAM row is destructive at the cell level: the tiny bitline signal is sensed and the value is restored as part of the access sequence.

~~~mermaid
flowchart LR
  subgraph Cell["Conceptual 1T1C DRAM cell"]
    C["Storage capacitor"] --- T["Access transistor"]
    T --- BL["Bitline"]
    WL["Wordline"] --> T
  end
  BL --> SA["Sense amplifier"]
  SA --> ROW["Row buffer / active row"]
  ROW --> COL["Column selection"]
  COL --> IO["I/O datapath"]
~~~

**Design rationale — documented fact:** capacitor-based DRAM offers much higher bit density than a conventional multi-transistor SRAM cell. **Engineering interpretation:** that density makes DRAM attractive for main memory, while its analog sensing, refresh and timing requirements justify a dedicated controller and PHY rather than treating it like a simple asynchronous SRAM.

## 3. Historical evolution and design rationale

### 3.1 From asynchronous DRAM to synchronous and double-data-rate memory

Early DRAM interfaces used asynchronous row and column strobes. As processor and system-bus frequencies rose, synchronous DRAM (SDRAM) introduced clocked commands, programmable burst behavior and bank organization. DDR SDRAM then transferred data on both edges of the data strobe/clocking scheme, increasing transfer rate without requiring the internal array to operate at the external pin rate.

Selected public milestones:

| Generation / family | Public milestone | Engineering significance |
|---|---:|---|
| SDR SDRAM | JEDEC standardization in the 1990s | Clocked command interface, banks and burst-oriented access |
| DDR SDRAM | JESD79 published in 2000 | Transfers on both edges; evolutionary migration from SDR |
| DDR2 | JESD79-2 published in 2003 | Higher prefetch and faster I/O while keeping array-core frequency manageable |
| DDR3 | JESD79-3 published in 2007 | Higher transfer rates, lower voltage and more extensive calibration features |
| DDR4 | JESD79-4 published 25 Sep 2012 | Bank groups, improved signaling and efficiency at higher data rates |
| LPDDR5 | JESD209-5 published 19 Feb 2019 | Mobile-focused bandwidth and power improvements |
| DDR5 | JESD79-5 published 14 Jul 2020 | BL16, more banks, two independent subchannels per standard DIMM, on-die ECC and improved high-speed signaling |
| LPDDR6 | JESD209-6 published in July 2025 | New low-power interface generation; use the actual standard and device revision for its channel, command and training details |
| HBM4 | JESD270-4 published in April 2025 | Very-wide stacked-memory interface aimed at AI/HPC bandwidth and capacity |

Sources for the dated milestones are listed in Section 27. Dates refer to standard publication, not first commercial shipment. Vendor products may appear before or after a standard milestone.

### 3.2 Why the families diverged

| Family | Main system objective | Typical integration style | Main trade-off |
|---|---|---|---|
| DDR | General-purpose high-capacity main memory | DIMM or discrete DRAM, board-level channel | Broad ecosystem and capacity; board routing and power are significant |
| LPDDR | Low-power mobile / embedded memory | Usually soldered close to the SoC, often package- or board-integrated | Lower power and compact integration; tighter platform-specific constraints |
| GDDR | High bandwidth for graphics and accelerators | Dedicated high-speed graphics memory channels | High bandwidth; power and board complexity |
| HBM | Extremely high bandwidth per package for HPC/AI | Vertically stacked DRAM beside/above logic through advanced packaging | Exceptional bandwidth and energy per bit; package, thermal, yield and cost complexity |

These are market and architecture goals, not strict application boundaries. For example, LPDDR appears in some automotive and PC systems, while HBM is used beyond GPUs.

### 3.3 Why DDR generations changed their architecture

The standard evolved through several levers:

- **Prefetch and burst length:** transfer more bits from the internal array per external command while keeping the core array clock lower than the I/O transfer rate.
- **More banks and bank groups:** allow more independent work to be in flight and improve scheduling opportunities.
- **Improved I/O signaling:** support higher pin rates with controlled impedance, termination, equalization and training.
- **Lower operating voltage and power modes:** reduce energy, although higher speed can increase dynamic power.
- **ECC and reliability features:** address scaling-related reliability and system RAS requirements.

JEDEC's DDR5 announcement dated 14 July 2020 describes BL16, 32 banks, two independent subchannels on a DIMM, DFE, on-die ECC and a 1.1 V supply relative to DDR4. Micron's DDR5 feature comparison is a useful device-vendor view of the differences. Do not generalize a DIMM organization to every soldered-down or x16 device configuration.

## 4. Full SoC memory-subsystem architecture

The CPU or DMA engine does not normally issue DRAM ACTIVATE and READ commands directly. It issues memory transactions on an SoC fabric. The controller converts those transactions into legal DRAM command sequences.

~~~mermaid
flowchart TB
  CPU["CPU / GPU / DSP / NPU"]
  DMA["Camera / Display / Audio / DMA"]
  FAB["AXI / NoC fabric"]
  QOS["Port queues, firewall and QoS"]
  MAP["Address mapping / channel selection"]
  SCHED["Command scheduler"]
  BANK["Bank machines and timing tracker"]
  REF["Refresh and power manager"]
  ECC["ECC / data integrity"]
  PHY["DDR / LPDDR PHY"]
  DRAM["DRAM devices / ranks / channels"]
  CPU --> FAB
  DMA --> FAB
  FAB --> QOS --> MAP --> SCHED
  SCHED --> BANK
  REF --> BANK
  BANK --> ECC --> PHY --> DRAM
  DRAM --> PHY --> ECC --> FAB
~~~

### Controller responsibilities

1. Accept AXI or another SoC-native request protocol.
2. Track IDs, bursts, byte enables, ordering and access permissions.
3. Map system addresses to channel, rank, bank, row and column.
4. Queue and schedule commands while enforcing all timing constraints.
5. Reorder requests where permitted to improve row locality and bank parallelism.
6. Generate refresh, mode-register and power-management commands.
7. Coordinate ECC, parity/CRC where supported, error reporting and counters.
8. Translate controller-native commands into the PHY interface.

### PHY responsibilities

The PHY handles the electrical and timing boundary between controller logic and the DRAM pins:

- Clock generation and distribution to the DRAM interface.
- DQ and DQS transmit/receive timing.
- Delay elements, per-bit deskew and read capture.
- Training patterns, Vref calibration and timing-window centering.
- I/O drivers, receivers, termination controls and calibration.
- Clock-domain crossing between the controller clock and PHY timing domains.

The exact partition between controller and PHY varies by vendor. Some PHYs expose a training sequencer; some controllers orchestrate the process. Treat the integration guide as authoritative for a selected IP.

## 5. DRAM organization: understand the hierarchy

A typical memory hierarchy is:

~~~text
SoC memory controller
  └── Channel (independent command/data interface)
       └── Rank or logical device-select group (where applicable)
            └── DRAM device / die
                 └── Bank group (generation-dependent)
                      └── Bank
                           └── Row (activated into row buffer)
                                └── Column (selected data burst)
~~~

Important distinctions:

- **Channel:** an independent memory interface with its own command/data resources.
- **Subchannel:** a narrower independent command/data path within a larger module/device organization. DDR5 DIMMs expose two independent subchannels; do not assume every DRAM package uses the same module topology.
- **Rank:** a set of devices selected together to provide the channel data width in module-based systems.
- **Die:** a physical memory die inside a package. One package can contain multiple dies.
- **Bank:** an independently managed array region with its own row state.
- **Bank group:** a grouping used by some generations to define additional timing relationships.
- **Row:** a wordline whose cells are sensed into the bank's row buffer.
- **Column:** selects a portion of the active row for a burst transfer.

A row buffer is not a separate SRAM cache in the ordinary CPU-cache sense. It is the set of sense amplifiers holding the currently active row for a bank.

## 6. Command protocol and bank state machine

The exact command encoding changes across generations. The following are conceptual command meanings shared by conventional DDR-style devices.

| Command | Purpose |
|---|---|
| ACTIVATE (ACT) | Open a row in a selected bank |
| READ (RD) | Select a column and schedule a read burst |
| WRITE (WR) | Select a column and schedule a write burst |
| PRECHARGE (PRE) | Close an active row and return a bank toward idle |
| REFRESH (REF) | Restore charge in the required rows/banks |
| MODE REGISTER SET (MRS) | Configure device operating behavior |
| ZQ calibration | Calibrate output/termination-related analog behavior on supported devices |
| Power-state commands | Enter/exit supported power-down or self-refresh modes |

### Conceptual per-bank state machine

~~~mermaid
stateDiagram-v2
  [*] --> Idle
  Idle --> Activating: ACT(row)
  Activating --> Active: tRCD satisfied
  Active --> Active: READ / WRITE to open row
  Active --> Precharging: PRE after legal constraints
  Precharging --> Idle: tRP satisfied
  Idle --> Refreshing: REF when eligible
  Refreshing --> Idle: tRFC satisfied
~~~

This is a teaching abstraction. Real DRAM devices have additional command, timing, power and refresh restrictions, and a controller tracks them at bank, bank-group, rank and channel scope.

### A row hit, row miss and row conflict

- **Row hit:** requested column belongs to the already-open row in that bank. A READ/WRITE can be scheduled subject to column and data-bus timing.
- **Closed-bank miss:** bank is idle; controller must ACTIVATE the requested row, wait for tRCD, then issue READ/WRITE.
- **Row conflict:** a different row is open. Controller must first satisfy constraints, PRECHARGE, wait tRP, ACTIVATE the new row, wait tRCD, then access the column.

A row hit is often faster, but a row-hit-only policy can starve other traffic. Fairness and QoS matter.

## 7. Timing parameters: the controller's legality rules

The DRAM protocol is not simply a list of commands. Each command starts timers and creates constraints on later commands.

| Parameter | Meaning | Engineering interpretation |
|---|---|---|
| tRCD | ACTIVATE to READ/WRITE delay | Row must be sensed before column access |
| CL / tCL | READ command to first read data latency | Data-return delay |
| CWL / tCWL | WRITE command to write-data timing | Write data launch relationship |
| tRP | PRECHARGE time | Bank recovery before next ACTIVATE |
| tRAS | Minimum ACTIVATE-to-PRECHARGE interval | Row must remain active long enough |
| tRC | ACTIVATE-to-next-ACTIVATE same-bank cycle | Often constrained by tRAS + tRP |
| tRRD | ACTIVATE spacing across banks | Limits activation current and power |
| tFAW | Four-activate window | Limits how many banks may be activated in a rolling window |
| tCCD | Column-command spacing | Controls READ/WRITE command rate |
| tWR | Write recovery before PRECHARGE | Written data must be safely committed |
| tWTR | WRITE-to-READ turnaround | Allows bus/device recovery after writes |
| tRTP | READ-to-PRECHARGE delay | Protects read operation before closing row |
| tRFC | Refresh cycle time | Device unavailable for the required refresh interval |
| tREFI | Average refresh interval | Refresh scheduling deadline/average cadence |
| tCK | DRAM clock period | Timing values may be specified in cycles, time, or both |

**Important:** names with suffixes such as bank-group variants, long/short values, per-bank refresh values and fine-granularity modes are generation-specific. The controller must implement the exact timing table for the selected DRAM part and operating point.

### Timing conversion

A timing requirement may be specified in clock cycles and/or nanoseconds. The controller must satisfy the stricter converted requirement.

For a timing value specified as both a minimum number of cycles and a minimum time:

\[
t_{\min} = \max(N_{\min}\cdot t_{CK},\ t_{\min,ns})
\]

For an illustrative CK period of 0.3125 ns, a 40-cycle requirement corresponds to 12.5 ns.

### Conceptual row-conflict walkthrough

~~~text
Bank 2 currently has Row A open.
Request arrives for Row B in the same bank.

PRECHARGE Bank 2
  wait until tRP and all prior read/write constraints are satisfied
ACTIVATE Bank 2, Row B
  wait tRCD
READ Bank 2, Column C
  wait the read latency; return the burst on DQ
~~~

A controller that omits one dependency can work in a short simulation and fail only under a particular row-hit/row-conflict pattern at high frequency.

## 8. Data interface and signals

Signal names and polarity are generation-specific. Use the selected device's pin list and PHY guide. Common signal concepts include:

| Signal group | Direction / role | Function |
|---|---|---|
| CK / CK_t, CK_c | Controller to DRAM | Differential command/address clock |
| CA / command-address | Controller to DRAM | Command and address information |
| CS_n | Controller to DRAM | Device/rank selection |
| CKE or generation-specific low-power control | Controller to DRAM | Clock/power-state control where defined |
| RESET_n | Controller to DRAM | Device reset |
| ODT | Controller/device behavior | On-die termination control |
| DQ | Bidirectional | Read and write payload data |
| DQS | Bidirectional / strobe | Source-synchronous data capture/launch timing |
| DM / DBI / CRC-related signals | Generation-dependent | Masking, bus inversion or integrity support |
| WCK | Controller to LPDDR device | High-speed write clock in applicable LPDDR generations |
| RDQS | Device to controller, where supported | Read data strobe in applicable LPDDR modes |

### Why DQS matters

The controller and DRAM do not rely on a single global clock edge to sample every high-speed DQ bit. DQS is associated with the data burst and is used as a source-synchronous timing reference. The PHY must position the sampling point inside the valid data eye.

~~~text
Conceptual read burst (not to scale)

DQS:   __/‾\__/‾\__/‾\__/‾\__
DQ :   --< D0 >< D1 >< D2 >< D3 >--
             ^      ^      ^
          sample near the center of each valid data eye
~~~

The actual preamble, postamble, edge relationships, burst length and training mode are defined by the selected generation and mode.

## 9. Controller microarchitecture and scheduling

A practical controller commonly has:

- AXI request acceptance and response tracking.
- Read/write queues and transaction metadata.
- Address mapper and interleave logic.
- Bank state machines.
- Per-bank open-row tracking.
- Command scheduler and global timing scoreboard.
- Read and write data FIFOs.
- Refresh manager.
- Power-state manager.
- ECC/integrity datapath.
- PHY command/data interface.

### Scheduling policies

**First-ready / first-come-first-served:** simple and predictable, but may miss row locality.

**FR-FCFS (first-ready, first-come-first-served):** among legal commands, prefer ready row hits, while preserving age/fairness rules. It can improve throughput but needs starvation controls.

**QoS-aware scheduling:** accounts for latency-critical ports, bandwidth reservations, urgency, traffic class and fairness. A camera/display real-time stream may need a different service guarantee from bulk DMA.

**Write draining:** controllers often batch writes to reduce read/write turnaround penalties. A write-drain threshold must not create unbounded read latency.

### AXI to DRAM path

~~~mermaid
sequenceDiagram
  participant M as AXI Manager
  participant C as DDR Controller
  participant P as PHY
  participant D as DRAM
  M->>C: ARADDR / ARLEN / ARID
  C->>C: Queue, map address, select bank/row/column
  C->>P: ACTIVATE command
  P->>D: ACT + row address
  Note over C,D: Wait tRCD and all bank/channel constraints
  C->>P: READ command
  P->>D: READ + column address
  D-->>P: DQ burst with DQS
  P-->>C: Captured read data
  C-->>M: RDATA / RID / RLAST / RRESP
~~~

The controller can accept a request long before the DRAM command is issued. AXI response ordering and IDs are managed at the controller/fabric boundary; DRAM bank scheduling does not remove the requirement to obey the upstream protocol's ordering rules.

## 10. Address mapping and bank interleaving

The system physical address is divided into fields that select byte, column, bank, row, rank, channel and sometimes subchannel. The exact bit mapping is configurable and product-specific.

A simplified illustrative map:

~~~text
Higher address bits
+---------+------+-------+--------+------------------+
| Channel | Rank | Bank  | Row    | Column / byte    |
+---------+------+-------+--------+------------------+
Lower address bits
~~~

**Do not assume one fixed bit slice.** Controllers may XOR selected address bits into bank/channel selection to reduce hot spots. A firmware-visible physical address map may be interleaved differently from the DRAM's physical organization.

### Mapping trade-offs

- Put more low-order bits into column/byte fields: improves sequential locality within a row.
- Interleave across banks: can expose bank-level parallelism.
- Interleave across channels: can increase aggregate bandwidth.
- Hash/XOR bank selection: may distribute power-of-two strides, but can make physical locality harder to reason about.
- Preserve large contiguous regions: can help DMA and page mapping, but may concentrate traffic depending on the mapping.

### Example

A 64-byte cache line is transferred through a 64-bit data channel as eight 8-byte beats. If the memory system uses a different data width or burst organization, the number of beats changes. The cache-line size is a CPU/fabric property; the DRAM burst is a DRAM protocol property. The controller maps between them.

## 11. Bandwidth calculations

Theoretical payload bandwidth is:

\[
BW_{\text{peak}} = \text{data rate (MT/s)} \times \frac{\text{data width (bits)}}{8}
\]

Because MT/s is million transfers per second, the result is MB/s when width is in bits.

### Example A — DDR5-6400, 64 data bits

\[
6400 \times 10^6 \times \frac{64}{8}
= 51.2 \times 10^9\ \text{bytes/s}
\]

**Peak theoretical payload bandwidth = 51.2 GB/s per 64-bit channel.**

A standard DDR5 DIMM exposes two independent 32-bit data subchannels (ECC bits, if present, are additional physical bits). Their combined payload width is 64 bits, so the aggregate theoretical payload bandwidth is still 51.2 GB/s at 6400 MT/s.

### Example B — LPDDR5, 32 data bits at 6400 MT/s

\[
6400 \times 10^6 \times \frac{32}{8} = 25.6\ \text{GB/s}
\]

This is the interface peak before command gaps, refresh, bus turnaround, bank conflicts and workload inefficiency.

### Sustained bandwidth

If a workload reaches 70% efficiency:

\[
BW_{\text{sustained}} = 51.2 \times 0.70 = 35.84\ \text{GB/s}
\]

The 70% factor is an example assumption, not a guaranteed controller efficiency. Real efficiency depends on traffic size, read/write mix, bank locality, refresh, QoS, bus turnaround and arbitration.

## 12. Latency calculation and row-hit comparison

Assume an *illustrative* DDR5-6400 operating point:

- CK frequency = 3200 MHz.
- tCK = 0.3125 ns.
- CL = 40 CK.
- tRCD = 40 CK.
- tRP = 40 CK.

Then:

\[
t_{CL} = 40 \times 0.3125 = 12.5\ \text{ns}
\]

\[
t_{RCD} = 40 \times 0.3125 = 12.5\ \text{ns}
\]

Approximate ACT-to-first-read-data timing, ignoring controller queueing, PHY pipeline and data-burst duration:

\[
t_{\text{ACT-to-data}} \approx t_{RCD}+t_{CL}=25\ \text{ns}
\]

A row conflict additionally needs PRECHARGE and tRP before the next ACTIVATE, subject to tRAS and other constraints. A row hit avoids the ACTIVATE/tRCD sequence.

**Interview trap:** DDR-6400 means 6400 million transfers per second, not a 6400 MHz CK clock. DDR transfers data on both edges of the relevant data timing scheme; CK is approximately half the data rate for conventional DDR signaling.

## 13. Refresh and its performance cost

A DRAM capacitor loses charge, so the controller must schedule refresh within the device's requirements. The standard defines refresh intervals and recovery times for each density, temperature range and mode.

Two useful parameters:

- **tREFI:** average interval between required refresh operations.
- **tRFC:** time for the relevant refresh operation to complete before the affected resource can be used again.

Illustrative overhead estimate:

Assume a refresh blocks a resource for 350 ns every 7.8 µs:

\[
\text{blocked fraction} \approx \frac{350\ ns}{7800\ ns}=4.49\%
\]

This is only a simplified upper-level estimate. Real refresh impact depends on all-bank versus per-bank/same-bank refresh, density, temperature, refresh postponement/pull-in rules, rank/bank parallelism and scheduler policy.

The controller should schedule refresh without violating deadlines. Delaying refresh too long is a correctness/reliability failure, not merely a performance issue.

## 14. DDR PHY training and initialization

A DRAM controller cannot simply start issuing random reads immediately after power-on. The device and PHY must be placed in a legal operating state and calibrated.

### Generic bring-up sequence

~~~mermaid
flowchart TD
  A["Power rails and reference clocks stable"] --> B["Assert / release DRAM reset per device requirements"]
  B --> C["Establish clock and device initialization state"]
  C --> D["Program mode registers and required calibration commands"]
  D --> E["PHY calibration and interface training"]
  E --> F["Controller enables normal traffic"]
  F --> G["Memory test, ECC initialization and error-counter baseline"]
  G --> H["Performance and margin characterization"]
~~~

The exact order is generation-specific. Use the device specification, PHY integration guide and board design guide; do not copy a sequence from a different DDR generation.

### Common training operations

| Training / calibration | Goal |
|---|---|
| PHY impedance / analog calibration | Set drive and termination behavior |
| DQS gate training | Identify the read-strobe burst window |
| Write leveling | Align write DQS timing to the DRAM clock relationship |
| Read DQS centering | Position the capture point inside the read eye |
| Per-bit deskew | Correct DQ-to-DQ skew within a byte lane |
| Vref training | Find a usable voltage reference margin |
| CA / command training | Calibrate command/address capture where supported |
| WCK-to-CK or WCK training | Align LPDDR write clocking where required |
| Write latency calibration | Confirm command-to-write-data alignment |

AMD's public Versal DDR4 guide is a useful implementation example: it documents initialization/calibration stages, write leveling, read-leveling and per-bit deskew. Its exact XPHY delay taps and sequence are AMD implementation details, not universal JEDEC values.

### Read-eye concept

~~~text
DQ valid window:
          <------------------------>
DQ:  -----xxxxxxxxxxxxxxxxxxxxxxxxxx-----
                  ^ 
             capture point

A good training algorithm sweeps delay and selects a point with margin
to both left and right edges, rather than selecting the first passing tap.
~~~

The center of the passing window is often a sensible target, but the actual algorithm must account for jitter, duty-cycle distortion, voltage/temperature drift and per-bit variation.

## 15. LPDDR architecture and low-power behavior

LPDDR is designed for energy-sensitive systems. Its architecture and commands evolve independently from DDR DIMM generations.

### Common LPDDR design themes

- Lower I/O voltage and power-oriented operating modes.
- More aggressive clock stop / clock gating opportunities.
- Self-refresh and deep low-power states.
- Dynamic frequency/voltage changes with required controller/PHY/device coordination.
- Generation-specific clocking, including WCK and read-strobe behavior in modern LPDDR interfaces.
- Package/board integration close to the SoC to control signal and power budgets.

### LPDDR versus DDR integration

| Topic | DDR DIMM-style platform | LPDDR SoC platform |
|---|---|---|
| Physical integration | Connector and DIMM/module routing | Usually soldered-down device or package integration |
| Data width | Commonly 64-bit payload channel, often with ECC extension | Platform-specific widths and channel organization |
| Power optimization | Power-down/self-refresh and platform power controls | Strong emphasis on low-power states and dynamic operating points |
| Board constraints | Module topology, fly-by routing and DIMM loading | Short high-speed routes, package escape and SoC/device co-design |
| Training | DDR-generation and topology-specific | LPDDR-generation and package/topology-specific |

### DVFS and low-power transition checklist

A frequency/voltage change is a coordinated subsystem operation, not merely a PLL register write:

1. Stop or quiesce new traffic as required.
2. Drain outstanding transactions and write queues to a safe point.
3. Put DRAM and controller into a supported transition state.
4. Change clocks/voltage in the specified sequence.
5. Update mode-register or PHY timing settings if required.
6. Recalibrate/retrain if the operating-point change requires it.
7. Exit low-power state and verify normal traffic.
8. Record transition status and error counters.

A failed sequence can corrupt data or hang the memory interface. Follow the exact vendor power-state protocol.

## 16. ECC, integrity and reliability

Do not confuse these three layers:

1. **DRAM on-die ECC:** corrects certain internal cell-array errors inside a DRAM die. DDR5 includes on-die ECC in its device architecture.
2. **Controller/system ECC:** protects the data path across DRAM devices and the controller interface, often with a sideband check-bit organization such as SECDED.
3. **CRC/parity and link protection:** protects selected command/data transfers or detects interface errors where the generation and configuration support it.

On-die ECC does not replace end-to-end system ECC. It cannot by itself detect/correct every error occurring on the external DQ path, controller, interconnect or system memory.

### RAS checklist

- Correctable and uncorrectable error counters.
- Address capture and syndrome reporting.
- Scrubbing policy and rate.
- Patrol scrub versus demand correction.
- ECC initialization before exposing memory to software.
- Error injection to validate reporting and interrupt behavior.
- Spare-row/repair behavior if supported by the device/platform.
- Thermal and refresh-rate interaction.

## 17. RowHammer and disturbance errors

In 2014, Kim et al. published an experimental study demonstrating that repeatedly activating a DRAM row can disturb data in physically adjacent rows. The work showed that scaling DRAM cells makes electrical coupling and charge disturbance a system-level security and reliability concern.

Mitigation is generation- and device-dependent and may include:

- Refresh-management features and targeted refresh mechanisms.
- Activation-rate tracking or throttling.
- Per-row activation management where supported.
- Firmware/OS allocation and isolation policies as defense in depth.
- ECC, which may detect/correct some resulting errors but is not a complete substitute for mitigation.

Do not claim a single universal RowHammer threshold. It depends on device, temperature, data pattern, timing and mitigation features.

## 18. Registers and software programming model

There is no universal DDR controller register map. A commercial controller may expose registers for:

- Controller enable and status.
- Timing parameters (tRCD, tRP, tRAS, tRFC, turnaround timing, etc.).
- Address mapping and interleave configuration.
- Rank/channel/device configuration.
- Refresh interval and policy.
- Mode-register command issue.
- PHY training start/status/result.
- ECC enable, syndrome, error count and injection.
- Performance counters, queue occupancy and QoS.
- Low-power entry/exit and DVFS coordination.

### Generic initialization pseudocode

~~~text
1. Confirm board power, reference clock, reset and DRAM configuration.
2. Configure controller geometry: channel/rank/bank/device width/density.
3. Program timing values from the selected DRAM speed bin.
4. Configure address mapping and interleaving.
5. Configure PHY drive, termination, Vref and clock settings.
6. Run the device initialization sequence required by the generation.
7. Run PHY training/calibration and inspect every stage's status.
8. Enable controller scheduling and refresh.
9. Initialize memory contents or ECC state before use.
10. Run walking-bit, address, burst and stress tests.
11. Enable production traffic and monitor counters.
~~~

**Do not copy register offsets or reset values from an unrelated controller.** The controller programming guide is the register-level contract.

## 19. Integration dependencies in a complete SoC

A DDR subsystem is connected to many other IPs:

- **AXI/NoC:** request acceptance, outstanding transactions, QoS and backpressure.
- **CPU caches:** cacheability, memory attributes, barriers and coherency.
- **DMA engines:** IOMMU, address translation, alignment and cache maintenance.
- **Security:** firewalls, secure/non-secure address regions, encryption and key handling where implemented.
- **Clock/reset:** controller clock, PHY reference clock, DRAM CK/WCK, reset sequencing and CDC.
- **Power manager:** self-refresh, retention, DVFS and wakeup.
- **Interrupt controller:** ECC error, training failure and controller fault interrupts.
- **Board/package:** topology, impedance, termination, skew, power integrity and thermal behavior.
- **Firmware:** memory training, topology discovery, ECC initialization and memory map publication.

### Bandwidth budgeting example

Suppose a camera ISP, display scanout and CPU/DSP share one DDR channel:

| Client | Average payload demand |
|---|---:|
| Camera write stream | 3.0 GB/s |
| Display read stream | 2.5 GB/s |
| DSP/CPU traffic | 4.0 GB/s |
| Total | 9.5 GB/s |

If the channel is 64-bit DDR5-6400, peak theoretical bandwidth is 51.2 GB/s. At an assumed 70% sustained efficiency, the approximate available bandwidth is 35.84 GB/s. Average demand is below that estimate, but worst-case bursts, refresh, read/write turnarounds and latency deadlines still need validation. Average bandwidth alone cannot prove display/camera deadlines.

## 20. Pre-silicon verification plan

### Protocol and controller tests

- Legal ACT/PRE/READ/WRITE sequencing.
- All relevant tX timing constraints and boundary cases.
- Row hit, closed-bank miss and row conflict.
- Bank-group and cross-rank constraints.
- Refresh due under continuous traffic.
- Read/write turnarounds and write draining.
- Burst length, masks, byte lanes and unaligned upstream accesses.
- AXI IDs, outstanding requests, response ordering and backpressure.
- Address mapping across channels/ranks/banks/rows.
- ECC correctable/uncorrectable error injection.
- Power-down/self-refresh entry/exit and DVFS transitions.
- Reset during idle, traffic and error recovery.
- Arbitration fairness and QoS under contention.

### PHY and board-level validation

- Training convergence across voltage/temperature corners.
- Read/write eye margin and per-bit skew.
- Vref and delay sweeps.
- Jitter and duty-cycle sensitivity.
- Simultaneous switching and crosstalk.
- Package/PCB extraction and signal-integrity simulation.
- Power-integrity and simultaneous-switching-noise analysis.

### Assertions and checkers to consider

~~~systemverilog
// Conceptual properties; adapt to the controller's actual internal events.
assert property (act_same_bank |-> bank_was_idle);
assert property (read_cmd |-> bank_row_is_active);
assert property (refresh_cmd |-> all_required_banks_are_idle);
assert property (write_response |-> write_data_was_accepted);
assert property (ecc_uncorrectable |-> error_status_is_latched);
~~~

These are conceptual examples, not drop-in assertions. The real checker needs the correct bank state, timing counters, command encoding and reset semantics.

## 21. Post-silicon bring-up and characterization

### Recommended staged sequence

1. Verify board rails, reference clocks, reset and DRAM assembly.
2. Boot using a conservative supported data rate.
3. Read controller/PHY initialization status and training logs.
4. Confirm each channel/rank/byte lane trained.
5. Run small memory tests in reserved regions.
6. Run address-line, walking-one/zero and burst tests.
7. Enable ECC and validate error injection/reporting if supported.
8. Sweep data rate and timing margins.
9. Run simultaneous CPU, DMA, camera, display and accelerator traffic.
10. Characterize voltage/temperature corners and long-duration retention/stress behavior.

### Failure-signature matrix

| Symptom | Possible causes | First debug action |
|---|---|---|
| No training completion | Reset/clock/power order, wrong topology, CA issue | Read first failing training stage and rails/clocks |
| One byte lane fails | DQ/DQS swap, lane mapping, solder/route, delay margin | Inspect lane-level training and continuity |
| Read failures only | Read DQS gate/centering, Vref, DQ deskew | Capture read-eye sweep and failing bit positions |
| Write failures only | Write leveling, DQS-to-DQ alignment, write latency | Inspect write training and pattern readback |
| Failures only at high rate | SI, jitter, skew, Vref, ODT, power integrity | Compare margin at lower rate and inspect eye data |
| Row-pattern-dependent failures | Address mapping, timing, disturbance or test algorithm | Correlate physical bank/row and test pattern |
| Failure after hot soak | Refresh policy, thermal margin, retention | Check temperature, refresh mode and error trend |
| Intermittent under camera/display load | QoS, queue pressure, read/write turnaround, SI/power noise | Correlate performance counters with lane errors |
| ECC errors increase over time | Device, timing, thermal, power or SI margin | Capture syndrome/address and temperature/voltage |

### Useful observability

- Per-channel and per-rank training results.
- Per-byte/per-bit delay taps and Vref values.
- Read/write eye margin.
- Correctable/uncorrectable ECC counters and addresses.
- Refresh and power-state counters.
- Read/write queue occupancy.
- Row-hit, row-conflict and bank-stall counters.
- AXI outstanding count, latency histogram and QoS stall cycles.
- Temperature, voltage and PLL lock status.

## 22. Worked silicon design review

**Requirement:** connect a 128-bit AXI port clocked at 500 MHz to a DDR5-6400 64-bit data channel.

AXI-side theoretical input bandwidth:

\[
128/8 \times 500\text{ MHz} = 8\text{ GB/s}
\]

DDR peak:

\[
6400\text{ MT/s} \times 64/8 = 51.2\text{ GB/s}
\]

The AXI port's peak is 8 GB/s, so it cannot saturate the DDR channel alone. However, multiple AXI ports may contend for it.

Review questions:

- Are AXI bursts coalesced efficiently into DRAM bursts?
- Is address mapping distributing camera and display streams across banks?
- Can writes be drained without violating read latency deadlines?
- Does the controller preserve required ordering and barriers?
- Is the channel width really 64 payload bits, or is ECC/sideband width being confused with payload width?
- Does the PHY meet timing across the full board/package and PVT range?
- Are refresh and power-state transitions visible in performance counters?

## 23. Common RTL and integration bugs

1. **Wrong geometry:** density, device width, rank count or bank-group configuration mismatches the actual device.
2. **Wrong timing units:** programming CK cycles where the register expects time, or vice versa.
3. **Incomplete timing scoreboard:** a rare bank-group or turnaround constraint is omitted.
4. **Refresh starvation:** continuous traffic prevents refresh from meeting the required cadence.
5. **Bad address map:** a stride causes channel/bank hot spots or unexpected aliasing.
6. **AXI response bug:** IDs or burst boundaries are lost during reordering.
7. **Read/write FIFO underflow/overflow:** controller and PHY timing assumptions disagree.
8. **Training false pass:** first passing delay tap is chosen instead of a robust margin point.
9. **ECC initialization gap:** software accesses memory before ECC state is initialized.
10. **Unsafe DVFS:** clocks or voltage change before outstanding traffic and DRAM state are quiesced.
11. **Reset-domain issue:** controller resets but PHY/device remains in a different state.
12. **SI mistaken for RTL:** failures correlate with speed, lane, voltage or temperature and are incorrectly debugged only at the command scheduler.

## 24. Interview questions

1. Why does DRAM need refresh while SRAM does not?
2. What is the difference between a row, bank and bank group?
3. Walk through ACTIVATE → READ → PRECHARGE.
4. What is the difference between tRCD, CL and tRP?
5. Why does a row hit usually have lower latency than a row conflict?
6. What is the purpose of tFAW?
7. Why are bank interleaving and channel interleaving useful?
8. Why is DDR-6400 not a 6400 MHz CK clock?
9. What is the role of DQS?
10. Explain write leveling versus read leveling.
11. What is the difference between DRAM on-die ECC and controller-side ECC?
12. Why can a design pass at low data rate and fail at high data rate?
13. How does a controller schedule refresh under heavy traffic?
14. What is the difference between DDR and LPDDR system integration?
15. When would HBM be preferred over DDR or LPDDR?

## 25. Hands-on exercises

### Exercise 1 — Capacity and address width

A DRAM device is organized as 8 banks, 16K rows per bank, 1K columns per row and x16 data width. Calculate raw capacity and the minimum address bits for a simple linear organization. State what additional information is needed to determine byte-address mapping.

### Exercise 2 — Bandwidth

Calculate peak bandwidth for:
- 32-bit LPDDR interface at 6400 MT/s.
- 64-bit DDR interface at 5600 MT/s.
- Four independent HBM stacks, each delivering 2 TB/s (use the HBM4 standard maximum example only as a theoretical aggregate).

### Exercise 3 — Timing

Given tCK = 0.3125 ns, tRCD = 40 cycles, CL = 40 cycles and tRP = 40 cycles, calculate ACT-to-first-data and approximate row-conflict extra delay. Explain why controller queueing is not included.

### Exercise 4 — Scheduler

Create a trace with:
- Bank 0 Row A read,
- Bank 1 Row C read,
- Bank 0 Row B read,
- Bank 0 Row A read again.

Compare FCFS and row-hit-aware scheduling. Discuss fairness and the latency of the third request.

### Exercise 5 — Refresh

Use a hypothetical tRFC of 350 ns and tREFI of 7.8 µs. Calculate the simplified blocked-time fraction. Explain why real refresh overhead can differ.

### Exercise 6 — Bring-up debug

A board passes memory tests at 3200 MT/s but fails at 6400 MT/s only on byte lane 3 at high temperature. Build a prioritized debug plan using training logs, DQ/DQS eye, Vref, SI, power integrity and temperature correlation.

## 26. Practical checklist for an SoC integration review

- [ ] Exact DRAM part number, density, width, speed bin and temperature grade recorded.
- [ ] Correct JEDEC generation and revision selected.
- [ ] Controller and PHY revisions are compatible.
- [ ] Channel/rank/bank/row/column geometry matches the device and board.
- [ ] Timing values are generated from the exact part and operating point.
- [ ] Refresh policy covers temperature and density requirements.
- [ ] PHY training stages and failure status are observable.
- [ ] Board/package routing and SI/PI simulations are reviewed.
- [ ] ECC initialization occurs before software exposure.
- [ ] AXI IDs, ordering, address map and QoS are validated.
- [ ] Cache maintenance/coherency and IOMMU behavior are defined for DMA.
- [ ] Low-power and DVFS transitions are sequenced and tested.
- [ ] Performance counters and lane-level margin data are available in silicon.

## 27. References and source discipline

### Normative standards

- [JEDEC standards portal](https://www.jedec.org/standards-documents) — obtain the exact current revision of JESD79 (DDR), JESD79-2/3/4/5 (DDR2/3/4/5), JESD209 family (LPDDR), and JESD270 family (HBM). The standard is normative for its defined device generation.
- [JEDEC home and technology focus areas](https://www.jedec.org/) — identifies DDR, HBM and LPDDR as JEDEC main/low-power memory areas.
- [JESD79-5 DDR5 standard, July 2020](https://studylib.net/doc/27960491/jesd79-5) — publicly indexed copy for study; prefer the JEDEC portal for the controlled official document.

### Historical milestones and design rationale

- [DDR5 standard announcement, 14 July 2020](https://www.design-reuse-embedded.com/news/202007056/jedec-publishes-new-ddr5-standard-for-advancing-next-generation-high-performance-computing-systems/) — JEDEC announcement reproduced by Design & Reuse.
- [DDR4 standard announcement, 25 September 2012](https://www.engadget.com/2012-09-26-final-ddr4-specification.html) — includes the JEDEC press-release text and initial DDR4 goals.
- [DDR3 standard announcement, 26 June 2007](https://www.engadget.com/2007-06-26-jedec-finally-unveils-official-ddr3-standards.html) — contemporary report on the standard release and features.
- [LPDDR5 announcement, 19 February 2019](https://www.techpowerup.com/252798/jedec-updates-standard-for-low-power-memory-devices-lpddr5) — reproduces the JEDEC announcement for JESD209-5.
- [LPDDR6 JESD209-6 publication metadata](https://www.intertekinform.com/en-au/standards/jedec-jesd209-6-2025-1408791_saig_jedec_jedec_3649933/) — lists July 2025 publication; consult JEDEC for the standard itself.
- [HBM4 JESD270-4 announcement, 16 April 2025](https://www.eetasia.com/jedec-releases-jesd270-4-hbm4-standard/) — reproduces JEDEC's HBM4 release.
- [HBM4 standard document, JESD270-4, April 2025](https://img.antpedia.com/standard/files/pdfs_ora/20250520/JEDEC%20JESD270-4-2025.pdf) — publicly indexed copy; use the official JEDEC document for compliance work.

### Vendor implementation examples

- [Micron DDR5 SDRAM product/feature comparison](https://in.micron.com/products/memory/dram-components/ddr5-sdram) — useful for DDR4/DDR5 device feature comparison; not a replacement for the standard.
- [Micron DDR5 new-features white paper](https://in.micron.com/content/dam/micron/global/public/products/white-paper/ddr5-new-features-white-paper.pdf) — refresh and device-feature implementation context.
- [AMD Versal DDR4 memory controller guide: write-latency calibration](https://docs.amd.com/r/en-US/pg353-versal-acap-soft-ddr4-mem-ip/Write-Latency-Calibration) — implementation-specific training example.
- [AMD Versal DDR4 memory controller guide: read leveling](https://docs.amd.com/r/en-US/pg353-versal-acap-soft-ddr4-mem-ip/Read-Leveling-Calibration-Overview) — DQ deskew and read-DQS centering example.
- [AMD Versal initialization and calibration sequence](https://docs.amd.com/r/en-US/pg353-versal-acap-soft-ddr4-mem-ip/Memory-Initialization-and-Calibration-Sequence) — public example of staged calibration.
- [AMD DDR4 PCB guidelines](https://docs.amd.com/r/en-US/pg150-ultrascale-memory-ip/PCB-Guidelines-for-DDR4) — board design constraints and integration guidance.

### Reliability and security research

- [Kim et al., “Flipping Bits in Memory Without Accessing Them,” ISCA 2014](https://istc-cc.cmu.edu/publications/papers/2014/kim-isca14_abs.shtml) — primary research abstract describing DRAM disturbance errors and RowHammer.

**How to use these references:** JEDEC defines normative device behavior. A Micron data sheet defines the selected component's electrical/timing requirements. An AMD guide documents one controller/PHY implementation. Academic research explains observed reliability phenomena. Keep these evidence classes separate in design reviews.

## 28. Key takeaways

- DRAM is dense because a 1T1C cell is compact, but its charge must be refreshed and its analog signal must be sensed.
- ACTIVATE opens a row; READ/WRITE selects columns; PRECHARGE closes the row; REFRESH restores charge.
- Timing parameters form a distributed set of bank-, bank-group-, rank- and channel-level legality constraints.
- The controller optimizes traffic while preserving protocol timing, upstream ordering and QoS.
- The PHY is responsible for high-speed electrical timing and training; the board/package is part of the channel.
- Peak bandwidth is easy to calculate. Sustained bandwidth and bounded latency require workload-aware validation.
- LPDDR, DDR and HBM serve different power, packaging, capacity and bandwidth goals.
- Training logs, eye margin, ECC telemetry and controller counters are essential post-silicon observability.

---

**Next session — Lesson 13: DMA Architecture.** We will follow descriptors and payloads from software rings through AXI/NoC, IOMMU, cache coherency, memory arbitration and interrupts, with practical camera/audio/display/DSP examples.
