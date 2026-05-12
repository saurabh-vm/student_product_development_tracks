# 48V Battery Management System — Testing Track

## Objective

Build the test infrastructure, methodologies, and tooling required to validate cells, the BMS electronics, and full battery packs — and to derive *defensible product claims* from that validation. The headline output is not the equipment (which is procured) but the **test design, automation, and analysis** that determines what we can credibly say about the product: what it does, where it breaks, and how it ages. Accuracy, repeatability, traceability, and documented assumptions are first-class concerns.

## Product Scope

- A laboratory and test-automation infrastructure for **cell-level** characterisation, **BMS board** verification, **pack-level** validation, and **end-of-line (EOL) production** testing.
- The lab itself (cyclers, electronic loads, climate chambers, EIS instruments, ESD/EFT/surge generators, near-field probes, vibration shaker, lab-grade reference instruments) is procured — the value the track delivers is not the equipment but the **integration, automation, and methodology** built on top of it.
- A documented methodology for producing accurate cell models (2RC Thevenin, with 1RC as a baseline) suitable for direct ingestion by the firmware track's state estimator.
- A custom **Hardware-in-the-Loop (HIL) rig** for the BMS, optimised for fault injection and the breadth of in-house testing that justifies sending only the formal/compliance work outside.
- A **defensible body of evidence** for every product claim — performance, accuracy, robustness, safety, life expectancy. If we can't measure it, we don't claim it.

## Design Rationale

### Methodology Over Equipment

The lab is procured. What students build is the *operating system* on top of it: the integration that lets a single Python pytest invocation drive a cycler, a chamber, a HIL rig, and a handful of reference instruments to the completion of a test plan, with results landing in queryable storage. The skill being developed — and the deliverable that makes this licensable — is the ability to **design a test that produces evidence, automate it so it produces evidence reliably, and analyse the evidence so it actually answers the engineering question being asked**. Procuring the equipment is the easy part.

### Claims and Evidence

Every product claim the BMS or pack makes — current rating, voltage envelope, cycle life, SOC accuracy, SOH degradation, fault response time, isolation, thermal limits — needs to map to a documented test, a documented procedure, a documented dataset, and a documented assumption set. "We tested it and it worked" is not the deliverable; the deliverable is a body of evidence that survives an outside engineer reading it three years later and asking *how do you know?*

This shapes how the track is run:

- Every test plan has a stated **claim it supports** and a **decision rule** for pass/fail before the test is executed.
- Raw data is preserved alongside the analysis, in a format another team could re-analyse from.
- Assumptions and instrument accuracy budgets are written down and propagated into the final stated claim.
- An accelerated test (calendar life via Arrhenius, cycle life via DoD/SoC stress) has the model and assumptions behind the acceleration documented; any claim derived from it carries those assumptions explicitly.

### Find Bugs, Don't Confirm Them

Validation that only exercises the happy path discovers nothing. The test design emphasises **error-case injection, boundary conditions, and adversarial inputs** — shorted sense leads, open thermistors, mismatched cells, transients on the supply, malformed CAN frames, mid-update power loss, sensor stuck-at faults. The custom HIL exists primarily to make this kind of testing fast and repeatable. The expectation is that the test suite *finds defects*; a run that uncovers nothing is suspect, not celebrated.

## Functional Targets

### Cell-Level

- Capacity tests across temperatures and C-rates, sized to the envelope of the chosen reference cells.
- Hybrid Pulse Power Characterisation (HPPC) or equivalent for resistance/impedance mapping across SOC and temperature.
- Open-Circuit Voltage curve extraction with appropriate relaxation times, per chemistry.
- EIS measurement campaigns to support SOH feature extraction.
- **Cell model parameter extraction** delivering 2RC Thevenin parameters as the production model and 1RC as the baseline, in the exchange format consumed directly by the firmware track's EKF.
- **Cell model accuracy targets** — terminal voltage prediction error of **≤ 30 mV RMS** across the SOC range under representative dynamic load profiles, and **≤ 10 mV RMS** under steady-state conditions, both at 25 °C reference. Degradation of these figures across the wider temperature envelope is documented, not eliminated.
- **Cycle life — methodology and early data only**. The framework, schedules, environmental control, and analysis pipeline are in scope; complete cycle-life datasets extend beyond any reasonable project window and are not.
- **Accelerated life testing plans** — Arrhenius-based calendar life, DoD/SoC-stress cycle life — with the underlying acceleration model and its assumptions documented as part of any claim derived from them.

