# SDRplay RX Bridge — Release Notes

## v0.99.2 — beta (2026-04-29)

- **Higher-contrast IF / transverter-offset readout** (tester request).
  The `IF tune: …  (offset …)` line under the main dial display now
  renders in bold red (`#ff3333`) so the SDR-tune-vs-dial relationship
  is unmissable on every glance. Same content as v0.99.1, just better
  contrast.

## v0.99.1 — beta (2026-04-29)

- **Transverter offset** (tester request — 10368 MHz operation through a
  144 MHz IF transverter). New Settings → "Transverter offset" field
  (signed MHz, kHz precision). The SDR is tuned to *(dial + offset)*
  while the GUI, WSJT-X, QMAP, and the LinradServer header all keep
  showing the operating dial. Example: dial 10368 MHz, offset
  −10224 MHz, SDR actually tunes to 144 MHz. The main-window frequency
  display gains a small "IF tune: 144.000000 MHz (offset −10224.000000 MHz)"
  line below the big dial readout whenever the offset is non-zero, so
  the operator can confirm at a glance the SDR is tuned where they
  expect.
- Persisted INI key: `radio/transverter_offset_hz` (int64, Hz).
- New CLI flag: `--transverter-offset <MHz>` (signed double).

## v0.99.0 — first beta (2026-04-29)

### Initial scope

| RSP model | Status in v0.99.x |
|---|---|
| **RSPduo** (single-tuner, Tuner A) | **Working** — verified on N6NU's bench. Full feature parity (RF / DAB notches, Tuner B bias-T toggles wired). |
| **RSPdx** | **Working** — beta-tester verified at 10 GHz EME via 144 MHz IF transverter. Streams, decodes; antenna defaults to Antenna A; bias-T / RF notch / DAB notch / HDR mode toggles are not yet wired (no-ops in v0.99.x — coming in a follow-up). |
| RSP1A / RSP1B / RSPdx-R2 / RSP2 / RSP1 | Should boot and stream — share the same code path. Untested at first ship. Reports welcome. Per-model bias-T / notch / antenna toggles will land as testers report on each. |

Dual-tuner / master-slave / diversity RX modes on the RSPduo are out of
scope for v0.99.x — single-tuner Tuner A only.

First release of the **SDRplay RX Bridge** — a Windows-only companion app
that lets a 14-bit SDRplay receiver add wideband Q65 (QMAP) reception
alongside an existing real radio (IC-905, IC-705, FT-991A, etc.) without
disrupting the real radio's TX setup. Sibling to the **HackRF RX Bridge**
and **RTL-SDR RX Bridge** — same shape, same WSJT-X / QMAP integration,
different (much higher dynamic range) front end.

### Use case

Your real rig handles TX and narrowband RX as it always has, controlled
by WSJT-X via Hamlib. An **SDRplay** — fed from a splitter on the same
antenna, or a separate RX-only antenna — runs alongside the real rig as
a **wideband observer**. This bridge:

- Listens to **WSJT-X UDP messages** (port 2237 by default) for the
  current dial frequency, mode, and transmit state
- Tunes the SDRplay to match
- Demodulates SSB to **VB-Audio Virtual Cable Line 1** so WSJT-X's
  "Sound input (RX)" sees the SDRplay audio
- Streams **96 kHz IQ to QMAP** (UDP 50004) for wideband Q65 decode
- Stops SDRplay RX during WSJT-X TX so the local-TX bleed-through can't
  slam the front end, and (optionally) drives the bias tee on/off to
  match WSJT-X's TX state for transverter sequencer / LNA power
  *(RSPduo only — bias-T sits on Tuner B's SMA)*

WSJT-X CAT continues to control the real radio. WSJT-X's "Sound output
(TX)" still goes to the rig's USB audio interface as before. Only the
**RX audio path** is replaced with bridge-fed audio from the SDRplay.

### What an SDRplay brings to weak-signal work

The RSPduo and its siblings use a 14-bit ADC after a Mirics MSi001 / 2500
front end — substantially better dynamic range than the 8-bit RTL-SDR
(~120 dB native vs ~50 dB) and meaningfully better than the HackRF too.
That headroom is most useful in strong-RF environments, near broadcast
sites, or with high-gain antennas where 8-bit SDRs run out of room
under modest gain settings.

- **Frequency range**: 1 kHz – 2 GHz native on the SMA path, full
  performance from 10 MHz up. Covers HF, 6 m, 2 m, 70 cm, 23 cm.
- **2 Msps** (matches the bridge-core pipeline) with 1.536 MHz analog
  bandwidth — wide enough for the 96 kHz QMAP wideband stream.
- **Hardware DC offset and I/Q imbalance correction** — enabled by
  default. The bridge-core IqBalancer is downstream of these and is
  usually a no-op on RSPduo.
- **Bias tee (RSPduo Tuner B SMA)** — drives an external LNA or
  transverter sequencer at 4.7 V. Manual toggle plus optional
  "follow WSJT-X TX state" behavior.

For the curious: the bridge's downstream DSP (`SsbDemodulator`,
`LinradServer`, `IqBalancer`) is currently 8-bit-IQ internally — the
top 8 bits of each 14-bit ADC sample feed the pipeline. A 16-bit-wide
internal path is on the BACKLOG.

### Installation note (read first)

The installer is **not code-signed** and is **64-bit only**
(Windows 10 / 11 x64). On first launch on a fresh Windows machine you
will see Microsoft Defender SmartScreen warn:

> Windows protected your PC.
> Microsoft Defender SmartScreen prevented an unrecognized app from
> starting.

Click **More info → Run anyway**. You should only see this once per
binary. The same warning may appear once on the installed
`sdrplay-rx-bridge.exe`; handle it the same way.

