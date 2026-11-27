# Lesson 21 — I3C: Dynamic Addressing, CCCs, IBI, HDR, RTL and Silicon Debug

## From an I²C-compatible two-wire bus to a discoverable, event-driven peripheral fabric

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 21 of 38  
**Target duration:** 60 minutes  
**Level:** protocol, electrical behavior, controller/target RTL, register programming, Linux integration, verification and post-silicon validation  
**Prerequisites:** GPIO/pinctrl and pad-ring behavior (Lesson 18), I²C (Lesson 19), SPI (Lesson 20), APB/AXI-Lite, interrupts, CDC, reset and power management

> **Scope and source discipline:** This lesson uses MIPI I3C v1.2 and I3C Basic v1.2 as the current-version baseline listed by MIPI as of November 28, 2026. The full I3C specification is member-access; I3C Basic has a public release. Public MIPI summaries, Linux documentation and vendor application notes are used for explanation. Exact timing limits, reserved-address rules, CCC behavior and optional feature requirements must be checked against the exact specification revision and target data sheet used by a project. Vendor register maps are implementation examples, not universal I3C registers.

---

## 0. One-hour learning plan

| Time | Topic | Engineering outcome |
|---:|---|---|
| 0–6 min | Why I3C exists; history and I²C limitations | Explain the design goals and trade-offs |
| 6–13 min | Bus roles, SDA/SCL and electrical modes | Draw a safe pure-I3C and mixed-bus connection |
| 13–23 min | SDR transactions, T-bit and CCCs | Decode controller writes, reads and bus-management commands |
| 23–32 min | Dynamic Address Assignment (DAA) | Walk through PID/BCR/DCR arbitration and address allocation |
| 32–39 min | IBI, Hot-Join and controller-role request | Explain event delivery without dedicated interrupt pins |
| 39–45 min | HDR modes and mixed I²C coexistence | Identify capability and compatibility limits |
| 45–52 min | Controller RTL, FIFOs, address tables and registers | Sketch a practical SoC controller |
| 52–60 min | Calculations, verification, silicon debug and labs | Build a bring-up and validation plan |

## 1. Why I3C exists

I²C solved a valuable system problem: connect many low-speed peripherals with two shared wires. Its simplicity and ecosystem made it ubiquitous. But modern systems added many sensors and management devices, and several limitations became more visible:

- Open-drain signaling with pull-up resistors limits edge rate as bus capacitance increases.
- The host commonly polls targets or uses separate interrupt GPIOs.
- Addresses are statically assigned, so board-level address conflicts can require straps, multiplexers or extra buses.
- Discovery and capability description are largely external to the basic I²C protocol.
- Sensor clusters can spend significant energy and host time moving small amounts of data using repeated polling.
- I²C and SPI often coexist in a design, creating separate pins and software paths.

MIPI I3C was designed as a two-wire, low-power control and utility bus that preserves a migration path for legacy I²C targets while adding dynamic addressing, standardized bus-management commands, in-band events and higher data rates. MIPI describes it as a successor to I²C that also incorporates useful capabilities associated with SPI. This does not mean I3C replaces SPI for every application: SPI remains attractive for simple point-to-point links, flash memory and devices whose command/data protocol is already SPI-based.

### 1.1 Historical milestones — documented evidence

| Date / revision | Documented milestone | Engineering significance |
|---|---|---|
| 2016 / Jan 2017 | MIPI's current page labels I3C v1.0 as 2016; a later MIPI press release says I3C v1.0 was released in January 2017 | Initial standardized successor path for sensor and peripheral control |
| 11 Dec 2018 | MIPI announced public I3C Basic v1.0 | Made a feature subset available to a broader implementer community |
| 2019 / 15 Jan 2020 | MIPI lists I3C v1.1 as 2019 and announced it in January 2020 | Continued the upgrade path for mobile, IoT, automotive and server manageability |
| June 2021 | I3C v1.1.1 and I3C Basic v1.1.1 | Clarifications and additional HDR / target-management capabilities |
| February 2025 | I3C v1.2 | Modularized specification and incorporated errata and clarifications |
| April 2025 | I3C Basic v1.2 | Current public Basic revision listed by MIPI |

MIPI states that I3C v1.2 does not introduce new features; it incorporates errata and clarifies dynamic-address management, target reset behavior, HDR CCC flows, timing, message lengths and error recovery. MIPI lists I3C Basic v1.2 as the public version available in April 2025.

**Documented design rationale:** MIPI positions I3C for sensor/actuator control, always-on imaging, memory sideband, server management, debug and power management. It emphasizes two-wire CMOS I/O, low power, dynamic addressing and I²C coexistence.

**Engineering interpretation:** the key architectural shift is not merely a faster serial clock. I3C moves part of what used to be board-specific policy—device discovery, capability exchange, event delivery and address assignment—into a defined bus protocol. That makes controller RTL and software more complex, but can simplify the overall system.

