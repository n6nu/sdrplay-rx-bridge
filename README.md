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

## Latest release — v1.0.2

Download: **[sdrplay-rx-bridge-1.0.2-setup.exe](sdrplay-rx-bridge-1.0.2-setup.exe)**

**Spectrum waterfall is now optional.** The built-in waterfall display
can be turned off when you don't need the visual debug info — useful if
you're running on a smaller screen or want to free a couple percent of
CPU.

- **View → "Show spectrum waterfall"** (or **Ctrl+W**) toggles it.
- New CLI flag `--no-waterfall` launches with the display already off.
- Default ON — existing testers see no change unless they turn it off.
- When off, the bridge skips the per-IQ-buffer FFT compute work, the
  paint events, and the 20 Hz row-poll timer. Roughly 2–5 % of one
  CPU core saved at 2 Msps; more on slower boxes.
- Setting persists across launches (INI key `gui/waterfall_enabled`).

Drop-in upgrade from v1.0.x — no INI migration, no behaviour change for
operators who like the waterfall.

### v1.0.1 — 14-bit audio + waterfall

**Audio path now runs at full 14-bit precision** to match the QMAP path
that v1.0.0 already widened. v1.0.0 routed int16 only to `LinradServer`
(the QMAP wire); `SsbDemodulator` and `FftEngine` were still fed the
int8 view, capping the WSJT-X audio path and the waterfall display at
the same 8-bit dynamic range we shipped through 0.99.x.

- `SsbDemodulator` gains an `int16` overload — WSJT-X RX audio now
  inherits the SDRplay's full 14-bit ADC range.
- `FftEngine` gains an `int16` overload — main-window waterfall
  contrast on weak signals improves accordingly.
- HackRF and RTL-SDR sibling apps unchanged (their HW is int8).

### v1.0.0 — first stable

**Exiting beta.** Two cumulative changes since the last `0.99.x`:

- **14-bit IQ end-to-end through the QMAP path.** The bridge previously
  down-converted the SDRplay's 14-bit ADC to int8 in `SdrplayDevice`
  before any DSP touched it — losing 36 dB of dynamic range on the way
  to QMAP. v1.0.0 keeps the int16 stream intact through `LinradServer`:
  the wire format to QMAP was already int16, but until now we'd thrown
  away the bottom 8 bits of every sample on entry. (v1.0.1 extends
  this to the audio path as well.)
- **First stable release.** The Phase 1b dedupe is settled, the Low-IF
  DC-spike fix is verified, the per-radio gain panels are clean, three
  RSP* models tested. The `0.99.x` beta line ends here; future
  development opens a `1.x` series. v1.0.0 is a drop-in upgrade — no
  INI changes, no migration steps.

What landed in v0.99.13 — Low-IF 450 kHz promoted to default.

(Earlier versions: see `RELEASE_NOTES.md`.)
Tester verified v0.99.12 Low-IF on the bench (sig-gen at 144.400,
sweeping WSJT-X dial): tuning correct, spectrum orientation correct,
DC spike no longer on the wanted signal. Promoted to default for
fresh installs in v0.99.13.

- Fresh installs come up in Low-IF 450 kHz mode automatically.
  DC artifact lives at −450 kHz from the wanted signal — well
  outside the 96 kHz QMAP window.
- Existing INIs keep their stored value. Operators who landed in
  Zero-IF on v0.99.10/.11/.12 stay there until they switch via
  Settings → "IF mode".
- The Settings combo lists Low-IF 450 kHz as the default and Zero-IF
  as a diagnostic option.

What landed in v0.99.12 — flipped NCO sign for Low-IF (the actual
fix that brought the spectrum into correct orientation). v0.99.7
through v0.99.11 were a saga of fix-attempts; v0.99.12 was the one
that worked end-to-end. v0.99.11's Q-conjugation was a no-op — the
LinradServer's mandatory `(-1)^n·conj()` transform (required by
QMAP for its FFT-direction quirk) silently undid it. The actual
fix was the NCO direction. Empirically the SDRplay puts the wanted
signal at −450 kHz (not +450 kHz) in IF_0_450 output IQ; combined
with the LinradServer's conjugation the right NCO sign is +,
not −.

**Test:** with sig-gen at 144.400 fixed:
- Dial 144.380 → carrier should appear at QMAP +20 kHz from centre
  (sig-gen above dial → above centre).
- Dial 144.410 → carrier at −10 kHz from centre.
- Dial 144.400 → carrier at QMAP centre.
- DC artifact: at −450 kHz, off-screen.

