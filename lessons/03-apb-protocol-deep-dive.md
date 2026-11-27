# Lesson 03 — AMBA APB: Protocol, RTL Architecture and Silicon-Level Debug

**Track:** SoC Basics → Masterclass  
**Level:** Protocol fundamentals through peripheral RTL integration  
**Estimated study time:** 60 minutes  
**Prerequisites:** Lessons 01–02; basic synchronous logic and memory-mapped registers  
**Outcome:** Explain APB's design rationale, execute reads and writes cycle by cycle, implement a small APB completer, integrate an APB bridge, and debug protocol and register-access failures.

> **Source discipline:** This lesson is an original teaching explanation. Normative behavior is summarized from Arm IHI 0024E (Issue E, 28 February 2023). Check the exact specification revision and interface configuration used by your project. APB2/APB3/APB4/APB5 are feature generations; not every APB5 signal is present in every implementation.

---

## 1. One-hour learning plan

| Time | Topic | Expected outcome |
|---:|---|---|
| 0–6 min | Why APB exists | Explain the low-bandwidth peripheral-bus design trade-off |
| 6–13 min | Historical evolution | Distinguish APB2, APB3, APB4 and APB5 features |
| 13–23 min | Signals and architecture | Identify requester, completer, decoder and bridge |
| 23–33 min | Transfer state machine | Read setup/access waveforms and wait states |
| 33–41 min | APB3/4/5 extensions | Understand errors, byte strobes, protection and wake-up |
| 41–49 min | RTL and register design | Implement a register bank and protocol checks |
| 49–55 min | Performance and integration | Calculate bus throughput and identify CDC/power risks |
| 55–60 min | Silicon debug and lab | Build a practical validation checklist |

## 2. Why APB exists: the engineering backstory

A modern SoC contains high-throughput requesters such as CPU cores, DMA engines, GPUs and media engines. It also contains many low-bandwidth control/status blocks: timers, watchdogs, UARTs, GPIO, clock controllers, interrupt controllers and I2C/SPI controllers.

These two classes of traffic have very different requirements:

- CPU cache misses and DMA transfers benefit from pipelining, bursts, multiple outstanding transactions and high throughput.
- A timer control-register write or UART status read usually transfers one small word and does not need a complex high-performance protocol.
- Putting every peripheral directly on a large high-performance fabric increases wiring, decode, clocking, verification and integration cost.
- Giving every simple register block a full high-performance interface can waste area and power.

APB is the deliberately simpler side of this design space. It provides a compact, synchronous, memory-mapped register-transfer interface. A bridge isolates APB devices from a high-performance system fabric.

### 2.1 Historical evolution

Arm's APB revision history gives the following milestones:

| Generation | Specification milestone | Important additions |
|---|---|---|
| APB2 | AMBA 2 / Issue A lineage | Basic two-phase peripheral access; no APB3 wait-state/error handshake |
| APB3 | AMBA 3 / Issue B; first release 25 Sep 2003 | PREADY wait-state support and PSLVERR error reporting |
| APB4 | Issue C; first release 13 Apr 2010 | PPROT transaction protection and PSTRB sparse byte writes |
| APB5 | Issue D; 9 Apr 2021 | PWAKEUP, user sideband signals and optional interface parity protection |
| APB5 with RME | Issue E; 28 Feb 2023 | Optional PNSE support for Root and Realm physical address spaces |

The naming can be confusing: Arm's current Issue E document is still the APB5 protocol. Do not confuse the older, obsolete document called APB Specification Rev E (1998) with the modern Issue E release from 2023.

### 2.2 What changed and why?

**APB2 — minimum complexity**

The original peripheral interface uses a simple setup phase followed by an access phase. Its fixed transfer duration keeps the completer logic small, but a peripheral cannot extend a transaction to wait for an internal operation.

**APB3 — variable-latency peripherals and error visibility**

PREADY allows a completer to extend the access phase. PSLVERR lets a peripheral report an error at transfer completion. This is useful for bridges, register windows and devices whose response time is not fixed.

**APB4 — software-visible access attributes**

PSTRB allows selected byte lanes to be written without overwriting neighboring bytes. PPROT carries protection attributes, including Secure/Non-secure information. These extensions make APB more useful in larger systems without turning it into a high-performance burst bus.

**APB5 — low-power and safety-oriented integration**

PWAKEUP provides a glitch-free activity indication that can be used by clock/power control. User signals provide optional application-defined metadata. Interface parity can detect single-bit corruption on protected signal groups. Issue E adds optional RME address-space information through PNSE.

**Engineering interpretation:** APB evolves by adding features at the boundary while preserving its fundamental two-phase, one-transfer-at-a-time model. That is a useful example of extending a widely deployed interface without changing its core transaction shape.

## 3. Where APB sits inside a real SoC

