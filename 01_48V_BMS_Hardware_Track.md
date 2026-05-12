# 48V Battery Management System — Hardware Track

## Objective

Design and build a production-grade PCB platform for a 48V battery management system that can be adapted across multiple end markets — light electric mobility, micro-mobility, industrial energy storage, backup power — without redesigning the full stack each time.

## Product Scope

- A 48V nominal BMS hardware platform built around a commercially available analog front-end (AFE) with electrochemical impedance spectroscopy (EIS) capability.
- A clean split between the **analog/power board** and the **digital/control board**, so each half can evolve independently and be revised without disturbing the other.
- **Current envelope** — designed for up to **250 A continuous** and **400 A peak** through the pack-side switching path, which covers the practical 48V envelope for both automotive and industrial use cases. Beyond this current level, real-world applications typically migrate to higher-voltage architectures.
- Two current-control variants sharing as much of the platform as possible, both operating within the same envelope but suited to different markets:
  - **Contactor-based** variant — preferred for industrial, stationary, and service-friendly applications.
  - **MOSFET-based** variant — preferred for compact, automotive, and frequent-switching applications.
- **Multi-chemistry support** — the platform should accommodate both **NMC** and **LFP** cells, with chemistry-specific behaviour handled in firmware where possible and chemistry-driven hardware sizing handled by design choice up front (see below).
- **Configurable cell count** — the same hardware should support any cell configuration from a documented minimum (set by the chosen AFE, typically 8s) up to 16s, in the style of established field-configurable BMS products such as JBD Smart BMS or Orion BMS. Wiring guidelines accompany the hardware; firmware adapts to the configured cell count at runtime.

## Design Rationale

### Chemistry Support — What is Hardware, What is Firmware

At 48V nominal, the two target chemistries land at meaningfully different series counts:

- **NMC** (~3.7 V/cell nominal) → roughly 13s–14s
- **LFP** (~3.2 V/cell nominal) → roughly 15s–16s

The AFE's per-cell measurement range typically spans 0–5 V, which covers both. The pack voltage envelope at 48V nominal also lands in a similar window across chemistries, so pre-charge timing, contactor and MOSFET sizing, and isolation barriers stay essentially common.

The **hardware-side** decisions that follow from chemistry choice are:

- AFE selection — must support up to 16 cells in series to cover LFP, which is the binding upper bound. A single 16-cell AFE covers both chemistries cleanly, keeping the digital board and cell-tap harness simple.
- Sense-harness connector pin count, sized for the highest cell count.
- Balance current sizing — implications differ slightly across chemistries based on typical cell capacities encountered in each.

Everything else — voltage thresholds, balancing trigger points, protection trip levels, charging profiles, OCV tables, SOC/SOH models — is firmware and stays out of the hardware scope. The team should explicitly document, per chemistry, which configuration values are firmware-only and which require a hardware change.

### Why EIS Capability

A simple DC resistance measurement only gives a lumped number. EIS measures impedance across a range of frequencies and separates it into distinct contributions — ohmic, SEI/passivation layer, charge-transfer, and diffusion — each of which correlates with different physical mechanisms inside the cell. This unlocks several capabilities a non-EIS BMS cannot provide:

- **Richer SOH** that goes beyond "capacity has faded by X%" and points at *why* it has faded.
- **Independent SOC observable for LFP**, where the very flat OCV curve makes traditional SOC estimation weak; impedance features add a second axis of information.
- **Internal cell temperature estimation** from charge-transfer impedance, which responds faster and more representatively than an external thermistor on the can.
- **Early detection** of lithium plating, SEI breakdown, and internal short precursors before they manifest as capacity loss or thermal events.
- A path to **condition-monitoring as a product feature**, which is a real differentiator in stationary storage, industrial, and second-life markets.

Modern AFEs from TI, ADI, and NXP are starting to integrate EIS stimulus and measurement hardware specifically for these reasons. Picking one that supports EIS up front avoids a hardware respin later when the firmware track wants better SOH or the testing track wants richer characterisation data.

### Robustness and Transient Immunity

A common failure mode of available BMS products — particularly at the cost-driven end of the market — is poor behaviour under real-world electrical conditions: power-line transients, load dump, alternator noise, ESD events, bus-coupled disturbances, and cold-start glitches. The hardware is often electrically functional on a clean bench and unreliable in the field, where the failure presents as an inexplicable shutdown, a wedged AFE, a corrupted measurement, or a phantom protection trip.

The platform treats robustness as a non-negotiable quality bar, not a "nice to have" once the rest works. Expectations:

- All power-line inputs filtered, clamped, and protected to **ISO 7637-2** transient classes appropriate to the target application, with **ISO 16750-2** as the supply specification reference.
- Reverse polarity, overvoltage, and load-dump protection at the input — designed in from the schematic, not bolted on later.
- Signal conditioning on every sensed input (cell taps, current sense, temperature, host bus) — anti-alias filtering, common-mode chokes, TVS clamping where relevant, and isolation where the topology requires it.
- EMC pre-compliance considered during layout, not at the end — return-path management, isolation-barrier integrity, and reference-plane discipline.
- A robustness test plan (handed off to the testing track) that exercises transient immunity, ESD, and reverse-polarity behaviour on every prototype.

