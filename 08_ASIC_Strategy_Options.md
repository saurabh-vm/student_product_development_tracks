# India-Aligned ASIC Options for the EV Platform

## Context

The engineering work already underway — BMS hardware, firmware, testing, and pack design; VCU design (hardware and firmware tracks); the vehicle mockup; and the vehicle monitoring system — generates deep, specific knowledge of what silicon a serious Indian EV platform needs. That knowledge is the foundation for an ASIC programme that is genuinely aligned with the team's work rather than a side project.

Two ASIC candidates emerge from this foundation. Both target real, unmet needs in the Indian EV ecosystem. Both can be developed by a student team with appropriate mentorship, scoped to be achievable rather than aspirational. Both can be FPGA-prototyped before tapeout (with caveats noted below) and fabricated on mature CMOS or BCD process nodes that are accessible to Indian designers today.

This document captures both options, the skills and pathway needed for students to deliver them, the FPGA-prototyping story, and the process-node considerations.

## Option A — Battery Management Analog Front-End (AFE)

### What it is

A purpose-designed mixed-signal IC that handles cell-voltage sensing, current sensing, cell balancing, fault detection, and host communication for a Li-ion battery pack. The chip sits between the cells and the host MCU (the BMS controller). It is the most safety-critical and the most cost-sensitive chip in a BMS.

### Reference incumbents

- **Texas Instruments**: BQ76952 (16-cell), BQ76942 (10-cell), BQ79616 (16-cell automotive)
- **Analog Devices**: LTC6813 (18-cell), LTC6811 (12-cell), MAX17853
- **NXP**: MC33775A (14-cell automotive)
- **Renesas**: RAA489204
- **STMicroelectronics**: L9963E (14-cell automotive)

All of these are imported by Indian BMS makers. None are designed for the cost, performance, and cell-count profile that the Indian 2W/3W/micro-EV market actually uses.

### The Indian opportunity

India is the world's largest 2W EV market by volume. Every Indian 2W BMS uses an imported AFE. The supply pain is real — automotive-grade analog IC lead times have repeatedly stretched to extreme values, and Indian BMS makers pay spot-market prices for parts that are 70% overspecified for their application.

A focused AFE targeting **8s–16s LFP and NMC packs for 2W, 3W, and micro-EV applications**, with Indian-market cost-and-feature optimisation, has a real, identifiable customer base. Ather, Ola, TVS, Bajaj, Hero, and 50+ smaller 2W makers all need this chip. The same chip, with minor variants, serves the 3W and micro-EV (Vayve-class) market.

### What we'd build

- **Cell-voltage sensing** for 8 to 16 cells in series, ±1 mV accuracy at room temperature, ±5 mV across automotive temperature range.
- **Coulomb-counter / current-sensing** front-end with at least 16-bit resolution.
- **Passive balancing** with per-cell control and current-limit protection.
- **Temperature sensing** for at least 4 NTC channels.
- **Communication** with the host MCU over an isolated SPI or daisy-chain protocol (the latter for multi-AFE stacks).
- **Safety and fault** features — overvoltage, undervoltage, overcurrent, overtemperature, communication-timeout, with hardware-enforced safe-state behaviour.
- **Operating envelope** — automotive-grade temperature, AEC-Q100 grade 2 target, ASIL-B capable.

### Process and node

A BMS AFE needs **BCD (Bipolar-CMOS-DMOS)** or HV-CMOS, not pure CMOS. The high-voltage transistors handle the per-cell voltage taps (60–120V common-mode), the bipolar devices handle the precision analog (bandgap references, low-offset op-amps), and the CMOS handles the digital control logic. All three coexist on the same die in a BCD process.

**Mature BCD nodes the design can target:**

- **180nm BCD** — Tower Semiconductor, HHGrace, X-Fab, Dongbu. The default starting point.
- **130nm BCD** — same foundries; tighter integration if needed.
- **90nm BCD** — available at fewer foundries; usually overkill for this application.

**SCL Mohali's 180nm CMOS process can host the digital control portion of the chip but does not currently offer the BCD variant needed for the analog front-end.** A partner-foundry strategy (Tower or HHGrace) is the realistic path. The fully-indigenous fabrication narrative for the AFE is therefore weaker than for a pure-digital chip — SCL alone cannot fabricate it today.

### Skill profile needed

The AFE is an analog-heavy chip. The team will need:

