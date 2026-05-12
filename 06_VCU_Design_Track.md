# Vehicle Control Unit — VCU Design Track

## Objective

Design and build a **Vehicle Control Unit** — a safety-rated MCU-based controller, its firmware, and casing — that integrates first into the vehicle mockup developed in parallel and, after mockup validation, into the race team's vehicle, Vayve Mobility products, and full passenger-car deployments. The unit must cover both vehicle profiles (2-wheeler and full passenger car) through NVM-driven configuration on a single firmware family. The design is intended to be licensable, so the safety case, security posture, and manufacturing readiness are v1 deliverables rather than roadmap items.

The work is organised into two coordinated tracks that share an architecture and ship a single product — a **hardware track** that owns the PCB, enclosure, and physical interfaces, and a **firmware track** that owns the RTOS, application logic, and software stack. Cross-cutting concerns (functional safety, real-time targets, robustness) are jointly owned. The track structure exists to give each lead a clear scope of ownership; the team is small enough that the two leads coordinate constantly and review architecture decisions together.

## Product Scope

- **Single-board MCU-based control unit** running the deterministic real-time control loops — torque request, brake-blend negotiation, protection state machine, peer-ECU arbitration, vehicle state machine, UDS, secure bootloader — and handling all the simple sensors the vehicle needs (ultrasonic parking sensors, IMU, GPS, environmental, chassis sensors, charging system, vehicle access).
- **Both vehicle profiles supported** — 2-wheeler and 4-wheeler (full passenger car) — via NVM-driven configuration on a single firmware family and (where the harness adapts) a single PCB family.
- **Two-stage controller progression** — STM32H7 for bring-up; **Infineon AURIX TC3xx**, **NXP S32K3**, **TI TMS570 (Hercules)**, or **TI AM263Px** as production-target candidates, with the team selecting.
- **Real-time RTOS** — **SafeRTOS** on the production target; **FreeRTOS** (API-compatible) for bring-up so migration is a license-and-configuration switch, not an architecture rework.
- **Functional safety** — **ASIL-B** at the platform level, **ASIL-C** for safety-critical functions.
- **Security baseline** — secure boot, signed firmware, HSM-backed key storage, secure debug lockout — all from v1.
- **MISRA C 2012** from day one, with documented deviations.
- **Custom DBC-driven CAN messaging** plus **UDS over ISO-TP** for diagnostics.
- **CAN-FD on every segment** from v1; classic CAN only where legacy peers exist.
- **All settings NVM-persisted** with a versioned schema.

## Design Rationale

### Modular Architecture, Configured at Boot

The controller is built as a **modular architecture** — a fixed overall structure (state machines, communication layers, real-time loops) into which **vehicle-specific modules** plug in. The structure stays the same regardless of which vehicle the controller is deployed onto; the modules are what change.

Examples of swappable modules:

- A **throttle module** for the pedal-based 4-wheeler, a different one for the grip-twist 2-wheeler. Same interface to the rest of the firmware, different implementation underneath.
- A **brake-blend module** with 4-channel ABS for the passenger car, a different one with 2-channel braking and cornering ABS for the 2-wheeler.
- A **peripheral set module** describing which lights, switches, and dashboard elements this vehicle has.
- A **BMS profile module** describing the battery configuration this vehicle uses.

The selection of which module variant runs is a **programmable parameter**. The parameter is configured once during commissioning, stored in **NVM (non-volatile memory)** on the controller, and read at boot-up. Changing the vehicle profile of an installed controller is a configuration write, not a firmware reflash.

Where the hardware genuinely differs (pin count, harness wiring), **profile-specific harness adapters** absorb the difference. The PCB family itself remains shared.

The result is **one engineering effort, multiple deployment shapes**. A single firmware build runs on a 2-wheeler and a full passenger car. This is what makes the platform licensable across vehicle classes — the architecture is the product, the vehicle-specific data is just data.

### RTOS Migration as a Configuration Switch

A real-time operating system (RTOS) is the layer that schedules tasks and handles timing on the microcontroller. The choice of RTOS matters for safety certification, but the *application code* on top — vehicle logic, state machines, control loops — does not need to care which RTOS runs underneath, as long as the API surface is consistent.

This is the same modular principle as above, applied to system software:

