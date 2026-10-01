# Olympus TTL Protocol Reference

Technical specifications for the Olympus hot-shoe TTL flash protocol, reverse
engineered from publicly available sources and electrical signal analysis.

> **Disclaimer.** All information was obtained from publicly available sources
> (manuals, teardowns) or through analysis of emitted electrical signals. No
> firmware was disassembled. No warranty is provided regarding accuracy or
> completeness. Several interpretations are based on educated guesses that
> appear correct but may be partially or significantly inaccurate. Product names
> may be registered trademarks; the author is not affiliated with their owners.
> See [Sources & References](#sources--references) for the origin of this
> material.

---

## 1. Physical / Electrical Layer

### 1.1 Physical interface

The Olympus hot shoe uses the ISO standard center trigger contact plus four
additional small contacts in the **same positions as Canon** hot shoes (with
different functions). This allows a Canon extension cable (e.g. JJC FC-E3) to
be used for off-camera Olympus speedlights.

Pin functions (safety-pin-hole side = "top"):

| Pin | Function / characteristic |
|---|---|
| Bottom right (camera side) | 3.3 V, short-circuit current 313 µA → ≈ 10 kΩ pull-up |
| Top right (camera side) | No voltage |
| Top right (flash side, FL-36) | 5 V, 100 µA → ≈ 50 kΩ pull-up |
| Bottom left | 5 V, 100 µA on flash |
| Center trigger | 5 V, 100 µA on flash |
| Top left (power) | Present only on newer camera bodies |

### 1.2 Link-level protocol

- **Full-duplex asynchronous serial, 20,800 baud, 8N1.**
- All signals are **open-drain**: the receiving side supplies a pull-up to its
  own logic-high voltage; the sending side pulls the line low to transmit a
  logic zero. Applies to every signal including the trigger.

JJC FC-E3 cable wire colors (shield = ground):

| Color | Signal |
|---|---|
| White | Power |
| Red | CameraRx (flash → camera) |
| Yellow | Trigger center pin |
| Black | CameraTx (camera → flash) |
| Green | Sense |

---

## 2. Command Reference

The camera initiates every exchange by sending (likely) a baud-rate calibration
byte `0x55`, then a command byte, optional parameter bytes, and sometimes a
checksum byte.

**Checksum rule (where present):** the lowest 8 bits of the sum of *all command
bytes excluding the `0x55` preamble* plus *all response bytes*.

### 2.1 `0x82` — Flash status query

Camera: `[0x55, 0x82]` → Flash responds 9 bytes: `[B0 … B7, CHSUM]`.

Transmitted frequently (E-30: every 300 ms).

| Byte / bits | Meaning |
|---|---|
| `B0[7]` | Charged and ready |
| `B0[6..4]` | Flash mode: `0x0` TTL (incl. FP TTL), `0x6` manual (incl. FP), `0x4` auto |
| `B0[3..0]` | Always `0xC` |
| `B1[7]` | 1 when flash points forward/slightly down, else 0 |
| `B1[6..0]` | Max power level for the `0x03` trigger |
| `B2[6..0]` | First pre-flash power for the `0x02` trigger |
| `B3[2]` | HSS enabled |
| `B4[7]` | Post-flash status (possible "good exposure" indicator) |
| `B4[6..0]` | Max power for HSS (`0x03`); varies with shutter speed. Godox XProII always `0x46` |
| `B5` | Unknown = `B4[6..0] − 0x20`. Godox XProII always `0x27` |
| `B6` | Unknown. FL-36R `0x02`, TT685II `0x03`, XProII `0xE7` |
| `B7` | Unknown. Godox always `0x00`; FL-36R = `B1[6..0] + 0x20` |

`B1[6..0] − B2[6..0]` is consistently 56 = 8 steps/EV × 7 EV (1/1 → 1/128).

### 2.2 `0x86`

Camera: `[0x55, 0x86]` → Flash responds 7 bytes: `[B0 … B5, CHSUM]`.
Function unknown.

### 2.3 `0x87` — Camera TTL information

Camera: `[0x55, 0x87, B0 … B9, CHSUM]` → Flash responds `[0x5A]`.

Receiving this switches the flash to TTL mode the first time. Conveys shutter
speed, focal length, aperture and ISO.

| Byte | Meaning |
|---|---|
| `B0` | ISO: `0x20` = ISO 200, each `+0x08` = +1 EV (`0x18`=100, `0x28`=400, `0x30`=800) |
| `B1` | Aperture: `0xA0` = F4.0, each `+0x08` = 1 EV closed (`0xA8`=F5.6, `0xB0`=F8, `0xC0`=F16) |
| `B2` | Zoom: 14–15 mm=`0x33`, 16–18=`0x34`, 19–20=`0x35`, 25=`0x36`, 30=`0x37`, 35=`0x38`, 42=`0x39`, 45=`0x39`, 108=`0x3F` (max) |
| `B3` | `0x1F` before fire, `0x1C` after fire, `0xDF` idle |
| `B4` | Does not change with zoom |
| `B5` | Camera-side flash exposure compensation: `0x20`=0 EV, `+0x08`=+1 EV, `0x18`=−1 EV, `0x08`=−3 EV |
| `B6` | `0x01` before / right after fire, `0x3D` idle |
| `B7` | Shutter speed: `0x00`=1/60 s, each `+0x10`=1 EV faster (`0x10`=1/125, `0x20`=1/250); longer than 1/60 s always `0x00` |
| `B8` | Summary exposure value (for flash distance display). Non-HSS: `203 + flash_max_power − aperture`. HSS: `203 − 0x10 + flash_max_power − aperture` |
| `B9` | Same as `B0` |

### 2.4 `0x88`

Camera: `[0x55, 0x88]` → Flash responds 13 bytes: `[B0 … B11, CHSUM]`.
Function unknown.

### 2.5 `0xD6` / `0xD7` / `0xD8` — Initial interrogation only

| Command | Camera sends | Flash responds | Notes |
|---|---|---|---|
| `0xD6` | `[0x55, 0xD6, 0x55, 0xAA]` | 9 bytes `[B0 … B7, CHSUM]` | Function unknown; observed only during initial interrogation |
| `0xD7` | `[0x55, 0xD7, 0x55, 0xAA]` | 5 bytes `[B0 … B4, CHSUM]` | Function unknown; init only |
| `0xD8` | `[0x55, 0xD8, 0x55, 0xAA]` | 5 bytes `[B0 … B4, CHSUM]` | Function unknown; init only |

For these, the checksum excludes only the *first* `0x55` preamble.

### 2.6 `0x02` — Pre-flash power

Camera: `[0x55, 0x02, power, CHSUM]` → Flash responds `[0x5A]`.

- First pre-flash power matches `B2[6..0]` from the `0x82` response.
- Second pre-flash power is always `+0x10` (16 dec = 2 EV) relative to the first.

### 2.7 `0x03` — Main-flash power

Camera: `[0x55, 0x03, power, CHSUM]` → Flash responds `[0x5A]`.

- Power increases by `0x08` per EV stop.
- Bit `power[7]` (MSB) set ⇒ high-speed sync flash required.
- First pre-flash uses a fixed low power (likely 1/128); the flash can go lower
  via camera-side flash exposure compensation.

> The checksum for `0x02`/`0x03` excludes the `0x55` preamble; the flash's `0x5A`
> reply has no checksum.

---

## 3. Protocol Examples

A flash must first be configured to a TTL mode (TTL or FP TTL). It switches to
TTL automatically on first receipt of the `0x87` command (or can be set manually
on the flash afterward).

### 3.1 Flash initialization

```
0x55	0x82
0x4C	0x4D	0x95	0x02	0x3D	0x1D	0x02	0x2D	0x3B
0x55	0x86
0x05	0x61	0x0A	0x00	0x00	0x00	0xF6
0x55	0x88
0x02	0x11	0x11	0x10	0x10	0x10	0x10	0x10	0x10	0x00	0x00	0x00	0x0C
0x55	0xD6	0x55	0xAA
0x54	0x00	0x46	0x00	0x00	0x30	0x00	0x00	0x9F
0x55	0xD7	0x55	0xAA
0x10	0x03	0x00	0x00	0xE9
0x55	0xD8	0x55	0xAA
0x01	0x09	0x69	0x36	0x80
0x55	0x87	0x70	0xFF	0x33	0xDF	0x00	0x21	0x3D	0x00	0x00	0x70	0xD6
0x5A
```

### 3.2 TTL power flash trigger sequence

```
0x55	0x82
0x8C	0x4D	0x95	0x00	0x3D	0x1D	0x02	0x2D	0x79
0x55	0x87	0x20	0xA8	0x38	0x1F	0x2E	0x20	0x01	0x20	0x78	0x20	0xAD
0x5A
0x55	0x02	0x15	0x17
0x5A
preflash 1
0x55	0x02	0x15	0x17
0x5A
preflash 2
0x55	0x03	0x00	0x03
0x5A
final flash trigger
0x55	0x87	0x20	0xA8	0x38	0x1C	0x2E	0x20	0x01	0x20	0x78	0x20	0xAA
0x5A
0x55	0x86
0x05	0x61	0x0A	0x00	0x00	0x00	0xF6
0x55	0x82
0x8C	0x4D	0x95	0x00	0x3D	0x1D	0x02	0x2D	0x79
0x55	0x87	0x20	0xA8	0x38	0xDF	0x2E	0x20	0x3D	0x20	0x78	0x20	0xA9
0x5A
```

---

## Appendix — Measurement notes

Olympus power/checksum relationships were derived from hundreds of captured
exchanges varying shooting mode, shutter, aperture, zoom, focal length, exposure
compensation, and flash exposure compensation, including lens-cap (fully dark)
captures at max power. A photodiode + logic analyzer setup correlated camera→
transmitter commands, received CC2500 radio commands, and analog flash output
simultaneously. The author notes that the aging E-30 did not produce correctly
exposed images in TTL over the extension cable despite apparently correct
protocol recordings (which always commanded full power).

Related documents: [Godox X Protocol Reference](GODOX-X-PROTOCOL.md) ·
[Interoperability with Godox X](INTEROP.md).

---

## Sources & References

- **Original article:** *Godox X receiver for Olympus FL-36 flash* by
  **Martin Sivák** — <https://marsik.codeberg.page/godox-olympus-protocol-article/>
  (last updated 2025-11-26). All Olympus TTL specifications above originate here.
- **Olympus FL-36R manual** (shutter-speed/HSS power table on p. 44) —
  <http://resources.olympus-europa.com/imaging//Manuals/AW2007/FL-36R_MANUAL_EN.pdf>
- **JJC FC-E3 off-camera flash cable** — used to intercept the protocol.
- **Raw protocol capture samples** (logic analyzer traces) —
  <https://codeberg.org/MarSik/godox-olympus-protocol-article/src/branch/pages/steps/logic>