- **Senior analog mentorship** — at least one experienced mixed-signal designer guiding the team. Analog design is craft-heavy and unforgiving; student teams without analog mentorship will struggle.
- **SPICE-level circuit design** — bandgap references, low-offset op-amps, comparators, sigma-delta ADCs, level shifters.
- **High-voltage layout** — guard rings, deep N-well isolation, well-tap discipline, ESD protection sized for automotive transients.
- **Mixed-signal verification** — co-simulating analog and digital, corner analysis, Monte Carlo on matching-critical circuits.
- **BCD process familiarity** — the foundry-specific PDK, available device types, layout rules.

### FPGA prototyping

The digital control logic, the balancing FSM, the ADC sequencer, and the communication protocol can all be prototyped on an FPGA before silicon. The standard approach:

- Build an **evaluation board** with a commercial AFE (TI BQ76952 or ADI LTC6813) connected to an FPGA.
- The FPGA runs the **digital control architecture** the team plans to integrate into the future ASIC — its FSM, its ADC sequencer logic, its communication protocol, its safety supervisor.
- This validates the digital architecture and the system-level behaviour against real cells and real currents.
- The custom analog is then designed in SPICE/Cadence Virtuoso, validated through multi-project-wafer (MPW) test silicon, and finally integrated with the validated digital architecture.

By the time the team commits to a full-mask tapeout, the digital portion is silicon-proven on FPGA and the analog portion is silicon-proven on MPW. The risk is bounded.

---

## Option B — Generic 32-bit Automotive Microcontroller

### What it is

A fully digital microcontroller in the class of STM32F4/G4 with automotive variants. The chip handles vehicle-level control and zonal-controller duties — running the VCU firmware, zone controllers, BMS host MCU, motor-controller supervisors, and charging-station controllers.

### Reference incumbents at the targeted spec point

- **STMicroelectronics**: STM32 automotive variants (SPC5, STM32 Automotive)
- **NXP**: S32K1xx family (the entry-level automotive MCU)
- **Renesas**: RH850/F1KM-S1, RL78 automotive
- **GigaDevice**: GD32 automotive variants
- **Artery**: AT32

This is the chip class that Indian 2W EV makers actually buy today. It is *not* AURIX, S32K3, or TMS570 territory — those are higher-end safety MCUs for passenger cars, ASIL-D, and the team would lose competing with them. The targeted spec point is the entry-level automotive MCU that runs zonal controllers and 2W VCUs.

### The Indian opportunity

Every Indian EV uses these MCUs. A typical 2W EV has 3–6 of them (BMS controller, VCU, motor controller, dashboard, charging controller, optionally telematics). A small passenger car has 15–30 of them across zones. India's EV market is in the millions of units per year; even modest market share creates volumes that justify silicon NRE.

India has zero indigenous automotive MCU production today. Every chip in this class deployed on an Indian EV is imported. A chip that matches or exceeds the imported alternative on the dimensions the customer actually cares about — cost, availability, performance, peripheral fit, qualification — has a clear market.

### What we'd build

- **CPU core** — Cortex-M4F licensed from ARM, *or* a RISC-V core from the open-source ecosystem (CV32E40P, Ibex, NEORV32, IIT Madras's Shakti family, C-DAC's VEGA family). RISC-V is the more accessible path for a student-led effort and avoids ARM licensing.
- **Clock** — 100–200 MHz target.
- **Memory** — 512 KB to 1 MB Flash, 128–256 KB SRAM.
- **Communication** — 4–6 CAN-FD, 2 LIN, 2 SPI, 2 I²C, 1 Ethernet MAC (optional), 1 USB (optional).
- **Analog** — 16–24 channel 12-bit SAR ADC, 2 DAC channels.
- **Motor control** — 8–16 PWM channels with motor-control-friendly features (centre-aligned, dead-time insertion, complementary outputs, fault inputs).
- **Safety features** — hardware watchdog, ECC on Flash and SRAM, lockstep optional (not for v1), basic CRC, optional lightweight crypto accelerator.
- **Operating envelope** — AEC-Q100 grade 2 target, **ASIL-B capable** (not ASIL-D). ASIL-B covers most 2W and zonal-controller applications.

### Process and node

The MCU is **fully digital with some embedded analog (ADC, DAC, PLL, oscillators)**. Pure-CMOS or low-mixed-signal CMOS is sufficient. Process options:

- **180nm CMOS** — SCL Mohali. Maximum clock speed ~50 MHz with aggressive design. Acceptable for a zonal controller, tight for a VCU.
- **90nm CMOS** — UMC, SMIC, GlobalFoundries. Good balance for a 100 MHz part.
- **65nm CMOS** — TSMC, Samsung, GlobalFoundries. Sweet spot for a 200 MHz Cortex-M4F class part. Matches modern STM32F4 performance.
- **55nm or 40nm** — matches mainstream automotive MCU node; better performance and power.

