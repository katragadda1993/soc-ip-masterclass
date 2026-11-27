# Lesson 11 — SRAM, ROM and OTP: From Bitcell Physics to Secure SoC Memory Integration

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 11 of 38 · **Duration:** 60 minutes · **Level:** Intermediate to advanced

> **Scope and evidence discipline:** This lesson connects transistor-level memory behavior to SoC-level memory controllers. Exact bitcell topology, macro pin names, latency, collision behavior, ECC code, fuse technology, programming voltage, repair policy and register map are implementation- or process-specific. They are not universal protocol requirements. OpenTitan and OpenRAM are public examples, not claims about proprietary commercial implementations.

## 1. Learning objectives and lesson plan

By the end, you should be able to:

1. Explain why an SoC uses SRAM, ROM and OTP for different purposes.
2. Walk through read, write and hold operation of a conventional 6T SRAM cell.
3. Explain bitlines, wordlines, precharge, sense amplifiers, write drivers and banking.
4. Read an SRAM macro timing/interface contract and identify implementation-specific behavior.
5. Calculate capacity, address width, raw bandwidth and ECC overhead.
6. Distinguish mask ROM, fuse PROM, EPROM, EEPROM, flash and embedded OTP.
7. Explain ECC, integrity checking, scrambling, MBIST and repair.
8. Understand ROM integrity and OTP provisioning in secure boot and device lifecycle.
9. Build a memory verification plan and diagnose representative silicon failures.

| Time | Topic |
|---:|---|
| 0–7 min | Memory choices and historical evolution |
| 7–20 min | 6T SRAM cell, read/write physics and array |
| 20–30 min | Macro interfaces, timing, banking, ECC and scrambling |
| 30–39 min | ROM architecture, boot ROM and integrity |
| 39–46 min | OTP/eFuse, lifecycle partitions and provisioning |
| 46–54 min | MBIST/BISR, verification and calculations |
| 54–60 min | Silicon debug, exercises and interview questions |

## 2. Historical backstory: three different memory problems

The terms SRAM, ROM and OTP are often grouped under “on-chip memory,” but they solve different problems.

- **SRAM:** Fast, directly addressable volatile storage. Useful when low and predictable access latency matters and capacity is modest enough for on-die memory.
- **ROM:** Non-volatile contents fixed for a particular silicon image or product revision. Useful for immutable boot code, tables and fixed control data.
- **OTP:** Small amounts of non-volatile information programmed during manufacturing or provisioning and then treated as irreversible or effectively irreversible. Useful for lifecycle state, calibration, identifiers, configuration and root-key material.

### 2.1 SRAM and the semiconductor-memory transition

Early computer memory used technologies such as magnetic core and delay-line storage. Semiconductor memories offered smaller, faster and more readily integrated storage, but different circuit choices traded density, speed, power and retention.

Intel's historical account of the 1101 describes reliability investigations in fall 1969. It calls the 1101 the first mass-produced chip to use MOS technology and explains how Dov Frohman's investigation of charge trapping in oxide led to the EPROM concept. This is a useful documented connection between SRAM process behavior and later non-volatile memory development; it is not a complete history of every SRAM technology.

