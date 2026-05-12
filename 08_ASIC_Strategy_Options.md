# India-Aligned ASIC Options for the EV Platform

## The Two Options

### Option A — Battery Management Analog Front-End (AFE)

A purpose-designed mixed-signal IC handling cell-voltage sensing, current sensing, balancing, fault detection, and host communication for a Li-ion pack. Competes with imported chips from TI (BQ76952, BQ79616), ADI (LTC6813), NXP (MC33775A), Renesas, and STMicroelectronics.

Target market: 8s–16s LFP and NMC packs for the Indian 2W, 3W, and micro-EV segment. Every Indian 2W BMS uses an imported AFE today; the chips are typically overspecified for the application and supply-chain fragile. A focused AFE designed for the actual Indian-market cell counts, cost point, and feature set has 50+ identifiable customers.

**Process**: mixed-signal, BCD (Bipolar-CMOS-DMOS), 180nm or 130nm. Partner foundry required (Tower, HHGrace, X-Fab, Dongbu); SCL Mohali does not offer BCD.

### Option B — Generic 32-bit Automotive Microcontroller

A fully digital MCU in the class of STM32F4/G4 with automotive variants. Targets zonal-controller and 2W-VCU duty across the Indian EV stack. Competes with STMicro automotive STM32, NXP S32K1, Renesas RH850/RL78, GigaDevice GD32 automotive, Artery AT32 — *not* AURIX/S32K3/TMS570 territory, which is the passenger-car ASIL-D class the team would lose competing with.

Target market: every Indian EV ECU — VCU, BMS host, motor controller supervisor, dashboard, charging, zonal controllers. 3–30 chips per vehicle. India has zero indigenous production in this class today.

**Process**: pure CMOS, multiple node options. SCL Mohali at 180nm is viable for a zonal-controller variant; 65–90nm at a partner foundry for a higher-performance VCU-class variant.

---

## Recommendation

**Start with the MCU. Follow with the AFE.**

For a long-term semiconductor capability, both chips, in that sequence. The MCU and AFE are the two highest-value silicon decisions in every Indian EV; together they form the kernel of a real indigenous automotive-semiconductor capability.

---

## Reasoning

### Why MCU first

1. **Fully FPGA-prototypable.** The entire digital design — CPU core, peripherals, bus fabric — can be synthesised onto a Xilinx or Intel FPGA and validated against real firmware running real automotive scenarios before silicon commitment. By tapeout, every line of RTL has been exercised by real workloads on real-time hardware. The risk of a digital functional bug surviving to silicon is small.

2. **Validation infrastructure already exists.** The vehicle mockup track is the test environment; the VCU firmware is the test stimulus. The FPGA prototype slots directly into work already happening — the FPGA SoC becomes a VCU candidate against the mockup rig. The work compounds rather than diverging.

3. **Skill profile matches what students arrive with.** Digital design, Verilog, computer architecture, C programming are standard ECE undergrad ground. Analog craft can be added later. Senior digital mentorship is easier to recruit than senior analog mentorship.

4. **Indigenous fabrication path is intact.** SCL Mohali's 180nm CMOS can fabricate a zonal-controller variant of the MCU fully in India. The AFE's BCD process requirement makes that path inaccessible.

5. **Broadest market reach.** 3–30 MCUs per vehicle versus 1 AFE per pack. Volume justifies the engineering investment, and the chip slots into many existing socket designs.

### Why AFE second

Once the MCU is in field, the team has foundry relationships, EDA-tool workflows, and silicon-design discipline learned on a bounded problem. The BMS testing track will also be mature enough to qualify analog characteristics at scale. With those prerequisites in place, the AFE becomes a focused analog effort built on existing silicon experience, rather than a first-silicon attempt at the harder of the two chip types.

### Why not the reverse sequence

The AFE is a strong opportunity in some ways — concentrated customer base, sharper supply-chain pain, smaller team can plausibly own the project. But first-silicon risk is higher (analog craft is unforgiving), it cannot be FPGA-prototyped in full, and it requires senior analog mentorship that isn't currently part of the project. For a student-led effort, the MCU is the safer and more compounding first project.

If senior analog mentorship is recruited at the start, AFE-first becomes defensible.

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
| Replacement vs new socket | New socket; BMS designs adapt | Drop-in candidate for STM32F4 / GD32 / AT32 socket if pinout matched |
| Indigenous fabrication path | Partner foundry required for BCD | Full SCL path possible for 180nm variant |

### Educational Phase Coverage

| Phase | BMS AFE Activities | Generic 32-bit MCU Activities |
| --- | --- | --- |
| **Phase 1 — Foundation** | Digital logic, Verilog, FPGA basics, C programming, computer architecture | Same |
| **Phase 2 — Specialisation** | Analog circuit design, SPICE simulation, data converters, mixed-signal layout, BCD process familiarity | CPU architecture (RISC-V / Cortex-M), peripheral IP design, bus protocols, verification methodologies |
| **Phase 3 — Integration** | FPGA + commercial AFE evaluation board; custom analog blocks in SPICE; MPW test silicon for analog | Full SoC on FPGA; firmware bring-up; validation against the mockup rig; physical design intro |
| **Phase 4 — Tapeout** | Integrated analog + digital BCD tapeout; AEC-Q100 qualification | Full-chip tapeout; MPW for early silicon validation; AEC-Q100 grade 2 qualification |

