# Lesson 16 — Power Management: Domains, UPF, DVFS, Idle Handshakes and Wakeup

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 16 of 38  
**Duration:** 60 minutes  
**Level:** SoC architecture → low-power RTL and power intent → verification/signoff → post-silicon characterization  
**Prerequisites:** Lessons 01–15; CMOS power, clock/reset architecture, AXI/NoC, DMA, cache coherency, CDC/RDC and basic Linux power management.

> **Central engineering question:** How can an SoC reduce energy by stopping clocks, lowering voltage or removing power from selected logic while preserving architectural state, transaction integrity, security and a bounded wake-up latency?

Power management is not one register or one FSM. It is a contract shared by RTL, power intent, physical design, package/board power delivery, firmware, operating system, peripherals and verification. A domain is safe to power down only when its consumers are quiescent, its state has been saved or declared disposable, its outputs are isolated, and its power/clock/reset dependencies are understood.

---

## 60-minute lesson plan

| Time | Topic | Engineering outcome |
|---:|---|---|
| 00–05 min | Recap and power vocabulary | Separate clock gating, DVFS and power gating |
| 05–12 min | Historical motivation and power equations | Understand why power became a first-class architecture constraint |
| 12–20 min | Domains, switches, isolation and retention | Read a power-domain block diagram |
| 20–29 min | Power controller FSM and low-power interfaces | Trace entry, abort, shutdown and wakeup |
| 29–37 min | UPF and physical implementation | Connect logical intent to implementation cells |
| 37–44 min | DVFS, OPPs and software model | Calculate safe voltage/frequency transitions |
| 44–52 min | Worked DSP/audio case study | Plan quiesce, save, isolate, power-off and resume |
| 52–57 min | Verification and silicon debug | Build a power-aware test and measurement plan |
| 57–60 min | Interview questions and exercises | Check architecture-level understanding |

## 1. What power management controls

An SoC power-management architecture coordinates several different mechanisms. They solve different problems and must not be treated as synonyms.

| Mechanism | What changes | State retained? | Typical use |
|---|---|---|---|
| Fine-grain clock gating | Stops clock toggles to selected registers | Yes, while supply remains valid | Idle pipeline stage, FIFO, register bank |
| Root clock gating | Stops a clock branch or whole clock domain | Yes | Idle peripheral or subsystem |
| DVFS | Changes operating voltage and frequency | Yes | Match performance to workload |
| Power gating | Disconnects a switched supply from a domain | No, unless retained separately | Deep idle or unused accelerator |
| Retention | Keeps selected state on a retention supply or retention-capable cells | Selected state only | Fast resume from sleep |
| Isolation | Clamps signals crossing from an off/invalid domain | Not a storage mechanism | Prevent X/unknown or back-power propagation |
| Body bias / adaptive voltage techniques | Changes transistor threshold or operating margin | Yes | Process/temperature optimization in supported processes |

A design may combine all of these. For example, a DSP can first stop new work, drain DMA, gate local clocks, save a small context, isolate its boundary, then power-gate the compute domain while retaining a wake controller and selected SRAM.

**Architecture rule:** Clock gating reduces dynamic switching power. Power gating can also reduce leakage in the switched domain, but adds area, sequencing, wake latency and state-management complexity.

## 2. Historical motivation and design rationale

### 2.1 Why power became a system-level problem

**Documented milestone — 1974:** Robert Dennard and colleagues published *Design of Ion-Implanted MOSFETs With Very Small Physical Dimensions* in IEEE Journal of Solid-State Circuits. The paper studied scaling relationships and the fabrication/characterization of smaller MOSFETs. This work is a foundational source for the scaling ideas later associated with constant-field scaling. It is not a paper about modern SoC power controllers, but it helps explain the historical context in which transistor dimensions, voltage, capacitance and power were considered together.

