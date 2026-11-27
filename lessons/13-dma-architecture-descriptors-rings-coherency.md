# Lesson 13 — DMA Architecture: From Descriptors to Silicon-Level Data Movement

**Curriculum:** SoC IP Masterclass — Basics to Silicon Engineering  
**Session:** 13 of 38  
**Target study time:** 60 minutes, plus optional lab  
**Focus:** DMA engines, descriptor rings, scatter/gather, peripheral handshakes, AXI data movement, IOMMU, cache coherency, interrupts, throughput and post-silicon debug.

> **Scope and source discipline:** DMA is an architectural function, not one universal wire protocol or one standardized register map. This lesson separates (1) generic DMA design concepts, (2) normative behavior inherited from bus/interface specifications such as AMBA AXI, (3) operating-system contracts such as the Linux DMA API, and (4) implementation-specific examples such as AMD AXI DMA PG021 and OpenTitan DMA. The example descriptor and state machines below are educational designs, not claims about a proprietary controller.

---

## 1. Learning objectives and 60-minute plan

By the end, you should be able to:

1. Explain why DMA exists and how it evolved from bus-cycle ownership to bus-mastering, descriptor-driven engines.
2. Draw a DMA block diagram and identify its memory, peripheral, control, descriptor and interrupt paths.
3. Distinguish memory-to-memory, memory-to-peripheral, peripheral-to-memory, cyclic, scatter/gather and 2-D/interleaved transfers.
4. Explain descriptor ownership, producer/consumer rings, completion writeback and doorbell ordering.
5. Follow an AXI memory-mapped read/write and AXI4-Stream transfer, including backpressure and 4-KiB boundary splitting.
6. Distinguish CPU virtual, CPU physical and device DMA/IOVA addresses.
7. Explain cache coherency, cache maintenance, DMA barriers and IOMMU protection.
8. Estimate bandwidth, buffer depth, outstanding transactions and interrupt load.
9. Build a verification and silicon-debug plan for a DMA data-integrity or throughput failure.

Suggested pacing:

| Time | Topic |
|---:|---|
| 0–5 min | Purpose, history and DMA use cases |
| 5–15 min | Architecture, channels, interfaces and transaction flow |
| 15–25 min | Descriptors, rings, scatter/gather and register programming |
| 25–35 min | AXI, buffering, address translation and coherency |
| 35–45 min | Bandwidth calculations, arbitration, power and interrupts |
| 45–55 min | RTL verification, integration bugs and silicon debug |
| 55–60 min | Review questions and lab assignment |

## 2. Why DMA exists

A CPU can move data by executing loads and stores:

1. Load a word from the source.
2. Hold or transform it in a register.
3. Store it to the destination.
4. Update pointers and a count.
5. Repeat until the block is complete.

This consumes instruction issue bandwidth, register resources, cache bandwidth and CPU cycles. It also creates interrupt and scheduling overhead when a peripheral needs service frequently.

A DMA engine moves the data as an independent bus initiator. The CPU configures the operation, establishes buffer ownership, starts the channel and handles completion or error. The DMA engine performs the repeated data movement.

DMA does **not** make memory bandwidth free. It changes who issues the transfers and can overlap data movement with CPU execution. The DMA still competes for NoC, cache, memory-controller and DRAM resources.

### Typical SoC use cases

- Audio playback: DRAM → DMA → I2S/TDM/SoundWire-facing FIFO.
- Audio capture: codec/I2S FIFO → DMA → DRAM.
- Camera: CSI/ISP output → DMA → frame buffers.
- Display: frame buffer → display fetch engine; some display blocks contain their own DMA-like fetch clients.
- Ethernet: packet buffers ↔ MAC FIFO.
- USB: system memory ↔ endpoint FIFOs.
- PCIe: host memory ↔ endpoint memory, subject to PCIe requester/completer rules.
- Storage: eMMC/UFS controller ↔ DRAM.
- DSP: DRAM ↔ local SRAM, TCM or DSP stream port.
- Crypto: memory ↔ AES/SHA accelerator input/output.
- Memory-to-memory copy, fill, checksum or format conversion.

A device with an integrated bus-master interface may contain its own DMA engine. A separate central DMA controller may instead service multiple peripherals. These are architectural alternatives, not mutually exclusive product categories.

## 3. Historical backstory: from channel I/O to descriptor-driven DMA

The original engineering problem was moving records or blocks while allowing computation to continue. The solution evolved through several architectural forms.

### 3.1 Early channel and controller approaches

- **1954 — NBS DYSEAC:** historical I/O research records describe early use of direct memory transfer and I/O synchronization outside the normal instruction-by-instruction CPU copy loop. Mark Smotherman's survey traces early DMA and channel-I/O designs and explicitly discusses uncertainty in assigning historical “firsts.” Treat this as a documented historical survey, not a claim that every later DMA design descends from one machine.
- **1957 onward — IBM 709 channel I/O:** channel controllers accepted I/O programs and moved records while the CPU performed other work. The channel was a more capable I/O processor than a simple address/count counter.
- **Intel 8237A — programmable multi-channel DMA:** the published 8237A data sheet documents four independent request channels, address and word-count registers, fixed/rotating priority, autoinitialization, demand/single/block modes, DREQ/DACK, HRQ/HLDA and memory/I/O control signals. Its bus ownership model is characteristic of the shared-bus microcomputer era.
- **1981 IBM PC / 1985 IBM PC AT technical reference:** the PC family used 8237A-compatible DMA controllers for selected devices; the PC AT reference documents two controllers, one cascaded into the other, with seven usable channels. The architecture exposed channel numbers and page registers to software.

### 3.2 Why the architecture changed

