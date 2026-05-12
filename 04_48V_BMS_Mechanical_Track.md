# 48V Battery Management System — Mechanical Track

## Objective

Design, build, integrate, and document **two reference battery packs** — one for an electric two-wheeler, one for a micro electric vehicle — covering cell holding, structural integrity, thermal management, environmental sealing, vehicle integration, and manufacturing readiness. The headline deliverable is not the two packs alone but the **engineering process that produced them**: the reference-vehicle analyses, the design rationale documents, the simulation work, the validation evidence, and the manufacturing-ready documentation. The packs are the artifacts; the process is what makes them licensable and what makes the track repeatable for future products.

## Product Scope

- **Two reference battery packs**, each targeting a different Indian-market vehicle class:
  - **Two-wheeler pack** — approximately **3 kWh** energy capacity as a starting envelope, final capacity derived from the chosen reference vehicle's range, mass, and duty cycle.
  - **Micro EV pack** — approximately **20 kWh** energy capacity as a starting envelope, again finalised from the chosen reference vehicle.
- **LFP cells only**, with the specific cell selected by the team based on availability, vehicle-imposed form-factor constraints, and feasibility findings handed back by the testing track.
- Each pack is **modular at the sub-assembly level** — cell holders, module-level interconnects, sensor mounting, thermal interface — without any requirement that the two packs share module geometry. They can be different products from a common engineering process.
- **No cell-level field serviceability**; **module-level and electronics-level serviceability are required**. Cells stay welded once assembled; modules and the BMS electronics are removable, replaceable, and re-provisioned in the field.
- **Both packs physically integrated onto representative vehicles** for shakedown — not just delivered as boxed assemblies with a documented interface.
- **Manufacturing-ready** output — DFM/DFA review, tooling implications, vendor specifications, and assembly sequence documented to the level a downstream manufacturer can pick up and quote against.
- **Design Rationale Documents (DRDs)** as a first-class artifact — every major decision (target vehicle choice, cell choice, module strategy, cooling strategy, mounting strategy, fusing strategy, sealing strategy) carries a DRD covering alternatives considered, evaluation criteria, decision, and justification.

## Design Rationale

### Process as Deliverable

The two packs are the visible outputs of this track, but the **engineering process** that produced them is the deliverable that travels. A downstream engineer should be able to read the DRDs, the reference-vehicle analyses, the simulation reports, and the validation evidence and re-derive the design decisions independently. That standard is what makes the work licensable — and it's what distinguishes a "we built a pack that works" outcome from a product with an engineering pedigree behind it.

Concretely:

- Every major decision has a **Design Rationale Document** — alternatives, criteria, decision, justification, and the open questions the decision *doesn't* answer.
- Every claim about the pack (mass, range, peak current, cycle life expectation, IP rating, thermal envelope) traces back to a reference, a calculation, or a test — never to a guess.
- The reference-vehicle analysis is **full-envelope** — dimensions, mass budget, mounting geometry, motor/controller spec, expected duty cycle, gradeability, range targets — derived from publicly available data and observable behaviour, with any non-public assumption captured in its own DRD.

### Why LFP

LFP is the single chemistry for both packs. The rationale propagates from the firmware/hardware tracks (chemistry parameterisation in firmware, AFE configuration in hardware), but at the pack level the LFP-only choice carries specific mechanical implications: lower volumetric energy density than NMC (which the design has to absorb), much better thermal stability (which simplifies but does not eliminate propagation containment), and a flatter discharge curve (which does not affect mechanical design but does interact with SOC-driven thermal effects).

### Two Distinct Packs, Common Engineering Process

The 2W and micro EV packs serve materially different products and need not share module geometry. The 2W pack is volume-, mass-, and cost-constrained in ways the micro EV pack is not; the micro EV pack faces crash and propagation requirements the 2W pack faces in a different form. **What the two packs share is the engineering process, not the design.** Each pack independently goes through reference-vehicle analysis, DRD-driven design, simulation, prototype, integration, validation, and manufacturing-readiness. The two efforts produce two products from one method.

