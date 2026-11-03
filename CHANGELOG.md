# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.4.1] - 2026-11-02

Built on CAS BACnet Stack 6.x (`1e7dffe2b`).

### Changed

- **CAS BACnet Stack updated** to 6.x `1e7dffe2b` (still version 6.0.23).
- **An I-Am also reaches a client on another UDP port.** At this stack
  version, a broadcast the stack sends without a destination address, such as
  the I-Am that answers a Who-Is, or an I-Have, is addressed by the example. It
  still goes to the local subnet broadcast address, and is now also sent
  unicast to the last client heard from a UDP port other than the example's own,
  so a tool on the same computer on another port hears the answer. Only the
  messages the stack addresses itself (network layer messages, I-Am, I-Have)
  are copied. Vendored `common/` updated to 3.6.1; see `common/CHANGELOG.md`.
- **README**: the profile example table is the same in every example; the
  blocked examples are marked with their open stack issue.

## [1.4.0] - 2026-10-27

Built on CAS BACnet Stack 6.x (`69b6249f`).

### Added

- **`--deviceName <text>`** sets the Device's Object_Name at start-up, like
  `--deviceID` sets its instance (ASHRAE 135-2024 cl. 12.1.1.3 and 12.1.1.4).
  Object_Name must be unique on the internetwork, so two units can now be
  installed together. The default is unchanged (`Chipkin Example B-SA`); a name
  must be 1 to 255 bytes with no control characters, otherwise a warning is
  printed and the default is kept. A WriteProperty of the Device's Object_Name
  is still refused.
- **`--xml`** prints each received and sent frame as an XML block.

### Changed

- **`--help` lists only the options the example reads.** It no longer lists
  `--dcc-password`, because a B-SA has no DeviceCommunicationControl.
- **Test scripts** under `tests/` default to this example's own port, 47826.
- Vendored `common/` updated to 3.5.0; see `common/CHANGELOG.md`.
- **Documentation**: the PICS and README follow the current stack version.

### Fixed

- **PICS: Database_Revision** is documented as 8, the value the device
  actually reports.
- **Requests no longer go unanswered on Windows**: a request arriving just as
  a receive timed out could be silently discarded (vendored `common/` 3.4.1).

## [1.3.0] - 2026-10-16

Built on CAS BACnet Stack 6.x (`64b2cfac5`, 6.0.23).

The actuator now performs the B-SA Profile Function (ANSI/ASHRAE 135-2024
Annex L.4.4): it acts on the winning command. The BACnet surface is unchanged
(DS-RP-B, DS-WP-B, DM-DDB-B, DM-DOB-B; the same eight objects). This release
also rolls up the earlier 1.2.1 through 1.2.3 work, which was never released
separately.

### Added

- **Actuator simulation**: once a second a heating valve moves the temperature
  (Analog Input 1) up to 1.0 C toward the winning Analog Output 1 setpoint;
  Binary Input 1 (relay feedback) and Multi-State Input 1 (drive speed) follow
  the winning Binary and Multi-State Output 1 commands (cl. 19.2). Before, the
  commands were stored and nothing used them.
- **Analog Output 1 `Min_Pres_Value` 10.0 and `Max_Pres_Value` 30.0**: a
  setpoint outside the range is refused with `value-out-of-range` and not
  stored. Before, any REAL was accepted.
- **Analog Output 1 `Reliability`**: `no-fault-detected`, or `no-output` while
  the simulated valve actuator is disconnected; the FAULT bit of `Status_Flags`
  follows (cl. 12.3.7, 12.3.9). Commands are still accepted while disconnected.
- **Automated self-tests** under `tests/`: object discovery, command priority
  and relinquish, out-of-range writes, a disconnected valve, and refused writes
  to inputs.

### Changed

- **Segmentation is supported** (`segmented-both`, 16 segments, 5000 ms segment
  timeout); `docs/PICS.md` updated.
- **Protocol_Revision** is 30.
- **Device renamed** from "Rainbow" to "Chipkin Example B-SA" so examples from
  this series can be told apart on one network.
- **Build** uses the stack's source mode by default
  (`cmake -B build -S .` then `cmake --build build --config Release`); no
  prebuilt static library step is needed.
- **Documentation** restructured into README, `TUTORIAL.md` and `docs/PICS.md`.
  The README now states what the outputs actually do and quotes Annex L.4.4
  correctly.
- The up and down keys disturb the simulated temperature; the valve brings it
  back.
- Vendored `common/` updated to 3.4.0 (a broadcast addressed to the socket's
  own address now goes to the subnet broadcast).
- Application_Software_Version is 1.3.0.

### Fixed

- **`Application_Software_Version` and `Firmware_Revision`** were hard-coded
  and stale; they now follow the example and stack versions.
- **README start-up sample** matches the real output (it omitted the
  `(Network Port 1)` suffix).

## [1.2.0] - 2026-09-15

Built on CAS BACnet Stack 6.x (`abd4cee1`, 6.0.21).

### Changed

- **Moved to CAS BACnet Stack 6.x**, linked through the
  `CASBACnetStack::Adapter` CMake target. `CAS_BACNET_STACK_LINK` selects
  `SOURCE`, `STATIC` or `DLL`; this release was built and documented against a
  prebuilt static library.
- **Stack callback signatures changed**: every `GetProperty*` callback gains a
  trailing `uint32_t* errorCode` out-parameter, and
  `BACnetStack_AddNetworkPortObjectWithNetworkNumber` was replaced by
  `BACnetStack_AddNetworkPortObject`. Update your own code if you copied from
  an earlier version.
- **Default device instance** is 389002 (was 389001) so several examples can
  run on one subnet. Override with `--deviceID`.
- Vendored `common/` updated; it carries its own version and changelog
  (`common/CHANGELOG.md`).
- **Documentation**: added the generated "Objects and properties" reference and
  a footprint table to the README; corrected the sample output and
  troubleshooting notes.
- This example is the reference for the series' commandable-output pattern
  (Analog Output 1, Binary Output 1, Multi-State Output 1).

## [1.0.0] - 2026-06-16

### Added

- **Initial B-SA (BACnet Smart Actuator) example** for the CAS BACnet Stack in
  C++: DS-RP-B, DS-WP-B, automatic Who-Is / I-Am, and an I-Am broadcast at
  start-up.
- **Device** object with its full identity (default instance 389001, vendor
  389).
- **Read-only inputs**: Analog Input 1, Binary Input 1, Multi-State Input 1.
- **Commandable outputs**: Analog Output 1, Binary Output 1, Multi-State
  Output 1, each with a 16-slot Priority_Array and Relinquish_Default.
  Writing NULL relinquishes a slot; writes to inputs and out-of-range output
  values are refused.
- **Network Port 1** with full BACnet/IP addressing.
- **Command-line options** `--port <n>` and `--deviceID <n>`; interactive keys
  `h`, `q`, and up/down to nudge Analog Input 1.
- Cross-platform CMake build that compiles the stack from source, with the
  stack included as a git submodule (clone with `--recursive`).

[1.4.1]: https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases/tag/v1.4.1
[1.3.0]: https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases/tag/v1.3.0
[1.2.0]: https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases/tag/v1.2.0
[1.0.0]: https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases/tag/v1.0.0