- All application code is written to **FreeRTOS API conventions** from day one — task creation, queues, semaphores, mutexes, all the standard calls.
- During development and bring-up, the code runs on **FreeRTOS** — an open-source, free, well-documented RTOS, easy to learn and a good fit for students.
- For production, the same code runs on **SafeRTOS** — a commercial RTOS certified for safety-critical use (IEC 61508 SIL 3, ASIL-D capable, ISO 26262 certified). SafeRTOS is API-compatible with FreeRTOS by design.
- The migration is essentially a license + porting-layer switch, not a code rewrite.

The team develops on an accessible RTOS and moves to a certified one for production without rewriting application code. The safety case the team has to build then lives in the application layer, where it belongs, rather than being absorbed by uncertified kernel work.

### Two-Stage Controller Progression

Building a vehicle controller directly on an automotive-grade MCU is expensive, has a steep learning curve, and consumes time better spent on architecture and firmware. Building on an accessible chip first, and migrating once the architecture is proven, is faster overall.

**Stage 1 — Bring-up on STM32**

**STM32** is a widely used, well-documented, low-cost family of microcontrollers that students will already have encountered. Development boards are inexpensive and widely stocked; the toolchain is free; tutorial material is abundant. The team uses **STM32H7** (a higher-end variant with the peripheral count and performance the VCU needs) for bring-up and the first two phases of the build.

**Stage 2 — Production-Target Migration**

Once the architecture and firmware are stable on STM32, the controller migrates to an automotive-grade MCU. Candidates:

- **Infineon AURIX TC3xx** — the industry's most established automotive safety-MCU family.
- **NXP S32K3** — automotive Cortex-M7 family with strong toolchain support.
- **TI TMS570 (Hercules)** — Cortex-R5F lockstep, mature safety pedigree.
- **TI AM263Px** — newer R5F-based, more compute, ASIL-D capable.

All four are **ASIL-D capable** (the highest automotive safety integrity level) and integrate an **HSM** (Hardware Security Module) for cryptographic operations. The team selects based on toolchain familiarity, peripheral fit, cost envelope, and team experience. Selection happens early — before significant production-targeted code is written — and is documented in a Design Rationale Document with the comparison made explicit.

Most of the architecture and firmware code is portable between Stage 1 and Stage 2 because of the modular structure and the FreeRTOS-to-SafeRTOS API compatibility described above. The migration is a focused effort, not a rebuild.

### Robustness in Hardware and Firmware

The VCU is the brain of the vehicle; sloppy electrical or signal handling at the front end results in the kind of "inexplicable shutdown" failure pattern that ends careers and recall programmes. The platform treats robustness as a non-negotiable quality bar, not a nice-to-have once the rest works. The expectations:

- Every power-line input is filtered, clamped, and protected to **ISO 7637-2** transient classes appropriate to the application, with **ISO 16750-2** as the supply specification reference.
- Reverse polarity, overvoltage, and load-dump protection at the input — designed in from the schematic, not bolted on later.
- Signal conditioning on every sensed input — anti-alias filtering, common-mode chokes, TVS clamping where relevant, isolation where required by the topology.
- EMC pre-compliance considered during layout, not at the end — return-path management, isolation-barrier integrity, reference-plane discipline.
- **Plausibility checks** on every sensed value — a measurement outside the physically possible range is rejected, not used.
- **Glitch rejection and debouncing** on protection trips — multiple consecutive valid samples before a trip latches, with instantaneous trips reserved for genuinely catastrophic events.
- **Windowed watchdog**, with all real-time tasks contributing to the kick rather than a single timer.
- **Hard separation** between safety-critical paths (protection decisions, contactor / disconnect control) and best-effort paths (telemetry, logging) at the scheduling and memory level, so a non-safety bug cannot starve a safety task.

## Track Structure

### Hardware Track

The hardware track owns everything physical: the control PCB, the enclosure, power supply and conditioning, all electrical interfaces (CAN/LIN/Ethernet transceivers, analog and digital I/O, driver outputs with current sensing), the HSM-equipped MCU and watchdog wiring, JTAG/SWD provisioning and lockout, mechanical mounting and ingress, and connector pinout management. Deliverables are PCB schematics and layout, mechanical CAD, environmental claims, and the bring-up and production hardware artifacts.

