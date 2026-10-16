# B-SA Wire Tests

Tests that drive the running smart actuator over BACnet/IP from outside it. A
B-SA's Profile Function is to be commanded - ANSI/ASHRAE 135-2024 Annex L.4.4:
"Ability to allow modification of some or all of its BACnet objects by another
device". So the tests read everything the device lists, the way a client
discovering it does; then command each output at two priorities and watch the
actuator obey the winner; then disconnect the actuator and send commands it must
refuse.
They are written against the series' shared harness in
[`../wiretest/`](../wiretest/README.md) (vendored, like `common/`); everything
specific to this example is here. Python 3 standard library only - nothing to
`pip install`.

## Running

Build the example (README "Build"), then from the repository root:

```bash
python3 tests/run_wire_tests.py --exe build/BACnetExampleBSA
```

On Windows, `--exe build\Release\BACnetExampleBSA.exe`. It ends with
`ALL WIRE TESTS PASSED` (exit 0) or the scripts that failed. `--port` (default
47821) is the UDP port the test starts the example on. It takes about a minute.
CI (`.github/workflows/release.yml`) runs it on Linux after the smoke test.

## The setup

```
test client (127.0.0.1, ephemeral UDP port)  - the Peer
    |  BACnet/IP, unicast
BACnetExampleBSA             UDP --port   - Device 389002
```

The Peer is a Python BACnet client (`wiretest.Client`), never another example's
binary. It plays a BACnet client such as Chipkin BACnet Explorer, or a controller
commanding the actuator. `bsa_setup.py` holds what the scripts share: the object
and property numbers, the outputs' defaults and range, the Simulation's valve
rate (they match `main.cpp`), and starting the example.

A request unanswered after 0.6 s is sent again, up to 3 times
(`bsa_setup.REQUEST_TIMEOUT` / `RETRIES`, what a BACnet client's
Number_Of_APDU_Retries does); each resend prints a `[RETRY]` line, so a lost
request shows in the output instead of being tolerated silently. A busy Windows
host loses the odd UDP datagram even on loopback.

The one thing a test does to the example other than over BACnet is the
Simulation's test-only disconnected-valve hook: `failure_modes_test.py` starts
the example with `CAS_EXAMPLE_TEST_ACTUATOR_FAULT_FILE` naming a file in the
temp folder, and the valve actuator is disconnected while that file exists.

## `discovery_test.py` - every listed property reads as a value

1. **Who-Is** (for 389002, unicast) -> the device's I-Am.
2. **Device `Object_List`** names exactly the Device, Analog Input 1, Binary Input 1, Multi-state Input 1, Analog Output 1, Binary Output 1, Multi-state Output 1 and Network Port 1. This is the control: a walk
   over nothing would pass.
3. For each object, **`Property_List`** reads, and a ReadProperty of every
   property it lists - plus `Object_Identifier`, `Object_Name`, `Object_Type`
   and `Property_List`, which cl. 12.1.1.4.1 leaves out of the list - returns a
   ComplexACK. An Error, Reject, Abort or no answer prints an `[UNREADABLE]` line
   and fails the check. 135-2024 cl. 12.1.1.4.1: Property_List holds "one
   property identifier for each property that exists within the object".
