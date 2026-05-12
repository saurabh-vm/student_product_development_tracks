# Vehicle Control Unit — Vehicle Mockup & Test Harness Track

## Objective

Build a full vehicle-level test jig that can emulate both a **2-wheeler** and a **4-wheeler (full passenger car)** — motor, motor controller, battery pack, zone controllers, sensors, actuators, peripherals (lights, switchgear, dashboard) and a multi-segment communication backbone — sized and behaved to match real reference vehicles in each class. The 4-wheeler profile is anchored to a full Indian-market passenger car (Tata Nexon EV, Tata Curvv EV, Mahindra XUV / BE-series, or similar) and designed as a **superset** that can then be deployed onto simpler 4W targets — including the race team's car and Vayve Mobility quadricycles — as configuration subsets. The 2-wheeler profile anchors to an Indian-market reference 2-wheeler (Ather, Ola, Bajaj, or TVS class). The jig becomes the **authoritative reference platform** against which any VCU is validated, regardless of which vehicle target it's destined for. Once a VCU passes the jig's test suite, it deploys onto real vehicles — initially the race car, eventually Vayve vehicles and full passenger cars. The mockup track also owns the on-vehicle integration that follows, so the path from bench to vehicle is a single team's responsibility.

## Product Scope

- **Two distinct vehicle profiles** — the jig must support emulation of a **2-wheeler** and a **4-wheeler (full passenger car)**. The 4W profile is sized as a passenger-car superset so that a VCU validated on the rig can deploy onto any 4W target — including simpler quadricycles like Vayve EVA/CT4 and the race team's car — as configuration subsets. The team decides whether to build a single reconfigurable platform or two parallel rigs, with the decision documented along with its cost and engineering trade-offs.
- A **full vehicle-level jig** containing:
  - A **real motor** on a **load bench** — a simple dyno using a higher-rated control motor to apply load profiles representative of real road conditions (grade, rolling resistance, wind load, transient changes).
  - A **data emulator path** as an alternative to the load bench, for simulated scenarios and destructive/fault cases that would damage real hardware. The two sub-rigs share the same VCU-facing electrical and message interfaces.
  - A **real production-spec motor controller** consuming torque commands over CAN.
  - A **real battery pack** (or a high-fidelity emulator when the real pack is not available), interfaced through the BMS the other tracks are producing.
  - **Real pedals** (throttle, brake), real gear/mode selectors, real ignition and wakeup lines — not analog stand-ins. For the 2-wheeler profile, **real handlebar switchgear and grip-twist throttle** in place of pedals.
  - **Full peripheral and switchgear emulation** for each vehicle profile — lighting, dashboard, steering stalk (4W) or handlebar switches (2W), HVAC where applicable, mirrors, doors and windows where applicable, side-stand and helmet sensors (2W where applicable).
  - Sensor stimulation hardware for the analog/digital inputs the VCU is expected to handle.
- A **multi-segment CAN architecture in a zonal layout** — zone controllers around the rig aggregating local I/O, connected to the central VCU via a backbone. This is the modern architecture serious production vehicles are moving toward, and the rig models it from day one rather than retrofitting later.
- **Behaviour-level emulation** for the BMS, motor controller, chassis ECU, and dashboard/HMI; **message-level emulation** for body, lighting, and any other peripheral ECUs.
- **Fault injection across all five axes** — bus, peer ECU, sensor, supply, and timing — scripted and reproducible.
- A **scriptable test framework** built on Python + pytest with a **starter scenario catalogue**, authored by this track and extended by the VCU design track during development.
- A **placeholder VCU** built on STM32 that exists solely to prove the rig is correct end-to-end. It is not a competing VCU design.
- **On-vehicle integration** onto the race team's car as the first real-vehicle target, with Vayve Mobility vehicles as the next-stage integration targets, owned by this track.

## Design Rationale

### Mockup as the Reference Platform

In a traditional flow, the mockup mimics the real vehicle. This track inverts the relationship: **the mockup is built to match real reference vehicles in specification and behaviour, but the mockup itself becomes the authoritative contract** any VCU is validated against. Once a VCU passes the mockup's full test suite, it deploys onto real vehicles with the expectation that they will behave consistently. Any divergence becomes a documented gap that is either patched in the mockup (because the mockup was wrong) or quirked in the VCU (because the real vehicle is a special case). The mockup definition is therefore a versioned, source-controlled artifact, not a snapshot of a particular day's wiring.

This framing has practical consequences:

- The mockup specification is treated as a product specification, with releases, change control, and traceability.
- Reference-vehicle data flows *into* the mockup spec, not directly into the VCU. The VCU only sees what the mockup says.
- The mockup team owns the on-vehicle integration too — the same team that defined the contract proves it in the field.

### Zonal Multi-Segment Architecture

Zonal architecture — physical zones around the vehicle aggregating local I/O, connected to a central compute via a backbone — is the direction modern production vehicles are heading. Tesla, Lucid, and an increasing number of OEMs have moved this way because flat or domain-segmented CAN topologies do not scale to the wiring, weight, and bandwidth demands of a software-defined vehicle.

Modelling this from day one in the rig has two motivations:

- **It teaches the right architecture.** Students who learn VCU development on a single-segment CAN bus will produce VCUs that don't generalise to current-generation vehicles. Building the rig zonally forces them to grapple with gateway behaviour, cross-zone fault propagation, backbone bandwidth, and message routing — all real production concerns.
- **It is honest about the integration targets.** A race car may be flat-segmented today, but Vayve products and any production deployment will not stay that way. A mockup ahead of the curve is more useful than a mockup that has to be retrofitted.

Backbone choice (Automotive Ethernet vs CAN-FD vs a hybrid) is itself a design decision the team owns, evaluated on bandwidth, cost, and integration-target realities.

### Real Hardware Where It Matters, Emulation Where It Doesn't

The motor, motor controller, battery pack, and pedals are real. The chassis ECU is **behaviour-level emulated** — it is tightly coupled to the VCU's torque-control and regen-blend loops, and message-level emulation would not catch real bugs there. Lighting and body controllers stay at message-level. The decision rule is simple: **anything the VCU's control loops actually have to *feel* or negotiate with in real time** — torque response, regen behaviour, pedal mechanics, current limits, vehicle dynamics, stability events — is real or behaviour-level emulated. **Anything the VCU just listens to or commands occasionally** — turn-signal state, door status, ambient-light sensor — is message-level emulated. This concentrates cost and complexity where the learning is and keeps the rig tractable.

### Fault Injection First

The rig exists primarily to find bugs in VCU candidates. The happy-path test catalogue is the smaller half of the work. **All five fault axes — bus, peer ECU, sensor, supply, timing — must be injectable on demand, scripted, and reproducible.** A test that requires an engineer to physically unplug a connector to simulate a fault is not a test the rig can run on its 100th repetition.

### Layered Capability Build-Up

The rig is structured as four layers of capability rather than as a flat list of subsystems:

1. **Base Rig** — what makes a VCU come alive end-to-end: real powertrain, real driver controls, single bus, behaviour-emulated peers.
2. **Chassis Emulation Engine** — vehicle dynamics, ABS / ESC / EPS / TC behaviour, regen-blend negotiation; the heaviest behavioural load on the rig.
3. **Full Peripheral and Sensor Build-Out** — multi-segment zonal CAN, full per-profile peripheral catalogue, comprehensive auxiliary sensors.
4. **Test Capabilities and Instrumentation** — fault injection, output verification, scenario framework, logging.

Each layer is buildable and validatable on its own, and each adds capability to the one before without restructuring it. The phased build path follows the layers in order. Rig safety is cross-cutting and applies regardless of which layer is active; on-vehicle integration is the final capability the bench is engineered toward.

## Functional Targets

### Layer 1 — The Base Rig

The base rig is the smallest setup that brings a VCU to life end-to-end: real powertrain, real driver controls, a single bus, behaviour-emulated peers. Sufficient for a basic drive cycle, and the foundation everything else extends. The question this layer answers is *what does it take for a VCU to spin a motor in response to a throttle command, with a BMS reporting state-of-charge?*

**Powertrain (real):**

- **Real motor** representative of the reference vehicle's class.
- **Load bench** — a simple dyno using a higher-rated control motor to apply load profiles representative of real road conditions (grade, rolling resistance, wind load, transient changes).
- **Real production-spec motor controller** consuming torque commands over CAN.
- The two motor sub-rigs — **real motor on the load bench** and **data emulator** — share the same VCU-facing electrical and message interface so test code is portable between them. The data emulator path is used for simulated scenarios and destructive/fault cases that would damage real hardware.

**Battery (real or high-fidelity emulator):**

- **Real battery pack** once available from the BMS family; **high-fidelity emulator** as a stand-in until then.
- Interfaced through the real BMS once that arrives; a **behaviour-level BMS emulator** stands in before then, modelling charge/discharge state, current limits, contactor responses, SOC reporting, and basic fault behaviour.

**Driver controls (real):**

- **Throttle pedal** (4W) or **grip-twist throttle** (2W).
- **Brake pedal** (4W) or **front and rear brake levers** (2W). Brake-pressure sensors and switches present where the reference vehicle has them.
- **Gear / drive-mode selector** — P / R / N / D for 4W; eco / sport / mode-select for 2W.
- **Ignition / power-mode selector and wakeup lines** — same wake-up philosophy as the production vehicle.

**Communication (initial):**

- **Single CAN segment** initially, with the VCU as the central node. Multi-segment zonal architecture comes in Layer 3.
- Real-time CAN logging and message replay.

**Peer ECU emulators at behaviour level:**

- **Motor controller** — torque-request consumption, current/voltage reporting, basic fault behaviour.
- **Dashboard / HMI** — SOC, speed, drive mode, basic warnings.
- **Body and lighting** — message-level only at this stage, sufficient for the VCU to acknowledge command/response cycles.

**Sensor stimulation hardware** for analog and digital inputs not present as physical hardware. Stimulation detail is covered in Layer 4; what Layer 1 needs is the basic capability for throttle, brake-pressure, and a small set of supply and wakeup signals.

**Mechanical and integration:**

- Mounting, harness routing, and access designed for repeated test setups.
- Reconfiguration paths planned for layering on peripherals and the second profile later.

**Placeholder VCU** built on an STM32-class development board, running through a basic drive cycle. Its only job is to prove the rig is end-to-end correct; it is not a competing VCU design.

