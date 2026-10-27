# VeriBACnet results

This is the latest VeriBACnet run of this folder's project, summarised by hand
from the run's `report.json`. The run's own Results Folder (reports, packet
captures, wire logs) is not published, because it holds the testing machine's
network addresses.

| | |
|---|---|
| Date | 2026-10-27 |
| Example | BACnet B-SA (Smart Actuator) Example - C++ v1.3.0 (with the unreleased changes in CHANGELOG.md), built on CAS BACnet Stack 6.x @ 69b6249f |
| Profile | B-SA (BACnet Smart Actuator) |
| Transport | BACnet/IP, UDP port 47826 |
| VeriBACnet | 0.20.0 (its test device uses CAS BACnet Stack 6.0.23.0) |
| Test plan | 54 BTL tests (ANSI/ASHRAE 135.1-2023) selected for the B-SA profile |
| Result | **42 PASS · 0 FAIL · 0 ERROR · 12 SKIP** |

**Judged BTL-clean on BACnet/IP.** No test failed and none ended in an error,
and every skipped test has a stated reason. See "What BTL-clean means" in
[README.md](README.md). This is not BTL certification.

**Measured coverage:** 42 of the 54 selected tests ran, and all 42 passed.
- 4 of the 12 skipped tests need an operator, and the run was unattended.
- 8 do not apply to this device: it cannot create or delete objects, its only
  renameable or renumberable object is the Device (changed by restarting it),
  and it has no writable non-commandable property, writable fixed-size array or
  optional-functionality property.

Every BIBB the example claims had tests that ran: reading properties (including
the 7.1.1 Read Support procedure, against the EPICS), writing and commanding
the outputs (including the 7.2.2 Write Support procedure on all three outputs),
Who-Is / I-Am and Who-Has / I-Have.

This run replaces the 2026-10-26 run (54 tests, 41 PASS · 0 FAIL · 13 SKIP, 41
of 54 ran). What changed:
- The test tool was updated. 9.22.2.1 (Writing Non-Array Properties with an
  Array Index) now runs on the writable Analog Output 1 the project declares,
  and passes: the device answers propertyIsNotAnArray and keeps its value. It
  was skipped before.
- Nothing in the example changed, and the generated EPICS (`.tpi`) is the same
  as the committed one.

## Tests that did not pass

| Test | Result | Why |
|---|---|---|
| 7.3.2.10.9 Ensure Device Object_Name is Configurable | SKIP | Needs an operator to restart the device with another name (`--deviceName`); the run was unattended. |
| 7.3.2.10.10 Ensure Device Object_Identifier is Configurable | SKIP | Needs an operator to restart the device with another instance (`--deviceID`); the run was unattended. |
| 9.32.1.12 Who-Has After Object_Name Changed | SKIP | The device refuses a network write to its name, so the test asks an operator to change it by a restart; the run was unattended. |
| 9.32.1.13 Who-Has After Object_Identifier Changed | SKIP | The Object_Identifier form of 9.32.1.12 (`--deviceID`), with the same reason. |
| 7.3.2.10.3 Database_Revision after creating an object | SKIP | The device cannot create objects, so the test does not apply. |
| 7.3.2.10.4 Database_Revision after deleting an object | SKIP | The device cannot delete objects by any means, so the test does not apply. |
| 7.3.2.10.5 Database_Revision after changing an Object_Name | SKIP | The only changeable Object_Name is the Device's, changed by restarting the device with a new configuration. The test plan skips the test in that case. |
| 7.3.2.10.6 Database_Revision after changing an Object_Identifier | SKIP | The only changeable Object_Identifier is the Device's. The test plan skips the test in that case. |
| 9.22.1.3 Writing a Non-Commandable Property with a Priority | SKIP | Needs a writable property that is not commandable; this device has none. |
| 9.22.1.6 Writing NULL to Non-commandable Properties | SKIP | Needs a writable non-commandable Present_Value; this device has none, and the test plan skips the test. |
| 9.22.2.10 Resizing a writable fixed size array property | SKIP | Needs a writable fixed-size array; this device has none. |
| 9.22.2.11 Writing a Property Value Related to Non-supported Optional Functionality | SKIP | Needs a property tied to optional functionality the device does not support; this device has none. |

No public issue has been opened for any of these.
