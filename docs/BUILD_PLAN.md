# ADS Version 1 — Hardware and Software Build Plan

**Project:** AssayFlux Diagnostic Systems  
**Document status:** Engineering plan; not yet implemented or validated  
**Target:** Pocket-size, manual, cuvette-based multispectral photometer

## 1. Version 1 design baseline

This document records the current agreed design baseline. Changes should be made through documented engineering decisions rather than silently changing the hardware scope.

| Subsystem | Version 1 selection |
|---|---|
| Main controller | ESP32; exact module/board and pinout to be confirmed |
| Optical detector | ams OSRAM AS7343 only |
| Illumination | One tungsten-halogen lamp, subject to optical, thermal and power feasibility testing |
| Sample interface | Manual cuvette insertion in a fixed, repeatable holder |
| Optical path | Transmitted-light measurement through the cuvette |
| Power/connectivity | USB-C, onboard voltage conversion, and selected ESP32 Bluetooth/Wi-Fi capability |
| User interface | Mobile application |
| Status indication | RGB status LED(s) |
| Reference photodiode | Not included in Version 1 |
| Temperature sensor | Not included in Version 1 |
| AS7341 | Not included in Version 1 |
| External wavelength filters | Not included initially; add only if testing demonstrates a specific need |
| Grating spectrometer | Not included |

**Scope limitation:** This is a multispectral photometer, not a continuous-spectrum spectrometer. A broad-spectrum lamp does not add detector channels. The AS7343 can only measure within its supported spectral response. This design must not claim validated 260 nm, 280 nm or 340 nm measurements. Standard electrolyte measurements also require a different measurement principle.

No assay is clinically validated by this document. The initial goal is a research/prototype instrument and a controlled assay-development workflow.

## 2. System architecture

The optical/electronic signal chain is:

1. Regulated tungsten-halogen lamp.
2. Light-tight optical chamber and fixed aperture/beam geometry.
3. Cuvette containing blank, standard, control or patient sample.
4. AS7343 receiving transmitted light.
5. ESP32 acquiring sensor data and applying the selected measurement protocol.
6. Mobile application presenting the workflow, calibration state, result and quality flags.

The lamp, cuvette and sensor must remain mechanically aligned. The cuvette insertion stop and orientation must be repeatable. The optical chamber must exclude room light.

## 3. Hardware work plan

### 3.1 Optical bench prototype

Build and test the optical bench before designing the final pocket enclosure.

- Start with an accessible external tungsten-halogen lamp supply so the optical geometry can be tested without committing to a miniature bulb.
- Select a lamp only after measuring its useful output across the AS7343 channels, electrical power, warm-up behaviour and heat generation.
- Use a fixed cuvette holder with a defined path length and insertion stop.
- Align the lamp, cuvette and AS7343 in a straight transmitted-light path.
- Use a light-tight enclosure, internal matte-black surfaces and an aperture to reduce stray light.
- Do not add external optical filters in the first prototype. Consider a filter only to solve a measured problem such as unwanted spectral leakage or excessive infrared/stray-light contribution.
- Record the exact cuvette type, path length, lamp-to-cuvette distance, cuvette-to-sensor distance and aperture dimensions.

**Important lamp caveat:** Tungsten-halogen lamps emit substantial infrared energy and produce heat. An ordinary lamp envelope must not be assumed to provide useful UV output. Neither a broad-spectrum lamp nor software calibration makes the AS7343 a validated UV detector. The sensor's near-infrared response and other channels must be checked against the actual lamp spectrum and assay chemistry.

### 3.2 Lamp electrical drive

- Use a dedicated lamp driver/supply rated for the selected lamp's nominal voltage and cold-filament inrush current.
- Do not drive the lamp from an ESP32 GPIO or directly from a sensor supply rail.
- Confirm whether the lamp needs constant-voltage operation or another manufacturer-specified drive method; do not assume a generic LED constant-current driver is appropriate for a tungsten filament.
- Measure lamp voltage/current and warm-up/stabilisation behaviour during development.
- Keep the lamp supply electrically and thermally separated from the sensor and ESP32.
- Add appropriate fuse/current protection, wiring gauge, connector ratings and strain relief.
- Test worst-case enclosure temperature with external calibrated test equipment during engineering validation. A temperature sensor is not part of the current product baseline.

### 3.3 AS7343 and ESP32 electronics

- Use the AS7343 datasheet to confirm supply voltage, logic levels, I2C timing, address, channel configuration, integration-time and gain limits before schematic capture.
- Connect the AS7343 to an ESP32 I2C controller using appropriate pull-ups and logic-level compatibility.
- There is only one AS7343 in this design, so there is no need for an I2C multiplexer for sensor-address conflict.
- Keep sensor power clean and route lamp current away from the sensor supply and I2C lines.
- Add local decoupling capacitors close to the sensor and controller as required by their datasheets.
- Keep USB-C power input, battery/charging design (if a battery is selected), voltage converters and lamp supply within their rated limits.
- Confirm the exact ESP32 variant before assigning pins or finalising the PCB.