A traditional third-party DMA controller requests ownership of a shared bus and temporarily controls it. Modern SoCs usually use multiple independent bus initiators. A peripheral or DMA engine becomes a bus master and sends transactions through an interconnect.

The modern engine often adds:

- Separate read and write paths.
- Multiple outstanding transactions.
- Burst generation and data realignment.
- Descriptor fetch and completion writeback.
- Scatter/gather lists and cyclic rings.
- Per-channel priorities and request handshakes.
- IOMMU translation and access protection.
- Cache-coherent ports or explicit cache-maintenance contracts.
- Interrupt coalescing and error reporting.

### 3.3 Design rationale and trade-offs

| Design choice | Why it exists | Cost or limitation |
|---|---|---|
| Programmed single transfer | Small register/control footprint | CPU must reprogram each operation |
| Descriptor chain | CPU can queue work in memory | Descriptor fetches, ownership protocol and ring management |
| Scatter/gather | One logical packet can span non-contiguous buffers | More descriptors, mapping and boundary handling |
| Cyclic ring | Continuous audio/video streaming | Producer/consumer overrun and underrun must be managed |
| Bus-master DMA | High throughput and CPU overlap | Arbitration, address protection and isolation become critical |
| Central DMA controller | Shared policy and reusable channels | Routing, channel allocation and contention |
| Peripheral-local DMA | Low latency and close FIFO coupling | Duplicated logic and distributed software model |
| Coherent DMA | CPU/device views remain coherent under the platform contract | Coherency traffic, fabric complexity and power |
| Non-coherent DMA | Simpler data path on some systems | Explicit mapping/synchronization and cache maintenance |

**Engineering interpretation:** the evolution is driven by the growth of data rates, the move from shared buses to switched fabrics, and the need to queue many small/large transfers without a CPU interrupt per beat. The public documents support the features listed; they do not reveal undocumented internal company motivations.

## 4. DMA taxonomy: do not confuse direction with topology

### 4.1 Transfer direction

| Mode | Source | Destination | Typical use |
|---|---|---|---|
| Memory-to-memory | DRAM/SRAM | DRAM/SRAM | Copy, move, fill |
| Memory-to-peripheral | Memory | Fixed peripheral FIFO | Playback, transmit |
| Peripheral-to-memory | Fixed peripheral FIFO | Memory | Capture, receive |
| Peripheral-to-peripheral | Peripheral | Peripheral | Special routing, if supported |

For a peripheral FIFO, one side is usually a **fixed address**. The memory side generally increments. If the FIFO register address increments accidentally, the DMA may read unrelated registers or write into the wrong peripheral register.

### 4.2 Descriptor and scheduling modes

- **Direct-register/simple DMA:** software programs source, destination and length registers for each transfer.
- **Scatter/gather (SG):** hardware fetches descriptors from memory and processes a linked list or ring.
- **Cyclic DMA:** hardware repeatedly traverses a circular set of buffers or periods until stopped.
- **Interleaved/2-D DMA:** transfers rows or segments with explicit stride and line/plane geometry.
- **Linked-list DMA:** each descriptor points to a next descriptor; a ring is one special case.
- **Memcpy engine:** software requests a memory-to-memory operation through a DMA framework.
- **Slave DMA:** a peripheral requests or paces transfers through a FIFO-ready/handshake interface.

These modes are implementation capabilities. AXI does not define a universal descriptor layout or require a DMA engine to support every mode.

## 5. Block-level architecture

A representative high-performance DMA contains these logical blocks:

~~~mermaid
flowchart TB
  CPU[CPU / Driver] -->|AXI4-Lite or APB control| REG[Register bank / doorbells]
  REG --> CTRL[Channel control and scheduler]
  PERIPH[Peripheral FIFO / AXI4-Stream] <--> STREAM[Stream adapter and handshake]
  STREAM <--> FIFO[Data FIFO / width and byte realigner]
  FIFO <--> READ[Read engine]
  FIFO <--> WRITE[Write engine]
  DESC[Descriptor fetch / decode / writeback] <--> MEM[System memory through AXI / NoC]
  CTRL --> DESC
  CTRL --> READ
  CTRL --> WRITE
  READ <--> MEM
  WRITE <--> MEM
  CTRL --> IRQ[Completion / error / coalescing]
  IRQ --> CPU
  CTRL --> PROT[IOMMU / firewall / address checks]
~~~

### 5.1 Main sub-blocks

1. **Control/status registers:** channel enable, pause, reset, source/destination, length, interrupt enable/status, current descriptor and tail/doorbell.
2. **Descriptor engine:** fetches descriptors, checks alignment and fields, tracks ownership, advances the ring and writes completion status.
3. **Address generator:** increments, holds or applies a stride/wrap rule; splits transfers at bus and page boundaries.
4. **Read engine:** issues address requests, accepts returned data and tracks IDs/responses.
5. **Write engine:** assembles write data, strobes and last-beat indication; waits for write responses.
6. **FIFO/buffer:** decouples peripheral timing from memory-system timing and absorbs backpressure.
7. **Data realignment/width conversion:** handles byte offsets, bus width differences and partial final beats.
8. **Request/handshake logic:** accepts a peripheral request or AXI-stream valid/ready traffic.
9. **Interrupt/status logic:** reports completion, errors, threshold and timeout events.
10. **Protection logic:** checks address windows, security attributes, privilege and IOMMU translation outcomes.

### 5.2 One engine or several?

A DMA can have a single shared datapath with many channel contexts, or separate engines for read and write, or multiple independent channels. A channel is a software-visible or hardware-visible transfer context; it does not necessarily correspond to a dedicated physical datapath.

## 6. Interfaces and signals

### 6.1 Control interface

Common implementation choices:

