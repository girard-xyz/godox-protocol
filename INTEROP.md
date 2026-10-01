# Interoperability: Godox X ↔ Olympus TTL

Bridge notes for translating Godox X 2.4 GHz trigger commands into Olympus
hot-shoe TTL commands (e.g. a custom receiver driving an Olympus FL-36/FL-36R).

See the companion references:
[Godox X Protocol Reference](GODOX-X-PROTOCOL.md) ·
[Olympus TTL Protocol Reference](OLYMPUS-TTL-PROTOCOL.md).

---

## Manual power

Whenever the receiver sees a Godox manual-power setting (`0xBC`), forward it to
the flash using the Olympus `0x03` command.

Power translation uses:

- the auto-detected maximum flash power (byte `B1` from the Olympus `0x82`
  status response), and
- the differing EV step sizes: **Olympus = 8 steps per EV, Godox = 10 steps per
  EV.**

## TTL control

The Godox transmitter uses a single command (`0xB9`) for both pre-flash and
main-flash power settings, whereas Olympus distinguishes them (`0x02` for
pre-flash, `0x03` for main).

Working algorithm:

1. Assume **two pre-flashes** will occur.
2. Use Olympus `0x02` until two trigger commands (`0xB4 0x01`) have been received.
3. After the second pre-flash trigger, switch to Olympus `0x03` for the main
   flash until the quench/release command arrives (`0xD5 0x21`).

## High-speed sync — unresolved

The Godox XProII (and likely other transmitters) does **not** transmit shutter
speed to the flashes. Olympus speedlights require shutter-speed information to
compute the appropriate maximum power level (and possibly to configure HSS pulse
duration/repetition rate). This issue remains unresolved in the reference
implementation.

Observed behaviour: HSS triggers a long burst sequence; the maximum reported
power changes with shutter speed, but the flash burst is always ≈ 8 ms long.

## Modeling lamp — not working

The Godox modeling-lamp control was mapped to the Olympus focus lamp. Despite a
believed-correct command being identified, the FL-36R did not respond.

## Missing / unimplemented features

- **Godox transmitter ID:** a transmitter configured with a specific ID does not
  synchronize with the receiver. Suspected cause: the CC2500 sync bytes change
  based on the ID. Not yet investigated.
- **Godox multi-flash mode:** capturing multiple exposures of a moving subject
  within a single frame is not implemented.

---

## Sources & References

- **Original article:** *Godox X receiver for Olympus FL-36 flash* by
  **Martin Sivák** — <https://marsik.codeberg.page/godox-olympus-protocol-article/>
  (last updated 2025-11-26).
