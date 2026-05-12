# 48V Battery Management System — Firmware Track

## Objective

Develop the control firmware and state-estimation algorithms for a 48V battery management system. The firmware should run initially on eval kits and development boards, then port cleanly to the custom hardware delivered by the hardware track.

## Product Scope

- A modular firmware stack covering AFE driver layer, measurement pipeline, protection logic, balancing control, host communication, diagnostics, and update/security services.
- A set of state-estimation algorithms — State of Charge (SOC), State of Health (SOH), and a path toward State of Power and State of Energy — implemented in a way that is testable in isolation and tunable per chemistry.
- A clear separation between application logic and hardware-specific drivers so the same algorithms can move across controllers as the project matures.
- **Multi-chemistry support** for **NMC** and **LFP**, with all chemistry-specific behaviour (thresholds, OCV tables, balancing policy, model parameters) data-driven rather than code-driven.
- **Runtime cell-count configurability** from 8s to 16s, in the JBD/Orion style — the same firmware binary supports any cell count in that range, configured at commissioning and persisted in NVM.
- **Control of both contactor-based and MOSFET-based** disconnect paths from a single firmware base, with the active variant selected at configuration time.
- **Functional safety** target — **ASIL-B** at the platform level, with safety-critical functions designed to **ASIL-C** where the architecture allows.
- **Security baseline** — **secure boot, signed firmware images, and OTA-capable update** from variant 1; not a retrofit.

## Design Rationale

### Configuration as Data, Not Code

Every parameter that varies across chemistries, cell counts, pack sizes, or end applications is held as **configuration data persisted in NVM**, not compiled into the firmware. Cell count, chemistry profile, OCV tables, protection thresholds, balancing policy, current-sensor scaling, model parameters, communication identifiers, calibration values — all of these are NVM-resident, versioned, and changeable via a defined service interface without reflashing.

This is what makes the same firmware binary deployable across the full envelope the hardware supports (8s–16s, NMC or LFP, contactor or MOSFET) and matches how production-grade configurable BMS products behave in the field.

The architecture this implies:

- A versioned NVM schema with forward-compatible migration on firmware update.
- An integrity-checked NVM (CRC at minimum, double-buffered for the critical regions) so corruption is detected and recoverable rather than catastrophic.
- A configuration service surface (UDS DIDs over CAN, also accessible over UART) for commissioning, service, and field-update workflows.
- Sensible defaults that allow the unit to come up safely if NVM is wiped or corrupt, declaring itself unconfigured rather than misbehaving.

### Robustness in Firmware

The hardware track sets a robustness bar for the electrical inputs. The firmware has a matching obligation on the signal-processing side:

- **Plausibility checks** on every sensed input — a measurement outside the physically possible range is rejected, not used.
- **Glitch rejection and debouncing** on protection trips — a transient on a sense line should not be able to drive a contactor open through the protection state machine. Multiple consecutive valid samples required before a trip latches; instantaneous trips reserved for genuinely catastrophic events (e.g. short-circuit current).
- **Watchdog discipline** — windowed watchdog, with all real-time tasks contributing to the kick, not a single timer.
- **Stuck-at and zero-failure detection** on sense paths — periodic sanity-injection where the architecture allows.
- Hard separation between **safety-critical paths** (protection decisions, contactor/MOSFET control) and **best-effort paths** (telemetry, logging) at the scheduling and memory level, so a non-safety bug cannot starve a safety task.

### Functional Safety Positioning

The platform targets **ASIL-B** overall, with **ASIL-C** on the safety-critical paths (disconnect control, overcurrent detection, isolation monitoring where applicable). The first prototype runs on STM32, which is realistic for ASIL-B with the right diagnostic libraries and a disciplined coding standard. ASIL-C platform-wide is explicitly a roadmap item, gated on migration to an automotive-grade controller.

Direct implications for v1 firmware:

- All code is written to **MISRA C 2012** from day one, with documented deviations. Retrofitting MISRA compliance is an order of magnitude more expensive than writing in-compliance from the start.
- Diagnostic coverage is tracked per safety-critical function (FMEDA-style accounting), even if a formal FMEDA is not produced in v1.
- The application architecture (separation of safety paths, decoupled scheduling, configuration-as-data) is designed so as not to preclude ASIL-C reachability on the controller migration.

