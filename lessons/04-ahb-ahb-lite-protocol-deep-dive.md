# Lesson 04 — AMBA AHB and AHB-Lite: Protocol, RTL Architecture and Silicon-Level Debug

**Track:** SoC Basics → Masterclass  
**Level:** Bus protocol fundamentals through RTL integration and silicon validation  
**Estimated study time:** 60 minutes  
**Prerequisites:** Lessons 01–03; synchronous digital design, memory-mapped registers and basic APB  
**Outcome:** Explain the AHB design rationale, follow pipelined address/data phases, generate legal transfers and bursts, implement a simple AHB-Lite register slave, integrate an AHB-to-APB bridge, and debug protocol failures.

> **Source discipline:** This lesson is an original teaching explanation based primarily on Arm AMBA Specification Rev 2.0 (IHI 0011A, first released 13 May 1999) and the publicly available AMBA 5 AHB Protocol Specification IHI 0033B.b. Arm's specification portal currently lists Version C as the latest AHB specification. Always check the revision and declared interface properties used by the actual IP in your project. AHB-Lite and AHB5 are not interchangeable labels for every optional signal configuration.

---

## 1. One-hour learning plan

| Time | Topic | Expected outcome |
|---:|---|---|
| 0–6 min | Why AHB exists and its history | Explain the move from a simple system bus to a pipelined high-performance bus |
| 6–14 min | AHB, AHB-Lite and AHB5 | Identify the architectural differences and optional capabilities |
| 14–25 min | Signals and two-phase pipeline | Explain address acceptance, data completion and HREADY |
| 25–34 min | HTRANS, HSIZE, HBURST and responses | Generate legal singles, bursts, wait states and errors |
| 34–42 min | Interconnect and AHB-to-APB bridge | Understand decode, response muxing and protocol conversion |
| 42–50 min | RTL register slave | Implement a simple 32-bit AHB-Lite target |
| 50–56 min | Performance, integration and verification | Calculate throughput and build a useful test matrix |
| 56–60 min | Silicon debug and exercises | Diagnose common bus failures |

## 2. Why AHB exists: the engineering backstory

A processor-based SoC needs a communication backbone between CPU cores, on-chip SRAM, external memory controllers, DMA engines and peripheral subsystems. A simple peripheral bus is not sufficient for all of these paths: memory traffic benefits from pipelining, bursts and the ability to share a bus among multiple requesters.

The original AMBA architecture deliberately separated high-performance system traffic from lower-bandwidth peripheral traffic. Arm's 1999 AMBA specification describes AHB as a high-performance, high-clock-frequency system backbone connecting processors, on-chip memories and external memory interfaces. It shows a bridge from AHB to APB, where lower-bandwidth peripherals such as UARTs, timers and parallel I/O reside.

### 2.1 Dated milestones

| Date / generation | Documented milestone | Engineering significance |
|---|---|---|
| 13 May 1999 | AMBA Specification Rev 2.0, IHI 0011A, first release | Defines AHB, ASB and APB; AHB is the high-performance pipelined backbone |
| 1999 | AMBA 2.0 announced publicly | AHB supports high-speed single-edge synchronous operation, burst transfers and multiple bus masters |
| 6 June 2006 | AMBA 3 AHB-Lite Issue A, first release | Defines the simplified single-master AHB-Lite interface |
| 25 June 2015 | AHB5 Issue B.a update; public release B.b on 30 October 2015 | Adds property-controlled capabilities such as secure transfers, extended memory types, exclusives, endianness and atomicity |
| Current documentation portal | Arm lists AHB Protocol Specification Version C as latest | The protocol continues to be maintained; use the latest licensed/current document for a new compliance project |

### 2.2 What problem did pipelining solve?

A non-pipelined bus must finish one complete address-plus-data operation before beginning the next address. That wastes cycles when address decode and data movement can overlap.

AHB separates each transfer into:

1. **Address phase:** address and control are presented for one HCLK cycle, unless the previous transfer is stretching the pipeline.
2. **Data phase:** read or write data is transferred; the selected slave can extend this phase using HREADY/HREADYOUT.

The address phase of transfer B overlaps the data phase of transfer A. This is the central performance idea.

### 2.3 Why the protocol evolved into AHB-Lite

The original AMBA 2 AHB supports multiple masters on a shared bus. It therefore needs arbitration signals and has response mechanisms such as SPLIT and RETRY. These features are useful when a slow target must release the shared bus so another master can make progress, but they increase the complexity of both the arbiter and the target.

AHB-Lite simplifies the interface to one bus master per AHB-Lite layer. A multi-master SoC can use a BusMatrix or multi-layer interconnect to create independent paths between masters and slaves. This moves concurrency into the interconnect topology rather than requiring every AHB-Lite interface to arbitrate among several masters.

**Engineering interpretation:** This is an architectural trade-off, not merely a renamed signal list. AHB-Lite removes shared-bus arbitration complexity from each layer and lets the system fabric provide parallelism. It is attractive for microcontrollers and embedded subsystems where predictable low latency and modest area matter more than AXI's independent channels and many outstanding transactions.

### 2.4 Why AHB5?

AHB5 keeps the AHB-Lite style of address/data transfer while adding optional capabilities declared as interface properties. Examples include secure/non-secure transfer indication, extended memory types, exclusive accesses, endianness definition, stable signaling between clock edges and atomicity properties.

