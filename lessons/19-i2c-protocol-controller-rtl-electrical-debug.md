# Lesson 19 — I²C Bus: Protocol, Controller RTL, Electrical Design and Silicon Debug

## From two shared wires to a robust multi-controller peripheral fabric

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 19 of 38  
**Target duration:** 60 minutes  
**Level:** protocol, RTL microarchitecture, electrical integration, software and post-silicon validation  
**Prerequisites:** GPIO/pinctrl and pad-ring behavior (Lesson 18), APB/AXI-Lite register interfaces, clock/reset/CDC and interrupt basics

> **Scope discipline:** This lesson teaches the bidirectional, open-drain I²C modes described by NXP UM10204 Rev. 7.0. Ultra Fast-mode is called out separately because it is unidirectional and push-pull; it is not electrically interchangeable with the normal bidirectional bus. Register names and internal FIFOs shown in the implementation study are OpenTitan-specific, not a universal I²C register map.

---

## 0. One-hour learning plan

| Time | Topic | Engineering outcome |
|---:|---|---|
| 0–5 min | Why I²C exists; history and trade-offs | Explain the two-wire design rationale |
| 5–13 min | Electrical layer and pad behavior | Calculate pull-ups and understand wired-AND |
| 13–25 min | START/STOP, bytes, ACK/NACK and addressing | Decode a transaction from a waveform |
| 25–33 min | Repeated START, arbitration and clock stretching | Understand multi-controller corner cases |
| 33–40 min | Modes, timing and throughput | Check timing and estimate transfer latency |
| 40–48 min | Controller/target RTL and register programming | Build a controller block diagram and FSM |
| 48–55 min | Verification, integration and silicon debug | Construct a focused validation plan |
| 55–60 min | Labs and interview review | Turn the lesson into implementation work |

## 1. Why I²C exists — historical problem and design rationale

### 1.1 The original system problem

The official NXP specification explains the original motivation: many consumer, telecom and industrial systems already contained a microcontroller plus common peripheral ICs such as LCD/LED drivers, remote I/O expanders, RAM, EEPROM, real-time clocks, ADCs/DACs, tuning and signal-processing circuits. Giving each device a dedicated parallel interface consumed pins, package resources and board routing. Philips Semiconductors developed a simple shared two-wire bus for efficient inter-IC control.

The original I²C release is dated **1982** in the revision history of NXP UM10204. The first numbered specification was v1.0 in **1992**. The v2.0 revision followed in **1998**, when the specification described I²C as a de facto standard used in more than 1,000 ICs and licensed to more than 50 companies. Fast-mode Plus was introduced in **June 2007** to support longer and/or faster buses by increasing output drive while preserving downward compatibility with Standard-mode and Fast-mode. UM10204 Rev. 7.0, dated **1 October 2021**, updated terminology from master/slave to controller/target and added an I3C discussion.

These dates and design motivations are documented in the NXP specification. They do not reveal undocumented internal company deliberations, and this lesson does not infer any.

### 1.2 What existed before and what I²C traded

| Alternative | Strength | Cost compared with I²C |
|---|---|---|
| Parallel GPIO/register bus | High transfer parallelism | Many pins and routing tracks |
| Dedicated point-to-point control wires | Simple endpoint logic | Pin count grows with peripheral count |
| SPI | Higher throughput, full duplex | Usually needs separate chip-select per target; no native target addressing/arbitration |
| UART | Few wires, simple asynchronous links | Usually point-to-point; no shared synchronous multi-drop arbitration |
| I²C | Two shared wires, addresses, ACK, multi-controller arbitration | Open-drain rise time, bus capacitance, half-duplex traffic and lower throughput |

**Engineering interpretation:** I²C deliberately optimizes system wiring, interoperability and control-plane simplicity rather than bulk data throughput. It is a natural fit for codec configuration, PMICs, sensors, EEPROMs, RTCs, board management and low-rate control.

### 1.3 Milestones and ecosystem

| Date | Documented milestone |
|---|---|
| 1982 | Original I²C release |
| 1992 | Numbered v1.0 specification |
| 1995 | SMBus developed by Intel and others; related to I²C but adds system-management conventions and constraints |
| 1998 | I²C v2.0; higher-speed and lower-voltage requirements addressed, including Hs-mode |
| 19 June 2007 | v3.0 / Fast-mode Plus: up to 1 Mbit/s and stronger drive |
| 13 February 2012 | Ultra Fast-mode documented as a unidirectional push-pull mode up to 5 Mbit/s |
| 4 April 2014 | v6.0 |
| 2017 | MIPI I3C introduced; intended to improve speed and power while retaining a defined I²C coexistence path |
| 1 October 2021 | UM10204 Rev. 7.0 |

The I²C bus also became a foundation for related system-management ecosystems such as SMBus, PMBus, IPMI and display DDC. These are not all identical protocols: electrical limits, timeouts, packet conventions and feature subsets differ.

**Design trade-off:** a two-wire shared bus is inexpensive and easy to route, but every attached pin, trace, connector, level shifter and probe contributes capacitance. That capacitance controls rise time and therefore limits bus speed.

## 2. Where I²C sits in an SoC

I²C is normally a low-bandwidth peripheral-control IP. A CPU or firmware configures the controller through an internal register bus; the external SDA/SCL pins connect through pinmux and I/O cells to board-level targets.