### Integration on Real Vehicles

Both packs are physically integrated onto representative vehicles for shakedown. "Documented interface specification" is not enough — interface documents miss what real integration exposes: harness routing under vibration, thermal interaction with vehicle structure, service access in the assembled state, the difference between a CAD clearance and a real one. Without integration the pack is a study; with integration it is a product.

### Thermal Runaway Propagation

LFP is materially safer than NMC but not exempt — single-cell thermal events still occur, and propagation containment is becoming a regulatory expectation (AIS-156 Phase 2 in India, GB 38031 in China, ECE R100 Phase 2 in Europe). Both packs include **module-level propagation barriers, defined vent paths, and thermal isolation between modules**, with the design choices justified by analysis. Single-cell thermal runaway should not become a pack-level event.

### Manufacturing Readiness

A prototype that works is not the goal — the goal is a design that a downstream manufacturer can quote, tool, and build at volume. This means **DFM/DFA notes alongside the CAD**, vendor specifications for non-trivial parts, assembly sequence with fixturing implications, tolerance stack-ups closed, and a service manual that a field technician can actually follow. This is the difference between a project and a product, and it aligns the track with the licensing intent of the broader effort.

## Functional Targets

### Reference Vehicle Analysis (Per Pack)

- Identification of a target Indian-market reference vehicle for each pack — examples to consider include the Ather 450X, Ola S1 Pro, Bajaj Chetak, and TVS iQube for the 2W; the MG Comet, Bajaj Qute, and Tata Tiago EV envelope (scaled) for the micro EV — with the final pick justified in a DRD.
- **Full-envelope analysis** for each reference vehicle — pack dimensional envelope, mass budget, mounting geometry, motor/controller power and current, expected duty cycle, gradeability, range targets, ambient operating envelope (Indian-market conditions).
- Energy capacity, peak and continuous discharge current, and cooling duty derived from this analysis, not assumed up front.
- Volumetric and gravimetric energy density targets benchmarked against the chosen reference, with any deliberate shortfall (e.g. for safety margin or service-friendliness) justified.

### Cell and Module

- **LFP** cell choice justified against availability, vehicle-imposed form-factor constraints, and the testing track's feedback on cell feasibility.
- Mechanical retention of cells under expected vibration, shock, and thermal-expansion conditions.
- Module-level construction with welded cell interconnects (no cell-level serviceability) and serviceable module-to-module interfaces.
- **Cell-level fusing** as an explicit design exploration — Tesla-style bond-wire fuses on each cell's interconnect are real engineering with real cost and complexity trade-offs. The team explores the option and produces a DRD justifying inclusion or exclusion per pack.

### Structural and Vibration

- Structural rigidity and torsional stiffness validated against representative vehicle load cases.
- **Modal and harmonic FEA** against an automotive random-vibration PSD appropriate to each vehicle class, with documented natural frequencies vs vehicle excitation spectra.
- **Crash analysis** — simplified static-equivalent FEA for the worst-case impact direction per the target vehicle's category, with mounting integrity as the primary failure criterion. Inputs and boundary conditions documented to the level a future formal transient crash analysis can be built from.
- **Pre-compliance shake-table testing** on a built prototype, in coordination with the testing track.

### Thermal Management

- Cooling strategy (active liquid, forced air, passive, or some combination) **derived from the chosen reference vehicle's duty cycle**, not pre-decided. The team produces a DRD weighing the options for each pack independently.
- Thermal performance keeping the pack within the operating window declared by the firmware/testing tracks, across the Indian-market ambient envelope.
- CFD / thermal simulation, validated against pack-level measurements from the testing track.
- Thermal interface materials and conductive paths sized for the worst-case duty derived from reference-vehicle analysis.

### Sealing and Environmental Protection

- **IP67 / IP68** as the target rating (IP67 minimum, IP68 where the application demands), justified per pack in a DRD.
- Design analysis — gasket compression, joint geometry, vent strategy, condensation behaviour, dust ingress paths.
- Validation on built prototypes via immersion and spray testing, in coordination with the testing track.

