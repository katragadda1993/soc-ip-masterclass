# Lesson 14 — Clock Architecture: From PLLs to Clock Trees and Silicon Timing

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 14 of 38  
**Duration:** 60 minutes  
**Level:** RTL integration → physical implementation → post-silicon characterization  
**Prerequisites:** Lessons 01–13; sequential logic, setup/hold timing, feedback systems and CDC basics.

> **Central engineering question:** How do we generate clocks with the required frequency, phase, duty cycle and spectral quality, distribute them across a large die, stop and restart them safely for power savings, and transfer information between domains without metastability or lost transactions?

This lesson studies the clocking *system*, not only a PLL. A production SoC clock architecture includes reference sources, oscillators, PLLs, dividers, muxes, buffers, clock-tree synthesis (CTS), integrated clock-gating (ICG) cells, clock monitors, reset sequencing, CDC/RDC structures, firmware controls and physical timing constraints.

---

## 1. One-hour learning plan

| Time | Topic | Expected outcome |
|---:|---|---|
| 0–5 min | Clock architecture and terminology | Draw a multi-domain clock plan |
| 5–14 min | Oscillators and PLLs | Explain frequency synthesis and lock |
| 14–22 min | Muxes, dividers and clock gating | Explain safe switching and gating |
| 22–32 min | Clock distribution and CTS | Understand skew, latency, jitter and tree topology |
| 32–40 min | CDC and RDC | Select a safe crossing architecture |
| 40–46 min | Reset and power sequencing | Build a safe clock start/stop sequence |
| 46–53 min | Registers, firmware and integration | Define the clock-manager contract |
| 53–60 min | Verification, silicon debug and exercises | Build a validation plan |

## 2. Why an SoC needs a clock architecture

A clock defines when synchronous state elements sample data and update state. It is therefore part of the functional contract of every synchronous IP.

A modern SoC rarely uses one frequency for every block. Different IPs have different throughput, latency, interface, power and accuracy requirements.

Illustrative clock plan (not a specification):

| Domain | Example frequency | Reason |
|---|---:|---|
| CPU cluster | 1.0 GHz | High instruction throughput |
| NoC/interconnect | 800 MHz | High aggregate transaction bandwidth |
| DSP | 600 MHz | Signal-processing workload |
| DDR controller UI | 500 MHz | Controller-side interface timing |
| Peripheral/APB | 100 MHz | Lower-rate register accesses |
| Audio reference | 24.576 MHz | Audio-family clock generation |
| USB reference | IP-specific | PHY/protocol reference requirements |
| Always-on (AON) | 32.768 kHz | RTC, wakeup and low-power monitoring |

These values are examples only. Actual clocks depend on SoC, IP specification, process, PHY, voltage and board-level reference sources.

### 2.1 Clock-domain vocabulary

- **Clock source:** oscillator, external reference, PLL output or another clock.
- **Clock domain:** sequential logic whose timing is defined by one clock and its constrained relationships.
- **Generated clock:** clock derived through a divider, PLL, mux or other known transformation.
- **Synchronous domains:** clocks with a known, constrained phase/frequency relationship.
- **Asynchronous domains:** no guaranteed phase relationship; crossings require CDC treatment.
- **Clock root:** source feeding a clock-distribution branch.
- **Clock leaf:** endpoint clock pin of a sequential element or macro.
- **Clock latency/insertion delay:** time from the defined reference point to a clock pin.
- **Clock skew:** difference in arrival time at two clock endpoints.
- **Jitter:** variation of clock edge timing relative to an ideal edge or reference.
- **Duty cycle:** fraction of a period for which the clock is high.
- **Clock gating:** suppressing clock pulses to an inactive block.
- **Clock switching:** selecting a different clock source or frequency.

A generated clock is not automatically asynchronous merely because it has a different frequency. If its relationship is deterministic and constrained, STA can analyze it as a related clock. Conversely, two clocks with nominally equal frequency may still be asynchronous if their phase relationship is not guaranteed.

## 3. Architecture: from reference source to sequential endpoint

~~~mermaid
flowchart TD
  XTAL[Crystal / external reference] --> REF[Reference conditioning]
  REF --> PLL[PLL / frequency synthesizer]
  RC[On-chip RC oscillator] --> MUX[Glitch-controlled source mux]
  PLL --> MUX
  MUX --> DIV[Programmable dividers]
  DIV --> ROOT[Clock roots]
  ROOT --> GATE[ICG / transactional clock gates]
  GATE --> CTS[Clock distribution tree / mesh]
  CTS --> CPU[CPU registers]
  CTS --> NOC[NoC registers]
  CTS --> DSP[DSP registers]
  CTS --> PERI[Peripheral registers]
  MON[Clock monitor / frequency counter] -. observes .-> ROOT
  PWR[Power manager] --> GATE
  RST[Reset manager] --> CPU
  RST --> NOC
  RST --> DSP
  RST --> PERI
~~~

The clock manager is usually a control-plane IP. The PLL and oscillator may be analog or mixed-signal macros. CTS is largely a physical-design implementation stage. Reset and power managers coordinate clock availability with the rest of the system.

### 3.1 Functional ownership