A key design principle is that an optional property is not automatically present. If a property is not declared for an interface, the corresponding capability must not be assumed by the connected component.

---

## 3. Where AHB sits inside a real SoC

~~~mermaid
flowchart LR
  CPU["CPU / instruction and data ports"] --> MUX["AHB master interface"]
  DMA["DMA engine"] --> M2["AHB master interface"]
  MUX --> FAB["AHB BusMatrix / multi-layer fabric"]
  M2 --> FAB
  FAB --> SRAM["On-chip SRAM"]
  FAB --> ROM["Boot ROM"]
  FAB --> MEM["External memory controller"]
  FAB --> BR["AHB-to-APB bridge"]
  BR --> APB["APB subsystem"]
  APB --> UART["UART"]
  APB --> TIMER["Timer / watchdog"]
  APB --> GPIO["GPIO / pin control"]
~~~

Typical use cases include:

- Cortex-M-class processor systems and microcontroller subsystems.
- CPU-to-SRAM and CPU-to-ROM paths.
- DMA access to memory-mapped buffers.
- AHB-to-APB bridges.
- Low-latency control fabrics inside DSP, sensor and always-on subsystems.
- AHB5 embedded systems that need optional security or exclusive-access signaling.

AHB is not a cache-coherent protocol by itself. AHB5 adds certain memory and atomicity properties, but a complete cache-coherent system still needs a coherent architecture and system-level rules.

## 4. AHB variants: do not mix their signal sets

| Capability | AMBA 2 AHB | AHB-Lite | AHB5 |
|---|---|---|---|
| Address/data pipelining | Yes | Yes | Yes |
| Multiple masters on one shared bus | Yes | No, one master per layer | One master per AHB-Lite-style layer |
| Arbitration request/grant | Yes | Supplied by external fabric if needed | Supplied by external fabric if needed |
| Fixed and incrementing bursts | Yes | Yes | Yes |
| SPLIT / RETRY responses | Yes | No | No in the AHB-Lite-style response model |
| Basic OKAY / ERROR response | Yes, encoded in HRESP[1:0] | Yes, one-bit HRESP | Yes, one-bit HRESP |
| Secure transfer attribute | System-specific legacy protection | Not in the base AHB-Lite signal set | Optional Secure_Transfers property |
| Exclusive accesses | Legacy locked transfers | No AHB5 exclusive extension by default | Optional Exclusive_Transfers property |
| Extended memory types | Limited legacy HPROT | Limited legacy HPROT | Optional Extended_Memory_Types property |
| User signaling | Implementation-specific | Not part of the base interface | Optional user signaling |

**Important:** AHB-Lite is not simply "AHB with fewer wires." Its response and arbitration behavior differ. When integrating legacy AHB, AHB-Lite and AHB5 IP, inspect the exact interface declaration, supported properties, data width and bridge behavior.

## 5. AHB signal reference

The following table describes a common 32-bit AHB-Lite interface. AHB5 can add optional signals. A full AMBA 2 AHB multi-master bus also includes arbitration signals.

### 5.1 Global and master-driven signals

| Signal | Direction / source | Meaning |
|---|---|---|
| HCLK | Clock source → all bus components | All transfers are timed relative to the rising edge |
| HRESETn | Reset controller → bus components | Active-low bus reset |
| HADDR | Master → decoder and slaves | Address phase address |
| HTRANS[1:0] | Master → slaves | IDLE, BUSY, NONSEQ or SEQ |
| HWRITE | Master → slaves | HIGH for write, LOW for read |
| HSIZE[2:0] | Master → slaves | Transfer size in bytes/bits encoding |
| HBURST[2:0] | Master → slaves | Single, incrementing or wrapping burst type |
| HPROT | Master → slaves | Legacy protection/access attributes |
| HWDATA | Master → slaves | Write data for the data phase of the preceding accepted address |
| HMASTLOCK | Master → interconnect/slaves | Indicates a locked transfer sequence where supported |

### 5.2 Slave and interconnect response signals

| Signal | Direction / source | Meaning |
|---|---|---|
| HSELx | Decoder → slave x | Selects the slave for the current address phase |
| HREADY | Interconnect → master and slaves | HIGH means the previous data phase has completed |
| HREADYOUT | Slave → interconnect | Slave's ready indication for its current data phase |
| HRDATA | Selected slave → master | Read data for the data phase |
| HRESP | Selected slave → master | OKAY or ERROR in AHB-Lite/AHB5 |
| HEXOKAY | Slave/interconnect → master | Optional AHB5 exclusive-access result |

### 5.3 Full AHB arbitration signals

| Signal | Role |
|---|---|
| HBUSREQx | Master x requests the shared bus |
| HGRANTx | Arbiter grants the bus to master x |
| HMASTER | Identifies the currently granted master in the interconnect |
| HSPLITx | Split-capable slave tells the arbiter which master can retry |

These signals belong to the full multi-master AHB architecture. Do not add them to a basic AHB-Lite master/slave port merely because a legacy AHB diagram contains them.

---

## 6. The central AHB concept: address phase versus data phase

AHB's pipeline is the most important concept in this lesson.

Consider two transfers:

