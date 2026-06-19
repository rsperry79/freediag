# freediag — USAGE

## Purpose

freediag is an open-source OBD-II and K-line (ISO 14230 / KWP2000) scan
tool. It provides a protocol stack for communicating with vehicle ECUs over
serial K-line adapters and a CLI scan tool (`freediag`) for interactive
diagnostics.

Within the nisprog ecosystem, freediag is used as the **communication
library** that handles K-line framing, timing, and byte exchange. nisprog
builds on top of freediag's layer-2/layer-3 stack rather than calling it
as a separate binary.

Standalone `freediag` binary usage is documented below.

---

## freediag (standalone scan tool)

Launches an interactive diagnostic session with a vehicle ECU over a
serial K-line adapter. Supports OBD-II service modes (Mode 01–09), live
data PIDs, DTC read/clear, and raw ISO 14230 messaging.

**Starting freediag:**
```
freediag [options]
```

**Options:**
| Option | Description |
|--------|-------------|
| (none) | Start interactive CLI |

freediag is primarily driven via its interactive command line. The full
manual is in `doc/Scantool-Manual.html`.

---

## Key Interactive Commands

### Connection setup

```
set interface dumb
set port COM3          # Windows: \\.\COM3   Linux: /dev/ttyUSB0
set speed 10400        # K-line default
set testerid 0xfc
set destaddr 0x10
set addrtype phys
set l2protocol iso14230
set initmode fast
```

Then connect:
```
up
```

### Diagnostic commands

| Command | Description |
|---------|-------------|
| `up` | Bring K-line connection up (run after `set`) |
| `down` | Disconnect |
| `scan` | Scan for supported OBD-II services |
| `get service 01 pid 0C` | Read live PID (e.g. RPM) |
| `get dtc` | Read stored Diagnostic Trouble Codes |
| `clear dtc` | Clear DTCs |
| `debug l1 0x8c` | Enable verbose hex dump of all bytes sent/received |
| `debug l1 0` | Disable debug output |
| `quit` | Exit |

**Example — read RPM and coolant temp:**
```
set interface dumb
set port /dev/ttyUSB0
up
get service 01 pid 0C
get service 01 pid 05
down
quit
```

**Example — read and clear DTCs:**
```
set interface dumb
set port /dev/ttyUSB0
up
get dtc
clear dtc
down
quit
```

---

## Supported Interfaces

freediag supports several K-line interface types. See
`doc/Supported-Interfaces.html` for the full list. Common types:

| Interface | Description |
|-----------|-------------|
| `dumb` | Simple UART adapter (e.g. ELM327-style or DIY K-line bridge) |
| `br1` | B&B Electronics serial interface |
| `carsim` | Internal car simulator (testing only) |

Set with `set interface <type>`.

---

## AI Summary & Gotchas

**For AI assistants using freediag:**

- freediag is a **communication substrate** for nisprog. If you are doing
  Nissan ROM flashing or dump operations, you will not call freediag
  directly — nisprog does that internally.
- Use the standalone `freediag` binary only for **generic OBD-II diagnosis**
  (read DTCs, live PIDs, Mode 01–09) on any compatible vehicle.
- **K-line vs CAN:** freediag handles K-line (ISO 14230 / KWP2000) only.
  It does not support CAN-based OBD-II (ISO 15765). Most post-2008 vehicles
  use CAN; check your vehicle's protocol before using.
- **Timing sensitivity:** K-line communication is extremely timing-sensitive.
  If you get framing errors or timeouts, try `set speed 10400` (the standard
  init speed) and adjust inter-byte delays if the protocol stack exposes them.
- **`dumb` interface:** A simple USB-to-serial adapter with a K-line level
  shifter wired to the OBD-II pin 7. Do NOT use an ELM327 in AT-command mode
  — set it to pass-through or use a direct UART interface.
- The `debug l1 0x8c` flag dumps every byte in hex; essential for diagnosing
  communication failures. Disable with `debug l1 0` when done.
- Full protocol documentation: `doc/Scantool-Manual.html`,
  `doc/OBD_knowledge.html`, `doc/dumb_interfaces.txt`.