**SCL Mohali at 180nm is genuinely viable for a zonal-controller variant of this chip.** A 50 MHz Cortex-M0+ or RISC-V core with CAN, GPIO, ADC, and basic peripherals fits the 180nm envelope. This is the "made fully in India" version of the chip — even if it's a step behind in performance, the indigenous fabrication path is fully intact. A higher-performance variant (200 MHz Cortex-M4F class) would use a 65nm partner foundry.

### Skill profile needed

The MCU is digital-heavy with some analog. The team will need:

- **Digital design** at production-RTL quality — Verilog or SystemVerilog, FSMs, pipelines, memory interfaces, bus protocols.
- **CPU architecture awareness** — how a Cortex-M or RISC-V core works internally, what's licensable, what needs to be designed.
- **Peripheral IP design** — CAN-FD controllers, SPI/I²C masters and slaves, UART, ADC sequencer, timers, DMA controllers.
- **Bus protocols** — AMBA AHB/APB or TileLink for the internal interconnect.
- **Verification** — UVM eventually, basic constrained-random and assertion-based verification from day one.
- **Physical design** — synthesis, place-and-route, timing closure, DRC, LVS. Open-source tools (OpenROAD, OpenLane) can take a team most of the way; commercial tools (Synopsys, Cadence) close the gap.
- **Some analog** — the ADC, DAC, PLL, and oscillators are mixed-signal. Often licensed as hardened IP from foundry libraries rather than designed from scratch.

### FPGA prototyping

**The MCU is fully FPGA-prototypable.** This is the standard industry path for new MCU designs, and it is exactly the educational pathway that suits a student team:

- Synthesise the CPU core (RISC-V open-source, or licensed ARM Cortex-M) plus all peripherals onto a Xilinx Artix-7 / Kintex-7 / Zynq, or an Intel Cyclone V FPGA.
- Run **real firmware** on it — including the actual VCU firmware the team is already developing for the STM32-based bring-up VCU. The application code is portable.
- Validate every peripheral, every interrupt, every bus interaction.
- Run the same test suite the VCU design track runs against the STM32 bring-up board.
- Iterate the RTL until it is silicon-ready.

By the time the chip tapes out, **every line of RTL has been exercised by real firmware running real automotive scenarios on real-time hardware**. The risk of a digital functional bug surviving to silicon is small.

This is the strongest argument for the MCU as a student-led project: the validation infrastructure (the mockup track) already exists, the firmware (the VCU design track) is already being written, and the FPGA prototype turns the future ASIC into the existing development target. The work compounds rather than diverging.

---

## Side-by-Side Comparison

### Technical Profile

| Dimension | BMS AFE | Generic 32-bit MCU |
| --- | --- | --- |
| Domain | Mixed-signal (analog + digital) | Digital with embedded analog IP |
| Process required | BCD (Bipolar-CMOS-DMOS) | Pure CMOS |
| Node options | 180nm / 130nm / 90nm BCD | 180nm / 90nm / 65nm / 40nm CMOS |
| SCL Mohali fabricable? | No (BCD not available) | Yes, for a 180nm zonal variant |
| Partner foundries | Tower, HHGrace, X-Fab, Dongbu | Tower, UMC, SMIC, GlobalFoundries, TSMC |
| FPGA prototype | Partial (digital + commercial AFE board) | Full prototype possible |
| Existing validation infrastructure | BMS testing track | VCU mockup, VCU firmware |
| CPU/IP licensing | Foundry analog IP libraries | ARM Cortex-M licensed, or open RISC-V (Shakti, VEGA, Ibex, CV32E40P, NEORV32) |
| Mentorship requirement | Senior analog designer (high) | Senior digital designer (moderate) |
| First-silicon risk | Higher (analog craft is unforgiving) | Lower (digital is silicon-predictable) |
| Production-test complexity | High (analog parametric testing) | Moderate (digital functional + small analog) |

### Market and Application Profile