- Transfer A: write 0x11223344 to address 0x40000000.
- Transfer B: read from address 0x40000004.

| Cycle | Address/control phase | Data phase | HREADY |
|---|---|---|---:|
| C0 | Address A, WRITE | Previous transfer / idle | 1 |
| C1 | Address B, READ | Write data for A | 1 |
| C2 | Address C or IDLE | Read data for B | 1 |

The master presents the address and control for B while the data phase for A is active. The data phase belongs to the address accepted one cycle earlier, not the address currently visible on HADDR.

### 6.1 Why write data appears one phase later

AHB has a shared address/control phase and a separate data phase. HWDATA for transfer A is driven during A's data phase, which follows the cycle in which A's address/control were accepted.

This means a slave must pipeline its write control and address. It must not decode the current HADDR and assume the current HWDATA belongs to that same address.

### 6.2 HREADY is the pipeline enable

When HREADY is HIGH at the active clock edge:

- The current data phase completes.
- The address/control phase can advance and a valid next transfer can be accepted.

When HREADY is LOW:

- The current data phase is extended.
- The address/control phase of the next transfer is also held.
- The write data associated with the pending data phase must remain stable.
- The master/interconnect cannot treat the stalled next address as an accepted transfer.

This is why a wait state in one slave can stall the entire shared AHB-Lite layer.

### 6.3 One wait-state example

| Cycle | HADDR | HTRANS | HWDATA | HREADY | Interpretation |
|---|---|---|---|---:|---|
| C0 | A | NONSEQ | — | 1 | A address accepted |
| C1 | B | NONSEQ | Data(A) | 0 | A is waiting; B is not yet accepted |
| C2 | B held | NONSEQ held | Data(A) held | 0 | Second wait cycle |
| C3 | B held | NONSEQ held | Data(A) held | 1 | A completes and B is accepted |
| C4 | C | NONSEQ/SEQ | Data(B) | 1 | B completes; C can be accepted |

The exact timing of a slave's HREADYOUT depends on its internal implementation. The key is that the master sees a single combined HREADY from the interconnect.

**Do not confuse HREADY with HTRANS.** HTRANS says whether the address/control phase represents a valid transfer. HREADY says whether the previous data phase has completed and the pipeline can advance.

---

## 7. HTRANS: the four transfer types

HTRANS[1:0] identifies the current address phase.

| HTRANS | Name | Meaning |
|---:|---|---|
| 00 | IDLE | No transfer is required |
| 01 | BUSY | Master is temporarily unable to continue a burst |
| 10 | NONSEQ | Single transfer or first transfer of a burst |
| 11 | SEQ | Remaining transfer in a burst |

### 7.1 IDLE

No bus transfer is requested. A slave must ignore the address/control as a valid access. A robust slave returns a zero-wait OKAY response for an IDLE phase.

### 7.2 BUSY

The master intends to continue the current burst but needs to insert a cycle before the next beat. BUSY does not transfer data. A slave must ignore the BUSY transfer and provide a zero-wait OKAY response.

### 7.3 NONSEQ

NONSEQ marks a single transfer or the first beat of a burst. It establishes the start address and control attributes.

### 7.4 SEQ

SEQ marks the subsequent beats of a burst. The address is related to the preceding transfer and normally advances by the transfer size. For a wrapping burst, the address wraps at the boundary determined by burst length multiplied by transfer size.

### 7.5 Valid transfer detection

For a conventional AHB-Lite slave, a valid address phase is:

~~~~systemverilog
wire ahb_valid = HSEL && HREADY && HTRANS[1];
~~~~

HTRANS[1] is HIGH for NONSEQ and SEQ. The HREADY qualification is essential: when the pipeline is stalled, the address phase is not accepted again on every clock.

A common bug is using only HSEL && HTRANS[1] as a write enable. If HREADY remains LOW for several cycles, the slave can capture or commit the same operation repeatedly.

---

## 8. HSIZE: transfer size and byte lanes

HSIZE[2:0] describes the size of one transfer.

| HSIZE | Transfer size | Name |
|---:|---:|---|
| 000 | 8 bits | Byte |
| 001 | 16 bits | Halfword |
| 010 | 32 bits | Word |
| 011 | 64 bits | Doubleword |
| 100 | 128 bits | 4-word line |
| 101 | 256 bits | 8-word line |
| 110 | 512 bits | — |
| 111 | 1024 bits | — |

The transfer size must not exceed the width supported by the interface. A 32-bit data bus therefore supports byte, halfword and word transfers, not a 64-bit transfer.

### 8.1 Alignment

For naturally aligned transfers:

- Byte: any byte address.
- Halfword: address bit 0 is zero.
- Word: address bits [1:0] are zero.
- Doubleword: address bits [2:0] are zero.

The actual data-lane mapping depends on bus width and endianness. A narrow transfer on a wider data bus uses the address and transfer size to determine the active byte lanes.

**Common integration bug:** A bridge converts a 32-bit AHB write into a 32-bit peripheral write but ignores HSIZE and address low bits. Byte and halfword writes then corrupt neighboring registers.

---

## 9. HBURST: single, incrementing and wrapping bursts

HBURST[2:0] describes the burst type.

