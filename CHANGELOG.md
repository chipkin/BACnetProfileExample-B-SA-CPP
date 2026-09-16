# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