| Function | Typical owner | Important contract |
|---|---|---|
| Reference oscillator | Analog/mixed-signal IP | Frequency, tolerance, startup, jitter |
| PLL | Analog/mixed-signal IP | Input/output range, lock, jitter, phase noise |
| Clock mux/divider | Clock-control RTL or hard macro | Safe switch, legal ratios, status |
| Clock gate | ICG cell plus enable control | No runt pulse; test override |
| Clock tree | Physical-design flow | Skew, insertion delay, transition, capacitance |
| Clock monitor | Clock manager / AON logic | Too-fast, too-slow, stopped-clock detection |
| Reset manager | Reset-control IP | Assert/deassert ordering per domain |
| CDC/RDC | RTL architecture and verification | No unsafe crossings or reset-release races |
| Firmware | Boot/runtime power software | Legal reconfiguration sequence |

## 4. Historical backstory and design rationale

Clock architecture evolved because the engineering problem changed from synchronizing a few oscillators to distributing low-jitter clocks across large, power-constrained dies.

### 4.1 PLLs: from receiver synchronization to on-chip frequency synthesis

**Documented history**

- **1932:** Henri de Bellescize described phase-synchronization techniques in the context of direct-conversion (homodyne/synchrodyne) radio receivers. The local oscillator needed to track a received signal despite frequency drift. A feedback loop comparing phase and correcting the oscillator addressed that problem.
- **Late 1930s onward:** synchronization techniques were used in television receiver sweep circuits.
- **Late 1960s / early 1970s:** monolithic PLL ICs made phase/frequency detection, oscillator control and loop filtering practical as integrated building blocks. Signetics' 565 family is a commonly cited early example; exact product introduction dates vary across historical accounts.
- **2000s onward:** PLLs became routine SoC clock-generation resources, alongside dedicated clock buffers, programmable dividers, jitter filtering and multiple clock outputs.
- **Current programmable SoC examples:** AMD Versal documents MMCM, XPLL and DPLL resources with different general-purpose and PHY-oriented roles.

**Design rationale**

The original problem was not “make a CPU faster.” It was “make a local oscillator follow a reference despite drift.” The same feedback principle later became useful for multiplying a stable low-frequency reference into high-frequency internal clocks.

A PLL offers frequency synthesis and phase tracking, but introduces analog design trade-offs: loop bandwidth, reference spur, phase noise, lock time, VCO tuning range, power and area.

### 4.2 Clock distribution: why a clock tree became a physical-design problem

**Documented history**

- **1986:** Eby G. Friedman and Scott Powell published a hierarchical clock-distribution study in IEEE Journal of Solid-State Circuits. It treated clock skew and distributed interconnect impedance as first-order VLSI design concerns.
- **1998:** research on clock-gating methods described the need to prevent glitches on gated clocks and the use of a latch to hold the enable stable around the active clock edge.
- **2001:** an IEEE JSSC paper described buffered trees combined with a global grid for large microprocessor clock networks and reported measured global skew as low as 22 ps on the chips studied.

**Design rationale**

A clock drives many sequential pins and has high switching activity. As the die grows, wire resistance, capacitance, fanout, placement and process variation make a simple RTL-level clock net physically inadequate.

A tree reduces fanout by inserting buffers and balancing path delays. A grid/mesh can improve local robustness and skew at the cost of more routing resources and clock power. Modern flows may use hybrids, regional networks and active deskew.

### 4.3 Clock gating: why it is more than an AND gate

A naïve combinational AND between a clock and enable can create a shortened pulse if enable changes while the clock is high. The result can violate minimum pulse-width requirements or clock internal state unexpectedly.

The established ICG pattern captures the enable while the clock is inactive, then holds it constant during the active phase. Production ASIC flows use characterized library ICG cells, often with scan/test enable. The exact latch polarity and cell behavior are library-specific.

**Engineering interpretation:** as dynamic power became a dominant constraint, clock gating became attractive because it stops internal clock-pin capacitance from toggling when a block is idle. The cost is added control, test, timing and verification complexity.

### 4.4 Modern clocking resources

Modern programmable devices illustrate the evolution clearly. AMD Versal documentation describes MMCM, XPLL and DPLL resources, clock buffers, regional/global routing and calibrated deskew. This is a vendor implementation example, not a universal ASIC architecture.

