# SDRplay RX Bridge — Beta Releases

Windows installer downloads for the **SDRplay RX Bridge** — a Qt6 C++
application that adds wideband Q65 (QMAP) reception via an SDRplay
receiver (RSPduo, RSP1A, RSPdx, …) to a station already running a real
radio (IC-905, IC-705, FT-991A, etc.) for TX. Sibling to the HackRF and
RTL-SDR RX Bridges, with a 14-bit-ADC SDRplay front end for
substantially better dynamic range than the 8-bit siblings.

The bridge listens to **WSJT-X UDP** for the dial frequency, tunes the
SDRplay to match, demodulates SSB to **VB-Cable Line 1** for WSJT-X RX
audio, and streams **96 kHz IQ to QMAP** for wideband Q65 decode. Your
real rig keeps doing TX (and narrowband RX, if you prefer). No
interference with your existing CAT / audio setup.

Author: **Andreas Junge, N6NU** &lt;<n6nu@arrl.net>&gt;.

---

## Latest beta — v0.99.7

Download: **[sdrplay-rx-bridge-0.99.7-setup.exe](sdrplay-rx-bridge-0.99.7-setup.exe)**

What's new in v0.99.7 — **Low-IF mode is now the default**, fixing the
visible DC spike at the dial frequency. Previously the bridge ran in
zero-IF mode which puts LO leakage and ADC offset right at the wanted
signal. v0.99.7 defaults to **`sdrplay_api_IF_1_620`** which moves
those artifacts to −1.62 MHz from the signal — well outside the
1.536 MHz analog bandwidth and the 96 kHz QMAP window.

Settings → new **"IF mode"** combo lets you flip between Low-IF
(recommended) and Zero-IF (diagnostic). Existing installs upgrade
to Low-IF automatically — DC spike just disappears. The bridge's
DSP pipeline is unaffected (SDRplay handles the digital
downconversion internally).

What landed in v0.99.6 — architectural refactor, no functional change.
GUI classes (`RxMainWindow` + `RxSettingsDialog`) now live in
`bridge-core/` and are shared with the HackRF and RTL-SDR sibling
apps. Future GUI features land once and propagate to all three.
SDRplay-specific Settings rows live in a new `SdrplayGainPanel`
widget the shared dialog embeds. Same SDRplay feature set as v0.99.5.

What was fixed in v0.99.5 — two bugs reported against v0.99.4:

- **Frequency display blank at startup with manual override on.** The
  GUI now reads the bridge's actual operating freq (set at startup
  from either the manual-override INI keys or the last persisted
  WSJT-X dial), so the readout is populated from the moment the
  bridge launches in either tracking or override mode.
- **Manual override could be silently undone by a WSJT-X dial change.**
  The GUI was retuning the SDR in parallel to the main event loop and
  not honouring the override flag. Removed — main event loop now owns
  SDR retuning end-to-end. Override mode now actually stays put.

What's still in from v0.99.4 — two tester requests:

- **Front-end overload indicator.** A red `⚠ FRONT-END OVERLOAD` banner
  appears under the dial display whenever the SDRplay API reports a
  power-overload event, latched for ~3 s after each detection. Mirrors
  the overload indicator in HDSDR. Lets you spot a too-hot RF input
  without watching the log.
- **Manual SDR frequency override.** Settings → "Manual SDR frequency
  (decouple from WSJT-X dial)" + an operating-MHz field. When checked,
  the bridge ignores WSJT-X UDP freq updates and stays parked on the
  manual freq. Transverter offset still applied. LinradServer reports
  the manual freq to QMAP so the wideband waterfall labels match the
  actual SDR center. Useful when activity spans more than 90 kHz around
  a dial — you can shift the QMAP wideband window off the WSJT-X dial
  to capture more of the band. **WSJT-X narrowband audio decode only
  works when the WSJT-X dial matches the manual freq** (deliberate
  trade-off for QMAP-priority observation). A bold orange banner under
  the dial display makes the override state unmistakable. CLI:
  `--manual-freq <MHz>`.