~~~mermaid
flowchart LR
  CPU["CPU / software"] --> FAB["AXI / AHB fabric"]
  DMA["DMA / debug requester"] --> FAB
  FAB --> BR["AXI-to-APB or AHB-to-APB bridge"]
  BR --> DEC["APB address decode"]
  DEC --> UART["UART"]
  DEC --> GPIO["GPIO"]
  DEC --> TIMER["Timer / watchdog"]
  DEC --> CLK["Clock / reset registers"]
  DEC --> I2C["I2C / SPI controller"]
~~~

A typical APB subsystem has:

1. An upstream bridge that accepts AXI/AHB or another system transaction.
2. A bridge control FSM and request holding registers.
3. An APB requester interface that drives address, control and data.
4. Address decode that selects one completer.
5. One or more APB completers that implement register behavior.
6. Response collection and mapping back to the upstream bus.

The APB protocol defines the interface behavior. It does not require one particular bridge microarchitecture, decoder topology, queue depth or register implementation.

### 3.1 Requester and completer

Current Arm terminology uses **Requester** for the component initiating an APB transfer and **Completer** for the selected peripheral responding to it. Older design documents often say master and slave. Both naming conventions are encountered in RTL and verification environments.

## 4. APB signal reference

The table describes common signals. Some are optional and depend on APB generation and interface configuration.

| Signal | Direction | Role |
|---|---|---|
| PCLK | Shared clock | All APB transfer sampling is synchronous to its rising edge |
| PRESETn | Shared reset | Active-low APB reset |
| PADDR | Requester → Completer | Address of the register/target location |
| PSELx | Requester → Completer | Selects a particular completer |
| PENABLE | Requester → Completer | Marks the Access phase |
| PWRITE | Requester → Completer | HIGH for write, LOW for read |
| PWDATA | Requester → Completer | Write-data bus |
| PRDATA | Completer → Requester | Read-data bus |
| PREADY | Completer → Requester | Completes an Access phase when HIGH |
| PSLVERR | Completer → Requester | Optional transfer error indication |
| PSTRB | Requester → Completer | Optional per-byte write strobes (APB4+) |
| PPROT | Requester → Completer | Optional protection attributes (APB4+) |
| PWAKEUP | Requester → power/clock logic | Optional APB5 activity/wake indication |
| PAUSER / PWUSER | Requester → Completer | Optional APB5 request and write-data metadata |
| PRUSER / PBUSER | Completer → Requester | Optional APB5 read-data and response metadata |
| PNSE | Requester → Completer | Optional APB5 Issue E RME address-space extension |

A peripheral normally receives a decoded select such as PSEL_UART or PSEL_TIMER. A system-level APB fabric can expose one PSEL per completer.

### 4.1 Important distinction: PSEL is not PENABLE

- PSEL says which completer is selected for the transfer.
- PENABLE distinguishes the Access phase from the Setup phase.
- A valid transfer always has a Setup phase before its Access phase.
- PENABLE is not a generic transaction-valid signal that can be asserted without the Setup phase.

## 5. The APB state machine

APB has three conceptual states:

~~~mermaid
stateDiagram-v2
  [*] --> IDLE
  IDLE --> SETUP: Request accepted
  SETUP --> ACCESS: Next rising PCLK edge
  ACCESS --> ACCESS: PREADY = 0
  ACCESS --> IDLE: PREADY = 1 / no next transfer
  ACCESS --> SETUP: PREADY = 1 / next transfer
  IDLE --> [*]
~~~

### 5.1 IDLE

- No transfer is active.
- PSELx is LOW.
- PENABLE is LOW.
- The requester may prepare the next transfer.

### 5.2 SETUP

- The requester asserts the selected PSELx.
- PENABLE remains LOW.
- PADDR, PWRITE and write data are presented as required.
- Setup lasts exactly one PCLK cycle.
- The next rising edge moves the transfer into Access.

### 5.3 ACCESS

- PENABLE is HIGH.
- PSELx remains HIGH for the selected completer.
- The completer asserts PREADY when it can finish the transfer.
- If PREADY is LOW, Access is extended by one or more wait cycles.
- Completion occurs on a rising PCLK edge when PSELx, PENABLE and PREADY are all HIGH.

The transfer-completion condition is:

~~~text
apb_complete = PSELx && PENABLE && PREADY
~~~

For a write, the register update event is typically:

~~~text
apb_write_fire = apb_complete && PWRITE
~~~

For a read, the response is sampled at completion:

~~~text
apb_read_fire = apb_complete && !PWRITE
~~~

Do not update a register merely because PSEL and PWRITE are high. That would cause a write during Setup and can repeat writes during wait states.

## 6. Cycle-by-cycle transfer walkthrough

### 6.1 Write with no wait states

Assume a 32-bit write to address 0x4000_0010 with data 0x1234_5678.

| PCLK edge | Phase before edge | PSEL | PENABLE | PREADY | Meaning |
|---|---|---:|---:|---:|---|
| T0 | IDLE | 0 | 0 | X | No transfer |
| T1 | SETUP | 1 | 0 | X | Address, direction and data presented |
| T2 | ACCESS | 1 | 1 | 1 | Completer ready; transfer completes at this edge |
| T3 | IDLE or next SETUP | 0 or 1 | 0 | X | Bus moves to idle or next transfer setup |

