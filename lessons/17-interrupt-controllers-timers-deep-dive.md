# Lesson 17 — Interrupt Controllers and Timers
## From asynchronous event capture to deterministic CPU service and timekeeping

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 17 of 38  
**Target duration:** 60 minutes  
**Level:** protocol, RTL, integration, verification and post-silicon  
**Prerequisites:** AXI/APB, CPU exception entry, clock/reset domains, power manager and DMA fundamentals

> **Scope discipline:** This lesson explains interrupt-controller and timer architecture. It is not a substitute for the applicable Arm GIC, RISC-V privileged/AIA, platform, or timer IP specification. Register names and sequences are architectural examples; physical addresses, reset values, supported features and routing policies are implementation/platform specific.

---

## 1. Learning objectives and one-hour plan

By the end, you should be able to trace a peripheral event to a CPU handler and back; distinguish source acknowledgement, controller claim, priority drop and deactivation; configure a GIC or PLIC-style controller; program a free-running timer and compare channel; calculate latency and timebase conversion; and build a verification plan for edge/level, priority, routing, reset, clock and low-power cases.

| Time | Topic | Outcome |
|---:|---|---|
| 0–5 min | Why interrupts and timers exist | Event-driven versus polling; timebase contracts |
| 5–12 min | Historical evolution | 8259A/PIT to GIC, PLIC, AIA and architectural timers |
| 12–22 min | Interrupt-controller architecture | Source, gateway, pending state, arbitration, CPU interface |
| 22–30 min | Protocol and programming models | Edge/level, claim/complete, acknowledge/EOI, MSI |
| 30–39 min | Timers and watchdogs | Counter/compare, periodic scheduling, always-on time |
| 39–46 min | Calculations and walkthroughs | Latency, resolution, wraparound, timeout |
| 46–54 min | Integration, verification and silicon debug | CDC, power, reset, ordering and failure signatures |
| 54–60 min | Exercises and interview review | Design-review checklist and lab assignments |

## 2. Why these IPs exist

An interrupt controller is an event concentrator and delivery policy engine. It accepts events from many devices, remembers requests when required, applies masks and priorities, selects a destination CPU or execution context, and provides a controlled acknowledgement/completion protocol.

A timer is a timebase and/or event generator. It lets software schedule work against elapsed time without consuming a CPU continuously in a polling loop. A watchdog adds a liveness contract: software must demonstrate progress before a deadline or the system takes a recovery action.

The three functions must not be conflated:

- **Peripheral interrupt source:** knows that a local condition occurred, for example DMA completion, FIFO threshold, parity error or packet arrival.
- **Interrupt controller:** remembers/routes/prioritizes the event and presents it to a CPU context.
- **CPU exception architecture:** decides whether the event is currently eligible, saves execution state and enters the handler.
- **Timer:** measures time or detects a compare condition. It may drive an interrupt-controller input, a CPU-local interrupt, a wakeup request or a reset request.

A timer expiration is not automatically a CPU interrupt. The timer's output may pass through a controller, a local interrupt path, an always-on wakeup controller, or a platform-specific fabric.

## 3. Historical backstory and design rationale

### 3.1 The 1970s: programmable peripheral building blocks

Intel's February 1979 *Peripheral Design Handbook* lists both the 8253 programmable interval timer and the 8259A programmable interrupt controller as standard microcomputer peripherals. The handbook describes the 8253 as three independent 16-bit counters and the 8259A as an eight-level priority controller expandable to 64 levels by cascading. The 8259A was static and did not require a clock input. This is direct documentary evidence of the period's modular peripheral design approach, not a claim that every system used these exact parts. [H1][H2]

**The problem behind the 8259A:** a CPU with only a small number of interrupt inputs should not need a separate CPU pin and custom priority logic for every peripheral. The 8259A captured requests in an Interrupt Request Register (IRR), tracked in-service levels in an ISR, applied an Interrupt Mask Register (IMR), resolved priority and signalled the CPU. Cascading master/slave devices expanded the number of sources. The original Intel datasheet documents IR0–IR7, INT/INTA, the data bus, cascade pins and the three core state registers. [H2]

**The problem behind the 8253:** software and peripherals needed programmable intervals, event counts, pulse generation and periodic rates. Three independently programmed 16-bit counters were a compact, reusable solution. The 1979 handbook documents binary/BCD counting and multiple counter modes. [H1]

### 3.2 From cascaded PICs to multicore interrupt fabrics

As systems moved from a single CPU to multicore processors, a simple fixed-priority concentrator was insufficient. A modern controller must account for per-CPU private interrupts, shared peripheral interrupts, inter-processor interrupts, affinity, security groups, virtualization, message-signalled interrupts and power state.

Arm's GIC family documents this progression:
- GICv1/v2 provide distributor and CPU-interface concepts, with GICv2 adding virtualization support.
- GICv3 expands support for more processing elements and interrupt IDs, adds system-register CPU-interface access and message-signalled interrupts.
- GICv4 adds direct injection of virtual interrupts; later revisions extend virtualization and non-maskable interrupt support.

Arm's 2025 GICv3/v4 overview is a useful dated guide to these generation-level differences. It is an explanatory guide; the normative architecture specification defines exact behavior. [H3]

### 3.3 RISC-V: simple wired interrupts and the move to AIA

