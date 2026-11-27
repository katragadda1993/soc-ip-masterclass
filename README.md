# SoC IP Masterclass — Basics to Silicon Engineering

A structured, protocol-first learning path from SoC fundamentals to advanced IP architecture, RTL integration, verification and post-silicon debug.

## Start here

1. [Lesson 01 — SoC Foundations: From IP Blocks to a Working Silicon System](lessons/01-soc-foundations.md)
2. [Lesson 02 — On-chip Interconnect Foundations](lessons/02-interconnect-foundations.md)
3. [Lesson 03 — AMBA APB: Protocol, RTL Architecture and Silicon-Level Debug](lessons/03-apb-protocol-deep-dive.md)
4. [Lesson 04 — AMBA AHB and AHB-Lite: Protocol, RTL Architecture and Silicon-Level Debug](lessons/04-ahb-ahb-lite-protocol-deep-dive.md)
5. [Lesson 05 — AMBA AXI Fundamentals: The Five Independent Channels](lessons/05-axi-fundamentals-five-channels.md)
6. [Lesson 06 — AXI Advanced: Bursts, Ordering, QoS, Atomics and Data Movement](lessons/06-axi-advanced-transactions-performance.md)
7. [Lesson 07 — AXI Interconnect Implementation: Crossbar Microarchitecture, Routing, Arbitration and Debug](lessons/07-axi-interconnect-implementation.md)
8. [Lesson 08 — Network-on-Chip: Architecture, Routers, Routing, Flow Control and Silicon Debug](lessons/08-network-on-chip-deep-dive.md)
9. [Lesson 09 — CPU Subsystem: From Instruction Execution to Silicon Integration](lessons/09-cpu-subsystem-deep-dive.md)
10. [Lesson 10 — Cache and Coherency: From SRAM Tags to Multi-Agent Consistency](lessons/10-cache-and-coherency-deep-dive.md)
11. [Lesson 11 — SRAM, ROM and OTP: From Bitcell Physics to Secure SoC Memory Integration](lessons/11-sram-rom-otp-deep-dive.md)
12. [Lesson 12 — DDR / LPDDR: DRAM Architecture, Controller Scheduling, PHY Training and Silicon Debug](lessons/12-ddr-lpddr-controller-phy-deep-dive.md)
13. [Lesson 13 — DMA Architecture: Descriptors, Rings, Scatter/Gather, IOMMU, Coherency and Silicon Debug](lessons/13-dma-architecture-descriptors-rings-coherency.md)
14. [Lesson 14 — Clock Architecture: PLLs, Clock Trees, Gating, CDC and Silicon Timing](lessons/14-clock-architecture-plls-clock-trees-cdc.md)
15. [Lesson 15 — Reset Architecture: POR, Reset Trees, Sequencing, RDC and Silicon Recovery](lessons/15-reset-architecture-reset-trees-rdc.md)
16. [Lesson 16 — Power Management: Domains, UPF, DVFS, Idle Handshakes and Wakeup](lessons/16-power-management-domains-upf-dvfs-wakeup.md)
17. [Lesson 17 — Interrupt Controllers and Timers: GIC, PLIC, AIA, Generic Timer, mtime, Watchdogs and Silicon Debug](lessons/17-interrupt-controllers-timers-deep-dive.md)
18. [Lesson 18 — GPIO and Pin Control: Pad Ring, Electrical Behavior, Pinmux, Pinctrl and Silicon Debug](lessons/18-gpio-pin-control-pad-ring-pinmux.md)
19. [Lesson 19 — I²C Bus: Protocol, Controller RTL, Electrical Design and Silicon Debug](lessons/19-i2c-protocol-controller-rtl-electrical-debug.md)
20. [Lesson 20 — SPI: Protocol, Controller RTL, Flash, QSPI and Silicon Debug](lessons/20-spi-protocol-controller-rtl-flash-qspi-debug.md)

[Full curriculum and roadmap](CURRICULUM.md)

## How each lesson is taught

Every IP is studied from four connected viewpoints:

- **Why it exists:** historical problem, design rationale, predecessor architecture and evolution.
- **How it works:** block architecture, interfaces, signals, state machines, transactions and timing.
- **How to integrate it:** address map, clocks, resets, power, interrupts, DMA, security, PHY and software contracts.
- **How to validate it:** pre-silicon verification, performance characterization, post-silicon bring-up and debug.

Historical claims are dated and sourced. Normative protocol requirements, documented implementation examples and engineering interpretation are kept distinct.

## Repository structure

~~~text
.
├── README.md
├── CURRICULUM.md
├── LICENSE
└── lessons/
    ├── 01-soc-foundations.md
    ├── 02-interconnect-foundations.md
    ├── 03-apb-protocol-deep-dive.md
    ├── 04-ahb-ahb-lite-protocol-deep-dive.md
    ├── 05-axi-fundamentals-five-channels.md
    ├── 06-axi-advanced-transactions-performance.md
    ├── 07-axi-interconnect-implementation.md
    ├── 08-network-on-chip-deep-dive.md
    ├── 09-cpu-subsystem-deep-dive.md
    ├── 10-cache-and-coherency-deep-dive.md
    ├── 11-sram-rom-otp-deep-dive.md
    ├── 12-ddr-lpddr-controller-phy-deep-dive.md
    ├── 13-dma-architecture-descriptors-rings-coherency.md
    ├── 14-clock-architecture-plls-clock-trees-cdc.md
    ├── 15-reset-architecture-reset-trees-rdc.md
    ├── 16-power-management-domains-upf-dvfs-wakeup.md
    ├── 17-interrupt-controllers-timers-deep-dive.md
    ├── 18-gpio-pin-control-pad-ring-pinmux.md
    ├── 19-i2c-protocol-controller-rtl-electrical-debug.md
    └── 20-spi-protocol-controller-rtl-flash-qspi-debug.md
~~~

## Current progress

- Completed: Lessons 01–20.
- Next: [Lesson 21 — I3C](CURRICULUM.md#curriculum-roadmap).

## Contribution and source discipline

- Prefer official protocol specifications, architecture manuals and vendor programming guides.
- Use public RTL/reference designs to explain concrete implementations.
- Do not copy proprietary specification text, diagrams or tables. Write original explanations and diagrams and cite the authoritative source.
- State assumptions for all performance calculations.
- Never generalize an implementation-specific behavior as a protocol requirement.
- Keep lesson numbering sequential and update the curriculum index when a lesson is published.

## License

The repository currently contains a root-level MIT LICENSE. Unless a later commit explicitly introduces per-directory licensing, repository contributions are covered by that existing license. Preserve required third-party notices and do not add material whose license is incompatible.
