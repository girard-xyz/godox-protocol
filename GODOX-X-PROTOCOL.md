# Godox X (2.4 GHz) Protocol Reference

Technical specifications for the Godox X 2.4 GHz flash trigger protocol, reverse
engineered from publicly available sources and RF/electrical signal analysis.

> **Disclaimer.** All information was obtained from publicly available sources
> (FCC filings, teardowns, manuals) or through analysis of emitted RF and
> electrical signals. No firmware was disassembled. No warranty is provided
> regarding accuracy or completeness. Product names may be registered
> trademarks; the author is not affiliated with their owners. See
> [Sources & References](#sources--references) for the origin of this material.

---

## 1. Physical / RF Layer

| Property | Value |
|---|---|
| Band | 2.4 GHz ISM |
| Modulation | MSK (Minimum Shift Keying) |
| Frequency range | 2.412999634 – 2.464499756 GHz |
| Symbol / data rate | ≈ 250 kHz (peaks spaced 250 kHz) |
| Radio transceiver | Texas Instruments CC2500 |
| MCU (observed) | Geehy F072 (Cortex-M) |
| Channels | 32 |
| Groups | up to 16 |
| Packet length | 4 bytes |
| Preamble | `0xAA` (alternating bits) |
| Sync word (observed) | `0xC3 0x68` |
| Encryption / pairing | none |

Notes:

- The CC2500 emits alternating preamble bits (`0xAA`), followed by 2 bytes
  `[Sync1 Sync2]`, optionally repeated as `[Sync1 Sync2 Sync1 Sync2]`.
- Per CC2500 datasheet §16.2: the MSK implementation *inverts* the sync word
  and data compared to e.g. signal generators — relevant when decoding raw SDR
  bitstreams.
- "All-shoot" mode lets multiple transmitters share flashes with no pairing,
  confirming there is no transmitter-specific encryption.
- No acknowledgements: the flashes do not reply to configuration messages.

A channel → center-frequency table is published in the Sekonic L-858 manual
(chapter 5) and corroborated by the FCC test report for the XProII (FCC ID
`2ABYN060`). The 20 dB bandwidth is documented in the same FCC report.

---

## 2. Message Framing

All messages are 4 bytes and use one of three shapes:

```
[0xA9, group,   setting, value]   # per-group configuration
[0xA9, 0x50,    setting, value]   # broadcast to all groups
[0xD5, trigger_code, x, x]        # global / trigger command
```

**Group codes** (`group` byte):

| Group byte | Meaning |
|---|---|
| `0x0A` … `0x0E` | Standard groups A, B, C, D, E (6 groups) |
| `0x0F, 0x01, 0x02, … 0x00` | Additional 10 groups |
| `0x50` | Broadcast (all groups) |

Broadcast transmissions are primarily used for TTL triggers and multi-mode
parameters.

---

## 3. Command Reference

### 3.1 Per-group configuration (`0xA9` prefix)

| Bytes | Name | Description |
|---|---|---|
| `[0xA9, group, 0xC0, 0x04]` | Prepare / reset | Seen during TTL sequences; signals a reset to allow clean configuration of power and mode. Broadcast (`0x50`). |
| `[0xA9, group, 0xB1, mode]` | Mode select | `0x0` = TTL (and enable); `0x1` = manual; `0x2` = multi strobe. Manual/multi combined with power `0xFF` disables the group. |
| `[0xA9, group, 0xBC, power]` | Manual power | `0x00` = full (1/1); each `+0x0A` (10 dec) reduces power by one EV (1/10-stop resolution). `0xFF` = disabled. |
| `[0xA9, group, 0xB9, power]` | TTL power | Higher value = higher output. `0x2D` = minimum pre-flash power reported by Olympus speedlights. Each `+0x0A` = one EV. |
| `[0xA9, group, 0xB2, zoom_mm]` | Zoom | Focal length in mm, **35 mm equivalent** (a 10 mm µ4/3 lens → `20`). |
| `[0xA9, group, 0xB3, hss]` | HSS enable | `0x1` = enable high-speed sync, `0x0` = disable. |
| `[0xA9, 0x50, 0xB4, 0x02]` | Selective trigger prepare | Issued before a trigger; lets the receiver issue preparatory commands. Broadcast. |
| `[0xA9, 0x50, 0xB4, 0x01]` | Selective trigger fire | TTL pre-flash fire command. Broadcast. |
| `[0xA9, group, 0xBD, power]` | Multi-mode manual power | Same encoding as `0xBC`; used in multi strobe mode. |
| `[0xA9, 0x50, 0xBE, count]` | Multi-mode pulse count | Number of strobe pulses. Broadcast. |
| `[0xA9, 0x50, 0xBF, frequency_hz]` | Multi-mode pulse frequency | Pulse frequency in Hz. Broadcast. |
| `[0xA9, group, 0xD3, enable]` | Modeling lamp (group) | `0x1` = ON, `0x0` = OFF. Both this and the global enable must be ON. |
| `[0xA9, group, 0xD6, proportional]` | Modeling lamp proportional | `0x1` = derive lamp power from flash power (ignore manual lamp power). |
| `[0xA9, group, 0xD1, power_pct]` | Modeling lamp manual power | Percentage of full power, `0x00`–`0x64` (0–100). |

### 3.2 Global / trigger commands (`0xD5` prefix)

| Bytes | Name | Description |
|---|---|---|
| `[0xD5, 0x81, x, x]` | Beep enable (global) | Enable flash beep for all flashes. |
| `[0xD5, 0x80, x, x]` | Beep disable (global) | Disable flash beep for all flashes. |
| `[0xD5, 0x81, x, x]` | Modeling lamp enable (global) | Globally enable modeling lamp control. |
| `[0xD5, 0x80, x, x]` | Modeling lamp disable (global) | Globally disable all modeling lamps. |
| `[0xD5, 0x11, x, x]` | Trigger, manual power | Fire using configured manual power. |
| `[0xD5, 0x09, x, x]` | Trigger, manual power (legacy hot shoe) | Fire using configured manual power. |
| `[0xD5, 0x19, x, x]` | Final trigger, TTL power | Final TTL flash fire. |
| `[0xD5, 0x21, x, x]` | Trigger release | Releases the trigger. |

> **Ambiguity in source:** beep enable (global) and modeling-lamp enable
> (global) are both documented as `0xD5 0x81`, and both disables as `0xD5 0x80`.
> Treat these as potentially context-dependent.

### 3.3 Power encoding

- Manual (`0xBC` / `0xBD`): `0x00` = 1/1, `+0x0A` per EV. `0xFF` = disabled.
- TTL (`0xB9`): higher = brighter, `+0x0A` per EV, `0x2D` = minimum.
- Resolution: **1/10 stop**.

---

## 4. Protocol Examples

### 4.1 TTL trigger of group A

```
0xA9, 0x50, 0xC0, 0x04
0xA9, 0x0A, 0xB1, 0x00
0xA9, 0x50, 0xB3, 0x00
0xA9, 0x0A, 0xB9, 0x2D
0xA9, 0x50, 0xB4, 0x02
0xA9, 0x50, 0xB4, 0x01
pre-flash fired
0xA9, 0x50, 0xC0, 0x04
0xA9, 0x0A, 0xB1, 0x00
0xA9, 0x50, 0xB3, 0x00
0xA9, 0x0A, 0xB9, 0x41
0xA9, 0x50, 0xB4, 0x02
0xA9, 0x50, 0xB4, 0x01
pre-flash fired
0xA9, 0x50, 0xC0, 0x04
0xA9, 0x0A, 0xB1, 0x00
0xA9, 0x50, 0xB3, 0x00
0xA9, 0x0A, 0xB9, 0x6E
0xA9, 0x50, 0xB4, 0x02
0xD5, 0x19, 0xFF, 0x7F
0xD5, 0x19, 0x8F, 0xFF
main flash fired
0xD5, 0x21, 0xF3, 0xF8
0xD5, 0x21, 0xFF, 0xFF
```

### 4.2 Manual power trigger of group A

```
0xA9, 0x50, 0xC0, 0x04
0xA9, 0x50, 0xB3, 0x00
0xA9, 0x50, 0xB4, 0x02
0xA9, 0x50, 0xC0, 0x04
0xA9, 0x50, 0xB3, 0x00
0xA9, 0x50, 0xB4, 0x02
0xA9, 0x50, 0xC0, 0x04
0xA9, 0x50, 0xB3, 0x00
0xA9, 0x50, 0xB4, 0x02
0xD5, 0x19, 0xFF, 0xFC
0xD5, 0x19, 0xEF, 0xFF
main flash fired
0xD5, 0x21, 0xE7, 0xEB
0xD5, 0x21, 0xFB, 0xFA
```

### 4.3 Configure manual power of group A (1/128 → 1/256, 1/10-stop steps)

```
0xA9, 0x0A, 0xBC, 0x47
0xA9, 0x0A, 0xBC, 0x48
0xA9, 0x0A, 0xBC, 0x49
0xA9, 0x0A, 0xBC, 0x4A
0xA9, 0x0A, 0xBC, 0x4B
0xA9, 0x0A, 0xBC, 0x4C
0xA9, 0x0A, 0xBC, 0x4D
0xA9, 0x0A, 0xBC, 0x4E
0xA9, 0x0A, 0xBC, 0x4F
0xA9, 0x0A, 0xBC, 0x50
```

### 4.4 Turn group A off

```
0xA9, 0x0A, 0xB1, 0x01
0xA9, 0x0A, 0xBC, 0xFF
```

### 4.5 Switch group A to TTL

```
0xA9, 0x0A, 0xB1, 0x00
0xA9, 0x0A, 0xBC, 0xFF
```

---

## Appendix — Measurement notes

Godox power levels were swept in 1/3 EV steps across −3 EV … +3 EV and found to
actually use **1/10-stop** resolution. A photodiode + logic analyzer setup
correlated camera→transmitter commands, received CC2500 radio commands, and
analog flash output simultaneously.

Related documents: [Interoperability with Olympus TTL](INTEROP.md) ·
[Olympus TTL Protocol Reference](OLYMPUS-TTL-PROTOCOL.md).

---

## Sources & References

- **Original article:** *Godox X receiver for Olympus FL-36 flash* by
  **Martin Sivák** — <https://marsik.codeberg.page/godox-olympus-protocol-article/>
  (last updated 2025-11-26). All Godox X specifications above originate here.
- **FCC filings:** Godox `2ABYN` device family — <https://fccid.io/2ABYN>;
  XProII test report — <https://fccid.io/2ABYN060> (frequency range, MSK
  modulation, 20 dB bandwidth).
- **Sekonic L-858 manual, ch. 5** — channel/frequency table —
  <https://images.salsify.com/image/upload/s--ydZZTNTN--/hfv6lkegqhecu6bdir9o.pdf>
- **Texas Instruments CC2500** datasheet (§16.2 sync/data inversion) —
  <https://www.ti.com/product/CC2500>
- **Teardown** revealing the CC2500 + Geehy F072 hardware —
  <https://www.edn.com/scrutinizing-a-camera-flash-transmitter/>
- **CC2500 module source** used for the reference receiver —
  <https://www.tinytronics.nl/en/communication-and-signals/wireless/rf/modules/cc2500-wireless-rf-module-2.4ghz>
