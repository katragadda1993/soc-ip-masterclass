# SoC IP Masterclass — Curriculum

A sequential Basics-to-Masterclass journey from system architecture to protocol-level IP engineering, integration, verification and post-silicon debug.

## Teaching standard for every IP

Every IP/protocol lesson should include:

1. Purpose, use cases and system context.
2. Historical backstory: original problem, dated milestones, predecessor designs and documented design rationale.
3. Architecture: block diagram, sub-blocks, interfaces and data/control paths.
4. Protocol: signals, transaction types, state machines, framing, timing and error handling.
5. Registers and software: address map, reset values, programming sequence and driver interaction.
6. Performance: bandwidth, latency, buffering, arbitration, QoS and power.
7. Integration: address, clock, reset, power, interrupts, DMA, security, pinmux and PHY dependencies.
8. Verification: assertions, directed/random tests, coverage, CDC/RDC, formal and fault injection.
9. Silicon: bring-up, characterization, trace/debug methodology and common failure signatures.
10. Practical exercises, calculations, interview questions and authoritative references.

Historical claims must be sourced and dated. Distinguish normative specification requirements, documented implementation details and engineering interpretation. Never invent proprietary design stories.

## Curriculum roadmap

| Session | Module | Coverage |
|---:|---|---|
| 01 | SoC foundations | SoC anatomy, IP taxonomy, address maps, transaction journeys, clock/reset/power/boot, integration contracts |
| 02 | Interconnect foundations | Masters/hosts, targets/devices, decode, arbitration, shared bus, crossbar, bridge, ordering, backpressure |
| 03 | APB | Setup/access phases, PSEL/PENABLE/PREADY/PSLVERR, register interface, bridge, low-power rationale |
| 04 | AHB / AHB-Lite | Address/data pipelining, HTRANS, HREADY, bursts, arbitration, wait states, split/retry history |
| 05 | AXI fundamentals | Five channels, VALID/READY, independent address/data, IDs, responses, outstanding transactions |
| 06 | AXI advanced | Bursts, unaligned transfers, strobes, ordering, QoS, atomics, AXI-Lite, AXI-Stream, width conversion and performance |
| 07 | Interconnect implementation | Crossbar microarchitecture, muxing, buffering, routing, deadlock, performance and timing closure |
| 08 | NoC | Routers, links, virtual channels, packetization, topology, routing, flow control, QoS, coherent transport, deadlock and performance |
| 09 | CPU subsystem | ISA vs microarchitecture, pipeline, hazards, retirement, privilege, MMU/MPU, TLB, PMP/PMA, exceptions, interrupts, debug, boot and bus adapters |
| 10 | Cache and coherency | Cache hierarchy, write policies, snoop, directory, ACE/CHI concepts, barriers and DMA coherency |
| 11 | SRAM / ROM / OTP | Macro interfaces, banking, ECC, scrambling, retention, BIST, ROM integrity and fuse semantics |
| 12 | DDR / LPDDR | DRAM organization, ACT/PRE/RD/WR/REF commands, timing, refresh, bank scheduling, address mapping, bandwidth/latency, DDR/LPDDR/HBM trade-offs, PHY training, SI/PI, ECC/RAS and silicon characterization |
| 13 | DMA architecture | Descriptors, rings, scatter-gather, alignment, IOMMU, coherency, interrupts and throughput |
| 14 | Clock architecture | PLLs, reference sources, frequency synthesis, muxes, dividers, glitch-free switching, ICG/clock gating, jitter, phase noise, CTS topologies, skew, setup/hold, CDC/RDC, clock monitors, DVFS sequencing and silicon characterization |
| 15 | Reset architecture | POR, external/brownout/watchdog/software/debug reset sources, reset trees, asynchronous assertion/synchronous release, reset sequencing, quiesce/drain/isolation, RDC, reset-cause retention, power/clock interaction and silicon recovery |
| 16 | Power management | Power domains, isolation, retention, DVFS, idle handshakes, wakeup and power-state coordination |
| 17 | Interrupts and timers | Edge/level, priority, PLIC/GIC concepts, claim/complete, timer, watchdog and latency |
| 18 | GPIO and pinmux | Pad ring, drive, pulls, muxing, debounce, straps, interrupts and electrical constraints |
| 19 | I2C | Open-drain signaling, START/STOP, ACK/NACK, arbitration, clock stretching, timing and bus recovery |
| 20 | SPI | Modes, chip select, full duplex, FIFO, DMA, timing and multi-device topologies |
| 21 | I3C | SDR/HDR, dynamic addressing, CCCs, in-band interrupts, legacy I2C coexistence |
| 22 | UART | Sampling, baud generation, framing, FIFOs, flow control and error detection |
| 23 | USB | Host/device, enumeration, descriptors, endpoints, transfers, controller DMA and PHY |
| 24 | PCIe | LTSSM, config space, BARs, TLPs, DMA, MSI/MSI-X, ordering, AER and PHY |
| 25 | Ethernet | MAC, PCS/PMA, DMA rings, descriptors, timestamping, flow control and link training |
| 26 | Storage | eMMC/UFS/NVMe concepts, command queues, DMA, boot partitions and error recovery |
| 27 | I2S / TDM | Framing, clocks, slots, sample packing, clock master/slave and audio DMA |
| 28 | SoundWire / HDA | Discovery/enumeration, frames, streams, dataports, CORB/RIRB, interrupts and codec integration |
| 29 | Display | Timing generator, planes, blending, memory fetch, bandwidth, underflow and PHY |
| 30 | Camera | CSI-2, D-PHY/C-PHY concepts, packet parsing, ISP, line buffers and DMA |
| 31 | Security | Secure boot, roots of trust, AES/SHA, key manager, lifecycle, firewalls and side-channel concerns |
| 32 | Debug and trace | JTAG, CoreSight concepts, trace funnels, triggers, scan, MBIST and silicon observability |
| 33 | Accelerators | GPU/NPU/DSP, local memory, command queues, DMA, cache, interrupts and power |
| 34 | PHY and SerDes | PLL/CDR, equalization, training, calibration, jitter, eye diagrams and lane management |
| 35 | Full SoC integration | IP selection, topology, address map, clock/reset/power, security and software contracts |
| 36 | Verification masterclass | UVM, assertions, formal, coverage closure, CDC/RDC, emulation and FPGA prototyping |
| 37 | Post-silicon validation | Bring-up, test plans, characterization, performance counters, trace and failure triage |
| 38 | Capstone | Build a reference SoC architecture, integration checklist, verification plan and debug playbook |