### Security Baseline

The platform ships with **secure boot and signed firmware images from variant 1**. The rationale parallels MISRA: bolt-on cybersecurity is expensive and frequently broken, while designed-in cybersecurity is cheap to maintain. Update workflows — whether wired or OTA — all flow through the same signed-image path, so OTA capability is a transport addition on top of an already-secure update mechanism, not a separate security story. Full ISO/SAE 21434 alignment is documented as a roadmap item; the foundations live in v1.

## Functional Targets

### Measurement and Estimation

- Reliable acquisition of cell voltages, temperatures, and pack current at deterministic timing, with the measurement pipeline supporting any cell count from 8s to 16s.
- **SOC estimation** — **Coulomb counting** as a baseline and a **Kalman observer (EKF or UKF)** built on a **2RC Thevenin equivalent circuit** as the production candidate, with a **1RC** variant as the simpler baseline for early bring-up and comparison.
- **SOH tracking** from capacity fade, internal resistance growth, and EIS-derived features where the AFE supports them.
- A path toward **State of Power** and **State of Energy** built on top of the same model.
- Pack current measurement and integration suited to the **250 A continuous / 400 A peak** envelope, in both directions.

### Protection and Control

- **Pack-level protection state machine** — overvoltage, undervoltage, overcurrent (charge and discharge), short-circuit, overtemperature, undertemperature, cell imbalance, isolation fault (where applicable), contactor/MOSFET diagnostic faults — with appropriate latching and recovery behaviour, all thresholds NVM-driven per chemistry.
- **Disconnect control** for both **contactor-based** and **MOSFET-based** topologies, selected at configuration time.
- **Cell balancing** strategy with thresholds, hysteresis, and policy NVM-driven per chemistry.
- **Plausibility checks, glitch rejection, and debouncing** on every protection-relevant input (see Design Rationale).

### Communication and Diagnostics

- **CAN** as the primary host bus, mandatory on every variant:
  - **Application-layer messaging** uses a **custom, versioned message catalogue** managed as a DBC artifact under version control, with a documented data dictionary and clear forward-compatibility rules.
  - **Diagnostic services** implemented as **UDS over ISO-TP** (ISO 14229 / ISO 15765), even if a minimal service subset to start.
- **UART** for debug and console access during development and service.
- **Automotive Ethernet** as an optional path, used where high-bandwidth telematics or diagnostics are required.
- **Event and fault logging** with timestamped post-mortem records, retrievable over UDS.

### Configuration

- **All operational settings persisted in NVM** — cell count, chemistry profile, thresholds, model parameters, calibration, communication IDs, balancing policy, and so on.
- **Versioned NVM schema** with migration on firmware update.
- **Integrity-protected NVM** (CRC, redundancy on critical regions); safe behaviour on first boot or corruption.
- **Cell count configured via UDS/UART command and persisted**; optional auto-detect at first commissioning, but never on every boot.

### Bootloader, Update, and Security

- **Secure bootloader** with **signed firmware image verification** at every boot, anti-rollback enforcement, and a recoverable fallback on failed update.
- **OTA update capability** from variant 1, sharing the signed-image pipeline used by the wired update path.
- **Secure debug** — debug interfaces locked in production builds, accessible only via authenticated unlock.
- **Key-management story documented** — where signing keys live, how device-unique keys are provisioned, how rotation works.

## Technical Considerations to Own

- **Coding standard and toolchain** — **MISRA C 2012 from day one**, with documented deviations. Static-analysis enforcement in CI (Cppcheck, PC-lint Plus, Polyspace, or equivalent). The team chooses the specific toolchain; the standard is fixed.
- **Scheduler choice** — RTOS vs bare-metal vs a thin custom scheduler, evaluated against:
  - Determinism on the safety-critical paths.
  - ASIL-readiness of the RTOS itself if RTOS is chosen (SafeRTOS, FreeRTOS with safety extensions, ThreadX/Azure RTOS) — or a bare-metal scheduler the team can build a safety case for.
  - Memory and CPU footprint against the chosen controller.