The RISC-V Platform-Level Interrupt Controller (PLIC) specifies a central model with source priorities, pending bits, per-context enables and thresholds, and a claim/complete register. A claim is a non-idempotent MMIO read that atomically returns the highest-priority eligible source ID and clears its pending bit. Completion is a write of the claimed ID to the corresponding context's claim/complete register. [H4][H5]

The RISC-V Advanced Interrupt Architecture (AIA) was ratified in June 2023. It adds the Advanced PLIC (APLIC) and Incoming MSI Controller (IMSIC), along with extensions for machine, supervisor and virtual-machine interrupt delivery. In an MSI-oriented system, a device can signal an interrupt with a write to an interrupt-controller message address instead of requiring one dedicated wire per source. The APLIC can also convert wired interrupts to MSIs for delivery to an IMSIC. [H6][H7]

**Engineering interpretation:** the progression from cascaded PICs to GIC/PLIC/AIA is not simply “more interrupt numbers.” It is a response to source count, multicore routing, virtualization, interrupt affinity, wired-signal routing cost and the need to deliver events efficiently across large fabrics. The standards document the architectural mechanisms; vendor-internal motivations should not be inferred.

### 3.4 Timer evolution

The 8253 represents a programmable peripheral timer with externally supplied clocking and multiple modes. Modern systems add always-on counters, per-CPU compare channels, virtual timers, high-resolution event scheduling, clock-domain isolation and watchdog escalation.

Arm's Generic Timer uses a system counter and timer compare/control registers. The architecture defines a programmer-visible effective frequency through CNTFRQ_EL0. The system counter must provide a uniform view of time and is specified to reside in an always-on power domain. Arm's A-profile architecture documentation describes counter width and effective-frequency evolution, including the 64-bit requirement from Armv8.6. [H8]

RISC-V defines a 64-bit memory-mapped mtime counter and mtimecmp compare register. The timer interrupt becomes pending when mtime is greater than or equal to mtimecmp and remains pending until the compare value is moved ahead of the current time. The platform must provide a constant-frequency timebase and a way to determine its tick period. [H9]

## 4. Interrupt-system architecture

### 4.1 End-to-end path

~~~mermaid
flowchart LR
  DEV[Peripheral source] --> GW[Gateway / synchronizer]
  GW --> PEND[Pending state]
  PEND --> ARB[Mask + priority + routing]
  ARB --> CPUINT[CPU interrupt interface]
  CPUINT --> EXC[CPU exception entry]
  EXC --> HANDLER[ISR / top half]
  HANDLER --> SRC[Clear peripheral cause]
  SRC --> EOI[EOI / deactivate / complete]
  EOI --> READY[Source can signal again]
~~~

Typical sources include GPIO edge detection, UART receive threshold, DMA completion, PCIe MSI, timer compare, thermal alarm and watchdog bark. The actual pipeline can be distributed across the peripheral, interrupt fabric, NoC, CPU interface and software.

### 4.2 Main architectural blocks

| Block | Responsibility | Common implementation choices |
|---|---|---|
| Input synchronizer | Bring asynchronous source into controller clock domain | Two-flop level synchronizer, pulse stretcher, toggle synchronizer, async event FIFO |
| Gateway | Convert source signaling into a controller request | Edge detector, level tracker, MSI receiver |
| Pending state | Remember an event until it is accepted | Sticky bit, request queue, pending table |
| Enable/mask | Suppress delivery without necessarily losing pending state | Per-source enable bitmap, per-context enable |
| Priority resolver | Select an eligible request | Fixed priority, programmable priority, threshold, preemption |
| Router | Select CPU/hart/VM destination | Affinity register, target mask, MSI destination |
| CPU interface | Present and acknowledge an interrupt | IRQ/FIQ lines, system registers, claim read |
| Completion logic | Track service completion and source re-arming | EOI, deactivate, claim/complete handshake |

Do not assume every architecture has every block as a physically separate RTL module. The table is a functional decomposition.

### 4.3 Interrupt classes

**Edge-triggered:** an event is recognized on a transition. A controller normally latches a pending indication so a short pulse is not lost after synchronization. If only one pending bit exists, multiple edges while already pending can coalesce into one pending event. If every event matters, use a counter/FIFO or a source protocol that preserves event multiplicity.

**Level-triggered:** a request remains asserted while the source condition remains true. The handler must clear or mask the peripheral condition before completion/unmasking, otherwise the controller can immediately present the interrupt again. Level signaling is useful for conditions that remain observable until serviced.

**Software-generated interrupt (SGI/IPI):** one processing element asks another to perform work, reschedule, invalidate or coordinate. A GIC has SGI concepts; RISC-V has software interrupt mechanisms and AIA IPI support.

**Private peripheral interrupt (PPI):** a per-PE source such as a local timer. It is not a globally shared peripheral line.

**Shared peripheral interrupt (SPI):** a device interrupt shared at system level and routed to one or more eligible PEs according to controller configuration.

**Locality-specific/message-signalled interrupt (LPI/MSI):** an interrupt represented by a message/write rather than a dedicated physical wire. In GICv3, LPIs are message-signalled and use an Interrupt Translation Service (ITS) in systems that implement it. RISC-V AIA uses IMSIC interrupt files for incoming MSIs.

**NMI/FIQ:** architecturally distinct high-priority or non-maskable exception classes. Exact masking, security and routing semantics are architecture-specific. Do not treat “high priority IRQ” as synonymous with NMI.

### 4.4 Edge versus level timing

~~~text
Edge source:   ____/‾\________________
Sync output:   ________/‾\____________
Pending bit:   ________/‾‾‾‾‾‾\_______  (cleared by controller protocol)