Earlier in v0.99.3 — diagnostic + RSPdx fixes prompted by tester report
("changing RF/IF gain doesn't affect audio reaching WSJT-X; spectrum
doesn't change when WSJT-X retunes"):

- **RSPdx antenna picker** in Settings (Antenna A / B / C). The RSPdx
  has three RF inputs and v0.99.x defaulted to Antenna A. If your
  signal is on Antenna B or C, the ADC saw nothing — gain changes
  did nothing because there was no signal to amplify. CLI:
  `--antenna A|B|C`. INI key `sdrplay/antenna_sel`.
- **RSPdx bias-T / RF notch / DAB notch now actually work.** v0.99.x
  wrote those fields to the RSPduo's params struct on every model
  and fired RSPduo Update reasons regardless of hardware — no-op on
  RSPdx. v0.99.3 routes each model to its own params and Update
  reason. RSP1A / RSP1B / RSP2 bias-T and notch toggles will also
  work now (still untested).
- **Periodic streaming-stats log line.** Every 5 seconds the bridge
  writes one diagnostic line:
  `[Stats] RX <N> samples in 5s (≈ <sps> sps), peak |IQ|=<X> (<Y> dBFS), last freq update <ms> ms ago`
  Lets you answer "is the SDR actually streaming?" and "is WSJT-X
  feeding freq updates?" from one screenshot.

What landed in v0.99.2: higher-contrast IF / transverter readout —
the `IF tune: …  (offset …)` line under the main dial renders bold
red so the SDR-tune-vs-dial relationship is unmissable.

What landed in v0.99.1: **transverter offset** (tester request — 10368 MHz
operation through a 144 MHz IF transverter). Settings → "Transverter
offset" field (signed MHz). The SDR is tuned to *(WSJT-X dial + offset)*
while the GUI, WSJT-X, QMAP, and the LinradServer header all keep
showing the operating dial. Example: dial 10368 MHz, offset −10224 MHz,
SDR actually tunes to 144 MHz. CLI flag `--transverter-offset <MHz>`.

Per-model status:

| RSP model | Status |
|---|---|
| **RSPduo** (single-tuner, Tuner A) | **Working** — verified on N6NU's bench. Full feature parity. |
| **RSPdx** | **Working** — beta-tester verified at 10 GHz EME via 144 MHz IF transverter (v0.99.1+) and Antenna A/B/C selector verified in v0.99.3. Bias-T / RF notch / DAB notch toggles are wired but not yet exercised. HDR mode is a deferred follow-up. |
| **RSPdx-R2** | Untested — shares the RSPdx code path. |
| RSP1A / RSP1B / RSP2 / RSP1 | Should boot and stream. v0.99.3 wires per-model bias-T and notches (RSP1 has none). Untested in actual operation; reports welcome. |

Full per-version notes, system requirements and known limitations are
in [`RELEASE_NOTES.md`](RELEASE_NOTES.md).

### Known issues in this build

This is a first-cut beta. Two things to flag before you start:

- **Q65 wideband decoding may show frequency-smeared tones in QMAP's
  zoomed Q65 view** even when the same signal decodes cleanly on the
  sound-card / FT8 path. Strongest current suspect: SDRplay master-clock
  vs nominal `fsHz` drift; PPM correction not yet calibrated. The
  bridge produces UDP packets QMAP accepts (and sometimes decodes), but
  the tone smearing reduces decode rate vs the sound-card path. Tester
  data points especially welcome here — `--ppm` values that helped,
  beacon offsets, screenshots of QMAP's bottom zoomed panel.
- **Quiet-baseline comb pattern** with antenna disconnected and minimum
  gain — small low-level spurs visible in QMAP's wideband waterfall.
  Doesn't affect decoding when a real signal is present (well above the
  comb), but it's there. Top suspects: int16 → int8 truncation rounding
  bias, internal SDR spurs.