### BMS Board-Level

- Measurement-accuracy verification (voltage, current, temperature) against traceable references, across the operating envelope.
- Protection trip-point verification across all defined fault modes, including timing of the trip relative to the trigger.
- **Configurable cell-count verification** — confirming firmware adapts correctly across any cell count from 8s to 16s, including boundary cases and reconfiguration transitions.
- **Communication conformance**:
  - CAN bus conformance (timing, error handling, bus-off recovery).
  - **UDS service catalogue conformance** — every documented DID and service exercised, with negative-response handling verified.
  - Automotive Ethernet conformance where the optional path is populated.
- **Diagnostic coverage validation** for every ASIL-B and ASIL-C function — verifying through fault injection that the claimed diagnostic mechanism actually catches the fault it claims to catch.
- **Secure-boot and signed-firmware end-to-end validation** — unsigned firmware rejected, downgrade attempts blocked, mid-update power loss recovered cleanly, OTA failures fall back safely.
- **Robustness and immunity testing** against ISO 7637-2 (transients), ISO 16750-2 (supply), IEC 61000-4-2 / -4 / -5 (ESD, EFT, surge), and radiated/conducted EMC pre-compliance.
- Balancing-behaviour verification across chemistry and cell-count configurations.
- Sleep/wake transitions and quiescent-current characterisation.

### Pack-Level

- Charge/discharge profiles under representative load shapes, scaled to the **250 A continuous / 400 A peak** envelope.
- Thermal mapping during operation, cross-checked against the mechanical track's simulations.
- Insulation resistance, isolation, and HV safety checks consistent with the application target.
- **End-to-end SOC/SOH accuracy assessment** against reference measurements, across temperature and aging.
- Fault-condition behaviour — pack response to short-circuit, overcurrent events, individual cell failures, and sensor failures.

### End-of-Line (EOL) Production Testing

- A separate, narrower test plan suited to **manufacturing** rather than engineering validation.
- Functional verification — every output drives, every input reads, every comms interface enumerates.
- **Calibration** — current sensor zero/scale, temperature reference, voltage references where the AFE permits.
- **Serialisation and secure provisioning** — device identity, signing keys, configuration baseline, traceability to BOM lot.
- **Cycle time budget** documented so the EOL rig can be sized for manufacturing volume.
- A working EOL test fixture and station as a licensable artifact.

### Test Automation and Data

- All tests authored as **Python / pytest**, with reusable custom instrument drivers.
- Test reports (Allure, pytest-html, or equivalent) generated per run, linked to raw measurement data.
- **Local logging in the immediate term** — a simple, self-contained data store (filesystem layout + SQLite or local TimescaleDB instance) that lets characterisation campaigns run today without waiting on the monitoring stack.
- **Path to integrate with the monitoring track's data infrastructure** documented and roadmapped, so test runs eventually flow into the same substrate as field telemetry.

## Technical Considerations to Own

