---
name: scx-diagnose
description: Diagnose why the Samsung SCX-4623FW will not print or scan on macOS. Use when a print job vanishes, prints garbage characters, the queue is paused or stuck, the scanner is not detected by SANE, or the printer is unreachable on the network.
---

# Printer and scanner diagnosis

Run the checks in order and stop at the first ✗ — later checks depend on earlier ones.

## 1. Run the script first

```sh
bin/scx-diagnose
```

It covers every check below and prints ✓/✗ per item. It needs macOS with the printer on
the LAN. If it is unavailable, run the commands in sections 2–4 by hand.

## 2. Network

```sh
/sbin/ping -c 2 -t 3 "$SCX_IP"
nc -z -G 2 "$SCX_IP" 9100   # printing
nc -z -G 2 "$SCX_IP" 9400   # scanning
nc -z -G 2 "$SCX_IP" 80     # SyncThru web UI
```

`SCX_IP` unset → the scripts stop with an error; there is no default address. Ask the
user for the printer's address rather than guessing one, and never write a real address
into the repo.

Unreachable → the printer is off, unplugged, or its address changed. Check the DHCP lease
list on the router. SANE keeps its own copy of the address, so a change means
`export SCX_IP=<new>` plus editing the `tcp <address>` line in
`/opt/homebrew/etc/sane.d/xerox_mfp.conf`.

## 3. Printing

```sh
lpstat -p Samsung_SCX4623FW
lpstat -o Samsung_SCX4623FW
```

| Symptom | Cause | Remedy |
|---|---|---|
| `disabled` | queue paused | `cupsenable Samsung_SCX4623FW` |
| jobs listed, nothing prints | stuck queue | `cancel -a Samsung_SCX4623FW` |
| code pages instead of text, or a noise and no page | PostScript driver | recreate the queue (below) |
| queue missing entirely | macOS upgrade wiped it | recreate the queue (below) |

Recreate the queue with the PCL driver — the printer advertises PostScript and that claim
is false:

```sh
lpadmin -p Samsung_SCX4623FW -E \
  -v "socket://$SCX_IP:9100" \
  -m drv:///sample.drv/generpcl.ppd \
  -D "Samsung SCX-4623FW" -L "Local network" \
  -o PageSize=A4 -o Resolution=600dpi -o Option1=True \
  -o printer-is-shared=false
lpoptions -d Samsung_SCX4623FW
```

CUPS answering `Printer drivers are deprecated` is expected and harmless.

## 4. Scanning

```sh
scanimage -L
```

Expected: `device 'xerox_mfp:tcp <printer-ip>' is a Samsung SCX-4623FW Serie`.

Not listed → check that `/opt/homebrew/etc/sane.d/xerox_mfp.conf` still contains the line
`tcp <printer-ip>`. That file is outside the repo, is not version-controlled, and a
`brew reinstall sane-backends` wipes it. This is the most common scanner failure.

Still nothing → confirm `brew list sane-backends` and that port 9400 answered in section 2.
There is no AirScan/eSCL fallback on this model: eSCL returns 404, there is no `_uscan._tcp`
Bonjour record, and WSD port 5357 is closed. Port 9400 through the `xerox_mfp` backend is
the only path.

## 5. Empty scans from the ADF

`scx-scan -a` exiting with "the feeder is empty" means the feeder is empty or the sheets
are loaded wrong: top tray, text face **up**, top edge first. The ADF is simplex only — a
double-sided original needs two passes.