| Dimension | BMS AFE | Generic 32-bit MCU |
| --- | --- | --- |
| Market breadth | BMS makers only | Every EV ECU (VCU, BMS host, motor controller, charging, dashboard, zonal) |
| Volume per vehicle | 1 per pack | 3–30 per vehicle |
| Customer-base maturity | Concentrated (50+ Indian BMS makers) | Broad (every Indian EV maker plus indirect via Tier-1 ECU suppliers) |
| Differentiation thesis | Cost-and-feature optimisation for Indian 2W/3W cell counts | Indigenous automotive MCU; volume play; supply-chain resilience |
| Replacement vs new socket | New socket; BMS designs adapt | Drop-in replacement for STM32F4 / GD32 / AT32 socket if pinout matched |
| Indigenous fabrication path | Partner foundry required for BCD | Full SCL path possible for 180nm variant |

### Educational Phase Coverage

| Phase | BMS AFE Activities | Generic 32-bit MCU Activities |
| --- | --- | --- |
| **Phase 1 — Foundation** | Digital logic, Verilog, FPGA basics, C programming, computer architecture | Same |
| **Phase 2 — Specialisation** | Analog circuit design, SPICE simulation, data converters, mixed-signal layout, BCD process familiarity | CPU architecture (RISC-V / Cortex-M), peripheral IP design, bus protocols (AHB/APB/AXI/TileLink), verification methodologies |
| **Phase 3 — Integration** | FPGA + commercial AFE evaluation board; custom analog blocks in SPICE; MPW test silicon for analog blocks | Full SoC on FPGA; firmware bring-up; validation against the mockup rig; physical design intro on a test block |
| **Phase 4 — Tapeout** | Integrated analog + digital BCD tapeout; AEC-Q100 qualification | Full-chip tapeout; MPW for early silicon validation; AEC-Q100 grade 2 qualification |

---

## Process Considerations in Detail

### Pure CMOS vs BCD vs HV-CMOS

A microcontroller is fundamentally a digital chip with some analog peripherals (ADCs, DACs, PLLs, oscillators). These analog blocks are usually low-voltage (3.3V or lower) and can be designed in standard CMOS with foundry-supplied IP libraries. Pure CMOS at 65–180nm covers the entire MCU design.

A BMS AFE, by contrast, must sense voltages spanning tens of volts across the cell stack (each cell tap sits at a different common-mode voltage), drive balancing currents at those voltages, and survive automotive transients. This needs:

- **High-voltage transistors** — 20V, 40V, 80V, or higher devices for the cell-tap front-end and the balancing drivers.
- **Bipolar devices** — for the precision analog (bandgap, low-offset amplifiers) where pure-CMOS performance is insufficient.
- **DMOS power transistors** — for the balancing current drivers.

A BCD process integrates all three (Bipolar + CMOS + DMOS) on one die. Without BCD, the AFE would need multiple chips, which defeats the integration thesis.

### Mature nodes (high-nm)

The chip industry's frontier is at 3nm and below. The maturity zone — well-understood, broadly available, multi-foundry sourceable — sits at 180nm down to 28nm. Both candidate ASICs target this maturity zone:

- **180nm**: still in volume production globally, supported at virtually every foundry. SCL Mohali fabricates here. Design rules are forgiving; MPW programmes are abundant.
- **130nm and 90nm**: the typical sweet spot for cost-sensitive automotive parts. Tower, HHGrace, X-Fab, UMC, SMIC all offer these nodes with both pure-CMOS and BCD variants.
- **65nm and 55nm**: where modern entry-automotive MCUs sit. NXP S32K1, STM32F4 automotive variants, GD32 — all 55–65nm pure CMOS.
- **40nm**: where current-generation automotive MCUs sit. AURIX TC3xx variants, S32K3, recent automotive variants, modern STM32 automotive.

The recommendation is to pick the *coarsest node that meets the performance target*, not the finest. Mature nodes are more predictable, easier to debug, and have larger MPW programmes.

### SCL Mohali specifically

SCL is India's only operational fab. Its 180nm CMOS process is suitable for:

- **A simple zonal-controller variant of the MCU** — Cortex-M0+ class or a basic RISC-V core, ~50 MHz, with the smaller peripheral set a zonal controller needs.
- **The digital control portion of the BMS AFE** — but not the analog/HV portion, which needs BCD elsewhere.

SCL is not suitable for:

- A high-performance VCU MCU (180nm is too coarse for >100 MHz).
- The full BMS AFE (no BCD).
- Any chip targeting modern automotive performance.

The pragmatic position is: **use SCL where it fits, partner foundries where it doesn't**, and document the fabrication path honestly.

---

## Educational Pathway for Students

A student team can deliver either ASIC through a phased programme structured as follows. The pathway is designed so the FPGA prototype is silicon-validatable before tapeout is attempted, with each phase landing a coherent capability.