~~~mermaid
flowchart LR
  CPU[CPU / firmware] --> NOC[AXI or NoC]
  NOC --> APB[APB bridge / register bus]
  APB --> I2C[I2C controller IP]
  I2C --> PAD[Pinmux and open-drain pad cells]
  PAD --> BUS[SDA / SCL board bus]
  BUS --> CODEC[Audio codec]
  BUS --> PMIC[PMIC]
  BUS --> EEPROM[EEPROM]
  BUS --> SENSOR[Sensor / RTC]
  I2C --> IRQ[Interrupt controller]
  IRQ --> CPU
~~~

A typical SoC I²C subsystem contains:

- APB or AXI-Lite register interface.
- Controller transmit/receive FIFO or command formatter.
- Target address matcher and target-side FIFOs, if target mode is implemented.
- START/STOP detector.
- SCL timing generator.
- SDA shift register and ACK/NACK logic.
- Arbitration and clock-synchronization logic.
- Clock-stretch detector and timeout logic.
- Input synchronizers and glitch filters.
- Interrupt status, enable and clear registers.
- Pinmux connections to SDA/SCL pads.

The protocol does not mandate APB, FIFO depth, register offsets, DMA, interrupt topology or a particular FSM encoding. Those are implementation choices.

## 3. Electrical layer — the most important physical concept

### 3.1 Open-drain / open-collector signaling

Normal bidirectional I²C uses open-drain-style outputs. A device can actively pull a line LOW or release it. An external pull-up resistor returns the shared line HIGH when every device releases it.

~~~text
             VIO
              |
             Rp
              |
SDA bus ------+------ device A open-drain transistor
              |       (pulls LOW or releases)
              +------ device B open-drain transistor
              |       (pulls LOW or releases)
              +------ device C open-drain transistor
~~~

The bus behaves like a wired-AND in positive logic:

- Any device pulling LOW makes the observed bus LOW.
- The bus is HIGH only when all devices release it.
- A controller must not drive a push-pull HIGH against another device pulling LOW.
- In RTL, represent the output as a pull-low enable or output-enable control; do not model a normal I²C pin as a push-pull data output.

A safe conceptual pad interface is:

~~~systemverilog
assign sda_pad_oe_n = !sda_pull_low; // illustrative active-low OE convention
assign sda_pad_out  = 1'b0;           // output data is always LOW when enabled
assign sda_in       = sda_pad_in;
~~~

Actual pad-cell polarity and names are found in the SoC I/O library documentation. The key requirement is that the controller can release the line to HIGH impedance.

### 3.2 SDA and SCL behavior

| Signal | Purpose | Normal behavior |
|---|---|---|
| SDA | Serial data and ACK/NACK | Bidirectional, open-drain |
| SCL | Serial clock | Normally generated by controller; may be held LOW by a stretching target or another controller |

The controller generates clock pulses, including the ninth ACK/NACK clock. A target may optionally stretch SCL by holding it LOW. Not every target implements clock stretching.

**Data validity rule:** SDA must remain stable while SCL is HIGH, except when generating START or STOP. Ordinary data transitions occur while SCL is LOW.

### 3.3 Voltage domains and power-off behavior

Before connecting devices, check:

- All devices' SDA/SCL VIH and VIL thresholds.
- Pull-up rail voltage and every pin's absolute maximum rating.
- Whether an unpowered target's pins are fail-safe or can clamp the bus through protection structures.
- Whether a level translator supports bidirectional open-drain operation.
- Whether the I/O bank is powered in the intended sleep state.
- Whether pad isolation or pinmux reset defaults accidentally pull a line LOW.

Do not assume that a 3.3 V pull-up is safe for a 1.8 V-only pad. Do not assume a powered-off target will release its pins; confirm the datasheet's I/O-off behavior.

## 4. Pull-up sizing and RC rise-time calculation

The pull-up resistor has two competing limits:

- It must be small enough to charge the bus capacitance within the allowed rise time.
- It must be large enough that a device can sink the resulting LOW-state current without exceeding its VOL or sink-current specification.

For the 30%-to-70% voltage interval used by UM10204:

\[
t_r = 0.8473 R_p C_b
\]

Therefore:

\[
R_{p(max)} = \frac{t_{r(max)}}{0.8473 C_b}
\]

A simplified lower bound is:

\[
R_{p(min)} = \frac{V_{DD(max)}-V_{OL(max)}}{I_{OL}}
\]

Use the actual device guarantees, supply tolerance and standard-specific current limits. The equations are a design aid; the complete compliance check must use UM10204 and every component datasheet.

### Worked example — Fast-mode, 3.3 V bus

Assumptions:

- VDD = 3.3 V nominal.
- VOL(max) = 0.4 V at the selected sink-current test point.
- IOL = 3 mA guaranteed.
- Estimated total bus capacitance Cb = 200 pF.
- Fast-mode maximum rise time tr(max) = 300 ns.

Lower bound:

\[
R_{p(min)} = \frac{3.3-0.4}{0.003} \approx 967\ \Omega
\]

Upper bound:

\[
R_{p(max)} = \frac{300\ ns}{0.8473 \times 200\ pF}
\approx 1.77\ k\Omega
\]

A candidate 1.5 kΩ pull-up is inside this illustrative interval.

\[
t_r = 0.8473 \times 1.5\ k\Omega \times 200\ pF
\approx 254\ ns
\]

LOW-state current at 0.4 V is approximately:

