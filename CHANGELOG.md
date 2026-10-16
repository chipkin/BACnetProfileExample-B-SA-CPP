# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.0] - unreleased

Profile-functional (series bar): the actuator now performs the B-SA Profile
Function - ANSI/ASHRAE 135-2024 Annex L.4.4, "Ability to allow modification of
some or all of its BACnet objects by another device" - by doing what the winning
command says, and shows Failure Modes. The BACnet Surface is unchanged (DS-RP-B,
DS-WP-B, DM-DDB-B, DM-DOB-B; the same eight objects).

### Added

- **Actuator Simulation** (`main.cpp` section 1b, `ActuatorSimulationTick()`,
  labelled Simulation in the README): once a second a heating valve moves the
  temperature (Analog Input 1) up to 1.0 C toward the winning Analog Output 1
  setpoint; Binary Input 1 (the relay's feedback contact) and Multi-State Input 1
  (the drive's speed) take the winning Binary / Multi-State Output 1 command.
  `ResolvedCommand()` gives the winning command (cl. 19.2). Before, commands were
  stored in the Priority_Array and nothing used them; the inputs were constants.
- **Analog Output 1 `Min_Pres_Value` 10.0 / `Max_Pres_Value` 30.0** (optional,
  now enabled); a setpoint outside them gets Error `property` /
  `value-out-of-range` and is not stored. Before, any REAL was accepted.
- **Analog Output 1 `Reliability`** (optional, now enabled): `no-fault-detected`,
  or `no-output` while the simulated valve actuator is disconnected; the stack
  derives the FAULT bit of `Status_Flags` from it (cl. 12.3.7, 12.3.9). Commands
  are still accepted while disconnected and obeyed once reconnected. Disconnected
  only through a test-only hook: the environment variable
  `CAS_EXAMPLE_TEST_ACTUATOR_FAULT_FILE` names a file, and the valve is
  disconnected while that file exists.
- **Wire Tests** `tests/commanding_test.py` (each output at two priorities: the
  lower-priority command changes nothing, relinquishing hands over, the feedback
  input follows the winner) and `tests/failure_modes_test.py` (disconnected and
  reconnected valve; out-of-range writes to all three outputs; a write to an
  input gets `write-access-denied`), with `tests/bsa_setup.py`. Every check is
  mutation-verified (tests/README.md).

### Changed

- The up/down keys disturb the simulated temperature; the valve brings it back.
- README: the scenario, the Simulation, the Wire Test table, Verify with the
  feedback; TUTORIAL: driving real outputs, reporting an output fault, refusing
  a value; PICS: the Analog Output optional properties, the objects' notes,
  Application_Software_Version 1.3.0.
- Wire Tests use the shared `wiretest/` 0.7.0: the Peer is `wiretest.Client(retries=3, timeout=0.6)` (the short timeout with visible `[RETRY]`s is now a Client option).

## [1.2.3] - unreleased

### Changed

- **Stack Pin moved to `6.x` @ `64b2cfac5` (CAS BACnet Stack 6.0.23)** from
  `6.x` @ `abd4cee1` (6.0.21), per the series rule that a pin moves to the latest
  `6.x` whenever an example is updated. No `main.cpp` change was needed (only
  APP_VERSION 1.2.2 -> 1.2.3 and a comment).
- **`common/` 3.0.0 -> 3.3.0**, copied verbatim from B-SS-CPP `b640a81`. Additive
  for this example: four new keys it does not handle (`k`, `o`, `f`, `p` are
  ignored) and `HelperSendMessage` now sends a broadcast the stack hands it with
  no address.
- **Protocol_Revision 24 -> 30** (the stack's default at the new pin): README,
  AGENTS.md, PICS, `main.cpp` comment.
- **Segmentation is now supported** (`segmented-both`, `Max_Segments_Accepted`
  16, `APDU_Segment_Timeout` 5000 ms): the stack's customer build compiles
  segmentation in from this pin (stack #2992). `docs/PICS.md` section 5 updated;
  section 11 regenerated.
- README, PICS and version lines: stack 6.0.23, `common/` 3.3.0,
  Protocol_Revision 30, example 1.2.3 (the README still said 1.2.0 and
  `common/` 2.5.0). Sample output shows the `- Unconfirmed: I-Am` suffix
  `common/` prints. Series profile table re-synced.
- **`common/` 3.3.0 -> 3.4.0**, copied verbatim from B-SS-CPP `5d78299`:
  `HelperSendMessage` now sends a broadcast the stack addresses to this device's
  own socket to the subnet broadcast, instead of unicasting it back to itself
  (stack #3933). No change in this example's Wire Test results.
- **`wiretest/` 0.5.0 -> 0.6.0**, copied verbatim from B-LS-CPP. Backward
  compatible: it adds codec helpers and the `peer`, `alarm`, `workstation` and
  `wait` modules this example does not use yet.
- **The discovery Wire Test retries its Who-Is once**, printing a visible
  `[RETRY]` line, when no I-Am comes back in 3 s. Who-Is is unconfirmed, so
  nothing else retries it, and the example's UDP receive can drop a datagram on
  Windows.

### Fixed

- **README misquoted Annex L.4.4.** It quoted the standard as "a simple
  actuating device with very limited resources"; 135-2024 L.4.4 reads "A B-SA
  is a simple control device with limited resources; it is intended for
  specific applications."
- **README said the outputs are "driven by a controller"** as if something acted
  on the command. It now says what happens: the write lands in the
  `Priority_Array`, the stack resolves `Present_Value`, a client reads it back,
  and nothing in the example consumes the commanded value yet.

### Added

- **Wire Tests** (`tests/`, README "Wire Tests"): `tests/discovery_test.py`
  (copied from B-SS-CPP `da3e96f`, object list adapted) does what a client's
  discovery does - Who-Is, the Device's `Object_List` (control: exactly the eight
  objects), then every object's `Property_List` and a ReadProperty of every listed
  property - and fails on any Error, Reject, Abort or silence.
  `tests/run_wire_tests.py` runs it; CI runs it on Linux after the smoke test.
  Passes 14/14 (114 reads) on Windows and Linux at the new pin.
- `wiretest/` 0.5.0, the series' shared Wiretest Harness, vendored verbatim
  from B-BBMD-CPP.

## [1.2.2] - unreleased

### Changed

- **Device renamed from the series' colour placeholder "Rainbow" to "Chipkin
  Example B-SA"** so devices from different examples in the series are
  distinguishable from each other on the same BACnet network - every example
  previously announced the identical Object_Name "Rainbow", which made two
  examples on one subnet indistinguishable by name. Sub-object names (Analog
  Input 1 "Bronze", etc.) are unchanged - only the Device object's name
  changed. `docs/colour-table.md` (series root) updated to match. APP_VERSION
  bumped 1.2.1 -> 1.2.2.

## [1.2.1] - unreleased

### Fixed

- **`Application_Software_Version` (12) and `Firmware_Revision` (44) were
  hardcoded and stale.** `Application_Software_Version` now reads
  `APP_VERSION` directly (one source of truth, can't drift from
  `--version`'s own banner again). `Firmware_Revision` is now built at
  runtime from the CAS BACnet Stack's own `BACnetStack_GetAPIMajorVersion()`/
  `GetAPIMinorVersion()`/`GetAPIPatchVersion()`/`GetAPIBuildVersion()` (the
  same 4 calls `common/CASExampleHelper.cpp`'s `PrintVersion()` already uses
  for the startup banner), populated once right after `LoadBACnetFunctions()`
  succeeds. Same fix as `BACnetProfileExample-B-SCHUB-CPP` v1.1.13, applied
  series-wide after a real device read via BACnet Explorer found the stale
  values.

### Changed

- **Documentation restructured into README + TUTORIAL + PICS**, matching the
  series pattern established in `BACnetProfileExample-B-SS-CPP`: `README.md` is
  cut down to what this example is and how to build/run/verify it (711 -> 402
  lines); the "Extending the example" recipes, the "Who serves what" table and
  the Troubleshooting table moved to the new **TUTORIAL.md**; a new
  **`docs/PICS.md`** (ANSI/ASHRAE 135 Annex A shape) replaces the README's old
  "Objects and properties" section, with its generated table now covering the
  **Device** object too (`docs/objects.json` gained a `Device` entry - the
  generated tables previously omitted it). Regenerating with
  `tools/gen-objects-properties.py BACnetProfileExample-B-SA-CPP` produces zero
  ⚠ rows.
- **Build switched from a prebuilt STATIC library to the adapter's default
  SOURCE mode**: `cmake -B build -S .` + `cmake --build build --config Release`
  is now the full, single-command-pair build on every platform, matching what
  every other restructured example in the series documents. Dropped
  `tools/build-stack-static.sh` and the `-DCAS_BACNET_STACK_LINK=STATIC` flag
  from `README.md` and `AGENTS.md`; `.github/workflows/release.yml` no longer
  builds or caches a static library, asserts `CAS_BACNET_STACK_LINK=SOURCE`
  instead of `STATIC`, records `"link_mode": "SOURCE"` in the published
  metrics, and now packages `TUTORIAL.md` and `docs/PICS.md` alongside the
  binary. The `## Footprint` table still shows the v1.2.0 STATIC-build numbers;
  the next tagged release refreshes them under the SOURCE build.
- The `CHANGE ALL OF THIS BEFORE YOU SHIP` block in `main.cpp` now carries the
  per-field guidance that used to live only in the README's "Before you ship"
  table, including the `DEVICE_NAME` uniqueness warning, so the checklist can't
  be skipped by someone who only reads the code.
- Corrected the README's "Expected output" sample, which was stale: the real
  start-up lines include a `(Network Port 1)` suffix
  (`FYI: Listening for BACnet/IP on UDP port 47808 (Network Port 1).` /
  `TX ... (broadcast) (Network Port 1)`) that the previous sample omitted.

## [1.2.0] - 2026-09-15

### Changed

- **CAS BACnet Stack re-pinned to `6.x` @ `abd4cee1` (reports 6.0.21)** and
  **linked as a prebuilt STATIC library** (`-DCAS_BACNET_STACK_LINK=STATIC`,
  built first by `tools/build-stack-static.sh`) instead of SOURCE. No DLL is
  shipped or documented; SOURCE remains available as the adapter's fallback
  mode only.
- `common/` synced to **v2.1.0** (see `common/CHANGELOG.md`), byte-identical
  with the rest of the series again.
- Interface changes that reached this example at the new pin: every
  `GetProperty*` callback (`Real`, `Enumerated`, `UnsignedInteger`,
  `CharacterString`, `Bool`, `OctetString`) gains a trailing `uint32_t*
  errorCode` out-parameter (declined without naming an error in every case
  here - see the new comment block in `main.cpp`);
  `BACnetStack_AddNetworkPortObjectWithNetworkNumber` was removed in favour of
  `BACnetStack_AddNetworkPortObject` taking the same argument list; and
  `CASExampleHelper::SetNetworkPortInstance(NETWORK_PORT_INSTANCE)` is now
  called before `RegisterCommonCallbacks()` so the shared transport callbacks
  know which Network Port object owns the bound socket.
- **Added the `docs/objects.json`-driven "Objects and properties" reference
  block**, the series-wide profile table block, and a `## Footprint` table
  placeholder to the README (§7 of the series runbook); `CMakeLists.txt` and
  the README's Build/Link-mode sections now describe the STATIC build only.
- `.github/workflows/release.yml` replaced with the series' proven template
  (windows-2022 + ubuntu-latest, static-library caching keyed on the stack
  commit, `metrics-*.json` publishing on a version tag).
- This is the repository canonical for **F-OUTPUTS**: AO 1 "Chartreuse", BO 1
  "Fuchsia", MSO 1 "Indigo", the `Commandable` struct, and the
  `SetPropertyWritable(Present_Value)` + Set/Null callback pattern other
  profile examples in the series copy.

### Changed (from the earlier CASBACnetStack::Adapter migration)

- **Links the CAS BACnet Stack through the `CASBACnetStack::Adapter` CMake target
  instead of compiling its `source/*.cpp` into this project directly.** `main.cpp`
  and `common/CASExampleHelper.cpp` now include `CASBACnetStackAdapter.h` and call
  `LoadBACnetFunctions()` once at the top of `main()`; **every `BACnetStack_*` call
  site is unchanged** — the adapter exposes the same export names in every link
  mode. `CAS_BACNET_STACK_LINK` (`SOURCE` default, or `STATIC`/`DLL`) now picks the
  link mode, so switching is a CMake flag rather than a code change. See the
  README's new "Link modes" section.
  - Stack pinned to `6.x-TestTool` @ `756371c1`, which carries the adapter
    (cas-bacnet-stack PRs #267 and #268).
  - `common/` bumped to **v1.5.1** (see `common/CHANGELOG.md`). The
    `LoadBACnetFunctions()` requirement is a contract change shared by every
    example in the series.
  - Release CI now passes `-DCAS_BACNET_STACK_LINK=SOURCE` **explicitly** and
    asserts it back out of `CMakeCache.txt`, so a published artifact stays a
    single self-contained executable even if the CMake default ever moves.
- **Documentation corrections** carried over from the B-ASC review: the version
  banner and sample output now match the shipped `common/` version (they claimed
  v1.3.0); the Troubleshooting table quoted a CMake error string that no longer
  exists and blamed `SO_REUSEADDR` for a symptom that on Windows surfaces as a bind
  failure (the socket asks for `SO_EXCLUSIVEADDRUSE`); added parallel-build
  guidance for the ~600-file first compile.

### Changed

- **Default device instance is now `389002`** (was `389001`), per the series'
  new device-instance table: each profile example has its own default so
  several examples can run on one subnet at once. Override with `--deviceID`.
- **CAS BACnet Stack pinned to the head of the `6.x` branch** (`14676437`).
  The previous pin was on a pre-6.x lineage; this brings ~248 commits of stack
  fixes and features. (The series is mid-migration to 6.x, so a few examples
  still pin the 5.x line; see the runbook's pin table for the current split.)
- `common/` is vendored at **v1.3.0** (see `common/CHANGELOG.md`): it carries its own
  version (`COMMON_VERSION`, printed at start-up) and its own changelog
  (`common/CHANGELOG.md`); the helper files are byte-identical with the rest
  of the series again.

## [1.0.0] - 2026-06-16

### Added

- Initial **B-SA (BACnet Smart Actuator)** profile example for the CAS BACnet
  Stack in C++.
- **Device** object "Rainbow" - default instance `389001`, vendor `389` (Chipkin
  Automation Systems) - with its full identity (name, description, vendor, model,
  firmware, application software version).
- Read-only sensor objects (the shared minimum across the example series):
  **Analog Input 1** "Bronze" (REAL, degrees Celsius, starts at 21.5), **Binary
  Input 1** "Emerald" (active/inactive), **Multi-State Input 1** "Hot Pink"
  (state 1..3, with `State_Text` "On"/"Off"/"Auto").
- Commandable **output** objects (the B-SA additions): **Analog Output 1**
  "Chartreuse" (REAL setpoint), **Binary Output 1** "Fuchsia" (active/inactive),
  **Multi-State Output 1** "Indigo" (state 1..3). Each is driven through a 16-slot
  `Priority_Array` + `Relinquish_Default`; the stack resolves `Present_Value` from
  the highest-priority non-null slot.
- **Network Port 1** "Vermilion" with full BACnet/IP addressing - `IP_Address`,
  `IP_Subnet_Mask`, `IP_Default_Gateway`, `BACnet_IP_UDP_Port`, `BACnet_IP_Mode`;
  `MAC_Address` is computed by the stack from the IP address and UDP port.
- **DS-RP-B** (ReadProperty) and **DS-WP-B** (WriteProperty), plus automatic
  **Who-Is / I-Am** discovery; an unsolicited I-Am is broadcast to the local
  subnet on start-up.
- WriteProperty of a value commands a priority slot; WriteProperty of NULL
  relinquishes it. Writes to read-only input objects are rejected, and writes
  outside an output's valid range (Binary Output != 0/1, Multi-State Output state
  outside 1..Number_Of_States) are rejected with `value-out-of-range`.
- All **required properties for Protocol_Revision 24** across every object.
- Interactive keys: `h` help, `q` quit, up/down nudge Analog Input 1 by +/-1.1.
- Command-line options: `--port <n>` and `--deviceID <n>`.
- Cross-platform CMake build that compiles the CAS BACnet Stack from source, with
  strict warnings (`-Wall -Wextra` / `/W4`) on the example's own sources only.
- Self-contained repository: the shared helper is vendored in `common/`, and the
  CAS BACnet Stack is included as a git submodule at
  `submodules/cas-bacnet-stack` - clone with `--recursive` and build.
- GitHub Actions workflow that builds Windows + Linux and publishes a release on
  a `vX.Y.Z` tag, with a smoke-test step before packaging.

[1.2.0]: https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases/tag/v1.2.0
[1.0.0]: https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases/tag/v1.0.0
