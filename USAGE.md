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

Launches the `diag>` prompt. If no config file is given, freediag looks for
a `.rcfile` or `freediag.ini` in the current directory. See
`doc/Scantool-Manual.html` in the freediag source for full command reference.

---

## Interface Adapters

freediag supports several K-line adapter types, selected via `set interface` in
nisprog. Use the exact names below — they are matched case-insensitively.

| `set interface` | Hardware | L1 protocols supported |
|-----------------|----------|------------------------|
| `DUMB` | Any dumb serial K-line cable (FT232, CP2102, CH340, optocoupler) | ISO9141, ISO14230, RAW |
| `ELM` | ELM327-based adapters | ISO9141, ISO14230, J1850-VPW, J1850-PWM, CAN |
| `BR1` | B. Roadman BR-1 | ISO9141, ISO14230, J1850-VPW, J1850-PWM |
| `MET16` | Multiplex Engineering T16 | ISO9141, ISO14230, J1850-VPW, J1850-PWM |
| `DUMBT` | Dumb interface test driver | All (test only — not for ECU comms) |

**Selection rule:** if the cable is a USB-serial bridge (FT232, CP2102, CH340,
PL2303) with nothing else on board, it is `DUMB`. If it has an ELM327 chip
(marked on PCB or in product listing), use `ELM`. When uncertain, `DUMB` is
almost always correct.

**ELM327 limitation:** ELM327 is a smart interpreted adapter and is very slow.
Flash and dump operations (`flrom`, `dm` via kernel) are not supported through
ELM327.

---

## `dumbopts` — dumb adapter option flags

When `interface DUMB` (or `DUMBT`) is selected, `set dumbopts` controls
adapter behaviour. It is a bitmask — set it to the sum of the flags you need.

| Flag | Value | Description |
|------|-------|-------------|
| `USE_LLINE` | `0x01` | Drive the L-line (RTS pin) during 5-baud init. Only for cables with a separate L-line wire (some VAGtool clones). |
| `CLEAR_DTR` | `0x02` | Hold DTR low (negative voltage) continuously. Unusual; default is DTR high. |
| `SET_RTS` | `0x04` | Hold RTS high continuously. Unusual. Do not combine with `USE_LLINE`. |
| `MAN_BREAK` | `0x08` | Force software-bitbanged 5bps break pulses. **Essential for USB-serial bridges** (CP2102, CH340, FT232). |
| `LLINE_INV` | `0x10` | Invert L-line polarity. Use only with `USE_LLINE`; very rare. |
| `FAST_BREAK` | `0x20` | Alternate fast-init: sends `0x00` at 360 bps (≈25 ms) instead of a hardware break. Try if `0x48` fails with `initmode fast`. |
| `BLOCKDUPLEX` | `0x40` | Remove echoed bytes at the message level (half-duplex cleanup when P4=0). |

Default value: `0x48` (`MAN_BREAK | BLOCKDUPLEX`) — correct for the vast
majority of USB K-line cables. Try `0x68` if fast init fails with `0x48`.

---

## Port Configuration

Set the serial port before connecting:

```
# Windows
set port \\.\COM19        # adjust COM number to match your device

# Linux
set port /dev/ttyUSB0     # or /dev/ttyS0 for physical serial
```

On Linux, add your user to the `dialout` group to access serial ports without
`sudo`:

```
sudo usermod -aG dialout $USER
```

Log out and back in for the group change to take effect.

---

## Protocol layers

freediag implements a layered protocol stack used internally by nisprog:

| Layer | Set via | Valid values | Notes |
|-------|---------|--------------|-------|
| L0 | `set interface` | `DUMB`, `ELM`, `BR1`, `MET16`, `DUMBT` | Hardware driver |
| L1 | `set l1protocol` | `ISO9141`, `ISO14230`, `J1850-VPW`, `J1850-PWM`, `CAN`, `RAW` | Physical framing |
| L2 | `set l2protocol` | `ISO14230`, `ISO9141`, `RAW`, `CAN`, `SAEJ1850`, `VAG` | Session protocol |
| initmode | `set initmode` | `FAST`, `5BAUD`, `CARB` | Bus wake-up sequence |

For Nissan MEC07 ECUs: `l1protocol ISO14230`, `l2protocol ISO14230`,
`initmode FAST`. Set both L1 and L2 explicitly — relying on auto-selection
has been observed to cause connection failures on some adapters.

Use `set l1protocol ?`, `set l2protocol ?`, or `set initmode ?` to list
compiled-in choices for your build.

---

## Adapter signal verification: `l0test`

Before connecting to a vehicle, use the `DUMBT` driver and `debug l0test` to
verify K-line signal levels and timing on the bench:

```
set interface DUMBT
set port \\.\COM19         # Windows — adjust port
set dumbopts 0x08          # MAN_BREAK only for test
debug
l0test ?                   # list available tests
l0test 1                   # send fast pulses on TXD (K-line polarity check)
```

`l0test` sends controlled pulses on TXD, RTS, and DTR so you can verify
level-shifting with a multimeter or oscilloscope before connecting to the ECU.
**Disconnect from vehicle before running l0test.**

---

## Module scanning: `diag fastprobe`

In the freediag standalone CLI (`diag>` prompt), `fastprobe` scans a range of
addresses using ISO14230 fast init and reports which modules respond:

```
diag fastprobe 0x10 0x50        # scan addresses 0x10–0x50 (physical addressing)
diag fastprobe 0x10 0x50 func   # same with functional addressing
```

Useful for discovering which KWP2000 modules are present on the bus before
setting `destaddr` in nisprog. In nisprog, the same command is available via
`diag fastprobe`.

---

## AI notes

- **freediag is not a standalone flashing tool.** It provides transport only.
  All ECU flash and dump logic lives in nisprog.
- **`DUMB` + `dumbopts 0x48`** works for the vast majority of USB K-line
  cables and is the correct starting point. `br_l0`, `br_l1`, and `scl` are
  not valid interface names — the correct names are `DUMB`, `ELM`, `BR1`,
  `MET16`, `DUMBT`.
- **ELM327 cannot be used for flash or fast dump.** Only `DUMB`-family
  adapters support the kernel upload and block operations nisprog requires.
- **DUMB supports ISO9141, ISO14230, and RAW at L1.** ELM also supports CAN
  and J1850 variants.
- freediag does not know which ECU is connected — ECU identification and
  security key handling are nisprog responsibilities.
- freediag's `libdiag` can be used directly as a C library for custom tools;
  see `doc/` in the freediag source tree.