4. Network Port 1's **`IP_DNS_Server` (406)**, **`FD_BBMD_Address` (418)** and
   **`FD_Subscription_Lifetime` (419)** are each listed and read as a value -
   the three reads Explorer 4.23.2.0 found failing at stack 6.0.21, reported
   against B-SS-CPP (#20, #21, #22). The Network Port object is the stack's, the
   same in every example; this test is copied from B-SS-CPP.

## `commanding_test.py` - the actuator obeys the winning command (the Profile Function)

Cl. 19.2: the highest-priority non-NULL `Priority_Array` slot is the
`Present_Value`; a NULL write relinquishes a slot; with every slot NULL,
`Relinquish_Default` applies.

1. **Analog Output 1 -> Analog Input 1.** Control: `Present_Value` 20.0
   (Relinquish_Default), `Current_Command_Priority` NULL, the temperature settles
   at 20.0. Write 24.0 @ 8: `Present_Value` 24.0, `Priority_Array[8]` 24.0,
   `Current_Command_Priority` 8; the temperature climbs to 24.0, with at least one
   reading in between (a valve, not a copy). Write 19.0 @ 12: stored in
   `Priority_Array[12]`, but `Present_Value` stays 24.0 and the temperature stays
   at 24.0 for 2.5 s. Relinquish 8: `Present_Value` 19.0, priority 12; the
   temperature falls to 19.0. Relinquish 12: back to 20.0, NULL priority; the
   temperature returns to 20.0.
2. **Binary Output 1 -> Binary Input 1** (1 @ 8, then 0 @ 4 wins, relinquish 4,
   relinquish 8) and **Multi-State Output 1 -> Multi-State Input 1** (3 @ 16,
   then 2 @ 10 wins, relinquish 10, relinquish 16): after every step
   `Present_Value` and `Current_Command_Priority` are the winner's, and the
   feedback input reads the winner within 2.5 s.

An example that only stores commands passes every BACnet-side check here and
fails every feedback check.

## `failure_modes_test.py` - a disconnected actuator, and commands it must refuse

1. **Control:** Analog Output 1 Reliability `no-fault-detected` (0), Status_Flags
   all FALSE, the temperature at 20.0.
2. **Disconnect the valve** (create the fault file): within 3 s Reliability reads
   `no-output` (6); Status_Flags is {FALSE, TRUE, FALSE, FALSE} - FAULT alone
   (12.3.7); a write of 26.0 @ 8 is still accepted (`Present_Value` 26.0), but
   the temperature is still 20.0 3 s later; Binary Output 1 is not faulted.
3. **Reconnect it** (delete the file): Reliability back to `no-fault-detected`
   within 3 s, Status_Flags all FALSE, and the temperature climbs to the 26.0 the
   output kept.
4. **Refusals** (15.9.1.3.1): `Min_Pres_Value` / `Max_Pres_Value` read 10.0 / 30.0;
   Error PROPERTY / VALUE_OUT_OF_RANGE (2/37) for a setpoint of 35.0 and 5.0, a
   Binary Output value 2, a Multi-State Output state 4 and 0; Error PROPERTY /
   WRITE_ACCESS_DENIED (2/40) for a write to Analog Input 1. After each,
   `Present_Value`, `Priority_Array[8]` and the feedback input are unchanged.

## Each check fails without its feature

Run 2026-10-16 on Windows (STATIC build, relinked each time - the binary's
mtime checked): one feature removed from `main.cpp`, rebuilt, the named script
run, the source restored.

| # | Feature removed (`main.cpp`) | Script | Fails with |
|---|---|---|---|
| A1 | `ActuatorSimulationTick()` not called | commanding, failure_modes | `Control: ... the temperature settles at 20.0 - ... temperature 21.5`; `... the temperature climbs to 24.0 ... - read [21.5, 21.5, ...]`; `... the feedback follows to 1 - read 0` |
| A2 | `ResolvedCommand()` ignores the Priority_Array | commanding | `... the temperature climbs to 24.0 ... - read [20.0, 20.0, ...]`; `... the feedback follows to 1 - read 0` |
| A3 | the LOWEST priority wins | commanding | `... and the temperature stays at 24.0 (the losing command moves nothing) - read 21.0`; `... the feedback follows to 0 - read 1`; `... to 2 - read 3` |
| A4 | the valve jumps straight to the setpoint | commanding | `... gradually - at least one reading on the way between 20.0 and 24.0 ... - read [20.0, 20.0, 20.0, 20.0, 24.0]` |
| A5 | the relay's feedback not updated | commanding | `... the feedback follows to 1 - read 0` |
| A6 | the drive's feedback not updated | commanding | `... the feedback follows to 3 - read 1` |
| A15 | Binary Input 1 serves a constant 0 | commanding | `... the feedback follows to 1 - read 0` |
| A7 | the valve never disconnects | failure_modes | `Valve disconnected: Analog Output 1 Reliability reads NO_OUTPUT within 3 s - read 0`; `... Status_Flags FAULT is TRUE ... - read [False, False, False, False]`; `... the temperature does not follow ... - read 23.0` |
| A8 | the valve moves while disconnected | failure_modes | `Valve disconnected: the temperature does not follow (3 s later, still 20.0) - read 23.0` |
| A9 | the fault latches | failure_modes | `Valve reconnected: Reliability back to NO_FAULT_DETECTED within 3 s - read 6`; `... the temperature now climbs to the command it held, 26.0 - read 20.0` |
| A10 | the Reliability branch declines | failure_modes | `Valve disconnected: ... Reliability reads NO_OUTPUT within 3 s - read 0` (the stack substitutes 0 = no-fault-detected, silently) |
| A11 | the setpoint range not checked | failure_modes | `Analog Output 1 setpoint 35.0 (above Max_Pres_Value) -> Error PROPERTY / VALUE_OUT_OF_RANGE - SimpleACK`; `... and nothing moved ... - before [20.0, None, 20.0], after [35.0, 35.0, 21.0]` |
| A14 | Min_Pres_Value not served | failure_modes | `Analog Output 1 serves its setpoint range: Min_Pres_Value 10.0, Max_Pres_Value 30.0` |
| A12 | the Binary Output 0/1 check | failure_modes | **survives**: the stack range-checks a BACnetBinaryPV write itself before calling `SetPropertyEnumerated` |
| A13 | the Multi-State Output 1..3 check | failure_modes | **survives**: the stack checks the state against the `Number_Of_States` the example serves |
| A13b | `Number_Of_States` served as 4 | failure_modes | **survives**: the callback's own check refuses state 4 |
| A13c | both (4 states and no check) | failure_modes | `Multi-State Output 1 state 4 (it has 3) -> Error PROPERTY / VALUE_OUT_OF_RANGE - SimpleACK`; `... nothing moved ... after [4, 4, 4]` |

So the Binary / Multi-State refusals are held by two layers - the stack's own
check and the callback's - and the checks here gate the pair (A13c). The Analog
Output range is the example's alone (A11). The `write-access-denied` for an input
is the stack's: there is no setter for an input to remove.