| HBURST | Type | Beats |
|---:|---|---:|
| 000 | SINGLE | 1 |
| 001 | INCR | Undefined length |
| 010 | WRAP4 | 4 |
| 011 | INCR4 | 4 |
| 100 | WRAP8 | 8 |
| 101 | INCR8 | 8 |
| 110 | WRAP16 | 16 |
| 111 | INCR16 | 16 |

### 9.1 Incrementing burst

Each beat accesses the next sequential address.

For a word-wide INCR4 beginning at 0x1000:

| Beat | HTRANS | HADDR |
|---:|---|---:|
| 0 | NONSEQ | 0x1000 |
| 1 | SEQ | 0x1004 |
| 2 | SEQ | 0x1008 |
| 3 | SEQ | 0x100C |

### 9.2 Wrapping burst

A wrapping burst wraps at a boundary equal to:

**Boundary = Number of beats × Bytes per beat**

Example: WRAP4 with 32-bit words has a 16-byte boundary. Starting at 0x34, the addresses are:

0x34 → 0x38 → 0x3C → 0x30

This is useful for cache-line or circular-buffer style access patterns where the address sequence is constrained to a fixed region.

### 9.3 The 1 KB boundary rule

AHB masters must not start an incrementing burst that crosses a 1 KB address boundary. This allows a system decoder to use a 1 KB region as a minimum decode granularity for many conventional AHB arrangements.

For example, a burst starting at 0x3F8 with word transfers must be planned so that it does not continue across 0x400. A DMA or cache-line refill engine should split such a request into legal bursts.

### 9.4 BUSY and fixed-length bursts

BUSY can be inserted within an undefined-length incrementing burst to pause between beats. Fixed-length INCR4/8/16 and WRAP4/8/16 bursts must terminate with the required final SEQ beat; a master must not use BUSY as the final beat of those fixed-length bursts.

---

## 10. HRESP and wait-state behavior

### 10.1 AHB-Lite / AHB5 response

AHB-Lite and AHB5 use a one-bit HRESP:

| HRESP | Meaning |
|---:|---|
| 0 | OKAY |
| 1 | ERROR |

The response is combined with HREADY/HREADYOUT:

- HRESP=0, HREADYOUT=0: transfer pending; wait state.
- HRESP=0, HREADYOUT=1: successful completion.
- HRESP=1, HREADYOUT=0: first cycle of an ERROR response.
- HRESP=1, HREADYOUT=1: second cycle; ERROR completes.

The two-cycle ERROR response is important because the next address may already have been placed on the bus while the current transfer's data phase is in progress. The first ERROR cycle gives the master time to cancel or control the next address phase.

### 10.2 Legacy AMBA 2 AHB response

Full AMBA 2 AHB uses HRESP[1:0]:

| HRESP[1:0] | Response | Meaning |
|---:|---|---|
| 00 | OKAY | Successful or pending response, qualified by HREADY |
| 01 | ERROR | Transfer failed |
| 10 | RETRY | Master should retry the transfer |
| 11 | SPLIT | Slave asks the arbiter to defer this master until it is ready |

ERROR, RETRY and SPLIT require a two-cycle response. The protocol uses the extra cycle to allow the pipelined next address to be cancelled when required.

### 10.3 RETRY versus SPLIT

- **RETRY:** The master must retry the transfer. The arbiter may grant the bus according to its normal arbitration policy.
- **SPLIT:** The slave records the master identity and later signals the arbiter through HSPLIT when it can complete the operation. The arbiter masks the split master's request until the slave is ready, allowing other masters to use the bus.

SPLIT is especially useful for a high-latency slave that should not hold a shared bus while waiting for an external event. AHB-Lite removes this response mechanism; a modern AHB-Lite layer generally uses a bounded wait-state response or an interconnect architecture that provides other independent paths.

**Important:** Do not connect a full AHB slave that can return SPLIT/RETRY directly to an AHB-Lite response port without a deliberate compatibility wrapper.

---

## 11. AHB interconnect: decoder, response mux and HREADY aggregation

A simple AHB system has a shared address/control path and a selected response path.

~~~mermaid
flowchart TB
  M["AHB master"] --> A["Address and control fanout"]
  A --> D["Address decoder"]
  D --> S0["Slave 0"]
  D --> S1["Slave 1"]
  D --> S2["Slave 2"]
  S0 --> R["Data / response mux"]
  S1 --> R
  S2 --> R
  R --> M
  S0 --> H["HREADY aggregation"]
  S1 --> H
  S2 --> H
  H --> M
~~~

### 11.1 Address decode

The decoder examines the address phase and asserts one HSELx signal. The selected slave must use HREADY to determine when the preceding transfer has completed before it accepts a new transfer.

### 11.2 Read-data and response multiplexing

All slaves can observe the address/control bus, but only the selected slave should return the response for the active data phase. The interconnect must delay or register the slave-select information so the read-data/response mux selects the slave corresponding to the previous accepted address, not the address currently visible.

This is a classic source of off-by-one-cycle bugs.

### 11.3 HREADYOUT versus HREADY

Each slave has its own HREADYOUT. The interconnect selects the HREADYOUT for the slave whose data phase is active and drives the shared HREADY to the master and all slaves.

Do not simply AND every slave's HREADYOUT together in a single-slave-select system. An unselected slow slave must not stall the bus. In a multi-layer fabric, each path/layer has its own ready/response control.

