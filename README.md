# AssayFlux Diagnostic Systems (ADS)

Pocket-size, manual, cuvette-based multispectral photometer development project.

## Current Version 1 design baseline

- ESP32 controller
- One AS7343 multispectral sensor
- One tungsten-halogen lamp, pending optical, thermal and power feasibility testing
- Fixed manual cuvette holder and light-tight transmitted-light optical chamber
- USB-C and onboard power conversion
- Mobile application for assay selection, calibration, measurements and results
- RGB status indication

The Version 1 baseline does not include AS7341, a reference photodiode, a temperature sensor, external wavelength filters or a grating spectrometer.

## Engineering plan

The hardware, firmware, mobile application, assay-development and validation roadmap is documented in [docs/BUILD_PLAN.md](docs/BUILD_PLAN.md).

The repository is currently documentation-first. A planned subsystem is not considered implemented, bench-tested, analytically validated or suitable for clinical use until its evidence and test records are added.

## Important measurement limitations

ADS Version 1 is a multispectral photometer, not a continuous-spectrum spectrometer. A broad-spectrum lamp does not expand the AS7343's supported detector response. The current architecture must not claim validated 260 nm, 280 nm or 340 nm measurements, nor direct electrolyte measurement. Candidate colorimetric assays remain development targets until individually validated with their specified reagent methods.

## Development approach

1. Validate the optical bench and lamp before committing to a pocket enclosure.
2. Confirm the exact ESP32 board, AS7343 electrical requirements, lamp and cuvette.
3. Implement and test firmware measurement primitives.
4. Implement the mobile workflow and versioned device protocol.
5. Validate each assay separately against appropriate standards and reference methods.
6. Integrate the PCB and enclosure only after the optical and thermal design passes.

No clinical performance, assay menu or regulatory status is claimed at this stage.