### Firmware Track

The firmware track owns everything that runs on the MCU: the RTOS, the application and vehicle logic, the communication and diagnostic stacks, NVM management, the security implementation (secure boot, signed firmware, OTA, key management), MISRA C 2012 compliance, and the firmware-level safety case. Deliverables are the firmware itself, the DBC file, the UDS service catalogue, the bootloader and secure-update specification, the NVM schema document, the security architecture document, the MISRA compliance report, and the diagnostic coverage worksheet.

### Shared Responsibilities

Some capabilities span both tracks and are jointly owned:

- **Functional safety** — ASIL-B platform / ASIL-C critical functions. Hardware diagnostic coverage and failure-mode analysis feed into the firmware-level safety case.
- **Real-time achievement** — firmware loop rates and jitter budgets are constrained by hardware design choices (peripheral selection, interrupt routing, clock topology, watchdog wiring).
- **Security** — HSM is a hardware feature with a firmware integration. Secure boot, signed firmware, and key management span both tracks.
- **Robustness posture** — the hardware-firmware co-design described in the Design Rationale.
- **Cross-cutting DRDs** — controller selection, RTOS, enclosure, profile-specific harness strategy.

The two tracks share a single architecture review and a single phased build plan. Each phase has hardware-track work and firmware-track work that lands together at a joint exit criterion.

## Functional Targets

### Hardware Track Requirements

#### Power and Electrical

- **Power supply** — automotive-grade input handling per ISO 16750-2; reverse polarity, overvoltage, and load-dump protection per ISO 7637-2.
- **Wakeup and ignition handling** — multiple wakeup sources (ignition, CAN, RTC, charge plug), defined sleep current budget.

#### Communication Interfaces (Physical Layer)

- Multiple **CAN-FD transceivers** — at least four physical segments, matching the zonal architecture (one per zone, plus powertrain or central).
- **Classic CAN transceiver** on at least one segment for legacy peers (race car, older Vayve products).
- **LIN transceivers** — at least two, for slow peripherals consistent with passenger-car topology.
- **Automotive Ethernet** — PHY, connector, and MAC populated on the control board.

#### Digital, Analog, and Driver I/O

- **Digital and analog I/O** sized to the union of both vehicle profiles' sensor counts, with the production harness selecting which pins are populated per profile.
- **Ultrasonic sensor interfaces** — front and rear arrays (typically 4+4 on 4W; configurable per profile).
- **High-side and low-side drivers** for actuators (lighting, contactor coils, motor relays, indicator drivers) with **current sensing on every driver output** so the firmware can verify what actually happened and so the mockup's output verification has something to instrument.

#### Hardware Security and Reliability

- **HSM** integrated on the MCU (AURIX TC3xx, S32K3, TMS570, and AM263Px all integrate one).
- **Hardware-enforced windowed watchdog**.
- **JTAG / SWD** locked in production with authenticated unlock.

#### Mechanical and Enclosure

- **Single enclosure** housing the control board.
- **Environmental rating** appropriate to the target vehicle class — temperature, humidity, vibration, ingress (IP67 minimum, IP68 for the 4W passenger car).
- **Connector pinout** documented and version-controlled, with profile-specific adapters where the two profiles diverge.
- **Thermal design** managing the control board under worst-case ambient and load.
- **Service-friendly** — single-tool access to internal components, board-level replaceability, secure debug interface accessible via authenticated unlock.

### Firmware Track Requirements

#### RTOS and Architecture

- **SafeRTOS** on the production target; **FreeRTOS** acceptable for bring-up (API-compatible).
- **Modular architecture** — drivers, communication stack, application/vehicle logic, and diagnostics in clearly separated layers, with safety-critical and best-effort paths separated at scheduler and memory level.

#### Vehicle Control Logic

- **Vehicle state machine** — off, accessory, ready, drive, reverse, charging, fault, limp-home — data-driven and NVM-configurable.
- **Pedal / grip-twist interpretation, drive-mode arbitration, torque request shaping** — profile-appropriate behaviour from NVM.
- **Brake-blend negotiation** with the chassis ECU at 100 Hz.
- **Protection state machine** — every documented fault condition has a defined response, latching behaviour, and recovery path.
- **Peer-ECU arbitration** — BMS, motor controller, chassis ECU — with defined behaviour when each peer goes silent or returns implausible values.