### 11.4 Why response mux select must be retimed

Suppose the address phase selects SRAM at cycle N, and the next address phase selects UART at cycle N+1. During cycle N+1, the response is still for SRAM while the decoder is already looking at UART's address. If the response mux uses the current combinational HSEL, it can route the SRAM response from the wrong source or route UART's stale response to the master.

The interconnect therefore stores the selection associated with the accepted address phase and uses that stored selection for the corresponding data phase.

---

## 12. AHB-to-APB bridge: how the two protocols meet

AHB and APB have different timing models:

- AHB overlaps address and data phases.
- APB has Setup followed by Access.
- AHB can issue a burst.
- APB performs one transfer at a time.
- AHB's HREADY/HRESP response must be translated to APB's PREADY/PSLVERR.

A typical bridge behaves as follows:

~~~mermaid
stateDiagram-v2
  [*] --> AHB_IDLE
  AHB_IDLE --> CAPTURE: Accept AHB address
  CAPTURE --> APB_SETUP: Hold address, direction, write data
  APB_SETUP --> APB_ACCESS: Assert PENABLE
  APB_ACCESS --> APB_ACCESS: PREADY = 0
  APB_ACCESS --> AHB_RESPONSE: PREADY = 1
  AHB_RESPONSE --> AHB_IDLE: Return OKAY / ERROR
  AHB_RESPONSE --> CAPTURE: Next AHB transfer accepted
~~~

### 12.1 Bridge sequence

1. Accept the AHB address/control when the AHB pipeline permits it.
2. Capture address, direction, transfer size and relevant protection attributes.
3. Capture the write data in the correct AHB data phase.
4. Drive APB Setup: PSEL=1, PENABLE=0.
5. Drive APB Access: PSEL=1, PENABLE=1.
6. Hold APB signals stable while PREADY=0.
7. When PREADY=1, capture PRDATA or PSLVERR.
8. Complete the AHB data phase with HREADYOUT=1 and the appropriate HRESP.
9. If another AHB request is pending, begin the next APB transfer without losing the pipeline state.

### 12.2 Common bridge bugs

- Using current HADDR with previous HWDATA.
- Dropping PSTRB or HPROT/PPROT information.
- Completing AHB before APB completes.
- Returning APB read data one cycle too early or too late.
- Mapping PSLVERR to a one-cycle AHB ERROR instead of the required two-cycle response.
- Failing to preserve the AHB write data during APB wait states.
- Allowing a second AHB request to overwrite the bridge's single-entry request register.
- Crossing between HCLK and PCLK without a proper asynchronous bridge.

---

## 13. A minimal AHB-Lite register slave in SystemVerilog

The following teaching example implements a simple 32-bit, zero-wait-state register target. It assumes legal, aligned 32-bit accesses to a small register window. It deliberately does not implement error responses, sub-word writes, bursts with side effects, security filtering or AHB5 optional properties.

The key idea is to capture the address/control in the address phase and use the captured address with HWDATA in the following data phase.

~~~~systemverilog
module ahb_lite_regs (
    input  logic        HCLK,
    input  logic        HRESETn,

    input  logic        HSEL,
    input  logic        HREADY,
    input  logic [31:0] HADDR,
    input  logic [1:0]  HTRANS,
    input  logic        HWRITE,
    input  logic [2:0]  HSIZE,
    input  logic [31:0] HWDATA,

    output logic [31:0] HRDATA,
    output logic        HREADYOUT,
    output logic        HRESP
);

    logic [31:0] ctrl_reg;
    logic [31:0] div_reg;
    logic [31:0] status_reg;

    logic [31:0] addr_q;
    logic        write_q;

    logic transfer_accept;

    // NONSEQ and SEQ are valid transfers.
    // HREADY qualifies acceptance of the address phase.
    assign transfer_accept = HSEL && HREADY && HTRANS[1];

    // Teaching example: no wait states and no errors.
    assign HREADYOUT = 1'b1;
    assign HRESP     = 1'b0;

    // Read data belongs to the address captured in the previous
    // accepted address phase.
    always_comb begin
        unique case (addr_q[7:2])
            6'h00:   HRDATA = ctrl_reg;
            6'h01:   HRDATA = div_reg;
            6'h02:   HRDATA = status_reg;
            default: HRDATA = 32'h0000_0000;
        endcase
    end

    always_ff @(posedge HCLK or negedge HRESETn) begin
        if (!HRESETn) begin
            ctrl_reg   <= 32'h0000_0000;
            div_reg    <= 32'h0000_0001;
            status_reg <= 32'h0000_0000;
            addr_q     <= 32'h0000_0000;
            write_q    <= 1'b0;
        end else if (HREADY) begin

            // Complete the previous write data phase.
            if (write_q) begin
                unique case (addr_q[7:2])
                    6'h00: ctrl_reg <= HWDATA;
                    6'h01: div_reg  <= HWDATA;
                    default: ; // Read-only or unmapped register.
                endcase
            end

            // Capture the next address phase only when accepted.
            if (transfer_accept) begin
                addr_q  <= HADDR;
                write_q <= HWRITE;
            end else begin
                write_q <= 1'b0;
            end
        end
        // If HREADY is LOW, preserve the pending transfer state.
    end

    // Example-only legality check; a production block must define
    // its response policy for unsupported sizes and addresses.
    property p_word_aligned_access;
        @(posedge HCLK) disable iff (!HRESETn)
        transfer_accept |-> (HSIZE == 3'b010 && HADDR[1:0] == 2'b00);
    endproperty

    assert property (p_word_aligned_access);