## Functional Targets

- Per-cell voltage monitoring across any configuration from the AFE-supported minimum (typically 8s) up to 16s, with the channel-mapping documented so end users can wire up intermediate configurations
- Per-cell or per-module temperature monitoring
- Pack-level current sensing across the **250 A continuous / 400 A peak** envelope, in both directions (charge and discharge), with bandwidth sufficient to catch short-circuit events well before any thermal damage
- Cell balancing — passive vs active to be decided per use case
- Pre-charge, isolation, and safe disconnect circuitry
- Communication interfaces:
  - **CAN** as the primary in-system bus — mandatory on every variant, isolated.
  - **UART** for debug and console access during development and service.
  - **Automotive Ethernet** as an optional placement, populated where high-bandwidth telematics or diagnostics are required.
- Onboard provision for EIS measurements via the chosen AFE
- Hardened protection circuitry against power-line transients, reverse polarity, load dump, ESD, and bus-coupled noise (see Design Rationale)
- Diagnostics that can support field servicing

## Technical Considerations to Own

- **Functional safety target** — the platform is designed to **ASIL-B** at the platform level, with safety-critical functions (contactor/MOSFET disconnect control, overcurrent protection, isolation monitoring where applicable) designed to **ASIL-C** where practical. ASIL-C platform-wide is not a goal for v1 given the STM32 starting point; what matters is that the architecture, diagnostic coverage, and protection paths don't preclude reaching it when the platform migrates to an automotive-grade controller.
- **Qualification path** — industrial-grade components (IEC 61000 family, broad operating temperature, conformal coating) are acceptable for the first prototype. The roadmap target is **automotive grade** (AEC-Q100 for actives, AEC-Q200 for passives) on the controller migration. Component selection from the start should bias toward parts that have an AEC-Q-graded equivalent in the same family, so the migration is a substitution rather than a redesign.
- Selection between candidate AFEs (TI, Analog Devices, NXP, or equivalent) — evaluated against:
  - Supported cell-count range, including a low enough minimum (≤ 8s) to support the configurable-cell-count requirement, and a high enough maximum (≥ 16s) to support LFP.
  - EIS stimulus and measurement capability (not all parts have this; it should be a hard filter rather than a nice-to-have).
  - Measurement accuracy, balancing capability, qualification grade (with an automotive-graded variant in the same family preferred), and supply chain.
- **Current sensing approach** — the team is free to choose between candidate topologies, but should compare them explicitly before committing:
  - **Shunt-based** (low-side or high-side with isolated amplifier) — typically the most accurate, lowest drift, but harder to lay out cleanly for the 250 A envelope, requires careful Kelvin sensing, and dissipates real power at the high end of the envelope.
  - **Hall-effect / fluxgate** — galvanically isolated by construction, no insertion loss, easy to lay out, but more drift and offset, more sensitive to nearby ferrous structures and stray fields.
  - **Hybrid (shunt + Hall)** — shunt for steady-state accuracy, Hall for fast peaks and short-circuit detection — used in higher-end automotive BMS designs.
- How much functionality is shared between the contactor and MOSFET variants vs duplicated.
- Connector strategy for cells, temperature sensors, current sensors, and HV interconnects — accommodating the configurable cell-count requirement without proliferating SKUs.
- Layout considerations for current sense accuracy, EMI, isolation barriers, creepage and clearance, and thermal flow.
- Test points and debug headers (JTAG/SWD for development, lockable for production), and serviceability features.
- Conformal coating compatibility, vibration tolerance, and any IP-rating implications on the layout.

## Starting Points

- Reference designs and eval boards for the shortlisted AFEs can be used to bootstrap and validate the architecture before committing to a custom layout.
- Where possible, the digital/control board should reuse the controller chosen by the firmware track to keep the early bring-up simple.

## Deliverables

- A comparative study of candidate AFEs against the functional targets, with a justified recommendation.
- Schematic and layout for the analog/power board (both variants).
- Schematic and layout for the digital/control board.
- A working prototype of both variants, brought up to a known good state.
- Bill of materials with sourcing notes, lead times, and second-source options for critical parts.
- Hardware design documentation: derating analysis, isolation strategy, EMC considerations, thermal calculations.
- A bring-up checklist and known-issue log handed off to the firmware and testing tracks.

## Out of Scope

- Cell chemistry selection or qualification.
- End-application enclosure and packaging (covered by the mechanical track).
- Algorithm development (covered by the firmware track).

## Inter-Track Dependencies

- **Firmware track** — consumes this hardware as the target platform; needs early access to bring-up boards.
- **Testing track** — uses these boards as DUTs and needs visibility into test points and diagnostics; co-owns the robustness, transient-immunity, and EMC pre-compliance test plans.
- **Mechanical track** — coordinates connector positions, mounting points, and any pack-level geometry constraints.