A no-wait APB transfer therefore consumes two protocol cycles: one Setup cycle and one Access cycle.

### 6.2 Write with two wait states

| PCLK edge | PSEL | PENABLE | PREADY | Interpretation |
|---|---:|---:|---:|---|
| T1 | 1 | 0 | X | Setup |
| T2 | 1 | 1 | 0 | Access wait 1 |
| T3 | 1 | 1 | 0 | Access wait 2 |
| T4 | 1 | 1 | 1 | Transfer completes |

During every Access wait cycle, the requester holds the transfer stable. In particular, PADDR, PWRITE, PSEL, PENABLE, PWDATA for writes, PSTRB and PPROT must not change. APB5 user attributes also have their specified stability requirements.

### 6.3 Read transfer

A read uses the same Setup and Access sequence, but PWRITE is LOW. The completer must provide PRDATA in time for the completion edge. If the completer is not ready, it holds PREADY LOW and the Access phase extends.

For APB4+, PSTRB must be driven LOW for reads.

### 6.4 Back-to-back transfers

If a transfer completes and another transfer follows, the bus can move directly from Access to the next Setup phase without an extra idle cycle. PENABLE goes LOW for the new Setup phase. The selected PSEL changes if the next transfer targets a different completer.

This removes an unnecessary bubble, but APB still has only one active transfer at a time on a given APB interface.

## 7. APB2, APB3, APB4 and APB5: feature comparison

| Feature | APB2 | APB3 | APB4 | APB5 |
|---|---:|---:|---:|---:|
| Basic Setup/Access phases | Yes | Yes | Yes | Yes |
| PREADY wait states | No | Yes | Yes | Yes |
| PSLVERR | No | Yes | Yes | Yes |
| PSTRB byte write enables | No | No | Yes | Yes |
| PPROT protection attributes | No | No | Yes | Yes |
| PWAKEUP | No | No | No | Optional |
| User sideband signals | No | No | No | Optional |
| Interface parity protection | No | No | No | Optional |
| PNSE for RME | No | No | No | Optional in Issue E |

Do not infer that a signal exists just because a document says APB5. Optional interface properties determine which optional signals are actually present. A bridge must be configured consistently with the connected requester and completer.

## 8. APB3 details: PREADY and PSLVERR

### 8.1 PREADY

PREADY is driven by the completer. During Access, LOW means the completer needs more time. HIGH means it can complete the transfer on the sampling edge.

PREADY can have any value while PENABLE is LOW; it is not used to complete a transfer during Setup.

A simple register bank can tie PREADY HIGH. A peripheral that waits for an internal operation can implement a wait-state counter or a completion handshake.

**Implementation warning:** If PREADY depends on an internal clock-domain signal, do not sample an unsynchronized signal directly. Use a safe CDC handshake or asynchronous bridge design.

### 8.2 PSLVERR

PSLVERR is meaningful only on the final completion cycle, when PSEL, PENABLE and PREADY are all HIGH.

It can report errors on reads and writes. Important subtlety: an error response does not guarantee that a write had no side effect. Whether the peripheral state changed is peripheral-specific. A read error can also return data that is not meaningful; the protocol does not require PRDATA to be zero.

This matters when designing a bridge:
- AXI-to-APB bridges commonly map PSLVERR to AXI SLVERR on the corresponding read or write response.
- AHB-to-APB bridges commonly map it to an AHB ERROR response.
- The exact mapping and any additional bridge policy must be checked against the bridge specification.

**Design recommendation:** Define the side-effect policy of each register access. If an illegal write must never partially modify state, validate the address, privilege and write mask before committing the write.

## 9. APB4 details: PSTRB and PPROT

### 9.1 PSTRB — byte lane write enables

There is one PSTRB bit per byte of PWDATA.

For a 32-bit data bus:

| PSTRB | Byte lane | Data bits |
|---:|---|---|
| PSTRB[0] | Byte 0 | PWDATA[7:0] |
| PSTRB[1] | Byte 1 | PWDATA[15:8] |
| PSTRB[2] | Byte 2 | PWDATA[23:16] |
| PSTRB[3] | Byte 3 | PWDATA[31:24] |

Example: PWDATA = 0xAABBCCDD and PSTRB = 4'b0101 means byte lanes 0 and 2 are enabled. The register update must preserve lanes 1 and 3.

~~~systemverilog
if (apb_write_fire) begin
  if (PSTRB[0]) reg_word[7:0]   <= PWDATA[7:0];
  if (PSTRB[1]) reg_word[15:8]  <= PWDATA[15:8];
  if (PSTRB[2]) reg_word[23:16] <= PWDATA[23:16];
  if (PSTRB[3]) reg_word[31:24] <= PWDATA[31:24];
end
~~~