Neither blocks operation; both are open follow-up work.

### First-launch SmartScreen warning

The installer is **not code-signed** and is **64-bit only**
(Windows 10 / 11 x64). On first launch you will see:

> Windows protected your PC.
> Microsoft Defender SmartScreen prevented an unrecognized app from
> starting.

Click **More info → Run anyway**. You should only see this once per
binary. The same warning may appear once on the installed
`sdrplay-rx-bridge.exe`; handle it the same way.

### What you'll need

- **Real radio + WSJT-X** — the rig is whatever you already have
  (IC-905, IC-705, FT-991A, etc.). WSJT-X drives it via Hamlib as
  always; this bridge does NOT replace that.
- **SDRplay API package** — free, separate download from
  <https://www.sdrplay.com/api/>. Installs `sdrplay_api.dll` and
  registers the `SDRplayAPIService` Windows service that mediates
  hardware access. **Required**: the bridge will refuse to open the
  SDRplay without it. Per the SDRplay redistribution agreement we do
  **not** bundle the DLL — please install the API package separately.
  You don't need SDRconnect or any other end-user SDRplay app, just
  the API.
- **VB-Audio Virtual Cable** — <https://vb-audio.com/Cable/>. Provides
  the `Line 1` virtual sound device the bridge feeds.
- **WSJT-X 2.7+** with the UDP server enabled —
  Settings → Reporting → "Accept UDP requests" → port `2237`.
  Without this WSJT-X doesn't broadcast its dial freq and the bridge
  has nothing to track.
- **QMAP 0.6+** — Network input enabled, UDP port `50004`.

### WSJT-X audio routing for this bridge

| Setting | Value |
|---|---|
| Radio | your real rig, via Hamlib |
| PTT method | CAT |
| Sound output (TX) | the real rig's USB audio interface |
| **Sound input (RX)** | **`Line 1 (Virtual Audio Cable)`** ← fed by this bridge |
| Settings → Reporting → Accept UDP requests | **enabled**, port 2237 |

Launch order: **real rig → WSJT-X → SDRplay RX Bridge → QMAP**.

### First-time configuration in the bridge

After install, launch the bridge, click **Settings…**, and:

1. Set **RX audio output** to `Line 1 (Virtual Audio Cable)` — the
   default is the system audio device, not Line 1, so you need to pick
   it explicitly the first time.
2. Set **IF gain reduction (gRdB)** and **LNA state** for your
   operating conditions. The default 40 dB / state 4 is a reasonable
   starting point on 2 m. Lower gRdB (towards 20) gives more IF gain;
   lower LNA state (towards 0) gives more LNA gain. Raise either if
   you're near broadcast FM or strong pagers and the meter is hot.
   Or check **SDRplay AGC (50 Hz)** to let the receiver manage gain
   automatically.
3. Leave **DC offset correction** and **I/Q imbalance correction**
   enabled — both are hardware corrections in the SDRplay front end.
4. Click **Apply**. Settings persist to
   `%APPDATA%\Roaming\n6nu\SDRplay RX Bridge.ini`.

---

## What an SDRplay brings to weak-signal work

SDRplay receivers (RSPduo, RSP1A, RSPdx, …) use a 14-bit ADC after a
Mirics MSi001 / 2500 front end. That's substantially more dynamic range
than the 8-bit RTL-SDR (~120 dB native vs ~50 dB) and meaningfully more
than the HackRF too. The headroom is most useful in strong-RF
environments, near broadcast sites, or with high-gain antennas where
8-bit SDRs run out of room under modest gain.

- **Frequency range**: 1 kHz – 2 GHz native on the SMA path; full
  performance from 10 MHz up. Covers HF, 6 m, 2 m, 70 cm, 23 cm.