endmodule
~~~~

### 13.1 Walk through the write pipeline

Suppose software writes 0x00000005 to the CTRL register at offset 0x00.

- Address phase: HSEL=1, HTRANS=NONSEQ, HWRITE=1, HADDR points to CTRL, HREADY=1.
- At the accepting edge: addr_q captures CTRL and write_q captures 1.
- Data phase: HWDATA carries 0x00000005.
- At the completion edge: write_q is 1, so ctrl_reg captures HWDATA.
- If HREADY had been LOW, the pending write would have remained pending and the data must be held stable until completion.

### 13.2 What this example intentionally omits

A production-quality AHB-Lite target should also consider:

- Correct ERROR response timing for unmapped or unsupported accesses.
- Address-size and byte-lane handling.
- Read-only, write-only and write-one-to-clear register semantics.
- Reset values and reset-domain behavior.
- HPROT and AHB5 HNONSEC security enforcement where supported.
- Optional exclusive-access behavior if the interface advertises it.
- Register side effects and interrupt clear/set races.
- Clock-domain crossings for registers implemented in another clock domain.
- Parameterized data widths and legal HSIZE support.
- Response stability under wait states.
- Formal and simulation checks for the complete protocol.

The code is a small learning model, not a drop-in compliant IP block.

---

## 14. Clock, reset, power and CDC integration

### 14.1 Clocking

AHB transfers are synchronous to HCLK. Address/control and response timing must meet setup and hold requirements around the active clock edge.

A high-frequency AHB path can become timing-critical because HADDR fans out to the decoder and many slave ports. The response mux and HREADY path can also become long combinational paths.

### 14.2 Reset

HRESETn is active LOW. In the original AMBA 2 specification, reset may be asserted asynchronously but must be deasserted synchronously after the rising edge of HCLK. All masters must drive a valid reset state, including HTRANS=IDLE.

Reset must not leave a slave's HREADYOUT or response outputs in an unknown state. A slave that holds HREADYOUT LOW after reset can deadlock the entire AHB layer.

### 14.3 Clock-domain crossings

AHB is a synchronous bus protocol. If an AHB manager and target operate on unrelated clocks, use a dedicated asynchronous AHB bridge or a well-defined clock-domain crossing wrapper.

Do not directly synchronize every bit of HADDR, HWDATA, HTRANS, HREADY and HRESP independently. A multi-bit bus transaction needs a coherent handshake, not independent two-flop synchronizers.

### 14.4 Power gating

If an AHB target can be power-gated, the system must define what happens to accesses while it is off:

- Block or firewall accesses before the target is isolated.
- Ensure the interconnect receives a defined response.
- Use wake-up sequencing if access should power the target.
- Prevent the target's unpowered outputs from corrupting HREADY/HRESP/HRDATA.
- Preserve or restore register state according to the retention design.

---

## 15. Performance calculations

### 15.1 Peak theoretical bandwidth

For a 32-bit AHB data bus at 100 MHz:

- Data width = 4 bytes.
- One beat per cycle in the ideal steady-state pipeline.
- Clock frequency = 100 million cycles per second.

Peak theoretical bandwidth:

**4 bytes × 100 MHz = 400 MB/s**

This is a protocol-level upper bound under ideal conditions. It does not include memory latency, wait states, arbitration gaps, bridge overhead or target limitations.

### 15.2 Effect of wait states

If each word transfer consumes one data cycle plus two wait cycles, the sustained transfer rate becomes:

**4 bytes / 3 cycles × 100 MHz = 133.3 MB/s**

This assumes the same shared bus is stalled for every transfer and there are no other overheads.

### 15.3 Burst efficiency

For a long, continuous burst with no wait states, the bus can approach one beat per cycle after pipeline startup.

For a finite four-beat burst, include the initial pipeline fill when calculating end-to-end time. A simple approximation is five bus cycles from first address presentation to final data completion, so:

**16 bytes / 5 cycles × 100 MHz = 320 MB/s**

The sustained rate of a long stream is higher because the pipeline startup cost is amortized.

### 15.4 Latency versus throughput

AHB can have low first-beat latency while still achieving one transfer per cycle in the steady state. AHB is pipelined, but it does not provide the independent address/data channels and many outstanding transaction IDs of AXI.

If one slow slave holds HREADY LOW on a single shared layer, it stalls all transfers on that layer. A multi-layer interconnect can isolate independent paths and reduce this head-of-line blocking.

---

## 16. Pre-silicon verification strategy

A useful AHB-Lite verification plan should cover protocol legality, data correctness, error handling and system integration.

### 16.1 Directed tests