### 3.4 Mechanical enclosure

Use three functional zones:

1. **Optical zone:** lamp, aperture, cuvette holder and AS7343. Light-tight, aligned and thermally isolated.
2. **Electronics zone:** ESP32, AS7343 interface, power conversion and status LEDs.
3. **Power zone:** USB-C input and optional battery/charging circuitry, separated from the hot lamp area.

The enclosure design must allow cuvette insertion/removal, cleaning, access for service and safe heat dissipation. Do not freeze enclosure dimensions until the selected lamp, cuvette and optical bench have been tested.

### 3.5 Hardware deliverables

- Block diagram and annotated optical-path drawing.
- Schematic and bill of materials (BOM).
- Lamp-driver design and power budget.
- AS7343-to-ESP32 wiring/pin map.
- Optical chamber and cuvette-holder CAD.
- PCB layout and assembly notes.
- Bench test report for light level, channel response, repeatability, power and thermal behaviour.

## 4. Firmware plan

Recommended implementation: ESP-IDF with C/C++ unless the existing project or selected ESP32 board gives a strong reason to use another supported framework. Keep hardware drivers, measurement logic and communication protocol separate.

### 4.1 Firmware modules

- as7343_driver: device initialisation, channel configuration, integration time, gain and raw data acquisition.
- lamp_control: safe on/off control and warm-up/stabilisation sequence.
- measurement: dark, blank, sample and repeat measurements.
- absorbance: dark-corrected transmittance and absorbance calculations.
- calibration: standard collection, curve fitting, model storage and expiry/version metadata.
- assay_profiles: assay-specific channel/settings, reagent method, timing, units and reportable range.
- quality_control: saturation, low signal, invalid blank, out-of-range result and QC checks.
- communications: versioned BLE/Wi-Fi message protocol to the mobile app.
- storage: configuration, calibration records, device identity and audit events.

### 4.2 Measurement sequence

For each measurement session:

1. Confirm that the cuvette is inserted and the selected assay profile is available.
2. Turn the lamp off and collect dark readings. The enclosure must be closed so ambient light does not contaminate this reading.
3. Turn the lamp on and wait for the empirically established stabilisation interval.
4. Measure the reagent blank using the same cuvette type, orientation, lamp settings and sensor settings as the sample.
5. Measure the sample using the same optical configuration.
6. Repeat readings when required by the assay profile and calculate a robust summary.
7. Apply dark correction and calculate transmittance/absorbance.
8. Apply the assay's validated calibration model and dilution factor.
9. Check analytical range, signal quality and QC status before displaying a result.

For blank intensity Ib, sample intensity Is and dark intensity Id, calculate:

T = (Is - Id) / (Ib - Id)

A = -log10(T)

Reject the measurement if the denominator is non-positive, the sample signal is invalid, the signal is saturated, or transmittance is outside the validated method range. Dark/blank correction does not fully remove lamp drift occurring between readings; lamp stability and measurement timing must therefore be characterised.

### 4.3 Calibration model

Each assay profile must define its own calibration method. Do not assume every assay is linear. Support only models that have been justified by the assay method and validation data, such as linear regression over a verified linear range.

A calibration record should include:

- Assay and reagent-method identifier.
- Reagent manufacturer, kit/catalogue identifier and lot where available.
- Sensor channel(s), gain, integration time and lamp configuration.
- Blank and dark readings.
- Standard concentrations, raw readings and calculated absorbances.
- Model type, coefficients, fitting range and fit diagnostics.
- Units, dilution factor and reportable range.
- Calibration date, operator/device identifier and expiry/recalibration rules.
- QC material, target range and QC results.

Do not transfer calibration coefficients from another device, sensor, lamp, cuvette type, reagent lot or optical assembly without a documented equivalence study.

## 5. Mobile application plan

The app should guide the operator through a defined workflow rather than merely display raw sensor values.

### 5.1 Core screens

1. **Device connection:** connect to the instrument, show device/firmware version and connection state.
2. **Assay selection:** show only assay profiles installed and enabled for that device.
3. **Reagent/setup:** show required reagent, sample type, volume, incubation and timing instructions sourced from the selected method.
4. **Calibration:** guide the operator through blank and standards, show curve/fit diagnostics, and save a versioned calibration.
5. **Sample measurement:** show the measurement steps and progress; prevent result release when required steps fail.
6. **Result:** display concentration, unit, reportable range, flags, assay/method and calibration status.
7. **QC and history:** show QC status, prior measurements, calibration history and export options.
8. **Device settings:** lamp diagnostics, sensor diagnostics, connectivity and firmware version.

### 5.2 App implementation

The repository has not yet established an app framework. Evaluate Flutter as a cross-platform candidate after confirming target Android/iOS requirements. Keep the device protocol independent of the UI framework.

Use a versioned, documented BLE GATT or Wi-Fi protocol. Messages should carry a schema/protocol version, request identifier, measurement identifier, status and units. The app must not invent missing measurements or silently reinterpret units.