\[
I_{LOW} = \frac{3.3-0.4}{1.5\ k\Omega}
\approx 1.93\ mA
\]

This example is not a universal recommendation. Recalculate with actual VDD tolerance, guaranteed VOL/IOL, board capacitance, temperature and all connected devices. A logic analyzer probe can add meaningful capacitance.

### Practical consequences

- Too-large Rp: slow rising edges; controller may sample before VIH is reached.
- Too-small Rp: excessive LOW current, VOL violation, extra power and possible pad overstress.
- Too much Cb: no resistor may satisfy both limits.
- Remedies include reducing bus speed, shortening/rerouting the bus, removing loads, using a suitable buffer/mux or selecting a supported stronger-drive mode.

## 5. Protocol fundamentals

### 5.1 START, repeated START and STOP

- **START (S):** SDA transitions HIGH to LOW while SCL is HIGH.
- **Repeated START (Sr):** same electrical condition as START, but the bus remains busy; no STOP occurred between transfers.
- **STOP (P):** SDA transitions LOW to HIGH while SCL is HIGH.
- After STOP, the bus becomes free only after the specified bus-free interval tBUF.

~~~text
Idle:       SDA=1, SCL=1

START:      SDA 1 -> 0 while SCL=1

Data bit:   SDA changes while SCL=0
            SDA stable while SCL=1

Repeated:   SDA 1 -> 0 while SCL=1
            previous transfer continues as one bus transaction

STOP:       SDA 0 -> 1 while SCL=1
~~~

### 5.2 Byte format and ninth clock

Every ordinary I²C byte is 8 bits, transmitted most-significant bit first, followed by one ACK/NACK clock. The receiver controls the ninth bit.

| Ninth-clock SDA | Meaning |
|---|---|
| LOW | ACK |
| HIGH | NACK |

The transmitter releases SDA during the ninth clock so the receiver can pull it LOW for ACK.

A NACK is not automatically a hardware fault. It can mean:

1. No target responded to the address.
2. The target is temporarily not ready.
3. The target rejected a command or data byte.
4. The target cannot accept more bytes.
5. The controller-receiver is ending a read by NACKing the final byte.

For a controller read, the controller ACKs each byte it wants to continue receiving and NACKs the final byte before STOP or repeated START.

### 5.3 7-bit addressing

The first address byte is:

~~~text
A6 A5 A4 A3 A2 A1 A0 R/W
<------- 7 bits ------> <1>
~~~

- R/W = 0: controller writes to target.
- R/W = 1: controller reads from target.
- The address byte is followed by target ACK/NACK.
- Software APIs generally expect the unshifted 7-bit address. Check the controller register format and driver API: some hardware registers store the address aligned with the R/W bit position.

Example: 7-bit target address 0x50.

- Write address byte = 0xA0.
- Read address byte = 0xA1.

Do not pass 0xA0 to an API that expects the 7-bit value 0x50. This address-shift mismatch is a common bring-up bug.

### 5.4 10-bit addressing

10-bit addressing uses a reserved prefix beginning 11110, two address MSBs in the first byte, then the remaining eight address bits in a second byte. A 10-bit read uses the defined repeated-START sequence: initial address phase with write direction, second address byte, repeated START, then the first header byte with read direction. Many common controllers and target devices support only 7-bit addressing; confirm capability before assuming 10-bit support.

Reserved address ranges must not be allocated casually. General Call, START byte, Hs-mode controller code, Device ID and 10-bit address prefixes occupy reserved patterns.

## 6. Transaction walkthroughs

### 6.1 Simple write

~~~text
S | SLA+W | ACK | DATA0 | ACK | DATA1 | ACK | P
~~~

Controller sends the address and data. Target ACKs each accepted byte. Controller generates STOP.

### 6.2 Simple read

~~~text
S | SLA+R | ACK | DATA0 | ACK | DATA1 | NACK | P
~~~

Target drives data. Controller ACKs bytes it wants to continue receiving and NACKs the final byte.

### 6.3 Combined register read — common codec/sensor sequence

Many register-based targets require the controller to write a register pointer, then read data without releasing the bus:

~~~text
S
| SLA+W | ACK
| REGISTER_INDEX | ACK
Sr
| SLA+R | ACK
| DATA_MSB | ACK
| DATA_LSB | NACK
P
~~~

Why repeated START? It changes direction while keeping the bus transaction active. The target can retain the selected register pointer across the repeated START. A STOP followed by a new START may reset or change target state, depending on that device's protocol.

### 6.4 Example transaction accounting

Assume:
- 7-bit address.
- 1-byte register index.
- 2-byte read payload.
- Combined write-pointer/read-data sequence.

Bytes clocked:
- Address + write: 1 byte.
- Register index: 1 byte.
- Address + read: 1 byte.
- Two data bytes: 2 bytes.
- Each byte consumes 8 data clocks plus one ACK/NACK clock.

Total = 5 × 9 = 45 SCL clocks.

At 400 kHz:

\[
t_{clocked} = \frac{45}{400,000} = 112.5\ \mu s
\]

This is a lower-bound clocked-byte estimate. Add START/repeated-START/STOP setup and hold times, bus-free time where applicable, rise/fall effects, clock stretching, software/FIFO gaps and any controller implementation overhead. Do not treat 400 kHz as a guaranteed sustained payload rate.

## 7. Multi-controller operation: clock synchronization and arbitration

### 7.1 Clock synchronization

Because SCL is wired-AND, a controller cannot force the bus HIGH while another device holds it LOW.

