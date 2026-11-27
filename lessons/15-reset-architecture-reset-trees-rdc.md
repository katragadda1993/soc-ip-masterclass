# Lesson 15 — Reset Architecture: From POR to Reset Trees, RDC and Silicon Recovery

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 15 of 38  
**Duration:** 60 minutes  
**Level:** RTL architecture → SoC integration → CDC/RDC signoff → post-silicon debug  
**Prerequisites:** Lessons 01–14; sequential logic, clock domains, PLL lock, CDC, AXI transactions, DMA, power domains and boot flow.

> **Central engineering question:** How can a chip enter a known, safe state when clocks may not yet exist, power rails may be ramping, transactions may be in flight, and different subsystems must reset or recover independently—without creating metastability, orphaned transactions, security gaps or an unrecoverable boot?

This lesson treats reset as a system architecture and recovery protocol, not merely a wire connected to every flip-flop. A production reset architecture defines reset sources, domain membership, assertion and release behavior, sequencing, isolation, retained state, reset-cause capture, software contracts, test behavior and verification evidence.

**Scope note:** There is no single universal AMBA reset protocol or standard reset-manager register map for all SoCs. Reset pins and semantics are defined by the relevant IP, device, board and integration specifications. Register examples in this lesson are explicitly identified as implementation examples.

---

## 1. One-hour learning plan

| Time | Topic | Expected outcome |
|---:|---|---|
| 0–5 min | Reset purpose, taxonomy and history | Explain why reset is a system contract |
| 5–13 min | Reset domains and controller architecture | Draw a reset tree for a multi-domain SoC |
| 13–21 min | Assertion, deassertion and synchronizers | Explain recovery/removal and safe release |
| 21–30 min | Sequencing, quiesce, isolation and RDC | Design a safe local-reset sequence |
| 30–37 min | Registers, software and reset-cause capture | Define the hardware/software contract |
| 37–44 min | Clock, power, retention and security interaction | Coordinate reset with power states |
| 44–53 min | RTL, assertions and verification | Build a reset verification plan |
| 53–60 min | Silicon debug, calculations and exercises | Diagnose reset failures systematically |

## 2. Why reset is harder than it looks

Reset establishes a known architectural state before normal operation. That does **not** mean every storage bit in the chip must always be cleared.

A reset design must answer at least these questions:

1. Which event caused the reset?
2. Which IP blocks are affected?
3. Which clocks are available while reset is asserted?
4. Can reset assert when a clock is stopped?
5. How is reset released relative to each destination clock?
6. What happens to requests and responses already in flight?
7. Which registers, memories, counters and debug records survive?
8. How are outputs made safe while a block is reset?
9. Can a block reset independently without corrupting neighboring blocks?
10. How does software distinguish a cold boot from watchdog recovery or low-power exit?

A design that resets all logic with one global signal can be simple, but it can also create a high-fanout physical net, destroy useful debug state, reset always-on logic unnecessarily and prevent local recovery. A design with many independently controlled resets is more flexible, but creates reset-domain crossings (RDC), ordering dependencies and a larger verification problem.

**Engineering interpretation:** The reset architecture evolves with the SoC. A small single-clock controller may need one external reset and a small number of local resets. A modern SoC with multiple PLLs, always-on logic, power-gated islands, CPUs, DSPs, PHYs, safety logic and debug retention needs a reset *manager* and an explicit reset-domain specification.

## 3. Historical and design-rationale backstory

The dates below are documented milestones, not a claim that a particular source invented reset.

### 3.1 Early microprocessors: an explicit way to start from a known state