#### Simple Sensor Handling

The firmware handles the following sensors directly on the MCU. The envelope is sensors with bounded bandwidth, deterministic processing, and well-understood algorithms.

- **Ultrasonic parking sensors** — pulse-echo timing, distance calculation per sensor, basic obstacle classification (near / mid / far), driver-warning output (visual + audible). Front and rear arrays on the 4W profile; absent or minimal on the 2W profile.
- **IMU** — orientation, lean angle (2W), yaw rate, lateral acceleration. Sensor fusion with wheel speeds and steering angle to produce a coherent vehicle motion estimate.
- **GPS** — position, speed, heading, altitude; clock source for telemetry timestamps.
- **Environmental sensors** — ambient temperature, ambient light, rain (where applicable), sun load (where applicable), cabin temperature/humidity.
- **HV and 12 V system sensors** — HVIL, isolation, contactor states, 12 V battery, DC-DC converter status.
- **Charging system sensors and signals** — proximity, control pilot, port flap, port temperature, AC line, DC handshake.
- **Vehicle access sensors** — door / hood / boot / charge-flap open detection, smart key, interior motion.

#### Communication and Diagnostics

- **Custom DBC-driven CAN messaging** — the DBC file is the source of truth for the message catalogue, versioned and source-controlled alongside the firmware, with a documented data dictionary and clear forward-compatibility rules (new fields and messages can be added without breaking existing consumers). DBC management workflow tracked in a Design Rationale Document.
- **CAN-FD on every segment** from v1; classic CAN only as a fallback for legacy peers.
- **LIN** for slow peripherals consistent with passenger-car wiring topology.
- **UDS over ISO-TP** with a documented service catalogue, security access, and routine identifiers.
- **Event and fault logging** with timestamped post-mortem records, retrievable over UDS.

#### Configuration and NVM

- **All operational settings persisted in NVM** — vehicle profile, peripheral set, peer-ECU IDs, threshold values, control-loop coefficients, calibration data, feature flags.
- **Versioned NVM schema** with migration on firmware update.
- **Integrity-protected NVM** (CRC, redundancy on critical regions); safe behaviour on first boot or corruption.
- **Configuration service surface** via UDS DIDs for commissioning and field service.

#### Bootloader and Update

- **Secure bootloader** with signed image verification, anti-rollback, OTA capability, and recoverable fallback.
- **OTA-capable secure update** from v1, sharing the signed-image pipeline used by the wired update path.

#### Security Implementation

- **Secure boot** with **HSM-backed key storage** (HSM provisioned by the hardware track).
- **Signed firmware** with anti-rollback enforcement.
- **Secure debug** — authenticated unlock of the JTAG/SWD interface provisioned by the hardware track.
- **Key management story** documented — signing-key custody, device-unique key provisioning, rotation policy, what happens when a device leaves the field.
- **ISO/SAE 21434** alignment as a roadmap item; v1 ships with the foundations.

#### Coding Standards

- **MISRA C 2012** with documented deviations; static-analysis enforcement in CI.

### Cross-Cutting Capabilities

These capabilities are jointly owned by the hardware and firmware tracks.

#### Real-Time Targets

The targets below are firmware achievements that depend on hardware design choices (peripheral selection, clock topology, interrupt routing).

- **Torque-request loop** to motor controller: **1 kHz**, **100 µs jitter budget**.
- **Brake-blend negotiation** with chassis ECU: **100 Hz**, **1 ms latency budget**.
- **Ultrasonic-driven warning latency**: **≤ 100 ms** from obstacle detection to driver warning.
- **Telemetry** to monitoring stack: **10 Hz minimum**, no real-time guarantee.
- **UDS service response** per ISO 14229 timing (typically 50 ms for routine services).

#### Functional Safety

- **ASIL-B** at the platform level; **ASIL-C** for safety-critical functions (motor-torque limiting, contactor disconnect, protection trips, fault-mode arbitration).
- **Diagnostic coverage accounting** per safety-critical function — FMEDA-style discipline across hardware failure modes and firmware coverage measures, even if no formal FMEDA in v1.
- **SafeRTOS** on the production controller carries the kernel-level safety case; the firmware track builds the application-layer safety case on top.
- Hardware FMEDA inputs (component failure rates, diagnostic coverage from current-sensing on driver outputs, watchdog coverage, HSM-backed integrity checks) feed the firmware-level safety argument.