- **Driver abstraction strategy** so the AFE and MCU can be swapped with bounded rework — the swap should be a port of the driver layer, not a rewrite of the application.
- **Cell model order and tuning methodology** — 2RC Thevenin is the production target; the team owns parameter identification (in coordination with the testing track), observability analysis, temperature dependence, and how the model adapts as cells age.
- **NVM layout and lifecycle** — schema definition, versioning, migration, integrity, redundancy, and the question of where the master copy of "what the device should be configured to" lives in the development workflow.
- **Bootloader and secure-update architecture** — image format, signature scheme, key hierarchy, rollback protection, OTA transport, recovery on failed update, factory-provisioning flow.
- **Diagnostic coverage accounting** — per safety-critical function, what diagnostic mechanism covers it and at what claimed coverage. FMEDA-style discipline even if no formal FMEDA is published in v1.
- **Functional-safety library** on the chosen controller — e.g. STMicro's X-CUBE-CLASSB / X-CUBE-STL, or equivalent on the eventual automotive controller. Use of these vs hand-rolled equivalents is a deliberate decision.
- **DBC management workflow** — where the source-of-truth DBC lives, how it's versioned, how firmware and integration tools stay in lockstep.
- **UDS service catalogue** — which services and DIDs are supported, security access levels, and how routine diagnostics are partitioned from manufacturing/factory-only services.
- **Test strategy** — unit tests, software-in-the-loop, hardware-in-the-loop, fault injection, and how the testing track plugs into all three.
- **Memory and CPU budget** against the chosen controller, with headroom for the eventual ASIL-C uplift.

## Starting Points

- **STM32** family (G4, F7, or H7 class depending on workload) as the initial controller, paired with the vendor's functional-safety libraries (X-CUBE-CLASSB / X-CUBE-STL) for ASIL-B-style self-test and diagnostic coverage.
- **Automotive-grade migration target** — NXP S32K, Infineon AURIX TC2xx/TC3xx, TI TMS570/Hercules, or STM SPC5/SR5 — chosen once the application is stable and the peripheral mix is well understood.
- **AFE eval kits** from the chosen vendor as the early bring-up platform, with the driver layer designed from the start to abstract eval-kit specifics.
- **1RC Thevenin model** for initial bring-up and algorithm validation against published reference data; **2RC Thevenin** as the production target once the testing track delivers measured parameters.
- **Open-source UDS / ISO-TP stacks** as a reference (e.g. iso-tp, open-uds), with the team deciding whether to build forward from those or write their own.
- **Established DBC tooling** (Vector CANdb++, Kvaser Database Editor, or open-source equivalents such as cantools) for managing the application-layer message catalogue.
- **Test framework** — Ceedling/Unity or similar for unit tests; the HIL framework is chosen in coordination with the testing track.

## Deliverables

- **Firmware architecture document** with module boundaries, data flow, scheduling model, safety-path separation, and configuration data model.
- A working firmware build on the eval/dev platform exercising all functional targets.
- A firmware build on the custom hardware from the hardware track, supporting both contactor and MOSFET variants and any cell count from 8s to 16s.
- A second firmware build on the production-target controller, demonstrating the migration path.
- **DBC file** for the application-layer CAN catalogue, versioned alongside the firmware.
- **UDS service catalogue** documenting supported services, DIDs, routines, and security-access levels.
- **Bootloader and secure-update specification** — image format, signature scheme, OTA transport, recovery behaviour, key-management notes.
- **NVM schema document** with versioning and migration policy.
- **Algorithm validation reports** for SOC and SOH against measured data from the testing track.
- **MISRA C compliance report** with the deviation log.
- **Diagnostic coverage worksheet** per safety-critical function.
- **Security architecture document** covering boot, update, debug lockout, and key management.
- **Test harness** (unit + SIL + HIL) handed over with the source, including fault-injection cases.

## Out of Scope

- PCB design and component selection (hardware track).
- Cell-level characterisation and model parameter extraction (testing track).
- Mechanical interfaces and enclosure design (mechanical track).

## Inter-Track Dependencies

- **Hardware track** — provides the target platform; firmware team should be involved in pinout, peripheral selection, and diagnostic-coverage features early. Co-owns the bring-up checklist and the protection-path verification on each prototype.
- **Testing track** — provides validated cell models and measurement data for algorithm tuning and verification; consumes firmware builds for SIL/HIL and pack-level tests; co-owns the UDS test suite and fault-injection campaigns.
- **Mechanical track** — informs firmware about sensor placement, thermal limits, and pack-level constraints used in protection thresholds.
