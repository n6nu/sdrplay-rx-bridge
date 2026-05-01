# SDRplay RX Bridge — Release Notes

## v0.99.11 — beta (2026-05-01)

**Spectrum mirroring fix in Low-IF mode.** v0.99.10's Low-IF received
the signal but the spectrum came out mirrored around the dial — a
sig-gen 10 kHz above dial appeared 10 kHz *below* dial in QMAP. Root
cause: the SDRplay outputs IQ with Q inverted in IF_0_450 mode (a
known chip-design quirk). Fix: conjugate Q (negate it) before the NCO
in `SdrplayDevice::onStream` when low-IF is active. Standard
"above dial = above baseband" orientation restored.

Verification: with sig-gen at 144.400 and dial sweeping
144.380 / 144.390 / 144.400, the carrier should now appear at the
expected position (above dial when dial < 144.400; right at dial
when they match). DC artifact stays at −450 kHz from signal.

## v0.99.10 — beta (2026-05-01)

**Bug fix against v0.99.9.** Switching to Low-IF mode at runtime via
Settings didn't actually change the SDRplay's IF. The new `if_khz`
value was stored in the Config and the bridge's NCO came online —
but the chip stayed in its previous IF mode. Net effect: NCO rotated
already-baseband samples by −450 kHz, pushing the wanted signal off
to −450 kHz (out of band). Symptom: signal disappears in Low-IF.

Fix: when `if_khz` (or `sample_rate`) changes via `configure()`, the
device class now does a full Uninit → re-apply params → Init cycle.
The chip's analog filter / LO offset / decimation factor / bandwidth
are all "structural" — they can't be hot-swapped via the API's
update reasons; the streaming pipeline has to be torn down and
rebuilt. Cold start (bridge launches with `if_khz=450` already in
INI) was already working — only the live Settings change was broken.

Default still Zero-IF until verified on a real signal.

## v0.99.9 — beta (2026-05-01)

**Low-IF mode is back, this time with the matching DSP.** v0.99.7
shipped half a fix; v0.99.9 ships the rest. Default is still Zero-IF
(no behavioural change for existing testers); Low-IF is opt-in via
Settings → "IF mode" while it's verified on real signals.

### What's actually in the box

- `SdrplayDevice` now supports `IF_0_450` (the only IF mode that fits
  within Nyquist at our 2 Msps complex output). When enabled:
  - `fsHz` set to **4 MHz** at the chip; SDRplay's API decimation
    (factor 2) brings the output stream to 2 Msps — matching the rest
    of the bridge pipeline.
  - Analog bandwidth filter narrows to 600 kHz (centred on the IF).
  - **Complex NCO downconverter** in `SdrplayDevice::onStream` rotates
    each int8 IQ pair by `exp(-j·2π·450 kHz·n / 2 Msps)` to shift the
    wanted signal from +450 kHz IF down to baseband. Implemented as a
    recursive complex rotor (cheap — ~10 multiplies per sample, with
    periodic renormalisation against FP drift).
  - **Net effect**: wanted signal at baseband, DC artifact (LO leakage,
    ADC offset, 1/f noise) at −450 kHz from signal — outside the 96 kHz
    QMAP window.

### Tester verification expected before defaulting

On the bench RSPduo this build streams at 1.99 Msps with peak |IQ|=19
in IF_0_450 (matching the 1.97 Msps / peak 20 we get in Zero-IF). The
math says the wanted signal should land back at baseband; that needs a
real-signal check before we make Low-IF the default. Suggested test:
sig-gen at 144.400 MHz, WSJT-X dial 144.400 MHz, look at QMAP wideband.
- Zero-IF: DC spike at dial, sig-gen carrier on top of it.
- IF_0_450: sig-gen carrier at dial (clean), DC spike off-screen at
  dial − 450 kHz.

If verification passes, v0.99.10 will flip the default to IF_0_450 and
mark Zero-IF as the diagnostic option.

### Why not Low-IF 1.62 MHz?

`IF_1_620` and `IF_2_048` would put the wanted signal beyond Nyquist
at our 2 Msps output (fs/2 = 1 MHz). Using them properly needs the
chip running at 6+ Msps with our own decimating filter on the way
into the bridge pipeline — deferred until somebody asks. The 96 kHz
QMAP wideband path is fully served by IF_0_450's 600 kHz analog BW.

## v0.99.8 — beta (2026-04-30)