### Phase 1 — Foundation Skills

For students of either track:

- **Digital logic and combinational/sequential design** — boolean algebra, FSMs, datapath/control separation.
- **Verilog or SystemVerilog** — RTL coding fundamentals, simulation discipline, testbench writing.
- **Computer architecture basics** — instruction sets, pipelines, memory hierarchy. Patterson & Hennessy as the standard reference.
- **FPGA fundamentals** — Xilinx Vivado or Intel Quartus, simple peripherals on a development board.
- **C programming** — for writing firmware that will run on the eventual silicon.

This is what most ECE undergrads already do. The platform builds on top.

### Phase 2 — Specialisation

**MCU track:**

- **CPU architecture in depth** — pipeline design, hazards, branch prediction (light), memory protection.
- **RISC-V specifically** — the ISA, the privilege model, the open-source cores (Ibex, CV32E40P, NEORV32 to start; eventually Shakti or VEGA).
- **Peripheral IP design** — design a CAN controller, a SPI master, a UART, an I²C, an ADC sequencer. Each one as a verified IP block.
- **Bus protocols** — AMBA AHB, APB, AXI. Or TileLink if going RISC-V native.
- **Verification basics** — assertion-based verification, coverage, constrained-random stimulus.

**AFE track:**

- **Analog circuit design** — operational amplifiers, comparators, bandgap references, current mirrors, biasing.
- **Data converters** — sigma-delta ADCs, SAR ADCs (theory and design).
- **SPICE simulation** — DC, AC, transient, noise, Monte Carlo, corner analysis.
- **Layout for analog** — matching, common-centroid, parasitic awareness, well taps, guard rings.
- **Mixed-signal verification** — co-simulating analog and digital, behavioural modelling of analog blocks in Verilog-A or Verilog-AMS.

### Phase 3 — Integration and FPGA Prototyping

**MCU track:**

- **Full SoC integration** — CPU + memory + bus fabric + all peripherals on a single FPGA design.
- **Firmware bring-up** — port FreeRTOS or the team's actual VCU firmware onto the FPGA-hosted MCU.
- **Validation against the mockup** — connect the FPGA SoC to the vehicle mockup rig and run the same test scenarios the VCU design track runs against the STM32 bring-up board. The FPGA SoC becomes a working VCU.
- **Physical design intro** — synthesise the design with OpenLane or commercial tools, do an initial place-and-route on a test block, learn the workflow.

**AFE track:**

- **Build the FPGA + commercial AFE evaluation board** described earlier — FPGA hosts the digital control architecture; commercial TI/ADI AFE provides the analog front-end.
- **Validate the digital architecture** against real cells, real currents, and the BMS testing track's characterisation infrastructure.
- **Design the custom analog blocks** in SPICE — bandgap, ADC, balancing driver, level shifters. Validate via simulation and (where MPW is accessible) via test silicon.
- **Tapeout-ready analog IP** prepared for integration with the digital architecture in the next phase.

### Phase 4 — Tapeout-Readiness and MPW Participation

- **Multi-Project Wafer (MPW) tapeout** — both tracks should participate in an MPW programme to get real silicon early. The landscape for affordable academic and open-source fabrication has shifted recently; the current options are:

  - **C-DAC chip programmes** — direct foundry partnerships often included for participating institutions.

  - **SCL Mohali academic programme** — for the fully-indigenous 180nm pathway.

  - **Tiny Tapeout** — originally on SkyWater 130nm via Efabless, now transitioning to **IHP SG13G2 130nm** (German fab with an open-source PDK, reportedly holding the record for fastest transistors) and exploring GlobalFoundries GF180MCU routes. The most accessible "first silicon" path for students.

  - **IHP SG13G2 130nm** directly — IHP open-sourced their 130nm BiCMOS PDK in 2023. The PDK is freely available; fabrication access is via Tiny Tapeout, university consortiums, or direct engagement.

  - **GlobalFoundries GF180MCU** — the PDK remains open-source (originally released by Google in 2022). Shuttle access is via IEEE TS-OSE Chipathon, university consortiums, or paid commercial shuttles.

  - **SkyWater SKY130** — the PDK remains open-source. Fabrication access changed when Efabless shut down in March 2025; new shuttle paths are being established by the FOSSi Foundation community and other ecosystem participants.

  - **University-consortium MPWs** — EUROPRACTICE (Europe), CMC Microsystems (Canada, successor to MOSIS), MUSE Semiconductor (US academic access). Academically priced; broader process-node availability including 65nm and 40nm.

