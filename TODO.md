# TODO

Open items for this example at v1.4.1 (CAS BACnet Stack `6.x` @ `1e7dffe2b`).

## Known defects

None. Every check of the B-SA Profile Function and its Failure Modes passes
(`tests/`, see [tests/README.md](tests/README.md)), and the BTL run recorded in
[`tests/veribacnet/RESULTS.md`](tests/veribacnet/RESULTS.md) has no example-class
finding open.

## By design, not planned

A B-SA (ANSI/ASHRAE 135-2024 Annex L.4.4) is a simple device, so this example
deliberately does not implement:

- ReadPropertyMultiple, WritePropertyMultiple, SubscribeCOV (DS-RPM-B, DS-WPM-B,
  DS-COV-B): a client reads and writes one property at a time.
- DeviceCommunicationControl, TimeSynchronization and ReinitializeDevice.
- Alarming and event reporting, scheduling and trending.

## Test-only hook

The disconnected-valve Failure Mode is driven by the environment variable
`CAS_EXAMPLE_TEST_ACTUATOR_FAULT_FILE`; nothing on the network can disconnect the
simulated valve. When you port the example to hardware, set `Reliability` from your
output driver's own diagnostics (see README "The actuator Simulation").
