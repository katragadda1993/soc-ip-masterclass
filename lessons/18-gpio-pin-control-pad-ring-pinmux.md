# Lesson 18 — GPIO and Pin Control
## From a software-visible bit to the pad-ring transistor and board-level signal

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 18 of 38  
**Target duration:** 60 minutes  
**Level:** architecture, electrical behavior, RTL, integration, verification and post-silicon  
**Prerequisites:** APB/AXI-Lite register access, interrupt controllers, reset/clock/power domains and basic CMOS logic

> **Scope discipline:** GPIO is a class of programmable digital I/O function, not one universal wire protocol. Register layouts, pin numbering, drive-strength codes, pull resistances, mux encodings, voltage limits and timing are implementation-specific. This lesson separates generic electrical principles, Linux software abstractions and documented reference-IP behavior. A product datasheet, pad-library specification, board schematic and applicable SoC reference manual remain authoritative for a real design.

---

## 1. Learning objectives and one-hour plan

By the end, you should be able to trace a software GPIO request all the way to a package ball; explain the separate roles of GPIO, pinmux and pad control; reason about input synchronization, output enable, open-drain behavior and interrupts; calculate a first-order pull-up rise time and debounce interval; configure a Linux pinctrl state; and construct a verification and silicon-debug plan.

| Time | Topic | Outcome |
|---:|---|---|
| 0–5 min | Why GPIO and pin control exist | GPIO versus peripheral functions and historical context |
| 5–15 min | Pad ring and electrical architecture | Understand the actual input/output cell |
| 15–25 min | GPIO RTL and registers | Trace DATA_IN, DATA_OUT, OE, interrupt and filter paths |
| 25–35 min | Pinmux and pinconf | Understand function selection, pad attributes and resource conflicts |
| 35–43 min | Timing and calculations | Synchronizer delay, debounce, pull-up RC and loading |
| 43–52 min | Integration and failure analysis | Boot straps, reset, voltage domains, interrupts and Linux |
| 52–60 min | Verification, labs and interview review | Build a validation plan |

### Learning outcomes

1. Distinguish a GPIO controller, pin controller, pinmux matrix and physical pad cell.
2. Explain the difference between a logical output value and the voltage actually observed at the package pin.
3. Explain push-pull, open-drain, open-source, high impedance, pull-up/down, keeper and Schmitt-trigger input.
4. Trace a GPIO input through pad, synchronization, optional filtering, data register and interrupt detection.
5. Configure a pin function without causing contention or a glitch.
6. Explain why pin groups and alternate functions are SoC-specific.
7. Debug a pin that works as GPIO but fails as I2C, SPI, UART or I2S.
8. Plan pre-silicon and board-level validation.

---

## 2. Why GPIO exists — historical backstory and design rationale

### 2.1 Before highly integrated SoCs

Early microprocessor systems commonly used external peripheral-interface devices to connect the CPU bus to switches, LEDs, relays, keyboard matrices, control lines and simple parallel data paths. Software needed a programmable way to decide whether a line was an input or output and, in richer devices, to select handshaking modes.