### What you'll need to install separately

- **SDRplay API package** — free download from
  <https://www.sdrplay.com/api/>. Installs `sdrplay_api.dll` into
  `C:\Program Files\SDRplay\API\x64\` and registers the
  `SDRplayAPIService` Windows service that mediates hardware access.
  *Required* — the bridge will refuse to open the SDRplay without it.
  This bridge does **not** bundle the SDRplay API DLL (per the SDRplay
  redistribution agreement).
- **VB-Audio Virtual Cable** — <https://vb-audio.com/Cable/>. Provides
  the `Line 1` virtual sound device the bridge feeds.
- **WSJT-X 2.7+** — for FT8 / FT4 / Q65 narrowband decoding.
- **QMAP 0.6+** — for wideband Q65. Set Network input = enabled, UDP
  port = 50004.

You don't need SDRconnect or any other SDRplay end-user app — the
SDRplay API package alone is enough.

### WSJT-X configuration

| Setting | Value |
|---|---|
| Radio | your real rig, via Hamlib (Settings → Radio → choose your rig and CAT method — *not* a `Hamlib NET rigctl` pointing at this bridge) |
| PTT method | CAT |
| Sound output (TX) | the real rig's USB audio interface |
| Sound input (RX) | `Line 1 (Virtual Audio Cable)` |
| Settings → Reporting → "Accept UDP requests" | **enabled**, port `2237` |

That last item is required — without it WSJT-X doesn't broadcast its
status messages and the bridge has no way to know the dial frequency.

Launch order: **real rig → WSJT-X → SDRplay RX Bridge → QMAP**.

### Features

- **WSJT-X UDP listener** (port 2237) for dial freq, mode, and TX
  state. Same protocol GridTracker / JTAlert / Log4OM use; works
  cross-machine.
- **SDRplay RX path** at 2 Msps with hardware DC + I/Q correction
  enabled. The 14-bit ADC feeds the existing 8-bit-IQ pipeline via a
  top-byte (`>> 8`) conversion.
- **SSB demod** via Hilbert phasing to VB-Cable Line 1 for WSJT-X.
- **96 kHz IQ wideband stream** to QMAP via UDP 50004 (Linrad
  protocol), with the same `(-1)^n·conj()` transform and adaptive
  I/Q balance correction as the HackRF / RTL-SDR bridges. SDRplay's
  hardware correction usually carries most of the rejection budget,
  so the post-stage IqBalancer settles near α≈1, φ≈0.
- **RX-blanking on WSJT-X TX**: SDRplay RX is uninitialised while
  WSJT-X is keying the real rig, so the local-TX bleed-through doesn't
  hammer the front end.
- **Bias-tee follows WSJT-X TX (RSPduo)**: if you have the bias tee
  enabled (manual toggle), the 4.7 V on the Tuner B SMA centre
  conductor follows WSJT-X's transmit state for transverter
  sequencer / external-LNA power.
- **Settings dialog**: IF gain reduction (gRdB 20-59 dB), LNA state
  (0-9), SDRplay AGC (50 Hz response, replaces manual gRdB), bias tee
  (RSPduo), broadcast-band notch (RSPduo), DAB notch (RSPduo), DC
  offset correction, I/Q imbalance correction, PPM correction, RX
  audio device, software RX audio gain, Linrad output gain, I/Q
  balance toggle with live α / φ / native-rejection diagnostics.
- **GUI status panel**: dial freq from WSJT-X, mode, WSJT-X UDP-link
  state, SDRplay model + serial, RX peak meter, waterfall.

### Command-line options

- `--gr-db <dB>` — IF gain reduction (20-59; lower = more gain)
- `--lna-state <n>` — LNA state (0-9; lower = more LNA gain)
- `--agc` — enable SDRplay AGC at 50 Hz (replaces manual gRdB)
- `--bias-tee` — enable RSPduo Tuner B bias-T
- `--rf-notch` — enable RSPduo broadcast-band notch
- `--dab-notch` — enable RSPduo DAB notch
- `--no-dc-correction` — disable hardware DC offset correction
- `--no-iq-correction` — disable hardware I/Q imbalance correction
- `--ppm <ppm>` — frequency correction (master clock)
- `--sample-rate <sps>` — SDRplay sample rate (default 2000000)
- `--rx-device <name>` — audio output device (default Line 1)
- `--linrad-gain <dB>` — Linrad/QMAP digital output gain (default 20)
- `--wsjtx-port <port>` — WSJT-X UDP listener port (default 2237)
- `--no-gui` — headless mode
- `--console` — open a debug console with full stderr log

`--help` shows the full set.

### Reporting

Send observations / decodes / bug reports to
**<n6nu@arrl.net>**. Useful info to include:

- SDRplay model (RSPduo / RSP1A / etc.) and serial number (shown in
  the bridge GUI status panel)
- SDRplay API version (logged at startup with `--console`)
- Windows version
- WSJT-X version + which real rig you're using
- The bridge log: relaunch with `sdrplay-rx-bridge.exe --console`,
  reproduce the issue, copy/paste the console output
- For QMAP issues, also `qmap.ini` and a wideband-waterfall
  screenshot

### License

Copyright (C) 2026 **Andreas Junge, N6NU** &lt;<n6nu@arrl.net>&gt;.
Licensed under the **GNU General Public License v3 or later** —
see [`LICENSE`](LICENSE). Bundled third-party components are
documented in [`THIRD_PARTY_LICENSES.md`](THIRD_PARTY_LICENSES.md).
The SDRplay API DLL is **not** bundled — install it separately from
sdrplay.com per their redistribution agreement.
**No warranty.** Install and run at your own risk.