Level source:  ____/‾‾‾‾‾‾‾‾\________
Sync output:   ________/‾‾‾‾‾‾‾\_____
Pending:       ________/‾‾‾‾‾‾‾‾\_____  (tracks active level / pending state)
~~~

The drawing is conceptual. Real synchronizer latency, edge-capture behavior, pending clear semantics and pulse-width requirements must be taken from the selected controller and source-interface specification.

## 5. Interrupt lifecycle and protocol semantics

A useful abstract lifecycle is:

~~~mermaid
stateDiagram-v2
  [*] --> Inactive
  Inactive --> Pending: source event
  Pending --> Active: accepted / acknowledged
  Active --> Inactive: source cleared + deactivate
  Active --> ActivePending: another event arrives
  ActivePending --> Pending: active instance deactivated
  Pending --> Inactive: edge pending explicitly cleared or cancelled
~~~

This is a conceptual lifecycle, not a universal encoding. Some controllers distinguish pending, active, active-and-pending, and disabled states; others implement only a subset.

### 5.1 Acknowledge, clear, EOI, deactivate: four different actions

1. **Acknowledge/claim:** controller tells the CPU which source it accepted. This may change pending/active state.
2. **Peripheral clear:** driver writes the device's status or consumes its FIFO so the original condition goes away.
3. **Priority drop / EOI:** CPU tells the interrupt interface that the current priority level can be dropped.
4. **Deactivate / complete:** controller is told the active interrupt is fully serviced and its gateway may accept another event.

These actions are not interchangeable. A common level-interrupt bug is to write EOI but leave the peripheral status asserted, causing immediate re-entry.

### 5.2 GICv3/v4 programmer model

Functional blocks:
- **Distributor:** system-wide SPI configuration, enable state, priority, group and routing.
- **Redistributor:** per-PE SGI/PPI and LPI-related state; also tracks whether the PE is online through GICR_WAKER.
- **CPU interface:** system-register interface used by the PE to acknowledge and complete interrupts.
- **ITS (when implemented):** translates device/event identifiers into LPI delivery.

The Arm GICv3/v4 overview states that the Distributor and Redistributors configure interrupts, while the CPU interface handles them. It also documents that a PE's Redistributor must be brought online before interrupts are forwarded to it. [H3]

Common architectural register names:
- Distributor: GICD_CTLR, GICD_ISENABLERn, GICD_ICENABLERn, GICD_IPRIORITYRn, GICD_ICFGRn, GICD_IROUTERn.
- Redistributor: GICR_WAKER and per-PE interrupt configuration/state registers.
- CPU interface: ICC_PMR_EL1, ICC_IGRPEN1_EL1, ICC_IAR1_EL1, ICC_EOIR1_EL1; ICC_DIR_EL1 when split deactivation is used.

**Typical SPI setup sequence (illustrative):**
1. Hold the source peripheral masked and clear any stale device status.
2. Confirm Distributor and target Redistributor are initialized and enabled.
3. Configure the interrupt group/security attributes as permitted by the platform.
4. Program trigger configuration (edge or level) to match the peripheral electrical/protocol behavior.
5. Set priority and route/affinity to the intended PE.
6. Clear stale pending state where the architecture and source protocol allow it.
7. Enable the interrupt at the Distributor.
8. Configure CPU-interface priority mask and group enable.
9. Configure the peripheral source and unmask it last.
10. In the handler, read ICC_IAR1_EL1, reject special/spurious INTIDs, service the device, then write the matching acknowledge value to ICC_EOIR1_EL1. If split EOI mode is enabled, also perform the required deactivation using ICC_DIR_EL1.

The exact security group, system-register access, barriers, enable order and split-EOI policy depend on the implemented GIC version and software execution level. Preserve the full acknowledge value required by the applicable architecture specification; do not reconstruct it from only a truncated interrupt number. [H10]

### 5.3 RISC-V PLIC programmer model

Core register concepts:
- **Priority[source]:** priority value per source; zero means never interrupt.
- **Pending[source]:** controller-owned pending indication.
- **Enable[context][source]:** whether a source is enabled for a target context.
- **Threshold[context]:** masks priorities less than or equal to the threshold.
- **Claim/Complete[context]:** read to claim highest-priority pending ID; write the claimed ID to complete.

A typical platform sequence:
1. Disable the device's local interrupt generation.
2. Program source priority to a nonzero value.
3. Enable the source in the target hart/privilege context.
4. Program the context threshold.
5. Install the trap vector and enable the relevant global/local CPU interrupt bits.
6. Clear stale device status and enable the device source.
7. In the trap handler, read the context claim register. ID zero means no claimable interrupt.
8. Service and clear the peripheral cause.
9. Write the same claimed ID to the context completion register.

PLIC claim is an atomic, non-idempotent MMIO read. Do not let a bridge, cache, speculative path or software abstraction treat it like an ordinary repeatable register read. The claim and completion semantics are defined in the PLIC specification. [H4][H5]

### 5.4 AIA: APLIC and IMSIC

APLIC accepts wired interrupt sources and can route them to harts. IMSIC provides per-hart interrupt files for incoming MSIs. AIA supports machine/supervisor-level delivery and extensions for virtual interrupt files. This moves scalable delivery toward messages and per-hart state instead of routing a unique interrupt wire for every event. [H6][H7]