## Phased Build Path

The team owns the actual phase plan; the decomposition below is a suggestion. Each phase lands hardware-track and firmware-track work together at a joint exit criterion.

### Phase 1 — Control Foundation

Goal: a control MCU that drives a placeholder VCU role on the Phase 1 mockup.

**Hardware track:**
- **STM32H7** development board as the placeholder control MCU.
- Minimal harness adapters to the mockup rig.

**Firmware track:**
- FreeRTOS, basic vehicle state machine.
- Single CAN segment talking to placeholder BMS and motor controller.
- Pedal/throttle interpretation, simple drive cycle execution.

**Joint exit criterion:** passes the mockup track's Phase 1 test suite.

### Phase 2 — Full Control Stack

Goal: production-shaped control behaviour on bring-up hardware.

**Hardware track:**
- Custom control PCB designed and fabricated with full I/O for both vehicle profiles — multi-segment CAN-FD transceivers, LIN, Automotive Ethernet, high-side/low-side drivers with current sensing, ultrasonic interfaces, HSM-equipped bring-up MCU.
- Hardware-enforced windowed watchdog operational.

**Firmware track:**
- SafeRTOS migration (or FreeRTOS-Plus with safety extensions until SafeRTOS licensing is in place).
- Multi-segment CAN-FD, LIN, chassis-ECU brake blend.
- Full protection state machine and peer-ECU arbitration.
- UDS over ISO-TP, secure bootloader scaffolding.
- MISRA C compliance posture established.
- Ultrasonic sensor handling and the rest of the simple-sensor envelope.

**Joint exit criterion:** passes the mockup track's Phase 2 test suite; validated end-to-end against the BMS firmware track's real interface.

### Phase 3 — Production-Target Migration

Goal: automotive-grade hardware and the certified firmware configuration that runs on it.

**Hardware track:**
- Control board ported from STM32H7 to the chosen production target (AURIX TC3xx, S32K3, TMS570, or AM263Px).
- AEC-Q-graded components on critical paths.
- Full automotive qualification path.

**Firmware track:**
- Firmware ported to the production target with a porting-layer adaptation only.
- SafeRTOS running on the certified configuration.
- HSM-backed secure boot operational end to end.

**Joint exit criterion:** end-to-end functional and qualification testing on production-target hardware.

### Phase 4 — Real-Vehicle Integration

Goal: deliver on the bench-to-vehicle commitment.

**Hardware track:**
- Profile-specific harness adapters for each integration target.
- Physical mounting and ingress validation in each vehicle.

**Firmware track:**
- NVM commissioning configuration per target.
- Integration tuning where mockup-vs-vehicle divergence is found.

**Joint integration:**
- Integration onto the race team's car (4W profile).
- Integration onto a Vayve Mobility product (4W profile, subset configuration).
- Integration onto an Indian-market 2W reference (2W profile).
- Field findings flow back into the design via DRDs and design updates.

## Technical Considerations to Own

### Hardware Track

- **Control MCU selection** from the candidate set (**Infineon AURIX TC3xx**, **NXP S32K3**, **TI TMS570 (Hercules)**, **TI AM263Px**; **STM32H7** for bring-up) — evaluated against toolchain familiarity, ASIL-readiness (all four production candidates are ASIL-D capable), HSM capability, peripheral fit, cost, and supply chain. Co-owned DRD with the firmware track.
- **Profile-specific harness adapters** — what's shared, what diverges, how the variation is documented and version-controlled.
- **PCB power, return-path, and EMC topology** — choices that drive what passes EMC pre-compliance and what doesn't.
- **Thermal headroom** on the production-target chip and driver outputs under worst-case ambient and load.

### Firmware Track

