# Samsung SCX-4623FW — printing and scanning on macOS 26

Set up on 19 September 2026 on macOS 26.7 (Apple Silicon). This document explains
what was done, why, and how to put it back if it ever stops working.

---

## The device

| | |
|---|---|
| Model | Samsung SCX-4623FW Series (mono laser multifunction, 2012) |
| Address | whatever your router hands it — give it a static lease |
| Web interface | `http://<printer-ip>` (SyncThru Web Service) |

Open ports: `9100` raw printing · `9400` Samsung scanning · `631` IPP · `515` LPD · `80` web.

### The printer's address

Both scripts require the printer's address in the `SCX_IP` environment variable and
refuse to run without it — there is no default:

```bash
export SCX_IP=192.168.1.50      # in ~/.zshrc to make it permanent
```

Find the address in your router's DHCP lease list, or on the printer itself under
Menu › Network › Network Configuration, which prints a page with it.

SANE has no such variable: its copy of the address lives in
`/opt/homebrew/etc/sane.d/xerox_mfp.conf`, on the `tcp <address>` line, and that file
has to be edited by hand (see [Scanning](#scanning)). So a changed address means two
edits, `SCX_IP` and that line.

`192.168.1.50` is the placeholder used throughout this README — substitute your own.

---

## Installation

From a fresh macOS, with the printer already on the LAN. Apple Silicon paths; on an
Intel Mac replace `/opt/homebrew` with `/usr/local` here and everywhere below.

```bash
git clone <this-repo> && cd samsung-scx-macos

# 1. dependencies: scanimage comes from sane-backends, img2pdf merges the pages
brew install sane-backends img2pdf

# 2. the printer's address, everywhere below: use yours, not this placeholder
export SCX_IP=192.168.1.50

# 3. tell SANE where the scanner is — no autodiscovery on this model
echo "tcp $SCX_IP" >> /opt/homebrew/etc/sane.d/xerox_mfp.conf

# 4. create the print queue by hand (PCL, never PostScript — see below)
lpadmin -p Samsung_SCX4623FW -E \
  -v "socket://$SCX_IP:9100" \
  -m drv:///sample.drv/generpcl.ppd \
  -D "Samsung SCX-4623FW" -L "Local network" \
  -o PageSize=A4 -o Resolution=600dpi -o Option1=True \
  -o printer-is-shared=false
lpoptions -d Samsung_SCX4623FW     # make it the default printer

# 5. put the two commands on the PATH
ln -sf "$PWD/bin/scx-scan"     /opt/homebrew/bin/scx-scan
ln -sf "$PWD/bin/scx-diagnose" /opt/homebrew/bin/scx-diagnose
```

Make the `export` in step 2 permanent by adding it to `~/.zshrc`: the scripts read
`SCX_IP` on every run and stop with an error if it is unset.

Everything else the scripts use — `scanimage` aside, that is `lpstat`, `lpoptions`,
`cupsenable`, `cancel`, `sips`, `open`, `nc`, `/sbin/ping` — ships with macOS.

### Check it, then scan

```console
$ scx-diagnose
── Network ───────────────────────────────
  ✓ 192.168.1.50 reachable
  ✓ port 9100 open (print)
  ✓ port 9400 open (scan)
  ✓ port 80 open (web-interface)

── Printing ──────────────────────────────
  ✓ queue ready

── Scanning ──────────────────────────────
  ✓ scanner detected by SANE
```

All ✓ means both halves work. Then, a sheet on the glass:

```console
$ scx-scan
Glass, 300 dpi (Gray). Scanning...
Saved: /Users/you/Desktop/scan-20260919-104512.pdf
```

The PDF opens in Preview on its own. A stack in the feeder, in colour, into one file
in a folder of your choice, without opening it at the end:

```console
$ scx-scan -a -c -o ~/Documents -q
Feeder, 300 dpi (Color). Scanning...
  page 1
  page 2
  page 3
  end of stack
3 pages merged into a single PDF.
Saved: /Users/you/Documents/scan-20260919-110338.pdf
```

And printing is just CUPS, so from the terminal too:

```bash
lp ~/Documents/something.pdf          # to the default queue
lp -d Samsung_SCX4623FW -n 2 file.pdf # two copies, explicit queue
```

---

## The original problem

Printing did not work. Two causes, on top of each other:

1. **No printer was configured on the Mac.** `lpstat` answered `No destinations added`.
   The network had always been fine: the printer replied to ping in ~9 ms with zero
   packet loss.
2. **macOS 26 removed every Samsung driver.** Apple dropped third-party print drivers
   from the system, so the Settings "add printer" wizard found nothing to install.
   Samsung no longer exists as a printer maker either: the division was sold to HP
   in 2017.

---

## Printing

### Current configuration

| | |
|---|---|
| Queue name | `Samsung_SCX4623FW` (the Mac's default printer) |
| Connection | `socket://<printer-ip>:9100` (raw TCP) |
| Driver | **Generic PCL Laser Printer** (`drv:///sample.drv/generpcl.ppd`) |
| Paper size | A4 |
| Resolution | 600 dpi |
| Duplex | enabled |

Use it normally with `Cmd+P` from any application.

### Why PCL and not PostScript

Queried over IPP, the printer **claims** to support PostScript:

```
document-format-supported = application/octet-stream, application/postscript,
                            text/plain, application/vnd.hp-PCL
```

That is false. With the PostScript driver the printer received the data, made a hint of
a noise and produced nothing: it took the bytes and discarded them without reporting an
error. This model only has PCL emulation and Samsung's proprietary SPL language, not a
real PostScript interpreter.

The lesson: if one day a print job comes out as pages of code, or does not come out at
all, **look at the driver first**, not at the network.

### Recreating the queue from scratch

If the queue disappears or gets corrupted:

```bash
lpadmin -p Samsung_SCX4623FW -E \
  -v "socket://$SCX_IP:9100" \
  -m drv:///sample.drv/generpcl.ppd \
  -D "Samsung SCX-4623FW" -L "Local network" \
  -o PageSize=A4 -o Resolution=600dpi -o Option1=True \
  -o printer-is-shared=false
lpoptions -d Samsung_SCX4623FW     # make it the default
```

The `Printer drivers are deprecated` message is expected and harmless: CUPS is warning
that it will only accept driverless printers in the future. When that happens, this
model will need a different solution.

---

## Scanning

### Why Image Capture does not see it

The SCX-4623FW predates **AirScan/eSCL**, the standard that lets macOS scan without a
driver. Verified directly:

| Protocol | Result |
|---|---|
| eSCL on ports 80 and 8080 | `HTTP 404` — not implemented |
| Bonjour `_uscan._tcp` | no announcement |
| WSD (port 5357) | closed |
| Samsung Network Scan (port 9400) | **open and working** |

The only live channel is Samsung's proprietary protocol on port 9400.

### The solution used here

**SANE** (Scanner Access Now Easy) with the `xerox_mfp` backend, which implements exactly
that protocol. Installed through Homebrew:

```bash
brew install sane-backends
```

and the printer registered as a network device by adding one line to
`/opt/homebrew/etc/sane.d/xerox_mfp.conf`:

```
tcp 192.168.1.50
```

Result:

```
$ scanimage -L
device `xerox_mfp:tcp 192.168.1.50' is a Samsung SCX-4623FW Serie
multi-function peripheral
```

That file is outside this repository, is not version-controlled, and a
`brew reinstall sane-backends` wipes it — check it first when the scanner "disappears".

### What the scanner can actually do

- Resolutions: 75, 100, 150, 200, 300, 600, 1200 dpi
- Modes: Lineart, Halftone, Gray, **Color**
- Sources: glass (Flatbed) and **automatic document feeder (ADF)**
- Maximum area: 215.9 × 297.18 mm (A4)

Yes, it scans in colour even though it only prints in black and white.

---

## The commands

They live in `bin/` inside this repository and are reachable from anywhere in the
terminal through symlinks in `/opt/homebrew/bin`:

```bash
ln -sf "$PWD/bin/scx-scan"     /opt/homebrew/bin/scx-scan
ln -sf "$PWD/bin/scx-diagnose" /opt/homebrew/bin/scx-diagnose
```

(On an Intel Mac the prefix is `/usr/local` instead of `/opt/homebrew`, here and
everywhere below.)

### `scx-scan`

```bash
scx-scan                 # glass, greyscale 300 dpi → PDF on the Desktop
scx-scan -c              # colour
scx-scan -r 600          # 600 dpi (75|100|150|200|300|600|1200)
scx-scan -a              # feeder: every page into ONE PDF
scx-scan -o ~/Documents  # a different destination folder
scx-scan -q              # do not open the PDF when done
scx-scan -h              # help
```

Files come out as `scan-YYYYMMDD-HHMMSS.pdf` and open automatically in Preview, unless
`-q` is given.

### The automatic document feeder (ADF)

With `-a` the whole stack is scanned and the pages end up in **one multi-page PDF**, in
loading order. The merge is done by `img2pdf`, which embeds the JPEGs without
recompressing them: no quality loss.

How to load the sheets: **top** tray, text **face up**, top edge of the sheet first.
Scanning starts on its own and stops when the stack runs out — there is no need to say
how many pages there are.

With an empty tray the script says so plainly and produces nothing:

```
No page scanned: the feeder is empty.
Load the sheets in the top tray, face UP, and try again.
```

**The ADF is simplex only.** This model has no duplex feeder: the SANE backend only
exposes `Flatbed|ADF|Auto`, with no duplex scanning option. A double-sided document
needs two passes — all the fronts first, then flip the stack and do the backs — and you
get two PDFs to reorder by hand in Preview. If that comes up often, interleaving can be
automated: just ask.

Practical hint: for text documents `-a` on its own (grey, 300 dpi) is the right
compromise. Go up to `-r 600` only for very small print, because the PDF grows a lot and
scanning slows down.

### `scx-diagnose`

A quick check when something is wrong: it verifies reachability, ports, queue state,
stuck jobs and scanner detection.

```bash
scx-diagnose
```

---

## When it stops working

**Always start with** `scx-diagnose`. It tells the cases apart straight away.

| Symptom | Likely cause | Remedy |
|---|---|---|
| Address unreachable | printer off, or address changed | check the DHCP lease list on your router, then update `SCX_IP` |
| Queue "paused" | an earlier error disabled it | `cupsenable Samsung_SCX4623FW` |
| Jobs stuck in the queue | corrupted job at the head | `cancel -a Samsung_SCX4623FW` |
| Code instead of text | wrong driver | recreate the queue with the PCL command above |
| Silent print (noise, no page) | PostScript driver put back by mistake | same |
| Scanner not detected | line lost from the SANE config | put `tcp <printer-ip>` back in `/opt/homebrew/etc/sane.d/xerox_mfp.conf` |

After a major macOS update it is worth running `scx-diagnose` again: updates can wipe
the CUPS queues.

---

## Inventory of changes

Everything touched on the Mac, so it can be undone:

| Item | Path |
|---|---|
| CUPS print queue | `/etc/cups/printers.conf` + `/etc/cups/ppd/Samsung_SCX4623FW.ppd` |
| SANE package and dependencies | `/opt/homebrew/Cellar/sane-backends` (with libpng, giflib, webp, libtiff, libusb, net-snmp) |
| Multi-page PDF merging | `/opt/homebrew/Cellar/img2pdf` (with qpdf) |
| Line added to SANE | `/opt/homebrew/etc/sane.d/xerox_mfp.conf` (at the end) |
| Scripts | `bin/` in this repository |
| Symlinks in the PATH | `/opt/homebrew/bin/scx-scan`, `/opt/homebrew/bin/scx-diagnose` |

### Removing everything

```bash
lpadmin -x Samsung_SCX4623FW
rm -f /opt/homebrew/bin/scx-scan /opt/homebrew/bin/scx-diagnose
brew uninstall sane-backends img2pdf
```

---

## Notes

- **Moving this repository** means redoing the symlinks: `/opt/homebrew/bin` points at
  these paths. After a move, run the two `ln -sf` commands above again.
- **The fax** was not configured and does not go through here: use it from the printer's
  own panel or from the web interface.
- **A static address matters.** Without a DHCP reservation, `SCX_IP` and the SANE
  config have to be updated every time the lease changes.