Sources: [MIPI I3C and I3C Basic overview](https://www.mipi.org/specifications/i3c-sensor-specification), [MIPI I3C Basic v1.0 announcement, 11 December 2018](https://www.mipi.org/press-releases/mipi-releases-i3c-basic-interface-specification), [MIPI I3C v1.1 announcement, 15 January 2020](https://www.mipi.org/press-releases/mipi-i3c-v1-1-strengthens-upgrade-path-for-i2c-implementers).

## 2. I²C versus I3C — the important architectural differences

| Property | Legacy I²C | I3C SDR |
|---|---|---|
| Wires | SDA, SCL | SDA, SCL |
| Typical electrical signaling | Open-drain, pull-up network | Open-drain for arbitration/control phases and push-pull for much of SDR data |
| Target address | Static 7-bit or 10-bit, depending on device | Dynamic 7-bit address for addressed I3C targets; static-address mechanisms also exist |
| Device discovery | Usually board/firmware description | Dynamic Address Assignment and capability registers |
| Data acknowledgement | ACK/NACK ninth bit after each byte | SDR uses a T-bit; controller-written data uses odd parity, target-read data uses end-of-data indication |
| Interrupt event | Usually separate GPIO/IRQ | In-Band Interrupt (IBI) can use the bus wires |
| Late device arrival | Board-specific handling | Hot-Join mechanism |
| Common command model | Device-specific register protocol | Standard Common Command Codes (CCCs) plus private transfers |
| Clock stretching | Supported by I²C protocol | I3C targets do not use I²C-style clock stretching during I3C transfers |
| Typical SDR clock | Depends on I²C mode and device | 12.5 MHz is a common maximum SDR clock target; check the selected bus mode and device |
| HDR | Not part of I²C | Optional High Data Rate modes, subject to revision and target capability |

Do not interpret “I²C backward compatible” as “every I²C device can coexist at full I3C speed.” Mixed-bus behavior depends on the legacy target's electrical filtering, timing limits, addressing and protocol behavior. Linux models pure, mixed-fast, mixed-limited and mixed-slow bus modes. A legacy target without the required spike filtering can constrain the bus.

MIPI describes typical I3C data rate around 11.1 Mbit/s, with optional higher-performance modes up to 100 Mbit/s. The 12.5 MHz figure is a common SDR SCL frequency, not a promise of 12.5 MB/s payload throughput.

## 3. Bus roles and physical interface

An I3C bus can contain:

- **Primary / active Controller:** currently owns the controller role and initiates ordinary transactions.
- **Secondary Controller:** capable of becoming active controller through the controller-role procedure, if implemented.
- **I3C Target:** addressed peripheral; can respond to controller transactions and may request permitted events.
- **Legacy I²C Target:** participates using I²C signaling and static address behavior.

Only one controller is active at a time. Multiple controller-capable devices do not mean simultaneous independent transfers.

### 3.1 Signals

| Signal / internal control | Purpose |
|---|---|
| SDA | Shared bidirectional serial data |
| SCL | Serial clock driven by the active controller |
| SDA input | Sample the bus level |
| SDA output value | Value driven during push-pull phases |
| SDA output enable | Select drive versus high impedance |
| SDA drive mode | Open-drain or push-pull control |
| SCL output | Controller-generated clock |
| SCL output enable | Safe controller ownership and reset control |
| Bus monitor | Detect START/STOP, arbitration, events and unexpected transitions |
| IBI request capture | Detect target event requests in the allowed bus opportunity |

I3C has no mandatory separate interrupt wire for IBI. A design may still have a dedicated GPIO interrupt for unrelated reasons, but the IBI itself is in-band.

### 3.2 Open-drain and push-pull

**Open-drain:** a participant actively drives LOW or releases the line. The pull-up (external or supported controller-side pull-up arrangement) creates HIGH. This allows multiple participants to arbitrate without a push-pull HIGH-versus-LOW short.

**Push-pull:** one authorized transmitter actively drives both LOW and HIGH. This improves edge control and reduces the RC rise-time penalty of a passive pull-up. It is safe only when the protocol guarantees that no other participant is simultaneously driving the line.

A controller RTL implementation must treat output-enable and drive-mode as first-class state, not as a late mux added after the serializer.

~~~text
                 SoC I3C Controller
             +--------------------------+
 SCL <------ | clock/timing generator   |
 SDA ------> | input sampler            |
 SDA <------ | open-drain / push-pull   |
             | output control           |
             | bus monitor / arbiter    |
             +--------------------------+
                      |
                 SDA / SCL
                      |
         +------------+-------------+
         |            |             |
      I3C target   I3C target   Legacy I2C
       sensor A     sensor B      target
~~~

**Board-level caution:** the pad cell, external pull-up network, target I/O voltage and any level translator must support the actual I3C electrical modes. A level shifter designed only for open-drain I²C can distort or block push-pull signaling. Validate the selected component's I3C suitability.

### 3.3 Mixed-bus modes

Linux's I3C bus model distinguishes:

- **Pure:** only I3C devices; no legacy I²C target constraints.
- **Mixed-fast:** I²C targets with a 50 ns spike filter; I3C clock-high pulse behavior is constrained so legacy targets do not misinterpret I3C activity.
- **Mixed-limited:** legacy targets without that filter but with compliance allowing the applicable SDR rate.
- **Mixed-slow:** legacy target characteristics constrain operation to a slower compatible mode.

The exact mode must be chosen from the target data sheets and current specification. Do not assume that adding an I²C device to a working pure-I3C bus is a software-only change.

## 4. SDR transaction model

I3C Single Data Rate (SDR) supports private transfers, CCC operations and event-related sequences.

### 4.1 Private write — conceptual sequence

~~~text
Controller: START
Controller: 7-bit Dynamic Address + W
Target:     ACK
Controller: DATA[7:0]
Controller: T-bit (odd parity for controller-written data)
Controller: DATA[7:0]
Controller: T-bit
...
Controller: STOP or Repeated START
~~~

The T-bit is not an I²C ACK/NACK bit.

For controller-written SDR data, the ninth bit is an odd-parity bit. For target-read data, the ninth bit is used as an end-of-data / continuation indication. The exact transitions and ownership of SDA are specified by the I3C SDR timing rules.

### 4.2 Private read — conceptual sequence

~~~text
Controller: START
Controller: Dynamic Address + R
Target:     ACK
Target:     DATA[7:0]
Target:     T-bit = 1  (more data follows)
Target:     DATA[7:0]
Target:     T-bit = 0  (end of data)
Controller: STOP or Repeated START
~~~

The read T-bit allows the target to indicate whether more data follows. This differs from I²C, where the controller terminates a read by NACKing the final byte.

### 4.3 Combined register read

A common peripheral access pattern is:

1. Controller addresses the target for write.
2. Controller sends the register index.
3. Controller issues a repeated START.
4. Controller addresses the same target for read.
5. Target returns the requested bytes and read T-bit indications.
6. Controller terminates the message.

The exact address-header and T-bit transitions must follow the I3C SDR state machine. Do not reuse an I²C byte engine unchanged and assume that replacing the clock rate makes it I3C-compliant.

### 4.4 T-bit details

- Controller-written data: T-bit is odd parity over the preceding 8-bit data word.
- Target-returned data: T-bit indicates continuation or end-of-data.
- Address header response and data T-bit are different protocol phases.
- A write data T-bit is not a per-byte target acknowledgement. A target that cannot accept more data must be handled using the defined transfer-length/flow-control mechanisms and target capability rules.

MIPI Basic v1.2 describes the ninth bit of controller-written SDR data as odd parity. ST's AN5879 gives examples and explains the read end-of-data use.

References: [ST AN5879 — I3C bus format](https://www.st.com/content/st_com/en/technical-documents/an5879-introduction-to-i3c-for-stm32-mcus-stmicroelectronics_asset/chunk-003-topic-an5879-bus-format.html), [Microchip I3C T-bit explanation](https://onlinedocs.microchip.com/oxy/GUID-20DC4AC0-78C3-43D4-B405-15D8B752B254-en-US-5/GUID-5C242700-C5A5-4818-BAD4-58917FEEBF5D.html).

## 5. Common Command Codes (CCCs)

CCCs standardize bus-management and device-capability operations. They are not the same as a target's private register protocol.

Linux documentation describes the CCC ID as an 8-bit command code. Its most significant bit distinguishes broadcast CCCs from direct/unicast CCCs:

- Broadcast CCC: intended for all applicable I3C targets.
- Direct CCC: addressed to a specific target.
- Some CCCs carry controller-written payload.
- Some direct CCCs return target data.
- Payload size/meaning is defined by the CCC, not by an arbitrary length field inserted by a generic serializer.

### 5.1 Useful CCC examples

| CCC name | Purpose |
|---|---|
| ENEC | Enable selected target events |
| DISEC | Disable selected target events |
| RSTDAA | Reset dynamic address assignments |
| ENTDAA | Enter Dynamic Address Assignment |
| SETDASA | Set a dynamic address from a target's static address |
| SETNEWDA | Change a target's dynamic address |
| GETPID | Read Provisioned ID |
| GETBCR | Read Bus Characteristics Register |
| GETDCR | Read Device Characteristics Register |
| GETSTATUS | Read target status, where supported |
| ENTHDRx | Enter a supported HDR mode |
| SETMRL / GETMRL | Set / get maximum read length |
| SETMWL / GETMWL | Set / get maximum write length |

The supported CCC set is capability- and revision-dependent. A controller must not assume every target supports every optional command.

### 5.2 Broadcast CCC flow

~~~text
START
  |
Broadcast address (7'h7E, write)
  |
ACK from applicable I3C targets
  |
CCC ID
  |
Optional defining byte / payload
  |
T-bit rules for SDR data
  |
STOP or Repeated START
~~~

### 5.3 Direct CCC flow

A direct CCC identifies one target and performs a standardized operation on that target. For example, GETPID can retrieve the target's Provisioned ID after it has a usable address.

The controller must keep CCC framing separate from private transfers. A common RTL mistake is to treat every command as “address + register + payload”; CCCs have their own standardized structure.

## 6. Dynamic Address Assignment (DAA)

DAA is one of the most important differences between I3C and I²C.

An I3C target needs a dynamic address before ordinary addressed private transfers. DAA discovers targets and assigns addresses. It also retrieves characteristics needed by the controller.

### 6.1 Device identity fields

| Field | Width | Meaning |
|---|---:|---|
| PID | 48 bits | Provisioned ID; identifies the target |
| BCR | 8 bits | Bus Characteristics Register; describes bus-related capabilities |
| DCR | 8 bits | Device Characteristics Register; describes device type/function category |

The concatenated identity/capability sequence is 64 bits. A duplicate PID on one bus can make discovery ambiguous; the Linux protocol overview explicitly notes that a PID collision can cause DAA failure.

### 6.2 ENTDAA sequence

The broadcast ENTDAA CCC uses command code 0x07. A simplified conceptual sequence is:

1. The active controller sends START and the I3C broadcast address 0x7E with write direction.
2. It sends ENTDAA CCC 0x07.
3. Unaddressed I3C targets that support the procedure participate.
4. Targets arbitrate by transmitting their concatenated PID, BCR and DCR fields using the defined open-drain arbitration behavior.
5. One target wins the arbitration round and completes its characteristics transmission.
6. The controller chooses and transmits a valid 7-bit dynamic address plus the required odd-parity bit.
7. The target checks the address parity and acknowledges a valid assignment.
8. The controller records PID/BCR/DCR and the assigned dynamic address.
9. The controller repeats the arbitration/assignment round for the remaining unaddressed targets.
10. The controller terminates DAA when no further targets participate or the configured procedure ends.

The target whose concatenated PID/BCR/DCR value wins arbitration is selected first under the documented wired-arbitration behavior. The controller chooses the dynamic address; the target does not simply choose any address it wants.

### 6.3 DAA state machine

~~~mermaid
stateDiagram-v2
    [*] --> BusIdle
    BusIdle --> SendENTDAA: broadcast CCC 0x07
    SendENTDAA --> TargetArbitration: eligible targets respond
    TargetArbitration --> CaptureIdentity: one winner remains
    CaptureIdentity --> SendDynamicAddress: PID/BCR/DCR captured
    SendDynamicAddress --> CheckAddressAck: address + parity sent
    CheckAddressAck --> RecordDevice: ACK
    RecordDevice --> TargetArbitration: repeat for remaining targets
    CheckAddressAck --> DAAError: NACK / parity failure
    TargetArbitration --> DAAComplete: no target responds
    DAAError --> BusRecovery
    DAAComplete --> BusIdle
~~~

This is an architectural model, not a replacement for the normative timing/state definitions.

### 6.4 Address allocation and reserved addresses

I3C dynamic addresses are 7-bit values, but not every possible 7-bit value is available. The specification reserves addresses for broadcast, events and other protocol functions; some address ranges are conditional on the legacy-I²C devices present.

Therefore:

- Maintain an address-slot table with FREE, RESERVED, I²C-occupied and I3C-occupied states.
- Exclude reserved addresses according to the exact spec revision.
- Avoid collisions with statically declared legacy I²C targets.
- Keep firmware-requested preferred addresses separate from guaranteed assignments.
- Rebuild the table after reset/re-discovery or controller handoff as required.

Do not hard-code a simplified contiguous address range as “all legal I3C addresses.”

### 6.5 Static address versus dynamic address

Some I3C targets also expose a static address. The controller can use mechanisms such as SETDASA where supported. A target without a usable static address can still participate in ENTDAA through its provisioned identity.

Linux's I3C core models both static and initial dynamic-address information. It also performs bus initialization steps including clearing stale dynamic addresses, disabling events, reserving addresses, applying configured static-address assignments and then performing DAA.

### 6.6 DAA hardware versus firmware assistance

Possible controller implementations include:

- Fully autonomous DAA engine with hardware-managed address table.
- Hardware captures PID/BCR/DCR and requests firmware to choose the dynamic address.
- Hybrid engine with programmable address allocation and software policy.

The MIPI protocol defines bus behavior; it does not mandate one internal microarchitecture. Linux's controller API notes that sending ENTDAA alone is not the whole procedure: the master driver must handle discovery and device instantiation.

## 7. In-Band Interrupts (IBI)

IBI allows an I3C target to request service over SDA rather than requiring a dedicated interrupt wire.

### 7.1 IBI lifecycle

1. The target has a dynamic address and advertises relevant capability in BCR.
2. The controller enables the applicable event using ENEC.
3. The target detects an event internally.
4. At the allowed bus-available opportunity, the target requests the bus.
5. The target and any other requesting devices participate in address arbitration.
6. The controller recognizes the winning target address and accepts or rejects the request according to protocol and controller policy.
7. If accepted, the target can send its Mandatory Data Byte (MDB) when its capability indicates that an MDB is present.
8. Optional payload may follow according to the target contract and limits.
9. The controller stores the event and notifies software.
10. The target returns to normal operation.

### 7.2 IBI arbitration and priority

If multiple targets request an IBI concurrently, the address arbitration mechanism resolves the request. Linux's I3C protocol overview describes the lower dynamic address as the higher-priority request in this arbitration context.

This means address allocation can affect event priority. Firmware should not treat address allocation as merely cosmetic.

### 7.3 Mandatory Data Byte

The MDB provides event information. In the documented I3C Basic v1.1.1 definition, the byte includes:
- MDB[7:5]: interrupt group identifier.
- MDB[4:0]: specific interrupt identifier.

Whether a target sends an MDB is indicated by its BCR capability. Do not assume all targets send an MDB or that every target's optional bytes share the same meaning.

### 7.4 ENEC and DISEC

- ENEC enables selected event types.
- DISEC disables selected event types.
- Events can be enabled/disabled for an individual target or broadcast where the CCC supports that form.
- The controller must coordinate the target's event enable state with its own IBI interrupt mask and receive resources.

### 7.5 IBI versus a GPIO interrupt

| Dedicated GPIO interrupt | I3C IBI |
|---|---|
| Extra package/board signal | Uses SDA/SCL |
| External pinmux and interrupt routing | Bus controller captures event |
| Can signal while bus is otherwise unavailable | Must follow I3C bus availability and arbitration rules |
| Simple direct interrupt path | Requires IBI enable, acceptance, payload capture and bus arbitration |
| Independent of I3C address | Uses dynamic address for identification and arbitration |

IBI reduces pins but adds bus arbitration and controller complexity. It is not automatically lower latency than a dedicated interrupt pin in every system.

## 8. Hot-Join and Controller Role Request

### 8.1 Hot-Join

Hot-Join allows a target that becomes available after initial bus setup to notify the active controller that it is waiting for address assignment.

Examples:
- A sensor is powered only after the main system has booted.
- A modular board is attached after the controller initialized the bus.
- A target exits a power state and needs to be discovered.

A Hot-Join request uses a reserved protocol address and is recognized as a special event, not as a normal private transfer. The controller can then initiate a new DAA procedure. The target must follow the required bus-mode and event-detection conditions.

Hot-Join is not the same as IBI:
- IBI: an already-addressed target reports an event.
- Hot-Join: a target without a current dynamic address requests to join the initialized bus.

### 8.2 Controller Role Request (CRR)

I3C can support multiple controller-capable devices, but only one is active at a time. A secondary controller can request the controller role using the defined procedure. The active controller can accept the request and perform a role handoff using the relevant CCC flow, or can manage the role according to system policy.

Do not confuse this with ordinary I²C multi-controller arbitration. The I3C controller-role protocol includes bus-management state and device capability requirements.

## 9. HDR — High Data Rate modes

HDR is optional and target/controller capability-dependent. It is not simply “turn SCL faster.”

### 9.1 Mode families and version discipline

The full I3C specification and I3C Basic subsets do not necessarily expose identical HDR mode sets.

- The Linux protocol overview describes HDR-DDR, HDR-TSP and HDR-TSL for the I3C protocol.
- MIPI's I3C Basic v1.1.1 summary describes HDR-DDR and HDR-BT, plus use of additional bus lanes for higher performance.
- The current MIPI v1.2 pages should be consulted for the exact current capability set and clarifications.

Do not mix HDR-TSP/HDR-TSL terminology from full I3C documentation with HDR-BT terminology from I3C Basic and assume they are interchangeable.

### 9.2 HDR entry and exit

The controller first configures the bus in SDR. It then uses the appropriate broadcast Enter HDR CCC (ENTHDRx) to move the bus into the selected HDR mode. The bus-wide mode affects all attached targets; only capable targets participate in the selected mode. The controller uses the mode's command framing and exit mechanism to return to SDR.

An HDR transaction is mode-specific. For example, Linux documentation describes HDR command/data words as 16-bit big-endian words with mode-dependent preambles and postambles. This is not a universal packet layout for every HDR mode.

### 9.3 HDR-DDR

HDR-DDR transfers data on both clock edges, increasing data per clock relative to SDR. The mode has defined command words, data words and parity/termination rules. A controller must implement the exact framing, turnaround and error behavior from the selected specification revision.

### 9.4 Performance trade-off

HDR can improve payload throughput but increases:
- Controller and target RTL complexity.
- Timing closure difficulty.
- Verification state space.
- Mode-transition and recovery requirements.
- Capability negotiation requirements.

For short register accesses, DAA, CCC and START/STOP overhead may dominate. SDR can be more efficient for small control messages even when HDR is available.

## 10. Controller and target block architecture

~~~mermaid
flowchart TB
    CPU[CPU / Firmware] --> BUS[APB or AXI-Lite]
    BUS --> REG[Register and command interface]
    REG --> CMD[Command / CCC sequencer]
    REG --> DAT[Dynamic Address Table]
    REG --> TX[TX FIFO]
    RX[RX FIFO] --> REG
    CMD --> DAA[DAA engine and PID arbiter]
    CMD --> SDR[SDR serializer / deserializer]
    CMD --> HDR[Optional HDR engine]
    IBI[IBI / Hot-Join / CRR detector] --> EVT[Event capture and queue]
    EVT --> IRQ[Interrupt generation]
    DAA --> PAD[Pad / bus electrical control]
    SDR --> PAD
    HDR --> PAD
    PAD <--> BUSPINS[SDA / SCL]
~~~

### 10.1 Major sub-blocks

| Sub-block | Responsibility |
|---|---|
| Register interface | Configuration, command submission and status |
| Command sequencer | Private transfers and CCC sequencing |
| SDR engine | Address, data, T-bit, START/STOP and repeated START |
| DAA engine | Identity arbitration, address assignment and table update |
| Address table | Dynamic address, PID, BCR, DCR and slot status |
| IBI engine | Request detect, arbitration, accept/reject, MDB/payload |
| Hot-Join detector | Capture late target request |
| Controller-role engine | Secondary-controller request/handoff, if supported |
| HDR engine | Optional mode entry, command framing, data and exit |
| TX/RX FIFOs | Buffer payload and decouple bus timing from system bus |
| Bus monitor | Detect illegal transitions, contention and timeout conditions |
| Timing generator | Open-drain and push-pull timing |
| Interrupt logic | Completion, error, IBI, Hot-Join, DAA and FIFO events |

### 10.2 Target-side architecture

A target typically needs:
- SDA input and output-enable control.
- Protocol state machine.
- Dynamic/static address storage.
- PID/BCR/DCR registers.
- CCC decoder.
- Private register or data interface.
- TX/RX buffering.
- IBI request logic and optional MDB generation.
- Optional Hot-Join and HDR logic.
- Reset and power-state recovery.

The target's application interface might connect to a sensor register bank, a PMIC management block, a codec control plane or a local microcontroller.

## 11. Register model and programming sequence

There is no single universal MMIO register map mandated for all I3C controllers. MIPI I3C HCI is a separate standardized host-controller interface; vendor controllers may expose different register maps.

### 11.1 Illustrative controller register groups

| Register group | Typical purpose |
|---|---|
| CONTROL | Enable, controller role, bus mode, reset |
| STATUS | Bus busy, command active, controller role, error state |
| TIMING_OD | Open-drain timing parameters |
| TIMING_PP | Push-pull timing parameters |
| DEVICE_ADDR_TABLE | Dynamic/static address, PID, BCR, DCR and capability |
| COMMAND_PORT | Private transfer or CCC descriptor submission |
| TX_DATA | Payload FIFO write port |
| RX_DATA | Payload FIFO read port |
| RESPONSE_QUEUE | Completion, NACK, parity, timeout and DAA result |
| IBI_STATUS | Pending IBI, source address and payload status |
| EVENT_ENABLE | ENEC/DISEC-related event state and controller masks |
| INTR_STATE / ENABLE | Completion, error, IBI, Hot-Join and FIFO interrupts |
| RESET / RECOVERY | Controller reset and bus recovery controls |

This table is an architectural checklist, not a normative register map.

### 11.2 Example bus initialization sequence

1. Enable the controller register clock and release reset.
2. Configure SDA/SCL pinmux and pad electrical behavior.
3. Confirm I/O voltage and bus pull-up/drive configuration.
4. Read controller capabilities: SDR/HDR, IBI, Hot-Join, DAA, multi-controller, FIFO sizes and DMA.
5. Program open-drain and push-pull timing.
6. Configure pure-I3C or mixed-bus mode using the board's actual legacy-target constraints.
7. Populate the static legacy-I²C target list and LVR/board information.
8. Clear stale dynamic addresses when required by system boot ownership.
9. Disable target events while bus maintenance is in progress.
10. Reserve addresses for known targets and any firmware-requested preferences.
11. Perform SETDASA for supported statically addressed targets if selected by policy.
12. Run ENTDAA for remaining unaddressed I3C targets.
13. Read or validate PID/BCR/DCR and build the device table.
14. Configure target maximum read/write lengths and supported capabilities as required.
15. Enable IBI with controller resources ready, then issue ENEC to the target.
16. Enable Hot-Join detection only after the controller can service it.
17. Register devices with the software framework.
18. Enable normal private transfers and interrupts.

**Important sequencing rule:** allocate the controller's IBI capture resources and unmask the relevant hardware path before enabling the target to send IBIs. Otherwise, an event can arrive before the controller has a place to store it.

### 11.3 Linux software model

Linux separates:
- I3C bus core and device descriptions.
- I3C master-controller driver operations.
- I3C target/device drivers.
- Legacy I²C device integration on the same bus.
- IBI payload slots and workqueue handling.

The master driver implements operations such as bus initialization, DAA, CCC transfer, private transfers and I²C transfers. The Linux documentation explicitly notes that DAA is more than transmitting ENTDAA: the driver must process discovered devices and instantiate them.

Linux also maintains bus-maintenance locking for operations that change bus-wide state, such as resetting dynamic addresses, enabling/disabling events and changing addresses. Normal transactions must be serialized by the controller driver so transfers are not interleaved.

## 12. Clock, reset, power and CDC integration

### 12.1 Clock domains

A practical controller may contain:
- Register bus clock (APB/AXI-Lite).
- Internal protocol-engine clock.
- Optional low-power / always-on event clock.
- External SCL timing observed at the pad.

The controller generates SCL in controller mode, but SDA and event observations are asynchronous to internal register and system clocks. Synchronize bus inputs, detect edges safely and avoid sampling asynchronous SDA directly in the register clock domain.

### 12.2 Reset behavior

Reset must define:
- SDA output enable released to safe high-impedance state.
- SCL output disabled or driven to the required idle state.
- Command engine idle.
- FIFOs and response queue flushed or marked invalid.
- IBI and Hot-Join state cleared or reinitialized.
- Dynamic address table invalidated if target state may have been lost.
- Software-visible reset cause and recovery state.

A controller reset does not guarantee every external target reset. If a target retained its dynamic address while the controller lost its address table, the system needs a defined re-discovery strategy.

### 12.3 Low-power behavior

I3C supports low-power sensor and management use cases. System integration should decide:
- Whether the controller remains in an always-on domain.
- Whether IBI can wake the host from a low-power state.
- Whether targets retain dynamic addresses in sleep.
- Whether bus state is preserved across controller power collapse.
- Whether the bus must be reinitialized after target or controller reset.
- Whether a target can be powered down independently without clamping SDA/SCL.

A target power switch that leaves SDA/SCL pins clamping the bus can prevent all other devices from communicating. Validate I/O fail-safe behavior.

## 13. Timing, throughput and performance calculations

### 13.1 SDR theoretical payload estimate

Use the common 12.5 MHz SDR SCL example. A controller-written data byte is followed by a T-bit, so a simple payload-only estimate is:

\[
R_{payload} = 12.5 \times 10^6 \times \frac{8}{9}
\]

\[
R_{payload} \approx 11.11 \text{ Mbit/s} \approx 1.389 \text{ MB/s}
\]

This is an idealized data phase. Address headers, START/STOP, bus turnarounds, CCCs, target processing and software overhead reduce end-to-end throughput. MIPI's overview quotes typical I3C data rate around 11.1 Mbit/s.

### 13.2 Example 32-byte private write

Assumptions:
- SCL = 12.5 MHz.
- Address header + response = 9 SCL periods.
- 32 data bytes × 9 SCL periods each.
- Ignore START/STOP setup/hold and bus-free time.

Total clock periods:

\[
N = 9 + (32 \times 9) = 297
\]

Time:

\[
t = \frac{297}{12.5 \times 10^6} = 23.76 \ \mu s
\]

Ideal payload rate for this simplified transaction:

\[
R = \frac{32}{23.76 \ \mu s} \approx 1.347 \text{ MB/s}
\]

The difference from 1.389 MB/s comes from the address header. Real throughput is lower once bus timing and software overhead are included.

### 13.3 HDR comparison

MIPI describes optional HDR modes with rates up to 100 Mbit/s for applicable I3C capabilities. This is a mode/capability headline, not a guaranteed payload rate for every I3C target or every Basic implementation.

For any product calculation, record:
- Exact I3C / I3C Basic revision.
- Supported HDR mode.
- Target capability and maximum message length.
- Controller implementation and timing.
- Mixed-bus or pure-bus topology.
- Number of command/address/turnaround cycles.
- Payload size and bus arbitration overhead.

### 13.4 IBI latency budget

A useful system-level decomposition is:

\[
T_{event} =
T_{target\_detect} +
T_{bus\_available} +
T_{arbitration} +
T_{controller\_accept} +
T_{payload} +
T_{host\_notify}
\]

IBI eliminates a dedicated interrupt wire but does not eliminate event latency. A busy bus, a higher-priority event, controller event inhibition or a delayed host interrupt handler can increase service latency.

## 14. Common design bugs and failure signatures

| Symptom | Likely cause | Debug method |
|---|---|---|
| DAA finds no targets | Wrong bus mode, SDA drive issue, target unpowered or ENTDAA sequence wrong | Capture 0x7E/W and ENTDAA CCC; inspect target participation |
| Only first target gets an address | DAA repeat/restart state machine or target re-arm bug | Trace arbitration rounds and target address-valid state |
| Duplicate dynamic address | Address-slot allocator or reset-state corruption | Dump full address table after each DAA round |
| Target ACKs address but data corrupts | T-bit parity, push-pull handoff, timing or SDA OE bug | Decode ninth bit and inspect pad drive mode |
| Controller write is NACKed unexpectedly | Address response or CCC capability mismatch | Separate address ACK from data T-bit handling |
| Read ends too early | Read T-bit interpreted as write parity or inverted end-of-data semantics | Decode target T-bit on every byte |
| IBI never reaches CPU | ENEC not issued, target BCR capability, controller mask or event FIFO issue | Check BCR, ENEC, IBI status and interrupt route |
| IBI storm | Target event not serviced/cleared or repeated request policy misunderstood | Correlate event source and software clear sequence |
| Hot-Join missed | Detection disabled, wrong bus mode or target request timing not handled | Probe bus-available window and Hot-Join event status |
| Legacy I²C device fails only at I3C speed | Missing/insufficient spike filter or incompatible bus mode | Check LVR, data sheet and mixed-bus SCL constraints |
| Bus stuck LOW after target power-down | Unpowered target clamps SDA/SCL or pad isolation issue | Power-cycle target separately; measure pin leakage/clamp |
| HDR works on one target only | Capability mismatch or wrong ENTHDR/exit handling | Read BCR/HDR capability and compare mode entry sequence |
| Warm boot DAA fails | Bootloader left dynamic addresses assigned; stale software table | RSTDAA and perform controlled re-discovery |
| Secondary controller never takes ownership | CRR/role handoff not supported or wrong policy | Check capability and controller-role state |

### 14.1 RTL-specific hazards

- Never enable push-pull HIGH while another target can legally pull SDA LOW.
- Make SDA output-enable changes occur at protocol-defined boundaries.
- Do not treat T-bit as I²C ACK/NACK.
- Use separate read and write T-bit logic.
- Check odd parity over the correct eight data bits.
- Handle arbitration loss by releasing the line and transitioning to the required listening/retry state.
- Do not allocate a reserved or occupied address.
- Do not accept an IBI when event storage is unavailable unless the specified controller behavior supports it.
- Do not enter HDR unless all required bus-level conditions and target capabilities are satisfied.
- Reset must not leave the pad driver actively sourcing a HIGH into a target that may be arbitrating.

## 15. Pre-silicon verification plan

### 15.1 Directed tests

1. Pure-I3C bus with one target.
2. Pure-I3C bus with multiple targets and DAA.
3. DAA with known PID/BCR/DCR values and deterministic address allocation.
4. DAA with reserved-address avoidance.
5. DAA arbitration collision and invalid dynamic-address parity.
6. Private SDR write with multiple bytes and parity checks.
7. Private SDR read with T-bit continuation and termination.
8. Broadcast CCC and direct CCC read/write.
9. ENEC/DISEC event enable/disable.
10. IBI accepted, IBI rejected and simultaneous IBI requests.
11. IBI with MDB and supported optional payload.
12. Hot-Join after initial bus setup.
13. Controller-role request/handoff if implemented.
14. Legacy I²C target on mixed-fast and mixed-slow configurations.
15. HDR entry, transaction and exit for each supported mode.
16. Maximum read/write length and over-length error handling.
17. Target reset, controller reset and warm-boot re-discovery.
18. FIFO full/empty, bus timeout and clock-domain stress.
19. Power down one target while other targets remain active.
20. Back-to-back private transfers and CCCs without frame interleaving.

### 15.2 Assertions to consider

These are conceptual properties. Adapt signal names, timing and reset semantics to the implementation and exact protocol revision.

~~~systemverilog
// Never actively drive push-pull while protocol state requires open-drain.
assert property (@(posedge clk_i) disable iff (!rst_ni)
  od_phase |-> !sda_push_pull_en);

// The controller must not assign a reserved or already occupied address.
assert property (@(posedge clk_i) disable iff (!rst_ni)
  daa_assign |-> address_is_legal && address_slot_free);

// A write data word must be followed by the correct odd-parity T-bit.
assert property (@(posedge clk_i) disable iff (!rst_ni)
  write_tbit_sample |-> (tbit == ~^last_write_byte));

// An IBI payload must not be committed unless the request was accepted.
assert property (@(posedge clk_i) disable iff (!rst_ni)
  ibi_payload_push |-> ibi_accepted);

// The command engine must not interleave two active transfer descriptors.
assert property (@(posedge clk_i) disable iff (!rst_ni)
  active_transfer |-> $stable(active_descriptor_id));

// No simultaneous internal push and FIFO-full overflow.
assert property (@(posedge clk_i) disable iff (!rst_ni)
  rx_fifo_push |-> !rx_fifo_full);
~~~

Additional properties:
- DAA winner identity is stable until address assignment completes.
- Every accepted dynamic address is unique.
- A parity error cannot mark an address slot valid.
- Read T-bit termination ends the read exactly once.
- Arbitration loss releases SDA output-enable.
- IBI source address maps to a valid assigned device.
- ENEC/DISEC software-visible state matches the controller's target-event state.
- HDR entry is only issued when the bus and target capabilities permit it.
- Controller reset clears/invalidates state that cannot be guaranteed to survive reset.
- Bus recovery eventually returns the controller to idle or reports a bounded fatal error.

### 15.3 Functional coverage

Cross:
- Pure/mixed-fast/mixed-limited/mixed-slow bus mode.
- Number of targets × DAA arbitration rounds.
- PID patterns × dynamic address choices.
- SDR read/write × message length.
- CCC broadcast/direct × payload direction.
- IBI accepted/rejected × MDB capability × simultaneous requests.
- Hot-Join timing × bus state.
- Reset type × DAA/IBI/transfer state.
- HDR mode × target capability × message length.
- FIFO watermark × interrupt/DMA/polling mode.

## 16. Post-silicon bring-up and debug

### Stage A — physical bus validation

1. Confirm target rails and I/O voltage.
2. Verify SDA/SCL pinmux and pad configuration.
3. Check pull-up arrangement and unpowered-target behavior.
4. Probe idle levels and open-drain low levels.
5. Confirm the controller can release SDA and that the line rises.
6. Validate rise/fall time and SCL pulse widths with a low-capacitance probe.

### Stage B — safe SDR bring-up

1. Start with one known I3C target.
2. Use a conservative supported bus rate.
3. Issue the required bus initialization and ENTDAA sequence.
4. Capture PID, BCR, DCR and assigned dynamic address.
5. Read a standard capability field/CCC.
6. Perform a short private write/read test.
7. Compare the raw waveform to the target data sheet and I3C specification timing.

### Stage C — event validation

1. Confirm the target advertises IBI capability.
2. Configure controller event storage and interrupt masks.
3. Enable the target event using ENEC.
4. Trigger one target event.
5. Verify arbitration, target address, ACK/NACK, MDB and optional payload.
6. Confirm host interrupt routing and software handler.
7. Test event rejection, masking and re-enable behavior.
8. Test Hot-Join independently from IBI.

### Stage D — mixed-bus and HDR validation

- Add one legacy I²C target only after pure-I3C is stable.
- Configure the correct mixed-bus mode based on its data sheet and LVR.
- Verify that legacy traffic is not corrupted by I3C push-pull phases.
- Enable HDR only when the controller and target both advertise the selected mode.
- Sweep supported frequencies and PVT corners.
- Validate error recovery and reset behavior after interrupted transfers.

### Useful trace fields

Capture:
- Timestamp and bus mode.
- SCL frequency and open-drain/push-pull phase.
- SDA output-enable and sampled input.
- START, repeated START and STOP.
- CCC ID and direction.
- Dynamic address, PID, BCR and DCR.
- DAA round number and arbitration-loss reason.
- T-bit value and parity result.
- IBI source address, acceptance and MDB.
- Hot-Join / controller-role event.
- FIFO occupancy and response status.
- Interrupt status and software service timestamp.
- Reset cause and address-table generation.

A useful debug sequence is: prove pad electrical behavior, decode bus framing, validate DAA/address table, validate event protocol, then debug Linux/firmware scheduling.

## 17. Practical SoC case study — sensor and audio-control subsystem

Assume a SoC contains:
- Motion sensor.
- Temperature sensor.
- Pressure sensor.
- Always-on sensor hub.
- DSP.
- Audio codec control interface.
- CPU and I3C controller.

I3C can consolidate control and event reporting for compatible sensors. It does not replace the high-bandwidth audio sample path: I²S/TDM, SoundWire or HDA may still carry audio data, while I3C may configure a compatible peripheral or sensor.

### Example architecture

~~~text
                     CPU / Firmware
                           |
                    APB / AXI-Lite
                           |
                    I3C Controller
                     /          \
                 SDA              SCL
                  |                |
       +----------+----------------+---------+
       |          |                |         |
    Motion     Temperature      Pressure   Sensor Hub
    sensor       sensor          sensor      target
       |          |                |         |
       +----------+---- IBI -------+---------+
                           |
                      Host IRQ
                           |
                      CPU / DSP

Audio sample data remains on I2S/TDM / SoundWire / HDA as designed.
I3C is the control/event path, not the audio streaming replacement.
~~~

### Example boot and runtime sequence

1. Boot ROM/firmware configures I3C pads and controller clock.
2. Controller initializes in the selected pure or mixed bus mode.
3. Firmware clears stale dynamic addresses if required.
4. Firmware performs DAA and builds the device table.
5. Firmware reads BCR/DCR/PID and verifies expected targets.
6. Device drivers configure sensor registers using private SDR transfers.
7. Controller IBI resources are allocated.
8. Firmware enables target events using ENEC.
9. A sensor event triggers IBI.
10. Controller captures source and MDB, then raises a host interrupt.
11. Driver schedules sensor data processing and returns to normal operation.

### System design question

If an IBI arrives while the DSP is power-gated, should it wake the CPU, the sensor hub, or the DSP? This is not decided by I3C alone. It is an SoC power/interrupt policy decision involving the always-on domain, wakeup routing, event buffering and software ownership.

## 18. Hands-on labs

### Lab 1 — I3C versus I²C waveform decoder

Write a decoder that distinguishes:
- I²C ninth-bit ACK/NACK.
- I3C controller-write parity T-bit.
- I3C target-read end-of-data T-bit.
- Broadcast 0x7E header.
- Private dynamic-address header.

### Lab 2 — DAA model

Create two target models with different PID/BCR/DCR values. Implement wired arbitration and prove:
- Only one target wins each round.
- The winning identity is captured correctly.
- The controller assigns a legal dynamic address.
- The target acknowledges a valid parity bit.
- The next unaddressed target can participate.

### Lab 3 — CCC engine

Implement a command sequencer for:
- RSTDAA.
- ENTDAA.
- GETPID.
- GETBCR.
- GETDCR.
- ENEC and DISEC.

Use a target BFM and check direct versus broadcast framing.

### Lab 4 — IBI engine

Implement target IBI request generation and controller capture. Test:
- One requester.
- Two simultaneous requesters.
- Controller accepts.
- Controller rejects.
- MDB present/absent.
- Event FIFO full.
- Event masked and re-enabled.

### Lab 5 — Mixed I²C/I3C bus

Connect an I3C target and a legacy I²C target. Model the legacy target's spike-filter capability and verify the controller selects a compatible bus mode. Confirm that I²C ACK/NACK semantics are not confused with I3C T-bit semantics.

### Lab 6 — Post-silicon triage

Given a trace where DAA succeeds but the first private read fails, build a decision tree separating:
- Incorrect dynamic address table.
- Wrong R/W direction.
- SDA push-pull/open-drain transition.
- Read T-bit termination bug.
- Target not ready after DAA.
- Timing and pad-level signal integrity.

## 19. Interview questions

1. What problem does I3C solve beyond I²C speed?
2. Why are there still only two bus wires?
3. Why does I3C use both open-drain and push-pull signaling?
4. What is the difference between I3C Basic and full I3C?
5. What is a dynamic address?
6. What are PID, BCR and DCR?
7. Explain the ENTDAA sequence.
8. Why must the controller maintain an address-slot table?
9. What is the difference between a broadcast CCC and a direct CCC?
10. Why is the I3C T-bit not an ACK/NACK bit?
11. How does the read T-bit indicate end-of-data?
12. What is IBI and how does it avoid an external interrupt pin?
13. How is IBI priority resolved?
14. What is the Mandatory Data Byte?
15. How is Hot-Join different from IBI?
16. How does a secondary controller request bus ownership?
17. What is the difference between SDR and HDR-DDR?
18. Why can a legacy I²C device limit an I3C bus?
19. What happens if the controller resets but targets retain dynamic addresses?
20. How would you debug a DAA pass followed by a private-transfer failure?

## 20. Design review checklist

- [ ] Exact I3C / I3C Basic specification revision selected.
- [ ] Full I3C versus Basic feature set documented.
- [ ] Controller and target SDR capabilities confirmed.
- [ ] HDR modes and target capability confirmed individually.
- [ ] Pure or mixed-bus mode justified from actual device data sheets.
- [ ] Legacy I²C devices and LVR information described.
- [ ] SDA pad supports required open-drain and push-pull behavior.
- [ ] SCL timing and electrical characteristics validated.
- [ ] DAA PID/BCR/DCR capture and address allocator verified.
- [ ] Reserved and occupied address slots excluded.
- [ ] T-bit write parity and read end-of-data implemented correctly.
- [ ] CCC support table and optional command behavior documented.
- [ ] IBI target capability, ENEC/DISEC and controller resources coordinated.
- [ ] Hot-Join handling defined.
- [ ] Controller-role handoff included only if actually supported.
- [ ] Reset and power-loss re-discovery strategy defined.
- [ ] Linux controller/device integration and bus locking reviewed.
- [ ] Mixed-bus and PVT validation planned.
- [ ] Error, timeout, arbitration and bus-recovery paths verified.

## 21. Normative facts, implementation examples and engineering interpretation

| Category | Examples |
|---|---|
| **Normative protocol facts** | Dynamic addressing, CCC framing, SDR T-bit semantics, IBI/Hot-Join behavior and HDR mode rules as defined by the selected MIPI revision |
| **Public explanatory documentation** | Linux I3C protocol and master-controller API; useful for software integration, not a substitute for the protocol specification |
| **Vendor implementation example** | ST AN5879 and STM32 I3C controller timing/register behavior |
| **Open RTL example** | CHIPS Alliance I3C Core repository: SystemVerilog implementation, Cocotb/Verilator tests and UVM environment; its stated compliance targets are I3C Basic v1.1.1, HCI v1.2 and TCRI v1.0 |
| **Engineering interpretation** | I3C's value is the combination of faster data phases, discovery, event delivery and reduced external pins—not just its clock frequency |

Do not generalize one vendor's register offsets, FIFO depths, open-drain timing or optional HDR support to every controller. Do not infer that an I3C target supports IBI or HDR merely because the controller supports those features.

## 22. Authoritative references and further reading

1. **MIPI Alliance — I3C and I3C Basic specification page.** Current versions listed: I3C v1.2 (February 2025), I3C Basic v1.2 (April 2025); previous revisions and public technical resources.  
   https://www.mipi.org/specifications/i3c-sensor-specification

2. **MIPI Alliance — I3C Basic v1.0 public-release announcement, 11 December 2018.** Documents the release and positioning of I3C Basic.  
   https://www.mipi.org/press-releases/mipi-releases-i3c-basic-interface-specification

3. **MIPI Alliance — I3C v1.1 announcement, 15 January 2020.** Documents the revision and market context.  
   https://www.mipi.org/press-releases/mipi-i3c-v1-1-strengthens-upgrade-path-for-i2c-implementers

4. **Linux kernel — I3C protocol overview.** Explains roles, DAA, PID/BCR/DCR, IBI, Hot-Join, CCCs, SDR and HDR from a software integration viewpoint.  
   https://docs.kernel.org/driver-api/i3c/protocol.html

5. **Linux kernel — I3C master-controller driver API.** DAA, CCC, IBI queues, bus locking, mixed-bus modes and controller operations.  
   https://docs.kernel.org/driver-api/i3c/master-driver-api.html

6. **STMicroelectronics AN5879 — Introduction to I3C for STM32 MCUs.** Public application note with bus format, DAA, CCC, IBI, timing and implementation examples.  
   https://www.st.com/resource/en/application_note/an5879-introduction-to-i3c-for-stm32-mcus-stmicroelectronics.pdf

7. **CHIPS Alliance — I3C Core.** Public SystemVerilog RTL and verification project. Its README states compliance targets of I3C Basic v1.1.1, I3C HCI v1.2 and TCRI v1.0; inspect the current repository status and limitations before using it as a reference.  
   https://github.com/chipsalliance/i3c-core

8. **MIPI I3C FAQ.** Clarifies IBI, Hot-Join and Controller Role Request differences.  
   https://www.mipi.org/resources/I3C-frequently-asked-questions

9. **Microchip online documentation — I3C broadcast address and T-bit.** Useful explanatory material for address 0x7E, SDR transaction phases and T-bit behavior.  
   https://onlinedocs.microchip.com/

10. **MIPI I3C Host Controller Interface (HCI).** A separate specification intended to standardize the host software interface to I3C controllers.  
    Accessible from the MIPI I3C specification page.

---

## Session outcome

After this lesson, you should be able to explain why I3C exists, distinguish it from I²C and SPI, decode SDR transfers and T-bits, implement the DAA state machine, reason about CCCs and dynamic-address tables, handle IBI and Hot-Join, understand HDR capability boundaries, integrate an I3C controller with Linux, and build a pre-silicon and post-silicon validation plan.

**Next: Session 22 — UART: asynchronous serial framing, baud generation, oversampling, FIFOs, flow control, error handling and silicon debug.**