PSTRB is optional. If the requester does not provide PSTRB but the completer does, the interface compatibility rules define the expected tie-off behavior. Read transfers drive all PSTRB bits LOW.

**Common bug:** treating PSTRB as a bit mask instead of a byte-lane mask, or reversing byte lanes during endian conversion.

### 9.2 PPROT — protection attributes

PPROT[2:0] carries three attributes:

| Bit | LOW | HIGH |
|---|---|---|
| PPROT[0] | Normal access | Privileged access |
| PPROT[1] | Secure access | Non-secure access |
| PPROT[2] | Data access | Instruction access |

PPROT is an attribute, not a complete security mechanism. A completer, firewall or upstream security controller must actually enforce the policy. PPROT[2] is an indication/hint and might not be accurate in every system.

If the system uses PPROT[1] to distinguish Secure and Non-secure accesses, dropping or incorrectly tying this bit in a bridge can become a security vulnerability.

## 10. APB5 details: wake-up, user metadata and parity

### 10.1 PWAKEUP

PWAKEUP indicates activity associated with an APB requester. It is designed to be glitch-free and suitable for asynchronous sampling by clock/power control logic.

Important integration points:
- It is synchronous to PCLK but must be suitable for sampling in another clock domain.
- It may assert before, during or after PSEL.
- It is recommended to assert it at least one cycle before PSEL to avoid delaying a new transfer.
- If PWAKEUP and PSEL are HIGH in the same cycle, PWAKEUP must remain asserted until PREADY is asserted.
- A completer is permitted to wait for PWAKEUP before asserting PREADY. If the requester never asserts it, the transfer can deadlock.
- Independently gating the completer clock can cause the Setup phase to be missed. The requester/completer clocking strategy must be designed together.

PWAKEUP is not a replacement for a correct clock-domain crossing or a complete power-state protocol.

### 10.2 User signals

APB5 defines optional:
- PAUSER: request metadata.
- PWUSER: write-data metadata.
- PRUSER: read-data metadata.
- PBUSER: response metadata.

The protocol defines when these signals are valid, but does not define their application meaning. Use them only when both sides agree on the interpretation. Arm explicitly warns that incompatible use can create interoperability problems.

### 10.3 Interface parity

APB5 can add parity checks to detect single-bit errors on signal groups. The specification defines odd parity and limits each parity bit to covering no more than eight payload bits. Critical control groups use a single parity bit.

Examples of check-signal groups include:
- PADDRCHK for address.
- PCTRLCHK for selected control/protection attributes.
- PSELxCHK and PENABLECHK for control.
- PWDATACHK and PRDATACHK for data.
- PSTRBCHK, PREADYCHK and PSLVERRCHK.
- Optional user and wake-up check signals.

Parity detects a class of corruption; it does not correct the data. The system must define how a detected error is reported, contained and recovered. End-to-end error detection requires consistent protection across the source, interconnect, bridge and destination.

### 10.4 Issue E: PNSE and RME

APB5 Issue E optionally adds PNSE for Arm Realm Management Extension address-space identification. PNSE combines with PPROT[1]:

| PNSE | PPROT[1] | Address space |
|---:|---:|---|
| 0 | 0 | Secure |
| 0 | 1 | Non-secure |
| 1 | 0 | Root |
| 1 | 1 | Realm |

If PNSE is implemented, PPROT must also be present. This feature is optional and should not be assumed on older APB5 or APB4 peripherals.

## 11. APB bridge architecture

An APB bridge is not merely a wire adapter. It converts a potentially pipelined or multi-channel upstream transaction into the sequential APB Setup/Access protocol.

~~~mermaid
flowchart TB
  UP["AXI / AHB requester"] --> Q["Request capture / queue"]
  Q --> FSM["Bridge control FSM"]
  FSM --> DEC["Address decode"]
  DEC --> APB["APB requester signals"]
  APB --> PER["Selected APB completer"]
  PER --> RSP["Read data / error capture"]
  RSP --> MAP["Upstream response mapping"]
  MAP --> UP
~~~

### 11.1 Typical bridge sequence

1. Accept an upstream address/control request.
2. Capture address, direction, write data and byte enables.
3. Decode the APB peripheral select.
4. Drive APB Setup with PENABLE LOW.
5. Drive Access with PENABLE HIGH.
6. Wait for PREADY HIGH.
7. Capture PRDATA and PSLVERR on the completion edge.
8. Return the upstream response.
9. Release or advance the request queue.

### 11.2 Bridge design choices

| Choice | Simpler implementation | Higher-throughput implementation |
|---|---|---|
| Request storage | One holding register | FIFO with several queued requests |
| Outstanding APB transfers | One | Still one active transfer per APB interface, but upstream requests can queue |
| Peripheral decode | Central decoder | Hierarchical or distributed decode |
| Clock domains | Same PCLK | Asynchronous bridge/FIFO |
| Error handling | Basic response mapping | Detailed fault capture and timeout reporting |
| Power | Always-on APB clock | Clock gating and wake-up coordination |

