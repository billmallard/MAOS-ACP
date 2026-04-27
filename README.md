# MAOS-ACP — Autopilot Control Panel

A hardware panel controller and avionics bus bridge for the [MAOS](https://github.com/billmallard/MAOS-DESIGN) open-source experimental aircraft and the [pyEfis](https://github.com/makerplane/pyEfis) EFIS display system.

---

## What It Is

MAOS-ACP is a small embedded controller that provides physical knobs and buttons for functions that belong on an autopilot control panel — altitude select, heading bug, vertical speed, course, autopilot mode engagement — without requiring a touchscreen or keyboard.

It bridges two worlds:

- **FIX network** — the data bus used by pyEfis and the MAOS avionics stack. The panel writes selected values (ALT_SEL, HDG_SEL, VS_SEL, etc.) into the FIX database so any connected display or controller can act on them.
- **Certified avionics** — optionally bridges serial data to/from existing panel-mounted instruments such as Garmin GI-275 and Aspen Evolution displays, making it useful in mixed experimental/certified panels.

The panel controller itself runs on a small microcontroller (STM32 or Arduino-class) and connects to the host (Raspberry Pi or x86 SBC) via USB serial, presenting as a FIX network client.

---

## Hardware

### Reference Panel Build

| Component | Part | Qty | Notes |
|-----------|------|-----|-------|
| Rotary encoder | Bourns PEC11R-4215F-S0024 | 3–4 | 24-detent, push-button, 6mm shaft, panel-mount |
| Knob | Davies Molding 1100-A or similar | 3–4 | Standard 6mm D-shaft |
| Buttons | Momentary SPST, 0.5" panel | 4–8 | AP engage, mode select, etc. |
| Controller | STM32F103 ("Blue Pill") or Arduino Pro Micro | 1 | USB HID or USB serial |
| Panel | 3/32" 6061 aluminum, laser-cut or hand-drilled | 1 | 7/16" holes for Bourns encoders |

### Suggested Panel Layout

```
[ ALT ▲▼ ]  [ HDG ◄► ]  [ V/S ▲▼ ]  [ CRS ◄► ]

[ AP ENG ]  [ ALT HLD ]  [ HDG  ]  [ NAV  ]  [ VS  ]  [ IAS ]
```

Encoder push-button toggles between coarse and fine steps:
- **ALT**: outer = 1000 ft steps, push-toggle = 100 ft steps
- **HDG / CRS**: 10° / 1° steps
- **V/S**: 100 fpm / 50 fpm steps

---

## Software Architecture

```
Physical panel
  └─ Bourns encoders + buttons
       └─ STM32 / Arduino firmware (this repo)
            └─ USB serial ──► Raspberry Pi / SBC
                                  └─ FIX network client (Python)
                                       ├─ Writes: ALT_SEL, HDG_SEL, VS_SEL, CRS_SEL
                                       ├─ Writes: AP_ENG, AP_ALT, AP_HDG, AP_NAV, AP_VS
                                       └─ Reads:  current values for LED/display feedback
```

The FIX network client runs as a lightweight Python process on the host. The panel firmware communicates a simple line-oriented protocol over USB serial; the host-side client translates that into FIX database reads and writes.

---

## Avionics Bridge — Garmin GI-275 and Aspen Evolution

For builders with existing certified instruments alongside experimental avionics, MAOS-ACP can optionally act as a **serial bus bridge**, translating data between the FIX network and common avionics serial protocols.

### Garmin GI-275

The GI-275 is a TSO'd multi-function replacement instrument (ADI, HSI, EI modes) with RS-232 serial ports. In an experimental installation alongside pyEfis:

| Direction | Data | Protocol |
|-----------|------|----------|
| GI-275 → FIX | Attitude (pitch, roll), heading, airdata | Garmin aviation serial / ARINC 429 |
| FIX → GI-275 | GPS position, track, groundspeed | Aviation serial GPS format |
| ACP → GI-275 | Course, nav source select | Serial command |

This allows pyEfis to use GI-275 AHRS data as a redundant attitude source, and the GI-275 to display navigation data computed by the MAOS avionics stack.

### Aspen Evolution (EFD1000 / EFD500)

The Aspen Evolution series uses RS-232 serial for external data and can output its computed attitude and heading for autopilot use.

| Direction | Data | Protocol |
|-----------|------|----------|
| Aspen → FIX | Attitude, heading, airdata | Aspen serial output |
| FIX → Aspen | GPS, nav data | NMEA 0183 / Aviation format |

### Interface Hardware

For RS-232 avionics serial: a standard USB-to-RS-232 adapter or RS-232 transceiver (MAX3232) on the STM32 UART. ARINC 429 (if needed for deeper GI-275 integration) requires a dedicated interface such as a Condor Engineering DEI1016 or AIM ARINC 429 module.

---

## FIX Keys

Keys this panel reads and writes on the FIX network:

| Key | Direction | Description |
|-----|-----------|-------------|
| `ALT_SEL` | Write | Selected altitude (ft) |
| `HDG_SEL` | Write | Selected heading (deg magnetic) |
| `VS_SEL` | Write | Selected vertical speed (fpm) |
| `CRS_SEL` | Write | Selected course (deg) |
| `AP_ENG` | Write | Autopilot engage (bool) |
| `AP_MODE` | Write | Active AP lateral mode |
| `ALT` | Read | Current altitude (for display feedback) |
| `HEAD` | Read | Current heading |
| `VS` | Read | Current vertical speed |

---

## Relation to MAOS

| Project | Role |
|---------|------|
| [MAOS-FCS](https://github.com/billmallard/MAOS-FCS) | Fly-by-wire flight control system; ACP feeds it mode commands and selected references |
| [MAOS-DESIGN](https://github.com/billmallard/MAOS-DESIGN) | Airframe design; ACP is a panel-level component |
| [pyEfis](https://github.com/makerplane/pyEfis) | EFIS display; reads ALT_SEL, HDG_SEL etc. from FIX for bug display and alerting |

---

## Status

Early planning stage. Contributions and discussion welcome.

## License

GNU General Public License v2 — consistent with pyEfis and the broader MAOS stack.
