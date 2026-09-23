# Can an eSCL/AirScan bridge make the SCX-4623FW visible to Image Capture?

Research for [issue #3](https://github.com/filippolmt/samsung-scx-macos/issues/3), part of
the map in [issue #2](https://github.com/filippolmt/samsung-scx-macos/issues/2).
Researched 2026-09-23 from primary sources (project source code, READMEs, issue threads).
**Nothing here was tested on the real SCX-4623FW or on macOS 26** — this was researched
from a Linux container. Claims marked *(untested)* are inferences from source code.

## Verdict

**Feasible, with caveats.** [AirSane](https://github.com/SimulPiscator/AirSane) is an
existing, maintained SANE-to-eSCL bridge that runs on macOS, uses Homebrew's
`sane-backends`, and publishes scanners over Bonjour on the *same* Mac, where Image
Capture and Preview pick them up. Its source maps the SCX-4623FW's `xerox_mfp` options
(Flatbed/ADF, Gray/Color) onto eSCL cleanly. The caveats: it must be built from source (no
Homebrew formula, no binaries), it runs as a root LaunchDaemon, macOS clients are known to
fail with error `-21345` on networks without IPv6, and the one reported attempt with a
Samsung `xerox_mfp` network MFP on macOS hit exactly that error and was never confirmed
working. Nobody has reported it on macOS 26.

## Which projects exist

| Project | eSCL server? | Runs on macOS? | Notes |
|---|---|---|---|
| [AirSane](https://github.com/SimulPiscator/AirSane) | Yes | Yes, build from source | GPL-3.0, v0.4.12 released 2026-06-03, C++ |
| [scanservjs](https://github.com/sbs20/scanservjs) | No — web UI only | No | README requires a "Linux host (or VM ...)" |
| [sane-airscan](https://github.com/alexpevzner/sane-airscan) | No — eSCL *client* | — | The opposite direction: a SANE backend that talks to eSCL scanners |

AirSane describes itself as "a SANE frontend, and a scanner server that supports Apple's
AirScan protocol. Scanners are detected automatically, and published through mDNS", and
names Apple Image Capture as an intended client
([README](https://github.com/SimulPiscator/AirSane/blob/129cc3bf7258251a0a694dee7741285b59d88f9f/README.md)).
It implements the Mopria eSCL protocol; the
[Mopria eSCL specification](https://mopria.org/mopria-escl-specification) is publicly
downloadable after accepting a license. AirSane itself recommends scanservjs to anyone who
wants a richer web frontend rather than eSCL. No other general SANE-to-eSCL server turned
up.

## Does macOS accept an eSCL scanner advertised from the same machine?

Yes, according to the AirSane author. The macOS README
([README.macOS.md](https://github.com/SimulPiscator/AirSane/blob/129cc3bf7258251a0a694dee7741285b59d88f9f/README.macOS.md))
installs the daemon on the Mac itself and states that any scanner "recognized by
`scanimage -L`, it will also be available in Apple Image Capture, Preview, and other
applications that rely on Apple's ImageCapture framework". The main README repeats this:
"AirSane may be run on a macOS installation in order to serve locally attached scanners to
eSCL clients such as Apple Image Capture", and in System Settings the scanner shows up as
a "Bonjour Scanner". On macOS it publishes via Apple's DNS-SD API
(`zeroconf/mdnspublisher-dnssd.mm`), not Avahi, so no extra mDNS daemon is needed.

The docs speak of USB scanners, but the SCX-4623FW reaches SANE over TCP. That should not
matter *(untested)*:

- AirSane's `local-scanners-only` option ("ignore SANE network scanners") defaults to
  `false` ([server/server.cpp](https://github.com/SimulPiscator/AirSane/blob/129cc3bf7258251a0a694dee7741285b59d88f9f/server/server.cpp)),
  and the shipped launchd plist does not set it.
- Even when set, it is only passed as `local_only` to `sane_get_devices()`
  ([sanecpp/sanecpp.cpp](https://github.com/SimulPiscator/AirSane/blob/129cc3bf7258251a0a694dee7741285b59d88f9f/sanecpp/sanecpp.cpp)),
  and `xerox_mfp`'s `sane_get_devices()` ignores that flag and returns every `tcp` device
  in its config file ([backend/xerox_mfp.c](https://gitlab.com/sane-project/backends/-/blob/master/backend/xerox_mfp.c)).

## What happens to the SCX-4623FW's capabilities

AirSane builds the eSCL capabilities from the SANE options
([server/scanner.cpp](https://github.com/SimulPiscator/AirSane/blob/129cc3bf7258251a0a694dee7741285b59d88f9f/server/scanner.cpp)).
Matched against `xerox_mfp`, whose `source` list is `"Flatbed", "ADF", "Auto"` and whose
`mode` list is Lineart/Halftone/Gray/Color
([backend/xerox_mfp.c](https://gitlab.com/sane-project/backends/-/blob/master/backend/xerox_mfp.c)):

- **Flatbed → eSCL Platen.** `findFlatbedName()` matches `"Flatbed"` first.
- **ADF → eSCL Feeder (simplex).** `findAdfSimplexName()` matches `"ADF"`.
- **No duplex.** `findAdfDuplexName()` looks for a source containing `"Duplex"`; the
  backend has none, which matches the hardware.
- **Colour modes: Grayscale8 and RGB24 only** (plus 16/48-bit when the backend reports a
  bit-depth above 8). AirSane only offers SANE's `Gray` and `Color`; Lineart and Halftone
  are not exposed. `bin/scx-scan` uses only Gray and Color, so nothing it does today is
  lost.
- **Resolutions** come from the backend's list. `xerox_mfp` reads the device's supported
  DPI values.
- **Output formats:** JPEG, PNG and PDF/raster, encoded while the scan runs.
- **Multi-page ADF:** eSCL clients pull one page at a time until the feeder is empty.
  Whether that works with this backend is untested. There is an open AirSane bug about a
  multi-page ADF crash, but it is specific to the HP `hpaio` backend
  ([#142](https://github.com/SimulPiscator/AirSane/issues/142)), and another about empty
  PDFs with feeder plus duplex ([#138](https://github.com/SimulPiscator/AirSane/issues/138)).

## Known problems and risks

1. **IPv6 / error -21345.** The README's troubleshooting section says: "Apple Image
   Capture fails to connect to the scanner (shows an **'error 21345'**). Enable IPv6 in
   your local network, and on the machine running AirSane." Same advice in
   [#44](https://github.com/SimulPiscator/AirSane/issues/44). Whether this also hits a
   same-machine setup, where the scanner is announced on the Mac's own interfaces, is
   untested.
2. **Closest precedent never confirmed working.**
   [#106](https://github.com/SimulPiscator/AirSane/issues/106): a Samsung CLX-3305 network
   MFP using `xerox_mfp` on macOS Ventura. `scanimage` worked, AirSane's web UI on port
   8090 listed the scanner, but Image Capture gave "Failed to open a connection to the
   device (-21345)". The author suggested IPv6; the reporter got partial progress and the
   issue was closed for inactivity. This is the biggest open risk for this repo.
3. **Build from source only.** No Homebrew formula (formulae.brew.sh has no `airsane`)
   and no macOS binaries; CI builds only on `ubuntu-latest`. You need Xcode command-line
   tools, `cmake`, `jpeg`, `libpng`. Apple Silicon build problems were reported and fixed:
   `/opt/homebrew` paths ([#95](https://github.com/SimulPiscator/AirSane/issues/95),
   now in `CMakeLists.txt`) and a clang error on macOS 14
   ([#124](https://github.com/SimulPiscator/AirSane/issues/124)). macOS 26 builds are
   unverified.
4. **Root LaunchDaemon and firewall prompt.** `sudo make install` puts `airsaned` in
   `/usr/local/sbin` and a plist in `/Library/LaunchDaemons` that runs as root with
   `KeepAlive` ([launchd plist](https://github.com/SimulPiscator/AirSane/blob/129cc3bf7258251a0a694dee7741285b59d88f9f/launchd/org.simulpiscator.airsaned.plist)).
   macOS asks whether `airsaned` may accept incoming connections. It listens on all
   interfaces on port 8090. The default `access.conf` (`allow local on *`) limits clients
   to local subnets, but the scanner is still published to the whole LAN. For a
   same-machine-only setup, `--interface` could bind it to loopback, but whether Image
   Capture still finds it then is untested.
5. **Colour rendering.** macOS ignores the colour space in AirScan images and applies a
   gamma-1.8 profile. The README recommends `gray-gamma`/`color-gamma 0.555555` in
   `options.conf` if scans look too dark.
6. **Two SANE configs to keep in sync.** A Homebrew-linked `airsaned` reads
   `/opt/homebrew/etc/sane.d/xerox_mfp.conf`, the same file this repo already warns about
   (see `CLAUDE.md`). Wiping it hides the scanner from Image Capture too *(inference)*.

## What it means for the app (map #2)

An eSCL bridge is a genuine alternative to a custom UI: install AirSane plus Homebrew
`sane-backends`, and every ImageKit app (Image Capture, Preview, the scan sheet in other
apps) becomes the UI. It trades app code for a compiled root daemon, an IPv6 dependency,
and an unconfirmed track record with `xerox_mfp` network devices. Before anyone relies on
it, one hands-on check on macOS 26 with the real printer is needed:

1. Build and install AirSane with Homebrew as in `README.macOS.md`.
2. Open `http://localhost:8090/` and do a preview scan (proves AirSane-to-SANE works).
3. Open Image Capture and scan from the glass, then from the ADF with several pages
   (proves macOS-to-eSCL works, including the -21345 risk and multi-page).
4. Repeat with IPv6 disabled on the Mac, to see whether the error hits a same-machine
   setup.