- **Reference cell supply** — characterisation work needs a small, named list of cells (one NMC, one LFP) from specific suppliers and lots, available in enough volume to sustain the project. **This is a prerequisite that must be resolved before serious characterisation begins** — sourcing, qualification, lot tracking, and storage handling are all open requirements for the team to figure out. Models derived from one-off samples are not reproducible and not licensable.
- **Accuracy budget propagation** — every instrument has an accuracy spec; how those propagate into the final claim (cell model accuracy, SOC accuracy, current sensor accuracy) is the analysis that distinguishes credible from cargo-cult validation.
- **Calibration discipline and traceability** — what's calibrated, against what reference, on what interval, with what records.
- **Custom HIL architecture** — built from commodity CAN/analog/digital I/O (Kvaser, PEAK, USB-DAQ class), sized for fault injection, with a clear contract on what *can* be tested in-house vs what *must* go to an external lab. Maximising the in-house surface is an explicit goal.
- **Data schema for results** — designed so the firmware track can ingest cell models programmatically, and so historical data remains useful as the test plan evolves.
- **Test fixture design** — DUT mounting, contact resistance management, sense-line layout, safety interlocks (especially for any rig handling the full 250 A / 400 A envelope).
- **EIS validation methodology** — how production-AFE EIS measurements are validated against laboratory-grade EIS equipment, and how the two are reconciled.
- **EOL test design** — what's in vs out, cycle time, operator-error proofing, failure-mode logging, and how the EOL station ships as part of the licensable bundle.
- **Accelerated test models** — Arrhenius parameters, DoD/SoC stress models, and how the acceleration factor is validated rather than assumed.
- **Certification mapping** — UN 38.3, IEC 62619, AIS-156, ECE R100 (and any others relevant to the target markets) mapped clause-by-clause to in-house tests, outsourced tests, or "not applicable with reason." The team produces the *plan*; formal certification testing is outsourced.
- **Diagnostic coverage verification methodology** — given a claimed coverage for a safety mechanism, how that coverage is actually measured through fault injection. Co-owned with the firmware track.

## Starting Points

- A modest cycler combined with a temperature chamber is enough to start cell characterisation while the wider rig comes online.
- Existing standards (IEC, USABC, ISO 12405 series, AIS-156, UN 38.3) as references for test procedures — adapted rather than reinvented.
- **Open-source ECM parameter extraction tools** as a baseline before writing custom pipelines.
- **Python + pytest** as the automation harness — same toolchain as the firmware track's HIL framework, which keeps the test surface coherent.
- **SQLite or a local TimescaleDB instance** as the immediate data store, with the long-term path being integration into the monitoring track's stack.
- **Lab-grade EIS (Biologic SP-150-class or equivalent)** as the reference instrument for validating production-AFE EIS measurements.

## Deliverables

- Test rigs (physical), with documented build, calibration, and operating procedures.
- Custom **HIL rig** for the BMS, with documented test interfaces and a catalogue of fault-injection cases it supports.
- Written test plans for cell-level, board-level, pack-level, and EOL — each plan linking explicitly to the product claims it supports and the pass/fail decision rule.
- A **reproducible cell-modelling methodology** — input data format, extraction procedure, output model format, and accuracy report against the stated targets.
- A library of characterised cell models (2RC and 1RC), versioned and tied to specific cell lots.
- **Validation reports** for each release candidate of the BMS firmware and hardware, including diagnostic-coverage verification.
- **Certification test plan documents** — clause-by-clause mapping for UN 38.3, IEC 62619, AIS-156, ECE R100 and other relevant standards, distinguishing in-house, outsourced, and not-applicable items.
- **EOL test station** — fixture, scripts, calibration procedures, secure-provisioning workflow, cycle-time documentation.
- **Test automation framework** — Python/pytest harness, instrument drivers, report generation, local data store.
- **Accelerated test methodology** documents — acceleration models, assumptions, and the procedure for converting accelerated results into product claims.
- **Integration roadmap** from local logging into the monitoring track's data infrastructure.

## Out of Scope

- Designing the BMS hardware or firmware (other tracks).
- Designing the pack enclosure (mechanical track) — though structural test rigs and their instrumentation are within scope.
- **Formal certification testing in accredited labs** — the team produces the plans and the in-house pre-compliance evidence; accredited execution is outsourced.

## Inter-Track Dependencies

- **Firmware track** — primary consumer of cell models and validation results; should agree on the model exchange format early. Co-owns the UDS test suite, the diagnostic-coverage verification methodology, and the fault-injection catalogue. Provides firmware builds for HIL and SIL.
- **Hardware track** — provides DUTs and benefits from early measurement-accuracy feedback. Co-owns the robustness, transient-immunity, and EMC pre-compliance test plans.
- **Mechanical track** — co-owns thermal and vibration testing; provides packs for pack-level testing and validates simulations against measured data.
- **Monitoring track** — eventual integration target for test data; the local data store is designed to be portable into the monitoring stack's substrate when that track is ready.