- AXI4-Lite: memory-mapped control/status registers.
- APB: low-bandwidth peripheral programming.
- A platform-specific control bus.

The control interface is separate from the high-bandwidth data path.

### 6.2 AXI4 memory-mapped master

A DMA that reads and writes memory typically uses:

**Read address channel**
- ARADDR: byte address.
- ARLEN: beats minus one.
- ARSIZE: bytes per beat encoded as log2(bytes).
- ARBURST: FIXED, INCR or WRAP.
- ARID: transaction identifier, if implemented.
- ARVALID / ARREADY: request handshake.

**Read data channel**
- RDATA, RID, RRESP, RLAST.
- RVALID / RREADY.

**Write address channel**
- AWADDR, AWLEN, AWSIZE, AWBURST, AWID.
- AWVALID / AWREADY.

**Write data channel**
- WDATA, WSTRB, WLAST.
- WVALID / WREADY.

**Write response channel**
- BID, BRESP.
- BVALID / BREADY.

A transfer occurs only on a rising clock edge where VALID and READY are both high. A producer must hold VALID and payload stable while VALID=1 and READY=0. The DMA must not drop or duplicate a beat when backpressure occurs.

AXI4 protocol requirements belong to the Arm AMBA AXI specification. Burst policy, queue depth, descriptor layout, data realignment and register programming are DMA implementation choices.

### 6.3 AXI4-Stream or peripheral FIFO interface

Common AXI4-Stream signals:

- TDATA: payload.
- TVALID / TREADY: beat handshake.
- TKEEP: byte-valid mask for the beat.
- TLAST: packet/transfer boundary as defined by the connected IP contract.
- TUSER: sideband metadata.
- TID / TDEST: optional stream identification/routing.

Do not assume TLAST always means “DMA descriptor complete.” It means what the stream endpoint contract defines. A DMA may translate packet boundaries into descriptor status, but the exact mapping is implementation-specific.

A FIFO-based peripheral may instead expose:
- request/ready or watermark,
- data,
- read/write enable,
- almost-full/almost-empty,
- request acknowledge.

### 6.4 Legacy request/acknowledge

The Intel 8237A uses DREQ and DACK per channel, plus HRQ/HLDA to request and receive CPU bus ownership. Modern peripheral DMA handshakes may use different semantics and clocks. Do not connect an asynchronous request directly into a synchronous state machine without a defined CDC/synchronizer strategy.

## 7. Transaction walkthroughs

### 7.1 Memory-to-stream (MM2S / transmit)

1. Software creates or selects a source buffer.
2. Software maps the buffer for the DMA device and obtains a DMA address.
3. Software fills descriptor fields: source address, byte count, control flags and next pointer.
4. Software publishes descriptor ownership and notifies hardware.
5. DMA fetches the descriptor.
6. DMA validates length, alignment, address range and permissions.
7. DMA issues AXI read bursts.
8. Returned beats enter an internal FIFO.
9. Stream adapter presents TDATA/TKEEP/TLAST and holds them stable under backpressure.
10. Completion is recorded after the defined final data/response condition.
11. DMA writes descriptor status and raises an interrupt or updates a completion queue.

### 7.2 Stream-to-memory (S2MM / receive)

1. Software allocates/maps destination buffers and posts receive descriptors.
2. DMA fetches descriptors and makes receive capacity available.
3. Peripheral asserts data valid or writes its FIFO.
4. DMA accepts data only when it has buffer space and a valid descriptor.
5. DMA writes AXI bursts to memory, using WSTRB for partial final beats.
6. The peripheral packet/period boundary is recorded according to the interface contract.
7. DMA writes actual byte count, error status and completion ownership.
8. Software observes completion, synchronizes the buffer for CPU access if required, then consumes the data.

**Critical distinction:** a receive buffer length is capacity, not necessarily the actual received length. The actual byte count must be captured and reported.

### 7.3 Memory-to-memory

The read and write engines can overlap. A FIFO decouples the source read latency from destination write acceptance. If destination backpressure fills the FIFO, the read engine must stop issuing or accepting more data. If the source stalls, the write engine must not fabricate data.

## 8. Descriptor architecture

A descriptor is a memory-resident command record. There is no universal DMA descriptor ABI. A typical conceptual descriptor contains:

| Field | Purpose |
|---|---|
| NEXT | Next descriptor DMA address, if linked |
| BUFFER_ADDR | Payload buffer DMA address |
| LENGTH | Requested byte count or capacity |
| CONTROL | Direction-specific flags, SOF/EOF, interrupt, checksum, stride |
| STATUS | Completion, actual bytes, error code, ownership |
| COOKIE / APP | Software metadata or peripheral-specific context |

### 8.1 Ownership model

A robust ownership lifecycle is:

1. FREE — software may allocate/reuse the slot.
2. PREPARED — software has filled address, length and controls.
3. PUBLISHED — descriptor contents are visible and ownership has been transferred to hardware.
4. IN_FLIGHT — hardware has fetched/accepted it.
5. COMPLETE — hardware has finished and returned ownership/status.
6. RECLAIMED — software has observed completion and may reuse it.

Only one side may modify ownership-controlled fields at a time. The exact bit polarity and phase semantics are IP-specific.

### 8.2 Ring pointers and wrap

A ring with N entries often uses producer and consumer indices. A common implementation adds a phase/generation bit to distinguish full from empty when indices wrap. Other designs reserve one slot or use monotonic counters.

For a power-of-two ring:
- index = counter & (N - 1)
- phase = (counter / N) mod 2

This is an example, not a required DMA architecture.

### 8.3 Descriptor publication ordering

The CPU must make descriptor contents visible before transferring ownership and ringing the doorbell. On Linux, coherent descriptor memory still needs the appropriate DMA memory-ordering barriers for shared ownership protocols. The kernel documentation gives the canonical pattern:

~~~c
/* Fill address, length and control fields first. */
desc->buffer_addr = dma_addr;
desc->length = length;
desc->control = control;

/* Order descriptor payload before ownership publication. */
dma_wmb();
desc->status = DEVICE_OWN;

/* Notify the engine using the platform's MMIO accessor. */
writel(DESC_NOTIFY, regs + DOORBELL);
~~~

On completion, software observes the ownership/status field and uses the required read ordering before consuming other device-written descriptor fields. The exact MMIO accessor and ordering contract must follow the OS and architecture documentation. A DMA barrier does not, by itself, define ordering for every MMIO access.

### 8.4 Scatter/gather

Scatter/gather lets one logical operation span multiple memory regions. A packet may have a header in one buffer and payload in another. A controller can:
- process each descriptor as an independent transfer,
- treat several descriptors as one packet,
- mark first/last fragments with SOF/EOF,
- report per-descriptor actual byte count.

Do not assume that one scatterlist entry maps to one hardware descriptor. The OS DMA mapper may merge adjacent segments; drivers must use the DMA-mapped segment count and DMA addresses/lengths.

### 8.5 Cyclic DMA

Cyclic DMA repeatedly traverses a buffer or descriptor ring. It is common for audio capture/playback and continuous sensor streams.

A period interrupt can report that a region is ready for the consumer/producer. The software must complete its work before hardware wraps and overwrites or rereads that region.

For a ring with B bytes and stream rate R bytes/s, wrap time is B/R. If software needs worst-case scheduling latency L, the ring and period design must leave adequate margin beyond L, plus burst and interrupt jitter.

## 9. Example register model and programming sequence

**Important:** this is an educational register model. It is not a universal DMA standard.

| Register | Example meaning |
|---|---|
| CTRL | Enable, pause, reset, direction, mode |
| STATUS | Idle, busy, halted, error, completion |
| SRC_ADDR_LO/HI | Source DMA address |
| DST_ADDR_LO/HI | Destination DMA address |
| LENGTH | Byte count |
| DESC_BASE_LO/HI | Descriptor ring base |
| RING_SIZE | Number of descriptors |
| HEAD | Hardware consumer pointer |
| TAIL / DOORBELL | Software producer notification |
| IRQ_ENABLE | Completion/error/threshold masks |
| IRQ_STATUS | Sticky completion/error status |
| ERROR_ADDR | Faulting DMA address, if implemented |

### 9.1 Safe simple-mode sequence

1. Reset/disable channel and wait for reset completion.
2. Clear stale status using the documented write-one-to-clear or other register semantics.
3. Program source and destination DMA addresses, including high bits.
4. Program direction, transfer attributes and byte length.
5. Ensure the data buffer is mapped and synchronized for the correct direction.
6. Enable the channel or write the length/doorbell last, according to the IP manual.
7. Observe active/busy status.
8. Wait for completion or error interrupt.
9. Confirm status and actual byte count.
10. Synchronize/unmap the buffer before CPU access or reuse.
11. On error, stop/reset using the documented recovery sequence and reclaim ownership safely.

### 9.2 Descriptor-mode sequence

1. Allocate a descriptor ring with the required alignment and DMA reachability.
2. Allocate/map each payload buffer.
3. Initialize descriptor fields and next pointers.
4. Initialize head/tail or producer/consumer state.
5. Publish descriptors with correct cache synchronization and memory ordering.
6. Program ring base and ring size.
7. Enable completion/error interrupts and coalescing policy.
8. Enable channel.
9. Ring the doorbell or update the tail pointer.
10. Reclaim only descriptors whose completion/ownership status has been observed.
11. Refill the ring and notify hardware again.

### 9.3 Vendor example: AMD AXI DMA PG021

AMD AXI DMA is a concrete implementation, not a normative AMBA DMA specification. PG021 v7.1 (release date shown as 24 June 2025) documents:
- MM2S (memory-mapped to stream) and S2MM (stream to memory-mapped) paths.
- AXI4-Lite control/status registers.
- Optional scatter/gather descriptor engine.
- Direct-register mode when SG is disabled.
- Current-descriptor and tail-descriptor pointers.
- Optional cyclic operation and byte realignment.
- Separate read/write datapaths and configurable widths.

In that implementation, example offsets include MM2S_DMACR 0x00, MM2S_DMASR 0x04, MM2S_CURDESC 0x08, MM2S_TAILDESC 0x10, MM2S_LENGTH 0x28; S2MM_DMACR 0x30, S2MM_DMASR 0x34, S2MM_CURDESC 0x38, S2MM_TAILDESC 0x40 and S2MM_LENGTH 0x58. These offsets apply to that IP configuration/documentation only.

AMD's guide states that writing the length register starts a simple transfer, and that the tail descriptor update is used to resume/extend descriptor processing. Follow the exact version and configuration-specific sequence rather than copying offsets into another controller.

## 10. Addressing, IOMMU and security

There are three address concepts:

- CPU virtual address (VA): what software dereferences.
- CPU physical address (PA): system physical address after CPU translation.
- DMA/bus address or IOVA: address presented by the device.

They may be equal on a simple system, but software must not assume that.

~~~mermaid
flowchart LR
  VA[CPU virtual address] -->|CPU MMU| PA[Physical address]
  PA -->|DMA mapping / IOMMU setup| IOVA[DMA address / IOVA]
  IOVA -->|IOMMU or SMMU| PA
  IOVA -->|Device issues bus transaction| FABRIC[AXI / NoC]
  FABRIC --> MEM[Memory]
~~~

### 10.1 IOMMU / SMMU role

