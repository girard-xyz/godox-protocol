# Godox X & Olympus TTL Protocol Reference

Reverse-engineered protocol reference for the **Godox X 2.4 GHz** flash trigger
system and the **Olympus hot-shoe TTL** flash protocol.

## Overview

Recent Godox triggers (XProII, X1T-S, …) speak a 2.4 GHz protocol, but Godox does
not ship a receiver for Olympus speedlights (the X1R-O variant is only mentioned
in a manual). This repository documents both protocols at the byte level so a
custom receiver can bridge Godox X triggers to Olympus flashes such as the
FL-36/FL-36R.

The Godox X side covers the RF layer, message framing, per-group and global
commands, and the power/mode/zoom/HSS encoding. The Olympus side covers the
hot-shoe electrical interface, the 20,800 baud link-level protocol, and the
command set used for TTL pre-flash and main-flash control. A bridging document
describes how to translate between the two.

## Documents

| File | Contents |
|---|---|
| [`GODOX-X-PROTOCOL.md`](GODOX-X-PROTOCOL.md) | RF layer, message framing, command reference, and protocol examples for Godox X. |
| [`OLYMPUS-TTL-PROTOCOL.md`](OLYMPUS-TTL-PROTOCOL.md) | Physical/electrical layer, command reference, and examples for Olympus hot-shoe TTL. |
| [`INTEROP.md`](INTEROP.md) | Godox X ↔ Olympus TTL translation: power mapping, EV-step conversion, HSS/modeling-lamp caveats. |

## Protocol highlights

**Godox X**

- 2.4 GHz, MSK modulation (TI CC2500 transceiver), ≈ 250 kHz data rate.
- 4-byte packets; preamble `0xAA`, observed sync word `0xC3 0x68`; no pairing or encryption.
- Per-group `0xA9` commands and global/trigger `0xD5` commands; broadcast group `0x50`.
- Power resolution of 1/10 stop.

**Olympus TTL**

- Hot-shoe: ISO center contact + four Aux contacts (Canon-compatible positions).
- Full-duplex asynchronous serial, 20,800 baud, 8N1, open-drain signaling.
- Commands include `0x82` (status), `0x87` (camera TTL info), `0x02` (pre-flash power), `0x03` (main-flash power).
- EV resolution of 8 steps per stop.

## Disclaimer

All information was obtained from publicly available sources (FCC filings,
teardowns, manuals) or through analysis of emitted RF and electrical signals.
**No firmware was disassembled.** No warranty is provided regarding the accuracy
or completeness of this information; several interpretations are educated
guesses. Product names may be registered trademarks; this project is not
affiliated with Godox, Olympus, or any other vendor.

## Attribution

The protocol analysis and measurements documented here originate from the article
[*Godox X receiver for Olympus FL-36 flash*](https://marsik.codeberg.page/godox-olympus-protocol-article/)
by **Martin Sivák** (last updated 2025-11-26). This repository is a derivative
reformatted into per-protocol reference documents. Please consult and credit the
original work.

## References

- Original article — <https://marsik.codeberg.page/godox-olympus-protocol-article/>
- Raw protocol capture samples — <https://codeberg.org/MarSik/godox-olympus-protocol-article/src/branch/pages/steps/logic>
- FCC filings (Godox `2ABYN`) — <https://fccid.io/2ABYN>
- Texas Instruments CC2500 datasheet — <https://www.ti.com/product/CC2500>
- Teardown (CC2500 / Geehy F072) — <https://www.edn.com/scrutinizing-a-camera-flash-transmitter/>
- Olympus FL-36R manual — <http://resources.olympus-europa.com/imaging//Manuals/AW2007/FL-36R_MANUAL_EN.pdf>

## License

Released under the [Creative Commons Attribution-ShareAlike 4.0 International
License](https://creativecommons.org/licenses/by-sa/4.0/). See [`LICENSE`](LICENSE).
