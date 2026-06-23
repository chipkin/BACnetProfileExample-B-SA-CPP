# BACnet B-SA (Smart Actuator) - C++ example

A minimal, copy-paste-friendly example showing how to implement the BACnet
**B-SA (BACnet Smart Actuator)** device profile in C++ using the
[CAS BACnet Stack](https://store.chipkin.com/services/stacks/bacnet-stack).
It listens on **BACnet/IP (UDP 47808)**, answers **ReadProperty** requests,
accepts **WriteProperty** to its commandable outputs, and is discoverable via
**Who-Is / I-Am**.

> **Versions:** this document describes **example v1.0.0**, built against
> **CAS BACnet Stack v6.x.x** at **Protocol_Revision 24**. (Note: v6.x.x
> is under active development; the exact linked build is printed at start-up.)

This is the second example in the series. It builds directly on the
[B-SS (Smart Sensor)](https://github.com/chipkin/BACnetProfileExample-B-SS-CPP)
example: same three read-only sensor inputs, plus the one thing that makes a
device an *actuator* - it can be **commanded** with WriteProperty.

## What is a B-SA (BACnet Smart Actuator) profile?

A **device profile** is a standard "template" defined in Annex L of ANSI/ASHRAE
135. It lists the capabilities a class of device must support so that any
compliant client knows what to expect, and the BACnet Testing Laboratories (BTL)
certify devices against it. (New to BACnet in general? See Chipkin's
[What is BACnet?](https://docs.chipkin.com/protocols/bacnet/) guide.)

**B-SA (BACnet Smart Actuator)** is the next step up from the Smart Sensor - the
standard describes it as *"a simple actuating device with very limited
resources."* It is meant for inexpensive, fixed-function actuators (a valve, a
damper, a relay, a setpoint) that need to **report state** and **be driven** by a
controller.

**What the profile requires:**

- **Data Sharing - ReadProperty - B side (DS-RP-B):** the device must answer
  **ReadProperty** requests for the values of its objects.
- **Data Sharing - WriteProperty - B side (DS-WP-B):** the device must accept
  **WriteProperty** requests so a controller can drive its outputs. This is what
  distinguishes a B-SA from a B-SS.
- **Discovery:** the device must be findable, so it answers **Who-Is** with
  **I-Am**, and announces itself with an unsolicited I-Am at start-up.

**What the profile does NOT require** - and this example therefore omits on
purpose: **alarming / event reporting**, **scheduling**, and **trending**.

**But it is still a full BACnet device.** Even a simple profile must present the
standard object model - a **Device** object, a **Network Port** object (every
device needs one), and its objects - and each object must expose all of its
**required properties**. The CAS BACnet Stack generates most of those
automatically (Object_Identifier, Object_Type, Status_Flags, Event_State,
Object_List, Protocol_*, ...); this example supplies the handful that are
application-specific. The result is conformant for **Protocol_Revision 24**.

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
Device 389001  "Rainbow"   (Vendor 389 - Chipkin Automation Systems)
    │
    ├── Analog Input  1       "Bronze"      Present_Value  21.5    (REAL, degrees Celsius; read-only)
    ├── Binary Input  1       "Emerald"     Present_Value  active  (0 = inactive / 1 = active; read-only)
    ├── Multi-State Input 1   "Hot Pink"    Present_Value  1       (state, 1..3; read-only)
    ├── Analog Output 1       "Chartreuse"  Present_Value  20.0    (REAL setpoint; WRITABLE, commandable)
    ├── Binary Output 1       "Fuchsia"     Present_Value  inactive(0/1; WRITABLE, commandable)
    ├── Multi-State Output 1  "Indigo"      Present_Value  1       (state, 1..3; WRITABLE, commandable)
    └── Network Port 1        "Vermilion"   the BACnet/IP port     (required on every device)
```

The three **input** objects (Bronze, Emerald, Hot Pink) are the shared minimum
every example in this series carries. The three **output** objects (Chartreuse,
Fuchsia, Indigo) are the B-SA additions; their `Present_Value` starts at the
`Relinquish_Default` (no slots commanded yet). Object names follow this series'
colour-naming convention (Device is always "Rainbow").

## What this example supports

The example implements exactly the capabilities below - and nothing more, which
is the point of a profile example. These capabilities satisfy the **B-SA (BACnet
Smart Actuator)** profile; because they also cover the baseline required by
**B-GENERAL**, this example satisfies the **B-GENERAL** profile as well.

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
| Device | 389001 | Rainbow | - |
| Analog Input | 1 | Bronze | read-only |
| Binary Input | 1 | Emerald | read-only |
| Multi-State Input | 1 | Hot Pink | read-only |
| Analog Output | 1 | Chartreuse | writable (commandable) |
| Binary Output | 1 | Fuchsia | writable (commandable) |
| Multi-State Output | 1 | Indigo | writable (commandable) |
| Network Port | 1 | Vermilion | - |

## Requires the CAS BACnet Stack (licensed product)

This example **builds against the CAS BACnet Stack, which is a commercial Chipkin
product** - it is not free or open source, and there is no public/trial build.
The stack is referenced here as the **private** git submodule
`submodules/cas-bacnet-stack`; you can only fetch and build it once you have a CAS
BACnet Stack license and access to that repository.

**To get the CAS BACnet Stack (and access to build this example), contact
Chipkin:** <https://store.chipkin.com/services/stacks/bacnet-stack> or
sales@chipkin.com.

You can still read all of this example's source on GitHub to evaluate the
approach and the amount of code involved.

## What's in this repository

This is a **self-contained** project. It ships:

- `main.cpp` - the example device.
- `common/` - the shared helper (UDP, callbacks, CLI, keyboard) vendored in.
- `submodules/cas-bacnet-stack/` - the **CAS BACnet Stack as a git submodule**
  (private; requires a license - see above). Compiled from source; no prebuilt
  library or DLL is shipped.

## Footprint & performance

This example statically compiles the **entire** CAS BACnet Stack into one
executable (no external runtime/DLL). Release-build sizes of the whole
application (stack + example):

| Platform | Binary | Size |
|----------|--------|------|
| Windows x64 (MSVC, Release) | `BACnetExampleBSA.exe` | ~2.6 MB |
| Linux x64 (GCC, Release) | `BACnetExampleBSA` | ~6 MB unstripped (`strip` cuts it substantially) |

These are whole-application sizes. The stack's flash/RAM footprint on a
constrained MCU, CPU cost per `BACnetStack_Tick()`, ReadProperty/WriteProperty
latency, and the maximum number of objects depend on your target and
configuration. For embedded-sizing and benchmark figures, contact Chipkin -
<https://store.chipkin.com/services/stacks/bacnet-stack>.

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

## Get the code

Clone this repository **and its submodule** (the CAS BACnet Stack):

```bash
git clone --recursive https://github.com/chipkin/BACnetProfileExample-B-SA-CPP.git
cd BACnetProfileExample-B-SA-CPP

# already cloned without --recursive? fetch the submodule:
git submodule update --init --recursive
```

## Build

```bash
cmake -B build -S .
cmake --build build --config Release
```

> **First build takes a few minutes** - it compiles the entire CAS BACnet Stack
> (~460 source files) once. Incremental rebuilds after that are fast.

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
BACnet B-SA (Smart Actuator) Example - C++ v1.0.0
CAS BACnet Stack version: <the linked build, printed at start-up>
FYI: Listening for BACnet/IP on UDP port 47808.
TX 21 bytes to 192.168.3.255:47808 (broadcast)
FYI: Device 389001 ("Rainbow") ready. Vendor ID 389. Press 'h' for help.
```

The `TX` line is the start-up I-Am the device broadcasts to announce itself. It
goes to the **local subnet broadcast** address (here `192.168.3.255`, computed
from the Network Port's interface), not the global `255.255.255.255`. As clients
talk to the device you'll see `RX ... bytes from ...` and `TX ... bytes to ...`
lines showing the traffic; a WriteProperty to an output also prints a line such
as `WriteProperty: Analog Output 1 (Chartreuse) <- 42.50 @ priority 8`.

The device listens on UDP **47808** (BACnet/IP). Allow that port through your
firewall. To use a different port, pass `--port` (see below).

### Command-line options

| Option | Default | Meaning |
|--------|---------|---------|
| `--port <n>` | `47808` | UDP port to listen on (BACnet/IP). |
| `--deviceID <n>` | `389001` | The device's BACnet instance number (BACnet requires this to be configurable). |

### Interactive commands

While the example runs, these keys are available (shared across all examples in
the series):

| Key | Action |
|-----|--------|
| `h` | Show the version information and this command list. |
| `q` | Quit. |
| `e` | Enter **edit mode** to change a sensor input's live value. |

In **edit mode** you pick which input to change and adjust it:

| Key | Action |
|-----|--------|
| number | Select an object (Analog Input 1, Binary Input 1, or Multi-State Input 1). |
| up / down arrow | Change the selected value (nudge the analog, step the multi-state). |
| space | Toggle a binary input, or step to the next multi-state value. |
| `esc` | Leave edit mode. |

Editing changes the live `Present_Value` of the selected **input**, so a client
re-reading it sees the new value. Only the three read-only inputs are editable;
the commandable **outputs** are driven by `WriteProperty` from a client (their
`Present_Value` comes from the `Priority_Array`), not from the keyboard.

## Verify

Use a BACnet client such as the
[**CAS BACnet Explorer**](https://store.chipkin.com/products/tools/cas-bacnet-explorer):

1. **Discover** - send a **Who-Is**. The device replies with **I-Am** from
   instance **389001** (vendor **389**). It also broadcasts an I-Am at start-up.
2. **Browse the object model** - the device shows seven objects: the Device
   (`Rainbow`), three inputs, three outputs, and the Network Port (`Vermilion`).
   Reading the Device's `Object_List` returns all seven.
3. **Read the Device** - ReadProperty `389001` -> `Object_Name` returns
   `"Rainbow"`; `Protocol_Revision` returns `24`; `Description` returns the
   profile description string.
4. **Read a sensor** - ReadProperty Analog Input `1` -> `Present_Value` returns
   `21.5`; `Units` returns `degrees-Celsius`; `Object_Name` returns `"Bronze"`.
5. **Command an output (the B-SA test)** - ReadProperty Analog Output `1`
   -> `Present_Value` returns `20.0` (its `Relinquish_Default`, nothing commanded
   yet). Now **WriteProperty** Analog Output `1` `Present_Value` = `42.5` at
   priority `8`. Re-read `Present_Value`: it returns `42.5`, and
   `Priority_Array[8]` shows `42.5`. **WriteProperty** `Present_Value` = `NULL`
   at priority `8` to relinquish; `Present_Value` returns to `20.0`. Repeat for
   Binary Output `1` (`"Fuchsia"`, active/inactive) and Multi-State Output `1`
   (`"Indigo"`, state 1..3).
6. **Confirm the profile boundary** - a **WriteProperty** to a read-only *input*
   (e.g. Analog Input 1) is rejected. That is correct: inputs are sensors.

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| CMake error: *"CAS BACnet Stack source not found"* | Submodules not initialized. Run `git submodule update --init --recursive` (or pass `-D CAS_STACK_DIR=...`). |
| `CASBACnetStackDLL.h: No such file or directory` | Same - submodules not checked out. |
| Windows: *"No CMAKE_CXX_COMPILER could be found"* | Install Visual Studio with the "Desktop development with C++" workload, then re-run from a fresh terminal. |
| First build seems stuck for minutes | Normal - it's compiling ~460 stack files. Only the first build is slow. |
| App prints *"Failed to bind UDP port 47808"* | Another BACnet program is already using 47808. Stop it, or run with `--port <n>`. |
| WriteProperty to an output is rejected | Write to the **output** objects (Analog/Binary/Multi-State Output), not the inputs. Inputs are read-only sensors by design. |
| Client sends Who-Is but sees no I-Am | Firewall is blocking UDP 47808, or the client and device are on different subnets (Who-Is is a broadcast). Allow the port; test on the same subnet first. |
| Replies show an unexpected device instance or vendor | Another BACnet device is already running on this host/port (the socket uses `SO_REUSEADDR`, so several can share 47808). Stop the other device, or run this example on its own machine/IP. |

## Extending the example

The example is intentionally small so it's easy to change.

**Change a value or name** - edit the constants / callbacks in `main.cpp` (e.g.
the `Relinquish_Default` of an output in its `Commandable` initializer, or the
`"Chartreuse"` string in `GetPropertyCharString`).

**Add a second analog output** - the edits mirror the existing one in `main.cpp`:
add a new instance constant and `Commandable`, teach `GetCommandable` about it,
`BACnetStack_AddObject` it in `main`, and enable its `Priority_Array` +
`Relinquish_Default` + writable `Present_Value` in the commandable-setup loop.

Going beyond read/write (COV, alarms, scheduling) means implementing a richer
profile - a later example in this series.

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

## Use this in your own project

This repository is self-contained: clone it (with the submodule) and build, then
copy what you need into your product. The example source code is dedicated to the
public domain under [CC0-1.0](LICENSE) - use it for anything, no attribution
required. The CAS BACnet Stack is a separate, commercially licensed product and
is not covered by CC0.

See also [CHANGELOG.md](CHANGELOG.md) and [AGENTS.md](AGENTS.md).