An IOMMU translates device-visible IOVAs to physical pages and can enforce access permissions. It can:
- isolate devices or virtual machines,
- prevent a buggy/malicious DMA engine from accessing arbitrary memory,
- map fragmented physical pages into a contiguous IOVA window,
- report translation/protection faults,
- constrain address width and reachable memory.

The DMA engine still needs correct device identity/stream ID, translation context, permissions and invalidation ordering. An IOMMU is not a substitute for descriptor bounds checking or secure channel configuration.

### 10.2 Address width and boundary rules

Check all of these:
- hardware address width and software DMA mask,
- descriptor pointer width,
- payload address width,
- alignment and transfer length,
- memory region accessibility,
- AXI 4-KiB burst boundary,
- cache-line boundary and IOMMU page boundary,
- peripheral FIFO address fixed/incrementing mode.

For AXI4 INCR bursts, a burst must not cross a 4-KiB address boundary. A DMA should split a longer request into legal bursts. Some small-footprint/micro modes intentionally omit boundary protection; that is an implementation constraint, not a general AXI permission to cross the boundary.

## 11. Cache coherency and software ownership

This is one of the most common sources of silent DMA corruption.

### 11.1 CPU-to-device (transmit)

CPU writes payload into cache. If the device reads DRAM through a non-coherent path, DRAM may still contain old data. The mapping/synchronization contract must make CPU writes visible before the device reads.

### 11.2 Device-to-CPU (receive)

Device writes new data to DRAM. CPU may still hold an old cache line. Before CPU consumes the buffer, the DMA API or platform coherency contract must make the device-written data visible.

### 11.3 Coherent vs streaming mappings

- **Coherent allocation:** CPU and device are given a consistent shared allocation under the platform's DMA-coherency contract. Coherent does not mean all accesses are automatically ordered or race-free.
- **Streaming mapping:** a buffer is mapped for a transfer direction and ownership is transferred between CPU and device. Reused mappings may require explicit sync-for-CPU and sync-for-device operations.
- **Descriptor rings:** often use coherent DMA allocation, but still require ordering between descriptor fields, ownership bits and doorbells.
- **Payload buffers:** often use streaming mappings or a subsystem-specific shared-buffer framework.

Linux DMA API terminology:
- DMA_TO_DEVICE: main memory → device.
- DMA_FROM_DEVICE: device → main memory.
- DMA_BIDIRECTIONAL: both directions.
- dma_map_single()/dma_map_sg() create streaming mappings.
- dma_unmap_*() ends a mapping after DMA activity is complete.
- dma_sync_*_for_cpu()/for_device() transitions a reused streaming mapping between ownership phases.
- dma_alloc_coherent() returns both CPU virtual address and device DMA handle.

Do not replace the DMA API with raw virt_to_phys() in a portable Linux driver. IOMMU translation, bounce buffers, bus offsets and device masks can make that wrong.

### 11.4 Descriptor barriers are not cache maintenance

A barrier orders accesses. It does not automatically clean dirty cache lines on a non-coherent platform. Cache maintenance makes data visible; barriers order publication/observation. The platform DMA API combines the required mechanisms for the driver.

A common ownership pattern is:
- CPU fills descriptor/payload.
- map or sync payload for device.
- write descriptor fields.
- DMA write barrier.
- publish device ownership.
- ring doorbell.
- device completes and returns ownership.
- CPU observes completion.
- DMA read barrier.
- sync receive payload for CPU if needed.
- CPU consumes the data.

## 12. Timing, bandwidth and throughput calculations

### 12.1 Peak bandwidth

For a memory-mapped bus:

B_peak = data_width_bits / 8 × clock_hz

Example: 128-bit data path at 500 MHz:

- 128/8 = 16 bytes per beat.
- 16 × 500,000,000 = 8,000,000,000 bytes/s.
- Peak = 8 GB/s (decimal).

At 70% sustained utilization, estimate 5.6 GB/s. The 70% is an assumption, not a protocol guarantee. Real efficiency depends on arbitration, DRAM row hits, refresh, burst length, turnaround, QoS and other masters.

### 12.2 Outstanding transactions and bandwidth-delay product

If a path delivers 8 GB/s and round-trip memory service latency is 200 ns:

- bandwidth-delay product = 8 × 10^9 × 200 × 10^-9 = 1,600 bytes.
- With 256-byte useful bursts, at least ceil(1600/256) = 7 bursts must be in flight to cover that latency in this simplified model.

Real controllers may need more or fewer due to bank parallelism, queueing, burst efficiency and read/write turnaround. This calculation is a starting point for sizing outstanding requests, not a final microarchitecture answer.

### 12.3 4-KiB boundary example

Assume 16-byte beats and a transfer beginning at address 0x1FF0. The current 4-KiB page ends at 0x1FFF, so only one 16-byte beat fits before address 0x2000. A larger transfer must be split:
- Burst 1: 0x1FF0, one beat.
- Burst 2: 0x2000, remaining beats.

### 12.4 Audio DMA sizing example

Assume:
- 32 channels,
- 48,000 samples/s,
- 32 bits/sample.

One direction:

32 × 48,000 × 4 = 6,144,000 bytes/s = 6.144 MB/s.

Full-duplex capture plus playback is approximately 12.288 MB/s before descriptor, framing and software overhead.

For a 4 ms period:

6.144 MB/s × 0.004 s = 24,576 bytes per direction.

If a design uses four periods per ring, it needs 98,304 bytes per direction. The period size, number of periods and callback latency must be selected together. A small ring reduces latency but leaves less scheduling margin.

### 12.5 Video example

3840 × 2160 × 60 frames/s × 4 bytes/pixel = 1,990,656,000 bytes/s, approximately 1.99 GB/s for one uncompressed 32-bit pixel stream. Multiple camera/display/ISP passes and frame-buffer reads/writes multiply the traffic. This is why DMA QoS and memory-controller arbitration matter even when one peripheral's own stream rate looks modest.