A bridge must preserve the information required by the upstream protocol: address, operation, data, byte enables, protection attributes and error status. If the upstream bus supports transaction IDs or multiple outstanding operations, the bridge must track which upstream request is being serviced even though APB itself has no transaction ID.

### 11.3 Address decoding

For example, a system might map:
- UART: 0x4000_0000–0x4000_0FFF
- Timer: 0x4000_1000–0x4000_1FFF
- GPIO: 0x4000_2000–0x4000_2FFF
- I2C: 0x4000_3000–0x4000_3FFF

The decoder should produce exactly one PSEL for a legal mapped address. An unmapped address should have a defined error response or be handled by a default error completer; it must not silently hang.

## 12. Register-bank architecture: practical example

Assume a 32-bit APB peripheral with this register map:

| Offset | Register | Access | Purpose |
|---:|---|---|---|
| 0x00 | CTRL | RW | Enable and mode |
| 0x04 | STATUS | RO | Busy and completion flags |
| 0x08 | IRQ_ENABLE | RW | Interrupt enables |
| 0x0C | IRQ_STATUS | W1C | Write-one-to-clear interrupt flags |
| 0x10 | DATA | RW | Data register |
| 0x14 | ID | RO | Peripheral identification |

### 12.1 Register write semantics

Do not treat every register as a normal storage register:
- RW: software can read and write.
- RO: writes should be ignored or reported according to the programming model.
- WO: reads may return a defined value or be unsupported.
- W1C: writing a one clears the corresponding status bit.
- W1S: writing a one sets a bit.
- Read-to-clear: reading changes state; this needs careful side-effect and retry design.
- Reserved bits: behavior must be documented and consistently implemented.

### 12.2 Illustrative APB completer RTL

The following is a small teaching example, not a drop-in production IP. It assumes a 32-bit APB4-style interface, one PCLK domain, no optional APB5 signals, and a single selected completer.

~~~systemverilog
module apb_example_regs (
  input  logic        PCLK,
  input  logic        PRESETn,

  input  logic [11:0] PADDR,
  input  logic        PSEL,
  input  logic        PENABLE,
  input  logic        PWRITE,
  input  logic [31:0] PWDATA,
  input  logic [3:0]  PSTRB,
  input  logic [2:0]  PPROT,

  output logic [31:0] PRDATA,
  output logic        PREADY,
  output logic        PSLVERR
);

  logic [31:0] ctrl_q;
  logic [31:0] irq_enable_q;
  logic [31:0] irq_status_q;
  logic [31:0] data_q;

  logic apb_complete;
  logic apb_write_fire;

  assign PREADY = 1'b1;

  assign apb_complete = PSEL && PENABLE && PREADY;
  assign apb_write_fire = apb_complete && PWRITE;

  always_comb begin
    PRDATA = 32'h0000_0000;
    PSLVERR = 1'b0;

    if (PSEL && !PWRITE) begin
      unique case (PADDR[7:0])
        8'h00: PRDATA = ctrl_q;
        8'h04: PRDATA = 32'h0000_0000; // STATUS example
        8'h08: PRDATA = irq_enable_q;
        8'h0C: PRDATA = irq_status_q;
        8'h10: PRDATA = data_q;
        8'h14: PRDATA = 32'h534F_4303; // ID
        default: begin
          PRDATA = 32'h0000_0000;
          PSLVERR = 1'b1;
        end
      endcase
    end

    if (PSEL && PWRITE) begin
      unique case (PADDR[7:0])
        8'h00, 8'h08, 8'h0C, 8'h10: PSLVERR = 1'b0;
        default: PSLVERR = 1'b1;
      endcase
    end
  end

  always_ff @(posedge PCLK or negedge PRESETn) begin
    if (!PRESETn) begin
      ctrl_q       <= 32'h0;
      irq_enable_q <= 32'h0;
      irq_status_q <= 32'h0;
      data_q       <= 32'h0;
    end else begin
      if (apb_write_fire) begin
        unique case (PADDR[7:0])
          8'h00: begin
            if (PSTRB[0]) ctrl_q[7:0]   <= PWDATA[7:0];
            if (PSTRB[1]) ctrl_q[15:8]  <= PWDATA[15:8];
            if (PSTRB[2]) ctrl_q[23:16] <= PWDATA[23:16];
            if (PSTRB[3]) ctrl_q[31:24] <= PWDATA[31:24];
          end

          8'h08: begin
            if (PSTRB[0]) irq_enable_q[7:0]   <= PWDATA[7:0];
            if (PSTRB[1]) irq_enable_q[15:8]  <= PWDATA[15:8];
            if (PSTRB[2]) irq_enable_q[23:16] <= PWDATA[23:16];
            if (PSTRB[3]) irq_enable_q[31:24] <= PWDATA[31:24];
          end

          8'h0C: begin
            if (PSTRB[0]) irq_status_q[7:0]   <= irq_status_q[7:0]   & ~PWDATA[7:0];
            if (PSTRB[1]) irq_status_q[15:8]  <= irq_status_q[15:8]  & ~PWDATA[15:8];
            if (PSTRB[2]) irq_status_q[23:16] <= irq_status_q[23:16] & ~PWDATA[23:16];
            if (PSTRB[3]) irq_status_q[31:24] <= irq_status_q[31:24] & ~PWDATA[31:24];
          end

          8'h10: begin
            if (PSTRB[0]) data_q[7:0]   <= PWDATA[7:0];
            if (PSTRB[1]) data_q[15:8]  <= PWDATA[15:8];
            if (PSTRB[2]) data_q[23:16] <= PWDATA[23:16];
            if (PSTRB[3]) data_q[31:24] <= PWDATA[31:24];
          end

          default: ;
        endcase
      end
    end
  end