**Hot fix**: revert v0.99.7's Low-IF-by-default change. Tester report
showed RX broken in Low-IF mode (low audio levels, no signal in QMAP
window). Root cause: in `IF_1_620` mode the SDRplay outputs IQ
*centered at the 1.62 MHz IF* (not at the dial frequency) — the
wanted signal is at +1.62 MHz from the output IQ center, outside our
96 kHz QMAP window and outside the SSB demod's audio passband. v0.99.7
shipped without the matching DSP change (a complex NCO mixer to shift
the IF signal back to baseband), so the wanted signal effectively
disappeared.

v0.99.8 forces Zero-IF on every launch. The "IF mode" Settings combo
is greyed out — only Zero-IF is selectable until the NCO downconverter
is wired into the bridge DSP pipeline. Tester INIs from v0.99.7 with
`sdrplay/if_khz = 1620` will be silently corrected to 0 on first run.

The DC-spike fix is back on the BACKLOG — pending a proper Low-IF
implementation with the NCO mixer (estimated ~4-6 hours of DSP work).

## v0.99.7 — beta (2026-04-30)

- **Low-IF mode is now the default** — fixes the visible DC spike at
  the dial frequency that testers reported (tester request).
  Previously the bridge ran in zero-IF (`sdrplay_api_IF_Zero`) which
  puts LO leakage, ADC offset, and 1/f noise right at the wanted
  signal. v0.99.7 defaults to **`sdrplay_api_IF_1_620`** which moves
  those artifacts to −1.62 MHz from the signal — well outside the
  1.536 MHz analog bandwidth and the 96 kHz QMAP window.
- Settings → new **"IF mode"** combo with two options:
  - **Low-IF 1620 kHz (recommended — DC out of band)** ← new default
  - **Zero-IF (DC at signal — has spike, diagnostic only)** ← previous
    behavior, still selectable for diagnostic comparison
- The SDRplay handles digital downconversion internally — output
  samples still arrive at the dial frequency at baseband regardless
  of IF mode, so the bridge's DSP pipeline (`SsbDemodulator`,
  `LinradServer`, `(-1)^n·conj()` transform) is unaffected.
  Bandwidth stays at 1.536 MHz.
- INI key: `sdrplay/if_khz` (0 = Zero-IF, 1620 = Low-IF). Existing
  installs upgrade to Low-IF on first launch — the DC spike just
  disappears.

Note: the `IqBalancer`'s blind α/sin(φ) estimator was tuned for
zero-IF signal stats; if you see I/Q balance correction behave
differently in low-IF mode, toggle it off in Settings and report
the comparison.

## v0.99.6 — beta (2026-04-30)

Architectural refactor — Phase 1b GUI consolidation. No functional
change for the user; the SDRplay-side feature set (overload indicator,
manual SDR freq override, transverter offset, streaming-stats log,
bold red IF readout, freq display sourced from device cfg) is identical
to v0.99.5. Behind the scenes:

- `RxMainWindow` and `RxSettingsDialog` now live in `bridge-core/` and
  are shared with the HackRF-RX-Bridge and RTL-SDR-RX-Bridge sibling
  apps. Future GUI features land once and propagate to all three.
- All radios derive from a new `IqDevice` abstract interface in
  `bridge-core/IqDevice.h`. Capability differences (overload events,
  multi-antenna selection) are surfaced through the interface.
- SDRplay-specific Settings rows (gRdB, LNA state, AGC, bias-T,
  notches, antenna picker, DC/IQ correction, PPM) now live in a
  self-contained `SdrplayGainPanel` widget that the shared dialog
  embeds.

The HackRF-RX-Bridge and RTL-SDR-RX-Bridge sibling apps gain feature
parity in their respective v0.99.2 releases (transverter offset,
manual SDR freq override, streaming-stats log).

## v0.99.5 — beta (2026-04-30)

Bug fixes against v0.99.4 reported by beta tester.

- **Frequency display blank at startup with manual override on.** The
  GUI freq readout sourced its value from `WsjtxFreqSource::frequencyHz()`,
  which is empty until WSJT-X has broadcast at least one status message.
  In manual override mode (where the bridge ignores WSJT-X anyway), the
  display would never populate. Fixed: the display now sources from
  `sdrplay->config().freq_hz` — the bridge's actual operating freq —
  which is set at startup from either the manual-override INI keys or
  the last persisted WSJT-X dial. Tracking mode and override mode both
  show a sensible value from the moment the bridge launches.
- **Manual override could be silently undone by a WSJT-X dial change.**
  `RxMainWindow::onWsjtxFreq` was calling `sdrplay->setFrequency()`
  directly in parallel to the main.cpp WSJT-X freq lambda. The lambda
  honours the override flag; the GUI handler did not. With override on,
  any WSJT-X dial broadcast would retune the SDR off the manual freq.
  Fixed: removed the duplicate retune from the GUI handler — main.cpp
  now owns SDR retuning end-to-end.

