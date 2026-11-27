# SoC IP Masterclass — Basics to Silicon Engineering

A structured, protocol-first learning path from SoC fundamentals to advanced IP architecture, RTL integration, verification and post-silicon debug.

## Start here

1. [Lesson 01 — SoC Foundations: From IP Blocks to a Working Silicon System](lessons/01-soc-foundations.md)
2. [Lesson 02 — On-chip Interconnect Foundations](lessons/02-interconnect-foundations.md)
3. [Full curriculum and roadmap](CURRICULUM.md)

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
└── lessons/
    ├── 01-soc-foundations.md
    └── 02-interconnect-foundations.md
~~~

## Contribution and source discipline

- Prefer official protocol specifications, architecture manuals and vendor programming guides.
- Use public RTL/reference designs to explain concrete implementations.
- Do not copy proprietary specification text, figures or tables. Write original explanations and diagrams and cite the authoritative source.
- State assumptions for all performance calculations.
- Never generalize an implementation-specific behavior as a protocol requirement.
- Keep lesson numbering sequential and update the curriculum checklist when a lesson is published.

## License

The repository currently contains a root-level MIT LICENSE. Unless a later commit explicitly introduces per-directory licensing, repository contributions are covered by that existing license. Preserve required third-party notices and do not add material whose license is incompatible.