**Integration warning:** an MSI is a write transaction. Its address/data encoding, IOMMU translation, security/firewall permissions, ordering and target interrupt-file configuration are part of the platform contract. A device can complete its data DMA correctly yet lose notification if its MSI write is blocked or misrouted.

## 6. Timer architecture

### 6.1 Common timer building blocks

| Block | Function |
|---|---|
| Reference/timebase clock | Defines elapsed-time scale |
| Prescaler / tick generator | Divides a faster clock to a desired resolution |
| Free-running counter | Monotonic count, ideally never reset during ordinary runtime |
| Compare register | Stores a deadline |
| Comparator | Detects counter >= compare |
| Status / enable / mask | Exposes and controls expiration |
| Interrupt or wakeup output | Signals a CPU/controller/PMU |
| Capture input (optional) | Latches current count on an external event |
| Watchdog state machine | Detects missing service and escalates |

~~~mermaid
flowchart LR
  CLK[Always-on reference] --> PRE[Prescaler / tick]
  PRE --> CNT[Free-running counter]
  CNT --> CMP[Compare logic]
  CVAL[Compare value] --> CMP
  CMP --> STAT[Expired status]
  STAT --> IRQ[Timer IRQ / wakeup]
  IRQ --> CTRL[Interrupt controller or PMU]
~~~

### 6.2 Timer types

- **Free-running counter:** monotonic timestamp source.
- **Compare timer:** interrupt when counter reaches a deadline.
- **One-shot:** one event, then stops or remains expired until reprogrammed.
- **Periodic:** repeated interval events, often by adding a period to the previous deadline.
- **Capture/compare:** hardware captures timestamp or drives output at a programmed time.
- **Watchdog:** requires a service/pet operation before timeout.
- **Windowed watchdog:** service is accepted only inside an allowed time window; too early or too late can be treated as failure.
- **RTC / AON timer:** continues through main-domain clock gating or power collapse, subject to retention and backup-power design.

### 6.3 Arm Generic Timer

The Arm Generic Timer separates a system counter from timer compare channels. Architectural examples include physical and virtual timer views. Software commonly uses:
- CNTFRQ_EL0: effective counter frequency.
- CNTPCT_EL0: physical counter value.
- CNTP_CVAL_EL0 or CNTP_TVAL_EL0: physical timer deadline/value.
- CNTP_CTL_EL0: enable, mask and status.
- CNTVCT_EL0 / CNTV_CVAL_EL0 / CNTV_CTL_EL0: virtual timer view.

For a one-shot deadline:
1. Read the current counter.
2. Convert desired duration to counter ticks.
3. Write compare value = now + delta, or use TVAL for a relative interval.
4. Enable timer and unmask its interrupt.
5. On expiration, service the timer PPI and program the next deadline.

Arm's architecture documentation describes compare expiration when the counter reaches/passes the compare value. CNTFRQ_EL0 must be initialized by higher-level software/firmware; do not assume it has a useful reset value. The system counter's effective frequency is an architectural time unit and need not equal the physical clock toggling rate. [H8][H11]

### 6.4 RISC-V mtime and mtimecmp

The RISC-V privileged architecture defines:
- mtime: 64-bit machine-level memory-mapped counter.
- mtimecmp: 64-bit compare value.
- MTIP: machine timer interrupt pending indication.
- MTIE: machine timer interrupt enable in mie.
- MIE: global machine interrupt enable in mstatus.

When mtime >= mtimecmp, MTIP becomes pending. It is cleared only after mtimecmp is moved greater than mtime, with the pending indication reflected eventually rather than necessarily in the same cycle. The platform must expose the timebase period. [H9]

**RV32 split-write hazard:** a 64-bit compare is accessed as two 32-bit writes. An intermediate torn value can briefly be less than the live counter and cause a spurious timer interrupt. The privileged specification provides a safe sequence for little-endian strongly ordered I/O: write all ones to the low half, then the new high half, then the new low half. Follow the exact sequence and memory-ordering assumptions in the implemented spec/platform. [H9]

**RV32 stable read pattern:** read high, read low, read high again; retry if the high halves differ. This prevents combining a low half from before rollover with a high half from after rollover. A hardware snapshot/latch is preferable when the IP provides one.

### 6.5 Always-on timer and watchdog example

OpenTitan's public AON Timer is a concrete integration example. Its documented design uses an approximately 200 kHz always-on clock, a 64-bit wakeup timer and a 32-bit watchdog timer. It connects to the CPU, PLIC, power manager and reset manager. Its documentation also warns that register writes cross from the system domain to the slow AON domain asynchronously; a readback is used when software must guarantee that prior writes have reached the underlying register. [H12]

This is an implementation example, not a universal watchdog register model.

## 7. Timer programming calculations

### 7.1 Convert duration to ticks

[
N = operatorname{round}(T 	imes f_{	ext{effective}})
]

For a 24 MHz effective timebase and a 1 ms deadline:

[
N = 0.001 	imes 24{,}000{,}000 = 24{,}000 	ext{ticks}
]

For a 19.2 MHz timebase and 250 microseconds:

[
N = 0.00025 	imes 19{,}200{,}000 = 4{,}800 	ext{ticks}
]

If the counter increments by a prescaled step, calculate from the documented effective rate, not the raw oscillator frequency.

### 7.2 Timer resolution and quantization

At 32.768 kHz:

[
T_{	ext{tick}} = rac{1}{32768} approx 30.5176 mu s
]