## v0.99.4 — beta (2026-04-30)

Two tester requests landed in this build.

- **Front-end overload indicator.** When the SDRplay API fires a
  `PowerOverloadChange / Detected` event, a red `⚠ FRONT-END OVERLOAD`
  banner appears under the dial display in the GUI for ~3 s after each
  detection. The bridge has always acknowledged these events (so the API
  resumes operation), but until now the user had no visual feedback.
  Mirrors the overload indicator in HDSDR.
- **Manual SDR frequency override.** New Settings checkbox **"Manual SDR
  frequency (decouple from WSJT-X dial)"** plus an operating-MHz field.
  When checked, WSJT-X UDP freq updates are dropped on the floor and the
  SDR stays tuned to the manual freq. The transverter offset is still
  applied. LinradServer reports the manual freq to QMAP so the wideband
  waterfall labels reflect the actual SDR center. Useful for QMAP-only
  observation when activity spans more than 90 kHz around a dial — the
  operator can shift the QMAP window off the WSJT-X dial. **Note:**
  WSJT-X narrowband audio decode only works when the WSJT-X dial
  matches the manual freq; that's the trade-off the operator accepts
  in this mode. A bold orange banner under the dial display makes the
  override state unmistakable. CLI: `--manual-freq <MHz>`. INI keys:
  `radio/manual_freq_override`, `radio/manual_freq_hz`.

## v0.99.3 — beta (2026-04-29)

Diagnostic + RSPdx fixes prompted by tester report ("changing RF/IF
gain doesn't affect audio reaching WSJT-X; spectrum doesn't change
when WSJT-X retunes").

- **RSPdx antenna picker.** New Settings → "Antenna (RSPdx)" combo
  with Antenna A / B / C. The RSPdx has three RF inputs and v0.99.x
  was implicitly defaulting to whatever the SDRplay API returned
  (typically Antenna A). If the tester's signal was on Antenna B or
  C, the ADC was seeing nothing — gain changes did nothing because
  there was no signal to amplify; freq changes didn't move a flat
  noise floor. The picker is greyed-out on non-RSPdx models. CLI:
  `--antenna A|B|C`. INI key `sdrplay/antenna_sel`.
- **RSPdx bias-T / RF notch / DAB notch now actually work.** v0.99.x
  wrote those fields to RSPduo's params struct on every model and
  fired RSPduo Update reasons regardless of hardware, which were
  no-ops on RSPdx. v0.99.3 routes each model to its own params and
  Update reason: RSPduo via `rspDuoTunerParams` + `Update_RspDuo_*`,
  RSPdx via `devParams->rspDxParams` + `Update_RspDx_*` (Ext1 axis),
  RSP1A/1B via `rsp1aTunerParams` + `rsp1aParams` + `Update_Rsp1a_*`,
  RSP2 via `rsp2TunerParams` + `Update_Rsp2_*`. RSP1A / RSP1B / RSP2
  bias-T and notch toggles will now work in addition to RSPduo /
  RSPdx, but those models are still untested in actual operation.
- **Periodic streaming-stats log line.** Every 5 seconds the bridge
  now writes one diagnostic line:
  `[Stats] RX <N> samples in 5s (≈ <sps> sps), peak |IQ|=<X> (<Y> dBFS), last freq update <ms> ms ago`
  Lets a tester (or N6NU reading their `--console` log) answer "is
  the SDR actually streaming?" and "is WSJT-X actually feeding freq
  updates?" from one screenshot.

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

| RSP model | Status |
|---|---|
| **RSPduo** (single-tuner, Tuner A) | **Working** — verified on N6NU's bench. Full feature parity (RF / DAB notches, Tuner B bias-T toggles wired). |
| **RSPdx** | **Working** — beta-tester verified at 10 GHz EME via 144 MHz IF transverter (v0.99.1+) and antenna A/B/C selector verified (v0.99.3, 2026-04-30). Bias-T, RF notch, DAB notch toggles all wired in v0.99.3 but not yet exercised by a tester. HDR mode is not yet exposed (deferred follow-up). |
| **RSPdx-R2** | Untested. Same code path as RSPdx; antenna picker, bias-T, and notches go through the same Update reasons. |
| RSP1A / RSP1B / RSP2 / RSP1 | Should boot and stream. v0.99.3 wires per-model bias-T and notches (RSP1 has none). Untested in actual operation; reports welcome. |

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
