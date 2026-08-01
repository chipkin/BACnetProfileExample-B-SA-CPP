# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - unreleased

> Not tagged yet: `v1.0.0` is the only tag in this repository. `release.yml` publishes binaries on a `v*.*.*`
> tag, so until that tag exists this section describes what is on the
> branch, not what shipped.

### Changed

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

[1.0.0]: https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases/tag/v1.0.0
