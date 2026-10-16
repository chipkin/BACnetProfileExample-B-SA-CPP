# BACnet B-SA (Smart Actuator) - C++ example

A minimal, copy-paste-friendly example showing how to implement the BACnet
**B-SA (BACnet Smart Actuator)** device profile in C++ using the
[CAS BACnet Stack](https://store.chipkin.com/services/stacks/bacnet-stack).
It listens on **BACnet/IP (UDP 47808)**, answers **ReadProperty** requests,
accepts **WriteProperty** to its commandable outputs, and is discoverable via
**Who-Is / I-Am**. Each output **does what the winning command says**: a heating
valve brings the temperature to the commanded setpoint, a relay follows its
command, a drive runs at the commanded speed, and each reports back through an
input. A disconnected valve shows up as a **fault** a client can read. The
valve, relay and drive are a [Simulation](#the-actuator-simulation); everything
BACnet does with them is real.

**[Download a prebuilt binary](https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases)**
(Windows and Linux x64) - or build it yourself, see [Build](#build) below.

- **[TUTORIAL.md](TUTORIAL.md)** - how to extend this example and how to review
  it for conformance. Read it when you start turning this into your own device.
- **[docs/PICS.md](docs/PICS.md)** - the Protocol Implementation Conformance
  Statement: every object, every property, and who answers it.

> **Versions:** this document describes **example v1.3.0**, built and verified
> against **CAS BACnet Stack 6.0.23** (`6.x` @ `64b2cfac5`), at
> **Protocol_Revision 30**, with the vendored `common/` helper at **v3.4.0**.
> Running the example prints all three - if what it prints disagrees with this
> line, trust the program and check `CHANGELOG.md`.

## What is the B-SA (Smart Actuator) profile?

**B-SA (BACnet Smart Actuator)**, defined in Annex L.4.4 of ANSI/ASHRAE
135-2024, is the next step up from the Smart Sensor - the standard describes it
as *"a simple control device with limited resources; it is intended for
specific applications."* In practice that is an inexpensive, fixed-function
actuator (a valve, a damper, a relay, a setpoint) that **reports state** and
**accepts commands** from a controller.

A B-SA device answers **ReadProperty** and accepts **WriteProperty** to its
commandable outputs - that is what distinguishes an actuator from a sensor. It
does not have to support **alarming / event reporting**, **scheduling**, or
**trending**, and this example implements none of them on purpose.

**What this example does with a command:** it stores the write in the
output's `Priority_Array`, the stack resolves `Present_Value` from it, and the
actuator Simulation drives its (simulated) equipment from that same resolved
value - so a client sees the command take effect on the matching input, not
just read back.

**But it is still a full BACnet device.** Even the simplest profile must present
the standard object model - a **Device** object, a **Network Port** object (every
device needs one), and its objects - and each object must expose all of its
**required properties**. The CAS BACnet Stack generates most of those
automatically (Object_Identifier, Object_Type, Status_Flags, Object_List,
Protocol_*, ...); this example supplies the handful that are
application-specific. The result is conformant for **Protocol_Revision 30**.
[docs/PICS.md](docs/PICS.md) lists every property and who answers it.

## Commandable outputs (the heart of B-SA)

A BACnet output object's `Present_Value` is not written directly. Instead it is
**commandable**: it is resolved from a 16-slot **`Priority_Array`** plus a
**`Relinquish_Default`**.

- A `WriteProperty(Present_Value, value, priority)` stores `value` in slot
  `priority` (1 = highest .. 16 = lowest). If the client omits the priority,
  BACnet uses 16.
- A `WriteProperty(Present_Value, NULL, priority)` **relinquishes** that slot.
- The stack reports the value in the **highest-priority non-null slot** as the
  effective `Present_Value`; if every slot is null, it reports
  `Relinquish_Default`.

This lets several controllers command the same output at different priorities
(e.g. a manual override at priority 8 beats a normal command at 16) and cleanly
hand control back by writing NULL. The application just stores the array (see the
`Commandable` struct in `main.cpp`); the stack does the priority resolution.

## The device this example creates

```
Device 389002  "Chipkin Example B-SA"   (Vendor 389 - Chipkin Automation Systems)
    │
    ├── Analog Input  1       "Bronze"      Present_Value  21.5 -> follows the setpoint  (REAL, degrees Celsius; read-only)
    ├── Binary Input  1       "Emerald"     Present_Value  follows Binary Output 1      (0 = inactive / 1 = active; read-only)
    ├── Multi-State Input 1   "Hot Pink"    Present_Value  follows Multi-State Output 1 (state, 1..3; read-only)
    ├── Analog Output 1       "Chartreuse"  Present_Value  20.0    (REAL setpoint, 10.0 .. 30.0 C; WRITABLE, commandable)
    ├── Binary Output 1       "Fuchsia"     Present_Value  inactive  (0/1 relay; WRITABLE, commandable)
    ├── Multi-State Output 1  "Indigo"      Present_Value  1       (speed, 1..3; WRITABLE, commandable)
    └── Network Port 1        "Vermilion"   the BACnet/IP port     (required on every device)
```

The three **input** objects (Bronze, Emerald, Hot Pink) are the shared minimum
every example in this series carries; here each is the **feedback** of one
output. The three **output** objects (Chartreuse, Fuchsia, Indigo) are the B-SA
additions; their `Present_Value` starts at the `Relinquish_Default` (no slots
commanded yet). Object names follow this series' colour-naming convention
(Device is always "Chipkin Example B-SA").

## The scenario: an actuator a controller commands

What a B-SA is for, in one sentence, from Annex L.4.4 of ANSI/ASHRAE 135-2024:
*"Ability to allow modification of some or all of its BACnet objects by another
device"* - a controller (the **Peer** - any BACnet client, or the Python client
in [`tests/`](tests/README.md)) commands an output, and the actuator does what
the **winning** command says:

1. The Peer writes Analog Output 1 `Present_Value` = `24.0` at priority 8. The
   valve opens and the temperature (Analog Input 1) climbs 1.0 C a second until
   it reads `24.0`.
2. A second Peer writes `19.0` at priority 12. Priority 8 still wins:
   `Present_Value` stays `24.0`, `Priority_Array[12]` holds `19.0`, and the
   temperature does not move.
3. The first Peer relinquishes priority 8 (writes NULL). Priority 12 wins now:
   the temperature falls to `19.0`. Relinquish priority 12 and the output
   returns to its `Relinquish_Default`, `20.0` - and so does the temperature.
4. Binary Output 1 and Multi-State Output 1 work the same way; Binary Input 1
   (the relay's feedback contact) and Multi-State Input 1 (the drive's speed)
   follow the winning command within a second.
5. **Failure Mode - a command it must refuse.** A setpoint outside
   `Min_Pres_Value` .. `Max_Pres_Value` (10.0 .. 30.0), a Binary Output value
   other than 0/1, or a Multi-State Output state outside 1..3 gets Error
   `property` / `value-out-of-range`, and nothing moves. A WriteProperty to an
   input gets Error `property` / `write-access-denied`.
6. **Failure Mode - a disconnected actuator.** When the valve actuator is
   disconnected, Analog Output 1 `Reliability` reads `no-output` (6, "No
   physical device is connected to the Output object", clause 12.1) and the
   **FAULT** bit of its `Status_Flags` is TRUE (clause 12.3.7). Commands are
   still accepted into the `Priority_Array`, but the temperature does not move.
   Reconnect it and `Reliability` returns to `no-fault-detected`, FAULT clears,
   and the valve obeys the winning command.

### The actuator Simulation

A real smart actuator drives a valve motor, a relay coil and a variable-speed
drive, and reads back what they did. This example has none of them, so it
**simulates** them - non-BACnet scaffolding, outside the BACnet Surface, in
section 1b of [`main.cpp`](main.cpp) (`ActuatorSimulationTick()`, called beside
`BACnetStack_Tick()`). Once a second:

- the **heating valve** moves the temperature (Analog Input 1) up to 1.0 C toward
  the winning Analog Output 1 setpoint;
- the **relay's feedback contact** (Binary Input 1) takes the winning Binary
  Output 1 command;
- the **drive's speed** (Multi-State Input 1) takes the winning Multi-State
  Output 1 command.

"Winning" is `ResolvedCommand()`: the highest-priority commanded slot, else
`Relinquish_Default` - the same answer the stack gives for `Present_Value`
(clause 19.2).

The **disconnected valve** is a test-only hook, not a feature: if the
environment variable `CAS_EXAMPLE_TEST_ACTUATOR_FAULT_FILE` names a file, the
valve actuator counts as disconnected for as long as that file exists. The Wire
Tests create and delete it. Nothing on the network can disconnect it.

To port the example to hardware, replace `ActuatorSimulationTick()` with code
that drives your outputs from `ResolvedCommand()` and samples your feedback
inputs, and set `Reliability` from your output driver's own diagnostics.

## What this example supports

The example implements exactly the capabilities below - and nothing more, which
is the point of a profile example.

### BIBBs (BACnet Interoperability Building Blocks)

| BIBB | Description | Supported |
|------|-------------|:---------:|
| DS-RP-B | Data Sharing - ReadProperty - B | ✅ |
| DS-WP-B | Data Sharing - WriteProperty - B | ✅ |
| DM-DDB-B | Device Management - Dynamic Device Binding - B | ✅ |
| DM-DOB-B | Device Management - Dynamic Object Binding - B | ✅ |

### Services (executed / B-side)

| Service | Notes |
|---------|-------|
| ReadProperty | Responds to property reads (DS-RP-B). |
| WriteProperty | Accepts writes to the commandable outputs' Present_Value (DS-WP-B). |
| Who-Is / I-Am | Answers Who-Is with I-Am, and broadcasts an I-Am on start-up (DM-DDB-B). |
| Who-Has / I-Have | Answers Who-Has with I-Have (DM-DOB-B). |

### Object types

| Object type | Instance | Name | Access |
|-------------|:--------:|------|--------|
| Device | 389002 | Chipkin Example B-SA | - |
| Analog Input | 1 | Bronze | read-only |
| Binary Input | 1 | Emerald | read-only |
| Multi-State Input | 1 | Hot Pink | read-only |
| Analog Output | 1 | Chartreuse | writable (commandable) |
| Binary Output | 1 | Fuchsia | writable (commandable) |
| Multi-State Output | 1 | Indigo | writable (commandable) |
| Network Port | 1 | Vermilion | - |

Every required property of every object, and who answers it, is in
[docs/PICS.md](docs/PICS.md).

## Requires the CAS BACnet Stack (licensed product)

This example **builds against the CAS BACnet Stack, which is a commercial Chipkin
product** - it is not free or open source, and there is no public/trial build.
The stack is referenced here as the **private** git submodule
`submodules/cas-bacnet-stack`; you can only fetch and build it once you have a CAS
BACnet Stack license and access to that repository.

**To get the CAS BACnet Stack (and access to build this example), contact
Chipkin:** <https://store.chipkin.com/services/stacks/bacnet-stack> or
sales@chipkin.com.

You do not need a stack licence to *read* this example, or to run a
[prebuilt release binary](https://github.com/chipkin/BACnetProfileExample-B-SA-CPP/releases).
The licence is what lets you *build* it - that is the part the stack submodule
gates.

## What's in this repository

This is a **self-contained** project. It ships:

- `main.cpp` - the example device.
- `common/` - the shared helper (UDP, callbacks, CLI, keyboard) vendored in.
- `CMakeLists.txt` - the build, the same on Windows, Linux, and macOS.
- `docs/PICS.md` - the conformance statement.
- `tests/` - the Wire Tests (see [Wire Tests](#wire-tests)), and `wiretest/`, the
  series' shared Python harness they are written against (vendored, like `common/`).
- `submodules/cas-bacnet-stack/` - the **CAS BACnet Stack as a git submodule**
  (private; requires a license - see above). Its sources are compiled into the
  executable, so there is no library or DLL to build, ship, or install.

## Prerequisites

- A C++17 compiler (MSVC, GCC, or Clang).
- CMake >= 3.15.
- Git (to fetch the stack submodule).

### Windows

- **C++ compiler** - install
  [Visual Studio Community](https://visualstudio.microsoft.com/downloads/)
  (free) and select the **"Desktop development with C++"** workload.
- **CMake** - from <https://cmake.org/download/>, or `winget install Kitware.CMake`.

### Linux / macOS

- Debian/Ubuntu: `sudo apt install build-essential cmake git`
- macOS: `xcode-select --install` and `brew install cmake`

## Build

CMake only, and the same two commands on every platform:

```bash
git clone --recursive https://github.com/chipkin/BACnetProfileExample-B-SA-CPP.git
cd BACnetProfileExample-B-SA-CPP

cmake -B build -S .
cmake --build build --config Release
```

Already cloned without `--recursive`? Run `git submodule update --init --recursive`
first - the build needs the stack submodule.

> **The first build takes a few minutes** - it compiles the entire CAS BACnet
> Stack (~600 source files) into the executable. Rebuilds after that are
> incremental and take seconds.

If your CAS BACnet Stack lives somewhere other than the bundled submodule, point
CMake at it: `cmake -B build -S . -D CAS_STACK_DIR=/path/to/cas-bacnet-stack`.

## Run

```bash
# Linux / macOS
./build/BACnetExampleBSA

# Windows
.\build\Release\BACnetExampleBSA.exe
```

Expected output:

```
BACnet B-SA (Smart Actuator) Example - C++ v1.3.0
CAS BACnet Stack version: 6.0.23.0
Common helper (common/) version: 3.4.0
FYI: Listening for BACnet/IP on UDP port 47808 (Network Port 1).
TX 21 bytes to 192.168.3.255:47808 (broadcast) (Network Port 1) - Unconfirmed: I-Am
FYI: Device 389002 ("Chipkin Example B-SA") ready. Vendor ID 389. Press 'h' for help.
```

The `TX` line is the start-up I-Am the device broadcasts to announce itself. It
goes to the **local subnet broadcast** address (here `192.168.3.255`, computed
from the Network Port's interface), not the global `255.255.255.255`. As clients
talk to the device you'll see `RX ... bytes from ...` and `TX ... bytes to ...`
lines showing the traffic; a WriteProperty to an output also prints a line such
as `WriteProperty: Analog Output 1 (Chartreuse) <- 24.00 @ priority 8`.

The device listens on UDP **47808** (BACnet/IP). Allow that port through your
firewall. To use a different port, pass `--port` (see below).

> **A wall of red `Error:` lines at start-up is expected and is not your bug** -
> it is the stack's own debug logging (the device hearing its own broadcast I-Am,
> and a one-time BACnet/SC UUID notice). [TUTORIAL.md](TUTORIAL.md#troubleshooting)
> explains both.

### Command-line options

| Option | Default | Meaning |
|--------|---------|---------|
| `--port <n>` | `47808` | UDP port to listen on (BACnet/IP). |
| `--deviceID <n>` | `389002` | The device's BACnet instance number (BACnet requires this to be configurable). |
| `--help`, `-h` | - | Show usage and exit. |
| `--version` | - | Print the example, stack, and `common/` helper versions, then exit. |

### Interactive commands

While the example runs, these keys are available:

| Key | Action |
|-----|--------|
| `h` | Show the version information and this command list. |
| `q` | Quit. |
| up arrow | Disturb the simulated temperature (Analog Input 1, `Bronze`) up by 1.1. |
| down arrow | Disturb the simulated temperature (Analog Input 1, `Bronze`) down by 1.1. |

The up/down keys push the simulated temperature off its setpoint; within a few
seconds the valve brings it back, and a client re-reading Analog Input 1 sees
both.

## Verify

Use a BACnet client such as the
[**CAS BACnet Explorer**](https://store.chipkin.com/products/tools/cas-bacnet-explorer):

1. **Discover** - send a **Who-Is**. The device replies with **I-Am** from
   instance **389002** (vendor **389**). It also broadcasts an I-Am at start-up.
2. **Browse the object model** - the device shows eight objects: the Device
   (`Chipkin Example B-SA`), three inputs, three outputs, and the Network Port (`Vermilion`).
   Reading the Device's `Object_List` returns all eight.
3. **Read the Device** - ReadProperty `389002` -> `Object_Name` returns
   `"Chipkin Example B-SA"`; `Protocol_Revision` returns `30`; `Description` returns the
   profile description string.
4. **Read a sensor** - ReadProperty Analog Input `1` -> `Present_Value` returns
   `20.0` once the valve has settled at the default setpoint (it starts at
   `21.5`); `Units` returns `degrees-Celsius`; `Object_Name` returns `"Bronze"`.
5. **Command an output (the B-SA test)** - ReadProperty Analog Output `1`
   -> `Present_Value` returns `20.0` (its `Relinquish_Default`, nothing commanded
   yet). Now **WriteProperty** Analog Output `1` `Present_Value` = `24.0` at
   priority `8`. Re-read `Present_Value`: it returns `24.0`, and
   `Priority_Array[8]` shows `24.0`. Read Analog Input `1` for a few seconds: it
   climbs to `24.0`. **WriteProperty** `Present_Value` = `NULL` at priority `8`
   to relinquish; `Present_Value` returns to `20.0`, and so does Analog Input `1`.
   Repeat for Binary Output `1` (`"Fuchsia"`, active/inactive - watch Binary
   Input `1`) and Multi-State Output `1` (`"Indigo"`, state 1..3 - watch
   Multi-State Input `1`).
6. **Confirm the profile boundary** - a **WriteProperty** to a read-only *input*
   (e.g. Analog Input 1) is rejected. That is correct: inputs are sensors.

For a property-by-property review against the conformance statement, see
[TUTORIAL.md](TUTORIAL.md).

## Wire Tests

The checks above, automated: a Python BACnet client (the Peer) drives the running
example over BACnet/IP from outside it. Python 3 standard library only.

| Script | What it proves |
|---|---|
| `discovery_test.py` | Who-Is, the Device's `Object_List`, then for every object its `Property_List` and a ReadProperty of every property it lists - every read comes back as a value. |
| `commanding_test.py` | **The Profile Function.** Commands each output at two priorities; `Present_Value`, `Priority_Array` and `Current_Command_Priority` follow the winning command, and the feedback input follows it too - through a lower-priority write that must change nothing, two relinquishes, and back to `Relinquish_Default`. Fails against an example that only stores commands. |
| `failure_modes_test.py` | **The Failure Modes.** Disconnects the valve actuator: `Reliability` `no-output`, `Status_Flags` FAULT, a command accepted but not obeyed; reconnects it: all clear, the valve obeys. Then out-of-range writes (`value-out-of-range`, nothing moves) and a write to an input (`write-access-denied`). |

Run them all:

```bash
python3 tests/run_wire_tests.py --exe build/BACnetExampleBSA          # Linux/macOS
python tests\run_wire_tests.py --exe build\Release\BACnetExampleBSA.exe   # Windows
```

It ends with `ALL WIRE TESTS PASSED` (exit 0). CI runs it on Linux after the
smoke test. [tests/README.md](tests/README.md) says what each check proves.


## The BACnet profile example series

<!-- PROFILE-TABLE:BEGIN (generated from cas-bacnet-stack-examples/docs/profile-table.md - do not edit here) -->
There is one example repository per standardized device profile in ASHRAE 135-2024 Annex L. An example marked ⛔ is blocked: the stack cannot yet perform that profile's function through its customer interface, and the example's README says what it does today. Pick the profile your device claims, then the language you build in. "Ask" means the example hasn't been built yet for that language - [contact Chipkin](https://store.chipkin.com/contact-us) if you need one.

### Controllers (Annex L.4)

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-SS** Smart Sensor | [B-SS-CPP](https://github.com/chipkin/BACnetProfileExample-B-SS-CPP) | [B-SS-Node](https://github.com/chipkin/BACnetProfileExample-B-SS-Node) | [B-SS-CS](https://github.com/chipkin/BACnetProfileExample-B-SS-CS) | [B-SS-Rust](https://github.com/chipkin/BACnetProfileExample-B-SS-Rust) | [B-SS-Python](https://github.com/chipkin/BACnetProfileExample-B-SS-Python) | [B-SS-Go](https://github.com/chipkin/BACnetProfileExample-B-SS-Go) |
| **B-SA** Smart Actuator | [B-SA-CPP](https://github.com/chipkin/BACnetProfileExample-B-SA-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-ASC** Application Specific Controller | [B-ASC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ASC-CPP) | [B-ASC-Node](https://github.com/chipkin/BACnetProfileExample-B-ASC-Node) | Ask | Ask | Ask | Ask |
| **B-AAC** Advanced Application Controller | [B-AAC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AAC-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-BC** Building Controller | [B-BC-CPP](https://github.com/chipkin/BACnetProfileExample-B-BC-CPP) | Ask | Ask | Ask | Ask | Ask |

### Life safety controllers (Annex L.5)

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-LSC** Life Safety Controller | [B-LSC-CPP](https://github.com/chipkin/BACnetProfileExample-B-LSC-CPP) 🚧 | Ask | Ask | Ask | Ask | Ask |
| **B-ALSC** Advanced Life Safety Controller | [B-ALSC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ALSC-CPP) | Ask | Ask | Ask | Ask | Ask |

### Access control controllers (Annex L.6)

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-ACC** Access Control Controller | [B-ACC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACC-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-AACC** Advanced Access Control Controller | [B-AACC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AACC-CPP) | Ask | Ask | Ask | Ask | Ask |

### Lighting controllers (Annex L.11)

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-LD** Lighting Device | [B-LD-CPP](https://github.com/chipkin/BACnetProfileExample-B-LD-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-LS** Lighting Supervisor | [B-LS-CPP](https://github.com/chipkin/BACnetProfileExample-B-LS-CPP) | Ask | Ask | Ask | Ask | Ask |

### Elevator controllers (Annex L.13)

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-EM** Elevator Monitor | [B-EM-CPP](https://github.com/chipkin/BACnetProfileExample-B-EM-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-EC** Elevator Controller | [B-EC-CPP](https://github.com/chipkin/BACnetProfileExample-B-EC-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-AEC** Advanced Elevator Controller | [B-AEC-CPP](https://github.com/chipkin/BACnetProfileExample-B-AEC-CPP) | Ask | Ask | Ask | Ask | Ask |

### Authentication and authorization (Annex L.14)

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-AS** Authorization Server | [B-AS-CPP](https://github.com/chipkin/BACnetProfileExample-B-AS-CPP) ⛔ blocked on [stack #2043](https://github.com/chipkin/cas-bacnet-stack/issues/2043) | Ask | Ask | Ask | Ask | Ask |

### Miscellaneous (Annex L.7, combinable with any one family)

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-BBMD** Broadcast Management Device | [B-BBMD-CPP](https://github.com/chipkin/BACnetProfileExample-B-BBMD-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-ACDC** Access Control Door Controller | [B-ACDC-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACDC-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-ACCR** Access Control Credential Reader | [B-ACCR-CPP](https://github.com/chipkin/BACnetProfileExample-B-ACCR-CPP) ⛔ blocked on [stack #3934](https://github.com/chipkin/cas-bacnet-stack/issues/3934) | Ask | Ask | Ask | Ask | Ask |
| **B-RTR** Router | [B-RTR-CPP](https://github.com/chipkin/BACnetProfileExample-B-RTR-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-GW** Gateway | [B-GW-CPP](https://github.com/chipkin/BACnetProfileExample-B-GW-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-DAP** Device Address Proxy | [B-DAP-CPP](https://github.com/chipkin/BACnetProfileExample-B-DAP-CPP) ⛔ blocked on [stack #2032](https://github.com/chipkin/cas-bacnet-stack/issues/2032) | Ask | Ask | Ask | Ask | Ask |
| **B-SCHUB** BACnet/SC Hub | [B-SCHUB-CPP](https://github.com/chipkin/BACnetProfileExample-B-SCHUB-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-GENERAL** General device (Annex L.8) | *(satisfied by every example above)* | — | — | — | — | — |

### Operator interfaces and workstations (Annex L.1–L.3, L.9–L.10, L.12)

Client-side profiles.

| Profile | C++ | Node.js | C# | Rust | Python | Go |
|---|---|---|---|---|---|---|
| **B-OD** Operator Display | [B-OD-CPP](https://github.com/chipkin/BACnetProfileExample-B-OD-CPP) | Ask | Ask | Ask | Ask | Ask |
| **B-OWS** Operator Workstation | planned | — | — | — | — | — |
| **B-AWS** Advanced Operator Workstation | planned | — | — | — | — | — |
| **B-XAWS** Extended Advanced Operator Workstation | planned | — | — | — | — | — |
| **B-LSAP** Life Safety Annunciator Panel | planned | — | — | — | — | — |
| **B-LSWS** Life Safety Workstation | planned | — | — | — | — | — |
| **B-ALSWS** Advanced Life Safety Workstation | planned | — | — | — | — | — |
| **B-ACSD** Access Control Security Display | planned | — | — | — | — | — |
| **B-ACWS** Access Control Workstation | planned | — | — | — | — | — |
| **B-AACWS** Advanced Access Control Workstation | planned | — | — | — | — | — |
| **B-LOD** Lighting Operator Display | planned | — | — | — | — | — |
| **B-ALWS** Advanced Lighting Workstation | planned | — | — | — | — | — |
| **B-LCS** Lighting Control Station | planned | — | — | — | — | — |
| **B-ALCS** Advanced Lighting Control Station | planned | — | — | — | — | — |
| **B-ED** Elevator Display | planned | — | — | — | — | — |
| **B-EWS** Elevator Workstation | planned | — | — | — | — | — |
| **B-AEWS** Advanced Elevator Workstation | planned | — | — | — | — | — |

🚧 = in progress. ⛔ = Blocked Example (blocked on the named stack gap). "Ask" = not yet built for that language; contact Chipkin if you need it. Profile definitions: ANSI/ASHRAE 135-2024 Annex L. BIBB definitions: Annex K. Get the stack: <https://store.chipkin.com/services/stacks/bacnet-stack>.
<!-- PROFILE-TABLE:END -->

## References

- **ANSI/ASHRAE Standard 135** (BACnet) - the protocol standard. Object model
  (Clause 12), services (Clause 15), BACnet/IP (Annex J), device profiles
  (Annex L). Purchase / preview via the [ASHRAE store](https://www.ashrae.org/technical-resources/standards-and-guidelines).
- **What is BACnet?** - Chipkin's introduction:
  <https://docs.chipkin.com/protocols/bacnet/>.
- **CAS BACnet Stack** - product page and documentation:
  <https://store.chipkin.com/services/stacks/bacnet-stack>.
- **CAS BACnet Explorer** - client for testing this device:
  <https://store.chipkin.com/products/tools/cas-bacnet-explorer>.
- **B-SS (Smart Sensor) example** - the read-only sibling this builds on:
  <https://github.com/chipkin/BACnetProfileExample-B-SS-CPP>.
- **Shared helper used by this example** - [`common/README.md`](common/README.md).

See also [TUTORIAL.md](TUTORIAL.md), [docs/PICS.md](docs/PICS.md),
[CHANGELOG.md](CHANGELOG.md), and [AGENTS.md](AGENTS.md).