If multiple controllers begin together:
- The SCL LOW interval lasts until all controllers release SCL.
- The controller with the longest LOW interval determines the shared LOW time.
- Once SCL rises, the first controller to finish its HIGH interval pulls it LOW again.
- The shared HIGH interval is therefore determined by the shortest controller HIGH interval.

The observed SCL, not merely the controller's internal timer, determines when the next bit can proceed.

### 7.2 Bitwise arbitration

Arbitration is non-destructive:

1. Two controllers may begin with START when the bus is free.
2. Both transmit their intended address/data bits.
3. A transmitted logical 1 means release SDA.
4. A transmitted logical 0 means pull SDA LOW.
5. While SCL is HIGH, each controller compares its intended SDA bit with the observed bus.
6. A controller that released SDA for 1 but observes 0 loses arbitration and releases its output.
7. The winning controller continues; the losing controller retries only after the bus becomes free.

The LOW bit dominates. The losing controller must not issue STOP or corrupt the winner's transfer.

Arbitration is not a priority-number system and does not guarantee fairness. A controller/target combo may need to switch into target mode if it loses during the address phase and the winner is addressing it.

### 7.3 Clock stretching

A target may hold SCL LOW after a byte to delay the next clock until it is ready. A controller must release SCL and wait until the physical SCL input actually reads HIGH. Merely waiting a fixed internal timer and assuming the line rose is incorrect.

Clock stretching is optional. Some controllers, bridges and bit-banged implementations do not support it. A system must verify that every controller on the bus handles the target's stretching behavior.

Timeouts are implementation- or companion-protocol-specific. I²C itself permits stretching without the SMBus cumulative timeout rule; SMBus adds a bounded timeout convention.

## 8. Operating modes and timing

### 8.1 Common bidirectional modes

| Mode | Maximum SCL frequency | Typical maximum rise time |
|---|---:|---:|
| Standard-mode | 100 kHz | 1000 ns |
| Fast-mode | 400 kHz | 300 ns |
| Fast-mode Plus | 1 MHz | 120 ns |
| High-speed mode | 3.4 MHz | See Hs-mode timing table and topology rules |

NXP UM10204 also documents Ultra Fast-mode up to 5 Mbit/s. UFm uses unidirectional push-pull USDA/USCL signaling and does not provide the normal bidirectional open-drain ACK/arbitration behavior. Treat it as a distinct electrical/protocol mode.

### 8.2 Selected F/S/Fm+ timing limits from UM10204 Rev. 7.0

| Parameter | Standard | Fast | Fast-mode Plus |
|---|---:|---:|---:|
| tLOW min | 4.7 µs | 1.3 µs | 0.5 µs |
| tHIGH min | 4.0 µs | 0.6 µs | 0.26 µs |
| tr max | 1000 ns | 300 ns | 120 ns |
| tf max | 300 ns | 300 ns | 120 ns |
| tBUF min | 4.7 µs | 1.3 µs | 0.5 µs |
| fSCL max | 100 kHz | 400 kHz | 1 MHz |

Use the complete specification table for all setup/hold, data-valid, spike-suppression, output-current, bus-capacitance and voltage limits. The table above is a teaching extract, not a complete compliance checklist.

### 8.3 Controller timing generator

A controller often derives SCL from an internal peripheral clock using programmed counters:

- LOW count.
- HIGH count.
- SDA setup and hold.
- START hold and setup.
- STOP setup.
- Bus-free interval.
- Input filter width.
- Rise/fall-time budget or sample delay.

For a simplified no-stretch timing estimate:

\[
T_{SCL} \approx t_{LOW}+t_{HIGH}+t_r+t_f
\]

The actual timing engine must also satisfy all protocol minima/maxima and account for synchronizer latency, pad delay, level shifter delay and measured bus rise/fall time.

**Important:** rise time is not generated by the digital controller counter. It is a physical RC property of the bus and its pull-up network. The controller must budget for it and sample safely.

## 9. Controller and target RTL architecture

### 9.1 Controller datapath

~~~mermaid
flowchart TB
  REG[APB / AXI-Lite registers] --> CMD[Command or format FIFO]
  TX[TX FIFO] --> SH[Transmit shift register]
  CMD --> FSM[Controller FSM]
  SH --> FSM
  FSM --> TIM[Timing counter]
  TIM --> SCL[SCL pull-low control]
  FSM --> SDA[SDA pull-low control]
  SDA --> PAD[Open-drain pad cells]
  SCL --> PAD
  PAD --> BUS[SDA / SCL]
  BUS --> SYNC[Input synchronizers and filters]
  SYNC --> MON[START/STOP and bus monitor]
  MON --> FSM
  BUS --> RX[Receive shift register]
  RX --> RXF[RX FIFO]
  FSM --> IRQ[Interrupt and error status]
~~~

### 9.2 Controller FSM — illustrative states

~~~text
IDLE / WAIT_BUS_FREE
  -> START_HOLD
  -> SEND_ADDRESS
  -> WAIT_ADDRESS_ACK
  -> WRITE_DATA or READ_DATA
  -> WAIT_DATA_ACK / DRIVE_ACK_OR_NACK
  -> NEXT_BYTE
  -> REPEATED_START or STOP
  -> DONE