What landed in v0.99.11 — Q conjugation attempt (turned out to be
a no-op due to LinradServer transform). v0.99.10's Low-IF received the
signal at the right amplitude but the spectrum came out *mirrored*
around the dial — sweeping the dial across a fixed sig-gen carrier
showed the apparent frequency moving in the *opposite* direction
from the real offset (apparent_offset = −real_offset). Root cause:
the SDRplay outputs IQ with Q inverted in IF_0_450 mode (chip-design
quirk known on the Mirics MSi001). Fix: conjugate Q before the NCO
in `SdrplayDevice::onStream` when low-IF is active. Standard "above
dial = above baseband" orientation restored.

**Test:** with sig-gen at 144.400 fixed and WSJT-X dial sweeping:
- Dial 144.400: carrier at QMAP centre.
- Dial 144.390: carrier 10 kHz **above** dial centre (was 10 kHz below in v0.99.10).
- Dial 144.380: carrier 20 kHz **above** dial centre.
- DC artifact: still off-screen at dial − 450 kHz.

What landed in v0.99.10 — fixed runtime IF-mode switching with
proper Uninit + re-init.

What landed in v0.99.9 — Low-IF 450 kHz with NCO downconverter
(opt-in via Settings → "IF mode"). When you flipped Low-IF on at runtime via
Settings, the chip's IF wasn't actually changing — only the bridge's
NCO came online while the SDRplay stayed in its old IF mode. Net
effect: NCO rotated already-baseband samples by −450 kHz, pushing
the wanted signal off-screen. v0.99.10 now does a proper
Uninit → re-apply params → Init cycle when `if_khz` (or sample rate)
changes — those are "structural" SDRplay settings that can't be
hot-swapped.

Please retry on your sig-gen at 144.400 with WSJT-X dial at 144.400:
- Open Settings → IF mode → "Low-IF 450 kHz" → Apply.
- The bridge's `--console` log should print
  `[SDRplay] IF mode / sample rate changed — uninit + re-init`.
- QMAP wideband: expected to show clean carrier at dial centre,
  DC spike off-screen at dial − 450 kHz.

Default still Zero-IF until verified.

What landed in v0.99.9 — Low-IF 450 kHz with NCO downconverter
(opt-in), but the runtime-switch path was broken.

When you select "Low-IF 450 kHz" in Settings:
- The SDRplay tunes its analog LO 450 kHz below the dial.
- Chip runs at 4 Msps internally; API decimation by 2 brings the
  output to 2 Msps (matching the rest of the bridge pipeline).
- A complex NCO downconverter inside the bridge shifts the wanted
  signal from +450 kHz IF down to baseband.
- Net effect: wanted signal at dial freq, DC artifact at −450 kHz
  from signal — outside the 96 kHz QMAP window.

**Verification request**: please test on your sig-gen at 144.400 with
WSJT-X dial at 144.400 and report what you see in QMAP wideband.
Expected: clean carrier at dial centre, DC spike off-screen at
dial − 450 kHz. If that's confirmed, v0.99.10 will flip the default
to Low-IF.

Why not Low-IF 1.62 MHz? At our 2 Msps complex output, +1.62 MHz is
beyond Nyquist (fs/2 = 1 MHz) and aliases. Using IF_1_620 properly
needs the chip running at 6+ Msps with our own decimating filter on
the way into the bridge — deferred until somebody asks. The 96 kHz
QMAP wideband path is fully served by IF_0_450's 600 kHz analog BW.

Earlier — v0.99.8 hot fix reverting v0.99.7's broken Low-IF default. v0.99.7
was based on the wrong assumption about how SDRplay's Low-IF mode
works: in `IF_1_620` the SDRplay outputs IQ centered at the IF
(1.62 MHz), not at the dial frequency, so the wanted signal sits at
+1.62 MHz in the output stream — outside the 96 kHz QMAP window
and outside the SSB demod's audio passband. The bridge needs an
NCO downconverter in its DSP pipeline to use Low-IF properly, and
that wasn't shipped in v0.99.7. Symptoms in v0.99.7: low audio
levels, no signal in QMAP, DC spike still visible.

v0.99.8 forces Zero-IF on launch. Tester INIs from v0.99.7 with
`sdrplay/if_khz = 1620` are silently corrected back to 0 on first
run; the IF mode dropdown is greyed out at "Zero-IF" until a future
version implements the NCO downconverter properly.

The DC-spike fix is back on the BACKLOG — it's still the right idea,
just needs more DSP work than I initially shipped.

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