- **1975 — Intel 8080 Microcomputer Systems User's Manual:** The manual documents the processor's clocked machine cycles and external control interface. Early microprocessor systems already needed board-level power-up and initialization circuitry around the CPU. The manual is a useful period reference for the single-processor era. [Intel 8080 manual (1975 scan)](https://bitsavers.trailing-edge.com/components/intel/MCS80/98-153B_Intel_8080_Microcomputer_Systems_Users_Manual_197509.pdf)
- **1978/1979 — Intel 8086 family documentation:** The 8086 RESET input is documented as an initialization mechanism. The user manual specifies a high pulse longer than four CLK cycles, an internal reset sequence, and a start address at physical location FFFF0h. It also documents a power-up delay requirement for that processor. These are **8086-specific** requirements, not universal reset rules. [Intel 8086 User Manual](https://manuzoid.com/manuals/gqvrn-Intel%208086%20User%20manual)

The early design problem was straightforward: after power becomes valid, the processor must not execute from arbitrary state. The board or system must hold it in reset until the required supply and clock conditions are met.

### 3.2 Reset assertion versus release became an explicit RTL design topic

- **March 2002 — SNUG reset-design paper:** Clifford Cummings and Don Mills presented reset design techniques that discussed the practical trade-offs of synchronous and asynchronous resets.
- **September 2003 — SNUG Boston Rev. 1.2:** Cummings, Mills and Steve Golson published an updated paper comparing both reset styles and explaining why asynchronous reset release must be treated carefully. The paper also discusses reset-tree distribution, DFT and RTL coding style. It is an influential engineering reference, not a normative standard. [Paper PDF](https://trilobyte.com/pdf/CummingsSNUG2003Boston_Resets_rev1_2.pdf) · [2003 conference record](https://trilobyte.com/2003/09/snug03_boston/)

### 3.3 Multi-domain SoCs created a separate RDC verification problem

- **2018 — DVCon paper, “A Specification-Driven Methodology for the Design and Verification of Reset Domain Crossing Logic”:** The authors describe how large SoCs with many asynchronous reset domains create RDC paths and propose a requirements-driven approach to reset ordering, isolation, synchronization and verification. [DVCon paper](https://dvcon-proceedings.org/wp-content/uploads/a-specification-driven-methodology-for-the-design-and-verification-of-reset-domain-crossing-logic.pdf)
- **2019 — RDC methodology article:** A later overview describes specification-driven RDC planning for complex SoCs. [Verification Academy overview](https://verificationacademy.com/topics/reset-domain-crossing/a-specification-driven-methodology-for-the-design-and-verification-of-reset-domain-crossing-logic/)

The market and technology context is the growth of multi-clock, multi-power-domain SoCs. Independent IP teams can deliver blocks with different reset behavior; integration must prove that signals crossing those reset boundaries remain safe.

### 3.4 Modern implementation examples

- **OpenTitan Earl Grey:** Its public reset-manager documentation describes cascaded POR, lifecycle, system/debug and module reset trees; reset-cause registers; crash-dump capture; reset consistency checking; and coordination with power management. [Theory of operation](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/theory_of_operation.html)
- **AMD Versal methodology:** The 2026.2 UG1387 page, released **2026-11-20**, recommends synchronous resets for many programmable-logic use cases and explains the routing, memory-corruption and recovery/removal considerations of asynchronous resets. This is vendor- and device-family guidance, not a universal ASIC rule. [UG1387 — Synchronous Reset vs. Asynchronous Reset](https://docs.amd.com/r/en-US/ug1387-acap-hardware-ip-platform-dev-methodology/Synchronous-Reset-vs.-Asynchronous-Reset)

### 3.5 Why the architecture changed

| Earlier design pressure | Modern SoC pressure | Resulting design response |
|---|---|---|
| Start one CPU after power-up | Start many independent clock and power domains | Reset manager and domain tree |
| One external reset pin | POR, watchdog, software, debug, security and power requests | Reset-source arbitration and cause capture |
| Reset the whole chip | Recover one failed peripheral or accelerator | Selective module resets |
| One clock relationship | Independent or stoppable clocks | Per-clock-domain reset release |
| Few retained diagnostics | Diagnose watchdogs and security escalations | Always-on reset-cause and crash-dump storage |
| Few reset boundaries | Hundreds of IP boundaries and power-intent cells | Static and dynamic RDC verification |

**Design-rationale interpretation:** Reset partitioning trades simplicity for recovery granularity. More reset domains can reduce recovery scope and preserve useful state, but each new boundary creates sequencing, isolation, status and verification obligations.

## 4. Reset terminology and taxonomy

### 4.1 Reset source types

| Reset source | Typical trigger | Typical scope (implementation-dependent) |
|---|---|---|
| POR — power-on reset | Power-good/voltage supervisor | Entire device or root reset hierarchy |
| External system reset | Board reset pin or PMIC | Device-defined system scope |
| Brownout reset | Supply falls below operating threshold | Domains affected by supply validity |
| Watchdog reset | Software fails to service watchdog | CPU/system or selected subsystem |
| Software reset | Firmware writes reset request | Whole system or selected module |
| Debug reset | Debug module requests reset | CPU/system while retaining debug state |
| Security escalation reset | Alert or lifecycle policy | Security-defined system scope |
| Module reset | Peripheral recovery request | One peripheral or IP block |
| Power-domain reset | Power island was switched off/on | All non-retained state in that island |
| Scan/test reset | DFT/test controller | Test-defined logic subset |
| PHY/link reset | Protocol or PHY recovery | PHY, PCS/MAC or link state machine as specified |

Do not assume “warm reset” has one universal meaning. One implementation may preserve the always-on domain, another may preserve debug state, and another may reset a CPU while retaining DRAM contents. The reset contract must state exactly what survives.

### 4.2 Reset signal conventions

- Active-low reset signals are commonly named with an _n suffix, for example **por_n** or **periph_rst_n**.
- Active-high reset signals may be named **reset** or **rst**.
- **reset_req** is a request; it is not necessarily the reset output.
- **reset_ack** or **reset_done** reports completion of a handshake or sequence.
- **reset_status** reports the current state.
- **reset_cause** records why a reset occurred.

Polarity is a naming and electrical/interface convention. Never infer polarity from a signal name without checking the IP specification.

### 4.3 Reset domain versus clock domain

A **clock domain** is defined by a clock and its timing relationship. A **reset domain** is the set of sequential elements that can be reset independently or by a common reset event.

They are not identical.

- One clock domain can contain multiple reset domains.
- One reset tree can produce separate synchronized leaves for several clock domains.
- A power domain can contain multiple clock and reset domains.
- An always-on block can share a reset source with a non-always-on block but require different reset leaves.

This is why CDC analysis alone does not prove reset safety.

## 5. System-level reset architecture

### 5.1 Reference architecture

The following is a generic architecture. The exact reset sources, hierarchy and signals are implementation-specific.

~~~mermaid
flowchart TD
  PGOOD[PMIC power-good / POR supervisor] --> AON[Always-on reset manager]
  EXT[External reset pin] --> AON
  WDT[Always-on watchdog] --> AON
  SEC[Security alert / escalation] --> AON
  SW[Software reset request] --> AON
  DBG[Debug reset request] --> AON
  PWR[Power manager] <--> AON
  PLL[PLL lock / clock-valid] --> AON
  AON --> ROOT[Root reset tree]
  ROOT --> CPU_SYNC[CPU reset synchronizer]
  ROOT --> NOC_SYNC[NoC reset synchronizer]
  ROOT --> DDR_SYNC[DDR controller reset synchronizer]
  ROOT --> DSP_SYNC[DSP reset synchronizer]
  ROOT --> PERI_SYNC[Peripheral reset synchronizer]
  CPU_SYNC --> CPU[CPU subsystem]
  NOC_SYNC --> NOC[Interconnect / NoC]
  DDR_SYNC --> DDR[DDR controller]
  DSP_SYNC --> DSP[DSP / accelerator]
  PERI_SYNC --> PERI[UART / I2C / SPI / audio]
  AON --> CAUSE[Always-on reset cause / crash dump]
  AON --> STATUS[Reset status / consistency monitor]
~~~

A reset manager typically contains:

1. Reset-source qualification and filtering.
2. Reset request arbitration and policy.
3. Reset sequencing state machine.
4. Root reset generation.
5. Per-clock-domain reset synchronizers.
6. Per-module reset controls.
7. Reset status and cause registers.
8. Reset consistency/error detection.
9. Crash-dump capture and always-on retention.
10. Interfaces to clock and power managers.

### 5.2 Example reset-domain inventory

| Domain | Example contents | Clock | Reset behavior |
|---|---|---|---|
| AON | POR monitor, RTC, watchdog, reset manager | 32.768 kHz / AON clock | Survives warm resets |
| CPU | CPU cluster, private cache control | CPU PLL | System reset or CPU-specific reset |
| NoC | Router and bridge control | NoC clock | Reset after traffic is quiesced or system abort policy is applied |
| DDR | Controller, PHY control and training state | DDR UI / PHY clocks | Reset according to DRAM and PHY recovery requirements |
| DSP | DSP core, local DMA and accelerator control | DSP clock | Local or system reset |
| Peripheral | UART, I2C, SPI, audio control | Peripheral clock | Individual software reset where allowed |
| Debug | Debug transport and trace retention | Debug/AON clock | May be retained during selected system resets |
| Security | Key manager, lifecycle and alert logic | Security/AON clocks | Policy-controlled; not casually software-resettable |

This table is an architectural example, not a reference SoC specification.

## 6. Reset assertion and deassertion

### 6.1 Synchronous reset

A synchronous reset is sampled by the active clock edge.

**Advantages**
- No asynchronous reset pin is required on every destination register.
- Reset behavior is naturally aligned to the destination clock.
- It may map efficiently to FPGA or standard-cell structures.
- Reset routing can sometimes be localized as data-path control.

**Trade-offs**
- The clock must be running to observe the reset.
- A short pulse between active edges can be missed.
- Reset must satisfy setup/hold requirements at the sampling edge.
- A synchronous reset cannot by itself force state when its clock is absent.

### 6.2 Asynchronous reset

An asynchronous reset can force a resettable sequential element without waiting for a clock edge.

**Advantages**
- Can establish a safe state while clocks are stopped or not yet stable.
- Useful for POR and emergency reset assertion.
- Can reset state during clock startup failure.

**Trade-offs**
- Reset release is asynchronous unless explicitly synchronized.
- Reset recovery/removal timing must be met.
- High-fanout asynchronous reset routing can be physically expensive.
- Resettable cells may have area, timing or power trade-offs.
- Resetting control signals that drive SRAM/BRAM ports can create unintended memory operations or corruption.

### 6.3 Common pattern: asynchronous assertion, synchronous deassertion

A widely used pattern is:

- Assert the reset asynchronously so the destination enters reset even if its clock is stopped.
- Release the reset through a synchronizer clocked by the destination clock.
- Use a separate synchronizer for each unrelated destination clock domain.

This is a design pattern, not a universal mandate. Some blocks use synchronous reset only, some use dedicated reset cells, and some macros have their own reset and initialization requirements. Follow the target library and IP integration guide.

### 6.4 Two-stage reset synchronizer example

The following educational SystemVerilog sketch asynchronously asserts a two-stage chain and shifts in ones after the destination clock runs. The output is released only after the synchronizer has advanced.

~~~systemverilog
module reset_sync #(
  parameter int STAGES = 2
) (
  input  logic clk,
  input  logic arst_n,
  output logic srst_n
);

  logic [STAGES-1:0] sync_ff;

  initial begin
    if (STAGES < 2)
      $error("STAGES must be at least 2");
  end

  always_ff @(posedge clk or negedge arst_n) begin
    if (!arst_n)
      sync_ff <= '0;
    else
      sync_ff <= {sync_ff[STAGES-2:0], 1'b1};
  end

  assign srst_n = sync_ff[STAGES-1];

endmodule
~~~

**Implementation cautions**
- The first stage can become metastable if asynchronous release violates recovery/removal; the second stage provides settling time.
- Use library- and tool-supported synchronizer attributes or cells where appropriate. Attribute spelling is vendor-specific.
- Keep the chain physically close and protect it from logic optimization that changes its intended structure.
- Do not add arbitrary combinational logic after the synchronizer output.
- The destination clock must be valid before reset is expected to release.
- A stopped clock means the synchronizer cannot complete release.
- Verify reset pulse width against the actual library cell and system source requirements.
- Scan insertion and DFT constraints must be reviewed for reset synchronizer flops.
- Do not blindly copy this RTL into a safety-critical design without library, STA, CDC/RDC and DFT review.

### 6.5 Reset release timing calculation

Assume:
- Destination clock = 200 MHz.
- Clock period = 1 / 200 MHz = 5 ns.
- Synchronizer depth = 2 stages.
- The asynchronous reset is released just before a destination edge.

The output generally becomes deasserted after two destination sampling edges, approximately 5–10 ns depending on phase and edge convention. For a conservative architecture budget, count the synchronizer stages and include clock-start and distribution latency separately.

If the same two-stage release is used on a 32.768 kHz AON clock, one period is about 30.52 µs, so two cycles are about 61.04 µs. This demonstrates why a reset manager should not assume all domains release at the same wall-clock time.

The exact latency depends on reset polarity, implementation, clock phase, synchronizer depth and output distribution. Do not treat the simple arithmetic as a universal guarantee.

## 7. Reset sequencing as a dependency graph

Reset release is not simply “deassert every reset at once.” It must respect the dependencies of the system.

### 7.1 Example power-on release sequence

~~~mermaid
flowchart TD
  A[Power rails ramp] --> B[Power-good / POR qualification]
  B --> C[Always-on domain released]
  C --> D[Reference oscillator stable]
  D --> E[PLL enable and lock qualification]
  E --> F[Clock mux / divider configured]
  F --> G[Reset synchronizers release]
  G --> H[Interconnect and SRAM controllers ready]
  H --> I[DDR controller and PHY initialization / training]
  I --> J[CPU released to boot ROM]
  J --> K[Boot firmware initializes peripherals]
  K --> L[Normal traffic enabled]
~~~

This is a generic dependency example. Some SoCs release the CPU earlier and let boot firmware initialize DDR; others use ROM or a boot controller to sequence memory before general CPU access. The actual boot architecture decides the ordering.

### 7.2 Assertion order is not always the reverse of release order

A common safe local-reset sequence is:

1. Stop accepting new work.
2. Quiesce request sources.
3. Drain or explicitly abort in-flight transactions according to the interface contract.
4. Mask or capture interrupts and error status.
5. Isolate outputs to neighboring domains.
6. Assert the local reset.
7. Hold reset for the specified interval.
8. Release reset through the destination-domain synchronizer.
9. Wait for the block's ready/initialization indication.
10. Reprogram required state.
11. Remove isolation and re-enable traffic.

For a full chip POR, the root reset may assert broadly and immediately. For a local reset, the design may need to quiesce consumers before resetting a provider. The correct sequence comes from the dependency graph and failure policy, not from a universal reverse-order rule.

### 7.3 Reset during an AXI transaction

AXI has no generic transaction-cancel handshake that lets a manager simply forget a request after it has been accepted. If a DMA master is reset after its address is accepted but before the response returns, the interconnect and subordinate may still have outstanding state.

A system must choose and verify one of these policies:

- **Drain:** Stop new requests and wait for all accepted transactions to complete.
- **Fence/quiesce:** Use an integration-level handshake to stop new traffic and confirm the fabric is idle for that initiator.
- **Coordinated abort/reset:** Reset the relevant interface participants together under a documented system-level reset contract.
- **Isolation and recovery:** Prevent malformed or stale outputs from escaping, then reinitialize all participants that share transaction state.

Do not assume that resetting only the DMA's internal state also resets the interconnect's ID tracker, response FIFO or memory-controller queue.

## 8. Reset-domain crossing (RDC)

### 8.1 What is an RDC?

An RDC exists when a signal crosses between sequential elements that can be reset independently or at different times. The clocks may be identical.

Example:
- Source block A is not reset.
- Destination block B is reset.
- A control signal from A remains high while B is in reset.
- When B releases reset, it may interpret the still-high signal as a new request.
- Conversely, if A is reset while B remains active, A's output can change asynchronously and B may sample the transition near a clock edge.

Potential failures:
- Metastability.
- False pulse or missed pulse.
- Stale request replay.
- FSM state mismatch.
- One side believes a handshake completed while the other forgot it.
- FIFO pointers or valid bits become inconsistent.
- A reset block drives X/unknown or invalid values into a live destination.
- A transaction is orphaned and the system waits forever.

### 8.2 RDC protection methods

| Method | When useful | Main requirement |
|---|---|---|
| Reset ordering | Source and destination can be sequenced | Define parent/child reset release rules |
| Data isolation | Destination must ignore a resetting source | Isolation control must itself be safe |
| Handshake reinitialization | Request/ack state spans domains | Both sides must agree on a fresh epoch/state |
| Synchronizer | Single-bit asynchronous status | Does not solve multi-bit coherence by itself |
| Async FIFO reset protocol | Data crosses clocks and reset domains | Pointer/flag reset and release must be coordinated |
| Valid/ready quiesce | Streaming or bus interfaces | Stop new transfers and drain accepted work |
| Epoch/generation tag | Stale messages may survive local reset | Both sides must agree on epoch update and wrap behavior |
| Parent/leaf consistency monitor | Large reset hierarchy | Detect leaf reset without authorized cause |

A two-flop CDC synchronizer is not automatically an RDC solution. If the source can reset independently, the protocol must also define what a reset-induced transition means.

### 8.3 RDC example: request/acknowledge handshake

A robust handshake needs an explicit reinitialization rule:
- During reset, the request and acknowledge state are driven to a known idle representation.
- The destination ignores the source until both sides report initialization complete.
- A new request is accepted only after the handshake has re-armed.
- If one side resets independently, a reset/epoch handshake prevents a stale high request from being interpreted as a new event.

For multi-bit payloads, synchronize a control/valid protocol or use an asynchronous FIFO; do not independently synchronize each data bit and assume the bus remains coherent.

### 8.4 Reset consistency monitoring

OpenTitan documents reset consistency checks that compare parent and leaf reset indications, allow for synchronization latency and report an error if a leaf reset asserts without a corresponding authorized cause. It also documents multiple reset trees and per-clock-domain output leaves. These are concrete implementation choices, not universal protocol requirements. [OpenTitan reset theory](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/theory_of_operation.html)

## 9. Reset interaction with clocks, power and retention

### 9.1 Clock manager dependency

- A reset synchronizer requires its destination clock to run before it can release reset.
- PLL lock alone may not prove the clock is safe for every consumer; the clock manager may need frequency, mux and divider status.
- When a clock is switched, reset may need to remain asserted until the new clock is stable.
- A clock can stop while a reset request is pending. The reset manager must not wait forever for a clocked acknowledgment from a stopped domain.

### 9.2 Power-domain dependency

A typical power-down sequence may be:

1. Request the block to quiesce.
2. Stop new transactions.
3. Save retention state.
4. Drain accepted work.
5. Assert isolation.
6. Assert the power-domain reset.
7. Remove or collapse the supply.

A typical power-up sequence may be:

1. Apply supply and wait for power-good.
2. Keep isolation asserted.
3. Enable reference and functional clocks.
4. Release the relevant reset leaves.
5. Restore retained state.
6. Wait for block-ready status.
7. Release isolation.
8. Re-enable traffic.

The exact order of reset and retention restore is macro- and power-intent-specific. Follow the UPF, retention-cell and power-controller requirements.

### 9.3 Reset does not necessarily clear memory

- A resettable SRAM controller may reset its control logic while the SRAM array retains bits.
- A power-gated SRAM without retention may lose contents.
- A DRAM device and its controller have separate reset/initialization requirements; resetting the controller while DRAM remains active can destroy protocol alignment.
- A PHY reset may invalidate training results.
- An OTP/eFuse array is nonvolatile; reset must not be confused with erasing it.
- A cache reset can invalidate tags without clearing all data-array bits, depending on the implementation.

The reset specification must identify which state is architecturally valid after each reset.

## 10. Reset interfaces, status and register programming

### 10.1 Common interface signals

| Signal | Typical direction | Meaning |
|---|---|---|
| por_n | Input | Qualified external or internal power-on reset |
| reset_req | Input | Request to reset a system or module |
| reset_ack | Output | Request accepted or sequence completed |
| reset_n | Output | Active-low reset to an IP or domain |
| clk_valid | Input | Clock source is qualified for use |
| pll_lock | Input | PLL reports lock; still subject to integration qualification |
| pwr_good | Input | Supply/rail is within required operating window |
| quiesce_req | Output | Request a block to stop new work |
| quiesce_ack | Input | Block has reached a safe quiescent state |
| isolation_en | Output | Clamp signals crossing a powered-down/reset boundary |
| retention_save | Output | Request retention capture |
| retention_restore | Output | Request state restoration |
| reset_status | Output | Reset state or sequencing status |
| reset_cause | Register | Latched reason for the reset |

Not every IP uses every signal. A reset pin is often a simple asynchronous input, while reset request/acknowledge is an integration-level control protocol.

### 10.2 No universal reset register map

A generic reset manager might expose registers such as:

| Register concept | Purpose |
|---|---|
| RESET_REQUEST | Request system reset |
| RESET_STATUS | Current reset state and domain readiness |
| RESET_CAUSE | POR, watchdog, software, debug, brownout, low-power exit |
| MODULE_RESET_CONTROL | Permitted software-controlled leaf resets |
| MODULE_RESET_STATUS | Per-module reset assertion/ready status |
| RESET_ENABLE / REGWEN | Lock or enable software reset controls |
| RESET_ERROR | Unauthorized reset or consistency error |
| CRASH_DUMP_CONTROL | Capture pre-reset CPU or alert state |

These names are conceptual only. Addresses, field meanings, reset values, write semantics and access permissions are IP-specific.

### 10.3 Real implementation example: OpenTitan Earl Grey

The public OpenTitan reset-manager register table documents:
- RESET_REQ at offset 0x04.
- RESET_INFO at offset 0x08.
- ALERT_INFO at offset 0x18.
- CPU_INFO at offset 0x28.
- Software-controlled reset registers beginning at 0x54.
- ERR_CODE at offset 0x7c.

The documented reset-cause categories include POR, low-power exit, non-debug-module reset, software request and hardware request. RESET_INFO is documented as write-one-to-clear. These offsets and semantics belong to that OpenTitan integration; do not reuse them as a generic SoC register map. [OpenTitan Earl Grey reset-manager registers](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/registers.html) · [Theory of operation](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/theory_of_operation.html)

### 10.4 Firmware sequence

A robust boot flow commonly performs:

1. Read reset-cause and retained crash-dump information before clearing it.
2. Determine whether the event was POR, watchdog, software reset, debug reset or low-power exit.
3. Reinitialize only the state that did not survive.
4. Revalidate clocks, memory and security lifecycle state.
5. Clear sticky reset-cause bits using the documented write semantics.
6. Enable only approved module resets.
7. For a local reset, quiesce the driver and hardware before issuing the reset request.
8. Wait for reset completion and block-ready status with a timeout.
9. Reprogram registers, DMA descriptors and interrupts.
10. Re-enable traffic and log the recovery result.

Do not issue a system reset and expect subsequent instructions to execute unless the reset controller explicitly implements a non-resetting request path.

## 11. Protocol-specific reset examples

Reset is not a packet protocol, but individual IPs define reset behavior that must be coordinated with the SoC reset manager.

| IP/protocol | Reset distinction to understand |
|---|---|
| AXI / NoC | Interface reset and fabric reset must account for outstanding transactions and internal route/ID state |
| DDR / LPDDR | Controller, PHY and DRAM-device reset/initialization are distinct; training and mode-register state may need reinitialization |
| PCIe | Fundamental reset such as PERST# is distinct from function-level reset and internal controller reset |
| USB | Bus reset signaling is distinct from resetting the USB controller or PHY |
| Ethernet | MAC, PCS and external PHY reset/restart have separate state and link-retraining implications |
| eMMC / UFS | Host-controller reset and device reset/recovery are distinct |
| SoundWire / HDA | Controller, bus, codec and stream state may need coordinated re-enumeration or reprogramming |
| I2C / SPI / UART | Peripheral reset does not necessarily reset external devices; bus recovery may be needed |
| Camera / display | Resetting a DMA or pipeline mid-frame requires buffer and interrupt cleanup |
| Security engine | Reset must follow key-zeroization, lifecycle and secret-retention policy |

A USB bus reset, PCIe link reset or DRAM device reset is not interchangeable with the SoC's global POR.

## 12. Security and safety considerations

Reset can be a security boundary.

- A software-accessible reset must not bypass lifecycle locks or reset a security monitor that is supposed to enforce them.
- Key material must be zeroized or rendered inaccessible according to the security policy.
- Resetting one side of a protected interface must not leave the other side accepting stale authorization state.
- Debug reset may intentionally preserve debug state, but only in permitted lifecycle states.
- Crash-dump registers can contain sensitive information and need access control.
- Reset-cause registers should be retained long enough for software to diagnose faults and then cleared according to the defined semantics.
- Reset consistency checks can detect unauthorized leaf resets.
- Safety designs may require independent reset monitors, timeout paths and evidence that all critical state reached a safe value.

OpenTitan documents restrictions on which modules can be software-resettable, including blocks that hold secrets, configuration locks or security-critical timing/control state. This is a useful public example of the security implications of reset policy. [OpenTitan reset theory](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/theory_of_operation.html)

## 13. Common reset bugs and silicon failure signatures

| Symptom | Likely root cause | Debug direction |
|---|---|---|
| Intermittent boot hang | Reset released before clock stable | Probe PLL lock, clock-valid and reset leaves |
| One domain starts one cycle early | Unsynchronized reset release or multiple release paths | Inspect per-domain synchronizer and RDC report |
| Peripheral works after POR but not warm reset | State assumed to be reset or retained incorrectly | Compare reset matrix and software reinitialization |
| AXI fabric hangs after DMA reset | Outstanding responses or ID state orphaned | Capture AXI channel handshakes and reset timing |
| Random false interrupt after reset | Pending/level state not cleared or re-armed | Inspect source and aggregator reset semantics |
| FIFO reports full/empty incorrectly | Pointer and flag reset mismatch | Verify coordinated FIFO reset sequence |
| DDR training fails after warm reset | Controller/PHY/DRAM state mismatch | Re-run documented initialization and training |
| Debug unavailable after watchdog | Debug domain reset unintentionally | Check reset-tree partition and retention |
| Reset repeats continuously | Brownout, watchdog or reset consistency loop | Read AON reset-cause/error registers |
| Security lock unexpectedly clears | Reset scope violates lifecycle policy | Audit reset permissions and retained locks |
| Power-on current spike | Too many blocks released simultaneously | Review staged reset release and power sequencing |
| Simulation passes, silicon fails | Reset recovery/removal or RDC issue hidden by ideal clocks | Gate-level timing, RDC analysis and phase-randomized tests |

## 14. Pre-silicon verification plan

### 14.1 Directed tests

- POR assertion during power-up and slow power ramp.
- External reset pulse at minimum and maximum allowed widths.
- Watchdog reset while CPU is executing.
- Software reset while peripherals are idle.
- Local peripheral reset while neighboring domains remain active.
- Reset while AXI read and write transactions are outstanding.
- Reset while DMA owns a descriptor.
- Reset during DDR initialization/training.
- Reset during low-power entry and exit.
- Reset while clocks are stopped and while PLL lock is lost.
- Debug reset with debug retention enabled.
- Brownout or reset-source glitches within the supported filter model.

### 14.2 Randomized reset stress

Randomize reset assertion at:
- Every phase of an AXI read/write.
- DMA descriptor fetch, payload transfer and completion writeback.
- FIFO empty/full transitions.
- Interrupt assertion and deassertion.
- CPU exception and cache refill.
- DDR refresh and self-refresh transitions.
- USB/PCIe link training.
- Audio period boundaries and camera frame boundaries.

Scoreboard expectations must distinguish transactions that should complete, transactions that are explicitly aborted by the integration policy, and state that must survive.

### 14.3 Example assertions

The following are illustrative properties. Adapt reset polarity, reset semantics and protocol rules to the actual design.

~~~systemverilog
// No new bus request is allowed before the block is initialized.
property no_request_before_init;
  @(posedge clk) disable iff (!rst_n)
    !init_done |-> !(awvalid || arvalid);
endproperty
assert property (no_request_before_init);

// Isolation must remain asserted while the domain is in reset.
property isolate_during_reset;
  @(posedge aon_clk)
    !domain_reset_n |-> isolation_en;
endproperty
assert property (isolate_during_reset);

// A local reset request must not assert until quiesce is complete.
property reset_only_after_quiesce;
  @(posedge aon_clk) disable iff (!aon_rst_n)
    $fell(local_reset_n) |-> $past(quiesce_ack);
endproperty
// This is a conceptual assertion; adapt event syntax and latency
// to the actual reset-manager state machine.
~~~

Additional checks:
- Reset cause is captured before the CPU domain is reset.
- A leaf reset has an authorized parent/request cause.
- Reset release is synchronized to the leaf clock.
- No valid/ready transfer occurs while the interface is isolated.
- Reset deassertion does not occur until clock-valid is true.
- Reset pulse width meets the library and IP requirement.
- Retained registers are not cleared by a reset that promises retention.
- Non-retained state is reinitialized before traffic resumes.
- Every reset sequence reaches completion or a timeout/error state.

### 14.4 Formal properties

Formal verification can prove:
- No module leaves reset before its prerequisite clock is valid.
- No reset request is lost while the manager is active.
- Every accepted reset request eventually completes or reports a defined error.
- Parent/child reset consistency is maintained.
- No transaction is silently duplicated after a reset.
- Isolation remains active until destination readiness.
- A reset cannot bypass lifecycle or software lock policy.

Liveness properties require explicit assumptions: for example, the destination clock eventually runs and the quiesce acknowledgment eventually arrives. Without those assumptions, a reset manager cannot guarantee finite completion.

## 15. Post-silicon bring-up and debug

### 15.1 Bring-up checklist

1. Verify board reset pin polarity and voltage levels.
2. Capture power-good, POR and reference clock on a scope.
3. Verify the reset pulse width and POR filter behavior.
4. Confirm the always-on clock and reset manager are alive.
5. Check PLL lock and clock mux status.
6. Read reset-cause registers before clearing them.
7. Confirm reset leaves release in the expected order.
8. Verify boot ROM fetch and reset vector.
9. Validate SRAM/DDR initialization before enabling traffic.
10. Test watchdog, software and peripheral resets independently.
11. Exercise warm reset repeatedly at different traffic levels.
12. Test low-power entry/exit and retained state.
13. Check debug access after each reset class.
14. Run voltage and temperature corners for reset timing margins.

### 15.2 Useful observability signals

Capture or expose:
- External reset and POR output.
- Power-good and brownout indication.
- Reference clock and PLL lock.
- Clock-valid per domain.
- Root and leaf reset states.
- Reset-manager FSM state.
- Reset request source and arbitration result.
- Quiesce request/acknowledgment.
- Isolation enable.
- Reset cause and error code.
- CPU PC/crash dump.
- Outstanding AXI transaction count.
- DMA active/descriptor state.
- DDR controller initialization/training status.

### 15.3 Example watchdog failure investigation

**Symptom:** The SoC occasionally hangs after watchdog reset, but POR always boots correctly.

Investigation sequence:
1. Read the always-on reset-cause register immediately after reboot.
2. Verify that the watchdog cause is recorded before the CPU reset.
3. Check whether NoC and DMA are reset together or whether accepted transactions remain outstanding.
4. Inspect whether the DDR controller is reset while DRAM contents and device state are expected to remain valid.
5. Verify that debug and crash-dump domains survive long enough to capture the previous CPU state.
6. Check that firmware reinitializes every non-retained register.
7. Reproduce the failure with randomized reset injection while DMA and memory traffic are active.

**Likely lesson:** A watchdog is not simply a wire to the CPU reset pin. It initiates a recovery policy spanning the bus fabric, DMA, memory, peripherals, debug and firmware.

## 16. Worked SoC reset exercise

Assume a SoC has:
- Always-on timer and reset manager at 32.768 kHz.
- CPU at 1 GHz.
- NoC at 500 MHz.
- DDR controller at 400 MHz UI clock.
- DSP at 600 MHz.
- UART and I2C at 100 MHz.

**Question A — POR release:** What must be ready before the CPU executes from ROM?

A reasonable dependency plan is:
1. Power-good and AON reset manager.
2. Reference clock and CPU/NoC/DDR/DSP clock sources.
3. Reset synchronizers for each destination clock.
4. Boot ROM and required on-chip memory.
5. Interconnect path needed for instruction fetch.
6. CPU release.
7. DDR initialization before software accesses external DRAM.
8. DSP and peripherals when their firmware/drivers are ready.

The actual sequence depends on whether the boot ROM resides in on-chip ROM, whether the CPU can boot without DDR, and whether memory initialization is hardware- or firmware-controlled.

**Question B — Local DSP reset:** The DSP has four outstanding NoC reads.

Do not simply clear the DSP's request state. Stop new issue, drain or coordinate abort of the accepted reads, isolate DSP outputs, assert the DSP reset, release it on the DSP clock, reinitialize local state and then re-enable traffic.

**Question C — Reset latency:** A two-stage synchronizer on a 500 MHz clock has a 2 ns period. Two destination edges correspond to approximately 4 ns of clocked release latency, excluding phase, clock-start and distribution effects.

**Question D — Reset-cause retention:** Which block should hold the watchdog reason if the CPU is reset?

The always-on reset manager or another retained always-on register bank. It must remain powered and clocked through the CPU reset and expose the cause to boot firmware.

## 17. Hands-on exercises

### Exercise 1 — Reset matrix
Create a table for POR, watchdog, software, debug, module and low-power-exit resets. For each, mark CPU, NoC, DDR, DSP, peripherals, debug and AON as reset, retained or reinitialized.

### Exercise 2 — Reset synchronizer
Implement a two-stage active-low asynchronous-assert/synchronous-release synchronizer. Simulate reset deassertion at multiple offsets relative to the destination clock. Explain why RTL simulation cannot model analog metastability.

### Exercise 3 — AXI reset recovery
Create an AXI testbench that accepts multiple reads and writes, then resets the DMA master. Verify the selected drain/abort policy and ensure no response is mis-associated after recovery.

### Exercise 4 — RDC analysis
Connect a source register reset by rst_a_n to a destination register reset by rst_b_n, both on the same clock. Demonstrate the stale-level problem and add an isolation/handshake reinitialization scheme.

### Exercise 5 — Reset-cause firmware
Write pseudocode that reads reset cause, preserves crash information, clears sticky status using the documented semantics and performs only the required reinitialization.

### Exercise 6 — Low-power recovery
Design a sequence that saves retention state, isolates a DSP power island, removes power, restores power, releases reset, restores state and re-enables NoC traffic.

## 18. Interview questions

1. Why is reset architecture a system-level problem rather than a flip-flop coding convention?
2. What is the difference between POR, warm reset and module reset?
3. Why is asynchronous assertion often paired with synchronous deassertion?
4. What are reset recovery and removal checks?
5. Why does every unrelated clock domain need its own reset-release synchronizer?
6. Can two blocks on the same clock still have an RDC problem?
7. How is RDC different from CDC?
8. Why can a reset during an AXI transaction hang an interconnect?
9. What is the purpose of quiesce and isolation before a local reset?
10. Why should reset cause be retained in an always-on domain?
11. Does resetting an SRAM controller necessarily clear the SRAM array?
12. Why can resetting DDR controller logic without coordinating the DRAM device be unsafe?
13. How do retention and reset interact during power-down and wakeup?
14. Why might a security engine be excluded from software-controlled reset?
15. What evidence would you capture to debug an intermittent watchdog recovery failure?

## 19. Design review checklist

Before reset architecture signoff, confirm:

- [ ] Every reset source is documented, qualified and assigned an authorization policy.
- [ ] Every sequential block belongs to a defined reset domain or has a documented no-reset initialization strategy.
- [ ] Every reset domain has a defined clock relationship and release mechanism.
- [ ] POR behavior covers supported supply ramp and clock-start conditions.
- [ ] Reset pulse width and recovery/removal are checked against library/IP requirements.
- [ ] Reset release is synchronized per destination clock where required.
- [ ] Parent/child reset ordering is explicit.
- [ ] Local resets define quiesce, drain, abort and isolation behavior.
- [ ] RDC crossings are structurally and functionally verified.
- [ ] Retained and non-retained state is documented.
- [ ] Reset-cause and crash-dump information survives the intended reset classes.
- [ ] Software-visible register semantics and locks are documented.
- [ ] DFT, scan and debug behavior is defined.
- [ ] Reset recovery is tested during traffic, low-power transitions and fault conditions.
- [ ] Post-silicon observability is available for every important reset path.

## 20. Summary: what a silicon reset engineer must remember

1. Reset establishes a known state, but not every memory bit must be cleared.
2. Reset source, reset domain, clock domain and power domain are separate architectural concepts.
3. Asynchronous reset assertion can work without a clock; asynchronous release can violate recovery/removal timing.
4. A reset synchronizer must be designed for each relevant destination clock.
5. Reset ordering and RDC are system-level contracts.
6. Local reset requires quiesce, drain or abort policy, isolation and reinitialization.
7. A reset manager should preserve reset cause and useful crash information in retained state.
8. Reset behavior for DDR, PCIe, USB, Ethernet, audio and other IPs is not interchangeable.
9. Verification must inject reset during real traffic and power transitions, not only while the system is idle.
10. Post-silicon reset debug requires observability across power, clock, reset, interconnect and software.

---

## 21. Authoritative references and implementation examples

### Historical and engineering papers
1. Intel, *8080 Microcomputer Systems User's Manual* (1975 scan): [PDF](https://bitsavers.trailing-edge.com/components/intel/MCS80/98-153B_Intel_8080_Microcomputer_Systems_Users_Manual_197509.pdf).
2. Intel, *8086 User Manual* (period documentation): [Manual](https://manuzoid.com/manuals/gqvrn-Intel%208086%20User%20manual).
3. Clifford E. Cummings, Don Mills and Steve Golson, *Asynchronous & Synchronous Reset Design Techniques — Part Deux*, SNUG Boston 2003, Rev. 1.2: [PDF](https://trilobyte.com/pdf/CummingsSNUG2003Boston_Resets_rev1_2.pdf).
4. Priya Viswanathan et al., *A Specification-Driven Methodology for the Design and Verification of Reset Domain Crossing Logic*, DVCon 2018: [PDF](https://dvcon-proceedings.org/wp-content/uploads/a-specification-driven-methodology-for-the-design-and-verification-of-reset-domain-crossing-logic.pdf).

### Vendor methodology and public RTL
5. AMD, *Versal Adaptive SoC Hardware, IP, and Platform Development Methodology Guide (UG1387), 2026.2*, released 2026-11-20: [Synchronous vs. asynchronous reset](https://docs.amd.com/r/en-US/ug1387-acap-hardware-ip-platform-dev-methodology/Synchronous-Reset-vs.-Asynchronous-Reset).
6. OpenTitan, *Earl Grey Reset Manager — Theory of Operation*: [Documentation](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/theory_of_operation.html).
7. OpenTitan, *Earl Grey Reset Manager — Registers*: [Register map](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/registers.html).
8. Siemens/Mentor, *A Specification-Driven Methodology for RDC*: [Overview](https://verificationacademy.com/topics/reset-domain-crossing/a-specification-driven-methodology-for-the-design-and-verification-of-reset-domain-crossing-logic/).

### Source classification
- **Normative requirements:** Use the exact processor, memory, PHY, board, library and IP integration specifications for reset pulse widths, pin polarity, power-good thresholds, recovery/removal, and device reset sequences.
- **Documented implementation examples:** OpenTitan reset trees/registers and AMD FPGA methodology illustrate real design choices.
- **Engineering interpretation:** The dependency graphs, example domain matrix, local-reset handshake and worked SoC sequences in this lesson are generic teaching examples. They are not claims about a proprietary vendor implementation.

---

**Next lesson — Session 16: Power Management — Power Domains, Isolation, Retention, DVFS, Power States and Wakeup**

The next session will connect reset sequencing to the power manager: how a subsystem safely saves state, isolates interfaces, switches rails, stops clocks, restores retained state and wakes up without corrupting transactions.