The Intel 8255A Programmable Peripheral Interface is a useful documented example from this era. Intel's 1978 Component Data Catalog lists the 8255A as a programmable peripheral interface; Intel's 1979 Peripheral Design Handbook describes its role as a general-purpose I/O component connected to the microcomputer system bus. It offered 24 I/O lines divided into ports and programmable operating modes, including modes with handshaking. This is a historical example, not the origin of every GPIO implementation. [Intel 1978 Component Data Catalog](https://bitsavers.trailing-edge.com/components/intel/_dataBooks/1978_Intel_Component_Data_Catalog.pdf) · [Intel 1979 Peripheral Design Handbook — 8255A](https://bitsavers.trailing-edge.com/components/intel/_dataBooks/1979_Intel_Peripheral_Design_Handbook.pdf) · [Cambridge University Press — 8255 PPI data sheets](https://doi.org/10.1017/CBO9780511622885.014)

### 2.2 Why GPIO became an SoC subsystem

As more functions moved on die, a separate external PPI for every simple control signal became unattractive. Integrating GPIO registers and interrupt logic reduced board components and allowed firmware to configure pins directly.

A second problem then became dominant: package pins are scarce. A modern SoC may contain UART, SPI, I2C, I3C, I2S/TDM, SD/MMC, PWM, JTAG, trace and wakeup signals, but it cannot dedicate a separate package ball to every signal. Pin multiplexing lets one physical pad serve one selected function at a time.

A third problem is electrical diversity. The same package pin may need different bias, drive, slew, input threshold, open-drain mode or voltage-domain behavior depending on the connected board device. That led to a separation between:
- **GPIO data/control:** what logical value to drive or sample.
- **Pinmux:** which internal peripheral signal is connected to a physical pad.
- **Pad configuration:** how the electrical pad behaves.

These are related but not interchangeable functions.

### 2.3 A documented software milestone

On 13 October 2011, a Linux kernel pin-control subsystem patch described a common framework for pin multiplexing and electrical configuration. Its stated motivation included reducing duplicated architecture-specific code and avoiding feature-heavy GPIO drivers that each reimplemented pin-control behavior. The patch history is useful evidence for the software abstraction's design rationale; it does not mean pin multiplexing itself began in 2011. [Linux kernel mailing-list patch, 13 October 2011](https://lkml.iu.edu/1110.1/01966.html)

### 2.4 Design choices and trade-offs

| Design choice | Benefit | Cost / risk |
|---|---|---|
| Dedicated pin per function | Simple routing and no mux conflicts | More package balls, larger package and board |
| Multiplexed pad | Fewer pins and flexible board variants | Functions compete for pins; software must configure correctly |
| Fixed-function GPIO | Simple, predictable control | Less flexible integration |
| GPIO plus pinctrl split | Clear separation of data and electrical/mux policy | Requires coordination across drivers and firmware |
| Stronger output driver | Better edge rate and load capability | More switching current, ringing and EMI |
| Slower slew | Reduced high-frequency content and ringing | Longer edge and timing margin consumption |
| Internal pulls | Fewer external components | Tolerance, resistance and leakage may be less suitable than an external resistor |
| Open-drain | Shared wired-AND line and level translation with suitable pull-up | RC-limited rising edge and lower speed |
| Per-pin programmable mux | Flexible routing | More mux logic, configuration state and verification |
| Group-based mux | Ensures related signals move together | Less flexible; group conflicts need arbitration |

**Engineering interpretation:** pinmux is primarily a package-resource and board-configuration solution. It trades pin count and routing simplicity for a configuration problem that must be solved consistently by RTL, firmware, device tree, board design and validation.

---

## 3. System architecture — from CPU register to package ball

### 3.1 The complete path

~~~mermaid
flowchart TD
  CPU[CPU / firmware] --> BUS[AXI-Lite / APB / peripheral bus]
  BUS --> GPIO[GPIO controller]
  GPIO -->|DATA_OUT and DATA_OE| PMUX[Pinmux / output selection]
  PMUX --> PAD[Pad control and I/O cell]
  PAD --> BALL[Package ball / board net]
  BALL --> EXT[External switch, sensor, LED or device]
  EXT --> BALL
  BALL --> PADIN[Pad input receiver]
  PADIN --> SYNC[Input synchronizer]
  SYNC --> FILT[Optional digital filter]
  FILT --> DIN[GPIO DATA_IN]
  FILT --> IRQ[Edge / level detector]
  IRQ --> INTC[Interrupt controller]
  INTC --> CPU
~~~

The physical path also includes power rails, ESD structures, package parasitics, board traces, external pull resistors and external device output stages. The block diagram is logical; a real pad ring has technology-specific cells and supply connections.

### 3.2 Three distinct control planes

| Plane | Main question | Typical controls |
|---|---|---|
| GPIO | What logic value should the GPIO peripheral drive or sample? | Output data, output enable, input data, interrupt enable |
| Pinmux | Which internal signal owns this pad? | GPIO, UART, SPI, I2C, I2S, PWM, JTAG selection |
| Pad control | What electrical behavior should the pad use? | Pull, drive strength, slew, input enable, Schmitt, open drain |

Do not assume every SoC has three separate register blocks. Some combine them physically or expose a single IOMUX controller. The conceptual responsibilities still need to be understood separately.

### 3.3 Example: one pad with several possible owners

~~~text
                         +---------------- GPIO_OUT
                         |
UART_TX -----------------|
SPI_MOSI ----------------|    +-------------------+
I2S_BCLK ----------------+--->| output mux         |----> pad output driver
PWM0 --------------------|    | selected function  |
                         |    +-------------------+
                         |
GPIO_IN <----------------+-------------------------- pad input receiver
UART_RX <----------------|    input mux
SPI_MISO <---------------|
I2S_BCLK_IN <------------|

Pad attributes: pull / drive / slew / input enable / open-drain / voltage bank
~~~

A pin may have separate input and output muxes, or a combined function selection. Never assume the exact topology without the SoC integration manual.

---

## 4. Pad-ring and I/O-cell electrical architecture

### 4.1 Simplified bidirectional pad

~~~text
                       VDDIO
                         |
                   +-----+-----+
                   | P-MOS     |  output-high device
                   +-----+-----+
                         |
Internal OUT ---------- mux ------ PAD ---------------- Package ball
                         |           |
                   +-----+-----+     +----> input receiver
                   | N-MOS     |     |       |
                   +-----+-----+     |    Schmitt / input buffer
                         |           |       |
                        VSS         ESD     input path
                                    clamp
Output enable / output mode controls the driver.
Pull-up/down and keeper circuits may connect to the pad internally.
~~~

This is a conceptual CMOS push-pull pad. Actual libraries may include pre-driver stages, selectable drive segments, separate input enable, keeper, bus-hold, ESD clamps, level shifters and multiple supply pins.

### 4.2 Push-pull output

A push-pull driver actively drives both logic levels:
- Output 1: high-side device sources current from the I/O supply.
- Output 0: low-side device sinks current to ground.
- Output disabled: both output devices are off, producing high impedance, subject to pad leakage and other configured circuits.

Use push-pull for ordinary single-driver signals when the receiver and voltage levels are compatible.

**Do not connect two push-pull outputs together** unless the hardware is explicitly designed for that arrangement. If one drives high and another drives low, large contention current can flow.

### 4.3 Open-drain output

An open-drain output actively drives low or releases the line:
- Logical 0: low-side transistor ON.
- Logical 1: output transistor OFF; an external or internal pull-up raises the line.

~~~text
VIO
 |
Rpullup
 |
 +-------------------- BUS -------------------- receiver
 |
 +---- drain of N-MOS
          source
            |
           GND

Drive low: N-MOS ON.
Release:   N-MOS OFF; pull-up raises the net.
~~~

Open-drain is useful for shared wired-AND/wired-OR signaling and I2C-style buses. It is not a substitute for a proper level-shifter or bus design where voltage, capacitance or current requirements exceed the pad specification.

### 4.4 Open-source output

Open-source is the complementary single-ended arrangement: the output actively drives high and releases the line; a pull-down establishes the low state. It is less common in SoC peripheral interfaces.

### 4.5 High impedance is not the same as logic 1

When a pin is Hi-Z, the SoC is not actively driving it. The voltage depends on:
- external pull-up or pull-down,
- another device,
- leakage,
- capacitance and previous charge,
- keeper/bus-hold circuit,
- measurement instrument loading.

A floating CMOS input can toggle unpredictably and consume extra current. A safe board and firmware design must define the idle state.

### 4.6 Pull-up and pull-down

Internal pulls are weak bias elements intended to establish a default level, not normally to drive a heavy load. Their resistance range and tolerance are device-specific.

Use an external resistor when the board requires a controlled resistance, stronger bias, better tolerance or a defined behavior while the SoC is unpowered.

### 4.7 Schmitt-trigger input

A Schmitt-trigger input uses different rising and falling thresholds. The hysteresis helps reject slow/noisy transitions near the threshold. It does not eliminate the need for synchronization when the input is asynchronous to the receiving clock.

### 4.8 Drive strength and slew rate

Drive strength controls the effective output transistor width or segment selection. It is not a universally portable current value. A vendor may expose codes such as 2/4/8/12 mA, but the actual VOH/VOL, edge rate and safe loading must be read from the pad electrical specification at the relevant VDDIO, temperature and process corner.

Slew-rate control changes edge speed. Faster edges may improve setup margin but increase ringing, overshoot, crosstalk and EMI. Slower edges may reduce noise but consume timing margin and fail receiver thresholds if too slow.

### 4.9 Voltage domains and level shifting

A pad bank may operate at a different I/O voltage from the core. Important checks:
- Is the pad tolerant of the board voltage when VDDIO is absent?
- Is the pad fail-safe or does it back-power through protection structures?
- Does the input threshold match the external device's VOH/VOL?
- Are internal pull resistors connected to the intended I/O rail?
- Is a level shifter required?
- Does the bank support mixed voltages, or must every pad in the bank use the same supply?

Never infer 1.8 V/3.3 V tolerance from the fact that a pad is called GPIO.

---

## 5. GPIO controller microarchitecture

### 5.1 Typical internal blocks

~~~mermaid
flowchart LR
  REG[APB / AXI-Lite register file]
  OUT[Output data register]
  OE[Output enable register]
  IN[Input synchronizer]
  FILT[Optional debounce / noise filter]
  IRQ[Interrupt edge and level detector]
  EVT[Interrupt state / enable]
  PAD[Pad and pinmux interface]
  REG --> OUT
  REG --> OE
  PAD --> IN --> FILT
  FILT --> IRQ --> EVT --> REG
  OUT --> PAD
  OE --> PAD
~~~

A typical GPIO controller contains:
- output data register,
- output-enable/direction register,
- synchronized input data,
- optional input filter,
- interrupt type/polarity/enable registers,
- interrupt pending/status register,
- optional wakeup and hardware-strap logic,
- bus interface and clock/reset logic.

### 5.2 Common logical signals

Names vary by RTL and integration. A common conceptual interface is:

| Signal | Direction relative to GPIO block | Meaning |
|---|---|---|
| gpio_in[n] | Input | Sampled pad input from pinmux/pad ring |
| gpio_out[n] | Output | Logical output value |
| gpio_oe[n] | Output | Output-driver enable |
| intr[n] | Output | GPIO interrupt request |
| strap_en | Input | Optional reset-time strap sampling enable |
| clk_i | Input | GPIO register/sample clock |
| rst_ni | Input | Active-low reset in this example |
| psel/penable/pwrite/paddr/pwdata | Input | Example APB register interface |
| pready/prdata/pslverr | Output | Example APB response |

These are illustrative names, not a mandated GPIO interface standard.

### 5.3 Input path and synchronizer

External input changes are asynchronous to the GPIO peripheral clock in the general case. A two-flop synchronizer reduces the probability that metastability propagates into downstream logic.

~~~text
External pin:   _________/-------------------
                         |
Clock:          _|---|---|---|---|---|---|_
FF1:            ________?----1---------------
FF2:            _____________1----------------
DATA_IN:        _____________1----------------
~~~

The first flop may become metastable. The second flop samples the first after approximately one destination-clock period, giving resolution time. A two-flop synchronizer does not make metastability impossible; it reduces its propagation probability. The required number of stages depends on clock rate, data toggle rate, library characteristics and the required MTBF.

**Important:** a synchronizer is not a debounce filter. A bouncing mechanical switch may produce many valid transitions, each of which passes through a synchronizer.

### 5.4 Optional digital input filter

A simple N-consecutive-samples filter accepts a new value only after it has remained stable for N samples.

~~~text
Raw synchronized input:  0 1 0 1 1 1 1 1 1 1 1
N=4 stable detector:     0 0 0 0 0 0 1 1 1 1 1
~~~

For a 16-sample filter clocked at 24 MHz, the nominal qualification window is about 16/24 MHz = 0.667 microseconds. Exact behavior depends on whether the implementation counts the first sample, uses a counter, or compares a shift register. The OpenTitan GPIO reference describes an optional 16-cycle input filter in one implementation. Do not generalize that number to other controllers. [OpenTitan GPIO theory of operation](https://opentitan.org/book/hw/ip/gpio/doc/theory_of_operation.html)

A mechanical switch may require several milliseconds of debounce, far longer than a small noise filter.

### 5.5 Output path

The output register and output-enable register usually feed the pinmux and pad control:
- DATA_OUT selects the desired logical value.
- DATA_OE enables or disables the output driver.
- pinmux determines whether GPIO owns the pad.
- pad control determines electrical mode and drive properties.

**Safe initialization rule:** preload the output value before enabling the output driver. Otherwise a transition from an unknown/reset value to the intended value can create an unintended pulse.

For an active-low reset output, for example, preload the inactive-high value before setting output enable—provided the board's pull and pad-reset behavior also guarantee a safe state.

### 5.6 Interrupt detection

Common per-pin interrupt modes:
- rising edge,
- falling edge,
- both edges (if supported),
- level high,
- level low.

A common software-visible register model includes:
- interrupt enable,
- interrupt state/pending,
- interrupt type or polarity,
- interrupt test,
- optional wakeup enable.

A write-one-to-clear (W1C) pending bit is common but not universal. Check the actual register specification.

**Edge capture:** a pulse shorter than the sampling clock/filter requirements may be missed unless the hardware has an asynchronous latch or pulse-stretch mechanism.

**Level interrupt:** the source must be removed or masked/cleared according to the controller design; otherwise the request may reassert immediately.

### 5.7 Hardware straps

Some SoCs sample selected pad values during a defined reset/boot window to determine boot mode, voltage options, security configuration or board identity.

Strap logic needs:
- a defined sampling event,
- a documented setup/hold interval around sampling,
- deterministic pull states,
- a retained snapshot register,
- a clear rule for later pin reuse.

Do not assume a GPIO data register read after boot returns the original strap sample. The pad may have been remuxed or the external source may have changed. Use the dedicated strap snapshot if provided.

---

## 6. Register map and programming model

There is no universal GPIO register map. The following is a **generic teaching example**, not an address map to copy into a real driver.

| Offset | Example register | Purpose |
|---:|---|---|
| 0x00 | DATA_IN | Synchronized input state; typically read-only |
| 0x04 | DATA_OUT | Output value latch |
| 0x08 | DATA_OE | Output enable / direction |
| 0x0C | INTR_ENABLE | Per-pin interrupt enable |
| 0x10 | INTR_STATE | Pending status; may be W1C |
| 0x14 | INTR_TYPE | Edge/level selection |
| 0x18 | INTR_POLARITY | Rising/high versus falling/low selection |
| 0x1C | INTR_BOTH_EDGE | Optional both-edge selection |
| 0x20 | INPUT_FILTER_EN | Optional digital filter enable |
| 0x24 | INPUT_FILTER_CYCLES | Optional filter threshold |
| 0x28 | OUT_SET | Atomic set bits, if implemented |
| 0x2C | OUT_CLEAR | Atomic clear bits, if implemented |
| 0x30 | OUT_TOGGLE | Atomic toggle bits, if implemented |

### 6.1 Why atomic set/clear matters

A read-modify-write sequence can lose updates if multiple software contexts manipulate different bits of the same register.

Example:
1. Thread A reads DATA_OUT = 0000.
2. Thread B reads DATA_OUT = 0000.
3. Thread A sets bit 0 and writes 0001.
4. Thread B sets bit 1 based on its stale read and writes 0010.
5. Bit 0 is lost.

Hardware set/clear aliases or masked-write registers avoid this class of software race. OpenTitan documents direct and masked GPIO output and output-enable views; its masked access encodes the mask and data in one write. This is a documented implementation example, not a universal GPIO standard. [OpenTitan GPIO theory](https://opentitan.org/book/hw/ip/gpio/doc/theory_of_operation.html)

### 6.2 Generic output initialization sequence

~~~text
1. Confirm the pin is available and owned by the intended device.
2. Select the GPIO function in the pinmux.
3. Configure pad voltage/bias/drive/slew/input enable as required.
4. Write the safe output value to DATA_OUT.
5. Set DATA_OE for output.
6. Read DATA_IN if physical feedback is required.
7. Confirm the board-level voltage with a scope or logic analyzer.
~~~

The exact ordering of mux, pad and output-enable writes is hardware-specific. Some SoCs provide shadow registers or atomic update mechanisms; others require a carefully designed transition sequence.

### 6.3 Generic input initialization sequence

~~~text
1. Select GPIO input function.
2. Disable output driver.
3. Configure pull-up/down or external bias.
4. Enable input receiver.
5. Configure synchronization/filter behavior.
6. Read DATA_IN and verify the idle level.
7. Configure edge/level interrupt type.
8. Clear stale pending state using the documented semantics.
9. Enable the GPIO interrupt and then the parent interrupt route.
~~~

### 6.4 Interrupt clearing sequence

For a W1C pending register, software clears a bit by writing one to that bit. Do not use a read-modify-write unless the register definition explicitly permits it: writing back other observed ones can clear unrelated pending events.

If an edge arrives concurrently with the W1C write, behavior depends on the hardware's set-versus-clear priority. This must be specified and verified.

---

## 7. Pinmux and pad configuration

### 7.1 Pinmux function selection

A mux can be represented conceptually as:

~~~text
PAD_12 output source select:
  000 = GPIO12
  001 = UART0_TX
  010 = SPI1_MOSI
  011 = I2S_BCLK
  100 = PWM3
  101 = JTAG_TCK
~~~

The encoding is purely illustrative. Actual mux tables are vendor- and package-specific.

A pinmux may provide:
- per-pin selection,
- group selection,
- separate input and output selection,
- daisy-chain input selection,
- dedicated pins that bypass the mux,
- retention or wakeup routing.

### 7.2 Input daisy chains

Some SoCs allow a peripheral input to be selected from more than one physical pad. A daisy-chain register selects which pad drives a particular internal input.

For example, UART0_RX may be available on either PAD_A or PAD_B. The output TX mux and RX input daisy selection can be separate. A correct TX mux does not guarantee that RX is routed correctly.

### 7.3 Pad configuration fields

Typical attributes:
- pull-up enable,
- pull-down enable,
- pull strength or resistance code,
- drive-strength code,
- slew-rate code,
- Schmitt-trigger enable,
- input receiver enable,
- open-drain enable,
- keeper/bus-hold,
- output disable / Hi-Z,
- voltage-bank selection (sometimes fixed by board rail rather than per-pin register).

Mutually incompatible settings must be rejected or resolved by the hardware/software contract. For example, pull-up and pull-down may be mutually exclusive. A function may require input enabled even when output is selected.

### 7.4 GPIO versus pinctrl in Linux

The Linux GPIO subsystem models GPIO lines and their direction/value/IRQ operations. The pinctrl subsystem models pins, mux functions/groups and electrical configuration. The two can be linked by GPIO ranges or firmware descriptions, but they are not the same API.

The Linux pinctrl documentation describes:
- pin enumeration and names,
- pin groups and functions,
- pinmux ownership,
- pin configuration such as bias and drive,
- standard states named default, init, sleep and idle,
- interaction with GPIO ranges.

Useful sources:
- [Linux pin control subsystem](https://www.kernel.org/doc/html/latest/driver-api/pin-control.html)
- [Linux GPIO driver interface](https://www.kernel.org/doc/html/latest/driver-api/gpio/driver.html)
- [Linux GPIO consumer interface](https://www.kernel.org/doc/html/latest/driver-api/gpio/consumer.html)

### 7.5 Device-tree state example

The syntax below is an illustrative Linux Device Tree pattern. Pin names, function names, group names and pin configuration properties are controller-specific.

~~~dts
&i2c0 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&i2c0_default_pins>;
    pinctrl-1 = <&i2c0_sleep_pins>;
    status = "okay";
};

&pinctrl {
    i2c0_default_pins: i2c0-default {
        /* SoC-specific pin group and bias properties */
    };

    i2c0_sleep_pins: i2c0-sleep {
        /* Safe sleep mux and bias configuration */
    };
};
~~~

A device core may select the default state before probing and sleep/idle states during power management, according to the platform and driver setup. A driver should not independently seize pins that the platform's pinctrl state already owns without a clear reason.

### 7.6 Pin conflict example

Suppose SPI1 uses pads 10–13 and UART0 uses pads 12–13. They cannot both own the overlapping pads at the same time. The pinctrl framework can prevent conflicts where the driver and mapping describe the groups accurately. Firmware must still choose a valid board configuration.

### 7.7 Safe runtime mux transition

A robust conceptual transition from GPIO output to SPI is:

1. Quiesce the GPIO client and stop toggling.
2. Ensure the external device is in a safe state.
3. Disable or tristate the old output as required.
4. Program pad bias and electrical attributes for SPI.
5. Program the SPI output/input muxes.
6. Configure the SPI controller and clock polarity/phase.
7. Enable the SPI peripheral output drivers.
8. Verify idle levels and the first transaction.

The actual sequence depends on whether mux and pad registers are shadowed, whether the peripheral drives immediately after mux selection, and whether the pad has a glitch-free update mechanism. A generic sequence must not be mistaken for a universal hardware procedure.

---

## 8. Timing and electrical calculations

### 8.1 Open-drain pull-up rise time

For a first-order RC model, the time to rise from 30% to 70% of the final voltage is approximately:

\[
t_r \approx 0.8473 R_P C_B
\]

Example assumptions:
- pull-up resistance \(R_P = 4.7\,k\Omega\),
- total bus capacitance \(C_B = 100\,pF\).

\[
t_r \approx 0.8473 \times 4700 \times 100 \times 10^{-12}
\approx 398\,ns
\]

This is a first-order estimate. Real rise time depends on nonlinear pad capacitance, resistor tolerance, leakage, probe loading, board routing and receiver threshold definitions. For I2C, compare the measured result with the selected I2C mode's timing limits; do not apply an arbitrary GPIO timing target.

### 8.2 Pull-up sink current

At a low level, approximate current through the pull-up is:

\[
I_{LOW} \approx \frac{V_{PU}-V_{OL}}{R_P}
\]

For \(V_{PU}=3.3V\), \(V_{OL}=0.4V\), and \(R_P=4.7k\Omega\):

\[
I_{LOW} \approx \frac{2.9}{4700} \approx 0.617\,mA
\]

Check that the device can sink this current while meeting its specified VOL. Also check the power-up and unpowered-device conditions.

### 8.3 Debounce interval

For a sampled digital filter requiring N consecutive stable samples:

\[
t_{QUAL} = \frac{N}{f_{SAMPLE}}
\]

For N = 320 and \(f=32\,kHz\):

\[
t_{QUAL}=10\,ms
\]

If the input clock changes under DVFS, the debounce interval changes unless the filter uses a stable clock or the count is adjusted. This is a common low-power/clock-management integration trap.

### 8.4 Synchronizer latency

A two-flop synchronizer typically adds about two destination clock stages before the synchronized value is observed, with phase-dependent sampling uncertainty. At 50 MHz, one clock period is 20 ns. A transition arriving just before or just after a sampling edge can produce different visible latency.

Do not call this a fixed two-cycle end-to-end pin latency: pad delay, input routing, metastability resolution, optional filtering and register-read timing all add to it.

### 8.5 Output RC and load

A rough lumped model uses:

\[
t_{10-90} \approx 2.2 R_{DRV} C_{LOAD}
\]

This is only a first-order approximation. A real CMOS output is nonlinear and its effective resistance changes with voltage, drive-strength code, PVT and transition direction. Use IBIS models and board simulation for high-speed or heavily loaded signals.

### 8.6 GPIO bandwidth is not just clock frequency

A GPIO register clocked at 100 MHz does not mean an external pin can toggle at 100 MHz. Limits include:
- bus write latency,
- register-to-pad delay,
- pad rise/fall time,
- output load,
- pinmux path,
- software access latency,
- synchronizer and input filter,
- board signal integrity,
- maximum toggle rate in the pad specification.

For high-rate waveforms use a timer/PWM, SPI, I2S, dedicated serializer or DMA-capable peripheral rather than bit-banging GPIO.

---

## 9. Clock, reset, power and wakeup dependencies

### 9.1 Clocking

GPIO may have separate clocks for:
- APB/register access,
- input sampling and filtering,
- always-on wakeup detection.

If the register clock is gated while the pad input remains active, wakeup logic may need an always-on clock or asynchronous event latch. Confirm the clock-domain architecture.

### 9.2 Reset defaults

Reset behavior must define:
- DATA_OUT reset value,
- DATA_OE reset value,
- pinmux reset function,
- pad pull and drive defaults,
- input receiver default,
- interrupt enable/pending state,
- strap sampling window,
- retention behavior.

A safe design usually keeps externally visible outputs in a board-safe state during reset. The exact state may be achieved through output disable, a pad pull, a board resistor or a dedicated reset cell.

### 9.3 Power domains

A GPIO controller may be in a switchable domain while selected wakeup pads are in an always-on domain. Consider:
- what happens to output drivers when VDDIO remains on but core power is off,
- whether pad configuration is retained,
- whether a wakeup detector can observe a pin while the main GPIO clock is stopped,
- how level shifters and isolation are controlled,
- whether external devices can back-power the SoC through an I/O pin.

### 9.4 Wakeup input

A wakeup pin may use a separate always-on synchronizer and event latch. If the wakeup pulse is shorter than the sampling period, it can be missed unless the pad or wakeup block provides asynchronous capture or pulse stretching. The wakeup path needs its own minimum pulse-width specification.

---

## 10. Integration dependencies and ownership

A GPIO/pinctrl integration is not complete until all of these agree:

| Integration artifact | Must define |
|---|---|
| SoC pin list | Pad names, package balls, bonded/unbonded pads |
| Pinmux table | Supported functions, groups, input daisy routes |
| Pad library | Voltage, drive, pull, slew, thresholds, leakage and ESD behavior |
| GPIO RTL | Number of lines, registers, interrupts, filters and reset values |
| Address map | Bus base, offsets, access permissions and security attribution |
| Interrupt map | GPIO interrupt output to GIC/PLIC source |
| Clock/reset | Register and sampling clocks, reset source and wakeup clock |
| Power intent | VDDIO/core rails, retention, isolation and power-off tolerance |
| Firmware | Boot straps, safe defaults, pin ownership and runtime states |
| Device tree/ACPI | Pin groups, functions, states and GPIO mappings |
| Board schematic | External pull resistors, level shifters, loads and connector wiring |
| Validation plan | Test points, voltage limits, test patterns and PVT coverage |

### 10.1 Security and safety

Not all GPIOs are ordinary user-accessible pins. Some may control:
- reset of external components,
- power enables,
- boot-mode straps,
- secure lifecycle signals,
- debug enable,
- tamper detection,
- safety interlocks.

Apply bus firewalls, privilege controls, lifecycle restrictions and secure ownership where required. A GPIO output that can reset or power-cycle another device is a control-plane security boundary.

### 10.2 Addressing and software ownership

GPIO line numbers are often local to a controller or chip and should not be assumed to equal package pad numbers. Pinctrl pin numbers are local to a pin controller. Device-tree aliases, GPIO offsets, package ball names and internal mux selectors are different namespaces.

Maintain an explicit mapping table rather than relying on arithmetic assumptions.

---

## 11. Common RTL, integration and board bugs

| Symptom | Likely cause | How to isolate |
|---|---|---|
| Output register reads 1 but pad stays low | Pinmux not selecting GPIO, OE disabled, pad forced low, short or external load | Read mux/OE; probe pad; disconnect board load if safe |
| GPIO output works but UART TX does not | Wrong alternate-function mux or pad configuration | Read mux and pinctrl state; probe internal peripheral output and pad |
| Input is inverted | Active-low board signal or polarity configuration | Check schematic, idle voltage and polarity |
| Interrupt storms | Level source not cleared, wrong polarity, W1C misuse, bounce | Capture raw input, pending state and parent IRQ status |
| Short pulse is missed | Synchronizer/filter sampling limitation | Sweep pulse width and phase against GPIO clock |
| Button produces many events | No debounce or unsuitable filter | Scope raw contact and filtered signal |
| I2C rise time too slow | Pull-up too weak or bus capacitance too high | Measure 30–70% rise; calculate RC; inspect probe capacitance |
| Two devices fight on a pin | Mux ownership conflict or push-pull contention | Check pad voltage/current and mux registers |
| Pin glitches during function switch | Unsafe mux/OE ordering or no atomic update | Trigger scope on mux transition; inspect control writes |
| Pin works after warm reset only | Reset defaults or strap sampling sequence incorrect | Compare POR and warm-reset register/pad states |
| Wakeup not detected in sleep | Wakeup path clock stopped or pulse too short | Probe AON clock, event latch and minimum pulse |
| One board variant fails | Pull, voltage bank, package bonding or pin assignment differs | Compare BOM, schematic, pin table and board revision |
| High-speed clock has ringing | Excessive drive/slew, poor termination or routing | Scope with suitable probe; inspect IBIS/SI and layout |
| Readback differs from output latch | DATA_IN reflects actual pad, not the output latch | Compare DATA_OUT, DATA_OE and physical voltage |

### 11.1 Register readback versus pad readback

A common debug mistake is to read DATA_OUT and assume that the external pin is high. DATA_OUT is often just the output latch. DATA_IN usually reflects the synchronized input receiver. A mux disconnect, disabled output driver, external short or voltage-domain issue can make these values differ.

### 11.2 W1C race

A pending bit may be set by a new event in the same cycle software clears it. The IP must define set/clear priority. Verification should test simultaneous set and clear, not only isolated cases.

### 11.3 Reset and boot strap race

If firmware changes mux/pull configuration before strap sampling completes, the strap may be sampled incorrectly. Hardware should provide a defined sample event and firmware must respect the sequencing contract.

---

## 12. Pre-silicon verification strategy

### 12.1 Directed tests

1. Reset values of all GPIO and pinmux registers.
2. Drive each output high and low with OE enabled.
3. Verify Hi-Z behavior with OE disabled.
4. Verify input sampling for every pin.
5. Verify each alternate mux function.
6. Verify pull-up/down and input-enable programming.
7. Verify edge and level interrupt modes.
8. Verify W1C behavior and simultaneous event/clear.
9. Verify filters for pulses shorter than, equal to and longer than the threshold.
10. Verify reset during input activity and output drive.
11. Verify wakeup while the main GPIO clock is gated.
12. Verify pinmux conflict and illegal group handling.

### 12.2 Assertions — conceptual SystemVerilog

These are verification examples. Signal names, reset polarity and semantics must be adapted to the actual RTL.

~~~systemverilog
// An output should not be enabled when the pin is not owned by GPIO.
// This property is only valid if the design contract requires that rule.
property gpio_oe_requires_gpio_mux;
  @(posedge clk) disable iff (!rst_n)
    gpio_oe[i] |-> (pad_mux_sel[i] == MUX_GPIO);
endproperty
assert property (gpio_oe_requires_gpio_mux);

// A disabled output driver should present high impedance at the pad model.
// Use a resolved net or a pad behavioral model in the testbench.
property disabled_driver_is_z;
  @(posedge clk) disable iff (!rst_n)
    !gpio_oe[i] |-> (pad_drive_en[i] == 1'b0);
endproperty
assert property (disabled_driver_is_z);

// A filtered input must not change until the required stable-sample count.
// This is pseudocode: implement the counter/reference model for the DUT.
~~~

### 12.3 Functional coverage

Cover:
- every pin,
- every mux function and group,
- input/output/bidirectional mode,
- all pad configurations,
- rising/falling/both-edge/level interrupt modes,
- filter enabled/disabled and boundary pulse widths,
- POR/warm reset/low-power wakeup,
- GPIO-to-peripheral and peripheral-to-GPIO transitions,
- legal and illegal ownership combinations,
- voltage-bank and package variants at the integration level.

### 12.4 Formal properties

Useful formal targets:
- no two exclusive functions own the same pad simultaneously,
- output enable is not asserted in forbidden power/reset states,
- W1C pending bits follow specified set/clear priority,
- interrupt state cannot become X after reset release,
- mux selection is always within the legal encoding set,
- strap snapshot changes only in the permitted sampling window,
- output data and OE update atomically if the architecture promises atomicity.

### 12.5 CDC and RDC

The pad input is an asynchronous source. Check:
- synchronizer recognition and placement,
- no combinational bypass around the synchronizer,
- filter clock and reset domain,
- wakeup capture crossing,
- reset assertion/deassertion behavior,
- mux switching while the input synchronizer is active.

Do not blindly false-path all pad inputs. The asynchronous path should be constrained and reviewed according to the CDC methodology; the synchronizer stages need correct placement and timing treatment.

---

## 13. Post-silicon bring-up and characterization

### 13.1 First GPIO smoke test

1. Read chip ID and GPIO controller reset registers.
2. Confirm the target pad is bonded on this package.
3. Check board voltage rail and external pull.
4. Configure a known-safe GPIO output.
5. Toggle slowly at 1 Hz.
6. Observe the package pin with an oscilloscope.
7. Test input mode using a known external stimulus.
8. Read DATA_IN and compare against measured voltage.
9. Test interrupts with controlled edge/level stimuli.
10. Repeat at voltage and temperature corners.

### 13.2 Pinmux validation matrix

For each critical signal, record:

| Item | Example |
|---|---|
| Package pad | PAD_A12 |
| Board net | CODEC_RESET_N |
| Selected function | GPIO |
| Direction | Output |
| Active polarity | Active low |
| Reset state | Hi-Z with external pull-up |
| Runtime state | Push-pull output |
| I/O supply | Board-defined VDDIO |
| Drive/slew | Pad-spec-approved setting |
| External load | Codec reset input plus trace |
| Test | POR, warm reset, DSP reset, sleep/wakeup |

### 13.3 Electrical characterization

Measure:
- VOH and VOL at specified load currents,
- input high/low thresholds,
- rise/fall times,
- output leakage in Hi-Z,
- pull-up/down resistance range,
- drive-strength code variation,
- slew-rate code variation,
- input hysteresis where specified,
- pad behavior when VDDIO or core supply is absent,
- crosstalk and ringing for fast edges.

Use the correct probe, ground spring and bandwidth. A long oscilloscope ground lead can create apparent ringing that is not present on the board.

### 13.4 Failure isolation order

When a pin fails, debug in this order:

1. Is the pad bonded and routed on this package?
2. Is the correct board rail present?
3. Is the external device driving or pulling the line?
4. Is the pad in reset, isolation or power-down?
5. Is the correct mux function selected?
6. Is the pad input/output enabled?
7. Is GPIO output data/OE correct?
8. Is the input synchronizer/filter configured?
9. Is the interrupt source enabled and routed?
10. Does the physical waveform meet electrical timing and voltage limits?

This order prevents spending hours debugging GPIO RTL when the actual issue is an unbonded pad, wrong board resistor or wrong mux setting.

---

## 14. Worked integration case — configure one pad as GPIO, I2C, SPI and I2S

Assume a hypothetical SoC exposes PAD_20–PAD_23 with the following alternate functions:

| Pad | GPIO function | Alternate function |
|---|---|---|
| PAD_20 | GPIO20 | I2C0_SCL / SPI0_SCK / I2S_BCLK |
| PAD_21 | GPIO21 | I2C0_SDA / SPI0_MOSI / I2S_DOUT |
| PAD_22 | GPIO22 | SPI0_MISO / I2S_DIN |
| PAD_23 | GPIO23 | SPI0_CS0 / I2S_LRCLK |

### Mode A — GPIO

- PAD_20 output high.
- PAD_21 output low.
- PAD_22 input.
- PAD_23 output disabled.

Verify pad voltage and DATA_IN separately.

### Mode B — I2C

- Select I2C0_SCL and I2C0_SDA.
- Configure both pads open-drain.
- Enable appropriate pull-ups (board resistor preferred when required by bus electrical/timing design).
- Ensure no push-pull high drive.
- Verify idle-high and low-to-release waveforms.

### Mode C — SPI

- Select SCK, MOSI, MISO and CS.
- Configure push-pull outputs for SCK/MOSI/CS and input for MISO.
- Set idle polarity through SPI controller configuration.
- Ensure CS is inactive before enabling clock output.

### Mode D — I2S

- Select BCLK, DOUT, DIN and LRCLK.
- Configure clock master/slave direction consistently with the codec.
- Set the pad drive/slew for the board load and target clock.
- Verify BCLK and LRCLK at the physical codec pins.

**Key lesson:** selecting a mux function does not program the peripheral protocol. The SPI controller still needs CPOL/CPHA and divider configuration; the I2S controller still needs sample rate, slot width and clock role; the I2C controller still needs open-drain-compatible pad behavior.

---

## 15. Hands-on labs

### Lab 1 — GPIO register model

Build a small APB GPIO model with 8 pins, DATA_IN, DATA_OUT, DATA_OE, interrupt enable and W1C pending state. Test read/write and reset behavior.

### Lab 2 — Pad behavioral model

Create a resolved bidirectional pad model supporting push-pull, open-drain and Hi-Z. Add an external pull-up and test contention.

### Lab 3 — Input synchronizer and filter

Add a two-flop synchronizer and programmable stable-sample filter. Sweep asynchronous input phase and pulse width. Measure detection latency.

### Lab 4 — Interrupt verification

Test rising, falling, both-edge and level modes. Inject an event in the same cycle as W1C clear and document the required priority behavior.

### Lab 5 — Pinmux conflict checker

Represent each function as a set of pads. Reject configurations that assign an exclusive pad to two active functions. Add package-specific unavailable pads.

### Lab 6 — Linux integration review

Read a real SoC pinctrl binding and trace:
- pin group,
- function,
- pin configuration,
- default/sleep state,
- GPIO range,
- consumer driver.

Use the Linux pinctrl and GPIO documentation listed below.

---

## 16. Interview questions

1. What is the difference between GPIO, pinmux and pad control?
2. Why does DATA_IN sometimes differ from DATA_OUT?
3. What happens electrically when a GPIO output enable is cleared?
4. Explain push-pull versus open-drain.
5. Why does open-drain require a pull-up?
6. Why can a strong pull-up create excessive sink current?
7. What is the difference between a synchronizer and a debounce filter?
8. Why can a short GPIO pulse be missed?
9. How does a Schmitt trigger help with noisy or slow edges?
10. Why should output data be preloaded before enabling output drive?
11. What is an input daisy-chain mux?
12. Why can SPI and UART pin groups conflict?
13. Why is a GPIO controller not necessarily the same IP as a pin controller?
14. What is a W1C register and why is read-modify-write dangerous for it?
15. How do GPIO interrupts work while the main peripheral clock is gated?
16. What is the difference between a GPIO line number and a package pin number?
17. How would you debug a pin that works in GPIO mode but not in I2C mode?
18. What electrical information is needed to choose a drive-strength setting?
19. How do you prevent a glitch during runtime pinmux switching?
20. What must be verified when a pad bank supports multiple I/O voltages?

---

## 17. Design-review checklist

### Architecture
- [ ] GPIO count and pin numbering are documented.
- [ ] Pinmux groups and legal combinations are documented.
- [ ] Input and output mux paths are understood.
- [ ] Dedicated and multiplexed pins are distinguished.
- [ ] Package bonding and board variants are checked.

### Electrical
- [ ] VDDIO range and power sequencing are defined.
- [ ] Input thresholds and fail-safe behavior are known.
- [ ] Pull-up/down resistance and tolerance are known.
- [ ] Drive and slew settings meet load and timing.
- [ ] Open-drain behavior and external pulls are correct.
- [ ] ESD, leakage and unpowered behavior are reviewed.

### RTL and registers
- [ ] Reset values are safe.
- [ ] Output latch and output enable are separate.
- [ ] Input synchronizers are present where required.
- [ ] Filter semantics and clock dependence are documented.
- [ ] W1C and simultaneous set/clear behavior are defined.
- [ ] Atomic set/clear/masked access is available where needed.

### Integration and software
- [ ] Address and interrupt maps are correct.
- [ ] Pinctrl states exist for default and sleep.
- [ ] GPIO-to-pin mappings are correct.
- [ ] Boot straps are sampled before pins are repurposed.
- [ ] Secure pins have appropriate access control.
- [ ] Reset and wakeup behavior is defined.

### Verification and silicon
- [ ] CDC/RDC checks are clean or waived with justification.
- [ ] Mux conflicts and illegal configurations are tested.
- [ ] Interrupt/filter boundary tests are covered.
- [ ] Board-level voltage and timing measurements are planned.
- [ ] POR, warm reset and sleep/wakeup are tested.

---

## 18. Reference specifications and implementation documentation

### Historical sources
1. [Intel 1978 Component Data Catalog](https://bitsavers.trailing-edge.com/components/intel/_dataBooks/1978_Intel_Component_Data_Catalog.pdf) — lists the 8255A programmable peripheral interface.
2. [Intel 1979 Peripheral Design Handbook](https://bitsavers.trailing-edge.com/components/intel/_dataBooks/1979_Intel_Peripheral_Design_Handbook.pdf) — 8255A functional description and bus interface.
3. [Cambridge University Press — 8255 PPI data sheets](https://doi.org/10.1017/CBO9780511622885.014) — historical context and 8255 operating modes.
4. [Linux kernel mailing list, 13 October 2011](https://lkml.iu.edu/1110.1/01966.html) — pin-control subsystem proposal and rationale.

### Linux software interfaces
5. [Linux pin control subsystem](https://www.kernel.org/doc/html/latest/driver-api/pin-control.html) — pins, groups, functions, pinconf, states and GPIO interaction.
6. [Linux GPIO driver interface](https://www.kernel.org/doc/html/latest/driver-api/gpio/driver.html) — gpio_chip, debounce, open-drain and pull configuration.
7. [Linux GPIO consumer interface](https://www.kernel.org/doc/html/latest/driver-api/gpio/consumer.html) — requesting and using GPIO lines.

### Public reference IP
8. [OpenTitan GPIO technical specification](https://opentitan.org/book/hw/ip/gpio/) — a documented 32-line GPIO controller example.
9. [OpenTitan GPIO theory of operation](https://opentitan.org/book/hw/ip/gpio/doc/theory_of_operation.html) — data paths, masked writes, input filtering, straps and interrupts.
10. [OpenTitan pinmux theory of operation](https://opentitan.org/book/hw/ip/pinmux/doc/theory_of_operation.html) — MIO/DIO, mux matrix, pad ring and strap routing.

### How to interpret these references
- **Normative/electrical requirements:** use the exact SoC datasheet, pad-library specification, board component datasheet and applicable electrical interface standard.
- **Software framework behavior:** Linux documentation defines Linux APIs and state conventions; it does not define a particular SoC's mux encoding.
- **Implementation example:** OpenTitan demonstrates one public architecture; its 32-bit register layout, 16-cycle filter and mux topology are not universal requirements.
- **Engineering interpretation:** the trade-offs and debug recommendations in this lesson are analysis, not claims about undocumented vendor design decisions.

---

## 19. Session summary

A GPIO is not merely a bit in a register. It is a complete path through software ownership, bus registers, output/input logic, mux selection, pad electrical circuitry, package, board and external device.

The essential mental model is:

~~~text
Software intent
   -> GPIO data / direction
   -> pinmux ownership
   -> pad electrical configuration
   -> I/O transistor and input receiver
   -> package and board
   -> external voltage and timing
~~~

For a reliable design, verify both directions:
- **Output:** register value -> mux -> output enable -> pad driver -> board voltage.
- **Input:** board voltage -> pad receiver -> synchronizer -> filter -> data/interrupt logic -> software.

**Next lesson:** Session 19 — I2C: open-drain signaling, START/STOP, ACK/NACK, arbitration, clock stretching, timing, controller state machines, bus recovery and silicon debug.