**Historical evidence and further reading**
- Henri de Bellescize / PLL historical overview: [Phase-locked loop history](https://en.wikipedia.org/wiki/Phase-locked_loop) — a secondary historical pointer, not a normative source.
- Analog Devices, *Phase-Lock Loop Applications Using the MAX9382* (26 June 2002): [PLL PFD and charge-pump explanation](https://www.analog.com/en/resources/technical-articles/phaselock-loop-applications-using-the-max9382.html).
- Friedman and Powell, *Design and Analysis of a Hierarchical Clock Distribution System for Synchronous Standard Cell/Macrocell VLSI*, IEEE JSSC, April 1986: [paper PDF](https://www.hajim.rochester.edu/ece/sites/friedman/papers/JSSC.4_86.pdf).
- *A Clock-Gating Method for Low-Power LSI Design* (ASPDAC 1998): [paper PDF](https://www.cs.york.ac.uk/rts/docs/SIGDA-Compendium-1994-2004/papers/1998/aspdac98/pdffiles/05c_3.pdf).
- *A Clock Distribution Network for Microprocessors*, IEEE JSSC, May 2001: [IEEE record](https://ieeexplore.ieee.org/abstract/document/918917).
- AMD, [Versal Clocking Resources Architecture Manual AM003](https://docs.amd.com/r/en-US/am003-versal-clocking-resources): vendor-specific example of clock-resource partitioning.

## 5. PLL architecture and frequency synthesis

### 5.1 Basic charge-pump PLL

~~~mermaid
flowchart LR
  REF[Reference clock] --> PFD[Phase / frequency detector]
  DIV[Feedback divider] --> PFD
  PFD --> CP[Charge pump]
  CP --> LF[Loop filter]
  LF --> VCO[Voltage-controlled oscillator]
  VCO --> OUT[PLL output]
  VCO --> DIV
~~~

The phase/frequency detector compares reference and feedback edges. It generates UP/DOWN information. The charge pump sources or sinks current into the loop filter. The loop filter converts that current into a control voltage. The VCO changes its output frequency. A feedback divider makes the loop lock at a programmed ratio.

### 5.2 Frequency equation

For a common integer-N PLL model:

[
f_{VCO} = f_{REF} \times \frac{N}{M}
]

[
f_{OUT} = \frac{f_{VCO}}{D}
]

Where:
- (f_{REF}): reference frequency
- (M): reference divider
- (N): feedback multiplication ratio
- (D): output divider

**Worked example (illustrative):**

- Reference: 25 MHz
- M = 1
- N = 40
- D = 1

Then the VCO/output frequency is 1 GHz. A real PLL must also satisfy its legal PFD frequency, VCO tuning range, output divider, loop-stability and jitter constraints. The arithmetic alone does not prove a legal PLL configuration.

### 5.3 PLL building blocks and their failure modes

| Block | Purpose | Common failure / concern |
|---|---|---|
| Reference oscillator | Stable input frequency | Startup failure, wrong frequency, excessive drift |
| PFD | Compare phase/frequency | Dead zone, reference spur, reset-state problem |
| Charge pump | Convert UP/DOWN to current | Current mismatch, leakage, ripple |
| Loop filter | Set loop dynamics | Stability, bandwidth, settling time |
| VCO | Generate high-frequency clock | Tuning range, phase noise, supply sensitivity |
| Feedback divider | Set multiplication ratio | Illegal divider settings, fractional spurs |
| Lock detector | Report lock condition | False lock, early lock indication, deassertion behavior |

### 5.4 Integer-N and fractional-N trade-offs

- **Integer-N:** output frequency is an integer multiple of the PFD/reference ratio. It is conceptually simpler and can have predictable spur behavior, but frequency resolution may be limited by reference frequency and divider constraints.
- **Fractional-N:** average division ratio can be fractional, allowing finer frequency steps. Sigma-delta modulation or related techniques distribute quantization error, but fractional spurs and phase-noise behavior require careful design.
- **Digital PLL (DPLL):** may use a time-to-digital converter, digital loop filter and digitally controlled oscillator. It is not simply an analog PLL with software control; implementation details vary.

Do not assume every PLL has the same register interface, lock algorithm, reset sequence or fractional architecture.

### 5.5 Jitter, phase noise and lock

- **Period jitter:** variation of one clock period from its nominal value.
- **Cycle-to-cycle jitter:** difference between adjacent clock periods.
- **Time-interval error (TIE):** edge displacement relative to an ideal reference.
- **Phase noise:** spectral representation of phase fluctuations.
- **Duty-cycle distortion:** deviation from the intended high/low time ratio.
- **Lock time:** time required to reach specified frequency/phase stability after enable or reconfiguration.

For independent, uncorrelated RMS jitter sources, a first-order estimate is:

[
J_{TOTAL,RMS} \approx \sqrt{J_1^2 + J_2^2 + \cdots + J_n^2}
]

This root-sum-square approximation is not valid for arbitrary correlated or deterministic jitter. A real budget must follow the relevant interface/PHY specification and measurement method.

## 6. Clock muxes, dividers and glitch-free switching

### 6.1 Why a normal data mux is unsafe on a clock

If source A is high and source B is low, changing a combinational select can immediately force the output low. That creates a shortened pulse. If the levels are reversed, it can create an unexpected edge.

Use a characterized glitch-free clock mux or a documented handshake architecture. The implementation must define what happens if:
- one source is stopped;
- one source is slower;
- source frequencies are unrelated;
- select changes during an active cycle;
- a source loses lock;
- power is removed from one source.

### 6.2 Safe clock-switch concept

A typical safe-switch concept (not a universal implementation):

1. Request a switch.
2. Prevent new transactions in the affected domain.
3. Wait for the domain to become idle or enter a defined quiescent state.
4. Ask the clock mux to perform a glitch-controlled source change.
5. Wait for switch acknowledgement.
6. Verify the selected clock is present and in range.
7. Release/continue the domain according to reset and protocol requirements.

Some hard clock muxes do not require software to stop the domain, but the SoC-level transaction and reset contract still must be designed.

### 6.3 Divider architecture

A binary counter can create integer divide-by-2, divide-by-4, etc. Odd divisors and arbitrary duty-cycle requirements need more care. Programmable dividers may be:
- integer;
- fractional;
- pulse-swallow;
- multi-modulus;
- PHY-specific.

A divider output is a generated clock. The timing constraints must describe its ratio and phase relationship to the source clock.

## 7. Clock gating and low-power control

### 7.1 ICG conceptual behavior

~~~text
                    +----------------------+
CLK ----------------| clock input          |
                    |                      |---- GCLK
EN ---- latch ------| enable logic         |
TE ----------------| test enable           |
                    +----------------------+

Concept: EN is captured while CLK is in its inactive phase.
The captured enable is held stable while the clock is active.
TE allows scan/test clock propagation when required.
~~~

The sketch is conceptual. Do not implement a production clock gate as arbitrary combinational RTL. Use the approved ICG cell/inference methodology and library checks.

### 7.2 What can be gated?

- CPU execution cluster when idle;
- DSP/accelerator clock while no work is outstanding;
- peripheral clock when the block is disabled;
- portions of an interconnect when safely quiescent.

Do not gate:
- the only clock needed to wake the block;
- the AON timer/interrupt path required for wakeup;
- a clock while an uncompleted transaction depends on that domain;
- a clock needed by a reset/handshake controller before the handshake completes.

### 7.3 Software gate versus transactional gate

- **Software gate:** firmware requests enable/disable; software must obey the documented idle sequence.
- **Transactional gate:** hardware knows whether the unit is idle and only honors a disable hint when safe.
- **Root/power gate:** power manager may override local enables during a controlled power-state transition.

OpenTitan's public clock-manager documentation is a useful implementation example: software hints and hardware idle state are combined for transactional clock gating. It also documents frequency measurement and stopped-clock timeout detection.

## 8. Clock-tree synthesis (CTS) and physical distribution

RTL describes logical clock relationships; physical implementation must deliver the clock to real sequential pins with acceptable skew, transition, capacitance, insertion delay and power.

### 8.1 Common distribution topologies

| Topology | Advantage | Trade-off |
|---|---|---|
| Balanced tree / H-tree | Predictable path balancing | Sensitive to placement and routing asymmetry |
| Buffered tree | Flexible for irregular floorplans | Buffer count, insertion delay and power |
| Clock grid/mesh | Strong local connectivity and skew robustness | Large metal/capacitance and power |
| Hybrid regional tree + mesh | Combines global reach and local robustness | More complex planning and analysis |
| Source-synchronous local clock | Useful near PHY/interface | Requires interface-specific timing analysis |

### 8.2 CTS flow

~~~mermaid
flowchart TD
  S[Clock definitions and constraints] --> P[Floorplan and clock sink placement]
  P --> C[CTS topology and buffer insertion]
  C --> B[Balance skew / insertion delay]
  B --> R[Route clock network]
  R --> STA[Post-CTS setup and hold STA]
  STA -->|violations| FIX[Resize / buffer / useful skew / data-path repair]
  FIX --> R
  STA -->|clean| SIGNOFF[Clock signoff: skew, transition, pulse width, power]
~~~

### 8.3 Timing terminology

- **Insertion delay:** clock arrival time from source/reference point to a sequential pin.
- **Local skew:** difference in launch and capture clock arrival times for a timing path.
- **Global skew:** maximum/minimum arrival spread across a defined clock sink set.
- **Useful skew:** intentionally shaping capture/launch arrival times to improve a critical path, while checking all affected setup and hold paths.
- **Clock uncertainty:** timing margin for jitter, modeling error and other uncertainty defined by the methodology.
- **Transition/slew:** edge rise/fall time; excessive slew can violate cell constraints and increase timing uncertainty.

### 8.4 Setup and hold equations

Let skew be:

[
t_{skew} = t_{capture\_clock} - t_{launch\_clock}
]

Positive skew means the capture clock arrives later.

A simplified setup condition:

[
t_{CQ,max} + t_{DATA,max} + t_{SETUP}
\leq T_{CLK} + t_{skew} - t_{uncertainty}
]

A simplified hold condition:

[
t_{CQ,min} + t_{DATA,min}
\geq t_{skew} + t_{HOLD}
]

Positive capture skew can help setup but hurt hold. This is why “more positive skew” is not universally good.

### 8.5 Worked timing budget

Assume:
- clock period = 1,000 ps (1 GHz);
- clock-to-Q max = 70 ps;
- data-path max = 650 ps;
- setup time = 50 ps;
- uncertainty = 40 ps;
- capture-minus-launch skew = +30 ps.

Setup slack:

[
1000 + 30 - 70 - 650 - 50 - 40 = 220\ ps
]

Now consider a short hold path:
- clock-to-Q min = 35 ps;
- data-path min = 20 ps;
- hold time = 30 ps;
- positive skew = 30 ps.

Hold slack:

[
35 + 20 - 30 - 30 = -5\ ps
]

The path has a 5 ps hold violation. A design can therefore pass setup and still fail hold after CTS.

## 9. Clock-domain crossing (CDC)

### 9.1 Why metastability exists

If a destination flip-flop samples a changing asynchronous signal near its setup/hold aperture, its internal state can become metastable. The output may take an unpredictable amount of time to resolve. A synchronizer reduces the probability that metastability propagates; it does not mathematically eliminate it.

### 9.2 Choose the crossing by information type

| Crossing type | Common technique | Main constraint |
|---|---|---|
| Single-bit level | 2-FF synchronizer | Level must remain stable long enough |
| Short pulse | Toggle synchronizer or request/ack handshake | Must not miss events |
| Multi-bit control word | Stable-data handshake | Data held stable until captured |
| High-throughput stream | Asynchronous FIFO | Correct pointer/flag CDC |
| Counter/status | Gray-coded counter or handshake | Only one Gray bit changes per increment |
| Source-synchronous bus | Source-synchronous capture/forwarding | Interface timing and skew budget |

### 9.3 Two-flop synchronizer

~~~mermaid
flowchart LR
  ASYNC[Async input] --> FF1[Destination FF1]
  DCLK[Destination clock] -. clocks .-> FF1
  FF1 --> FF2[Destination FF2]
  DCLK -. clocks .-> FF2
  FF2 --> LOGIC[Destination logic]
~~~

Use a recognized synchronizer structure and preserve the intended physical separation/placement constraints. Do not place combinational logic between synchronizer stages. Mark synchronizer flops using the flow's supported attributes.

### 9.4 Multi-bit crossings

Do not independently synchronize each bit of a changing binary bus and assume the received word is coherent. Different bits can settle on different cycles, producing a mixed value.

For high-throughput data, use an asynchronous FIFO:
- dual-clock storage;
- write pointer in source domain;
- read pointer in destination domain;
- Gray-coded pointer synchronization;
- full/empty logic;
- carefully reviewed reset behavior.

### 9.5 Gray-code example

Binary sequence: 000, 001, 010, 011, 100  
Gray sequence: 000, 001, 011, 010, 110

Only one Gray bit changes per adjacent increment. This reduces the risk of a sampled pointer appearing to jump to an unrelated value. It does not remove the need for synchronizers, correct constraints and FIFO verification.

### 9.6 CDC verification checklist

- Are all asynchronous inputs synchronized or handled by a proven CDC primitive?
- Are pulse widths guaranteed?
- Are multi-bit controls transferred atomically?
- Are FIFO pointers Gray-coded and synchronized?
- Are synchronizer attributes and placement constraints present?
- Are clock relationships correctly declared in STA?
- Are asynchronous clock groups cut only where appropriate?
- Is reset release safe for each receiving domain?
- Does the CDC tool waiver have a documented architectural reason?

## 10. Reset-domain crossing and clock/reset sequencing

Clock and reset design are inseparable. A block cannot be safely released from reset until its clock is present and stable according to the block's contract.

### 10.1 Asynchronous assertion, synchronous deassertion

A common reset synchronizer pattern asserts reset asynchronously and releases it synchronously to the destination clock. The exact implementation must be reviewed for the target library and reset polarity.

~~~systemverilog
module reset_sync (
  input  logic clk,
  input  logic arst_n,
  output logic srst_n
);
  (* async_reg = "true" *) logic [1:0] sync_ff;

  always_ff @(posedge clk or negedge arst_n) begin
    if (!arst_n)
      sync_ff <= 2'b00;
    else
      sync_ff <= {sync_ff[0], 1'b1};
  end

  assign srst_n = sync_ff[1];
endmodule
~~~

This is an educational pattern, not a substitute for the project's approved reset macro, reset-tree methodology or RDC signoff.

### 10.2 PLL lock and reset release

A typical sequence:
1. Power and reference source become valid.
2. PLL is enabled.
3. PLL lock detector asserts after its documented qualification.
4. Lock status is synchronized into the relevant control domain.
5. Derived clocks are enabled.
6. Clock presence/frequency checks pass if required.
7. Reset is deasserted synchronously in each destination domain.
8. IP initialization completes.
9. Interconnect transactions are permitted.

Do not use an asynchronous PLL LOCKED signal directly as a synchronous reset release without synchronization and qualification. Vendor guidance explicitly recommends synchronizing lock status before using it in synchronous logic.

### 10.3 Reset-domain crossing (RDC)

RDC bugs arise when logic in different reset domains observes signals whose reset assertion/deassertion ordering is not coordinated. A block can leave reset while its producer remains reset, or one side can interpret stale handshake state as a new transaction.

Review:
- reset assertion and deassertion ordering;
- independent resets inside one clock domain;
- reset synchronization for every destination clock;
- FIFO pointer reset alignment;
- valid/ready state reset consistency;
- reset while a transaction is outstanding;
- reset during clock switch or power transition.

## 11. Clock manager registers and programming sequence

There is no universal clock-manager register map. Offsets, lock status semantics, divider encodings, write-enable protection and clock-gate handshakes are implementation-specific.

### 11.1 Illustrative register map

| Register | Example fields | Purpose |
|---|---|---|
| CLK_ROOT_SEL | source select | Select oscillator/PLL source |
| PLL_CTRL | enable, bypass, divider config | Configure and enable PLL |
| PLL_STATUS | lock, fault | Report PLL state |
| CLK_DIV_CFG | divider ratios | Configure derived frequencies |
| CLK_GATE_EN | per-IP enable | Request software-gated clocks |
| CLK_GATE_STATUS | active/idle/ack | Confirm gate state |
| CLK_HINTS | transactional disable hints | Allow hardware to gate idle units |
| CLK_MEAS_CTRL | enable, min/max window | Configure frequency measurement |
| CLK_MEAS_COUNT | measured cycles | Return frequency measurement |
| CLK_MON_STATUS | too-fast/slow/timeout | Report monitor faults |
| REGWEN | write enable | Lock critical configuration |

**Public implementation example — OpenTitan:** its clock-manager register documentation includes external clock control, software clock enables, transactional clock hints, frequency measurement controls and recoverable/fatal error status. Use the linked documentation for exact offsets and field semantics; do not transplant those offsets into another SoC.

### 11.2 Safe firmware programming sequence

1. Read current source, divider, lock and fault status.
2. Confirm the target clock configuration is legal for the IP and voltage state.
3. Quiesce transactions in the affected domain.
4. Switch to a safe source if required by the PLL programming procedure.
5. Program PLL configuration while disabled or in the documented programming mode.
6. Enable PLL and wait with a bounded timeout for qualified lock.
7. Check clock monitor/frequency status.
8. Program dividers and safe mux selection.
9. Request the domain clock enable.
10. Wait for clock-active acknowledgement.
11. Release reset in the required sequence.
12. Resume traffic and verify functional progress.
13. Record configuration, measured frequency and any timeout/error code.

Do not assume firmware may reprogram a live PLL while logic is consuming its output. Some devices support dynamic reconfiguration; others require a safe clock switch and quiescence.

## 12. Power management and DVFS interaction

Clock frequency is one component of dynamic power:

[
P_{dynamic} \approx \alpha C V^2 f
]

Where:
- alpha is switching activity;
- C is effective switched capacitance;
- V is supply voltage;
- f is clock frequency.

For a clock network, activity is high because the clock toggles regularly. Gating an unused branch can reduce the capacitance that switches. Reducing frequency lowers dynamic power approximately linearly if other factors stay constant. Reducing voltage has a quadratic effect in this simple model.

### 12.1 DVFS sequencing

**Increasing performance:**
1. Request a higher voltage operating point.
2. Wait for regulator/PMIC settling and power-good.
3. Configure/lock the higher frequency.
4. Switch to the new clock safely.
5. Update timing-sensitive software assumptions and performance state.

**Reducing performance:**
1. Quiesce or coordinate clients.
2. Switch to a lower frequency safely.
3. Confirm the lower frequency.
4. Lower voltage to the matching operating point.

The exact order and handshake are platform-specific. A DVFS controller must respect voltage/frequency operating-point tables, PLL constraints, thermal limits and power-good signals.

### 12.2 Low-power entry and wakeup

- Stop new transactions.
- Wait for IP idle/flush acknowledgements.
- Save state or enter retention as required.
- Gate clocks.
- Apply isolation/power-off in the documented order.
- Keep AON wakeup/RTC paths active.
- On wakeup, restore power, clocks and reset in the required order.
- Reinitialize non-retained clock-manager registers if necessary.

## 13. Clock monitoring and observability

A clock manager may measure a clock against an independent always-on reference:
- count target-clock cycles during a fixed AON interval;
- compare count with programmed minimum and maximum;
- detect too-fast or too-slow behavior;
- use a timeout handshake to detect a stopped clock.

Measurement resolution depends on the reference frequency and measurement window. If a 24 MHz clock is measured against a 200 kHz AON clock over one AON cycle, the simple count quantization is coarse; a longer measurement window improves resolution at the cost of response time.

OpenTitan documents clock-frequency measurement, too-fast/too-slow detection and timeout detection. Its public verification plan includes randomized frequency-window checks and stopped-clock tests.

### Silicon observability signals

Useful observability:
- PLL lock and unlock counters;
- selected clock-source status;
- clock divider readback;
- clock gate request/ack/status;
- frequency measurement count;
- missing-clock timeout;
- reset cause and reset-domain status;
- power-state transition state;
- debug clock output (only if safely designed and allowed).

A clock output pin or divided debug clock is useful, but it may not represent the exact clock at an internal endpoint because downstream gating, muxing and physical skew can differ.

## 14. Common clocking bugs and failure signatures

| Symptom | Likely cause | First checks |
|---|---|---|
| PLL never locks | Wrong reference, illegal divider, power/reset issue | Reference frequency, PLL enable, lock timeout, analog supply |
| PLL locks but CPU crashes | Wrong output ratio, jitter, reset released early | Divider readback, frequency monitor, reset sequence |
| Intermittent failures at high frequency | Setup/hold margin, jitter, skew, voltage/temperature | STA corners, jitter budget, shmoo, critical paths |
| Random extra counter increment | Glitch from clock mux/gate | Mux cell, ICG implementation, enable timing |
| Peripheral stops after idle | Clock gate not reopened or missing handshake | Gate request/ack, transactional idle status |
| Wakeup timeout | AON clock gated or reset/clock dependency cycle | AON path, wakeup FSM, root clock status |
| CDC-only failures | Unsafe single-bit or multi-bit crossing | CDC report, pulse width, FIFO pointer logic |
| Failure after warm reset | RDC or stale handshake state | Reset-domain matrix, FIFO/valid-ready reset |
| Audio/video underrun | Clock ratio or jitter/clock-switch disturbance | BCLK/MCLK, stream clock, FIFO level, source switch |
| Frequency monitor false alarm | Incorrect threshold or too-short measurement window | AON reference, tolerance, window size |
| Scan fails after clock changes | Test-enable or test-clock mux path | DFT clock mode, ICG test pin, scan constraints |

## 15. Pre-silicon verification plan

### 15.1 RTL simulation

- PLL model covers lock delay, unlock, missing reference and frequency change.
- Clock mux tests switch between unrelated clocks and stopped clocks.
- Divider tests verify ratios and duty cycle where specified.
- ICG tests toggle enable near both clock phases and check no runt pulse in the supported model.
- Clock gate re-enable is tested while a request is pending.
- Reset is asserted/deasserted at different phases.
- Clock is stopped while the IP is idle and while transactions are active.
- Clock-monitor thresholds are tested at min, max, below, above and timeout conditions.

Digital simulation cannot accurately prove analog PLL phase noise or physical jitter. Those require analog/mixed-signal simulation, extracted analysis and silicon measurement.

### 15.2 Assertions and properties

Illustrative properties to implement in the real verification environment:

~~~systemverilog
// Conceptual properties: bind to the actual clock-manager interface.
// 1. A clock-gate disable acknowledgement must imply the unit is idle.
// 2. A PLL-derived domain reset must remain asserted until qualified lock.
// 3. A clock switch request must eventually acknowledge or report timeout.
// 4. A gated clock must not produce pulses shorter than the library/IP minimum.
// 5. Frequency measurement must flag values outside programmed bounds.
// 6. A stopped monitored clock must eventually raise timeout status.
~~~

Avoid writing a naïve SVA that samples a clock which is intentionally stopped. Use an independent reference/AON clock for timeout and liveness checks.

### 15.3 Static and formal signoff

- STA: primary/generated clocks, clock groups, uncertainty, source/network latency, input/output delays.
- CDC: synchronizers, handshakes, asynchronous FIFOs, pulse crossings.
- RDC: asynchronous reset assertion and synchronous release, reset ordering, reset convergence.
- Lint: accidental data-as-clock, combinational clock logic, inferred latches outside approved structures.
- Formal: clock-gate protocol, reset sequencing, no transaction accepted while domain unavailable, request/ack liveness under assumptions.
- DFT: scan clock paths, test enable, at-speed capture clocks, OCC (on-chip clock controller) requirements.
- UPF/power intent: clock and reset behavior across power states.

### 15.4 Example timing constraints (methodology sketch)

~~~tcl
# Illustrative SDC only. Real values and syntax depend on STA tool and methodology.
create_clock -name ref_clk -period 40.000 [get_ports ref_clk]

# Example generated clock: divide-by-2 from a PLL output.
# The source pin and target pin must match the actual netlist.
create_generated_clock -name clk_div2 -source [get_pins u_pll/clk_out] \
  -divide_by 2 [get_pins u_div2/clk_out]

# Declare truly asynchronous clock groups only after reviewing all crossings.
set_clock_groups -asynchronous \
  -group [get_clocks clk_cpu] \
  -group [get_clocks clk_aon]
~~~

The example is not a complete signoff constraint file. Do not cut paths between related generated clocks. Add uncertainty, source/network latency, input/output delays, false paths and multicycle paths only when justified by the interface and design contract.

## 16. Post-silicon bring-up and characterization

### 16.1 First-clock bring-up

1. Verify board reference oscillator frequency and amplitude using the approved probing method.
2. Check power rails, sequencing and reset pins.
3. Read oscillator/PLL status through a safe boot or debug path.
4. Confirm root clock source and divider configuration.
5. Use clock monitor counters to check frequency.
6. Release one low-risk domain at a time.
7. Verify AON timer and UART first, then interconnect and CPU.
8. Enable high-performance domains after their clocks and resets are validated.
9. Run stress tests at voltage/temperature corners.

### 16.2 Characterization matrix

| Variable | Sweep | Observe |
|---|---|---|
| Frequency | Min → nominal → max | Functional pass, timing errors |
| Voltage | Allowed operating range | Lock, jitter, path margin |
| Temperature | Cold / room / hot | Lock time, drift, stability |
| Reference | Tolerance extremes | Output accuracy and jitter |
| PLL setting | Legal divider combinations | Lock and spur behavior |
| Clock gate | Repeated idle/wakeup | Missing pulse, wake latency |
| Source switch | Every supported transition | Glitch-free behavior, lock |
| Reset | POR/warm/watchdog/partial | Correct sequencing and recovery |

### 16.3 Lab equipment and measurement caution

- Frequency counter: frequency accuracy and drift.
- Oscilloscope with suitable bandwidth/probes: edge shape, duty cycle and gross jitter.
- Phase-noise analyzer: phase-noise spectrum and integrated jitter.
- Time-interval/jitter analyzer: time-domain jitter measurements.
- On-chip monitor: functional frequency window and stopped-clock detection.
- Thermal chamber and programmable supply: characterization over corners.

Probe loading, ground inductance, bandwidth and trigger/reference quality can distort measurements. A clock measurement at a package pin does not automatically equal the timing seen at an internal register.

## 17. Worked SoC case study: CPU + NoC + DSP + audio

Assume:
- CPU root: 1 GHz;
- NoC root: 800 MHz;
- DSP root: 600 MHz;
- APB root: 100 MHz;
- audio reference: 24.576 MHz;
- AON: 32.768 kHz.

A runtime request lowers DSP performance.

**Safe sequence:**
1. DSP driver stops accepting new jobs.
2. Firmware waits for DMA and DSP command queues to drain.
3. DSP asserts idle acknowledgement.
4. Clock manager switches DSP to a safe source if required.
5. Divider/PLL configuration is changed according to the platform's legal sequence.
6. New frequency is checked with the monitor.
7. DSP reset is released or retained according to the clock-change protocol.
8. Driver resumes submissions and checks a known workload.

**Failure injection:**
- Stop the reference before lock.
- Drop PLL lock during a long DMA.
- Request gating while a stream is active.
- Assert warm reset during a clock switch.
- Change frequency while a CDC FIFO is almost full.

Expected outcome: the architecture must either complete safely or report a bounded, diagnosable error. It must not silently lose a transaction.

## 18. Interview questions

1. What is the difference between frequency accuracy, jitter and skew?
2. Explain the PFD, charge pump, loop filter and VCO in a charge-pump PLL.
3. Why is PLL lock not necessarily proof of a clean clock?
4. What is the difference between integer-N and fractional-N synthesis?
5. Why can a combinational clock mux create a runt pulse?
6. How does an ICG prevent glitches on a gated clock?
7. What is the difference between clock latency and skew?
8. Why can positive capture skew improve setup but worsen hold?
9. What is the difference between a generated clock and an asynchronous clock?
10. When is a 2-FF synchronizer appropriate, and when is it not?
11. Why is Gray coding used for asynchronous FIFO pointers?
12. Why should reset deassertion be synchronized separately in each domain?
13. How should firmware coordinate DVFS with PLL lock and voltage changes?
14. How can an AON clock monitor detect a stopped clock?
15. Why can a clock pass RTL simulation but fail at voltage/temperature corners?

## 19. Hands-on exercises

### Exercise A — PLL frequency plan
Given a 24 MHz reference, calculate possible output frequencies for M=1, N=40, D=1 and M=1, N=50, D=2. List the additional analog constraints needed before accepting either configuration.

### Exercise B — Setup/hold
Use the timing example in Section 8. Calculate setup slack if skew becomes -20 ps. Calculate hold slack if skew becomes +45 ps. Explain why balancing skew is not the only CTS objective.

### Exercise C — Clock-gate verification
Write a testbench that requests gate-off at every possible phase of the source clock. Check that the approved ICG model never produces a pulse shorter than the specified minimum and that test enable overrides functional disable.

### Exercise D — CDC selection
Choose a crossing method for:
- one sticky interrupt;
- a 5 ns pulse into a 100 MHz domain;
- a 32-bit configuration word updated rarely;
- a 128-bit streaming datapath at 400 MB/s;
- a monotonically increasing event counter.

Justify each selection.

### Exercise E — Reset/clock sequencing
Create a reset-domain matrix for CPU, NoC, DSP, APB, DDR controller and AON. Define which clocks must be present before each reset can deassert.

### Exercise F — Silicon debug
A DSP works at 400 MHz but fails intermittently at 600 MHz only at low voltage and high temperature. Build a debug plan separating:
- PLL lock and jitter;
- CTS/skew and setup/hold;
- voltage droop;
- CDC/RDC;
- IP-specific timing assumptions.

## 20. Reference implementations and authoritative documentation

### Vendor architecture and methodology

1. **AMD Versal Clocking Resources Architecture Manual (AM003):** [Clocking resources, MMCM, XPLL, DPLL and clock routing](https://docs.amd.com/r/en-US/am003-versal-clocking-resources).
2. **AMD UG1387:** [Versal device clocking methodology](https://docs.amd.com/r/2025.1-English/ug1387-acap-hardware-ip-platform-dev-methodology/Clock-Primitives).
3. **AMD clock-modifying blocks:** [MMCM, XPLL and DPLL usage](https://docs.amd.com/r/en-US/2026.1/ug1387-acap-hardware-ip-platform-dev-methodology/Using-Clock-Modifying-Blocks-MMCM-XPLL-and-DPLL).

### Open implementation and verification example

4. **OpenTitan Clock Manager:** [IP specification and architecture](https://opentitan.org/book/hw/ip/clkmgr/index.html).
5. **OpenTitan clock-manager theory:** [Root clocks, reset domains, gating and frequency measurement](https://opentitan.org/book/hw/ip/clkmgr/doc/theory_of_operation.html).
6. **OpenTitan clock-manager registers:** [Documented register map and field semantics](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/clkmgr/doc/registers.html).
7. **OpenTitan clock-manager verification:** [DV plan and frequency/timeout tests](https://opentitan.org/book/hw/ip/clkmgr/dv/index.html).
8. **OpenTitan reset manager:** [Reset causes and reset-domain behavior](https://opentitan.org/book/hw/top_earlgrey/ip_autogen/rstmgr/doc/theory_of_operation.html).

### Research and design background

9. **Friedman & Powell (1986):** [Hierarchical clock distribution for VLSI](https://www.hajim.rochester.edu/ece/sites/friedman/papers/JSSC.4_86.pdf).
10. **ASPDAC (1998):** [Clock-gating method for low-power LSI design](https://www.cs.york.ac.uk/rts/docs/SIGDA-Compendium-1994-2004/papers/1998/aspdac98/pdffiles/05c_3.pdf).
11. **IEEE JSSC (2001):** [A clock distribution network for microprocessors](https://ieeexplore.ieee.org/abstract/document/918917).
12. **Analog Devices (2002):** [PLL phase/frequency detector, charge pump and jitter](https://www.analog.com/en/resources/technical-articles/phaselock-loop-applications-using-the-max9382.html).
13. **Cummings (SNUG 2008):** [CDC design and verification techniques using SystemVerilog](https://www.sunburst-design.com/papers/CummingsSNUG2008Boston_CDC.pdf).

### Source discipline

- **Normative constraints:** the selected foundry/library, PLL/PHY datasheet, IP specification, clock/reset specification and signoff methodology define actual limits.
- **Vendor examples:** AMD and OpenTitan documents show concrete architectures, but their register maps and clock resources are not universal.
- **Research history:** cited papers document specific approaches and measured results for the studied designs; do not generalize those numbers to every SoC.
- **Engineering interpretation:** recommendations in this lesson are design guidance and must be checked against the actual technology, library and IP contracts.

---

## Lesson 14 completion checklist

- [ ] Draw a complete clock-source-to-endpoint diagram.
- [ ] Calculate PLL ratios and output frequencies.
- [ ] Explain lock, jitter, phase noise and duty-cycle distortion.
- [ ] Explain glitch-free muxing and ICG behavior.
- [ ] Calculate setup and hold impact from skew.
- [ ] Select safe CDC structures for level, pulse, bus and stream crossings.
- [ ] Define reset release relative to clock validity.
- [ ] Define clock-manager firmware programming and timeout behavior.
- [ ] Build a pre-silicon CDC/RDC/STA/DFT verification plan.
- [ ] Build a post-silicon clock characterization matrix.

**Next lesson — Session 15: Reset Architecture.** We will study POR, warm reset, watchdog reset, reset trees, reset-domain crossing, sequencing, isolation, retention and recovery.