## 13. Buffering, backpressure and QoS

### 13.1 FIFO sizing

A FIFO must cover the maximum data accumulated during a downstream stall plus implementation latency and burst granularity. A first-order estimate:

FIFO_bytes ≥ input_rate_bytes_per_second × worst_case_stall_seconds + in_flight_bytes + margin

Example: a 400 MB/s stream may be stalled for 2 microseconds:
- stall accumulation = 400 × 10^6 × 2 × 10^-6 = 800 bytes.
- Add in-flight data and margin; a 1 KiB FIFO may be insufficient depending on burst and pipeline depth.

This is an estimate; use a cycle-accurate traffic model for final sizing.

### 13.2 Backpressure

- If stream TREADY deasserts, the DMA must hold TVALID and payload stable.
- If a memory read response stalls, the response FIFO must not overflow.
- If write responses stall, the DMA must limit accepted writes to available tracking capacity.
- A descriptor must not be marked complete before the required data and response conditions have been satisfied.
- A reset/abort must account for accepted but not yet completed transactions.

### 13.3 QoS and arbitration

Multiple DMA clients compete with CPU cache refills, display, camera, GPU and DSP. Fixed priority can starve low-priority channels. Round-robin can give fairness but may violate hard real-time latency. Weighted or deadline-aware scheduling may be appropriate.

Define:
- maximum latency,
- minimum sustained bandwidth,
- burstiness,
- priority,
- starvation bound,
- backpressure behavior,
- interrupt/coalescing policy.

A display underflow or audio FIFO underrun may be a system-level memory arbitration failure, not a local DMA RTL defect.

## 14. Clock, reset, power and CDC

- The control bus, descriptor fetch, data mover and stream endpoint may use different clocks.
- Use asynchronous FIFOs or verified CDC bridges for data/control crossings; do not synchronize a multi-bit payload bus with independent two-flop synchronizers.
- Synchronize request/acknowledge events and preserve event semantics.
- Reset release must be synchronized to each destination clock domain.
- On reset, define whether in-flight descriptors are aborted, replayed or reported as errors.
- Drain or safely discard outstanding AXI transactions according to the bus/interconnect contract before power removal.
- Clock gating must not stop a clock while a handshake is half-complete unless the interface has an explicit quiesce protocol.
- Power-domain isolation must prevent X propagation and accidental transactions while a domain is off.
- Retained descriptor pointers and FIFO state must be consistent with software-visible reset semantics.
- Wakeup logic must remain active if the DMA is expected to respond to a peripheral request in a low-power state.

## 15. Interrupts and completion reporting

Common events:
- transfer complete,
- period complete,
- descriptor complete,
- packet complete,
- threshold reached,
- bus error,
- descriptor fetch error,
- address translation/protection fault,
- FIFO overflow/underflow,
- timeout or watchdog.

Interrupt coalescing can reduce CPU interrupt rate by triggering after N completions or a timer expires. This trades interrupt overhead for completion latency. A cyclic audio stream may need period notifications; an Ethernet receive path may batch multiple packets.

Use sticky status carefully:
- define W1C vs read-clear semantics,
- avoid losing a new event while software clears an old one,
- define interrupt masking and unmasking order,
- ensure status is visible before interrupt assertion,
- distinguish channel-local from global error status.

## 16. Common RTL and integration bugs

| Symptom | Likely cause | Debug approach |
|---|---|---|
| First transfer works, next hangs | Ring head/tail or ownership wrap bug | Trace indices, phase bit and doorbell |
| Correct length, corrupt final bytes | WSTRB/TKEEP/byte count error | Check last beat and partial-byte mask |
| Random stale payload | Cache synchronization or coherency mismatch | Compare DRAM, CPU cache and DMA view |
| Descriptor fields occasionally wrong | Ownership published before fields visible | Check barriers/cache clean and fetch trace |
| DMA accesses wrong memory | CPU PA used instead of DMA/IOVA | Trace address translation and DMA mask |
| Hangs at 4-KiB boundary | Burst not split or length math off by one | Check AR/AW address, LEN, SIZE |
| RX buffer overwritten | Capacity too small or descriptor reused early | Compare actual bytes, ownership and ring depth |
| TX packet truncated | TLAST/SOF/EOF or descriptor length mismatch | Trace stream boundary and descriptor control |
| Throughput low | Small bursts, low outstanding count, arbitration | Measure accepted beats, stalls and memory service |
| CPU sees old data | Missing sync-for-CPU or wrong mapping direction | Inspect DMA API sequence |
| Duplicate completion | Status clear/IRQ race or descriptor replay | Correlate descriptor sequence IDs |
| Error after reset | Outstanding transactions or stale descriptor fetch | Trace reset, AXI responses and ring reinitialization |
| Only high addresses fail | Address high bits truncated | Test above 4 GiB and inspect all address registers |
| Peripheral FIFO overrun | DMA service latency or request handshake lost | Trace request, grant, FIFO watermark and QoS |

## 17. Pre-silicon verification plan

### 17.1 Directed tests

1. Zero, minimum, one-beat, exact-burst and maximum supported lengths.
2. Aligned and every supported unaligned starting offset.
3. Partial final beat and all WSTRB/TKEEP patterns.
4. Transfers ending immediately before, at and after a 4-KiB boundary.
5. Source/destination in each legal memory region.
6. Fixed peripheral address and incrementing memory address.
7. Descriptor chain of one, two, maximum and wraparound entries.
8. Empty ring, full ring, tail update while active and stopped-at-tail behavior.
9. SOF/EOF across multiple descriptors.
10. Cyclic period rollover and late software service.
11. AXI AR/AW/W/R/B backpressure on every channel.
12. Peripheral request pauses and stream TREADY stalls.
13. Read/write error responses, timeout, IOMMU fault and firewall denial.
14. Reset/power interruption in every state.
15. Concurrent channels with priority and starvation checks.