A requested 1 ms delay cannot be represented exactly as an integer number of these ticks. Choose 33 ticks (about 1.00708 ms) or 32 ticks (about 0.97656 ms), depending on whether the requirement is “not earlier than” or “nearest.” For hard deadlines, round in the direction required by the safety contract.

### 7.3 Timebase error

If the oscillator error is 50 ppm, the time error accumulated over one hour is:

[
3600 	imes 50 	imes 10^{-6} = 0.18 	ext{s}
]

Resolution and accuracy are different. A 1 ns counter resolution does not guarantee 1 ns accuracy; oscillator tolerance, temperature drift, calibration and synchronization dominate long-term accuracy.

### 7.4 Wraparound

Unsigned counter comparisons must be designed for wraparound. For a width-W counter, use modular-difference logic with a maximum scheduled horizon smaller than half the counter range when using signed-difference deadline tests. Do not compare arbitrary timestamps with naïve greater-than logic across wrap.

### 7.5 Watchdog timeout example

At 200 kHz, a 32-bit counter spans:

[
rac{2^{32}}{200{,}000} approx 21{,}474.8 	ext{s} approx 5.96 	ext{hours}
]

This agrees with the approximate six-hour watchdog range documented for the OpenTitan AON Timer. The exact prescaler and bark/bite policy are implementation-specific. [H12]

## 8. Interrupt latency, jitter and throughput

### 8.1 Latency decomposition

[
L_{	ext{IRQ}} =
L_{	ext{source}} +
L_{	ext{CDC}} +
L_{	ext{controller}} +
L_{	ext{fabric}} +
L_{	ext{CPU-entry}} +
L_{	ext{masking}}
]

This is a budget model, not a guaranteed architectural equation. The measured point must be defined: source event to IRQ pin, source event to first handler instruction, or source event to useful work completion.

Illustrative assumptions:

| Stage | Assumption | Time |
|---|---:|---:|
| Source synchronizer | 2 cycles at 50 MHz | 40 ns |
| Controller arbitration | 2 cycles at 500 MHz | 4 ns |
| Interrupt route/fabric | 6 cycles at 250 MHz | 24 ns |
| CPU exception entry | 14 cycles at 1 GHz | 14 ns |
| **Subtotal** | Excludes masking and software | **82 ns** |

If a higher-priority handler holds interrupts masked for 3 microseconds, the actual worst-case response can exceed 3.082 microseconds even though the hardware pipeline is only 82 ns.

### 8.2 Priority and starvation

A strict fixed-priority policy can starve low-priority sources under sustained high-priority traffic. Rotating/round-robin arbitration improves fairness but may increase latency variance for urgent events. Priority grouping and preemption thresholds must be designed together with the software critical-section model.

### 8.3 Interrupt storm and event coalescing

If a source asserts continuously because the peripheral cause was not cleared, the CPU can re-enter immediately after EOI. Conversely, a one-bit pending latch can coalesce multiple edges. If the peripheral reports “N completions,” a count/FIFO or descriptor-ring scan is safer than assuming one IRQ per completion.

## 9. Register and software programming patterns

### 9.1 Generic source-driver initialization order

1. Disable the source in the peripheral.
2. Clear stale status using the peripheral's documented clear semantics (write-one-to-clear, read-to-clear, FIFO drain, etc.).
3. Configure controller trigger type, priority and target.
4. Register/install the handler and configure CPU-side masks.
5. Enable the controller route.
6. Enable the peripheral source last.
7. In the handler, identify the source, service the condition, clear device status, perform required ordering/readback, then complete/EOI the controller protocol.

For a shared interrupt, the handler must determine whether its device asserted the shared line. Linux generic IRQ documentation describes shared and threaded IRQ handling and the need to initialize hardware/handler state before enabling the source. [H13]

### 9.2 Timer initialization order

1. Start and validate the timer reference clock.
2. Program prescaler/step before enabling the counter, if the IP requires it.
3. Read or establish the effective frequency.
4. Initialize the monotonic counter policy; avoid resetting a shared system counter during ordinary runtime.
5. Program compare value.
6. Clear/mask stale expiration state according to the IP contract.
7. Enable the timer output.
8. Enable the controller route and CPU interrupt bit.
9. Verify with a known interval and a timestamp capture.

### 9.3 Ordering and posted MMIO writes

A write to clear a device status bit may be posted through a bridge. If software writes EOI before the clear reaches the peripheral, the level can still be asserted and immediately retrigger. Use the architecture/platform-prescribed device-memory ordering and, where required, a readback from the same device to flush posted writes. A CPU memory barrier does not magically clear a device interrupt source.

## 10. Clock, reset and power integration

### 10.1 Clock domains

- Peripheral clock may differ from interrupt-controller clock.
- Controller clock may differ from CPU clock.
- AON timer clock should continue while main clocks are gated if wakeup/deadline continuity is required.
- A narrow pulse crossing an asynchronous boundary needs pulse capture, toggle synchronization or an event queue; a two-flop level synchronizer alone can miss a short pulse.
- A multi-bit timestamp crossing domains requires a coherent transfer protocol, Gray-coded counter or snapshot handshake; independent bit synchronizers can produce torn values.

### 10.2 Reset behavior

Define reset behavior for:
- source pending state,
- controller pending/active state,
- CPU interface masks and priority state,
- timer counter and compare registers,
- watchdog bark/bite state,
- reset-cause retention.

A warm reset that clears controller active state while a peripheral level remains asserted may immediately re-pend the source. A timer reset that restarts the system timebase can break scheduler deadlines and distributed timestamp comparisons. These are platform contracts, not universal behavior.