1. Single read and write.
2. Back-to-back read/read, write/write and read/write.
3. NONSEQ followed by SEQ burst.
4. BUSY insertion in a legal undefined-length burst.
5. IDLE cycles between transfers.
6. One, two and many wait states.
7. Read and write response with HREADYOUT stalls.
8. Error response with the required two-cycle timing.
9. Burst crossing attempt at a 1 KB boundary.
10. Byte, halfword and word accesses for supported bus widths.
11. Reset during idle and reset after a completed transfer.
12. Unmapped address and default-slave behavior.
13. Concurrent masters on separate BusMatrix layers.
14. Arbitration fairness for full AHB.
15. SPLIT and RETRY for legacy AHB only.
16. HNONSEC and exclusive sequences for AHB5 interfaces that declare those properties.

### 16.2 Assertions and protocol checks

Illustrative properties for a basic AHB-Lite environment:

~~~~systemverilog
// A valid accepted transfer must be NONSEQ or SEQ.
assert property (@(posedge HCLK) disable iff (!HRESETn)
    (HSEL && HREADY && HTRANS[1]) |-> (HTRANS inside {2'b10, 2'b11})
);

// If a valid address phase is stalled by HREADY, its control
// must remain stable for the wait cycle. Exclude an ERROR
// response cycle, where the master can cancel the next address.
assert property (@(posedge HCLK) disable iff (!HRESETn)
    (HTRANS[1] && !HREADY && !HRESP)
    |=> $stable({HADDR, HWRITE, HSIZE, HBURST, HPROT, HTRANS})
);

// A write data phase held by a wait state must keep its data stable.
// In a full checker, qualify this with a tracked accepted write phase.
assert property (@(posedge HCLK) disable iff (!HRESETn)
    (write_data_pending && !HREADY) |=> $stable(HWDATA)
);
~~~~

These are examples, not a complete protocol checker. A complete checker must track the accepted address phase and the associated data phase, handle reset and response exceptions, and use the correct signal set for the interface revision.

### 16.3 Scoreboard model

The scoreboard should maintain:

- Accepted address and control.
- Expected data-phase association.
- Expected read data.
- Expected write side effects.
- Current burst address and remaining beat count.
- Wait-state count.
- Response status.
- Security/exclusive attributes where present.

For every accepted transfer, verify exactly one completion or one legal error outcome. Never count an address phase as completed merely because HTRANS is NONSEQ or SEQ.

---

## 17. Post-silicon bring-up and debug

### 17.1 First bring-up checklist

1. Confirm HCLK frequency and duty cycle.
2. Confirm HRESETn assertion and synchronous deassertion.
3. Check that master drives HTRANS=IDLE during reset.
4. Confirm HREADY is HIGH when the system is idle and reset is released.
5. Read a known ROM or SRAM location.
6. Write and read back a scratch SRAM location.
7. Access one APB register through the AHB-to-APB bridge.
8. Run back-to-back transfers.
9. Exercise a short burst if supported by the master.
10. Enable performance counters or bus trace if available.

### 17.2 Common failure signatures

| Symptom | Likely cause | What to probe |
|---|---|---|
| CPU hangs on first peripheral read | HREADY stuck LOW, wrong HSEL, bridge FSM stuck | HADDR, HSEL, HREADYOUT, HREADY, bridge state |
| Read returns previous register value | Response mux select not retimed or address/data phase misaligned | Accepted HADDR, delayed select, HRDATA |
| Writes go to the wrong register | Current HADDR used with previous HWDATA | Captured address/control and HWDATA |
| Write occurs twice | Write enable not qualified by HREADY/completion | Write-fire pulse and pending state |
| Only bursts fail | Incorrect HTRANS/HTRANS progression, address increment, boundary split | HTRANS, HSIZE, HBURST, HADDR |
| Fails only with wait states | Master or slave changes control/data while stalled | HREADY, HREADYOUT, HWDATA, address/control stability |
| AHB-to-APB access never completes | PREADY not propagated, PSEL/PENABLE sequence incorrect | APB FSM, PREADY, AHB HREADYOUT |
| Error response corrupts next transfer | Incorrect two-cycle ERROR handling | HRESP, HREADY, HTRANS cancellation |
| Secure access reaches wrong target | HNONSEC lost or misinterpreted in AHB5 bridge | HNONSEC, firewall decode, bridge sidebands |
| Rare hang under contention | Arbiter starvation, lock held too long, shared-layer bottleneck | Request/grant, HMASTLOCK, HREADY, arbitration trace |

### 17.3 A practical trace method

Capture these signals together in one trace window:

- HCLK, HRESETn
- HADDR, HTRANS, HWRITE, HSIZE, HBURST
- HSELx
- HWDATA, HRDATA
- HREADY, HREADYOUT
- HRESP
- HMASTLOCK and arbitration signals if using full AHB
- HNONSEC/HEXCL/HEXOKAY if using AHB5 features

Start by identifying the address acceptance edge. Then label the following data phase. This simple annotation catches a large fraction of AHB debug mistakes.

---

## 18. Hands-on lab: build and validate an AHB-Lite peripheral

### Lab objective

Create a small AHB-Lite peripheral with four registers:

| Offset | Register | Access |
|---:|---|---|
| 0x00 | CTRL | Read/write |
| 0x04 | DIVIDER | Read/write |
| 0x08 | STATUS | Read-only |
| 0x0C | IRQ_CLEAR | Write-one-to-clear |

### Tasks

