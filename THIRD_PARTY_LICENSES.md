# Third-Party Components

The HackRF WSJT-X Bridge installer bundles binaries of the following
third-party libraries and tools. Each is governed by its own license.
This document is included with the installer to satisfy attribution
and source-availability requirements.

The bridge itself is GPLv3; see `LICENSE`.

---

## libhackrf

- **Use:** `hackrf.dll` — USB control and streaming for the HackRF One
  device.
- **License:** GNU General Public License, version 2 (GPLv2).
- **Upstream:** <https://github.com/greatscottgadgets/hackrf>
- **Source:** the upstream repository at the same URL. The bridge
  links unmodified upstream sources.

## FFTW (fftw3 / fftw3f)

- **Use:** `fftw3.dll`, `fftw3f.dll` — Fast Fourier transform routines
  for the bridge's spectrum/waterfall display.
- **License:** GNU General Public License, version 2 or later
  (GPLv2+). FFTW is also available under a commercial license; this
  build uses the GPL version.
- **Upstream:** <http://www.fftw.org/> — and
  <https://github.com/FFTW/fftw3>.
- **Source:** the upstream releases at the URL above.

## Qt 6

- **Use:** `Qt6Core.dll`, `Qt6Gui.dll`, `Qt6Widgets.dll`,
  `Qt6Network.dll`, `Qt6Multimedia.dll`, `Qt6Svg.dll`, plus the
  bundled Qt platform plugins under the installation directory's
  `platforms/`, `imageformats/`, `multimedia/`, `tls/`, etc.
- **License:** GNU Lesser General Public License version 3 (LGPLv3).
  Dual-licensed commercially by The Qt Company; this build uses
  LGPLv3.
- **Upstream:** <https://www.qt.io/> — source at
  <https://download.qt.io/official_releases/qt/>.
- **Source:** Qt 6.8.3 (the version of Qt this build links against).

## SoXR (libsoxr)

- **Use:** `soxr.dll` — high-quality rational-rate sample-rate
  conversion (used to decimate 2 Msps from the HackRF down to 96 kHz
  for QMAP and 48 kHz for the SSB demodulator).
- **License:** GNU Lesser General Public License version 2.1
  (LGPLv2.1).
- **Upstream:** <https://sourceforge.net/projects/soxr/>.

## libusb-1.0

- **Use:** `libusb-1.0.dll` — cross-platform USB I/O used by libhackrf.
- **License:** GNU Lesser General Public License version 2.1 or later
  (LGPLv2.1+).
- **Upstream:** <https://libusb.info/> — source at
  <https://github.com/libusb/libusb>.

## pthreads4w (pthreadVC3.dll)

- **Use:** `pthreadVC3.dll` — POSIX-threads emulation for Windows,
  pulled in as a dependency of soxr / hackrf builds on MSVC.
- **License:** Apache License 2.0 (the project is dual-licensed with
  the Apache 2.0 option active for binary distributions).
- **Upstream:** <https://sourceforge.net/projects/pthreads4w/>.

## FFmpeg shared libraries

- **Use:** `avcodec-61.dll`, `avformat-61.dll`, `avutil-59.dll`,
  `swresample-5.dll`, `swscale-8.dll` — pulled in transitively by Qt
  Multimedia's FFmpeg backend on Windows.
- **License:** GNU Lesser General Public License version 2.1 or later
  (LGPLv2.1+); some optional components are GPL but the LGPL build of
  FFmpeg is what Qt Multimedia ships.
- **Upstream:** <https://ffmpeg.org/>.

## Zadig

- **Use:** Bundled `zadig.exe` (Pete Batard's WinUSB driver-installer
  GUI from the libwdi project). Launched as an *optional* installer
  task to set up the WinUSB driver for the HackRF One. The user may
  decline this task at install time.
- **License:** GNU General Public License version 3 (GPLv3).
- **Upstream:** <https://github.com/pbatard/libwdi> and
  <https://zadig.akeo.ie/>.
- **Source:** the upstream repository at the URLs above. The
  bridge installer ships the unmodified upstream binary.

## VB-Audio Virtual Cable (NOT bundled, but required at runtime)

For completeness: the bridge expects the user to install VB-Audio
Virtual Cable separately. It is **not** redistributed by this
installer. It is donationware under VB-Audio's own license — see
<https://vb-audio.com/Cable/>.

---

## Source availability

All bundled GPL- and LGPL-licensed components are freely available
from their upstream maintainers at the URLs above; the bridge links
unmodified upstream releases. If you need a copy of the source for
any specific bundled binary version and cannot retrieve it from
upstream, contact Andreas Junge (N6NU) at **<n6nu@arrl.net>** and a
copy will be provided in accordance with GPL section 6 / LGPL
section 4.