Store records locally for offline operation. If cloud sync is introduced later, define consent, security, retention and access controls separately. Do not put patient-identifying information into device debug logs.

## 6. Assay development strategy

Begin with a small visible colorimetric assay set. Candidate methods for feasibility testing include glucose GOD-POD, total cholesterol CHOD-PAP, triglycerides GPO-POD, uric acid, total protein Biuret, albumin BCG, urea Berthelot, creatinine Jaffe, bilirubin, calcium and selected phosphorus methods.

These are candidates only. Actual reagent instructions, reaction timing, sample type, wavelength requirements and clinical reference methods must be obtained from the specific reagent-kit documentation.

The selected AS7343 channel must be chosen from measured performance with the actual reagent and sample matrix, not merely by choosing the nearest nominal channel wavelength. The channel's spectral bandwidth and overlap matter.

Do not include standard 340 nm NADH kinetic AST/ALT methods, UV 260/280 nm methods, or direct sodium/potassium/chloride electrolyte measurements in the initial supported menu. Alternate assay chemistries may be investigated as separate development projects, but they require their own evidence and validation.

## 7. Verification and validation plan

### 7.1 Instrument verification

- Dark signal and dark repeatability.
- Blank stability over the intended measurement interval.
- Lamp warm-up and drift.
- Channel response and saturation limits.
- Short-term repeatability and between-day reproducibility.
- Cuvette repositioning/orientation effects.
- Stray-light and ambient-light susceptibility.
- Power consumption and battery runtime, if battery-powered.
- Thermal safety of the enclosure and sample region.
- Communication loss/recovery and stored-data integrity.

### 7.2 Assay validation

For each assay separately:

- Confirm the reagent method and intended specimen type.
- Establish calibration model and reportable measuring interval.
- Evaluate precision, linearity, analytical measurement range, recovery and interference.
- Evaluate carryover/contamination risks relevant to the manual workflow.
- Compare results against a suitable reference method/analyser using an appropriate sample set.
- Define QC materials, acceptance limits and recalibration triggers.
- Document limitations and clinically relevant flags.

Do not describe an assay as supported, accurate or clinically validated until the relevant evidence has been reviewed and recorded. Regulatory and quality-system requirements must be assessed before clinical deployment.

## 8. Build phases and exit criteria

| Phase | Work | Exit criteria |
|---|---|---|
| 0. Requirements freeze | Confirm ESP32 variant, cuvette, sample types, power source, size target and app platforms | Approved requirements and unresolved-items list |
| 1. Optical bench | Test tungsten-halogen source, AS7343 and cuvette geometry | Adequate unsaturated signal, repeatable blank/sample readings, acceptable measured heat/power |
| 2. Electronics prototype | ESP32, AS7343, lamp drive, USB-C/power and status | Stable acquisition and safe power operation on bench |
| 3. Firmware alpha | Driver, dark/blank/sample, absorbance, profiles and diagnostics | Reproducible raw data and unit tests for calculations |
| 4. App alpha | Pairing, assay workflow, calibration, result and history | Reliable end-to-end device/app communication |
| 5. Assay feasibility | Test a small number of selected reagent methods | Documented feasibility data; no unsupported clinical claims |
| 6. Enclosure/PCB | Integrate only after optical bench passes | Mechanical, optical, electrical and thermal tests pass |
| 7. Verification/validation | Instrument and assay performance studies | Approved reports and controlled supported-assay list |

## 9. Initial repository structure proposal

The following is a proposed structure, not a claim that these components already exist:

- README.md
- docs/BUILD_PLAN.md
- docs/decisions/
- docs/hardware/
- docs/firmware/
- docs/app/
- docs/validation/
- hardware/electronics/
- hardware/optics/
- hardware/mechanical/
- hardware/bom/
- firmware/components/
- firmware/main/
- firmware/test/
- app/
- protocols/
- tests/

Add implementation files only when their design, dependencies and acceptance tests are defined.

## 10. Open engineering decisions

The following must be resolved with measurements or confirmed requirements:

- Exact ESP32 module/board and available I2C/GPIO pins.
- Exact AS7343 package or breakout/PCB implementation and supply/logic levels.
- Tungsten-halogen lamp part number, rated voltage, current, spectrum, envelope and thermal requirements.
- Whether the first prototype uses an external lamp supply and what the final power source will be.
- Cuvette type and optical path length.
- Target external dimensions and mass.
- Mobile app target platforms and preferred framework.
- Initial reagent kits and specimen types.

## 11. Version 1 non-goals

- No AS7341.
- No second UV/visible LED arrangement.
- No reference photodiode.
- No temperature sensor.
- No generic external optical filter wheel.
- No miniature grating spectrometer.
- No claim to measure every routine laboratory assay.
- No clinical-use or regulatory-compliance claim before appropriate validation.

## 12. Engineering change control

Any change to the detector, lamp, cuvette, optical geometry, power supply, firmware measurement settings or assay method can affect calibration. Record the change, identify affected tests, repeat the relevant verification and recalibrate/validate before treating old calibration records as equivalent.