endmodule
~~~

### 12.3 Production RTL issues to fix before reuse

This example intentionally omits several production concerns:
- STATUS and IRQ_STATUS update sources and set/clear priority.
- PPROT checking and security policy.
- Address alignment and unsupported access-size policy.
- Reset values and reset-domain requirements from the actual IP spec.
- Read-data timing and output registering if required for timing closure.
- APB5 parity and optional user/wake signals.
- Formal properties, assertions, CDC/RDC and error injection.
- A defined policy for writes to RO registers and accesses to reserved offsets.

The example's combinational PSLVERR must be valid only when the transfer completes. A production design should qualify or register error state carefully so it is stable on the final Access cycle. It should also ensure that no state changes occur for an access that is rejected by its protection or address policy.

## 13. Verification: assertions and test scenarios

### 13.1 Core protocol properties

~~~systemverilog
// A SETUP phase must be followed by ACCESS on the next sampled clock.
property setup_to_access;
  @(posedge PCLK) disable iff (!PRESETn)
    (PSEL && !PENABLE) |=> (PSEL && PENABLE);
endproperty
assert property (setup_to_access);

// If ACCESS is stalled, the transfer remains selected and stable.
property hold_during_wait;
  @(posedge PCLK) disable iff (!PRESETn)
    (PSEL && PENABLE && !PREADY)
    |=> (PSEL && PENABLE &&
         $stable({PADDR, PWRITE, PWDATA, PSTRB, PPROT}));
endproperty
assert property (hold_during_wait);

// A write side effect must only occur on a completed write.
property write_only_on_completion;
  @(posedge PCLK) disable iff (!PRESETn)
    apb_write_fire |-> (PSEL && PENABLE && PREADY && PWRITE);
endproperty
assert property (write_only_on_completion);
~~~

These are teaching properties. Adapt optional signals, reset behavior, multiple PSELs, and tool syntax to the actual interface. A complete protocol checker must also verify legal PSEL/PENABLE sequencing, read-data validity, error validity, and APB5 signal-specific requirements.

### 13.2 Directed and constrained-random tests

| Test | What it catches |
|---|---|
| Read every implemented register | Decode and read mux bugs |
| Write every RW register | Write-enable and address bugs |
| Write each PSTRB pattern | Byte-lane mapping errors |
| Read with PSTRB nonzero | Requester protocol violation |
| Insert 1, 2 and many wait states | Payload instability and early completion |
| Error on read and write | PSLVERR qualification and bridge mapping |
| Back-to-back accesses to same target | Missing setup phase or extra bubbles |
| Back-to-back accesses to different targets | PSEL decode transition bugs |
| Reset during idle, setup and access | State-machine recovery |
| Access unmapped addresses | Default error/hang behavior |
| Privileged/secure/non-secure accesses | PPROT loss or policy bypass |
| APB5 wake-up with gated clock | Missed setup and deadlock |
| Inject parity corruption | Error detection and containment |
| Concurrent status set and W1C clear | Register side-effect priority |

### 13.3 Functional coverage

Collect coverage for:
- Read versus write.
- Every PSEL target.
- Every legal register offset.
- All supported PSTRB masks.
- Zero, one and multiple wait cycles.
- PSLVERR on read and write.
- Back-to-back same-target and different-target transfers.
- Reset in each FSM state.
- Each implemented protection encoding.
- APB5 wake-up, user and parity configurations where implemented.

## 14. Performance and timing calculations

APB has no burst mechanism and one transfer completes at a time on a given interface.

Let:
- f = PCLK frequency.
- W = number of wait-state cycles in Access.
- D = bytes transferred per completed APB transfer.

For the standard two-phase transfer:

~~~text
Cycles per transfer = 2 + W
Transfer rate       = f / (2 + W)
Payload bandwidth   = D * f / (2 + W)
~~~

### Example 1 — 32-bit APB, no wait states

Assume:
- PCLK = 100 MHz.
- Data width = 32 bits = 4 bytes.
- W = 0.

Then:

~~~text
Transfer rate = 100 MHz / 2 = 50 million transfers/s
Payload rate  = 50 million * 4 bytes = 200 MB/s
~~~