Source: [IBM Research — Dennard et al., 1974](https://research.ibm.com/publications/design-of-ion-implanted-mosfets-with-very-small-physical-dimensions).

As integration grew, a chip could contain more transistors and more active functional units. Battery-powered products and thermal limits made it impractical to run every block at peak performance continuously. Leakage also became a larger concern as process technologies evolved. The engineering response was not a single invention: clock gating, multiple voltage domains, power gating, retention, DVFS, thermal control and software-managed idle states developed as a coordinated toolbox.

### 2.2 Power intent standardization

Low-power implementation became difficult to describe using RTL alone. RTL can describe functional behavior, but does not fully express supply nets, switchable domains, isolation strategy, retention supplies, power states or the relationship between logical instances and physical power-management cells.

- **2006–2007:** Accellera developed and published the first Unified Power Format (UPF) version, with the goal of describing power intent in a tool-independent way.
- **2009:** IEEE published IEEE Std 1801-2009, standardizing UPF.
- **2013–2014:** Subsequent UPF revisions expanded and clarified the power-intent methodology.
- **March 19, 2025:** Accellera announced that IEEE Std 1801-2024 (UPF 4.0) was available through the IEEE GET program. The announcement highlights supply-net tunneling, value-conversion methods, retention enhancements, refinable macros and supply abstractions.

Sources:
- [Accellera: IEEE 1801-2024 / UPF 4.0 announcement, March 19, 2025](https://vw.eda.org/news/press-releases/414-accellera-announces-ieee-standard-1801-2024-is-available-through-ieee-get-program)
- [Semiconductor Engineering: UPF overview and history](https://semiengineering.com/knowledge_centers/standards-laws/standards/unified-power-format/)

**Design rationale:** A common power-intent language lets IP providers describe requirements and SoC integrators refine them. It improves consistency across power-aware simulation, synthesis and physical implementation. UPF specifies intent; it does not mandate one transistor-level implementation or one universal power sequence.

### 2.3 Software-visible power states

The operating system also needs an abstraction for power/performance choices.

- Linux CPUIdle separates generic idle-state selection from the platform driver that enters the hardware state. States are characterized by target residency and exit latency.
- Linux OPP represents supported operating performance points as frequency/voltage pairs associated with a device or performance domain.
- Hardware low-power interfaces, such as Arm Q-Channel and P-Channel, provide standardized request/accept/deny/activity handshakes between IP and power/clock controllers.

Sources:
- [Linux CPUIdle documentation](https://www.kernel.org/doc/html/v6.6/driver-api/pm/cpuidle.html)
- [Linux OPP documentation](https://cdn.kernel.org/doc/html/latest/power/opp.html)
- [Arm CoreLink DMA-350 low-power interfaces](https://support.arm.com/documentation/102482/latest/DMAC-interfaces/LPI-interfaces)
- [Arm CoreLink CCI-500 clock and power control interfaces](https://support.arm.com/documentation/100023/0100/functional-description/interfaces/clock-and-power-control-interfaces)

The exact state names, register model and sequencing remain implementation-specific.

## 3. Power equations every SoC engineer should know

### 3.1 Dynamic power

A first-order CMOS switching-power model is:

\[
P_{dyn} = \alpha C_{eff} V^2 f
\]

where:

- \(\alpha\) is the activity factor (effective switching probability per cycle under the chosen convention).
- \(C_{eff}\) is effective switched capacitance.
- \(V\) is supply voltage.
- \(f\) is clock frequency.

This model is useful for architecture comparisons. Real power estimation must account for internal cell power, glitching, clock-tree activity, memory macros, IO loads, short-circuit current and actual activity.

### 3.2 Leakage and total power

\[
P_{leak} = V \cdot I_{leak}
\]

\[
P_{total} \approx P_{dyn} + P_{leak} + P_{short} + P_{other}
\]

Leakage depends strongly on process, voltage, temperature, threshold options, state and power-gating implementation. Do not assume it is a constant percentage of total power.

### 3.3 Worked dynamic-power comparison

Assume one illustrative domain has:

- Effective switched capacitance: 20 nF
- Activity factor: 0.15
- OPP-A: 0.9 V at 500 MHz
- OPP-B: 1.0 V at 800 MHz

At OPP-A:

\[
P_A = 0.15 \times 20\,nF \times (0.9)^2 \times 500\,MHz
= 1.215\,W
\]

At OPP-B:

\[
P_B = 0.15 \times 20\,nF \times (1.0)^2 \times 800\,MHz
= 2.4\,W
\]

The illustrative dynamic-power ratio is:

\[
P_B/P_A \approx 1.975
\]

A 60% frequency increase caused almost a 2× dynamic-power increase in this example because voltage also increased. These are model values, not measured silicon results.

### 3.4 Why voltage matters so much

If frequency is approximately proportional to voltage over a limited operating region, then:

\[
P_{dyn} \propto V^3
\]

This is a useful intuition, not a universal law. Real frequency/voltage curves are constrained by process corners, temperature, path delay, SRAM limits, PLL ranges, IR drop, aging and reliability margins.

### 3.5 Break-even time for a deep idle state

Power gating has transition energy and wake latency. A simple energy break-even estimate is:

\[
T_{break-even} =
\frac{E_{entry}+E_{exit}}{P_{shallow}-P_{deep}}
\]

Example:

- Combined entry and exit energy: 12 µJ
- Power saved versus shallow idle: 300 mW

\[
T_{break-even} = \frac{12\,\mu J}{0.3\,W}=40\,\mu s
\]

If the expected idle interval is shorter than about 40 µs, this simplified model says the deep state may not save energy. Real policy must include uncertainty in idle duration, wakeup probability, transition costs, rail behavior and latency constraints.

## 4. Power-domain architecture

A power domain is a set of design elements that share a defined supply behavior and power-state policy. A practical SoC may contain:

- **AON (always-on):** wakeup detectors, PMU, slow timer, reset cause, selected interrupt logic.
- **CPU domain:** CPU cores and local caches.
- **System/interconnect domain:** NoC routers, bridges, system control.
- **DSP/audio domain:** DSP core, local SRAM, audio processing IP.
- **GPU/NPU domain:** compute arrays and local memories.
- **Peripheral domain:** I2C, SPI, UART, GPIO controllers.
- **Memory domain:** SRAM banks or external-memory controller/PHY logic.
- **IO/PHY domains:** pad ring, USB, PCIe, Ethernet or display PHY analog/digital sections.

These are examples, not a recommended universal partition. The best partition depends on traffic, state-retention cost, wake latency, physical floorplan, supply availability and whether the blocks can be independently quiesced.

### 4.1 Example domain hierarchy

~~~mermaid
flowchart TB
  PMU["Always-on PMU / wake controller"]
  AON["AON supply and slow clock"]
  CPU["CPU + private cache"]
  NOC["NoC / system fabric"]
  DSP["DSP + local SRAM"]
  GPU["GPU / NPU"]
  PER["Peripheral cluster"]
  DDR["DDR controller + PHY"]
  IO["IO / SerDes PHY domains"]
  PMU --> AON
  AON --> CPU
  AON --> NOC
  AON --> DSP
  AON --> GPU
  AON --> PER
  AON --> DDR
  AON --> IO
  CPU <--> NOC
  DSP <--> NOC
  GPU <--> NOC
  PER <--> NOC
  NOC <--> DDR
~~~

The drawing shows conceptual control and traffic relationships only. A real design can have multiple rails, nested domains, retention rails, shared regulators and dependencies that do not match this simplified tree.

### 4.2 Power-domain dependency graph

A domain can be powered down only if the dependencies required for safe operation are satisfied.

Example:

- A DSP may depend on its local SRAM, NoC access, reference clock, reset controller and audio wake source.
- The AON domain must remain powered if it is responsible for detecting wakeup.
- If the DSP shares a PLL with another active subsystem, the PLL cannot simply be disabled when the DSP sleeps.
- If a shared SRAM is used by another master, it cannot be power-gated just because the DSP is idle.

**Integration question:** Which block owns the state and who is allowed to turn off the resource? Ownership must be explicit.

## 5. Physical mechanisms: switches, isolation, retention and level shifting

### 5.1 Power switches

Power gating inserts header or footer switch cells between a real supply and a virtual supply used by a domain.

- A header implementation uses high-side switching devices.
- A footer implementation uses low-side switching devices.
- Large domains may use many distributed switch cells, often enabled in stages.
- The virtual supply must rise and settle before logic is treated as powered and valid.

Important physical effects:

- Switch resistance causes voltage drop under load.
- Simultaneously enabling many switches can create inrush current.
- Inrush can cause supply droop, ground bounce and disturbance to neighboring domains.
- Switch sizing trades area, IR drop, wake time and leakage.
- Power-grid routing and decoupling must be designed with the switching profile.

### 5.2 Isolation cells

When a source domain is off, its outputs can become unknown or electrically invalid. Isolation cells clamp signals crossing into an active domain.

Typical clamp choices:

- Clamp to logic 0.
- Clamp to logic 1.
- Retain or clamp selected signals according to a library-specific cell and design intent.

Isolation is a boundary function. It does not save state, finish bus transactions or make an unsafe shutdown sequence safe.

### 5.3 Retention cells

Retention flops or retention-capable memories preserve selected state using an always-available retention supply or a special retention mechanism.

Questions to answer:

- Which registers are architecturally required after wake?
- Which state can be reconstructed from software or memory?
- What is the save/restore control polarity and timing?
- Is the retention rail guaranteed during the entire off interval?
- What happens if power is lost during save or restore?
- Does a retained register require reset after restore, or must its value survive reset?

Retention costs area and leakage. Retaining too much state can erase the energy advantage of power gating.

### 5.4 Level shifters

Signals crossing voltage domains may need level shifting. The direction and cell type depend on source/destination voltage, library characterization and whether the path is unidirectional or bidirectional.

A power-intent review must identify:

- High-to-low and low-to-high crossings.
- Signals that remain active while one side is off.
- Isolation plus level-shifter ordering.
- Control-signal crossings for isolation, retention and power-good.
- Analog/IO interfaces with separate supply behavior.

### 5.5 Conceptual boundary

~~~mermaid
flowchart LR
  subgraph AON["Always-on domain"]
    PMU["PMU / controller"]
    WAKE["Wake detector"]
  end
  subgraph SW["Switchable domain"]
    ISO["Isolation boundary"]
    CORE["DSP / accelerator logic"]
    RET["Retention state"]
  end
  VDD["Main supply"] --> SW
  RVDD["Retention supply"] --> RET
  AVDD["AON supply"] --> AON
  PMU --> ISO
  PMU --> CORE
  WAKE --> PMU
  CORE --> ISO
~~~

The diagram is conceptual. Actual isolation cells are inserted on the crossing paths and the switched supply is implemented in the physical power-distribution network.

## 6. Power controller architecture and state machine

The PMU/power controller usually combines:

- An always-on clock and reset.
- A register interface, commonly APB or another low-bandwidth control bus.
- Wakeup and interrupt inputs.
- Software requests and hardware activity indications.
- Power-switch control outputs.
- Isolation and retention controls.
- Clock-manager and reset-manager interfaces.
- Power-good, regulator-ready and PLL-lock inputs.
- Timeout, error and reset-cause recording.

### 6.1 Example power-down state machine

~~~mermaid
stateDiagram-v2
  [*] --> ACTIVE
  ACTIVE --> QUIESCE: low-power request
  QUIESCE --> ABORT: wake / new work / timeout
  ABORT --> ACTIVE
  QUIESCE --> DRAIN: clients stopped
  DRAIN --> ABORT: outstanding work remains
  DRAIN --> SAVE: fabric and DMA idle
  SAVE --> ISOLATE: retention saved
  ISOLATE --> CLOCK_STOP: boundary safe
  CLOCK_STOP --> POWER_OFF: clocks stopped
  POWER_OFF --> OFF: switch disabled and status valid
  OFF --> POWER_ON: wake event
  POWER_ON --> POWER_GOOD: supply stable
  POWER_GOOD --> RESTORE: clocks and rails ready
  RESTORE --> CLOCK_START: retained state restored
  CLOCK_START --> RELEASE: clocks stable
  RELEASE --> ACTIVE: reset and isolation sequence complete
~~~

The state names and order are implementation-specific. The important property is that the controller does not remove power until the domain is quiescent and its outputs are safe.

### 6.2 Entry sequence: engineering checklist

1. Software or hardware requests a target state.
2. PMU checks policy, dependencies and wakeup configuration.
3. IP clients receive quiesce requests.
4. New DMA/NoC work is blocked or held.
5. Outstanding transactions are drained or explicitly canceled by a documented protocol.
6. Cache and memory state is cleaned, retained or invalidated as required.
7. IP reports idle/accepts the low-power request.
8. Required state is saved to retention.
9. Isolation is asserted at the domain boundary.
10. Clocks are stopped only after the IP and clock controller agree that it is safe.
11. Power switches are disabled according to the physical sequence.
12. PMU records the entered state and wake configuration.

A controller must support abort/fall-through behavior. A wakeup arriving during entry can require the sequence to return to ACTIVE without reaching OFF.

### 6.3 Exit sequence: engineering checklist

1. A retained AON wake detector captures the event.
2. PMU requests the required supply.
3. Power switch is enabled.
4. Wait for power-good and any required rail-settling interval.
5. Enable required clocks and wait for PLL/clock-valid.
6. Restore retained state.
7. Apply reset release according to the domain's reset contract.
8. Deassert isolation only when source outputs are valid.
9. Re-enable clients and traffic.
10. Record wake cause and resume software.

**Important:** The relative order of reset release, retention restore, clock start and isolation release is technology- and IP-dependent. The implementation must use the retention-cell, macro, reset and library requirements. Do not copy a generic order without checking the actual macro timing.

## 7. Low-power handshakes at IP boundaries

A power controller cannot safely shut down an IP merely because software wrote a bit. It needs a protocol for the IP to indicate whether it is idle and whether it can accept a state change.

### 7.1 Arm Q-Channel and P-Channel concepts

Arm's Low Power Interface family defines handshakes for managing component clock and power states.

- **Q-Channel:** used for clock-state quiescence/control.
- **P-Channel:** used for power-state transitions, including more than a simple clock on/off decision.
- Activity indications such as QACTIVE or PACTIVE communicate that a component is active or needs the relevant resource.
- Request/accept/deny behavior allows the component to refuse a transition when it is not safe.

The exact signal set and semantics must be taken from the applicable Arm Low Power Interface specification revision and the component integration guide. Do not assume every Arm IP implements every optional signal.

Useful public implementation references:
- [Arm CoreLink DMA-350 LPI interfaces](https://support.arm.com/documentation/102482/latest/DMAC-interfaces/LPI-interfaces)
- [Arm CoreLink CCI-500 clock and power control interfaces](https://support.arm.com/documentation/100023/0100/functional-description/interfaces/clock-and-power-control-interfaces)
- [Arm CoreLink PCK-600 low-power integration overview](https://developer.arm.com/community/arm-community-blogs/b/internet-of-things-blog/posts/new-pck-600-for-efficient-and-rapid-integration-of-soc-power-management-infrastructure)

### 7.2 Why an idle indication is not enough

An IP can report internal idle while an external transaction remains outstanding. For example:

- DMA has no new descriptor but an AXI write response is still pending.
- A NoC request was accepted but has not completed.
- A cache writeback is still in progress.
- An audio FIFO is empty but the codec clock must continue for a frame boundary.
- A peripheral has no internal work but a wake event is pending in a different clock domain.

The low-power contract must define exactly what idle means and whether it includes downstream transactions.

### 7.3 Example handshake trace

The following is conceptual rather than a normative waveform for one named interface.

| Phase | PMU request | IP response | Meaning |
|---|---|---|---|
| Run | No request | Active | IP operates normally |
| Request | Quiesce requested | Active | IP stops accepting new work |
| Drain | Request held | Busy | Existing work completes |
| Accept | Request held | Quiescent/accept | Safe boundary reached |
| Gated | Clock/power transition | No activity | Controller can proceed |
| Wake | Resource-on request | Activity request | IP asks for resource |
| Resume | Clock/power restored | Active | IP resumes work |

The request must remain asserted until the documented handshake completes. A one-cycle pulse is unsafe unless the interface explicitly defines pulse semantics.

## 8. UPF: expressing power intent

IEEE 1801 UPF is the standard power-intent language used to describe supply networks, power domains, power states, isolation, retention, power switches and related low-power behavior.

### 8.1 What RTL does and does not say

RTL describes functional logic and its behavior when powered and clocked. Power intent adds information about:

- Which instances belong to which power domain.
- Which supplies power those domains.
- Which domains can be switched off.
- Which signals require isolation.
- Which state elements require retention.
- Which power states are legal.
- How abstract power intent is refined into implementation cells.

### 8.2 Conceptual UPF model

The following is an architecture sketch, not copy/paste-ready UPF syntax. Exact commands, options and semantics depend on the IEEE 1801 revision and EDA-tool support.

~~~tcl
# Conceptual intent only — adapt to IEEE 1801 revision and tool syntax.
# 1. Declare top-level supply ports/nets and supply sets.
# 2. Create AON and switchable power domains.
# 3. Assign RTL instances to domains.
# 4. Associate primary and retention supplies.
# 5. Define the switchable domain's legal power states.
# 6. Define isolation strategy for outputs leaving the switchable domain.
# 7. Define retention strategy for state that must survive power-off.
# 8. Define power-switch strategy and control mapping.
# 9. Define level-shifter strategy for voltage crossings.
# 10. Validate power-state transitions and all crossing paths.
~~~

A real project should use the exact UPF version selected by the methodology and run the EDA tool's UPF parser/checker. Do not mix command syntax from different UPF revisions without checking compatibility.

### 8.3 UPF verification and implementation flow

~~~mermaid
flowchart TD
  RTL["RTL + IP hierarchy"] --> UPF["UPF power intent"]
  UPF --> PA["Power-aware simulation"]
  UPF --> SYN["Synthesis / low-power mapping"]
  SYN --> PNR["Place and route"]
  PNR --> STA["Timing / power / IR / EM signoff"]
  PA --> COV["Power-state and crossing coverage"]
  STA --> GDS["Final implementation"]
  COV --> SIGNOFF["Low-power signoff"]
  GDS --> SIGNOFF
~~~

UPF is not a replacement for RTL verification, CDC/RDC, static timing analysis, IR-drop analysis or physical power signoff. It connects the power architecture to these flows.

## 9. DVFS and operating performance points

Dynamic voltage and frequency scaling changes the operating point of a performance domain while maintaining functional and timing correctness.

### 9.1 OPP model

An OPP is a supported voltage/frequency operating point. A simple example:

| OPP | Frequency | Minimum voltage (illustrative) | Intended mode |
|---|---:|---:|---|
| 0 | 300 MHz | 0.72 V | Low load |
| 1 | 600 MHz | 0.82 V | Normal load |
| 2 | 900 MHz | 0.94 V | High performance |
| 3 | 1.2 GHz | 1.05 V | Burst performance |

These values are fictional and do not describe a real process or chip. Real OPP tables are characterized across process, voltage, temperature, aging and workload constraints.

### 9.2 Safe transition direction

A common safe policy is:

- **Raising performance:** raise voltage to the required level, wait for regulator settling/power-good, then raise frequency or switch to the faster clock.
- **Lowering performance:** lower frequency first, then reduce voltage once the slower operating point is active.

Why? Raising frequency before voltage is sufficient can violate setup timing. Lowering voltage before reducing frequency can also violate timing.

A real implementation may use a safe intermediate clock, PLL reprogramming, clock mux handshakes and regulator/PMIC acknowledgements. The sequence must include timeout and rollback behavior.

### 9.3 PLL and clock dependencies

DVFS may require:

- Selecting a safe parent clock before PLL reconfiguration.
- Programming PLL feedback/reference dividers.
- Waiting for lock and stability qualification.
- Switching to the target clock through a glitch-safe mux.
- Updating bus/peripheral dividers.
- Updating timing-sensitive memory-controller settings if the memory clock changes.
- Verifying that asynchronous interfaces remain safe during the transition.

Not every DVFS event changes the PLL. Some systems use dividers or independent PLLs. The clock architecture determines the sequence.

### 9.4 Linux software model

Linux commonly uses:

- **CPUFreq** for CPU performance/frequency policy.
- **OPP** for supported voltage/frequency pairs.
- **CPUIdle** for idle-state selection.
- **Runtime PM** for per-device activity-based power management.
- **System suspend** for coordinated system-wide low-power entry.

CPUIdle states are characterized by target residency and exit latency. The governor selects an appropriate state based on predicted idle duration and constraints. The platform driver performs the hardware-specific entry.

The Linux OPP library represents operating points as frequency/voltage pairs per device/domain. The SoC framework can make OPPs available or unavailable, for example in response to thermal constraints.

## 10. Idle-state policy and energy break-even

A shallow state may stop a clock but leave the supply on. A deeper state may power-gate a subsystem but have longer wake latency and greater transition energy.

The OS/firmware policy should compare:

- Expected idle duration.
- Entry latency.
- Exit latency.
- Transition energy.
- Power saved in the state.
- Wakeup-source availability.
- Shared-domain dependencies.
- Latency QoS constraints.

A deep state is not automatically better. If the system wakes quickly, the energy spent entering and exiting may exceed the energy saved.

### 10.1 Worked wake-latency budget

Illustrative deep-sleep exit:

| Component | Time |
|---|---:|
| Power switch and rail ramp/settle | 18 µs |
| Clock/PLL stabilization | 35 µs |
| Retention restore | 8 µs |
| Reset/isolation release and IP initialization | 12 µs |
| **Total** | **73 µs** |

This is a simple additive budget. Real wake paths can overlap some steps, and worst-case timing must include voltage, temperature, process, regulator behavior and software latency.

## 11. Register model and programming sequence

There is no universal PMU register map. The following is a generic register architecture for learning and review.

| Register (illustrative offset) | Purpose |
|---:|---|
| 0x000 | POWER_REQ — request target domain/state |
| 0x004 | POWER_STATUS — current/acknowledged state |
| 0x008 | DOMAIN_ENABLE — permitted domain transitions |
| 0x00C | WAKE_ENABLE — wake source mask |
| 0x010 | WAKE_STATUS — latched wake causes |
| 0x014 | IDLE_STATUS — IP quiescent/active status |
| 0x018 | RETENTION_CTRL — save/restore control |
| 0x01C | ISOLATION_STATUS — boundary isolation state |
| 0x020 | POWER_GOOD_STATUS — rail readiness |
| 0x024 | CLOCK_STATUS — clock/PLL readiness |
| 0x028 | OPP_REQUEST — desired performance point |
| 0x02C | OPP_STATUS — achieved performance point |
| 0x030 | ERROR_STATUS — timeout/denial/fault |
| 0x034 | RESET_CAUSE — retained reset/wakeup context |

These offsets, names and bit meanings are illustrative only. A real PMU register specification must define access permissions, reset values, write-one-to-clear behavior, atomicity, clock-domain synchronization, timeout semantics and security attribution.

### 11.1 Generic software entry sequence

1. Stop or pause clients that cannot tolerate the target state.
2. Mask non-wakeup interrupts at the relevant interrupt sources.
3. Configure and enable wake sources in the always-on domain.
4. Ensure DMA descriptors and memory buffers are complete and visible.
5. Request device/runtime suspend or system idle through the OS/firmware contract.
6. Program the desired low-power state.
7. Set the low-power hint or request.
8. Execute the architecture's wait-for-interrupt/idle instruction where required.
9. On return, read the wake reason and transition result.
10. Reinitialize blocks that lost state and restore drivers/streams.

The OpenTitan power-manager programmer guide is a useful public example: it documents wakeup/reset enables, a low-power hint, CDC synchronization and WFI. It explicitly warns that WFI alone does not guarantee low-power entry; the sleeping indication must be sampled and entry may fall through or abort.

Reference: [OpenTitan power-manager programmer guide](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/pwrmgr/doc/programmers_guide.html).

## 12. Worked SoC case study: power-gating an audio DSP

Consider a subsystem containing:

- Audio DSP core.
- Local SRAM and instruction/data memories.
- SoundWire/I2S/TDM interfaces.
- DMA engine connected to NoC.
- Interrupt and timer sources.
- Shared PLL and shared DRAM controller.
- Always-on wake detector.

### 12.1 The key architecture decision

If an audio stream is active, the DSP may not be eligible for deep power-off. The system can instead use a shallow clock-gated state or move processing to another engine. If the stream is stopped, the DSP can be quiesced and power-gated.

The shared PLL or DDR controller must not be disabled if another active subsystem depends on it.

### 12.2 Safe entry sequence

1. Driver stops accepting new audio periods.
2. Audio framework stops or drains the stream at a valid frame/period boundary.
3. DMA stops fetching new descriptors.
4. Outstanding AXI reads/writes and responses drain.
5. Software synchronizes non-coherent buffers and descriptors as required.
6. DSP firmware reaches a safe point and reports context-save complete.
7. Required DSP context and selected SRAM contents are retained or copied.
8. PMU asserts isolation on DSP-to-NoC and DSP-to-audio control paths.
9. DSP clocks are stopped.
10. DSP power switch is disabled.
11. AON wake detector and required codec/wake path remain operational.

### 12.3 Safe exit sequence

1. Wake event is captured by AON logic.
2. PMU enables DSP supply and waits for power-good.
3. Required clock source is enabled and stabilized.
4. Retention state is restored.
5. Reset is released according to the DSP integration contract.
6. Isolation is removed after outputs are valid.
7. DSP firmware resumes and validates local state.
8. DMA descriptors and audio stream state are reconstructed.
9. Driver restarts the stream only after codec, clocks and datapath are ready.

### 12.4 Common failure signatures

| Symptom | Likely investigation |
|---|---|
| First audio samples corrupted after wake | Retention restore, SRAM initialization, stale DMA cache lines |
| DSP never acknowledges idle | Pending mailbox, interrupt, DMA response or firmware task |
| NoC hangs after wake | Isolation ordering, reset release, outstanding transaction bookkeeping |
| Wake interrupt is lost | Wake source not in AON domain, edge not latched, CDC pulse too short |
| PLL fails to lock | Supply ramp, reference clock, PLL configuration, power-good sequencing |
| High current in deep sleep | Unswitched leakage path, clock still toggling, IO clamp or back-power path |
| Intermittent failures only at low voltage | IR drop, marginal regulator transient, retention or timing margin |
| Stream resumes with wrong frame phase | Codec clock/frame synchronization and restart sequencing |

## 13. Power verification strategy

### 13.1 Directed tests

- Enter and exit every supported power state.
- Abort entry at every transition boundary.
- Assert wakeup just before, during and after isolation.
- Wake during power-switch ramp.
- Reset during retention save and restore.
- Deny a low-power request while an IP is active.
- Inject power-good timeout and PLL-lock timeout.
- Test simultaneous wake sources.
- Exercise every legal and illegal power-state transition.
- Verify reset cause and wake cause survive the intended state.

### 13.2 Assertions and properties

Examples of properties to implement in the project's assertion language:

~~~systemverilog
// Conceptual properties. Adapt signal names and sampling clocks.
property no_power_off_before_quiescent;
  @(posedge aon_clk) disable iff (!aon_rst_n)
    power_switch_off |-> (domain_quiescent && outstanding_count == 0);
endproperty

property isolation_before_switch_off;
  @(posedge aon_clk) disable iff (!aon_rst_n)
    power_switch_off |-> isolation_active;
endproperty

property no_resume_before_power_good;
  @(posedge aon_clk) disable iff (!aon_rst_n)
    domain_active |-> power_good;
endproperty

property wake_is_recorded;
  @(posedge aon_clk) disable iff (!aon_rst_n)
    wake_event |-> ##[0:WAKE_CAPTURE_MAX] wake_status;
endproperty
~~~

These are review examples, not a complete signoff suite. Real properties must account for legal sequencing, synchronizer latency, reset behavior, power-aware simulation semantics and the clock that remains active in each state.

### 13.3 Coverage model

Cover at least:

- Every domain and supported power state.
- Every legal state transition.
- Every abort and timeout path.
- Every wake source individually and in selected combinations.
- Every retention group save/restore.
- Every isolation boundary.
- Every voltage/frequency OPP transition.
- Concurrent DMA, NoC and interrupt activity at low-power request time.
- Reset and error injection during each transition phase.

### 13.4 Formal and static checks

- UPF consistency and power-domain membership.
- Isolation completeness and clamp correctness.
- Retention mapping and supply connectivity.
- Level-shifter coverage.
- No illegal power-state transition.
- No active transaction crossing into an off domain.
- No deadlock in request/acknowledge state machines.
- No lost wake event under the documented assumptions.
- CDC/RDC and reset recovery/removal checks.
- Power-aware X-propagation and corruption checks.

## 14. Post-silicon bring-up and characterization

### 14.1 Bring-up order

1. Validate rails and always-on supply first.
2. Validate reset and reference clocks.
3. Validate PMU register access and reset values.
4. Enter the shallowest clock-gated state.
5. Measure current before/after clock gating.
6. Validate one power domain at a time.
7. Validate isolation and wake signaling.
8. Validate retention with known patterns.
9. Validate DVFS at each OPP.
10. Run thermal and voltage sweeps.
11. Test repeated sleep/wake cycles and long-duration retention.
12. Test simultaneous wake sources, watchdog and reset interactions.

### 14.2 Measurement equipment and observability

Useful instruments and data:

- PMIC telemetry and regulator status.
- High-bandwidth oscilloscope and differential probes for rail transients.
- Current analyzer or precision power monitor.
- Thermal chamber and temperature sensors.
- Clock/frequency monitor and PLL lock status.
- Trace timestamps around low-power entry/exit.
- Performance counters, NoC outstanding counters and DMA status.
- Firmware logs for requested/achieved power state and wake reason.

Measure both average energy and transition behavior. A low average current is not enough if the wake path occasionally fails or produces a rail transient that corrupts another domain.

### 14.3 Characterization matrix

| Variable | What to sweep |
|---|---|
| Voltage | Min/nominal/max supported rail |
| Temperature | Cold, room, hot corners |
| Frequency | Every supported OPP and transition |
| Idle duration | Below, near and above break-even |
| Wake source | Timer, GPIO, interrupt, watchdog, external event |
| Repetition | Single, burst and long soak |
| Load | Idle, background DMA, heavy NoC traffic |
| Retention | Pattern set, duration, repeated cycles |
| Power sequencing | Nominal and allowed ramp/settling corners |

## 15. Common design bugs and root causes

1. **Power-off before drain:** outstanding AXI/NoC responses are orphaned.
2. **Isolation too late:** an active receiver sees X/invalid values during collapse.
3. **Isolation removed too early:** invalid outputs escape before the domain is stable.
4. **Wake pulse too short:** AON logic misses an event across a clock boundary.
5. **Retention supply not guaranteed:** retained state silently corrupts.
6. **Clock stopped before handshake completion:** the IP cannot acknowledge quiescence.
7. **Incorrect OPP order:** frequency rises before voltage is sufficient.
8. **Shared resource shut down:** another IP loses a common PLL, SRAM, DDR or regulator.
9. **Software assumes WFI means power-off:** hardware policy or pending interrupt prevents entry.
10. **PMU register CDC race:** software reads stale state or starts a transition before configuration is synchronized.
11. **Wake cause cleared too early:** software loses diagnostic evidence.
12. **Inrush current not managed:** rail droop resets neighboring domains.
13. **Security state mishandled:** keys or sensitive state survive when policy requires zeroization, or are lost when retention is required.
14. **No timeout recovery:** a stuck IP leaves the PMU permanently in QUIESCE.
15. **Thermal/OPP mismatch:** a frequency is selected without checking thermal or voltage availability.

## 16. Interview questions

1. Explain the difference between clock gating and power gating.
2. Why is isolation needed if the power domain is already switched off?
3. What state should be retained, and how do you decide?
4. What is the purpose of UPF? What does it not specify?
5. Why can a power controller not trust a single IP idle bit?
6. How do you handle outstanding AXI transactions before power-off?
7. Why should voltage generally be raised before frequency during an up-transition?
8. What is an OPP?
9. Explain target residency and exit latency.
10. What is the energy break-even time for a deep idle state?
11. How can a wake event be lost across clock domains?
12. What is the purpose of Q-Channel and P-Channel?
13. What happens if power-good arrives before the clock is stable?
14. How would you verify retention across a long sleep interval?
15. How would you debug unexpectedly high deep-sleep current?
16. Why can an active audio stream prevent DSP power gating?
17. How do isolation and level shifting differ?
18. What is inrush current and why does staged switch enable help?
19. How do you verify a PMU FSM cannot deadlock?
20. What should be captured in reset/wakeup cause registers?

## 17. Hands-on exercises

### Exercise A — Power budget

Given Ceff = 12 nF, activity factor = 0.20, V = 0.85 V and f = 600 MHz, calculate first-order dynamic power. Repeat at 0.75 V and 400 MHz. Compare the ratio and state which assumptions make the estimate approximate.

### Exercise B — Break-even idle time

A deep state saves 180 mW versus shallow idle. Entry plus exit energy is 9 µJ. Calculate the break-even duration. Explain how a 25 µs predicted idle interval should influence state selection.

### Exercise C — Power-domain partition

Draw a domain diagram for CPU, DSP, GPU, NoC, DDR, I2C, GPIO and AON wake logic. Identify shared clocks, shared SRAMs, shared regulators and always-on dependencies. Mark which domains can be independently switched off.

### Exercise D — RTL FSM

Implement an FSM with ACTIVE, QUIESCE, DRAIN, SAVE, ISOLATE, CLOCK_OFF, POWER_OFF, POWER_ON, RESTORE and RESUME states. Add timeout, wake-abort and error recovery. Prove that power cannot be removed while outstanding transactions are nonzero.

### Exercise E — UPF review

Write a power-intent review checklist for a switchable DSP domain. Identify primary supply, retention supply, power switch, isolation boundary, retention elements and voltage crossings. Validate with your chosen tool and IEEE 1801 revision.

### Exercise F — Silicon debug

A chip enters deep sleep at the expected time, but current remains 3× above target. Build a hypothesis tree covering clocks, power switches, IO leakage/back-power, retention rail, wake sources, regulator state and firmware. Specify the measurement that would discriminate each hypothesis.

## 18. Integration checklist

Before signoff, confirm:

- [ ] Power-domain membership and supply ownership are documented.
- [ ] AON wake and PMU logic remain operational in every supported low-power state.
- [ ] Shared resources have explicit dependency/ownership rules.
- [ ] Every IP has a defined quiesce/deny/accept contract.
- [ ] DMA, cache and NoC transactions are drained or safely handled.
- [ ] Isolation coverage is complete.
- [ ] Retention state and restore sequence are specified.
- [ ] Level shifters and isolation cells are mapped correctly.
- [ ] Clock and reset sequences are coordinated with power.
- [ ] DVFS transitions include regulator, PLL and clock-switch acknowledgements.
- [ ] PMU registers define access, reset, CDC and timeout behavior.
- [ ] Wake events are latched and cause information is retained.
- [ ] Software has a documented entry/exit sequence.
- [ ] Power-aware simulation, formal and static checks are complete.
- [ ] Physical IR/EM, inrush and thermal signoff are complete.
- [ ] Silicon measurements cover voltage, temperature and repeated transitions.

## 19. Normative requirements vs implementation examples

| Material | How to interpret it |
|---|---|
| IEEE 1801-2024 | Normative power-intent language and semantics for the selected revision |
| Arm Low Power Interface specification | Normative only for IP/integration that claims the relevant interface revision |
| Linux CPUIdle / OPP documentation | Software framework behavior, not a hardware power-switch protocol |
| OpenTitan pwrmgr | Public implementation example, not a universal PMU architecture |
| Power equations in this lesson | First-order engineering models, not signoff power analysis |
| Generic register map and FSM here | Illustrative architecture, not a vendor register specification |

## 20. Authoritative references and further study

### Standards and historical context

1. Robert H. Dennard et al., *Design of Ion-Implanted MOSFETs With Very Small Physical Dimensions*, IEEE JSSC, 1974: https://research.ibm.com/publications/design-of-ion-implanted-mosfets-with-very-small-physical-dimensions
2. Accellera, *IEEE Std 1801-2024 / UPF 4.0 available through IEEE GET*, March 19, 2025: https://vw.eda.org/news/press-releases/414-accellera-announces-ieee-standard-1801-2024-is-available-through-ieee-get-program
3. Semiconductor Engineering, *Unified Power Format*: https://semiengineering.com/knowledge_centers/standards-laws/standards/unified-power-format/

### Open implementations and software contracts

4. OpenTitan Power Manager overview and specification: https://opentitan.org/book/hw/ip/pwrmgr/index.html
5. OpenTitan Power Manager theory of operation: https://opentitan.org/book/hw/ip/pwrmgr/doc/theory_of_operation.html
6. OpenTitan Power Manager programmer guide: https://opentitan.org/book/hw/top_earlgrey/ip_autogen/pwrmgr/doc/programmers_guide.html
7. Linux CPUIdle: https://www.kernel.org/doc/html/v6.6/driver-api/pm/cpuidle.html
8. Linux OPP: https://cdn.kernel.org/doc/html/latest/power/opp.html

### Low-power interface examples

9. Arm CoreLink DMA-350 Q-Channel/P-Channel interfaces: https://support.arm.com/documentation/102482/latest/DMAC-interfaces/LPI-interfaces
10. Arm CoreLink CCI-500 clock and power control: https://support.arm.com/documentation/100023/0100/functional-description/interfaces/clock-and-power-control-interfaces
11. Arm PCK-600 power/clock management integration: https://developer.arm.com/community/arm-community-blogs/b/internet-of-things-blog/posts/new-pck-600-for-efficient-and-rapid-integration-of-soc-power-management-infrastructure

---

## Lesson 16 completion criteria

You should now be able to:

1. Explain the difference between clock gating, DVFS, retention and power gating.
2. Draw a power-domain hierarchy and dependency graph.
3. Explain power-switch, isolation, retention and level-shifter roles.
4. Trace a complete low-power entry/exit handshake.
5. Explain what UPF contributes to the design and signoff flow.
6. Calculate dynamic power, OPP comparisons and break-even idle time.
7. Identify DMA/cache/NoC conditions that must be satisfied before shutdown.
8. Build a power-aware verification and silicon characterization plan.

**Next lesson — Session 17: Interrupt Controllers and Timers.** We will study interrupt routing, prioritization, claim/complete, edge/level semantics, GIC/PLIC concepts, timer architecture, watchdogs, wakeup integration and interrupt-latency debugging.

**Progress after this lesson: 16 of 38 sessions (42.1%).**