Exceptional exits:
  LOST_ARBITRATION -> RELEASE_LINES -> WAIT_BUS_FREE
  STRETCH_WAIT -> RESUME when SCL is observed HIGH
  TIMEOUT -> ABORT / RELEASE_LINES / INTERRUPT
  BUS_ERROR -> SAFE_RELEASE / STATUS
~~~

A production FSM needs precise transition conditions for every clock edge, ACK phase, repeated START, STOP and error. The above is a structural guide, not synthesizable complete RTL.

### 9.3 Target FSM — illustrative states

~~~text
IDLE
  -> START_DETECT
  -> ADDRESS_SHIFT
  -> ADDRESS_COMPARE
  -> ACK_ADDRESS or IGNORE
  -> RECEIVE_DATA / TRANSMIT_DATA
  -> ACK_RECEIVED_BYTE / SAMPLE_CONTROLLER_ACK
  -> WAIT_FOR_NEXT_BYTE
  -> STOP_DETECT or REPEATED_START_DETECT
  -> IDLE
~~~

A target that transmits must release SDA for the controller's ACK/NACK bit. A target that receives must release SDA during the ninth clock so it can pull LOW for ACK or leave HIGH for NACK.

### 9.4 CDC and input filtering

SCL/SDA are external asynchronous signals. Typical input path:

1. Pad input buffer.
2. Optional glitch/spike filter.
3. Two-or-more-stage synchronizer.
4. START/STOP edge detector and bit sampler.

The exact filter must not suppress legal transitions or distort timing. Synchronizer latency, pad delay and RC rise time must be included in timing budgets. A digital simulation with ideal zero-delay pads cannot prove electrical timing compliance.

## 10. Registers and programming model

There is no universal I²C register map. A typical implementation exposes categories such as:

| Register group | Purpose |
|---|---|
| CTRL | Enable controller/target mode and options |
| STATUS | Bus busy, FIFO levels, current state, error state |
| TIMING | SCL low/high and setup/hold counters |
| TX_FIFO / FMT_FIFO | Address and bytes/commands to transmit |
| RX_FIFO | Received data |
| TARGET_ADDR | Target address and mask |
| INTR_STATE | Latched interrupt events |
| INTR_ENABLE | Interrupt enables |
| INTR_TEST | Verification/software interrupt injection |
| TIMEOUT_CTRL | Stretch or bus timeout controls |
| OVERRIDE | Debug pin control, if implemented |

### 10.1 Public implementation study — OpenTitan

OpenTitan's public I²C IP is a controller-target combo. Its documentation describes:
- Controller and target functions enabled through CTRL fields.
- Two-flop synchronization of SDA/SCL inputs.
- A bus monitor for START/STOP, bus busy and arbitration-loss detection.
- A format FIFO controlling bytes, reads, START and STOP insertion.
- TX/RX and target acquisition queues.
- Programmable timing fields and timeout/interrupt behavior.
- Pinmux connection to SDA/SCL.

Its register names include CTRL, FDATA, RX FIFO, target ID and TIMING0 through TIMING4. These are OpenTitan implementation details, not I²C-standard register names.

Reference:
https://opentitan.org/book/hw/ip/i2c/doc/theory_of_operation.html

### 10.2 Safe software initialization sequence

1. Ensure the I/O bank and pull-up rail are powered.
2. Configure pinmux to the I²C function.
3. Configure the pads for the required open-drain/release behavior and correct voltage tolerance.
4. Confirm both SDA and SCL are HIGH before enabling the controller.
5. Assert/deassert the IP reset according to the SoC reset sequence.
6. Disable the controller while programming timing and FIFO thresholds.
7. Program timing for the selected mode, internal clock and measured bus load.
8. Configure target address/mask if target mode is required.
9. Clear stale interrupt/error status using the documented W1C semantics.
10. Enable controller/target and interrupt sources.
11. Submit a transaction and wait for completion or error.
12. On completion, verify STOP/bus-idle state and drain RX data.
13. On timeout/arbitration loss, release the lines and recover without disturbing another active controller.

### 10.3 Example register access — pseudocode

~~~c
// Illustrative only: replace with the actual SoC driver and register definitions.
i2c_disable(ctrl);
i2c_clear_status(ctrl, I2C_STATUS_W1C_MASK);
i2c_program_timing(ctrl, timing_for_fast_mode(pclk_hz, bus_cap_pf));
i2c_set_target_address(ctrl, target_addr_7bit);
i2c_enable_controller(ctrl);

i2c_start(ctrl);
i2c_write_byte(ctrl, (target_addr_7bit << 1) | I2C_WRITE);
check_ack_or_abort(ctrl);
i2c_write_byte(ctrl, register_index);
check_ack_or_abort(ctrl);
i2c_repeated_start(ctrl);
i2c_write_byte(ctrl, (target_addr_7bit << 1) | I2C_READ);
check_ack_or_abort(ctrl);
data[0] = i2c_read_byte(ctrl, I2C_ACK);
data[1] = i2c_read_byte(ctrl, I2C_NACK);
i2c_stop(ctrl);
~~~

Do not mix software bit-banging with a hardware controller still driving the same pins. Disable or place the hardware block in a documented override mode before GPIO-based recovery.

## 11. SoC integration dependencies

### 11.1 Address and bus integration

- Map the register interface into the SoC peripheral address map.
- Use APB or AXI-Lite for low-rate configuration.
- Verify byte-lane and endianness behavior of FIFO/register accesses.
- Ensure FIFO accesses are not accidentally reordered or merged by an unsuitable bridge.
- If DMA is implemented, treat DMA as an optional controller-specific optimization, not a protocol requirement.