### 10.3 Power management

An interrupt source may be the reason a power domain wakes. The wakeup path must not depend on a clock that has already been gated. AON timer and wakeup logic should remain powered, and controller/redistributor state must be restored or retained according to the platform's power sequence. OpenTitan's AON timer documentation shows separate AON and system-domain outputs and the dependency on starting the AON clock before the main clock. [H12]

## 11. Verification strategy

### 11.1 Directed and constrained-random scenarios

- One edge, one level, short pulse and pulse near a clock boundary.
- Two simultaneous sources with equal and different priorities.
- Mask while pending; unmask while source remains asserted.
- New edge arrives while previous instance is active.
- Nested higher-priority interrupt and return.
- Wrong EOI/complete ID, duplicate EOI, spurious claim.
- CPU target offline/online and affinity change.
- Multicore claim race and interrupt migration.
- MSI address/data decode, permission fault and ordering.
- Timer compare exactly equal to count, one tick before and one tick after.
- Counter rollover during split read/write.
- Prescaler reconfiguration while active.
- Timer clock changes, main-clock gating and AON wakeup.
- Watchdog bark, bite, service window violation and reset cause.
- Reset during pending, active and masked interrupt states.

### 11.2 Useful assertions (conceptual SystemVerilog)

~~~systemverilog
// A level source must remain represented as pending/active while asserted,
// unless the architecture explicitly defines a source-side masking protocol.
property p_level_not_lost;
  @(posedge irq_clk) disable iff (!rst_n)
    level_sync && source_enabled |-> pending_or_active;
endproperty
assert property (p_level_not_lost);

// A compare event must set sticky status until software clears/reprograms it.
property p_timer_expire_sticky;
  @(posedge timer_clk) disable iff (!rst_n)
    (count >= compare && timer_enable && !status_clear)
      |=> timer_status;
endproperty
assert property (p_timer_expire_sticky);

// An accepted interrupt ID must be the one completed by that context.
// Use a scoreboard/property appropriate to the chosen claim/EOI protocol.
~~~

These are intent examples, not drop-in assertions. The exact pending-state invariant depends on edge/level semantics, active/pending support and clear protocol.

### 11.3 Coverage points

Cross source type × trigger mode × priority × target × mask state × active/pending state × reset/power state. For timers, cross clock rate × prescaler × compare distance × rollover × enable/mask × low-power state. Track latency bins and worst-case masking duration, not only functional pass/fail.

## 12. Post-silicon bring-up and debug

### 12.1 First bring-up checklist

1. Confirm timer and interrupt-controller clocks are running.
2. Read controller identification/type registers and confirm implemented feature count.
3. Confirm CPU interface and target Redistributor/context are online.
4. Keep all peripheral sources masked; configure one known timer interrupt.
5. Program a long, easy-to-observe interval.
6. Toggle a GPIO at handler entry and exit; capture with a scope/logic analyzer.
7. Read pending/active/claim state before and after service.
8. Confirm peripheral cause is cleared and controller completion occurs.
9. Repeat with edge, level and multiple priority sources.
10. Test wakeup from clock-gated/deep-idle state.

### 12.2 Failure-signature matrix

| Symptom | Likely causes | Evidence to collect |
|---|---|---|
| Interrupt never arrives | Source disabled, wrong route, CPU mask, target offline, clock off | Peripheral status, enable bitmap, route, CPU mask, clock status |
| Interrupt arrives once only | Missing EOI/complete, source clear issue, gateway not re-armed | Active/pending state, source status, completion trace |
| Interrupt storm | Level source not cleared, wrong polarity, status write semantics misunderstood | IRQ pin, device status, repeated INTID and timestamp |
| Wrong handler runs | Vector mapping, interrupt ID decode, stale table, MSI translation | Acknowledge ID, vector base, ITS/IMSIC translation |
| Lost edge | Pulse too narrow, CDC defect, pending coalescing | Source pulse width, synchronizer output, pending transitions |
| Timer drifts | Wrong frequency, clock switching without compensation, prescaler error | CNTFRQ/timebase, oscillator tolerance, counter deltas |
| Spurious timer IRQ | Torn RV32 compare write, stale pending update, compare too close | mtime/mtimecmp snapshots, MTIP, write ordering |
| Wakeup fails | AON clock/power off, wake route masked, retained state invalid | AON status, PMU state, wake request, reset cause |
| High latency/jitter | Masking, priority inversion, fabric contention, cache/TLB misses | Timestamp trace, CPU PMR/threshold, bus and cache counters |

### 12.3 Measuring latency correctly

Use a hardware timestamp at source generation and at handler entry if available. If using GPIO toggles, account for GPIO bus-write latency, synchronizer delay and output pad delay. Collect minimum, median, 99th percentile and worst observed latency under stress; also record interrupt masking windows and competing DMA/NoC traffic.

## 13. Practical system case study — 32-channel audio DMA plus camera and PCIe

Assume:
- 32-channel audio playback/capture uses cyclic DMA.
- DMA emits one interrupt per completed period, not per sample.
- Camera generates frame/line events.
- PCIe raises MSI/MSI-X events.
- DSP and CPU share an interrupt fabric.