- **Full-mask tapeout** — for the eventual production chip, the team needs commercial foundry access (Tower, HHGrace, UMC, SMIC, GlobalFoundries, X-Fab), AEC-Q100 qualification planning, ASIL-B assessment, and a production-test strategy. This is typically where the academic effort transitions to a startup or industry partnership.

**A note on the current landscape:** the era of Google-sponsored free shuttles (Google + SkyWater 2020–2023; Google + GlobalFoundries 2022–2023) made early-stage open-source silicon dramatically more accessible. That sponsorship has wound down, and Efabless — the operational platform behind both programmes — shut down in March 2025. The underlying technology stack (open PDKs, OpenLane, Verilator, KLayout, RISC-V cores, Caravel harness) is more mature than ever, but the path from design to fabrication now requires more deliberate planning. IHP, the FOSSi Foundation community, and university consortiums are the practical pathways today.

### Tools — Open-Source and Commercial

**Open-source toolchain (fully viable for the student pathway):**

- **Simulation**: Icarus Verilog, Verilator, GHDL.
- **Synthesis**: Yosys.
- **Place-and-route**: OpenROAD (digital), Magic VLSI (digital + simple analog).
- **Full RTL-to-GDSII**: OpenLane (SKY130, GF180MCU, IHP SG13G2 PDKs).
- **Analog schematic and SPICE**: Xschem, ngspice, KLayout.
- **FPGA**: Xilinx Vivado WebPACK (free for smaller devices), Intel Quartus Prime Lite, fully open-source toolchains for Lattice ICE40/ECP5 (Yosys + nextpnr).

**Commercial toolchain (for production-grade tapeout):**

- **Cadence**: Virtuoso, Innovus, Spectre. Industry-standard for analog and full-flow digital.
- **Synopsys**: IC Compiler II, VCS, PrimeTime, Custom Compiler. Industry-standard for digital and verification.
- **Mentor (Siemens EDA)**: Calibre (DRC/LVS), Questa (simulation).

University programmes (Cadence Academic Network, Synopsys University Program, Siemens EDA Academic Programme) provide access at substantial discount or free for educational use.

---

## Risk Comparison

**BMS AFE risks:**

- Analog craft is unforgiving. First silicon often needs a second mask iteration to fix subtle issues (offset, noise, matching, temperature drift). Plan for it.
- BCD process choice locks the team into a specific foundry. Switching foundries means redesigning analog blocks.
- AEC-Q100 qualification for an analog/mixed-signal automotive part is non-trivial and requires dedicated qualification work.
- Customer trust in a new AFE is slow to build. BMS makers will pilot the chip on low-volume products before committing to mainstream platforms.

**MCU risks:**

- Toolchain and ecosystem matter as much as the silicon. A chip without a usable compiler, debugger, JTAG support, ARM CMSIS or RISC-V SDK, and an active developer community is unsold. Ecosystem work has to happen alongside silicon work.
- Differentiation from STM32 / GD32 / AT32 must be real. "Made in India" is necessary but not sufficient. The chip must be at least as good as the imported alternative on the dimensions that matter to the customer (cost, availability, performance, peripheral fit, qualification).
- ASIL-B certification is reachable but requires dedicated effort separate from the design work.
- If RISC-V is chosen, the toolchain story is more fragile than ARM's. The team takes on more ecosystem responsibility.

---

## Recommended Sequence

Both ASICs are credible. The question is sequencing, not selection.

**For a student-led effort with strong digital and firmware competence (which describes the team's profile based on the VCU design track):**

**Start with the MCU.** Reasoning:

- The MCU is fully FPGA-prototypable. The student team can validate the entire digital design before committing to silicon, dramatically reducing first-silicon risk.
- The validation infrastructure already exists. The mockup track is the test environment; the VCU firmware is the test stimulus. The FPGA prototype slots directly into work already underway.
- SCL Mohali can fabricate a 180nm zonal-controller variant fully indigenously, giving the fabrication-path story its strongest form.
- Skill profile is closer to what most ECE students arrive with. Analog craft can be added later; digital RTL is the more accessible starting point.
- The eventual chip has the broadest market in the EV stack — every ECU is a candidate socket.

**Follow with the AFE, once the team has:**

- One working chip in the field (the MCU) and the silicon-design experience that comes with it.
- Foundry relationships and EDA-tool workflows established.
- Senior analog mentorship recruited.
- BMS testing-track infrastructure mature enough to validate AFE characteristics at scale.