### 17.2 Scoreboard

Maintain a reference byte array and model:
- accepted descriptor sequence,
- source/destination address,
- requested length,
- actual transferred bytes,
- packet/period boundaries,
- byte strobes,
- completion/error status.

Compare every committed destination byte. For packet modes, compare packet order and metadata as well as payload.

### 17.3 Assertions and invariants

The exact signal names depend on the implementation. Useful properties include:

- VALID and payload remain stable while stalled.
- A stream beat is consumed only on TVALID && TREADY.
- A memory write beat is generated only from accepted source data.
- No byte outside the programmed destination range is modified.
- No descriptor is fetched unless it is published/owned by hardware.
- No descriptor is returned to software before required completion conditions.
- Each accepted descriptor completes at most once.
- Completion byte count never exceeds capacity.
- No AXI burst crosses a 4-KiB boundary.
- Outstanding counters never underflow or exceed configured limits.
- Reset/abort eventually reaches a documented idle or error state.
- No request is silently dropped when a FIFO is full.

Example property sketch:

~~~systemverilog
property p_stream_stable_when_stalled;
  @(posedge aclk) disable iff (!aresetn)
    tvalid && !tready |=> tvalid &&
      $stable({tdata, tkeep, tlast, tuser});
endproperty

assert property (p_stream_stable_when_stalled);
~~~

This is an illustrative assertion. Adapt the sampled fields and reset semantics to the actual interface.

### 17.4 Coverage

Cross:
- direction × length bucket × alignment,
- descriptor count × ring wrap × interrupt policy,
- AXI response × stall duration,
- stream packet size × TLAST position,
- reset state × outstanding count,
- address region × IOMMU permission,
- channel priority × contention pattern.

## 18. Post-silicon bring-up and debug

### 18.1 First bring-up checklist

1. Confirm clocks, reset release and power-domain state.
2. Read IP version/capability registers.
3. Verify control-register access and reset values.
4. Test a small coherent or uncached buffer in a known-safe region.
5. Verify DMA address width and address translation.
6. Run one aligned memory-to-memory transfer.
7. Compare source/destination using an independent CPU path.
8. Test peripheral-to-memory and memory-to-peripheral separately.
9. Enable interrupts only after polling mode is proven.
10. Add SG, ring wrap, cacheable buffers and multiple channels incrementally.

### 18.2 Useful observability

- Current descriptor address and ring pointers.
- Last completed descriptor and sequence number.
- Outstanding read/write count.
- AXI AR/AW/W/R/B handshake counters.
- Stall cycles by channel.
- FIFO high-water/low-water marks.
- Bytes requested vs bytes completed.
- Error code and fault address.
- IOMMU fault status and stream/device ID.
- Interrupt cause, mask and coalescing counters.
- Reset/abort state and watchdog timeout.
- DRAM controller QoS and latency counters.

### 18.3 Triage by symptom

**No data moves:** check clock/reset, channel enable, descriptor ownership, address decode, IOMMU permissions and request handshake.

**Data moves but wrong:** check address mapping, cache maintenance, byte lane/strobe, width conversion, endianness, descriptor length and source/destination overlap.

**Correct data but low throughput:** check burst length, outstanding depth, FIFO occupancy, memory QoS, read/write turnaround and interrupt frequency.

**Only fails under concurrency:** check ring ownership races, channel arbitration, coherency, interconnect ordering, shared FIFO credits and reset sequencing.

## 19. Reference implementation case studies

### AMD AXI DMA PG021

Use the public PG021 guide to study an implementation with:
- MM2S and S2MM channels,
- AXI4 memory-mapped data paths,
- AXI4-Stream peripheral paths,
- AXI4-Lite control,
- optional SG descriptor fetch/update,
- direct-register mode,
- cyclic mode and optional data realignment.

The guide is useful for understanding a concrete register map and programming sequence. It is not a generic DMA standard.

### OpenTitan DMA

OpenTitan documents a single-channel DMA that moves data among memory-mapped regions using TL-UL and a custom system interface. Its documentation describes memory-to-memory, memory-to-peripheral and peripheral-to-memory transfers; incrementing, fixed and circular addressing; hardware handshakes; interrupts; and hardware isolation. This is a useful contrasting architecture to a descriptor-heavy AXI DMA.

### Linux DMAengine and DMA API

Linux separates the peripheral/client driver from the DMA controller driver:
- client requests a channel,
- configures peripheral address, width and burst,
- prepares a slave SG/cyclic/interleaved descriptor,
- submits it,
- issues pending work,
- receives completion callback.

The DMA API separately handles device-visible addresses, mapping, coherency and synchronization. Do not confuse the DMAengine software descriptor with a hardware descriptor fetched by the controller; they are related abstractions but not necessarily the same object.

## 20. Hands-on lab: 32-channel audio DMA

### Scenario

Design a conceptual full-duplex audio DMA path:
- 32 channels,
- 48 kHz sample rate,
- 32-bit container per channel,
- playback and capture,
- 128-bit AXI data bus at 500 MHz,
- cyclic period buffers,
- non-coherent payload path for the exercise.

### Tasks