### Thermal Runaway and Safety

- Module-level **propagation barriers, defined vent paths, and thermal isolation** between modules.
- Pack-level vent geometry that handles the worst-case single-cell event without breach of the primary enclosure in a hazardous direction.
- Design choices justified by analysis with reference to AIS-156 Phase 2, ECE R100 Phase 2, and equivalent propagation-containment expectations.
- Fusing geometry, isolation, and high-current path layout reviewed against fault-condition failure modes.

### Vehicle Integration

- **Physical integration of both packs onto representative vehicles** — not just an interface specification.
- Harness routing, connector access, service access, thermal interaction with vehicle structure, and acoustic behaviour all evaluated on the integrated assembly.
- Mounting hardware, isolation strategy, and torque specifications documented.

### Manufacturing Readiness

- Tolerance stack-up analysis closed for all critical interfaces.
- **DFM/DFA review** with documented findings and the design changes that follow from them.
- Assembly sequence with fixturing requirements and operator-error proofing.
- Vendor specifications for non-trivial parts (cells, busbars, structural elements, cooling components, fasteners, seals).
- **Service manual** suited to a field technician — module replacement, electronics replacement, sealing restoration, post-service validation.

## Technical Considerations to Own

- **DRD framework and discipline** — how DRDs are templated, where they live, how they're reviewed, and how they stay in sync with the design as it evolves. The DRD trail is the licensable bridge from problem to design.
- **Reference vehicle analysis methodology** — what data is gathered from public sources, what is reverse-engineered from observation, and how non-public assumptions are documented and bounded.
- **Cell selection workflow** — short-list candidate LFP cells against availability, vehicle form-factor constraints, and the testing track's feasibility input. **Cell qualification is shared with the testing track**: this track owns the requirement and the short-list; the testing track owns the characterisation campaign that confirms or rejects each candidate. The handoff must be explicit, not assumed.
- **Module strategy** — cell count per module, series/parallel topology, interconnect method (laser welding vs ultrasonic vs alternatives), and module-level fusing decision.
- **Materials selection** — frame, enclosure, busbars, thermal interface, propagation barriers, gaskets — balanced across weight, cost, manufacturability, thermal/electrical performance, and end-of-life recyclability.
- **Cooling strategy decision per pack** — informed by the reference vehicle's duty cycle, ambient envelope, and packaging constraints, documented in a DRD.
- **Vibration isolation strategy** — mounting points, damping elements, and how the pack interfaces to the host vehicle's chassis.
- **Sealing strategy** — gasket selection, joint design, vent membrane choice, condensation management, service-time re-sealing approach.
- **Propagation containment** — barrier materials, vent geometry, inter-module spacing, and how failure is contained without becoming a different kind of failure.
- **Manufacturability and DFM/DFA discipline** — tolerances, assembly sequence, fixturing, in-process inspection, and tooling implications surfaced early enough to influence the design rather than annotate it after the fact.
- **Integration coordination** with the host vehicle for each pack — mounting interface, harness routing, service access, and the practical realities the CAD doesn't show.
- **Standards mapping** — UN 38.3, IEC 62619, AIS-156, ECE R100, ECE R136 (2W-specific) mapped clause-by-clause to design features, design analyses, and validation tests. Same posture as the BMS: design to the bar, generate the evidence in-house where possible, send the formal stamp out.

## Required Simulation and Analysis

- Structural FEA covering static, modal, and shock load cases for each pack.
- **Modal and harmonic vibration FEA** against an automotive random-vibration PSD appropriate to each vehicle class.
- **Crash analysis** — simplified static-equivalent FEA for the worst-case impact direction per the target vehicle category, with mounting integrity as the primary failure criterion.
- **CFD / thermal simulation** for the chosen cooling strategy of each pack, validated against measurements from the testing track.
- **Thermal runaway and propagation analysis** — single-cell event, propagation behaviour, vent path, and the case for the chosen barrier strategy.
- **Sealing analysis** — gasket compression, joint geometry, and the path to claimed IP rating.
- Tolerance stack-up analysis for all critical interfaces (cell-to-busbar, sensor-to-cell, pack-to-vehicle, module-to-module).