This is a theoretical protocol-level upper bound for a continuous stream of 32-bit transfers with no wait states. It is not a realistic expectation for a UART or timer and excludes implementation-specific gaps and peripheral limitations.

### Example 2 — two wait states

~~~text
Cycles per transfer = 2 + 2 = 4
Transfer rate       = 100 MHz / 4 = 25 million transfers/s
Payload rate        = 25 million * 4 = 100 MB/s
~~~

Wait states halve the ideal transfer rate in this example.

### Example 3 — why APB is not a DMA data bus

A DMA engine moving a 4 KiB buffer over a 128-bit AXI path can use bursts and exploit multiple outstanding requests. APB serializes each small transfer through Setup and Access. It is therefore a poor choice for sustained bulk data movement.

Use APB to configure the DMA engine and read its status. Use a high-performance memory-mapped or streaming interface for the actual payload.

## 15. Clock, reset, power and CDC integration

### 15.1 Clock

APB signals are synchronous to PCLK. The requester and completer on a conventional APB connection share that clock. APB itself does not define a general asynchronous handshake.

If the upstream AXI/AHB clock differs from PCLK, the bridge must safely cross the clock boundary. Common designs use asynchronous request/response FIFOs or a well-designed handshake bridge.

Never put independent two-flop synchronizers on every bit of PADDR or PWDATA. That can capture a mixed, incoherent word.

### 15.2 Reset

PRESETn is active-low. Define:
- Reset assertion behavior for requester, bridge and completer.
- Whether reset is asynchronous or synchronized internally.
- Reset release synchronization relative to PCLK.
- What happens to an in-flight APB transfer.
- Whether the upstream bridge returns an error, aborts or retries after reset.

A common silicon failure is resetting the peripheral but not the bridge state, leaving a stale request or response pending.

### 15.3 Clock gating and power

APB's simple register traffic makes it attractive for low-power peripheral islands. However:
- Do not gate PCLK in the middle of an Access transfer.
- Drain or safely retain outstanding bridge state before gating.
- Ensure the requester can wake the completer when required.
- If PWAKEUP is used, verify its glitch-free generation and deassertion.
- Verify reset and wake sequencing across independently powered domains.

## 16. Common RTL and integration bugs

| Symptom | Root cause to investigate |
|---|---|
| Write happens twice | Register update not qualified by Access completion |
| Register changes during wait | Write enable asserted before PREADY completion |
| Read returns previous value | PRDATA decode/latency not aligned with completion |
| One APB target never responds | PSEL decode or PREADY stuck low |
| Bus hangs on invalid address | No default completer/error response |
| Byte writes corrupt adjacent fields | PSTRB lane mapping or mask bug |
| Secure access reaches protected register | PPROT dropped, inverted or not checked |
| PSLVERR appears during Setup | Error not qualified to final completion |
| Back-to-back access fails | Incorrect PENABLE deassertion or PSEL switching |
| Failure only after warm reset | Bridge/FSM state not flushed |
| APB5 peripheral misses first transfer | Clock gated independently; wake-up/Setup sequencing issue |
| Rare parity errors | Incorrect parity grouping, stale check bit or routing corruption |
| Software sees random register values | Reset values, CDC, read mux or address alias issue |

## 17. Post-silicon validation and debug workflow

When software reports that a peripheral register is not accessible, debug in this order:

1. Confirm the CPU or debug requester generated the intended physical address.
2. Confirm the address translation/firewall permits the access.
3. Check the upstream interconnect decode and bridge request acceptance.
4. Capture APB PADDR, PWRITE, PSEL, PENABLE and PWDATA.
5. Confirm Setup lasts one PCLK cycle.
6. Confirm Access begins with PENABLE HIGH.
7. Check whether PREADY becomes HIGH.
8. If PREADY stays LOW, inspect peripheral clock, reset, internal busy state and power domain.
9. On completion, inspect PRDATA and PSLVERR.
10. Confirm the bridge returned the response to the correct upstream requester.
11. Check whether the register changed exactly once and whether byte strobes were applied correctly.

### Recommended debug counters

- APB read/write transfer count per completer.
- Wait-state histogram.
- Maximum consecutive wait cycles.
- PSLVERR count by address window.
- Unmapped access count.
- Clock-wake request count and wake latency.
- Parity error count by signal group.
- Bridge outstanding request count.
- Reset-aborted transaction count.

For an intermittent failure, use a trace trigger on PSEL && PENABLE && !PREADY for more than a chosen threshold, or on PSLVERR. Preserve the address and requester metadata around the event.

## 18. Hands-on lab: build a timer APB peripheral

Design a timer with the following registers:

| Offset | Register | Behavior |
|---:|---|---|
| 0x00 | CTRL | bit 0 enable, bit 1 periodic |
| 0x04 | LOAD | reload value |
| 0x08 | COUNT | current count, read-only |
| 0x0C | IRQ_STATUS | W1C interrupt flag |
| 0x10 | IRQ_ENABLE | interrupt enable |