### 11.2 Clock and reset

- Peripheral clock frequency determines timing-counter resolution.
- Clock changes require timing-register recalculation or a safe reconfiguration sequence.
- Reset must release the I²C IP without creating spurious START/STOP edges.
- A reset during a transfer must release SDA/SCL and leave the bus monitor in a defined state.
- In multi-controller systems, a local IP reset must not be confused with permission to reset the whole external bus.

### 11.3 Power and pinmux

- The I²C block may be clock-gated while the bus is idle.
- A target may need to remain powered to ACK a wakeup or management transaction.
- Always-on PMIC or wakeup I²C paths need explicit power-domain planning.
- Pinmux ownership conflicts can make the IP appear broken even when the controller RTL is correct.
- Level shifters and bus switches add capacitance and propagation delay.

### 11.4 Interrupts

Common events:
- TX FIFO threshold/empty.
- RX FIFO threshold/full.
- Transfer complete.
- Address/data NACK.
- Arbitration lost.
- Stretch timeout or bus timeout.
- Unexpected STOP/START or signal interference.
- Target-side acquisition or transmit underflow.

Interrupt clear semantics must be checked. A W1C status bit should be cleared with the documented write-one-to-clear behavior, not by read-modify-write if that can clear unrelated events.

## 12. Common design and integration bugs

| Symptom | Likely cause | Evidence / debug |
|---|---|---|
| Address NACK | Wrong address, wrong shifted/unshifted convention, target unpowered | Decode first address byte and verify rail |
| SDA never rises | Missing pull-up, short, target stuck LOW, wrong pinmux | Scope both lines and isolate devices |
| SCL never rises | Target stretching, short, pad configured push-pull LOW | Check pad OE and target state |
| Works at 100 kHz but fails at 400 kHz | Rise time, setup/hold, filter or synchronizer budget | Measure 30–70% rise and SDA setup |
| Intermittent data corruption | Glitch filter too short, noise, ringing, marginal VIH | Analog scope plus protocol decoder |
| Multi-controller deadlock | Arbitration loss not handled; controller drives after losing | Force simultaneous START in simulation |
| Read returns first byte repeatedly | Missing repeated START or target register-pointer semantics | Decode full transaction |
| Final byte missing | Controller ACK/NACK handling wrong | Verify final NACK and STOP |
| Bus stuck after target reset | Target reset mid-byte; SDA remains held | Follow bus-clear/reset procedure |
| Works only when debugger attached | Timing margin, reset/power sequencing or probe capacitance | Reproduce without debugger; inspect scope loading |
| Random failure after DVFS | Timing counters not updated for new peripheral clock | Recompute timing after clock transition |
| Failure only on one board | Pull-up, capacitance, level shifter, connector or routing | Compare BOM and analog waveforms |

### 12.1 Bus recovery — use the right procedure

NXP UM10204 describes:
- If **SDA is stuck LOW**, the controller should send up to nine clock pulses; the device holding SDA may release it during those clocks.
- If **SCL is stuck LOW**, first use a hardware reset if the devices provide one; otherwise power-cycle the affected devices to invoke their POR behavior.
- If SDA remains LOW after nine clocks, use hardware reset or power-cycle the offending device.

Recovery is not permission to pulse a bus that another controller is actively using. First establish bus ownership and ensure the bus is genuinely wedged. A robust SoC may temporarily mux the pins to GPIO, release SDA, pulse SCL open-drain, generate a legal STOP if possible, then restore pinmux and reinitialize the controller.

## 13. Pre-silicon verification plan

### 13.1 Protocol tests

- START and STOP detection.
- Repeated START with no STOP.
- 7-bit and 10-bit address cases if supported.
- Read and write, zero/one/multiple-byte transfers.
- ACK and NACK at address and data phases.
- Controller NACK of final read byte.
- Reserved-address handling.
- General Call behavior if implemented.

### 13.2 Multi-controller and target tests

- Simultaneous START.
- Arbitration loss on address and data bits.
- Identical prefixes followed by different bits.
- Clock synchronization with different controller LOW/HIGH periods.
- Target clock stretching.
- Target address match and non-match.
- Controller reset while another controller owns the bus.
- Controller/target dual-role transition if supported.

### 13.3 Timing and fault tests

- Minimum/maximum timing counters.
- Input spike filter boundary.
- Slow rise time and long fall time.
- SCL stretch near timeout.
- Reset during address, data, ACK and STOP.
- FIFO overflow/underflow.
- APB wait states and software service latency.
- Clock frequency change while idle and during active transfer (the latter should be prohibited or handled explicitly).
- Pinmux reset and power-domain sequencing.

### 13.4 Useful assertions — conceptual examples

~~~systemverilog
// SDA must not change while SCL is high, except for START/STOP.
// A real checker must qualify legal START/STOP and filter/sample semantics.
property p_data_stable_when_scl_high;
  @(posedge sample_clk) disable iff (!rst_n)
    scl_sync && !start_seen && !stop_seen
      |-> $stable(sda_sync);
endproperty
assert property (p_data_stable_when_scl_high);

// When the controller loses arbitration, it must release SDA.
property p_release_after_lost_arbitration;
  @(posedge clk) disable iff (!rst_n)
    arbitration_lost |=> !sda_pull_low;
endproperty
assert property (p_release_after_lost_arbitration);