1. Implement a zero-wait-state AHB-Lite slave using the teaching code above.
2. Add a default response for unmapped addresses.
3. Add a proper two-cycle ERROR response for unsupported transfer sizes.
4. Add byte/halfword support with correct byte-lane updates.
5. Create an AHB-Lite bus functional model.
6. Test back-to-back reads and writes.
7. Add a two-cycle wait-state mode.
8. Assert that write data and the accepted address remain aligned during stalls.
9. Add a small AHB-to-APB bridge or connect the target behind an existing bridge.
10. Capture a waveform and annotate address/data phases.

### Expected waveform questions

- On which edge is the address accepted?
- Which transfer does the current HWDATA belong to?
- When is the register actually updated?
- What remains stable while HREADY is LOW?
- Which HSEL must drive the response mux?
- Does the slave return OKAY for IDLE and BUSY?
- Is ERROR asserted for the required two cycles?

### Stretch goals

- Add an AHB5 HNONSEC input and a simple secure-only register.
- Add an exclusive monitor for a small scratch-memory region.
- Add a timeout monitor for a target that never returns HREADYOUT.
- Measure arbitration latency with two masters on a shared AHB bus.
- Compare a single shared AHB layer with two independent AHB-Lite matrix layers.

---

## 19. Interview and design-review questions

1. Why does AHB have separate address and data phases?
2. Why is HWDATA associated with the previous accepted HADDR?
3. What exactly does HREADY mean?
4. What is the difference between HREADY and HREADYOUT?
5. Why must the interconnect retime HSEL for the response mux?
6. What is the difference between NONSEQ and SEQ?
7. What is the purpose of BUSY?
8. How do HSIZE and HBURST determine the address sequence?
9. Why must an incrementing burst not cross a 1 KB boundary?
10. Why does an AHB-Lite ERROR response take two cycles?
11. What is the difference between SPLIT and RETRY in full AHB?
12. Why did AHB-Lite remove arbitration and SPLIT/RETRY?
13. How can a BusMatrix provide parallelism when each AHB-Lite layer has one master?
14. How do you prevent a write from occurring twice during a wait state?
15. How do you translate APB PSLVERR into an AHB response?
16. Which signals must be held stable when HREADY is LOW?
17. What can go wrong if HSEL is not delayed for the data phase?
18. What additional checks are needed for AHB5 secure transfers?
19. How would you detect a target that permanently holds HREADYOUT LOW?
20. When would you choose AHB instead of AXI for a subsystem?

---

## 20. Key takeaways

- AHB's key innovation is the overlap of address and data phases.
- HREADY is the global pipeline advance/completion signal.
- HWDATA belongs to the accepted address from the preceding phase.
- HTRANS distinguishes IDLE, BUSY, NONSEQ and SEQ.
- HSIZE defines the beat size; HBURST defines the burst pattern.
- AHB-Lite simplifies the shared multi-master AHB model to one master per layer.
- AHB5 adds optional property-controlled capabilities; never assume optional sidebands exist.
- The response mux must select the slave associated with the data phase, not the current address phase.
- AHB-to-APB bridges must preserve the different timing and error semantics of both protocols.
- The most valuable debug habit is to label every accepted address with its corresponding data phase.

## 21. References and further reading

### Normative and primary sources

1. **Arm, AMBA Specification Rev 2.0, IHI 0011A, first release 13 May 1999.** Original AHB, ASB, APB, burst, arbitration, SPLIT/RETRY and bridge architecture.  
   https://documentation-service.arm.com/static/5f916403f86e16515cdc3d71

2. **Arm, AMBA 5 AHB Protocol Specification, IHI 0033B.b, 30 October 2015 public release.** AHB-Lite and AHB5 signals, transfer types, bursts, response timing, interconnect, secure transfers, exclusive transfers and atomicity.  
   https://documentation-service.arm.com/static/5f91607cf86e16515cdc3b4b

3. **Arm AHB specification portal.** The portal lists the latest specification revision and superseded versions.  
   https://developer.arm.com/documentation/ihi0033/c/

4. **Arm, Introduction to AMBA AXI4, 25 August 2020.** Useful for understanding where AHB sits relative to AXI and later AMBA generations.  
   https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf

### Historical and implementation context

5. **Arm Community, “New AMBA specification extends security to embedded design: AMBA 5 AHB5.”** Historical explanation of the 2015 AHB5 security and embedded-system motivation.  
   https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/new-amba-specification-extends-security-to-embedded-design-amba-5-ahb5

6. **Arm CoreLink SIE-200 System IP Technical Reference Manual.** Public implementation documentation for AHB5 BusMatrix, default slave, example slave, master/slave multiplexers, timeout monitor, SRAM and ROM interfaces.  
   https://developer.arm.com/documentation/ddi0571/latest/functional-description/ahb5-bus-matrix

7. **Arm Community discussion: AMBA AHB revisions.** Clarifies the practical distinction between original AHB, AHB-Lite and AHB5. Treat community discussion as explanatory context, not as the normative specification.  
   https://community.arm.com/forums/f/soc-design-and-simulation-forum/46395/amba-ahb/164581

---

**End of Lesson 04.** Next in sequence: **Lesson 05 — AMBA AXI Fundamentals: Five Channels, VALID/READY Handshakes, IDs, Outstanding Transactions and Response Ordering.**