- **HSM key management workflow** — provisioning, rotation, custody, what happens when a device leaves the field. Hardware track provisions the keys; firmware track designs the workflow.
- **OTA update orchestration** — atomic update, fallback behaviour, rollback policy.
- **SafeRTOS licensing and certification scope** — what configuration is certified, what code lives outside the certified boundary, how the application's safety case interacts with the kernel's.
- **MISRA C 2012 from day one**, with documented deviations. Static-analysis enforcement in CI (Cppcheck, PC-lint Plus, Polyspace, or equivalent). The team chooses the specific toolchain; the standard is fixed. Retrofitting MISRA compliance is far more expensive than writing in-compliance from the start.

## Starting Points

### Hardware Track

- **STM32H7 / H5** as the bring-up control MCU.
- **Infineon AURIX TC3xx**, **NXP S32K3**, **TI TMS570 (Hercules)**, and **TI AM263Px** as production candidates.
- AEC-Q-graded passive and active component families on critical paths.

### Firmware Track

- **FreeRTOS** for bring-up, **SafeRTOS** as the production target.
- **Open-source UDS / ISO-TP stacks** as starting references.
- **Established DBC tooling** (Vector CANdb++, Kvaser Database Editor, cantools) for the message catalogue.
- **Python + pytest** for the test harness — the common toolchain across the project's other tracks, keeping the test surface coherent.

## Deliverables

### Hardware Track

- **Control PCB** — schematic, layout, working prototype on the bring-up MCU, second prototype on the production target.
- **Single enclosure** — mechanical design housing the control board, with environmental claims documented and validated.
- **Connector pinout document** — version-controlled, with profile-specific adapter definitions.
- **EMC pre-compliance report** and any in-house emissions/immunity evidence.

### Firmware Track

- **Firmware architecture document** with safety-path separation and configuration data model.
- **DBC file** for the application-layer CAN catalogue, versioned alongside the firmware.
- **UDS service catalogue** documenting services, DIDs, routines, security-access levels.
- **Bootloader and secure-update specification**.
- **NVM schema document** with versioning and migration policy.
- **MISRA C compliance report** with the deviation log.

### Joint Deliverables

- **Security architecture document** covering boot, update, debug lockout, key management — co-authored across hardware (HSM, debug provisioning) and firmware (key workflow, signed update).
- **Diagnostic coverage worksheet** per safety-critical function — hardware failure modes and firmware coverage measures together.
- **Test reports** showing the unit passes the mockup track's full test suite at each phase boundary.
- **Design Rationale Documents (DRDs)** for major decisions — controller selection, RTOS, enclosure, profile-specific harness strategy.

## Out of Scope

- Building the test rig or test catalogue (mockup track).
- Designing the BMS or motor controller (BMS family and external suppliers).
- Designing the actual vehicle (race team and Vayve teams).
- **Any SAE Level 3+ autonomous-driving function** — a different regulatory and engineering universe.
- Formal certification testing in accredited labs — design plans and in-house pre-compliance evidence are in scope; accredited execution is outsourced.

## Inter-Track Dependencies

- **Vehicle mockup track** — provides the development bench, validation suite, and authoritative interface contract. The VCU design track validates against this rig before any vehicle integration. The two tracks co-author the scenario catalogue and the fault-injection mapping to VCU fault responses.
- **48V BMS firmware track** — the VCU is a peer of the BMS on the bus; the two teams jointly own the BMS-VCU section of the DBC and the protection-handoff behaviour.
- **Vehicle monitoring track** — consumes telemetry from the VCU; the VCU exposes telemetry data on a documented schedule and format.
- **Race team and Vayve vehicle teams** — provide the deployment targets and the integration support for Phase 4.

## Future Add-Ons

### Perception Module

Adding a perception coprocessor board to the platform enables ADAS functions — Forward Collision Warning, Automatic Emergency Braking, Lane Departure Warning, Adaptive Cruise Control, and beyond. The coprocessor consumes camera, radar, and (optionally) LiDAR streams and produces object lists, lane geometries, and occupancy grids that the control MCU can consume over the Automotive Ethernet interface present on the control PCB.

Typical coprocessor SoC candidates: NXP S32G2/G3, TI Jacinto J784S4, Renesas R-Car V4H, Qualcomm Snapdragon Ride, NVIDIA DRIVE Orin. Linux + PREEMPT_RT or an automotive equivalent (Adaptive AUTOSAR, QNX) runs on the SoC. Adding perception brings **ISO 21448 (SOTIF)** into scope alongside ISO 26262.