**Python + pytest harness** with one or two starter scenarios — enough to demonstrate a VCU drives through a basic throttle-and-brake cycle on the rig.

### Layer 2 — The Chassis Emulation Engine

The chassis emulator is the heaviest behavioural load on the rig and the part that distinguishes a credible VCU test platform from a powertrain bench. It owns vehicle dynamics, brake arbitration, and stability behaviour, and negotiates with the VCU in real time. It is its own engineering sub-product within the rig.

#### Why Software-Emulated, Not Physical

Real ABS modulators, brake actuators, steering racks, and brake calipers are physical apparatus tied to physical wheels. A bench mockup has no wheels, no hydraulics, and no steering rack. Replicating them physically is neither feasible nor necessary — the VCU under test experiences these systems only as CAN messages and electrical signals (wheel-speed reports, brake-pressure values, steering angle, ABS-active flags, torque-cut requests, ESC intervention events). Producing those signals from software is faithful to what the VCU sees and unlocks behavioural fault injection that physical hardware cannot match.

What the bench cannot validate is the hydraulic latency, steering-rack mechanics, or brake-actuator response curves of a real vehicle. Those validations belong to real-vehicle integration, not the bench. **The bench validates the VCU's logic; the vehicle validates the physics it operates against.**

#### Emulation Pattern (ABS / ESC / EPS / Traction Control)

The same pattern applies across all four systems:

- The chassis ECU emulator runs the **real ABS / ESC / EPS / TC control logic** in software, fed by the vehicle dynamics model.
- **Virtual actuator commands** (per-wheel brake force, steering assist torque, selective-braking patterns) are computed by the control logic and fed back into the vehicle dynamics model, where they affect simulated vehicle motion.
- The dynamics model's outputs — wheel speeds, accelerations, yaw rate, lateral acceleration, steering angle — report to the VCU over the appropriate CAN segments at production-realistic rates and noise levels.
- CAN signalling to the VCU carries the user-visible behaviours — ABS-active flags, ESC intervention events, torque-cut requests, fault and degraded-mode reports.
- The VCU's responses (motor torque actually delivered, regen modulation, drive-mode behaviour) feed back into the dynamics model on the next cycle.

Implications for what is and isn't physical:

- The **motor is real** and responds to actual torque commands from the VCU. The **brake pedal/lever is real** for haptic feel and brake-pressure signal generation. The **steering wheel / handlebar** is real as an input device.
- **No real brake actuator, no real ABS hydraulics, no real steering-rack motor** is present. Brake force, ABS modulation, steering assist, and selective braking all happen in software, projected into the dynamics model.
- **Fault injection becomes trivial** — making ABS misbehave, forcing an ESC trip, simulating a stuck EPS, or producing asymmetric brake patterns is a value change in the emulator. This is a major advantage of the software-emulated approach.

#### Vehicle Dynamics Model

- **Bicycle (single-track) model** as the minimum fidelity bar for the 4W profile.
- **Motorcycle dynamics** with lean-angle and roll dynamics as the minimum fidelity bar for the 2W profile.
- The model is a **versioned spec artifact** alongside the rest of the mockup spec.
- Open-source starting points (CommonRoad-Vehicle-Models, OpenScenario-aligned tooling) or commercial libraries (IPG CarMaker, dSPACE ASM, Simulink Vehicle Dynamics Blockset) — the team chooses. A custom bicycle model is an acceptable starting point.

#### Chassis Behaviour — Both Profiles

- **Vehicle dynamics model** producing physically plausible wheel speeds, vehicle speed, and acceleration in response to motor torque, brake input, and road conditions imposed by the load bench or data emulator.
- **Regen-blend negotiation** with the VCU at 100 Hz. The emulator and the VCU under test are jointly responsible for delivering driver-requested deceleration through some mix of regen torque and friction braking. Pedal or lever feel must remain consistent regardless of the blend. This is arguably more delicate on 2W where front/rear brake balance interacts with regen.
- **ABS behaviour** — wheel-slip detection, modulator activation, torque-cut request to the VCU on activation, time-correct release. Single-channel and dual-channel modes selectable per the reference vehicle's spec.
- **Wheel-speed plausibility** consistent with the vehicle dynamics state.
- **Fault modes** — sensor failures, bus-side faults from the chassis ECU's reported state, degraded modes — all injectable from the test framework (Layer 4).

#### Chassis Behaviour — 4W Passenger Car Additions

- **ESC (Electronic Stability Control)** — yaw-rate-and-lateral-acceleration-driven detection of skid, selective braking response, and torque-cut commands to the VCU.
- **Traction control** activation on acceleration-side wheel slip.
- **EPS / steering behaviour** — steering-angle reporting and assist-torque commands.
- **TPMS aggregation** across all four wheels.

#### Chassis Behaviour — 2W Additions

- **Cornering ABS** using IMU lean angle — modulating ABS response based on lean to prevent low-side under braking in a turn.
- **Combined Braking System (CBS)** where the reference vehicle supports it — coordinated front/rear braking from a single lever input.
- **Lean-angle reporting** for VCU functions that consume it (drive-mode arbitration, regen limiting in turns).

### Layer 3 — Full Peripheral and Sensor Build-Out