1. Calculate playback, capture and combined payload bandwidth.
2. Choose a period duration and calculate bytes per period.
3. Choose a ring depth and calculate total buffering time.
4. Define descriptor fields for buffer DMA address, capacity, actual bytes, period ID, direction and ownership.
5. Draw MM2S and S2MM paths independently.
6. Define the FIFO handshake between DMA and audio peripheral.
7. Specify how the DMA handles the final partial beat.
8. Write a software ownership sequence for playback and capture.
9. Add cache synchronization points for a non-coherent system.
10. Define completion/period interrupt policy.
11. Create tests for FIFO backpressure and delayed interrupt service.
12. Inject an IOMMU fault and verify error reporting and safe recovery.

### Expected design review questions

- Can playback and capture use independent channels?
- Is the audio clock asynchronous to the DMA clock?
- Does the peripheral generate request pulses or a level/watermark?
- Can one descriptor span multiple audio periods?
- What happens if software misses a period deadline?
- Is a buffer reused before hardware completion?
- Is the DMA address an IOVA or a physical address?
- What is the worst-case memory arbitration latency?
- Does reset preserve or discard queued descriptors?
- How will the debug trace distinguish peripheral starvation from DRAM starvation?

## 21. Interview and review questions

1. Why is DMA useful if the CPU can execute load/store instructions?
2. What is the difference between a DMA channel and a DMA engine?
3. Explain bus-master DMA versus a legacy third-party DMA controller.
4. What is the difference between simple mode and scatter/gather mode?
5. Why does a descriptor need ownership/status?
6. What is the purpose of a tail pointer or doorbell?
7. How does cyclic DMA differ from a finite descriptor chain?
8. Why can a scatterlist map to fewer DMA segments than the original CPU scatterlist?
9. Why must the driver use the DMA address returned by the DMA API?
10. What is the difference between cache coherency and memory ordering?
11. When are dma_sync_* operations needed?
12. Why can coherent descriptor memory still require dma_wmb/dma_rmb?
13. Why must AXI bursts be split at a 4-KiB boundary?
14. How do outstanding transactions improve throughput?
15. How would you size a FIFO for a known maximum downstream stall?
16. How does an IOMMU help isolate a DMA-capable device?
17. What is the risk of reusing a descriptor before completion?
18. How would you debug an RX buffer that contains old data?
19. How can interrupt coalescing improve throughput, and what latency does it add?
20. How would you prove that reset cannot cause duplicate writes or descriptor replay?

## 22. Key takeaways

- DMA is a bus-initiating data-movement architecture; it is not one universal protocol.
- The CPU configures and manages ownership; the engine moves the payload.
- Descriptor rings improve autonomy and queue depth but create ordering and ownership problems.
- Scatter/gather supports fragmented memory; cyclic DMA supports continuous streams.
- AXI handshake, burst boundaries, IDs, responses and backpressure must be handled correctly.
- CPU virtual, physical and DMA/IOVA addresses are distinct concepts.
- Cache maintenance, DMA barriers and MMIO ordering solve different problems.
- IOMMU and firewalls are essential parts of safe DMA integration.
- Performance depends on outstanding depth, FIFO sizing, memory QoS and DRAM behavior—not just bus width.
- Verification must cover data integrity, boundaries, ownership, reset, concurrency and faults.
- Post-silicon observability should expose descriptors, byte counts, stalls, FIFO levels, fault addresses and outstanding transactions.

---

## References and further study

### Historical and architectural sources

1. Mark Smotherman, *A Survey and Taxonomy of I/O Systems*, updated July 2006: https://mark.people.clemson.edu/io_hist.html
2. Intel, *8237A High Performance Programmable DMA Controller* data sheet (public scan): https://www.pcjs.org/documents/datasheets/intel/INTEL_8237A_DMA.pdf
3. IBM, *Personal Computer AT Technical Reference*, September 1985 (DMA chapter): https://www.bitsavers.org/pdf/ibm/pc/at/6139362_PC_AT_Technical_Reference_Sep85.pdf

### Protocol and implementation references

4. Arm, AMBA AXI and ACE Protocol Specification, IHI 0022: https://developer.arm.com/documentation/ihi0022/latest/
5. AMD, *AXI DMA LogiCORE IP Product Guide PG021 v7.1*, release 24 June 2025: https://docs.amd.com/r/en-US/pg021_axi_dma/
6. AMD PG021 feature summary: https://docs.amd.com/r/en-US/pg021_axi_dma/Feature-Summary
7. AMD PG021 scatter/gather mode: https://docs.amd.com/r/en-US/pg021_axi_dma/Scatter-Gather-Mode
8. AMD PG021 descriptor management: https://docs.amd.com/r/en-US/pg021_axi_dma/Descriptor-Management
9. OpenTitan DMA documentation: https://opentitan.org/book/hw/ip/dma/index.html

### Software, coherency and address translation

10. Linux kernel, Dynamic DMA Mapping Guide: https://docs.kernel.org/core-api/dma-api-howto.html
11. Linux kernel, DMA Engine API Guide: https://docs.kernel.org/driver-api/dmaengine/client.html
12. Linux kernel, Memory Barriers: https://docs.kernel.org/core-api/wrappers/memory-barriers.html
13. Linux kernel, DMA-BUF framework: https://docs.kernel.org/driver-api/dma-buf.html

### Reading notes

- **Normative:** Arm AXI specification requirements for AXI channel handshakes, bursts, responses and 4-KiB boundaries.
- **Software contract:** Linux DMA API, DMAengine API and memory-barrier documentation.
- **Implementation example:** AMD PG021 register map, descriptor fields and programming sequence.
- **Reference architecture:** OpenTitan DMA.
- **Historical evidence:** Intel 8237A data sheet, IBM PC AT technical reference and Smotherman's historical survey.
- **Interpretation:** generic block diagrams, example descriptor state machines, sizing equations and lab assumptions in this lesson.

Always check the exact IP revision, configuration parameters, interconnect, memory attributes and OS version used by a real design.