---

## Process Notes

A BMS AFE needs BCD — high-voltage transistors for cell-tap front-ends, bipolar devices for precision analog, DMOS for balancing currents. SCL Mohali offers 180nm pure CMOS, not BCD; partner foundries (Tower, HHGrace, X-Fab, Dongbu) are the realistic path.

The MCU is fully digital with embedded analog IP from foundry libraries — pure CMOS at 65–180nm. SCL Mohali's 180nm fits a ~50 MHz zonal-controller variant. A 100–200 MHz VCU-class variant needs a partner foundry at 65–90nm.

Both target mature nodes (180nm down to 40nm). The leading-edge frontier is irrelevant for automotive applications at this scope. Pick the coarsest node that meets the performance target — mature nodes are predictable, debuggable, and well-supported by MPW programmes.

---

## FPGA Prototyping

**MCU — fully prototypable.** Synthesise the CPU core (RISC-V or licensed Cortex-M) plus all peripherals onto a Xilinx Artix-7/Kintex-7/Zynq or Intel Cyclone V FPGA. Run the actual VCU firmware on it; validate every peripheral, every interrupt, every bus interaction; iterate the RTL until silicon-ready.

**AFE — digital portion prototypable, analog not.** The digital control logic (balancing FSM, ADC sequencer, communication protocol, safety supervisor) runs on an FPGA paired with a commercial AFE chip on a daughter-board to provide the real analog front-end during system-level validation. Custom analog is designed in SPICE and validated through MPW test silicon before integration with the FPGA-validated digital architecture for full-mask tapeout.

---

## Educational Pathway

### Phase 1 — Foundation Skills (both tracks)

Digital logic and FSMs, Verilog/SystemVerilog, computer architecture (Patterson & Hennessy level), FPGA basics on Xilinx Vivado or Intel Quartus, C programming.

### Phase 2 — Specialisation

**MCU**: CPU architecture in depth, RISC-V ISA and open-source cores (Ibex, CV32E40P, NEORV32, Shakti, VEGA), peripheral IP design (CAN, SPI, I²C, UART, ADC sequencer), bus protocols (AMBA AHB/APB/AXI or TileLink), verification basics (assertions, coverage, constrained-random).

**AFE**: analog circuit design (op-amps, comparators, bandgap references, current mirrors), data converters (sigma-delta, SAR), SPICE simulation across corners and Monte Carlo, analog layout (matching, common-centroid, guard rings), mixed-signal verification.

### Phase 3 — Integration and FPGA Prototyping

**MCU**: full SoC integrated on FPGA; firmware bring-up using the team's actual VCU firmware; validation against the vehicle mockup rig; physical-design intro using OpenLane or commercial tools.

**AFE**: FPGA + commercial AFE evaluation board for system-level validation against real cells and currents; custom analog blocks designed in SPICE; MPW test silicon for critical analog blocks.

### Phase 4 — Tapeout

MPW participation via Tiny Tapeout (transitioning to IHP SG13G2 130nm), IHP directly, GlobalFoundries GF180MCU via IEEE TS-OSE Chipathon, university consortiums (EUROPRACTICE, CMC Microsystems, MUSE Semiconductor), C-DAC chip programmes, or SCL Mohali's academic programme. Full-mask production tapeout via commercial foundries (Tower, HHGrace, UMC, SMIC, GlobalFoundries, X-Fab) with AEC-Q100 qualification and ASIL-B assessment as parallel workstreams.

### Tools

**Open-source**: Verilator, Icarus Verilog, Yosys, OpenROAD, OpenLane, Magic VLSI, Xschem, ngspice, KLayout. Open PDKs: SkyWater SKY130, GlobalFoundries GF180MCU, IHP SG13G2.

**Commercial**: Cadence Virtuoso/Innovus/Spectre, Synopsys IC Compiler/VCS/PrimeTime, Mentor Calibre/Questa. University programmes (Cadence Academic Network, Synopsys University Program, Siemens EDA Academic Programme) provide academic access.

---

## Risks

**AFE**: Analog craft is unforgiving; first silicon often needs a second mask iteration to correct subtle issues (offset, noise, matching, temperature drift). BCD process choice locks the team to a specific foundry. AEC-Q100 qualification for analog parts is non-trivial. Customer trust in a new AFE builds slowly — BMS makers pilot on low-volume products before committing to mainstream platforms.

**MCU**: Toolchain and ecosystem are as important as silicon — compiler, debugger, JTAG support, SDK, developer community must exist or the chip is unsold. Differentiation from STM32/GD32/AT32 must be real beyond a "made in India" label. ASIL-B certification requires dedicated effort separate from design. Choosing RISC-V over ARM means the team takes on toolchain responsibility.