// Never actively drive a logical HIGH on a normal open-drain output.
assert property (@(posedge clk) disable iff (!rst_n)
  !(sda_drive_enable && sda_drive_data));
~~~

These are teaching sketches. Production assertions need clocking/event definitions aligned with the pad sampling path, legal START/STOP exceptions, and the exact active-low/active-high pad convention.

### 13.5 Coverage ideas

- Every address ACK/NACK outcome.
- Every byte index and transfer direction.
- Repeated START after write and after read.
- Arbitration loss at each address/data bit position.
- Stretch durations at zero, short, near-timeout and timeout.
- Bus recovery at each stuck-line condition.
- Reset/power event at every FSM state.
- FIFO boundary and overflow/underflow.
- Timing mode × peripheral clock × bus capacitance configuration.

## 14. Post-silicon bring-up and validation

### Stage A — electrical sanity

1. Check I/O supply and pull-up rail.
2. Confirm correct pinmux and pad configuration.
3. Measure idle SDA/SCL voltage.
4. Measure rise/fall time using 30% and 70% VDD thresholds.
5. Check LOW voltage at the actual sink current.
6. Confirm no unpowered device clamps the lines.

### Stage B — protocol smoke test

1. Start at 100 kHz.
2. Read a known ID register from one target.
3. Decode address, R/W, ACK and data.
4. Test repeated START register reads.
5. Test a deliberate NACK (wrong address) and confirm graceful error reporting.
6. Move to 400 kHz only after the analog timing margin is demonstrated.

### Stage C — stress and corner characterization

- Multiple targets and address conflicts.
- Long cable/board route if applicable.
- Minimum/maximum supply.
- Temperature corners.
- Clock frequency variation.
- Clock stretching.
- Bus contention and target reset.
- Repeated transactions and long-duration soak.
- Power-domain off/on and resume.
- EMI/noise injection within the product validation plan.

### Instrument selection

| Instrument | What it reveals |
|---|---|
| Oscilloscope | Analog rise/fall, ringing, VOL, overshoot and timing margin |
| Logic analyzer | START/STOP, address, ACK/NACK and byte sequence |
| Protocol analyzer | Multi-controller/arbitration and decoded transaction history |
| JTAG/trace | FSM state, FIFO pointers, interrupt status and timeout counters |
| Firmware log | Driver retries, error codes, reset cause and target sequence |

A logic analyzer can decode a transaction but cannot prove analog rise-time compliance. Use an oscilloscope for electrical validation.

## 15. Case study — configuring an audio codec over I²C

A SoC contains:
- CPU.
- I²C controller.
- Audio codec target.
- I²S/TDM audio interface.
- Audio DSP and DMA.

A typical bring-up sequence is:

1. Power the codec analog/digital rails in the datasheet-defined order.
2. Release codec reset.
3. Confirm I²C bus idle HIGH.
4. Read codec ID/revision.
5. Program PLL and system clock source.
6. Program sample rate and serial-port format.
7. Configure TDM slot width/channel mapping.
8. Unmute and enable the audio path only after clocks/data are stable.

The I²C bus is only the control plane. Audio samples travel through I²S/TDM or another audio data interface, not as bulk I²C transfers.

A useful diagnostic split:
- I²C NACK: check codec power, reset, address straps and pinmux.
- I²C ACK but wrong register: check register-pointer width, byte order, repeated START and codec page/bank selection.
- Registers correct but no audio: debug MCLK/BCLK/LRCLK, TDM slots, clocks, reset and DMA separately.
- Audio starts then stops: inspect codec power state, clock source, mute sequencing and firmware writes.

## 16. Comparison with related buses

| Feature | I²C | SPI | UART | I3C |
|---|---|---|---|---|
| Wires | 2 shared | Usually SCLK, MOSI, MISO + CS | TX/RX | 2 shared |
| Clock | Synchronous, SCL | Synchronous, SCLK | Asynchronous | Synchronous |
| Electrical | Open-drain for normal bidirectional I²C | Usually push-pull | Usually push-pull | Mixed signaling defined by I3C |
| Addressing | In-band 7/10-bit | Usually chip-select | Usually point-to-point or higher-level framing | Dynamic addressing and defined legacy coexistence |
| Multi-controller arbitration | Yes | Not generally built into basic SPI | No native shared-bus arbitration | Defined I3C mechanisms |
| Typical use | Low-rate control | Faster peripheral data | Console/serial links | Modern sensor/control fabric |

I3C is the next lesson. It should be understood as a newer protocol with its own electrical, dynamic-addressing and in-band-interrupt rules—not merely “faster I²C.”

## 17. Hands-on labs

### Lab 1 — Decode a codec register read
Given a captured waveform, identify START, 7-bit address, write bit, register pointer, repeated START, read bit, returned bytes, final NACK and STOP.

### Lab 2 — Pull-up calculator
Write a small script or spreadsheet that accepts VDD, VOL, IOL, Cb and tr(max), then calculates Rp(min), Rp(max) and rise time for candidate resistor values. Reject infeasible intervals.

### Lab 3 — Controller RTL
Implement an I²C controller FSM with open-drain SDA/SCL controls. Start with one controller and 7-bit writes; add reads, repeated START and NACK handling.

### Lab 4 — Multi-controller arbitration
Instantiate two controllers with different addresses/data. Make them issue START together and verify that the controller transmitting 1 while observing 0 loses arbitration without corrupting the winner's transaction.