Sources:
- Intel, [“A Success…Out of Quality Control Issues — Invention of the EPROM”](https://www.intel.com/content/www/us/en/history/virtual-vault/articles/eprom.html), historical account covering the 1969 investigation and 1971 EPROM demonstration.
- Computer History Museum, [“1971: Reusable semiconductor ROM introduced”](https://www.computerhistory.org/storageengine/reusable-semiconductor-rom-introduced/).

### 2.2 ROM, PROM and iterative firmware development

The Computer History Museum documents these milestones:

- Early integrated-circuit mask ROMs were non-volatile but required a custom mask for each design.
- In 1970, Radiation Inc. introduced a 512-bit user-programmable bipolar PROM using fuse links. It could be programmed by the user, but only once.
- In 1971, Intel announced the 1702, a 2,048-bit UV-erasable EPROM associated with Dov Frohman. A quartz package window allowed ultraviolet light to erase stored charge.
- The museum documents thin-oxide EEPROM development in 1976–1978 and Intel's first EEPROM in 1978.
- Fujio Masuoka's Toshiba work produced flash-memory developments in 1980 and a NAND structure in 1987; commercial products followed in the late 1980s.

**Why this mattered:** mask ROM is suitable for a finalized high-volume image but inconvenient while firmware is changing. PROM and EPROM made it practical to iterate firmware without manufacturing a new ROM mask for every revision. EEPROM and flash later offered electrical erase/program behavior suited to different system needs.

Source: Computer History Museum, [Reusable semiconductor ROM milestone](https://www.computerhistory.org/storageengine/reusable-semiconductor-rom-introduced/), including references to the 1967 floating-gate work, 1971 EPROM, 1978 EEPROM and 1980/1987 flash developments.

### 2.3 OTP is a function, not one universal cell

“One-time programmable” describes the programming contract, not a single transistor structure. Embedded OTP may be implemented with eFuse, antifuse, charge-storage or other process-specific technologies. Their physical behavior, programming conditions, retention characteristics, read margins and failure modes differ.

A fuse-style cell changes a conductive link; an antifuse-style cell creates a conductive path where one did not previously exist. eFuse implementations may use electrically induced material changes. These are conceptual categories only: the actual foundry IP specification controls the pulse, sensing, redundancy and reliability requirements.

OpenTitan's public OTP controller specification makes this separation explicit: its open-source controller presents a technology-independent interface, while the underlying OTP macro is proprietary and provides technology-specific storage, redundancy/error correction and physical defenses. The controller adds logical partitioning, access policy, integrity checks and scrambling.

Source: OpenTitan, [OTP Controller Technical Specification](https://opentitan.org/book/hw/ip/otp_ctrl/).

### 2.4 Design rationale — why not use one memory type everywhere?

| Requirement | Typical choice | Reason |
|---|---|---|
| Low-latency working data, cache, scratchpad | SRAM | Fast random access; no refresh while powered |
| Immutable first-stage code | Mask ROM | Fixed image, no runtime write path |
| Small irreversible provisioning data | OTP/eFuse/antifuse | One-time lifecycle and configuration storage |
| Large updateable persistent data | Flash/eFlash or external NVM | Higher density and field update capability |
| Large working set | DRAM | Much higher density than SRAM, with controller/PHY complexity |

**Engineering interpretation:** memory selection is a system-level PPA, reliability, security and manufacturing decision. SRAM is not automatically “better” than DRAM; it is usually faster but much less dense. ROM is not a secure-boot solution by itself; authenticity and access control require a larger trust architecture.

## 3. SRAM architecture: from one bit to a macro

### 3.1 Conventional 6T SRAM bitcell

A conventional 6-transistor SRAM cell contains:
- Two cross-coupled CMOS inverters (four transistors) storing complementary internal nodes Q and QB.
- Two access transistors controlled by the wordline (WL).
- Differential bitlines BL and BLB connected to column circuitry.

The cross-coupled inverters form a bistable latch. As long as the cell remains powered and its static noise margin is not overcome, feedback preserves the stored state. SRAM does not require the periodic refresh operation used by conventional DRAM.

The 6T topology is a standard teaching model, not a guarantee that every SRAM macro uses a 6T cell. Low-voltage, high-density, multi-port or specialized memories may use 8T, 10T, 12T or other bitcells and different sensing schemes.

~~~text
                 VDD                         VDD
                  |                           |
              PMOS pull-up                PMOS pull-up
                  |                           |
      BL -- access NMOS -- Q  <----->  QB -- access NMOS -- BLB
                 gate=WL       cross-coupled      gate=WL
                  |             inverters          |
              NMOS pull-down                NMOS pull-down
                  |                           |
                 GND                         GND

        4 transistors in the latch + 2 access transistors = 6T
~~~

Reference: Alorda, Torrens and Bota, [“6T CMOS SRAM Stability in Nanoelectronic Era: From Metrics to Built-in Monitoring”](https://doi.org/10.5772/intechopen.73539), published 1 August 2018.

### 3.2 Hold operation

When WL is low:
1. The two access transistors isolate storage nodes from BL/BLB.
2. Cross-coupled inverters reinforce Q and QB.
3. Bitlines are typically precharged/equalized in preparation for a later access, depending on array architecture.
4. The cell retains its value while supply and retention conditions remain valid.

A low-power memory may have a retention mode with reduced supply. Minimum retention voltage is not the same as minimum voltage at which read or write is guaranteed.

### 3.3 Read operation — why the sense amplifier matters

A simplified differential read sequence:

1. **Precharge:** BL and BLB are charged to a common level, often near VDD, and equalized.
2. **Precharge release:** precharge devices turn off.
3. **Row selection:** WL is asserted for the selected row.
4. **Small differential develops:** stored state creates a slight discharge on one bitline relative to the other.
5. **Sense:** a sense amplifier detects the small BL/BLB difference and amplifies it to a valid logic level.
6. **Capture:** column selection and output logic deliver the selected data.
7. **Recovery:** WL is deasserted and bitlines are prepared for the next access.

~~~text
time  --->

PRECHARGE   ____/¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯
WL          ____________/¯¯¯¯¯____________
BL          ¯¯¯¯¯¯¯¯¯¯¯¯¯¯________________
BLB         ¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯¯
SA_ENABLE   ________________/¯¯¯¯__________
DOUT        xxxxxxxxxxxxxxxxxxxx==== VALID ===
                         small ΔV  amplified
~~~

The drawing is conceptual. Real timing depends on precharge strategy, bitline capacitance, replica-bitline timing, sense-amplifier offset, column mux placement and macro organization.

**Silicon insight:** read operation disturbs internal storage nodes because the access transistor connects the cell to a precharged bitline. The cell must be sized and biased to preserve read stability while allowing fast sensing. The 2018 open-access SRAM stability chapter explains read static noise margin (RSNM), butterfly curves and cell-ratio trade-offs. [DOI: 10.5772/intechopen.73539](https://doi.org/10.5772/intechopen.73539).

### 3.4 Write operation

1. The write driver forces BL and BLB to complementary values corresponding to new data.
2. Selected WL is asserted.
3. Driven bitlines overpower the cell's previous state through the access path.
4. Internal latch changes state.
5. WL is deasserted after the write requirement is satisfied.
6. Cell retains the new state.

A stronger access path can improve writability but worsen read stability. Pull-down, pull-up and access transistor sizing must balance read margin, write margin, area and leakage. Low-voltage SRAMs may use assist circuits such as wordline boosting, negative bitline, supply collapse or other foundry-qualified techniques. These assists are implementation-specific and require reliability review.

### 3.5 Read disturb, half-select and stability

A selected row can expose unselected cells on the same wordline to bitline conditions. In some array organizations, these half-selected cells experience stress during another cell's write. Multi-port arrays introduce additional interactions between ports.

Common stability metrics:
- Hold static noise margin (HSNM).
- Read static noise margin (RSNM).
- Write margin / write trip point.
- Read and write current metrics derived from an N-curve.
- Minimum operating voltage and retention voltage.

A memory can pass nominal functional simulation yet fail at low voltage, high temperature, process corners or after aging. Transistor-level characterization and foundry signoff are separate from digital functional verification.

### 3.6 SRAM array and peripheral circuits

A macro is much more than a large collection of bitcells.

~~~mermaid
flowchart TB
  A[Address / command / write data] --> B[Input registers and control]
  B --> C[Row predecoder and row decoder]
  B --> D[Column decoder / column mux]
  C --> E[Wordline drivers]
  E --> F[Bitcell array]
  F --> G[Bitline precharge and equalization]
  F --> H[Sense amplifiers]
  D --> H
  I[Write drivers] --> F
  H --> J[Output latch / data mux]
  J --> K[Read data]
  L[Replica bitline / timing control] --> B
  M[MBIST / test interface] --> B
~~~

Typical sub-blocks:
- **Row decoder:** selects a wordline from row address.
- **Column decoder/multiplexer:** selects one or more bit groups from a row.
- **Wordline driver:** drives capacitive WL load.
- **Bitline precharge/equalization:** establishes initial bitline condition.
- **Sense amplifier:** converts small differential signal into digital output.
- **Write driver:** forces desired differential write data.
- **Replica bitline/self-timing:** tracks representative array delay to help time sense enable or internal sequencing.
- **Control and output logic:** sequences accesses and provides the macro interface.
- **DFT/repair logic:** supports manufacturing test and optional redundancy.

OpenRAM architecture documentation lists bitcell arrays, decoders, wordline drivers, column muxes, bitline precharge, sense amplifiers, write drivers and replica-bitline control. It is a useful public example of how a compiler-generated SRAM is assembled. [OpenRAM architecture](https://github.com/VLSIDA/OpenRAM/blob/stable/docs/source/architecture.md).

### 3.7 Banking and ports

**Banking** divides memory into independently selectable regions. It can improve throughput when concurrent accesses map to different banks. It does not guarantee conflict-free operation: two requests to the same single-ported bank still contend.

| Port style | Meaning | Typical use |
|---|---|---|
| 1RW | One port that performs read or write | Compact scratchpad |
| 1R1W | Independent read and write ports | Producer/consumer buffers |
| 2RW | Two read/write-capable ports | Register files and shared local storage |
| ROM/read-only | Read port only | Lookup tables and boot code |

Multi-port SRAM can increase cell area, peripheral complexity, routing and dynamic power. A “dual-port” label must be clarified: true dual-port, simple dual-port, 1R1W and time-multiplexed pseudo-dual-port have different collision semantics.

## 4. SRAM macro interface and timing contract

### 4.1 There is no universal SRAM macro pinout

A synchronous single-port macro may expose signals resembling:

| Signal | Common purpose | Important contract question |
|---|---|---|
| CLK | Clock | Which edge captures the command? |
| CEN / CSN | Chip enable, often active-low | What happens when disabled? |
| WEN / WENB | Write enable, polarity varies | Is write active high or low? |
| ADDR | Word address | Byte-addressed or word-addressed? |
| DIN | Write data | Required setup/hold? |
| DOUT | Read data | Combinational, registered or latency-N? |
| BWE / MASK | Per-byte or per-bit write mask | What does mask polarity mean? |
| SLEEP / RET | Low-power / retention controls | What data is retained and when? |
| TEST / MBIST | Test-mode controls | Which mux owns the macro in test mode? |

The real macro databook and Liberty/LEF/Verilog views are authoritative. Signal names above are illustrative only.

### 4.2 Read and write timing

For a synchronous macro, a typical contract is:
- Address, chip enable and command are sampled at a clock edge.
- Read result appears after a defined number of cycles or after a clock-to-output delay.
- Write occurs on the accepted write edge, subject to setup/hold and mask requirements.
- Back-to-back requests may be accepted every cycle if the macro is specified for that throughput.

For asynchronous SRAM devices, datasheets may instead specify address access time (tAA), chip-enable access (tCE), output-enable access (tOE), write cycle time (tWC), data setup (tDS) and data hold (tDH). Do not mix asynchronous external SRAM timing names with synchronous embedded macro timing without reading the correct specification.

### 4.3 Read-during-write and collision behavior

For same-address accesses, possible behaviors include:
- Read-first: old data is returned.
- Write-first: new data is returned.
- No-change: output retains its previous value.
- Undefined / don't-care: macro does not guarantee a value.
- Port-priority behavior: one port wins a collision.

A wrapper must implement the documented macro contract. Do not assume an inferred RTL array and a compiled SRAM macro have identical collision behavior.

### 4.4 Address mapping example

Assume a 64 KiB SRAM with a 32-bit data word:
- Capacity = 65,536 bytes.
- Bytes per word = 4.
- Number of words = 65,536 / 4 = 16,384.
- Word address bits = log2(16,384) = 14.
- Byte offset within a word = 2 bits.
- If mapped at 0x1000_0000, byte range is 0x1000_0000–0x1000_FFFF.

The interconnect may decode byte address, while the macro sees a word index. The wrapper must handle byte lanes, write masks and unaligned access policy. A bus-to-macro adapter must not silently discard address bits or shift the wrong byte lane.

### 4.5 Macro wrapper: illustrative synchronous contract

The following is a wrapper sketch, not a vendor macro model. It illustrates where a SoC adapter holds a request and associates a returned read with a response-valid pulse. Real implementations must match the macro's actual latency and enable semantics.

~~~systemverilog
module sram_port_adapter #(
  parameter int ADDR_W = 14,
  parameter int DATA_W = 32,
  parameter int LATENCY = 1
) (
  input  logic                 clk_i,
  input  logic                 rst_ni,
  input  logic                 req_i,
  output logic                 gnt_o,
  input  logic                 we_i,
  input  logic [ADDR_W-1:0]    addr_i,
  input  logic [DATA_W-1:0]    wdata_i,
  input  logic [DATA_W/8-1:0]  wmask_i,
  output logic                 rvalid_o,
  output logic [DATA_W-1:0]    rdata_o,
  output logic                 ram_cen_n_o,
  output logic                 ram_wen_n_o,
  output logic [ADDR_W-1:0]    ram_addr_o,
  output logic [DATA_W-1:0]    ram_din_o,
  output logic [DATA_W/8-1:0]  ram_bwe_n_o,
  input  logic [DATA_W-1:0]    ram_dout_i
);

  logic [LATENCY-1:0] read_pipe_q;
  logic               accept;

  assign gnt_o  = 1'b1; // Example only: real grant depends on buffering/port busy.
  assign accept = req_i && gnt_o;

  // Example active-low macro controls.
  assign ram_cen_n_o = !accept;
  assign ram_wen_n_o = !we_i;
  assign ram_addr_o  = addr_i;
  assign ram_din_o   = wdata_i;
  assign ram_bwe_n_o = ~wmask_i;

  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      read_pipe_q <= '0;
      rvalid_o    <= 1'b0;
      rdata_o     <= '0;
    end else begin
      read_pipe_q <= {read_pipe_q[LATENCY-2:0], accept && !we_i};
      rvalid_o    <= read_pipe_q[LATENCY-1];
      if (read_pipe_q[LATENCY-1])
        rdata_o <= ram_dout_i;
    end
  end
endmodule
~~~

**Review before using:** the parameterized slice above needs a generate branch for LATENCY=1 and LATENCY=0; macro data sampling edge and output latency must be modeled precisely; write-mask polarity must match the macro; and grant logic must not accept a request the memory cannot execute. This is a teaching sketch, not drop-in signoff RTL.

## 5. SRAM performance, power and reliability

### 5.1 Raw bandwidth

For a single port transferring one 32-bit word per cycle at 500 MHz:
- Bytes per transfer = 32 / 8 = 4 bytes.
- Peak raw bandwidth = 4 bytes × 500 million transfers/s = 2,000 MB/s = 2 GB/s (decimal units).

For a 256-bit port at 1 GHz:
- 256 / 8 = 32 bytes per cycle.
- Peak raw bandwidth = 32 × 1 billion = 32 GB/s.

These are ideal interface rates. Actual sustained throughput can be lower due to bank conflicts, read latency, write recovery, port sharing, ECC, arbitration, power-state transitions and access patterns.

### 5.2 Banking example

Suppose a four-bank scratchpad has one 32-bit port per bank at 500 MHz. The theoretical aggregate peak is 4 × 2 GB/s = 8 GB/s if all four banks accept one transfer per cycle. If all clients continuously target the same bank, the bank still provides only its own port rate and arbitration adds stalls.

### 5.3 ECC and integrity overhead

A Hamming-style SECDED code for 32 data bits needs six Hamming check bits because parity count r must satisfy:

2^r >= m + r + 1

For m=32, r=6 is sufficient (64 >= 39). An additional overall parity bit yields a 39-bit protected word in a common SECDED construction. Exact code and bit placement are implementation-specific.

ECC can provide:
- Single-bit error correction.
- Double-bit error detection.
- Error status, counters or interrupt/alert reporting.
- Optional scrubbing to rewrite corrected data.

ECC does not automatically protect address decoders, control logic, all multi-bit faults or malicious modification. The fault model and code properties must be understood.

OpenTitan's SRAM controller is a concrete example: its documented design stores 32 data bits plus 7 integrity bits per word and passes both through its scrambling datapath. Its sub-word writes require read-modify-write to preserve integrity information, reducing documented throughput for sub-word writes. This is an implementation example, not a universal ECC format.

Source: [OpenTitan SRAM controller theory of operation](https://opentitan.org/book/hw/ip/sram_ctrl/doc/theory_of_operation.html).

### 5.4 Scrambling is not encryption

Memory scrambling can reduce exploitable regular patterns and protect against some physical data-remanence or probing scenarios. It is not a replacement for cryptographic authentication, access control or secure key management.

An implementation may transform addresses and data using a key/nonce-derived function. The controller must ensure that:
- The same logical address maps consistently under the active key.
- Key renewal cannot make live data inaccessible without an explicit wipe/reinitialize policy.
- Key material crosses clock domains safely.
- Entropy and key derivation dependencies are ready before memory is released.
- Error and escalation paths block access as intended.

OpenTitan documents a concrete SRAM controller with a TileLink adapter, memory tiles, initialization LFSR, key request logic, PRINCE-based scrambling and integrity bits. Its programmer guide says key renewal implicitly wipes existing SRAM contents, so software must ensure data is no longer needed before renewing the key.

Sources: [SRAM controller architecture](https://opentitan.org/book/hw/ip/sram_ctrl/doc/theory_of_operation.html), [hardware interfaces](https://opentitan.org/book/hw/ip/sram_ctrl/doc/interfaces.html), [programmer's guide](https://opentitan.org/book/hw/ip/sram_ctrl/doc/programmers_guide.html).

### 5.5 Power and retention

Memory power modes commonly include active, standby, light sleep, deep sleep and retention, but exact modes vary by macro.

Questions for the macro specification:
- Is the array powered down or only its periphery?
- Is data retained at a reduced supply?
- Is output state retained?
- How many cycles are required to wake?
- Must the controller wait for power-good or ready?
- Are writes accepted during entry/exit?
- What is minimum retention voltage across process and temperature?
- Does MBIST or scan override low-power controls?

Never gate a memory clock or lower its supply without a coordinated handshake. A power-state transition during an outstanding read or write can lose data or create an invalid response.

## 6. ROM architecture and secure boot

### 6.1 Mask ROM implementation

Mask ROM stores fixed information through a manufacturing-defined pattern. A simplified architecture includes:
- Bitcell or transistor-pattern array.
- Row predecoder and row decoder.
- Wordline drivers.
- Column selection/multiplexing.
- Sense/read circuitry.
- Output registers and bus adapter.

OpenRAM's public architecture documentation also describes a ROM organization using a bitcell array, row decoder, wordline driver, column mux, predecoder, bitline precharge and control logic. This is a useful reference architecture, not a universal foundry ROM implementation.

Source: [OpenRAM architecture](https://github.com/VLSIDA/OpenRAM/blob/stable/docs/source/architecture.md).

### 6.2 Why boot ROM is security-critical

Boot ROM is often the first software executed after reset. If it can be modified without detection, the software trust chain can be compromised. A secure boot ROM is therefore usually designed to be small, immutable and limited in responsibility.

A conceptual boot sequence:

~~~mermaid
flowchart TD
  A[Power-on reset] --> B[ROM controller / integrity check]
  B --> C{ROM integrity valid?}
  C -- No --> D[Hold CPU / alert / fail-safe]
  C -- Yes --> E[CPU starts at reset vector]
  E --> F[ROM establishes minimal clocks and security state]
  F --> G[Read next-stage image]
  G --> H[Hash image and verify signature]
  H --> I{Authentic and permitted?}
  I -- No --> J[Error / recovery policy]
  I -- Yes --> K[Transfer control to next stage]
~~~

Details differ by architecture. Some systems check ROM integrity before releasing the CPU; others have a controller perform background or boot-time verification. ROM may authenticate a next-stage image stored in flash, eMMC, UFS or another non-volatile medium.

### 6.3 Integrity versus authenticity

- **Parity/ECC:** Detects or corrects a defined class of bit errors.
- **CRC/checksum/hash:** Detects changes relative to a known expected value, subject to construction and trust in the expected value.
- **Digital signature verification:** Establishes that an image was signed by a key accepted by the boot policy and that signed content matches.
- **Access control:** Restricts who can read, write, execute or configure a memory region.

A checksum stored beside an image is not automatically an authenticity guarantee. An attacker able to replace both data and checksum can defeat a non-keyed check.

### 6.4 OpenTitan ROM controller example

OpenTitan documents a ROM controller that:
- Serves ROM reads over the system bus.
- Computes a ROM digest and provides it to the key manager.
- Signals the power manager when the ROM check has completed and whether the result is acceptable.
- Uses address/data scrambling in its documented implementation.
- Runs a ROM checker early in startup.

Its documentation says the power manager waits for checker completion before allowing the host processor to start in the documented Earl Grey flow. The ROM checker hashes ROM contents using KMAC/SHA3 and compares the computed result with an expected value stored in the ROM image.

This is a public implementation example. Do not generalize its exact bus, cipher, digest placement or startup handshake to every secure SoC.

Sources: [OpenTitan ROM controller specification](https://opentitan.org/book/hw/ip/rom_ctrl/), [ROM controller theory of operation](https://opentitan.org/book/hw/ip/rom_ctrl/doc/theory_of_operation.html), [secure boot overview](https://opentitan.org/book/doc/security/specs/secure_boot/index.html).

### 6.5 ROM integration checklist

- Reset vector maps to intended ROM address.
- ROM bus access is available before normal DRAM initialization.
- ROM controller and CPU reset ordering is correct.
- ECC/integrity errors block execution or trigger documented recovery.
- Digest/checksum uses correct byte order and address range.
- Expected digest is included in the correct image-generation step.
- Signature verification uses intended key hierarchy and lifecycle policy.
- Debug/test modes cannot bypass production boot policy unintentionally.
- ROM patch/overlay mechanisms, if any, are lifecycle-gated and auditable.

## 7. OTP, eFuse and provisioning architecture

### 7.1 What belongs in OTP?

Typical uses include:
- Silicon revision and manufacturing configuration.
- Analog trims and calibration.
- Unique device identifiers.
- Root public-key hashes or key-encryption material.
- Lifecycle state and debug authorization policy.
- Feature-disable and security configuration fuses.
- Repair information or memory redundancy maps.

Secret material should not be exposed through a generic software-readable register path unless security architecture explicitly permits it. Many designs deliver secrets over dedicated hardware interfaces to key managers or cryptographic engines.

### 7.2 Conceptual OTP block diagram

~~~mermaid
flowchart TB
  SW[CPU / firmware] --> BUS[System bus adapter]
  LC[Lifecycle controller] --> POLICY[Partition and access policy]
  KM[Key manager] --> POLICY
  BUS --> POLICY
  POLICY --> SHADOW[Shadow registers / logical partitions]
  SHADOW --> OTPCTRL[OTP controller FSM]
  OTPCTRL --> ECC[Integrity / ECC / redundancy]
  ECC --> PHY[Technology-specific OTP macro]
  OTPCTRL --> ALERT[Alert and error reporting]
  OTPCTRL --> STATUS[Status / digest / lock CSRs]
~~~

The controller abstracts the physical macro. It may implement:
- Logical partitions and access permissions.
- Read/write sequencing and programming FSMs.
- Redundant encoding and error correction.
- Integrity digests and periodic checks.
- Shadow-register loading.
- Lifecycle gating and irreversible lock operations.
- Scrambling or key derivation for secret partitions.
- Alerts on programming, sensing or integrity failures.

### 7.3 Programming is not an ordinary SRAM write

An OTP programming operation may require a technology-specific sequence involving a programming pulse, sensing, verification and redundant storage. Some implementations use internal high-voltage generation or charge pumps; others use a different physical mechanism. The macro specification determines the exact sequence.

A software-visible command might look like a register write, but the physical operation is not equivalent to a normal bus write. It can be slow, irreversible and sensitive to power integrity and sequencing.

**Safe conceptual provisioning flow:**

1. Enter the authorized manufacturing/provisioning lifecycle state.
2. Verify device identity, partition target and programming data.
3. Check that target is blank or in the expected pre-program state.
4. Issue the technology-qualified programming command through the controller.
5. Wait for completion; do not assume a fixed short latency.
6. Read back through the documented verification path.
7. Check ECC/redundancy and integrity digest status.
8. Lock or digest-lock the partition if lifecycle policy requires it.
9. Record provisioning result in the manufacturing system.
10. Verify that production lifecycle transition removes unauthorized access.

Never invent programming pulse widths, voltages or retry limits. Those values belong to foundry/macro qualification documents and must not be guessed from a generic tutorial.

### 7.4 Shadow registers and lifecycle

Some OTP controllers load selected fuse contents into volatile shadow registers at reset. This gives fast access to configuration while preserving a non-volatile source of truth.

Potential hazards:
- Shadow values not loaded before dependent IP starts.
- Reset during shadow reload.
- Digest/lock bit interpreted with wrong polarity.
- Lifecycle state sampled before OTP initialization completes.
- Secret partition exposed through debug or an unintended bus path.
- Redundancy/ECC failure not propagated to a system alert.
- Firmware retries an irreversible programming operation after an ambiguous timeout.

OpenTitan's OTP controller documents partition locking, integrity checks, periodic comparisons of shadowed and stored content, a dedicated lifecycle partition and technology-specific physical defenses.

Source: [OpenTitan OTP Controller Technical Specification](https://opentitan.org/book/hw/ip/otp_ctrl/).

## 8. MBIST, BISR and manufacturing test

### 8.1 Why memory test is different from logic test

Embedded SRAM macros contain a very large number of repeated cells and dense analog-sensitive structures. Scan testing surrounding logic does not necessarily exercise every memory cell, decoder path, bitline, sense amplifier and coupling interaction. Memory BIST provides a dedicated test engine that applies patterns and compares returned data.

A typical MBIST architecture contains:
- Test-mode mux between functional port and BIST controller.
- Address counter (up/down).
- Pattern generator.
- Read/write sequencer.
- Expected-data generator and comparator.
- Fail address/data capture.
- Status, pass/fail and interrupt interface.
- Optional repair-analysis and redundancy programming interface.

### 8.2 March C- example

March C- is a commonly used March test sequence. In standard notation:

{ ↑(w0); ↑(r0,w1); ↑(r1,w0); ↓(r0,w1); ↓(r1,w0); ↑(r0) }

Where:
- ↑ means increasing address order.
- ↓ means decreasing address order.
- w0 means write zero.
- r0 means read and expect zero.
- w1 means write one.
- r1 means read and expect one.

The sequence has 10 memory operations per location in common accounting. It is often used to detect a useful set of stuck-at, transition, address-decoder and coupling faults under the algorithm's fault model. It does not detect every possible analog, retention, dynamic or multi-fault condition.

References: Arm, [PMC-100 MBIST March C- algorithm](https://support.arm.com/documentation/101528/0001/Production-test-March-Algorithm/March-C--algorithm?lang=en); van de Goor, “Using March Tests to Test SRAMs,” IEEE Design & Test of Computers, 1993, [DOI: 10.1109/54.199799](https://doi.org/10.1109/54.199799).

### 8.3 BISR and repair

Built-in self-repair can replace a failing row or column with a spare, if the macro has redundancy. A repair flow may:
1. Run MBIST and collect failing addresses.
2. Analyze whether failures can be covered by available spare rows/columns.
3. Allocate spare resources.
4. Store repair mapping in fuses or another persistent repair store.
5. Reload repair mapping at power-on.
6. Re-run test to verify repaired operation.

Repair capability, fuse map and algorithm are macro-specific. A macro without redundancy may only report failure for rejection or binning.

### 8.4 Functional test versus MBIST

| Test | Main purpose |
|---|---|
| RTL memory model tests | Functional protocol and wrapper correctness |
| Gate-level/DFT simulation | Test muxes, clocks, reset and test-mode connectivity |
| MBIST | At-speed or macro-specific internal memory fault testing |
| ATE memory test | Manufacturing screening and characterization |
| Retention test | Verify stored state after specified power/voltage/time conditions |
| Shmoo test | Find operating margin across voltage, frequency and temperature |

Do not assume MBIST proves the complete CPU-to-memory integration path. It may bypass the normal bus adapter, ECC path or security logic.

## 9. Verification plan: from macro model to SoC

### 9.1 Functional tests

- Read/write every address.
- Walking-one and walking-zero data patterns.
- Checkerboard and inverse checkerboard.
- Address-as-data and inverse address-as-data.
- Random address/data sequences with reproducible seeds.
- Byte/bit write-mask tests.
- Back-to-back reads and writes.
- Same-address read/write collision cases.
- Bank conflict and simultaneous-port tests.
- Reset and sleep/wakeup transitions.
- Out-of-range and protection-fault accesses.
- ECC single-bit injection and double-bit detection.
- Scrambling key initialization and renewal behavior.
- ROM digest mismatch and OTP integrity error handling.

### 9.2 Assertions and scoreboard ideas

Useful properties:
- A request accepted by the wrapper must produce exactly one response according to configured latency.
- Read data must correspond to the accepted address and must not be shifted by one transaction.
- A disabled macro must not capture an unintended write.
- Write masks must map to documented byte lanes.
- MBIST and functional access must never drive the macro simultaneously.
- Reset must clear control state without corrupting retained memory contents unless the design explicitly wipes them.
- Sleep entry must wait for outstanding transactions to drain.
- ECC error status must be associated with the correct address/transaction.
- OTP lock state must prevent unauthorized future programming.

These are verification intentions. Actual SVA must be adapted to interface handshake, latency and reset semantics.

### 9.3 Macro model fidelity

A behavioral array model is useful for functional tests, but it may not model:
- Read latency.
- Write setup/hold.
- Port collision behavior.
- X propagation and unknown initialization.
- ECC and repair behavior.
- Retention and power-down timing.
- MBIST-only access.
- Physical timing and read/write margin.

Use the foundry/vendor functional model for integration simulation and correct timing/power views for implementation signoff. The synthesis flow must infer or instantiate the intended macro, not accidentally synthesize a huge standard-cell register array.

## 10. Common RTL and integration bugs

| Bug | Observable symptom | Likely root cause |
|---|---|---|
| Address truncated | Aliased locations | Wrong depth/address-width calculation |
| Byte lane reversed | Corruption in selected bytes | Endianness or mask mapping error |
| Read data one cycle late/early | Every second transaction wrong | Macro latency mismatch |
| Wrong read-during-write assumption | Simulation/silicon mismatch | Model collision semantics differ |
| MBIST and CPU both access RAM | Nondeterministic corruption | Test mux or ownership bug |
| ECC not updated on partial write | False ECC errors | Missing read-modify-write |
| Key renewed while data live | SRAM appears corrupted | Scrambling key changed without wipe/reinit |
| Reset deasserted too early | First accesses fail | Macro/power/clock not ready |
| Sleep entered with request pending | Lost response or data | Missing quiesce handshake |
| ROM digest computed over wrong range | Boot halted | Image tool/controller range mismatch |
| OTP lock applied prematurely | Provisioning impossible | Lifecycle sequencing error |
| OTP programming retry after timeout | Irreversible inconsistent state | Completion/readback ambiguity not handled |

## 11. Post-silicon bring-up and characterization

### 11.1 First-silicon checklist

1. Confirm memory supply rails and retention rails.
2. Confirm clocks, reset polarity and reset release order.
3. Confirm macro power-up/initialization completion.
4. Run MBIST and capture failing address/data.
5. Verify CPU bus reads/writes to a reserved SRAM region.
6. Exercise each bank and port independently.
7. Validate ECC and integrity reporting with controlled fault injection where supported.
8. Validate sleep, retention and wake-up sequencing.
9. Validate ROM read path and boot integrity status.
10. Validate OTP shadow loading, partition status and lifecycle policy using approved test modes.

### 11.2 Characterization matrix

Sweep:
- Process corner or silicon speed bin.
- Supply voltage.
- Temperature.
- Frequency.
- Data background (all zero, all one, checkerboard, inverse).
- Address pattern (same row, same bank, alternating bank, boundary).
- Read/write mix.
- Retention duration.
- Low-power entry/exit cycles.

A memory shmoo can reveal the boundary where read margin, write margin, sense timing or retention fails. The failing address distribution matters: a repeated row/column pattern may point to decoder or physical-array locality rather than a random logic bug.

### 11.3 Failure signatures

| Failure pattern | Investigation direction |
|---|---|
| Same bit position across many addresses | Data path, lane, sense amplifier or I/O issue |
| Same row across many columns | Wordline/row decoder/row repair |
| Same column across many rows | Bitline/column mux/sense path |
| Only at low voltage | Read/write margin, assist timing, retention |
| Only at high temperature | Leakage, retention, timing margin |
| Only during simultaneous port activity | Port collision, half-select or arbitration |
| Only after sleep/wakeup | Retention rail, isolation or wake sequencing |
| Only after ECC enabled | ECC encode/decode, byte write or syndrome mapping |
| Only after key renewal | Scrambling key/nonce sequencing or unintended wipe |
| ROM hash mismatch on some lots | ROM image generation, macro data integrity or physical defect |

### 11.4 Debug data to capture

- Physical and logical failing address.
- Expected and actual data.
- ECC syndrome and corrected/uncorrectable status.
- Bank, row, column and macro instance.
- Clock frequency and supply voltage.
- Temperature and silicon lot/stepping.
- Test pattern and operation history.
- MBIST algorithm and repair map.
- Reset/power-state history.
- Whether the failure is repeatable across power cycles.

Do not expose secret OTP contents in logs. Use approved redacted status, digest or manufacturing diagnostics.

## 12. Worked engineering calculations

### Example A — Capacity and address width

A scratchpad is 128 KiB with 64-bit words.

- Capacity = 128 × 1024 = 131,072 bytes.
- Bytes per word = 8.
- Word count = 131,072 / 8 = 16,384.
- Word address width = 14 bits.
- Byte offset = 3 bits.

A byte-addressed interface therefore uses 17 address bits to cover the 128 KiB region (14 word index bits + 3 byte offset bits).

### Example B — Peak bandwidth

A 64-bit SRAM port accepts one transfer each cycle at 800 MHz.

- 64 / 8 = 8 bytes per transfer.
- 8 × 800 million = 6.4 GB/s peak raw bandwidth.

If the controller sustains only 75% utilization due to arbitration and bubbles, a simple estimate is 4.8 GB/s. This utilization factor is an assumption, not a memory specification value.

### Example C — ECC storage overhead

For a 32-bit data word protected by 7 check bits:
- Physical stored width = 39 bits.
- Overhead relative to data = 7 / 32 = 21.875%.
- Data efficiency = 32 / 39 ≈ 82.05%.

This excludes macro rounding, spare rows/columns, parity banking and controller overhead.

### Example D — MBIST execution time

Assume a 64 KiB SRAM with 32-bit words:
- Entries = 65,536 / 4 = 16,384.
- March C- uses 10 operations per entry in common accounting.
- Total memory operations = 163,840.
- If MBIST performs one memory operation per 100 MHz test clock, ideal sequence time = 1.6384 ms.

This is an idealized estimate. Real time includes macro latency, turnaround, compare cycles, algorithm implementation overhead, repair analysis and test-mode transitions.

## 13. Hands-on exercises

### Exercise 1 — Draw the macro

Draw a 6T bitcell and a 16-row × 8-column array. Add row decoder, wordline drivers, bitline precharge, write drivers, sense amplifiers and column mux. Trace one read and one write.

### Exercise 2 — Build an SRAM wrapper

Create a parameterized synchronous single-port wrapper with:
- 32-bit data.
- 12-bit word address.
- Four byte write enables.
- One-cycle read latency.
- Request/grant and response-valid.
- A defined reset and read-during-write contract.

Write assertions for accepted requests and response latency.

### Exercise 3 — Implement March C-

Write a small testbench or BIST FSM that performs:

{ ↑(w0); ↑(r0,w1); ↑(r1,w0); ↓(r0,w1); ↓(r1,w0); ↑(r0) }

Inject a stuck-at fault and an address alias fault in a behavioral memory model. Observe which failures are detected. Do not claim complete physical fault coverage from this simple model.

### Exercise 4 — ROM secure boot

Create a flow diagram that distinguishes:
- ROM data integrity.
- Next-stage image hash.
- Signature verification.
- Lifecycle/debug policy.
- CPU release.

Explain why a checksum alone does not authenticate firmware.

### Exercise 5 — OTP provisioning

Design a state machine with states IDLE, CHECK_POLICY, PROGRAM, WAIT_DONE, READBACK, VERIFY, LOCK, DONE and ERROR. Add timeout and irreversible-operation handling. Use only abstract commands; do not invent physical fuse pulse values.

### Exercise 6 — Silicon failure triage

You observe a 1-bit error at the same bit lane across many SRAM addresses, only at low voltage and high frequency. List at least five hypotheses and the measurements needed to distinguish them.

## 14. Interview questions

1. Why does SRAM retain data without refresh while DRAM needs refresh?
2. What are the six transistors in a conventional 6T SRAM bitcell?
3. Why are SRAM bitlines differential?
4. Why are bitlines precharged before a read?
5. What does a sense amplifier do?
6. Why can a read disturb an SRAM cell?
7. Explain cell ratio and pull-up ratio trade-offs.
8. What is the difference between 1RW and 1R1W SRAM?
9. What is a same-address read/write collision and why is its behavior macro-specific?
10. How do banking and port count affect throughput and area?
11. What is the difference between mask ROM, PROM, EPROM, EEPROM and flash?
12. Why is OTP not a substitute for field-updateable flash?
13. What is the difference between ECC and scrambling?
14. Why does a partial write sometimes require read-modify-write when ECC is present?
15. What is the purpose of MBIST and how does March C- work?
16. What is BISR and how can spare rows/columns be mapped?
17. Why can an SRAM key renewal require the memory contents to be wiped?
18. Why does ROM integrity checking not replace signature verification?
19. What can a row- or column-correlated MBIST failure indicate?
20. Which memory properties must be re-characterized across voltage and temperature?

## 15. Quick revision

- SRAM is volatile, fast and does not need DRAM-style refresh while powered.
- A conventional 6T cell stores a bit using two cross-coupled inverters and two access transistors.
- Precharge, WL timing and sense-amplifier timing determine a correct read.
- Read stability and write ability are competing bitcell design objectives.
- A compiled SRAM macro has a strict interface, timing and collision contract.
- ROM stores fixed code/data; boot ROM is part of the root-of-trust architecture.
- OTP is a one-time provisioning function implemented using process-specific non-volatile technology.
- ECC detects/corrects a defined fault class; scrambling transforms stored representation; neither replaces access control or secure boot.
- MBIST tests memory arrays; BISR can repair some failures if redundancy exists.
- Post-silicon memory debug requires address, data, syndrome, bank/row/column, voltage, frequency, temperature and operation history.

## 16. Authoritative references and public implementation examples

### Historical sources
1. Intel, [EPROM invention and the 1101 investigation](https://www.intel.com/content/www/us/en/history/virtual-vault/articles/eprom.html) — historical account, 1969–1971.
2. Computer History Museum, [Reusable semiconductor ROM introduced](https://www.computerhistory.org/storageengine/reusable-semiconductor-rom-introduced/) — mask ROM, fuse PROM, EPROM, EEPROM and flash milestones; includes cited primary literature.

### SRAM architecture and stability
3. Alorda, Torrens and Bota, [6T CMOS SRAM Stability in Nanoelectronic Era](https://doi.org/10.5772/intechopen.73539), 1 August 2018 — read/write stability, noise margins and cell sizing.
4. VLSIDA, [OpenRAM architecture](https://github.com/VLSIDA/OpenRAM/blob/stable/docs/source/architecture.md) — public memory-compiler architecture: bitcells, decoders, muxes, precharge, sense amps, write drivers and replica timing.
5. VLSIDA, [OpenRAM project](https://openram.org/) — open-source compiler that generates layout, netlists, timing/power models and integration views. The project cites its ICCAD 2016 paper, “OpenRAM: An Open-Source Memory Compiler.”

### SoC memory controllers, ROM and OTP
6. OpenTitan, [SRAM controller theory of operation](https://opentitan.org/book/hw/ip/sram_ctrl/doc/theory_of_operation.html).
7. OpenTitan, [SRAM controller hardware interfaces](https://opentitan.org/book/hw/ip/sram_ctrl/doc/interfaces.html).
8. OpenTitan, [SRAM controller programmer's guide](https://opentitan.org/book/hw/ip/sram_ctrl/doc/programmers_guide.html).
9. OpenTitan, [ROM controller specification](https://opentitan.org/book/hw/ip/rom_ctrl/).
10. OpenTitan, [ROM controller theory of operation](https://opentitan.org/book/hw/ip/rom_ctrl/doc/theory_of_operation.html).
11. OpenTitan, [OTP controller specification](https://opentitan.org/book/hw/ip/otp_ctrl/).
12. OpenTitan, [Secure boot overview](https://opentitan.org/book/doc/security/specs/secure_boot/index.html).

### Memory test
13. Arm, [PMC-100 MBIST March C- algorithm](https://support.arm.com/documentation/101528/0001/Production-test-March-Algorithm/March-C--algorithm?lang=en).
14. van de Goor, [“Using March Tests to Test SRAMs,” IEEE Design & Test of Computers, 1993](https://doi.org/10.1109/54.199799).

**Reading discipline:** use the macro databook, Liberty timing model, LEF/abstract, functional Verilog model, DFT/MBIST specification, foundry reliability guidance and security architecture for the actual target design. Public examples teach the architecture; they do not replace licensed implementation documents.

## 17. Next session — Lesson 12

**DDR / LPDDR — DRAM Architecture, Controller Scheduling, Training and PHY Bring-up**

The next lesson moves from on-die SRAM/ROM/OTP into external high-density memory:

- DRAM cell and bank/row/column organization.
- ACT, PRE, READ, WRITE and REFRESH commands.
- tRCD, tRP, tRAS, tRFC and timing constraints.
- DDR/LPDDR generations and channel/rank topology.
- Controller queues, bank scheduling and reordering.
- DQ/DQS, CA, CK and training concepts.
- PHY initialization, leveling, calibration and training.
- Refresh, self-refresh, power-down and retention.
- ECC, address mapping, interleaving and bandwidth calculations.
- SI/PI, eye diagrams, margining and post-silicon memory shmoo.

**Session 12 goal:** understand the complete path from a CPU/AXI memory request through the DRAM controller and PHY to a real DDR/LPDDR device, then build a practical bring-up and debug checklist.