The base rig and the chassis engine are the rig's core. This layer is the breadth that turns the rig into a passenger-car-grade validator — multi-segment zonal CAN, full per-profile peripheral catalogue, comprehensive auxiliary sensors. None of this is necessary to validate a basic VCU drive cycle, but all of it is necessary to validate a production-shaped VCU.

#### Multi-Segment Zonal Communication

- **Multi-segment CAN in a zonal layout** — at least four zones (front-left, front-right, rear-left, rear-right) plus a central compute zone hosting the VCU, with documented mapping of which ECUs and sensors live in which zone.
- **Backbone choice** between Automotive Ethernet, CAN-FD, or a hybrid — justified in a design document evaluating bandwidth, cost, and integration-target realities.
- **CAN-FD** for the higher-bandwidth chassis and drivetrain segments where the reference passenger car uses it.
- **LIN buses** for slower peripherals consistent with passenger-car wiring topology — HVAC control panel, mirror motors, rain and light sensors, steering wheel switch matrices, seat motors and memory, ambient lighting nodes. Multiple LIN segments expected, each rooted on a zone controller.
- **Gateway behaviour** modelled either inside the VCU under test or in a dedicated rig component — the rig should support testing either architecture.
- Bus loading, termination, and error injection per segment, controllable from the test framework.

#### Expanded ECU Emulation

Beyond the Layer 1 minimal set:

- **Behaviour-level emulation** for the BMS (when the real BMS isn't connected), motor controller, dashboard/HMI. The chassis ECU emulator (Layer 2) is the heaviest peer in the rig and is covered there.
- **Message-level emulation** for body, lighting, comfort, charging gateways, infotainment, and any other peripheral ECUs.
- Each emulator's behaviour is **versioned and configurable** from the test framework so a single test can put the rig into a specific peer-ECU state.

#### Peripherals and Switchgear (Per Vehicle Profile)

The jig emulates the **full peripheral ecosystem** of both vehicle profiles. Many items overlap between profiles; vehicle-specific items are called out. Every peripheral is documented in the mockup spec with its protocol (CAN ID, analog input, etc.), its expected behaviour in normal operation, and the fault modes it can be put into for testing.

**4-Wheeler (Full Passenger Car) Profile**

- **Lighting**: projector or LED headlights (low/high beam) with auto-leveling, DRLs, taillights, brake lights, indicators (front, rear, side repeaters on mirrors), fog lights (front and rear where applicable), reverse light, cabin and dome lights, ambient cabin lighting, boot light, glove-box light.
- **Steering stalk inputs**: turn signals, wiper control (with rain sensing where applicable), washer trigger, high-beam flash, cruise control (standard, with adaptive cruise where the reference vehicle supports it).
- **Steering wheel controls**: horn, audio / HMI controls, cruise buttons, paddle shifters for regen-level selection, voice-command activation, driver-display controls, driver assistance toggles.
- **Pedals**: throttle, brake; brake-pedal feel tuned for the one-pedal regen behaviour the VCU implements.
- **Switchgear**: ignition / power-mode selector, drive-mode selector (eco / city / sport), gear / direction selector (P / R / N / D), electronic parking brake, hazard switch, all-window switches, all-door switches, central locking, child lock indicators.
- **Dashboard / HMI**:
  - Driver instrument cluster (digital, 7–12") — speedometer, SOC and range, drive-mode indicator, warning lights (ASIL-coded), gear indicator, odometer and trip meters, navigation prompts.
  - Center infotainment display (10–12" touchscreen) with Android Auto / Apple CarPlay surface emulation.
- **HVAC**: dual-zone or full automatic climate control, blower and temperature control, vent direction, defogger (front and rear), heated steering wheel and heated/ventilated seats where the reference vehicle supports them.
- **Mirrors**: power adjust, fold / unfold, heating, auto-dimming for the interior rear-view mirror.
- **Comfort and convenience**: power windows on all doors, sunroof / panoramic roof where applicable, wireless phone charging, USB-C ports, ambient lighting nodes.
- **Chassis sensors**: wheel speed (one per wheel), steering angle, yaw rate, lateral acceleration, IMU, GPS, TPMS (one per wheel), brake pressure, suspension position where applicable.
- **Active safety sensors**: ABS wheel sensors, ESC / yaw-stability sensors, airbag sensors (front, side, curtain) with deployment outputs treated as commanded actuators, seat occupancy sensors, seatbelt sensors.
- **Parking and proximity sensors**: ultrasonic arrays, typically four front and four rear, configurable by reference vehicle.

**2-Wheeler Profile**

- **Lighting**: headlight (low/high beam), taillight, brake light, indicators (front and rear), hazards.
- **Handlebar switches**: high/low beam, indicators, horn, hazard, mode select, ignition kill.
- **Throttle**: grip-twist potentiometer (different mechanics from a pedal; torque-mode commanded the same way over CAN).
- **Brake levers**: front and rear, with brake-light switch and CAN report; brake-pressure sensing where the reference vehicle supports it.
- **ABS and braking subsystem**: front and rear wheel-speed sensing for ABS, ABS modulator status, cornering ABS where the reference vehicle supports it (using IMU lean angle), Combined Braking System (CBS) behaviour where applicable.
- **Switchgear**: ignition / power-mode selector, drive-mode selector (eco / sport), side-stand sensor, helmet sensor where the reference vehicle has one.
- **Dashboard / HMI**: speedometer, SOC and range, drive-mode indicator, warning lights including **ABS status warning**, gear indicator where applicable, odometer.
- **Chassis sensors**: wheel speed (one per wheel), IMU (lean and tilt sensing, used by both stability functions and cornering ABS), GPS.

#### Auxiliary and System Sensors (Both Profiles)

Beyond the user-visible peripherals, the VCU consumes a set of system-level and auxiliary sensors essential to vehicle operation but not directly part of the driver experience. These follow the same emulation pattern as the chassis-system sensors — there is no physical equivalent on the bench, and the VCU sees only their CAN or analog signals. All values are computed or scripted by the rig software, published at production-realistic rates with realistic noise, and exposed to the fault-injection framework.

*Environmental sensors:*

- **Ambient temperature** (external) — consumed by HVAC, BMS pre-conditioning, range estimation, and motor/inverter thermal management.
- **Ambient light sensor** — for auto-headlight activation and instrument-cluster auto-dimming.
- **Rain sensor** — for auto-wiper activation; 4W only typically.
- **Sun-load sensor** (cabin) — for automatic HVAC control; 4W only.
- **Cabin temperature** — closed-loop HVAC reference.
- **Cabin humidity** and **air-quality** sensors where the reference vehicle supports automatic recirculation or air filtration.

*High-voltage system sensors and signals:*

- **HV Interlock Loop (HVIL)** continuity — safety-critical; an HVIL break must propagate to a defined safe state.
- **Isolation monitoring** report — typically reported by the BMS, but the VCU consumes the result and arbitrates the vehicle response.
- **HV contactor commanded state and feedback** — pre-charge, main, fast-charge contactors.

*Charging system sensors and signals:*

- **Charge-port connection detected** — proximity and control-pilot signalling per the relevant standard (Type 2, CCS, CHAdeMO, or 2W-specific connector).
- **Charge-port flap** position.
- **Charge-port temperature** — for high-current DC fast-charge thermal management.
- **AC line voltage and current** at the onboard charger.
- **DC fast-charge handshake state** — IEC 61851 / DIN 70121 / ISO 15118 messaging surface as appropriate.
- **Charging contactor states** and commanded outputs.

*Thermal management sensors:*

- **Battery coolant temperature, pump status, flow** (where the reference vehicle uses liquid cooling).
- **Motor coolant temperature**.
- **Inverter / MCU temperature**.
- **Radiator fan speed** and commanded state.
- **Heat-pump refrigerant pressures** and compressor state where the reference vehicle uses a heat pump.

*12 V auxiliary system:*

- **12 V battery voltage** — critical for wakeup, sleep, and quiescent power management.
- **12 V battery temperature** where instrumented.
- **DC-DC converter status** — output current, fault state, thermal status.

*Vehicle access and security:*

- **Door open/closed sensors** (per door, 4W) — distinct from the door switches that command locking.
- **Hood / bonnet open** sensor.
- **Boot / trunk open** sensor.
- **Charge-port flap open** detection.
- **Smart-key / proximity / passive-entry** signals where the reference vehicle supports them.
- **Interior motion sensor** for alarm functionality where applicable.

*Safety and crash:*

- **Crash detection** — front, side, and rear impact sensors and rollover sensor (commonly part of the airbag control unit, but the VCU consumes the trigger and orchestrates the vehicle response — HV disconnect, hazard lights, door unlock, eCall trigger).
- **Seatbelt buckle status** per seat (4W).

*Service and maintenance:*

- **Brake fluid level** sensor (4W) — purely a CAN signal in the mockup since no real hydraulics exist.
- **Washer fluid level**.
- **Coolant level** (where liquid cooling is present).

*2W-specific additions:*

- **Stand-position sensor** for centre stand where the reference 2W has one, in addition to the side-stand sensor covered earlier.
- **Swappable battery dock sensor** — battery present, latched, communicating — where the reference 2W uses removable batteries (Ola, Bounce, and a number of newer Indian-market 2W products).

### Layer 4 — Test Capabilities and Instrumentation

The rig's reason for existing. The layers above are stage props; what makes the platform valuable is the ability to stimulate, inject faults, verify outputs, and run repeatable scenarios at scale. These capabilities apply across all preceding layers and grow with them — a test framework that can drive Layer 1 should drive Layers 2 and 3 by extension, not by rework.

#### Sensor Stimulation

- Real peripheral switchgear, pedals or grip-twist throttle, and brake levers are physical (Layer 1 and Layer 3). The targets below cover the **signal-level** stimulation of those peripherals and of any analog/digital inputs the rig cannot economically reproduce in physical form.
- Plausibility characteristics on real sensors — dual-channel sensors, ratiometric outputs, realistic noise floor — preserved end-to-end so a VCU's input front-end sees production-realistic signals.
- **Stimulation hardware** for analog and digital inputs not reproduced in physical form — generating clean signals with controllable impedance, noise, levels, and fault states.
- **Ultrasonic sensor stimulation** — simulated pulse-echo timing patterns producing the distance values the VCU's ultrasonic processing expects, configurable per-sensor and per-array, including obstacle scenarios at varying distances.

#### Output Verification and Instrumentation

A complete test rig stimulates the VCU's inputs *and* verifies the VCU's outputs. CAN observation alone is not sufficient — the VCU may broadcast "turn signal on" while failing to energise the lamp output, or report "motor torque 50 Nm" while the actual delivered torque is different. The rig must measure what the VCU actually *does*, not only what it says it's doing.

For every output the VCU commands, the rig instruments the output path:

- **Lighting and lamp drivers** — current sensing on each driver output to confirm the lamp actually receives current when commanded on, with timing accurate enough to verify duty cycles for PWM-driven LEDs.
- **Motor torque** — measured at the load bench (torque sensor or current-based estimation at the motor controller) and compared against the commanded value, with delivery latency reported.
- **HV contactor states** — auxiliary contacts or current sensing on the contactor coil and on the load side, confirming the contactor actually closed or opened when commanded.
- **Brake-light, brake-pressure, and indicator outputs** — measured on the output rail rather than read back from the same CAN signal the VCU populated.
- **HVAC blower, wiper motor, washer pump, mirror motors** — current sensing on each output, confirming activation and approximate runtime.
- **Charging contactor and DC-DC converter commands** — verified at the actual contactor and converter outputs, not only on CAN.
- **Safety-critical outputs** — HV disconnect, hazard lights, eCall trigger, door unlock under crash — instrumented separately because their misfiring carries the highest cost.

Output instrumentation closes the validation loop. A test case is "pass" only if the VCU produces the expected CAN message *and* the rig measures the corresponding physical effect on the output. Discrepancies between the CAN report and the measured output are themselves a defect class — the VCU is misreporting its own actuation, which is a real bug in production VCUs.

#### Fault Injection — All Five Axes

- **Bus faults**: bus-off injection, malformed frames, ID collisions, late responses, missing messages, repeated frames, scrambled signals. For any Automotive Ethernet backbone segment: frame drops, sequence errors, MAC-layer corruption, link-loss and recovery cycles.
- **Peer ECU faults**: BMS or motor controller silent / reporting fault / returning implausible values / responding slowly / responding late.
- **Sensor faults**: pedal stuck, sensor open/short, plausibility violations, out-of-range values. For ultrasonic arrays: phantom echoes, missing echoes, cross-talk, frozen distance values.
- **Supply faults**: low voltage, transient drop-out, brown-out during operation, slow rise, slow fall, ripple injection.
- **Timing faults**: jitter on scheduled messages, scheduler stress on the VCU's expected service rate.
- Every documented VCU fault response in the design track's spec has at least one corresponding rig test case that triggers it.

#### Drive Profiles and Scenarios

- **Scriptable scenario framework** in Python — every scenario is code, versioned and reviewable.
- **Starter catalogue** covering acceleration sweeps, regen profiles, hill-hold and hill-start, cruise-like behaviours, low-SOC behaviour, full-throttle-to-release, fault-condition recovery, and a structured drive-cycle loop.
- The VCU design track authors additional scenarios; the framework supports co-development of the catalogue across the two tracks.
- The motor interface remains torque-mode commanded over CAN, so VCU candidates implement their own current loop, top-speed governor, and drive-mode arbitration on top.

#### Test Runner and Reporting

- **Python + pytest** as the harness — same toolchain as the BMS firmware and testing tracks.
- Allure or pytest-html for reports, linked to raw bus captures and measurement data.
- Regression-suite execution against any VCU candidate, with pass/fail outcomes and trend reporting across runs.
- Test descriptions readable enough that a non-author engineer can debug a failure.

#### Logging and Telemetry

- **Full bus capture at line rate** on every segment, with timestamps and lossless storage.
- All sensor stimuli logged at their stimulation rate.
- All rig-internal state (load-bench setpoints, emulator state machines, fault-injection commands) logged at 100 Hz minimum.
- Data dropped into the same local store the BMS testing track uses (SQLite or local TimescaleDB), with a path to the monitoring stack.
- All logs queryable from the same Python harness used to author tests.

### Rig Safety

Cross-cutting; applies regardless of which layer is active and is built up from Layer 1.

- **E-stop** physically accessible at all operator positions, breaking high-current paths.
- **Current limits** on the load bench and the battery pack interface.
- **Thermal cutoffs** on the motor, controller, and battery emulation.
- **Isolation monitoring** on the high-current circuit.
- **Rig integrity watchdog** — the rig asserts its own sane state before allowing any DUT operation. A misbehaving rig stops itself before it stops a DUT.

### On-Vehicle Integration

The mockup track owns the bench-to-vehicle handoff. A VCU validated on the rig has a clear path onto a real vehicle, with the same Python harness running where the interfaces match.

- **Race team's vehicle** as the first real-vehicle integration target, once the rig validates a VCU candidate to readiness.
- **Vayve Mobility vehicles** as the next-stage integration targets.
- The same Python harness runs against the real vehicle where the interfaces match, so test continuity from bench to vehicle is preserved.
- Field findings flow back into the mockup specification as either bug fixes (mockup was wrong) or documented divergences (real vehicle is a special case).

## Phased Build Path

A rig of this scope is not built in one go. The phasing below follows the layered structure: each phase lands one layer (or a coherent slice of one) and gives the VCU design track something usable. The phasing is a **suggested decomposition only — the team owns the actual phase plan, decides what slots into which phase, and adjusts as priorities and constraints become clearer**. What matters is that the phasing exists, is documented, and is reviewed at the close of each phase.

### Phase 1 — Base Rig ("Hello, VCU")

Goal: Layer 1 standing up — a single-profile rig that can drive a VCU through a basic end-to-end test pass.

- Single vehicle profile (2W recommended as the simpler starting point).
- Single CAN segment, no zones yet.
- Real motor on the load bench, torque-commanded over CAN, with a placeholder controller emulating dyno load behaviour.
- BMS and motor-controller emulators at behaviour level, covering charge/discharge and torque execution basics.
- Placeholder VCU running through a simple drive cycle.
- Pedals or grip-twist throttle, ignition, drive-mode selector, dashboard SOC/speed.
- Python + pytest harness with one or two scenarios.
- Local logging.
- E-stop, current limits, basic safety interlocks.

### Phase 2 — Chassis Emulation Engine and Fault Injection

Goal: Layer 2 added — chassis-level coupling, vehicle dynamics, regen-blend, ABS behaviour — plus structured fault injection from Layer 4.

- Multi-segment CAN with simple gateway behaviour.
- Chassis ECU emulator with vehicle dynamics model (bicycle or motorcycle per profile), ABS, regen-blend negotiation, basic stability behaviour.
- Fault injection on bus and sensor axes built into the framework.
- Regression catalogue built out from the starter scenarios.

### Phase 3 — Full Peripheral, Sensor Build-Out, and Complete Test Capabilities

Goal: Layers 3 and 4 completed for the chosen profile.

- Zonal CAN architecture with documented zones, plus LIN buses for slow peripherals.
- Full ESC, traction control, EPS in the chassis emulator (4W); cornering ABS and CBS (2W).
- Full peripheral and switchgear emulation per profile.
- Full HVAC, comfort, and the simple sensor envelope (ultrasonic stimulation, IMU, GPS, environmental sensors).
- Full charging system emulation, thermal management sensors.
- Full auxiliary and system sensors (HV, 12 V, access, crash, service).
- Output verification instrumentation across all output classes.
- Fault injection across all five axes.

### Phase 4 — Second Vehicle Profile

Goal: the second profile reaches feature parity.

- The second profile (4W if 2W was first, or vice versa) built up through Phase 1–3 equivalents.
- If a single reconfigurable rig was chosen, the peripheral swap, motor swap, and harness reconfiguration paths are exercised and documented.
- If two parallel rigs were chosen, the second rig is built out.
- Cross-profile scenario library and harness coherence.

### Phase 5 — Real-Vehicle Integration

Goal: deliver on the bench-to-vehicle commitment.

- Adapters and harnesses for the race team's car.
- First integration of a mockup-validated VCU into the race car.
- Field findings flowing back into the mockup spec.
- Adapters and harness for the first Vayve product.
- Second integration of a mockup-validated VCU.
- Integration playbook for taking validated VCUs to new vehicle targets.

### Phase 6+ — Continuous Improvement (Optional)

Goal: the rig grows with the product line.

- Additional vehicle profiles or trims.
- New reference vehicles incorporated into the mockup spec.
- Coverage expansion for emerging features (V2X, ISO 15118-20, evolving regulatory requirements).
- Hardware refreshes as procurement opportunities arise.

## Technical Considerations to Own

- **Mockup specification as a versioned product** — change control, release notes, traceability between reference-vehicle data and mockup behaviour.
- **Reference vehicle selection** — which vehicle's specs anchor v1 of the mockup, and how subsequent reference vehicles are incorporated as their data becomes available.
- **Motor and dyno sizing** — the load-bench control motor needs to be higher-rated than the DUT motor to apply realistic loads without saturating; the team owns the analysis that justifies the sizing.
- **Backbone choice** — Automotive Ethernet vs CAN-FD vs hybrid — for the zonal architecture, evaluated against bandwidth, cost, and integration-target realities.
- **Zone controller implementation** — additional MCUs in dedicated zone boxes vs zone behaviour folded into a rig PC. Trade-off between fidelity and complexity.
- **Gateway / routing behaviour** — whether the VCU is expected to be the gateway, or whether a separate gateway component is in the architecture. The rig should support testing either model.
- **Sensor signal fidelity** — impedance, noise floor, levels, and how cleanly the rig fools the VCU's input stages. Production-grade VCUs have plausibility-checking input front-ends that a sloppy rig will trigger spuriously.
- **Vehicle dynamics model fidelity** for the chassis ECU emulator — bicycle model vs single-track vs higher-fidelity multi-body — chosen against what the VCU's control loops actually need to feel, the team's modelling capacity, and the available open-source or commercial libraries. The model is part of the mockup spec and is versioned alongside it.
- **Scenario description language** — Python is the harness, but how scenarios are *expressed* (helper functions, declarative DSL, pure-script) affects how readable and reviewable the catalogue stays as it grows.
- **Rig configuration management** — the rig has many interlocking parts (zones, emulators, motor sub-rig, sensor stimulators), and the configuration that matched yesterday's test must be reproducible today.
- **Fault-injection commanding** — how faults are described, scheduled, triggered, and cleared from inside a test case. Faults need to be first-class objects in the framework, not afterthoughts.
- **Real-vehicle integration tooling** — the harness, adapters, and instrumentation needed to take a mockup-validated VCU into a real vehicle and continue running scenarios against it. The Python test harness should run identically against the real vehicle where the interfaces match.

## Starting Points

- **Multi-channel CAN interfaces** (Kvaser, PEAK PCAN-USB X6, or open-source equivalents) for instrumenting each segment of the zonal layout.
- **Automotive Ethernet interfaces and switches** (Microchip LAN9662-class, Marvell 88Q5050-class, or vendor evaluation hardware) for the zonal backbone if Ethernet is chosen.
- **Zone controllers** built on STM32 or similar MCUs — same family as the BMS and VCU design choices, so the team's toolchain stays coherent.
- **STM32**-class development board as the placeholder VCU.
- **Reference vehicle data** anchored to:
  - **4W (full passenger car) profile**: Tata Nexon EV, Tata Curvv EV, Tata Punch EV, Mahindra XUV400 / BE-series, MG ZS EV, or similar Indian-market full passenger cars. The team picks based on accessibility and publicly available documentation. Vayve EVA/CT4 and the race team's car remain integration targets, treated as configuration subsets of the full-car profile.
  - **2W profile**: an Indian-market reference such as Ather 450X, Ola S1 Pro, Bajaj Chetak, or TVS iQube.
- The process for incorporating Vayve product data and race-car-specific data is documented as a follow-up activity.
- **Python + pytest + Allure or pytest-html** as the harness and reporting stack — same as the BMS firmware and testing tracks.
- **Off-the-shelf load-bench components** sized to the chosen reference motor, with the dyno control motor selected based on the load-profile analysis.
- **Vehicle dynamics modelling references** for the chassis ECU emulator — open-source options (CommonRoad-Vehicle-Models, BeamNG/Speed Dreams physics, OpenScenario-aligned tooling) or commercial libraries (IPG CarMaker, dSPACE ASM, Simulink Vehicle Dynamics Blockset) — evaluated against fidelity needs and team capacity. A simple custom bicycle model is also an acceptable starting point.

## Deliverables

- A working bench mockup covering **both vehicle profiles (2-wheeler and 4-wheeler)** — single reconfigurable rig or two parallel rigs per the team's decision — with multi-segment zonal CAN architecture, behaviour-level BMS/MCU/dashboard emulation, message-level peripheral ECU emulation, real pedals or grip-twist throttle per profile, real peripheral switchgear (lights, dashboard, steering stalk or handlebar switches, etc.), real motor on the load bench, data-emulator alternative path, and full sensor-stimulation hardware.
- **Mockup specification** as a versioned, source-controlled artifact, with change control and release notes.
- **Reference vehicle analysis** documents — what specs and behaviours from real vehicles inform the mockup, what assumptions are made, what's still open.
- **Emulator firmware/software stacks** for each emulated ECU, with documented message behaviour and fault behaviour. The chassis ECU emulator is a sub-deliverable in its own right, with vehicle-dynamics-model documentation and its own test report.
- **Motor sub-rig** with documented characterisation — load envelope, response time, accuracy, and known limitations.
- **Scriptable test framework** (Python + pytest) with the starter scenario catalogue and a documented scenario-authoring guide.
- **Fault-injection catalogue** covering all five axes, exhaustively mapped to documented VCU fault responses.
- **Placeholder VCU** build that passes the rig's full test suite, demonstrating end-to-end correctness.
- **Test reports** for every VCU candidate that runs through the rig, with regression trends across runs.
- **On-vehicle integration** of a mockup-validated VCU into the race team's car, with field findings documented and flowing back into the mockup spec.
- **Integration playbook** for taking a mockup-validated VCU to a new real-vehicle target (race car, Vayve, future).

## Out of Scope

- Designing the production VCU hardware, firmware, or enclosure (covered by the VCU design track).
- Designing the actual vehicle (the race car and Vayve teams' domain) — though the mockup tracks their architecture and the integration goes onto their vehicles.
- BMS firmware development (covered by the BMS firmware track) — though this track produces the behaviour-level BMS emulator used in the rig.

## Inter-Track Dependencies

- **VCU design track** — primary consumer of this rig; co-authors scenarios; eventually delivers VCU candidates for validation here before any vehicle deployment.
- **BMS firmware track** — provides the real BMS that will be plugged into the rig; the BMS emulator in the rig is the behavioural shadow of the real BMS.
- **Vehicle monitoring track** — can use the rig as a data source while real vehicles are being built; eventually ingests rig and real-vehicle telemetry into the same monitoring stack.
- **Race team and Vayve vehicle teams** — provide the reference vehicles, the architecture data feeding the mockup spec, and the real-vehicle targets for integration.

## Future Add-Ons

### Perception Stimulation

When perception sensors are added to the VCU design as an extension, the mockup adds the corresponding stream-stimulation capability: camera and radar streams over Automotive Ethernet (synthetic generation and recorded-and-replayed real-world data), LiDAR streams where the target vehicle supports them, surround-view and driver-monitoring camera coverage, and stream-level fault injection (frame drops, corruption, timing jitter, link loss, bandwidth starvation). ADAS-related driver-engagement signals — hands-on-wheel capacitive sensing, drowsiness, attention — sit alongside this expansion.