### Lab 5 — Clock stretching and recovery
Make a target hold SCL LOW for programmable cycles. Verify the controller waits for observed SCL HIGH. Add a stuck-SDA test and a nine-pulse recovery sequence.

### Lab 6 — Linux integration
Bind a real or emulated target to an I²C adapter. Use a combined write/read transfer and verify that the driver emits repeated START rather than STOP+START.

## 18. Interview questions

1. Why does I²C use open-drain signaling instead of push-pull?
2. Why does every byte require a ninth clock?
3. Who drives the ACK bit during a write? During a read?
4. Why does a controller NACK the final byte of a read?
5. What is the difference between START and repeated START?
6. Why must SDA remain stable while SCL is HIGH?
7. How does multi-controller arbitration avoid corrupting the winning transfer?
8. Why does a transmitted 1 but observed 0 mean arbitration loss?
9. How is SCL synchronized when two controllers have different timing?
10. What is clock stretching and which device may use it?
11. Why can a bus work at 100 kHz but fail at 400 kHz?
12. Derive the 0.8473 RC rise-time relation.
13. Calculate pull-up bounds for a given VDD, VOL, IOL and bus capacitance.
14. Why is 0xA0 not always the correct software address for a 7-bit target at 0x50?
15. What is the 10-bit address prefix?
16. Why is a repeated START often required for register reads?
17. What can cause SDA to remain LOW after a target reset?
18. When are nine recovery clocks appropriate, and what if SCL is stuck LOW?
19. Why does a logic analyzer not prove rise-time compliance?
20. What must be reprogrammed if the I²C peripheral clock changes?

## 19. Design-review checklist

- [ ] Protocol mode and all target capabilities are known.
- [ ] 7-bit/10-bit address conventions are unambiguous.
- [ ] Reserved addresses are excluded.
- [ ] Pull-up minimum and maximum are calculated.
- [ ] Total bus capacitance includes pins, traces, connectors, translators and probes.
- [ ] Pad cells release HIGH and never push-pull-drive HIGH in normal bidirectional mode.
- [ ] Voltage and powered-off behavior are verified for every target.
- [ ] Timing counters account for input synchronizer, pad and RC delays.
- [ ] Repeated START, final read NACK, arbitration and clock stretching are verified.
- [ ] Timeout policy is explicit and distinguishes I²C from SMBus rules.
- [ ] Reset and power-down release the bus safely.
- [ ] Bus recovery cannot disrupt another active controller.
- [ ] Interrupt clear semantics and FIFO corner cases are verified.
- [ ] Scope and logic-analyzer validation are both planned.
- [ ] Linux adapter capabilities match the hardware implementation.

## 20. Normative specifications and public implementation references

### Protocol and electrical specification

1. **NXP UM10204 Rev. 7.0 — I²C-bus specification and user manual, 1 October 2021**  
   https://www.nxp.com/docs/en/user-guide/UM10204.pdf  
   Primary reference for bus behavior, addressing, START/STOP, ACK/NACK, arbitration, clock stretching, timing, electrical characteristics and pull-up sizing.

2. **NXP I²C product and application documentation**  
   https://www.nxp.com/products/interfaces/ic-spi-i3c-interface-devices/i2c-bus  
   Device and system-level I²C interface resources.

### Public RTL and software examples

3. **OpenTitan I²C HWIP specification**  
   https://opentitan.org/book/hw/ip/i2c/  
   Public controller/target IP, FIFOs, interrupts, pinmux, timing and reset integration. This is an implementation example, not the normative I²C specification.

4. **OpenTitan I²C theory of operation**  
   https://opentitan.org/book/hw/ip/i2c/doc/theory_of_operation.html  
   Useful for studying controller/target modules, bus monitor, arbitration, format FIFO, timing counters and timeout handling.

5. **Linux I²C and SMBus subsystem documentation**  
   https://docs.kernel.org/driver-api/i2c.html  
   Adapter, client, driver and transfer abstractions. This documents Linux software interfaces, not electrical compliance.

6. **Linux I²C fault-code documentation**  
   https://docs.kernel.org/i2c/fault-codes.html  
   Useful for distinguishing NACK, arbitration loss, timeout and bus errors in driver diagnostics.

### Related protocol

7. **MIPI I3C**  
   https://www.mipi.org/specifications/i3c-sensor-specification  
   The next-generation sensor/control interface; Session 21 studies its architecture and coexistence rules in detail.

### Evidence discipline

- **Normative protocol/electrical facts:** NXP UM10204.
- **Implementation example:** OpenTitan register names, FIFOs, FSM decomposition and reset/interrupt integration.
- **Software framework:** Linux I²C adapter/client/driver model.
- **Engineering interpretation:** bus topology choices, debug prioritization and example calculations. These are explicitly presented as design guidance, not as requirements imposed by the I²C specification.

---

## Lesson 19 completion summary

You should now be able to trace an I²C transaction from CPU register writes through controller RTL, pad-cell open-drain behavior, external pull-up RC rise, target ACK/NACK, and back through interrupt/status handling. You should also be able to explain why a protocol failure can originate in software addressing, controller state, CDC/filter timing, pinmux, voltage-domain integration or board-level capacitance—and select the correct instrument to isolate it.

**Next lesson — Session 20: SPI.** We will compare clock modes, chip-select timing, full-duplex shift paths, FIFO/DMA integration, multi-device topologies, timing closure and silicon debug.