## Session status

- [x] 01 — SoC Foundations ([lesson](lessons/01-soc-foundations.md))
- [x] 02 — Interconnect Foundations ([lesson](lessons/02-interconnect-foundations.md))
- [x] 03 — APB ([lesson](lessons/03-apb-protocol-deep-dive.md))
- [x] 04 — AHB / AHB-Lite ([lesson](lessons/04-ahb-ahb-lite-protocol-deep-dive.md))
- [x] 05 — AXI Fundamentals ([lesson](lessons/05-axi-fundamentals-five-channels.md))
- [x] 06 — AXI Advanced ([lesson](lessons/06-axi-advanced-transactions-performance.md))
- [x] 07 — AXI Interconnect Implementation ([lesson](lessons/07-axi-interconnect-implementation.md))
- [x] 08 — Network-on-Chip ([lesson](lessons/08-network-on-chip-deep-dive.md))
- [x] 09 — CPU Subsystem ([lesson](lessons/09-cpu-subsystem-deep-dive.md))
- [x] 10 — Cache and Coherency ([lesson](lessons/10-cache-and-coherency-deep-dive.md))
- [x] 11 — SRAM / ROM / OTP ([lesson](lessons/11-sram-rom-otp-deep-dive.md))
- [x] 12 — DDR / LPDDR ([lesson](lessons/12-ddr-lpddr-controller-phy-deep-dive.md))
- [x] 13 — DMA Architecture ([lesson](lessons/13-dma-architecture-descriptors-rings-coherency.md))
- [x] 14 — Clock Architecture ([lesson](lessons/14-clock-architecture-plls-clock-trees-cdc.md))
- [x] 15 — Reset Architecture ([lesson](lessons/15-reset-architecture-reset-trees-rdc.md))
- [ ] 16–38 — Remaining sessions — planned

## Current next lesson

- [x] 10 — Cache and Coherency: cache lines, tags, replacement, write policies, miss handling, coherence protocols, ACE/CHI concepts, DMA coherency and cache maintenance ([lesson](lessons/10-cache-and-coherency-deep-dive.md)).
- [x] 11 — SRAM / ROM / OTP: macro interfaces, 6T bitcell operation, banking, ECC/integrity, scrambling, retention, MBIST/BISR, ROM integrity and OTP provisioning ([lesson](lessons/11-sram-rom-otp-deep-dive.md)).
- [x] 12 — DDR / LPDDR: 1T1C DRAM, command/state model, timing scoreboard, controller scheduling, address mapping, bandwidth/latency, PHY training, LPDDR power states, ECC/RAS, verification and silicon debug ([lesson](lessons/12-ddr-lpddr-controller-phy-deep-dive.md)).
- [x] 13 — DMA Architecture: descriptors, rings, scatter-gather, alignment, IOMMU, coherency, interrupts and throughput ([lesson](lessons/13-dma-architecture-descriptors-rings-coherency.md)).
- [x] 14 — Clock Architecture: reference sources, PLL frequency synthesis, muxes/dividers, glitch-free switching, ICG, CTS, jitter/skew, setup/hold, CDC/RDC, reset sequencing, DVFS, clock monitoring and post-silicon characterization ([lesson](lessons/14-clock-architecture-plls-clock-trees-cdc.md)).
- [x] 15 — Reset Architecture: POR, reset-source taxonomy, reset trees, asynchronous assertion/synchronous release, reset sequencing, quiesce/drain/isolation, RDC, reset-cause retention, clock/power interaction and silicon recovery ([lesson](lessons/15-reset-architecture-reset-trees-rdc.md)).
- [ ] 16 — Power Management: power domains, isolation, retention, DVFS, idle handshakes, wakeup and power-state coordination.

## Reference design philosophy

Use public reference implementations such as OpenTitan, lowRISC Ibex, CVA6, open-source RISC-V SoCs and public vendor documentation to connect theory to real RTL. Clearly label which details are specific to a reference implementation and which are protocol requirements.