Every simulation report carries stated assumptions, mesh quality, convergence evidence, and a documented link to the validation that confirms (or qualifies) the result.

## Starting Points

- **Candidate reference vehicles** (the team picks one per pack and justifies in a DRD):
  - **2W**: Ather 450X, Ola S1 Pro, Bajaj Chetak, TVS iQube.
  - **Micro EV**: Vayve Mobility EVA, Vayve Mobility CT4.
- Established standards as design boundary conditions — UN 38.3 (transport), IEC 62619 (industrial), AIS-156 (India), ECE R100 (Europe), ECE R136 (2W-specific). Recent propagation requirements — AIS-156 Phase 2, ECE R100 Phase 2, GB 38031 — as the propagation-containment bar.
- Existing pack designs in the public domain reviewed as references, not templates to copy.
- The testing track's vibration, thermal, and immersion capabilities should drive what is realistic to validate in-house vs send to an external lab — the same posture used for the BMS.
- A **DRD template** agreed early and reused for every major decision, so the discipline is consistent across the team and across the two pack designs.

## Deliverables

- **Reference Vehicle Analysis** document for each pack — dimensional envelope, mass budget, mounting geometry, motor/controller spec, duty cycle, gradeability, range, ambient envelope, and the derived pack-level requirements.
- **Design Rationale Documents (DRDs)** for every major decision per pack — alternatives, criteria, decision, justification, open questions.
- **Pack architecture document** for each pack — sizing, mass budget, thermal budget, module layout, electrical layout, mechanical layout.
- **CAD models and manufacturing drawings** for each pack and its sub-assemblies, to a level a downstream manufacturer can quote against.
- **Simulation reports** covering structural, modal, vibration, crash (static-equivalent), thermal, sealing, and propagation analyses — each with stated assumptions, mesh quality, convergence evidence, and validation linkage.
- **Two built and validated prototypes**, one per pack class.
- **Both packs integrated onto representative vehicles**, with integration documentation covering harness, mounting, service access, and the practical findings the CAD didn't predict.
- **DFM/DFA review notes** and the design changes that resulted from them.
- **Assembly sequence document** with fixturing requirements and operator-error proofing.
- **Service manual** for module replacement, electronics replacement, sealing restoration, and post-service validation.
- **Vendor and material specifications** for non-trivial parts (cells, busbars, structural elements, cooling components, fasteners, seals, propagation barriers).
- **Standards mapping document** — UN 38.3, IEC 62619, AIS-156, ECE R100, ECE R136 clauses mapped to design features, design analyses, and validation tests, with in-house/outsourced/not-applicable distinguished.

## Out of Scope

- BMS PCB and electrical design (hardware track).
- Cell characterisation campaigns and pack-test execution (testing track) — though this track defines the geometry, sensor placement, and instrumentation requirements that those tests rely on.
- **Full transient crash FEA** — out of scope; static-equivalent crash analysis is in. Transient crash is a discipline of its own and a documented roadmap item.
- **Formal certification testing in accredited labs** — design plans and in-house pre-compliance evidence are in scope; accredited execution is outsourced.

## Inter-Track Dependencies

- **Hardware track** — coordinates connector locations, sensor placement, busbar geometry, BMS-board mounting, and how the contactor or MOSFET disconnect path lives inside the pack.
- **Firmware track** — provides the thermal envelope and current limits that drive cooling design; consumes the pack-level sensor placement to set protection thresholds.
- **Testing track** — provides cell-feasibility findings that constrain cell selection; provides validation data for thermal, vibration, sealing, and crash analyses; co-owns the pre-compliance test execution. Cell qualification is an explicit shared workflow: this track owns the requirement and short-list, the testing track owns the characterisation that confirms or rejects each candidate.
- **Vehicle integration partners** — provide the host vehicles for both pack integrations and the mounting/harness interfaces those integrations require.