### Tasks

1. Draw the APB requester-to-completer connection and address decode.
2. Implement IDLE, SETUP and ACCESS control in the requester/bridge.
3. Implement the register bank and byte strobes.
4. Add a programmable wait-state option for COUNT reads.
5. Assert PSLVERR for an unmapped address.
6. Add assertions for Setup-to-Access and stable payload while stalled.
7. Verify W1C behavior when a new timer event arrives on the same cycle as a software clear.
8. Add a warm-reset test during a stalled Access phase.
9. Add a PPROT check so non-secure software cannot modify a protected control register.
10. Measure transfer rate with zero, one and three wait states.

### Expected design discussion

The timer's counter runs independently of APB transactions. APB accesses configure and observe it; the counter must not stop merely because APB is idle. The IRQ flag is set by the timer event and cleared by a W1C write. Define set-versus-clear priority explicitly so an event is not lost if it coincides with software clearing the flag.

## 19. Interview and review questions

1. Why was APB created instead of connecting every peripheral directly to AXI?
2. What is the minimum number of PCLK cycles for an APB transfer?
3. What is the difference between Setup and Access?
4. When is PREADY meaningful?
5. What exact condition completes an APB transfer?
6. Why must address and control remain stable during wait states?
7. Can PSLVERR be asserted for a read?
8. Does PSLVERR guarantee a failed write did not update the peripheral?
9. How does PSTRB map to byte lanes?
10. What is the purpose of PPROT?
11. Is PPROT itself an access-control firewall?
12. Why can a bridge queue multiple upstream requests even though APB executes one transfer at a time?
13. What can go wrong if the completer clock is gated independently?
14. What does PWAKEUP solve, and what deadlock can occur?
15. What does APB5 parity detect, and what does it not do?
16. What does PNSE add in APB5 Issue E?
17. Why is APB unsuitable for sustained DMA payload movement?
18. How would you debug a peripheral that holds PREADY LOW forever?

### Short answers

1. To isolate low-bandwidth control traffic and reduce peripheral-interface complexity.
2. Two cycles: one Setup and one Access, with no wait states.
3. Setup selects/presents the transfer; Access performs/completes it.
4. During Access; it is sampled for completion when PSEL and PENABLE are HIGH.
5. PSEL && PENABLE && PREADY sampled at a rising PCLK edge.
6. So the completer sees one coherent transfer throughout the Access phase.
7. Yes, on either reads or writes.
8. No. The side effect is peripheral-specific.
9. One strobe bit per byte lane.
10. To carry access protection attributes.
11. No. A completer/firewall must enforce policy.
12. The bridge can buffer upstream traffic while serializing APB transfers.
13. The completer can miss Setup or fail to complete the Access phase.
14. It lets power/clock logic wake the APB domain; failure to assert it can deadlock a completer that waits for wake-up.
15. It detects selected single-bit signal corruption; it does not correct the data or provide complete end-to-end protection alone.
16. Optional Root/Realm address-space distinction.
17. APB has no burst or multiple outstanding data-transfer model.
18. Trace PSEL/PENABLE/PREADY, then inspect PCLK, reset, power, internal busy logic and wake-up.

## 20. Reference and source notes

### Normative protocol source

1. Arm, **AMBA APB Protocol Specification, ARM IHI 0024E, ID022823**, Issue E, 28 February 2023. This is the primary source for APB2/APB3/APB4/APB5 feature history, transfer timing, PSTRB, PPROT, PWAKEUP, user signaling, parity and RME support.  
   https://documentation-service.arm.com/static/63fe2c1356ea36189d4e79f3

2. Arm, **AMBA specifications index**. Official index of current and prior AMBA protocol families.  
   https://www.arm.com/architecture/system-architectures/amba/amba-specifications

3. Arm, **AMBA 4 / AMBA 3 / AMBA 2 overview**. Historical context for APB3, AHB and AXI generations.  
   https://www.arm.com/architecture/system-architectures/amba/amba-4

4. Arm, **Introduction to AMBA AXI4**, section describing AMBA generation history and APB's low-bandwidth register-access role.  
   https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf

### How to use the sources

- Use IHI 0024E for normative APB signal behavior and optional-interface compatibility.
- Use the Arm AMBA overview for generation-level history.
- The RTL in this lesson is an original educational example, not copied from Arm or a vendor implementation.
- Do not copy specification figures or tables into the repository. The summaries and diagrams here are original.
- Check the specification license and your organization's IP policy before using any vendor specification content beyond permitted reference and implementation.

---

## Next lesson

**Lesson 04 — AHB / AHB-Lite protocol deep dive**

Planned topics: address/data pipelining, HTRANS, HREADY, HRESP, HADDR, HWDATA, HRDATA, burst types, wait states, arbitration, AHB-Lite simplification, APB bridge comparison, RTL timing, assertions, verification and post-silicon debug.
