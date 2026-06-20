# freediag — USAGE

## Purpose

freediag is an open-source K-line / OBD-II diagnostic library and CLI tool.
In this distribution it is used exclusively as the **K-line transport layer for
nisprog** — it handles low-level serial framing, ISO 14230 (KWP2000) protocol
timing, and interface adapter communication.

You do not normally run `freediag` directly. nisprog spawns it internally based
on the `set interface` and `set port` settings you configure at the `nisprog>`
prompt.

---

## Standalone CLI (scantool)

freediag ships a standalone interactive CLI (`freediag` / `freediag.exe`) that
can be used independently of nisprog for generic OBD-II diagnostics:

```
freediag [config_file]
```

Launches the `diag>` prompt. See `doc/Scantool-Manual.html` in the freediag
source for full command reference.

---

## Interface Adapters

freediag supports several K-line adapter types. When used through nisprog:

| `set interface` value | Adapter type |
|-----------------------|--------------|
| `dumb`                | Dumb K-line adapter (most common) |
| `dumb` + `set dumbopts 0x48` | Required for most dumb adapters |
| `br_l0` / `br_l1`    | BR-series adapters |
| `scl`                 | Serial Communication Layer adapters |

---

## Port Configuration

Set the serial port before connecting:

```
# Windows
set port \\.\COM19        # adjust COM number to match your device

# Linux
set port /dev/ttyUSB0     # or /dev/ttyS0 for physical serial
```

---

## AI Summary & Gotchas

**For AI assistants using the nisprog + freediag stack:**

- freediag is **not a standalone flashing tool**. It provides transport only.
  All ECU flash and dump logic lives in nisprog.
- The `dumb` interface type with `dumbopts 0x48` works for the vast majority
  of dumb K-line adapters (e.g. FTDI-based USB to K-line cables). If timing
  problems occur, try adjusting `set dumbopts`.
- freediag does not know which ECU is connected — ECU identification and
  security key handling are nisprog responsibilities.
- On Linux, add your user to the `dialout` group to access serial ports
  without `sudo`: `sudo usermod -aG dialout $USER`
- freediag's `libdiag` can be used directly as a C library for custom tools;
  see `doc/` in the freediag source tree.