A robust design should:
1. Keep high-rate audio service close to the DSP or use a local interrupt/doorbell path.
2. Avoid one CPU interrupt per small audio sample; batch by period or descriptor.
3. Assign priority based on deadline and recovery cost, not simply device prestige.
4. Use counters/rings to preserve completion multiplicity if events can coalesce.
5. Route MSI writes through the correct IOMMU/firewall aperture.
6. Ensure CPU/DSP power-state transitions preserve or restore interrupt routing.
7. Measure worst-case service latency while camera and PCIe traffic saturate the NoC.
8. Distinguish “interrupt delivered” from “DMA descriptor consumed” and “audio period completed.”

For 48 kHz audio with a 1 ms period, the software has a nominal 1 ms period cadence, but the safe processing budget is smaller because buffer handoff, scheduling, cache maintenance and downstream processing consume time.

## 14. Historical and architectural comparison

| Family | Core model | Strength | Trade-off / context |
|---|---|---|---|
| Intel 8259A PIC | 8 wired inputs, IRR/ISR/IMR, fixed/rotating priority, cascade | Simple, compact, programmable | Limited scaling and multicore awareness |
| 8253/8254 PIT | Three independent programmable counters | Reusable periodic/event timing | External clock assumptions and limited width |
| x86 APIC/IOAPIC | Local CPU interrupt controllers plus I/O routing; MSI support in later systems | Multicore routing and device-message support | Platform topology and compatibility complexity |
| Arm GICv2 | Distributor + CPU interface | Multicore, priority, security and virtualization evolution | CPU interface model differs from later GICv3 system-register model |
| Arm GICv3/v4 | Distributor + per-PE Redistributor + system-register CPU interface; MSI/LPI and virtualization evolution | Scalable affinity, MSI and virtual interrupt delivery | More state, firmware setup and ITS/redistributor integration |
| RISC-V PLIC | Wired source gateway, priorities, per-context enables/threshold, claim/complete | Straightforward platform-level interrupt model | Large per-context structures; less direct virtual MSI model |
| RISC-V AIA | APLIC + per-hart IMSIC and interrupt files | Wired-to-MSI conversion, scalable per-hart and virtual delivery | More sophisticated address, virtualization and software setup |
| 8253/8254 PIT | Programmable count modes | General purpose interval generation | Coarse legacy platform timebase |
| Arm Generic Timer | System counter + physical/virtual compare channels | Stable architectural timebase across OS/virtualization | Requires platform frequency and power-domain integration |
| RISC-V mtime | 64-bit constant-frequency counter + mtimecmp | Simple architectural timer contract | MMIO split-access concerns on RV32; platform timebase remains a system integration responsibility |

**Design-rationale summary:** central arbitration is easy to understand and integrate in a small system. Distributed per-CPU state, message-based interrupts and virtual interrupt files are more scalable but increase configuration, coherency, security and debug complexity. A system should select the simplest architecture that meets source count, CPU count, virtualization, latency and power requirements.

## 15. Hands-on labs

### Lab A — Build a small interrupt controller
Implement 8 sources with edge/level configuration, enable, pending, priority and one CPU output. Add claim/complete or acknowledge/EOI behavior. Verify simultaneous requests, masking and re-pending.

### Lab B — Build a compare timer
Implement a 64-bit free-running counter, compare register, sticky expiration status and enable. Add a prescaler. Test exact match, rollover and compare updates near expiration.

### Lab C — Measure interrupt latency
Create a test source in one clock domain, route it through a synchronizer/controller model, and timestamp source and handler entry. Sweep source pulse width and phase relative to destination clock.

### Lab D — AON wakeup experiment
Gate the main clock while leaving the AON timer active. Program a wake deadline, enter low power, and verify wake request, reset/clock sequencing and interrupt delivery after resume.

### Lab E — Debug a level interrupt storm
Intentionally omit peripheral status clear. Observe repeated handler entry. Then clear the device before EOI and prove the line deasserts.

### Lab F — RV32 timer compare race
Simulate a 64-bit counter with a 32-bit MMIO interface. Demonstrate the torn compare write hazard and verify the specification's safe write sequence.

## 16. Interview questions

1. What is the difference between an interrupt source, interrupt controller and CPU exception?
2. When should a peripheral use edge-triggered versus level-triggered signaling?
3. Why can a one-bit pending register lose event multiplicity?
4. What is the difference between pending, active and active-and-pending?
5. Why are claim and completion separate in a PLIC?
6. What is the difference between GIC priority drop and interrupt deactivation?
7. Why must a level source be cleared before unmasking/re-enabling it?
8. What are SGI, PPI, SPI and LPI in the GIC model?
9. How does an MSI differ from a dedicated interrupt wire?
10. What are the APLIC and IMSIC roles in RISC-V AIA?
11. Why does a timer need an always-on clock domain?
12. What is the difference between timer resolution and accuracy?
13. Why can an RV32 write to mtimecmp create a spurious interrupt?
14. How do you calculate source-to-handler latency?
15. How can a high-priority interrupt starve lower-priority sources?
16. What happens if a controller is reset while the peripheral source remains asserted?
17. Why is a readback sometimes required after a timer register write across clock domains?
18. How would you debug an interrupt that works in simulation but fails after deep idle?
19. Why should watchdog bark and bite be separate escalation stages?
20. How would you validate interrupt latency under NoC and DMA saturation?

## 17. Design-review checklist