- **2 Msps** matches the bridge-core pipeline; **1.536 MHz analog
  bandwidth** — wide enough for the 96 kHz QMAP wideband stream.
- **Hardware DC offset and I/Q imbalance correction** — enabled by
  default. The bridge-core IqBalancer is downstream of these and is
  usually a no-op on RSPduo.
- **Bias tee (RSPduo Tuner B SMA)** — drives an external LNA or
  transverter sequencer at 4.7 V. Manual toggle plus optional
  "follow WSJT-X TX state" behavior. Note: v0.99.0 ships single-tuner
  on Tuner A only, so the bias-T toggle does nothing in hardware on
  Tuner A. Tuner B selection is on the v0.99.x roadmap.

For the curious: the bridge's downstream DSP is currently 8-bit-IQ
internally — the top 8 bits of each 14-bit ADC sample feed the pipeline.
A 16-bit-wide internal path is on the roadmap.

---

## Command-line options

Double-clicking the installed shortcut launches the GUI. From a terminal
you can also pass flags. The most useful ones for testers:

| Flag | What it does |
|---|---|
| `--help` | Print the full list of options and exit. |
| `--console` | Open a separate debug-console window with the bridge's full `stderr` log (banner, audio I/O, Linrad packet stats, I/Q balance estimator state, SDRplay API version + serial). Closing the console quits the bridge. |
| `--no-gui` | Headless mode — Linrad UDP + audio I/O run, no main window. |
| `--gr-db <dB>` | IF gain reduction in dB (20-59; lower = more gain). |
| `--lna-state <n>` | LNA state (0-9 for RSPduo SMA; lower = more LNA gain). |
| `--agc` | Enable SDRplay AGC at 50 Hz (replaces manual gRdB). |
| `--ppm <ppm>` | Master-clock correction. Useful if your beacon offsets show consistent drift. |
| `--linrad-gain <dB>` | Linrad/QMAP digital output gain (default +20). |

Example — relaunch with full logging and a 5 ppm correction:

```
"C:\Program Files\SDRplay RX Bridge\sdrplay-rx-bridge.exe" --console --ppm 5
```

The full set of flags is in `--help`.

---

## Reporting

Send observations / decodes / bug reports directly to
**<n6nu@arrl.net>**. Useful information to include:

- SDRplay model (RSPduo / RSP1A / etc.) and serial number — both shown
  in the bridge GUI status panel
- SDRplay API version (banner line at the top of `--console` output)
- Windows version
- WSJT-X version + which real rig you're using
- The bridge log: relaunch with `sdrplay-rx-bridge.exe --console`,
  reproduce, copy/paste the console output
- For QMAP issues, also `qmap.ini` and a wideband-waterfall screenshot,
  especially of the bottom zoomed Q65 panel — that's where the current
  tone-smearing issue is most visible.

---

## License

Copyright (C) 2026 Andreas Junge, N6NU &lt;<n6nu@arrl.net>&gt;.
Licensed under the **GNU General Public License version 3 or later**;
see [`LICENSE`](LICENSE).

This program is distributed in the hope that it will be useful, but
**WITHOUT ANY WARRANTY**; without even the implied warranty of
**MERCHANTABILITY** or **FITNESS FOR A PARTICULAR PURPOSE**. Use it at
your own risk.

The **SDRplay API** is **not bundled** with this installer — please
install it from sdrplay.com per their redistribution agreement.

Other bundled third-party components — including FFTW3 (GPLv2+),
Qt 6 (LGPLv3), SoXR (LGPLv2.1), and FFmpeg shared libraries
(LGPLv2.1+) — are documented in
[`THIRD_PARTY_LICENSES.md`](THIRD_PARTY_LICENSES.md). Source code for
the bridge itself is available on request from N6NU under the GPLv3
"written offer" provision (§6) at the email address above; a public
source-code repository will be linked here once the project leaves
beta.