- [ ] Every interrupt source has documented polarity, trigger type, pulse width and clock domain.
- [ ] Edge-event multiplicity is preserved where required.
- [ ] Level source clear semantics are explicit.
- [ ] Interrupt ID, priority, security group and target affinity are documented.
- [ ] Claim/acknowledge and EOI/deactivate/complete protocol is implemented correctly.
- [ ] MSI address/data, IOMMU and firewall permissions are verified.
- [ ] Timer effective frequency and accuracy are documented.
- [ ] Compare programming is safe for the CPU register width.
- [ ] Timer continues or intentionally pauses across each low-power state.
- [ ] Watchdog service window, bark, bite and reset-cause behavior are specified.
- [ ] Reset behavior for pending/active/compare state is specified.
- [ ] CDC/RDC and AON wakeup paths are verified.
- [ ] Worst-case latency and jitter budgets include masking and fabric contention.
- [ ] Post-silicon observability includes pending/active state and timestamps.

## 18. Normative specifications versus implementation examples

| Source | Classification | Use in this lesson |
|---|---|---|
| Arm IHI 0069 GIC Architecture Specification | Normative architecture | GIC interrupt states, registers, priority and deactivation |
| RISC-V Privileged Architecture | Normative ISA/platform contract | mtime, mtimecmp and MTIP behavior |
| RISC-V PLIC specification | Normative platform-level architecture | Priority, pending, enable, threshold, claim and completion |
| RISC-V AIA 1.0 | Ratified architecture (June 2023) | APLIC, IMSIC, MSI and virtual interrupt architecture |
| Intel 1979 Peripheral Design Handbook | Historical primary-source documentation | 8253 and 8259A architecture and feature context |
| OpenTitan AON Timer / rv_timer docs | Public implementation example | AON clocking, timer prescaler, wakeup and watchdog integration |
| Linux generic IRQ docs | Software framework documentation | Edge/level flow handlers, threaded IRQs and driver lifecycle |
| Arm GICv3/v4 overview | Explanatory vendor guide | Generation comparison and bring-up overview |

## 19. References

### Historical primary sources
- [H1] Intel, *Peripheral Design Handbook*, February 1979, 9800676B — 8253/8253-5 data sheet and application material: https://bitsavers.trailing-edge.com/components/intel/_dataBooks/1979_Intel_Peripheral_Design_Handbook.pdf
- [H2] Intel, *8259A Programmable Interrupt Controller* data sheet (archival copy): https://pdos.csail.mit.edu/6.828/2008/readings/hardware/8259A.pdf

### Architecture specifications and official guides
- [H3] Arm, *Learn the architecture — Generic Interrupt Controller v3 and v4*, Issue 3.2-03, 22 April 2025: https://documentation-service.arm.com/static/68077c770887e84c5ae973da
- [H4] RISC-V International, *Platform-Level Interrupt Controller Specification*: https://docs.riscv.org/reference/plic/
- [H5] RISC-V PLIC claim and completion chapters: https://docs.riscv.org/reference/plic/plic-claims.html and https://docs.riscv.org/reference/plic/plic-completion.html
- [H6] RISC-V International, *Advanced Interrupt Architecture 1.0*, ratified June 2023: https://docs.riscv.org/reference/aia/
- [H7] RISC-V AIA introduction and wired/MSI architecture: https://docs.riscv.org/reference/aia/intro.html
- [H8] Arm A-profile Architecture Reference Manual, Generic Timer system counter: https://support.arm.com/documentation/ddi0487/mc/-Part-D-The-AArch64-System-Level-Architecture/-Chapter-D12-The-Generic-Timer-in-AArch64-state/-D12-1-About-the-Generic-Timer/-D12-1-2-The-system-counter
- [H9] RISC-V Privileged Architecture, machine timer registers: https://docs.riscv.org/reference/isa/priv/machine.html
- [H10] Arm GIC system-register acknowledge/EOI architecture guide: https://documentation-service.arm.com/static/6012f0024ccc190e5e68124f
- [H11] Arm CNTP_TVAL_EL0 register description: https://support.arm.com/documentation/ddi0601/2025-03/AArch64-Registers/CNTP-TVAL-EL0--Counter-timer-Physical-Timer-TimerValue-Register

### Public implementation and software examples
- [H12] OpenTitan AON Timer theory of operation: https://opentitan.org/book/hw/ip/aon_timer/doc/theory_of_operation.html
- [H12a] OpenTitan rv_timer theory of operation: https://opentitan.org/book/hw/ip/rv_timer/doc/theory_of_operation.html
- [H12b] OpenTitan rv_timer programmer's guide: https://opentitan.org/book/hw/ip/rv_timer/doc/programmers_guide.html
- [H13] Linux kernel generic IRQ handling: https://www.kernel.org/doc/html/latest/core-api/genericirq.html
- [H14] Intel 8254 timer implementation documentation in a modern Intel chipset datasheet: https://edc.intel.com/content/www/us/en/design/ipla/software-development-platforms/servers/platforms/intel-pentium-silver-and-intel-celeron-processors-datasheet-volume-1-of-2/006/8254-timers/

---

## 20. Session summary

The essential engineering contract is:

- The source must create an event that survives the relevant clock-domain crossing.
- The controller must preserve, prioritize and route it according to its architecture.
- The CPU must accept it according to current mask/priority/privilege state.
- Software must clear the device cause and complete the controller protocol correctly.
- The timer must maintain a known timebase across clock and power transitions, or explicitly document when time pauses.
- Verification and silicon debug must observe the whole path, not only the CPU interrupt pin.

**Next session:** Lesson 18 — GPIO and Pin Control: pad ring, electrical behavior, muxing, pulls, drive strength, debounce, interrupts, boot straps and silicon bring-up.
